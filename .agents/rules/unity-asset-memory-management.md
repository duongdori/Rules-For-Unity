---
trigger: model_decision
description: "Asset and memory lifecycle management in Unity 6: Addressables standards, Resources.Load prohibition, and native buffer cleanup."
---

# Unity 6 Asset & Memory Management Standards

This rule governs asset loading, memory lifecycle, and unmanaged GPU/RAM resource cleanup to prevent memory bloat and application crashes.

---

## 1. Absolute Prohibition of `Resources.Load()`

Never place gameplay assets inside a `Resources/` folder or call `Resources.Load()`.
- **The Problem**: All assets in `Resources/` are bundled into the binary application payload. They cannot be stripped, bloat initial download sizes, and force Unity to build an in-memory metadata tree at startup, drastically degrading cold start times.
- **The Standard**: Use **Direct Serialized References** for small/core scenes, and **Unity Addressables** (`UnityEngine.AddressableAssets`) for dynamic, modular, or downloaded content.

---

## 2. Addressables & Asset Lifecycle

### A. AssetReference Loading with `Awaitable`
Always load addressables asynchronously and propagate `CancellationToken`:

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class AssetLoader : MonoBehaviour
{
    [SerializeField] private AssetReferenceGameObject _enemyPrefabRef;
    private AsyncOperationHandle<GameObject> _handle;

    public async Awaitable<GameObject> SpawnEnemyAsync(Vector3 position, Quaternion rotation)
    {
        _handle = _enemyPrefabRef.InstantiateAsync(position, rotation);
        await _handle.Task;

        if (_handle.Status == AsyncOperationStatus.Succeeded)
        {
            return _handle.Result;
        }

        Debug.LogError($"Failed to load addressable asset: {_enemyPrefabRef}");
        return null;
    }

    private void OnDestroy()
    {
        // MANDATORY: Release handle when component is destroyed
        if (_handle.IsValid())
        {
            Addressables.ReleaseInstance(_handle);
        }
    }
}
```

### B. Release Contract
Every loaded Addressable asset or instantiated instance **must have an explicit release lifecycle**:
- If loaded via `Addressables.LoadAssetAsync<T>()`: Call `Addressables.Release(handle)` when finished.
- If instantiated via `Addressables.InstantiateAsync()`: Call `Addressables.ReleaseInstance(instanceGameObject)` instead of `Destroy(gameObject)`.

---

## 3. Texture & Sprite Atlas Memory Invariants

### A. Disable Read/Write on Textures & Meshes
By default, keep **Read/Write Enabled** unchecked on all Textures, Models, and Sprites.
- When enabled, Unity retains two full copies in memory: one in system RAM and one in GPU VRAM (doubling memory footprint).
- Only check this box if C# CPU scripts must sample raw pixel data via `GetPixels()` or modify vertex buffers directly.

### B. Sprite Atlas v2 Batching
- Never leave isolated UI sprites unbundled.
- Group UI sprites into **Sprite Atlases** organized by screen or feature (e.g. `Atlas_MainMenu`, `Atlas_HUD`, `Atlas_Icons`).
- This ensures all sprites in a UI panel are batched into a single material draw call.

---

## 4. Unmanaged Resource Cleanup (RenderTexture & Procedural Mesh)

Objects generated procedurally at runtime are not automatically collected by the C# Garbage Collector:

```csharp
public class CustomRenderCamera : MonoBehaviour
{
    private RenderTexture _renderTexture;
    private Mesh _proceduralMesh;

    private void Awake()
    {
        _renderTexture = new RenderTexture(512, 512, 16, RenderTextureFormat.ARGB32);
        _proceduralMesh = new Mesh();
    }

    private void OnDestroy()
    {
        // MANDATORY: Release native GPU memory
        if (_renderTexture != null)
        {
            _renderTexture.Release();
            Destroy(_renderTexture);
            _renderTexture = null;
        }

        if (_proceduralMesh != null)
        {
            Destroy(_proceduralMesh);
            _proceduralMesh = null;
        }
    }
}
```
