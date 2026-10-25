---
trigger: glob
globs: "*.cs"
description: "Performance optimization, zero-allocation hot paths, memory management, and modern async patterns for Unity 6."
---

# Unity 6 Performance & Garbage Collection Optimization

This rule defines non-negotiable memory, CPU, and garbage collection (GC) optimization practices for Unity 6 C# development. High-frame-rate, stable games require strictly managed allocations and efficient algorithms.

---

## 1. Zero-Allocation Hot Path Discipline

### Definition of Hot Path
Any code executing per-frame or at high frequencies:
- `Update()`, `FixedUpdate()`, `LateUpdate()`
- Physics callbacks (`OnTriggerStay`, `OnCollisionStay`)
- Render loop hooks or animation callbacks
- Frequently fired game loop events (e.g., enemy ticks, particle triggers)

### Strict Prohibitions in Hot Paths:
1. **NO `new` allocations**: Never instantiate classes, arrays, or reference types inside hot paths.
2. **NO LINQ**: Disallow all LINQ queries (`.Where()`, `.Select()`, `.Any()`, `.First()`, `.ToList()`). LINQ creates hidden enumerator objects and delegates on the managed heap.
3. **NO String Concatenation**: Avoid string `+` or interpolated strings (`$"Health: {hp}"`) in hot paths.
4. **NO Boxing**: Avoid casting value types (structs, ints, enums) to `object` or interfaces.

```csharp
// FORBIDDEN: Allocates multiple objects every single frame
private void Update()
{
    // String allocation
    _healthText.text = "HP: " + _currentHealth;

    // LINQ allocation
    var aliveEnemies = _enemies.Where(e => e.IsAlive).ToList();

    // Anonymous lambda / closure allocation
    _enemies.ForEach(e => e.CheckDistance());
}

// CORRECT: Zero allocations in hot path
private void Update()
{
    // Use TextMeshPro's zero-alloc overload
    _healthText.SetText("HP: {0}", _currentHealth);

    // Standard index-based loop over preallocated collection
    for (int i = 0; i < _enemies.Count; i++)
    {
        var enemy = _enemies[i];
        if (enemy.IsAlive)
        {
            enemy.CheckDistance();
        }
    }
}
```

---

## 2. Modern Asynchronous Programming: Unity 6 `Awaitable` & Hybrid `UniTask` Interop

Traditional Unity Coroutines generate heap allocations each time a yield instruction is created (`new WaitForSeconds(...)`) and allocate state machine enumerators. Unity 6 introduces `UnityEngine.Awaitable`, a high-performance, struct/pooled async primitive tightly integrated with the C++ engine layer.

### A. Primary Architecture Standard: `UnityEngine.Awaitable`
- **Default Choice**: All standard gameplay routines, scene transitions, cooldown timers, frame-waits, and public API contracts must use `Awaitable` or `Awaitable<T>`.
- **Core Primitives**:
  - `Awaitable.NextFrameAsync(token)`: Wait for next frame without allocations.
  - `Awaitable.FixedUpdateAsync(token)`: Sync with the physics step.
  - `Awaitable.EndOfFrameAsync(token)`: Wait for rendering completion (screen captures, frame readback).
  - `Awaitable.WaitForSecondsAsync(duration, token)`: Timer delays without garbage collection.
  - `Awaitable.BackgroundThreadAsync()` & `Awaitable.MainThreadAsync()`: Thread context switching.

```csharp
public async Awaitable PerformCooldownAsync(float durationSeconds)
{
    var token = destroyCancellationToken;

    IsOnCooldown = true;
    try
    {
        await Awaitable.WaitForSecondsAsync(durationSeconds, token);
    }
    catch (OperationCanceledException)
    {
        // Handled gracefully when object is destroyed or disabled
        return;
    }
    finally
    {
        IsOnCooldown = false;
    }
}
```

### B. When to Introduce `UniTask` (Gaps `Awaitable` Does Not Solve)
While `Awaitable` handles 90%+ of gameplay tasks natively, `UniTask` is permitted as an **internal utility layer** for specific complex concurrency patterns that `Awaitable` lacks:

| Problem / Gap in `Awaitable` | `UniTask` Solution | Usage Guidelines |
| :--- | :--- | :--- |
| **Heterogeneous Multi-Task Combinators** (`Task<T1>`, `Task<T2>`) | `UniTask.WhenAll(...)` returns strongly-typed tuples: `(T1, T2) = await UniTask.WhenAll(task1, task2)` | Use inside method body to load concurrent data, but keep method return type as `Awaitable` |
| **Racing Asynchronous Operations** | `UniTask.WhenAny(...)` returning winner index or first result | Use for racing user input vs timeout, or multi-endpoint network requests |
| **Timeout & Advanced Cancellation** | `TimeoutController`, `CancellationTokenSource.CancelAfterSlim()` | Zero-alloc timeout handling without creating heavyweight .NET timer threads |
| **Pub/Sub Streams & Concurrency Channels** | `Channel<T>`, `AsyncReactiveProperty<T>`, Async LINQ | High-throughput decoupled data streaming between background workers and UI |
| **Leak Detection in Editor** | **UniTask Tracker Window** (`Window > UniTask Tracker`) | Audit pending tasks during development to verify cancellation token coverage |
| **Third-Party Async Wrappers** | Built-in extensions for DOTween (`tween.ToUniTask()`), Addressables | Eliminates manual completion source boilerplate for legacy third-party assets |

### C. The Bridge & Boundary Rules (Hybrid Architecture)
1. **Public Contract Invariant**: Interfaces, abstract methods, and public component methods must expose `Awaitable` or `Awaitable<T>`. Never leak `UniTask` into public domain signatures.
2. **Internal Utilization**: Await `UniTask` operations directly within an `async Awaitable` method:

```csharp
// CORRECT: Public contract is Awaitable; UniTask is used internally for tuple WhenAll
public async Awaitable<(UserProfile profile, GameInventory inventory)> LoadPlayerDataAsync(CancellationToken token)
{
    // UniTask handles parallel execution with strongly-typed tuple returns
    var (profile, inventory) = await UniTask.WhenAll(
        FetchProfileViaNetworkAsync(token),
        FetchInventoryFromDiskAsync(token)
    );

    return (profile, inventory);
}

// CORRECT: Timeout control via UniTask inside an Awaitable workflow
public async Awaitable<bool> TryConnectWithTimeoutAsync(float timeoutSeconds, CancellationToken token)
{
    using var timeoutController = new TimeoutController();
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(token, timeoutController.Timeout(TimeSpan.FromSeconds(timeoutSeconds)));

    try
    {
        await ConnectToServerAsync(linkedCts.Token);
        return true;
    }
    catch (OperationCanceledException) when (timeoutController.IsTimeout())
    {
        Debug.LogWarning("Connection attempt timed out.");
        return false;
    }
}
```

3. **CancellationToken Propagation**:
   - Every async method must accept a `CancellationToken` (defaulting to `default` or `destroyCancellationToken`).
   - Never use fire-and-forget `async void` unless handling an isolated top-level UI event handler wrapped in a `try/catch` block.

### D. Legacy Coroutine Exception:
If legacy Coroutines must be used, **never allocate yield instructions dynamically**:
```csharp
// FORBIDDEN
yield return new WaitForSeconds(0.5f);

// CORRECT: Pre-cache yield instructions
private static readonly WaitForSeconds WaitHalfSecond = new(0.5f);

private IEnumerator CooldownRoutine()
{
    yield return WaitHalfSecond;
}
```

---

## 3. Object Pooling with `UnityEngine.Pool`

Do not call `Instantiate()` or `Destroy()` repeatedly during gameplay. Allocate reusable pools for entities created and destroyed dynamically (projectiles, floating damage texts, visual effects, enemies).

### Implement Built-In `ObjectPool<T>`:
```csharp
using UnityEngine;
using UnityEngine.Pool;

public class ProjectileLauncher : MonoBehaviour
{
    [SerializeField] private Projectile _projectilePrefab;
    [SerializeField] private int _defaultPoolSize = 20;
    [SerializeField] private int _maxPoolSize = 100;

    private IObjectPool<Projectile> _projectilePool;

    private void Awake()
    {
        _projectilePool = new ObjectPool<Projectile>(
            createFunc: CreateProjectile,
            actionOnGet: OnGetFromPool,
            actionOnRelease: OnReleaseToPool,
            actionOnDestroy: OnDestroyPoolObject,
            collectionCheck: true,
            defaultCapacity: _defaultPoolSize,
            maxSize: _maxPoolSize
        );
    }

    private Projectile CreateProjectile()
    {
        var projectile = Instantiate(_projectilePrefab);
        projectile.SetPool(_projectilePool);
        return projectile;
    }

    private void OnGetFromPool(Projectile projectile) => projectile.gameObject.SetActive(true);
    private void OnReleaseToPool(Projectile projectile) => projectile.gameObject.SetActive(false);
    private void OnDestroyPoolObject(Projectile projectile) => Destroy(projectile.gameObject);

    public void Fire(Vector3 position, Quaternion rotation)
    {
        var projectile = _projectilePool.Get();
        projectile.transform.SetPositionAndRotation(position, rotation);
        projectile.Initialize();
    }
}
```

### Collection Pooling:
For temporary collections in methods, use `UnityEngine.Pool.ListPool<T>`, `DictionaryPool<K, V>`, or `HashSetPool<T>`:
```csharp
using UnityEngine.Pool;

public void ProcessEnemiesInRange(Vector3 center, float radius)
{
    // Rents list from pool without heap allocation
    using (ListPool<Enemy>.Get(out var nearbyEnemies))
    {
        PopulateEnemies(center, radius, nearbyEnemies);
        for (int i = 0; i < nearbyEnemies.Count; i++)
        {
            nearbyEnemies[i].Alert();
        }
    } // Automatically returned to pool on dispose
}
```

---

## 4. Non-Allocating Physics Queries

Standard physics queries (`Physics.RaycastAll`, `Physics.OverlapSphere`) allocate a new array on every call. Use the `NonAlloc` variants with a static or cached preallocated buffer.

```csharp
public class RadarScanner : MonoBehaviour
{
    private static readonly Collider[] ColliderBuffer = new Collider[32];

    public int ScanSurroundings(Vector3 center, float radius, LayerMask mask)
    {
        // Zero GC allocation
        int hitCount = Physics.OverlapSphereNonAlloc(center, radius, ColliderBuffer, mask);

        for (int i = 0; i < hitCount; i++)
        {
            var hitCollider = ColliderBuffer[i];
            // Process collider hit
        }

        // Clean buffer references if retaining large objects
        System.Array.Clear(ColliderBuffer, 0, hitCount);
        return hitCount;
    }
}
```

---

## 5. Math & Transform Optimizations

### A. Distance Comparisons: Always Use `sqrMagnitude`
Calculating distance via `Vector3.Distance(a, b)` or `vector.magnitude` executes an expensive square root (`Mathf.Sqrt`). When comparing distances, compare against the squared threshold.

```csharp
// FORBIDDEN: Incurs Mathf.Sqrt every check
if (Vector3.Distance(transform.position, target.position) < detectionRadius)

// CORRECT: Pure multiplication, zero square root overhead
float sqrRadius = detectionRadius * detectionRadius;
if ((transform.position - target.position).sqrMagnitude < sqrRadius)
{
    // In range
}
```

### B. Transform Operations: Batch Position and Rotation
Setting `transform.position` and `transform.rotation` separately forces Unity to recalculate the internal transform matrix twice. Use `SetPositionAndRotation()`:

```csharp
// FORBIDDEN: Two matrix recalculations
transform.position = newPos;
transform.rotation = newRot;

// CORRECT: Single matrix recalculation
transform.SetPositionAndRotation(newPos, newRot);
```

### C. Cache `Camera.main`
`Camera.main` internally performs an expensive `GameObject.FindWithTag("MainCamera")` call unless cached. Store the main camera reference in `Awake()` or access it via a dedicated singleton manager.

---

## 6. Rendering, Shaders & Animation Optimization

### A. Avoid `renderer.material` Leak (The Hidden Instantiation Trap)
Accessing `renderer.material` causes Unity to silently instantiate an entire duplicate copy of the Material on the heap and GPU. This breaks SRP Batcher / GPU Instancing batches and leaks memory unless manually destroyed.
- **Rule**: Never call `renderer.material` to modify properties per-object.
- **Correct**: Use a cached `MaterialPropertyBlock` to modify per-instance properties without cloning materials or breaking GPU instancing.

```csharp
public class EntityHighlighter : MonoBehaviour
{
    private static readonly int BaseColorPropertyId = Shader.PropertyToID("_BaseColor");

    [SerializeField] private Renderer _meshRenderer;
    private MaterialPropertyBlock _propertyBlock;

    private void Awake()
    {
        _propertyBlock = new MaterialPropertyBlock();
    }

    public void SetHighlightColor(Color color)
    {
        // Zero allocations, does not clone material, maintains SRP Batching
        _meshRenderer.GetPropertyBlock(_propertyBlock);
        _propertyBlock.SetColor(BaseColorPropertyId, color);
        _meshRenderer.SetPropertyBlock(_propertyBlock);
    }
}
```

### B. Cache Shader Property IDs via `Shader.PropertyToID`
Passing string names to materials or property blocks (`material.SetFloat("_Emission", 1f)`) calculates a 32-bit integer string hash on every call.
- **Rule**: Store property hashes in `static readonly int` variables:
```csharp
private static readonly int EmissionIntensityId = Shader.PropertyToID("_EmissionIntensity");
private static readonly int DissolveAmountId = Shader.PropertyToID("_DissolveAmount");
```

### C. Cache Animator Parameters via `Animator.StringToHash`
Passing string names to `Animator` (`animator.SetTrigger("Attack")` or `animator.Play("Walk")`) incurs string hashing overhead on every invocation.
- **Rule**: Pre-calculate and cache all Animator parameter and state hashes:
```csharp
public static class PlayerAnimationHashes
{
    public static readonly int MoveSpeed = Animator.StringToHash("MoveSpeed");
    public static readonly int IsGrounded = Animator.StringToHash("IsGrounded");
    public static readonly int AttackTrigger = Animator.StringToHash("Attack");
}
```
