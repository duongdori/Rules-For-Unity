---
trigger: manual
description: "Pre-release audit and build verification checklist for Unity 6 (IL2CPP, asset stripping, profiling, missing references)."
---

# Unity 6 Pre-Release & Build Audit Checklist

This rule provides a verification checklist to audit Unity 6 projects prior to producing release builds, App Store submissions, or milestone deliverables.

> **Activation**: Trigger this rule manually at any time by mentioning `@unity-audit-release` in the chat prompt.

---

## 1. Scene & Asset Integrity Audit

- [ ] **Missing Scripts Check**: Scan all active Scenes and Prefab assets for `Missing (Mono Script)`. Unresolved missing scripts corrupt scene serialization and fail builds.
- [ ] **Unassigned Serialized References**: Verify all non-optional `[SerializeField]` fields on scene game objects and prefabs are assigned.
- [ ] **Duplicate Asset Names**: Ensure Addressable / Resource assets do not contain duplicate keys or unreferenced dependencies.

---

## 2. Profiler & Garbage Collection (GC) Verification

- [ ] **Zero GC in Gameplay Loop**: Attach the Unity Profiler to a standalone development build. Record 60 seconds of core gameplay:
  - **GC Alloc must read `0 B`** in all hot frames.
  - No frequent spikes in the GC allocation column.
- [ ] **Frame Budget Compliance**:
  - **60 FPS target**: Total CPU frame time must remain under `16.6 ms`.
  - **120 FPS target**: Total CPU frame time must remain under `8.3 ms`.
- [ ] **Physics Tick Budget**: `FixedUpdate` execution must stay well below `fixedDeltaTime` (typically `20 ms` for 50Hz) to prevent the "physics spiral of death".

---

## 3. IL2CPP & Managed Code Stripping

- [ ] **Scripting Backend**: Verify **IL2CPP** is selected for Release builds (Project Settings -> Player -> Other Settings -> Scripting Backend).
- [ ] **Managed Stripping Level**: Configured to **Medium** or **High**.
- [ ] **`link.xml` Verification**: Ensure any classes instantiated via reflection, serialization libraries (e.g. `Newtonsoft.Json`), or dynamic plugins are protected from bytecode stripping:
  ```xml
  <linker>
    <assembly fullname="Game.Runtime" preserve="all"/>
    <assembly fullname="UnityEngine.CoreModule">
      <type fullname="UnityEngine.GameObject" preserve="all"/>
    </assembly>
  </linker>
  ```

---

## 4. Textures, Models & Audio Compression

- [ ] **Texture Compression Targets**:
  - **Mobile (Android/iOS)**: Textures must use **ASTC** (Block size: 4x4 or 6x6 for albedo; 8x8 for UI/lightmaps).
  - **PC / Console**: Textures must use **BC7** or **BC3** compression.
  - Textures must have `Generate Mip Maps` enabled for 3D world elements, disabled for 2D UI sprites.
- [ ] **Mesh Compression**: Static environment props have mesh compression and read/write disabled unless procedural mesh access is strictly required.
- [ ] **Audio Compression**:
  - Short SFX (< 2s) set to `Decompress On Load`.
  - Medium sounds (2-10s) set to `Compressed In Memory`.
  - Music & Ambience (> 10s) set to `Streaming`.

---

## 5. Shader Stripping & Render Pipeline (URP / HDRP)

- [ ] **Shader Variant Stripping**: Verify shader stripping is enabled in Project Settings -> Graphics -> Shader Stripping.
- [ ] **Unused Render Features**: Remove unused URP Renderer Features (e.g., Decal, Screen Space Ambient Occlusion, RenderGraph passes) if not actively utilized in current scenes.
- [ ] **Overdraw Inspection**: Inspect high-density particle effects and transparent alpha blended geometry with the Overdraw view mode.

---

## 6. Build Settings & Security Cleanup

- [ ] **Development Build**: Unchecked in Build Settings.
- [ ] **Autoconnect Profiler & Deep Profiling**: Unchecked.
- [ ] **Debug Logging Stripping**: Ensure `Debug.Log` statements are stripped or silenced in production:
  ```csharp
  #if !DEVELOPMENT_BUILD && !UNITY_EDITOR
  Debug.unityLogger.logEnabled = false;
  #endif
  ```
- [ ] **Target Architecture**:
  - Android: ARM64 enabled (ARMv7 deprecated). Target SDK matches latest Google Play requirements.
  - iOS: Target minimum iOS version configured; arm64 architecture selected.
  - Windows: x86_64 architecture selected.
