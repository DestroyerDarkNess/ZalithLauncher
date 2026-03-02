# ZalithLauncher — Android 11+ Crash Fixes

## Problema Original
La aplicación crasheaba con `java.io.IOException: Unable to create target directory` en dispositivos Android 11+ al intentar instalar componentes y descargar versiones de Minecraft. El problema raíz es que Android 11+ introdujo **Scoped Storage** y restricciones **SELinux/FUSE** que impiden la creación de directorios en rutas de almacenamiento externo que antes eran accesibles.

---

## Bug #1: Falta solicitud del permiso `MANAGE_EXTERNAL_STORAGE`

**Síntoma:** La app nunca pedía el permiso de "Acceso a todos los archivos" en Android 11+.

**Causa:** `SplashActivity.kt` solo manejaba el permiso legacy `WRITE_EXTERNAL_STORAGE` para Android ≤10.

**Solución en `SplashActivity.kt`:**
- Se usó `StoragePermissionsUtils.checkPermissions()` que maneja ambos casos (Android 10- y 11+).
- Se añadió `onActivityResult` para reanudar el flujo de la app al volver de la pantalla de configuración de permisos.

---

## Bug #2: SELinux bloquea creación de directorios en la raíz de datos de la app

**Síntoma:** `IOException: Unable to create target directory` durante la extracción de componentes.

**Causa:** `PathManager.DIR_DATA` estaba configurado como `context.filesDir.getParent()` → `/data/user/0/<pkg>/`. Android 11+ SELinux bloquea la creación de nuevos directorios directamente en esta raíz.

**Solución en `PathManager.kt`:**
```diff
-DIR_DATA = DIR_FILE.getParent()!!
+DIR_DATA = DIR_FILE.absolutePath
```

**Solución en `PojavApplication.java`:**
```diff
-PathManager.DIR_DATA = getDir("files", MODE_PRIVATE).getParent();
+PathManager.DIR_DATA = getFilesDir().getAbsolutePath();
```

> **IMPORTANTE:** `getDir("files", MODE_PRIVATE)` devuelve `app_files/` mientras que `getFilesDir()` devuelve `files/`. Son **directorios completamente distintos**. Usar el incorrecto causaba que las cuentas se guardaran en un lugar y se buscaran en otro (Bug #8).

---

## Bug #3: Condición de carrera FUSE al recrear directorios

**Síntoma:** `IOException` cuando `UnpackComponentsTask` intentaba borrar y recrear un directorio inmediatamente después.

**Causa:** `requestEmptyParentDir()` llamaba a `FileUtils.deleteDirectory()` y luego `mkdirs()`. En FUSE (Android 11+), el caché de dentry del SO aún considera el directorio "borrado" cuando `mkdirs()` se ejecuta, causando que devuelva `false`.

**Solución en `UnpackComponentsTask.kt`:**
```diff
-if (exists() and isDirectory) {
-    FileUtils.deleteDirectory(this)
-}
-mkdirs()
+if (exists() && isDirectory) {
+    runCatching { FileUtils.cleanDirectory(this) }
+} else {
+    mkdirs()
+}
```

---

## Bug #4: `AccessDeniedException` en almacenamiento externo

**Síntoma:** `AccessDeniedException: /storage/emulated/0/Android/data/<pkg>` al crear directorios dentro del directorio de archivos externos de la app.

**Causa:** Tanto `File.mkdirs()` como `Files.createDirectories()` recurren internamente **HACIA ARRIBA** en el árbol de directorios. Cuando llegan al directorio del paquete (`/Android/data/<pkg>`), que está por encima del límite de FUSE, `exists()` devuelve `false` y `mkdir()` falla. Toda la cadena colapsa.

**Solución — Todos los componentes se movieron a almacenamiento interno:**

**En `Components.kt`:**
```diff
-OTHER_LOGIN("other_login", ..., false),
-CACIOCAVALLO("caciocavallo", ..., false),
-CACIOCAVALLO17("caciocavallo17", ..., false),
-LWJGL3("lwjgl3", ..., false),
+OTHER_LOGIN("other_login", ..., true),
+CACIOCAVALLO("caciocavallo", ..., true),
+CACIOCAVALLO17("caciocavallo17", ..., true),
+LWJGL3("lwjgl3", ..., true),
```

**En `PathManager.kt` — `getExternalStorageRoot()`:**
```diff
 return if (VERSION.SDK_INT >= 29) {
-    ctx.getExternalFilesDir(null)!!
+    File(ctx.filesDir, "games")
 }
```

---

## Bug #5: `FileUtils.ensureDirectory` no es suficientemente robusto

**Síntoma:** Múltiples `IOException: Unable to create target directory` en diferentes partes del código.

**Causa:** Múltiples problemas en el `ensureDirectory()` original:
1. Confiaba únicamente en el valor de retorno de `mkdirs()` (poco fiable en FUSE).
2. Lanzaba excepción si un archivo ocupaba el nombre del directorio objetivo.
3. Se usó `NIO Files.createDirectories()` que recorre todo el árbol de rutas y lanza `AccessDeniedException` en directorios padre restringidos.

**Solución en `FileUtils.java`:**
- Se añadió un método helper `createDirectoriesFromAncestor()`: sube HACIA ARRIBA para encontrar el ancestro accesible más profundo, luego crea directorios UNO A UNO HACIA ABAJO con `mkdir()`.
- Si `targetFile.isFile()`, lo elimina primero en lugar de lanzar excepción.
- Fallback a creación basada en ancestros si `mkdirs()` falla.

---

## Bug #6: `checkStorageRoot()` falla para almacenamiento interno

**Síntoma:** "Zalith Launcher requires external storage" al abrir la app.

**Causa:** `Tools.checkStorageRoot()` usaba `Environment.getExternalStorageState()` que solo funciona para rutas de almacenamiento externo. Después de mover `DIR_GAME_HOME` a almacenamiento interno, siempre devolvía "no montado".

**Solución en `Tools.java`:**
```diff
 public static boolean checkStorageRoot() {
     File externalFilesDir = new File(PathManager.DIR_GAME_HOME);
+    if (externalFilesDir.getAbsolutePath().startsWith(PathManager.DIR_DATA)) {
+        externalFilesDir.mkdirs();
+        return externalFilesDir.isDirectory() && externalFilesDir.canWrite();
+    }
     return Environment.getExternalStorageState(externalFilesDir)
         .equals(Environment.MEDIA_MOUNTED);
 }
```

---

## Bug #7: `NullPointerException` — Ninguna cuenta seleccionada

**Síntoma:** Crash `NullPointerException` en `LaunchGame.kt:127` al lanzar Minecraft.

**Causa:** `AccountsManager.currentAccount!!` hacía force-unwrap de un valor null. La instalación limpia con las nuevas rutas de almacenamiento interno no tenía cuentas guardadas.

**Solución en `LaunchGame.kt`:**
```diff
-var account = AccountsManager.currentAccount!!
+var account = AccountsManager.currentAccount ?: run {
+    activity.runOnUiThread {
+        Toast.makeText(activity, "No account selected...", Toast.LENGTH_LONG).show()
+    }
+    return
+}
```

---

## Bug #8: Las cuentas no persisten / Desajuste de rutas

**Síntoma:** Las cuentas locales aparecían en la UI pero desaparecían tras reiniciar. El juego mostraba "No account selected" incluso con una cuenta visible.

**Causa:** Dos problemas:
1. `PojavApplication.java` usaba `getDir("files", MODE_PRIVATE)` → `app_files/`, mientras que `PathManager.kt` usaba `context.filesDir` → `files/`. Las cuentas se guardaban en `app_files/accounts/` pero se buscaban en `files/accounts/`.
2. `PathManager.initContextConstants()` tenía una línea que **BORRABA** `$DIR_DATA/accounts` en cada inicialización.

**Solución:**
- Se cambió `PojavApplication` para usar `getFilesDir()` (ver Bug #2).
- Se eliminó la llamada `deleteQuietly` sobre el directorio de cuentas en `PathManager.kt`.
- Se cambió `DIR_APP_CACHE` de `context.externalCacheDir!!` a `context.cacheDir`.

---

## Bug #9: Referencias restantes al almacenamiento externo tras la migración

**Síntoma:** `StringIndexOutOfBoundsException: -1` en `Tools.getLWJGL3ClassPath()` y datos faltantes de componentes tras la migración a almacenamiento interno.

**Causa:** Varios archivos aún referenciaban `DIR_GAME_HOME` para componentes que habían sido movidos a `DIR_DATA`.

**Soluciones:**

| Archivo | Cambio |
|---|---|
| `PathManager.kt` | `DIR_APP_CACHE` → `context.cacheDir`, default `DIR_GAME_HOME` → vacío |
| `LibPath.kt` | `other_login`, `caciocavallo`, `caciocavallo17` → `DIR_DATA` |
| `Tools.java` | `getLWJGL3ClassPath()` → `DIR_DATA` + guardia contra lista vacía de jars |
| `FilesFragment.kt` | Botón "Software Private" → `DIR_GAME_HOME` |

---

## Archivos Modificados (Lista Completa)

| Archivo | Cambios |
|---|---|
| `SplashActivity.kt` | Solicitud de permiso + `onActivityResult` |
| `PathManager.kt` | `DIR_DATA` → `filesDir`, `DIR_GAME_HOME` → interno, `DIR_APP_CACHE` → interno, eliminada borrado de cuentas |
| `PojavApplication.java` | `getDir("files")` → `getFilesDir()` |
| `Components.kt` | Todos `privateDirectory = true` |
| `UnpackComponentsTask.kt` | `cleanDirectory` en lugar de `deleteDirectory` |
| `FileUtils.java` | `ensureDirectory` robusto con creación basada en ancestros |
| `Tools.java` | `checkStorageRoot()` + `getLWJGL3ClassPath()` |
| `LibPath.kt` | Rutas de componentes → `DIR_DATA` |
| `LaunchGame.kt` | Chequeo null-safe de cuenta |
| `FilesFragment.kt` | Botón Software Private → `DIR_GAME_HOME` |
