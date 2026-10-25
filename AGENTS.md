# Unity 6 C# Development Standards & Project Rules

This repository defines the core architecture, coding standards, and invariant rules for Unity 6+ game development using C#. All agents and contributors must strictly adhere to the standards outlined below.

---

## 1. Core Principles & Invariants

- **Engine Target**: Unity 6 (6000.x.x) or higher. Leverage modern Unity 6 features such as `UnityEngine.Awaitable`, UI Toolkit, modern Input System, and RenderGraph.
- **Zero-Allocation Hot Paths**: No heap allocations (`new`, boxing, closures, string formatting, LINQ) inside frame loops (`Update`, `FixedUpdate`, `LateUpdate`).
- **Assembly Isolation**: Strictly isolate code using Assembly Definitions (`.asmdef`). Never compile Editor code into Runtime assemblies.
- **Explicit Serialization**: Prefer `[SerializeField] private` fields over `public` variables. Maintain clear separation between inspector-configured data and runtime state.
- **Decoupled Architecture**: Favor event-driven architectures and ScriptableObject data containers over direct singleton references or tight coupling.

---

## 2. Standard Directory & Assembly Layout

All game code and assets must follow a structured hierarchy with dedicated `.asmdef` files:

```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Runtime/             # Runtime gameplay & system scripts (Game.Runtime.asmdef)
│   │   │   ├── Core/            # Foundation, singletons/service containers, interfaces
│   │   │   ├── Gameplay/        # Characters, mechanics, physics, state machines
│   │   │   ├── UI/              # UI Toolkit views, presenters, UGUI controllers
│   │   │   └── Data/            # ScriptableObjects, config structures, save models
│   │   ├── Editor/              # Custom inspectors, windows, drawers (Game.Editor.asmdef)
│   │   └── Tests/               # Unit and integration tests (Game.Tests.asmdef)
│   ├── Art/                     # Textures, models, materials, animations
│   ├── Audio/                   # Sound effects, music tracks, audio mixer presets
│   ├── Prefabs/                 # Game object prefabs organized by category
│   └── Settings/                # URP/HDRP configs, Input Actions, project settings
```

### Assembly Definition Rules
1. `Game.Runtime`: Contains all gameplay and engine logic. Does **not** reference any Editor assemblies.
2. `Game.Editor`: Set platform target to `Editor` only. References `Game.Runtime`.
3. `Game.Tests`: Set platform target to `Editor` and test runner platforms. References test frameworks and runtime code.

---

## 3. High-Priority Invariants

### A. MonoBehaviour Lifecycle Rules
- **No empty magic methods**: Never declare empty `Awake()`, `Start()`, `Update()`, etc. Unity's native-to-managed invocation incurs measurable overhead even if the method body is empty.
- **Initialization separation**: Use `Awake()` strictly for self-contained initialization and caching local components (`GetComponent<T>()`). Use `Start()` for cross-object references and wiring external dependencies.
- **Event unsubscription**: Always unsubscribe from C# events, UnityEvents, or Action delegates in `OnDisable()` or `OnDestroy()` to prevent memory leaks and dangling handlers.

### B. Unity Object Lifetime ("Fake Null" Trap)
- `UnityEngine.Object` overrides the `==` and `!=` operators to check whether the underlying native engine C++ object has been destroyed.
- **Do not use null-coalescing (`??`) or null-conditional (`?.`) operators on `UnityEngine.Object` subclasses** (e.g. `MonoBehaviour`, `ScriptableObject`, `Transform`, `GameObject`).
- **Correct**:
  ```csharp
  if (target != null)
  {
      target.Execute();
  }
  ```
- **Forbidden**:
  ```csharp
  // Bypasses Unity's custom equality check and causes unexpected MissingReferenceException
  target?.Execute();
  var obj = target ?? fallbackObj;
  ```

### C. Modern Concurrency: Unity 6 `Awaitable` & Hybrid `UniTask` Interop
- **Primary Standard**: Use `UnityEngine.Awaitable` / `Awaitable<T>` as the default first-class async standard for all public APIs, game services, and MonoBehaviour lifecycle routines in Unity 6+.
- **Zero Coroutines**: Eliminate legacy `IEnumerator` coroutines.
- **Hybrid `UniTask` Interoperability**:
  - `UniTask` is permitted as an **internal utility layer** for complex async operations that `Awaitable` does not provide natively:
    1. Heterogeneous tuple multi-task combinators (`UniTask.WhenAll(Task<A>, Task<B>)`).
    2. Task racing & timeout control (`UniTask.WhenAny`, `TimeoutController`).
    3. Advanced reactive data flows (`Channel<T>`, `AsyncReactiveProperty<T>`, Async LINQ).
    4. Task leak diagnostics using the **UniTask Tracker Window** in Editor.
  - **Contract Boundary Invariant**: Public API signatures and component interfaces must expose `Awaitable` / `Awaitable<T>`. Confine `UniTask` usage to method internals and bridge back to `Awaitable`.
- **Cancellation Propagation**: Always pass and check `CancellationToken` (prefer `destroyCancellationToken` in MonoBehaviours). Never trigger unmonitored `async void` fire-and-forget tasks without try/catch error boundaries.
  ```csharp
  // Standard Awaitable
  public async Awaitable FadeOutAsync(CancellationToken cancellationToken)
  {
      while (_canvasGroup.alpha > 0f)
      {
          cancellationToken.ThrowIfCancellationRequested();
          _canvasGroup.alpha -= Time.deltaTime;
          await Awaitable.NextFrameAsync(cancellationToken);
      }
  }

  // Hybrid UniTask Bridge (Internal helper returning standard Awaitable)
  public async Awaitable<(PlayerData, LevelData)> LoadMatchDataAsync(CancellationToken cancellationToken)
  {
      // UniTask.WhenAll provides strongly-typed tuple completion not built into Awaitable
      var (player, level) = await UniTask.WhenAll(
          FetchPlayerDataAsync(cancellationToken),
          FetchLevelDataAsync(cancellationToken)
      );
      return (player, level);
  }
  ```

---

## 4. Rule Discovery & Progressive Loading

Specialized rules are loaded based on file patterns and context:
- **`unity-csharp-conventions.md`**: Automatically applied when inspecting or writing C# (`*.cs`) files.
- **`unity-performance-gc.md`**: Automatically applied when writing C# files to enforce memory and execution budgets.
- **`unity-ui-toolkit.md`**: Automatically applied when authoring UI Toolkit files (`*.uxml`, `*.uss`, UI presenters/views).
- **`unity-editor-tooling.md`**: Automatically applied when working on Editor tools and scripts (`*Editor*.cs`).
- **`unity-input-system.md`**: Automatically applied when working with inputs (`*Input*.cs`, `*.inputactions`).
- **`unity-architecture-patterns.md`**: Triggered on demand for architecture design and refactoring.
- **`unity-ui-physics-audio.md`**: Triggered on demand for UGUI, physics simulation, or audio integration.
- **`unity-asset-memory-management.md`**: Triggered on demand for Addressables, memory management, and asset lifecycle.
- **`unity-data-persistence.md`**: Triggered on demand for save game systems, serialization, and state persistence.
- **`unity-job-system-burst.md`**: Triggered on demand for high-performance computing, Job System, and Burst compilation.
- **`unity-testing-diagnostics.md`**: Triggered on demand for unit testing, UTF, logging, and runtime assertions.
- **`unity-audit-release.md`**: Manually triggered before release via `@unity-audit-release`.
