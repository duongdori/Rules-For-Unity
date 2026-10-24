---
trigger: model_decision
description: "High-performance computing standards for Unity 6: C# Job System, Burst Compiler, and Native Collections memory safety."
---

# Unity 6 C# Job System & Burst Compiler Standards

This rule governs multithreaded, high-throughput data processing using Unity's C# Job System, Burst Compiler, and the `Unity.Collections` package.

---

## 1. When to Use Jobs & Burst

- **MANDATORY FOR**:
  - Heavy mathematical computations (spatial partitioning, distance sorting, boids flocking, procedural mesh generation).
  - Raycast or physics batching (`Physics.RaycastCommand`).
  - Grid pathfinding (A* path calculations over large maps).
- **FORBIDDEN FOR**:
  - Direct manipulation of `UnityEngine.Object` references (`Transform`, `GameObject`, `AudioSource`). Jobs execute on worker threads where the Unity Engine Object API is inaccessible.

---

## 2. Burst Compilation Constraints

Mark all jobs with `[BurstCompile]`. The Burst Compiler compiles pure C# into optimized native SIMD machine code.

### Invariants for Burst Code:
1. **Unmanaged Data Only**: Job structs must only contain blittable value types (`float`, `int`, `bool`, structs of primitives) and `Unity.Collections` types (`NativeArray<T>`).
2. **No Managed References**: Absolutely no classes, strings, arrays (`T[]`), or interfaces.
3. **Use `Unity.Mathematics`**: Favor `float3`, `quaternion`, and `math.*` functions over `UnityEngine.Vector3` and `Mathf` for peak vectorization.

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;
using Unity.Mathematics;

[BurstCompile(CompileSynchronously = true, FloatMode = FloatMode.Fast)]
public struct CalculateDistancesJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float3> Positions;
    [ReadOnly] public float3 TargetPosition;
    [WriteOnly] public NativeArray<float> OutputDistances;

    public void Execute(int index)
    {
        // Zero allocations; SIMD-optimized vector math
        OutputDistances[index] = math.distance(Positions[index], TargetPosition);
    }
}
```

---

## 3. Native Collections & Allocator Lifecycle

Failure to dispose a `NativeArray` creates a native C++ memory leak that the managed Garbage Collector cannot reclaim.

### Allocator Selection Guide:
| Allocator | Lifetime Budget | Typical Use Case |
| :--- | :--- | :--- |
| `Allocator.Temp` | Single frame (must not outlive function scope) | Short-lived calculations in local methods |
| `Allocator.TempJob` | Max 4 frames | Asynchronous jobs scheduled across 1–3 frames |
| `Allocator.Persistent` | Lifetime of scene or application | Static buffers, entity pools, long-lived meshes |

### Safe Disposal Pattern:
```csharp
public class SpatialSystem : MonoBehaviour
{
    private NativeArray<float3> _positions;

    private void Awake()
    {
        _positions = new NativeArray<float3>(1000, Allocator.Persistent);
    }

    private void OnDestroy()
    {
        // MANDATORY: Check IsCreated and dispose
        if (_positions.IsCreated)
        {
            _positions.Dispose();
        }
    }
}
```

---

## 4. Scheduling & Synchronization Discipline

Avoid calling `.Complete()` immediately after `.Schedule()`. Immediate completion blocks the main thread, causing a pipeline stall that nullifies multithreading benefits.

### The Correct Frame-Stagger Pattern:
1. **Schedule in `Update()`**: Fire background work across worker threads.
2. **Complete in `LateUpdate()`**: Wait for results only when they are required for rendering.

```csharp
public class EntityProcessor : MonoBehaviour
{
    private JobHandle _jobHandle;
    private NativeArray<float> _results;

    private void Update()
    {
        var job = new MyHeavyJob { Output = _results };
        // Dispatch to worker threads
        _jobHandle = job.Schedule(_results.Length, 64);
    }

    private void LateUpdate()
    {
        // Wait for workers to finish before consuming results
        _jobHandle.Complete();
        ConsumeResults(_results);
    }
}
```
