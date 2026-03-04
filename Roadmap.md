# Zalith Launcher Evolution Roadmap (Core Engine & Patches)

This roadmap outlines the major **engine-level improvements, bug fixes, and compatibility patches** introduced in **Zalith Launcher 2**, explicitly excluding UI overhauls. It serves as a guide for porting these back-end stability enhancements into our current v1 project.

## ⚙️ Core Engine Stability & Fixes

These are critical patches addressing crashes or logical failures in the game installation and launch process:

- [ ] **Android 11+ App Storage Migration**: Implement internal storage migration and robust file operations (similar to what we partially addressed in v1) directly in `PathManager`.
- [ ] **Native Dependencies Downloader**: Fix missing handling of native libraries by explicitly extracting `.so` native libraries from downloaded artifacts before launch.
- [ ] **Forge 1.7.2 Sorting Patch**: Enable patched sorting functions specifically for Forge 1.7.2 to prevent launch crashes caused by invalid Java Environments being sorted incorrectly.
- [ ] **Missing ImGui-Java Library**: Fix issue causing the **Axiom mod** to crash/fail to run by ensuring `imgui-java` library is not filtered out during launch.
- [ ] **JRE Dynamic Linking Patch**: Resolve crash caused by `dlopen` null renderer library upon installing/using specific JRE outputs (Incorrect ABI output).
- [ ] **Fix Service Binding Crash**: Resolve Android 14+ `GameService` crash due to incorrect foreground service types not meeting restrictions.
- [ ] **ByteHook Exit Hook Extraction**: Move the exit hook into a separate library (`libbytehook.so`) to fix namespacing issues where the execution hook wasn't found when loaded by LWJGL.
- [ ] **Robust Process Phoenix Restarting**: Utilize Process Phoenix to safely restart the app logic after changing critical environment variables or language settings, avoiding context leaks.

## 🧩 Compatibility Parity & Mod Support

These patches address specific bugs preventing certain mods, versions, or renderers from functioning correctly:

- [ ] **Sodium/Fabric Optimizations Warnings Disable**: Patch startup flags or mod checks to gracefully disable/bypass warning screens injected by Sodium that can freeze the launcher on Android.
- [ ] **Angelica Mod Crash**: Apply specific patches to resolve Null Pointer Exceptions or classloading issues caused by the Angelica mod.
- [ ] **Lunar Client Internal LWJGL Conflict**: Fix crash when launching Lunar Client caused by conflicts between Lunar's internal LWJGL version and the launcher's wrapped LWJGL implementations.
- [ ] **YSM Mod Compatibility Check**: Enhance the version compatibility check for the Yggdrasil Skin Server Mod (YSM) so it doesn't fail parsing custom player data.
- [ ] **GL4ES Initialization Shift**: Move GL4ES initialization code earlier in the process to prevent white screens when using the VirGL or Mesa driver payloads.
- [ ] **XaeroPlus Mod SQLite Fix**: Update `MioLibPatcher` to ensure XaeroPlus mod can load its SQLite databases without `UnsatisfiedLinkError` on Android.
- [ ] **RandomPatches Display Creation Fix**: Return the actual display creation state in `lwjglx` instead of overriding it to always return `true`, which directly fixes startup crashes with the RandomPatches mod.
- [ ] **NeoForge Identification**: Fix parsing logic that previously caused NeoForge version strings or artifact metadata to be misidentified as standard Forge.
- [ ] **OpenAL Updating**: Update OpenAL Natives mapping to resolve sound glitches and incompatibility issues with mods like `soundfilters`.

## 📦 Resource Download & Asset Management

Backend fixes for the download fragment that resolve parsing, extraction, and installation issues:

- [ ] **Modpack Import Crash Fix**: Resolve `NullPointerException` or zip traversal crash when importing `.zip` Modpacks by thoroughly verifying the bundle structure and extracting safely.
- [ ] **Offline Library Checking**: Resolve an issue where the launcher fails to check the size or integrity of local libraries when completely disconnected from the network.
- [ ] **Parallel Download Timeout**: Set explicit read/write timeouts for file network streams to prevent infinite blocking/hanging when Modrinth/CurseForge APIs throttle requests.
- [ ] **Invalid Filename Scrubbing**: Enhance `File Management` to perform comprehensive illegality checks, explicitly disallowing inputs with `/` or reserved characters when naming instances, preventing filesystem corruption.
- [ ] **Prevent Mod/Version Overwriting**: Add checks in `ensureDirectory` to reject downloading or generating a profile if the target folder name already exists as a non-working file.

## 🛠️ Proposed Porting Strategy (To v1)

Since these are pure logic/engine fixes, they can be directly ported to v1 without touching the XML layouts. 

### Phase 1: High-Priority Crash Fixes
- Implement the **Native Dependencies Downloader** extraction logic for `.so` files.
- Integrate the updated `MioLibPatcher` `.jar` into our assets to address XaeroPlus and NeoForge instantly.
- Hand-port the **Lunar Client** and **RandomPatches** LWJGL initialization bypasses.

### Phase 2: Mod Compatibility Layer
- Implement the specific checks for **Sodium Warnings**.
- Adjust the OpenAL binaries lookup table to resolve sound mods (like `soundfilters`).
- Add parsing for **NeoForge** strings in the version identifier logic.

### Phase 3: Infrastructure Resilience
- Backport the strict filename validation (`/` bans, duplicate folder checks).
- Force exit timeouts on API network calls to solve the "infinite loading" bug on slow connections.
- Cleanly separate the JRE `dlopen` call as seen in the v2 commits.
