---
trigger: glob
globs: "*.cs"
description: "C# coding standards, naming conventions, MonoBehaviour lifecycle, and object reference safety for Unity 6."
---

# Unity 6 C# Coding Conventions & Standards

This rule governs coding standards, naming conventions, class structure, and lifecycle discipline for all C# source files within Unity 6 projects.

---

## 1. Naming Conventions

### General Naming Rules
- Names must be **descriptive and unambiguous** — avoid abbreviations unless universally understood (`UI`, `HUD`, `TMP`).
- **Spell out local variable and field names in full.** Do not shorten a type or role to an acronym or clipped form — this applies even to short-lived locals:
  - ❌ `vlg`, `btn`, `img`, `rt`, `go`, `tmp`, `mat`, `idx`, `cfg`, `req`, `tex`
  - ✅ `layoutGroup`, `button`, `image`, `rectTransform`, `gameObject`, `text`, `material`, `index`, `config`, `request`, `texture`
- **No Hungarian notation** except the private field prefix (`_`) below.
- **No single-letter names** except loop counters (`i`, `j`) and LINQ lambdas (`x`).

### Casing & Identifier Patterns
Maintain consistent naming across all scripts:

| Identifier | Convention | Example | Notes |
| :--- | :--- | :--- | :--- |
| **Classes / Structs / Records** | `PascalCase` | `PlayerController`, `InventoryItem` | Clear, noun-based naming |
| **Interfaces** | `IPascalCase` | `IDamageable`, `IInteractable` | Prefix with capital `I` |
| **Enums (Type)** | `PascalCase` (singular) | `WeaponType`, `GameState` | Plural for `[Flags]` (`DamageTypes`) |
| **Enum Members** | `PascalCase` | `None`, `Melee`, `Ranged` | No prefixes; always define `None = 0` |
| **Methods** | `PascalCase` | `TakeDamage()`, `CalculateTrajectory()` | Verb or verb-phrase |
| **Properties** | `PascalCase` | `Health`, `IsGrounded` | Encapsulate backing fields |
| **Public / Protected Fields** | `PascalCase` | `MaxSpeed` | Avoid public fields; use properties |
| **Private / Internal Fields** | `_camelCase` | `_moveSpeed`, `_rigidbody` | Prefix with underscore `_` |
| **Constants / Readonly Static** | `PascalCase` or `UPPER_SNAKE` | `MaxEnemies`, `DEFAULT_TIMEOUT` | Consistent within the project |
| **Events / Delegates** | `PascalCase` | `OnHealthChanged`, `OnDied` | Prefix event handlers/publishers clearly |
| **Local Variables / Parameters**| `camelCase` | `delta`, `targetPosition` | Meaningful, readable names |

### Enum Rules & Standards

#### 1. Naming & Structure
- **Singular for standard enums**: Use singular noun phrases (e.g., `WeaponType`, `CharacterState`, `AudioChannel`).
- **Plural for bit flags**: Use plural noun phrases when decorated with `[Flags]` (e.g., `StatusEffects`, `CollisionLayers`, `ItemCategories`).
- **No prefix on members**: Never prepend the enum type name, `E_`, or Hungarian prefixes to members:
  - ❌ `WeaponType_Sword`, `E_Sword`, `eSword`
  - ✅ `Sword`, `Bow`, `Staff`

#### 2. Always Define an Explicit Zero Value (`None = 0`)
In C#, default initialization zeroes memory (`default(TEnum) == 0`). Always explicitly assign `None = 0` (or `Default = 0` / `Unknown = 0`) as the first member to prevent uninitialized fields from accidentally adopting an unintended active state.

```csharp
// CORRECT
public enum GameState
{
    None = 0,
    MainMenu = 1,
    Loading = 2,
    Gameplay = 3,
    Paused = 4,
    GameOver = 5
}

// FORBIDDEN: Default(GameState) would silently become MainMenu!
public enum GameState
{
    MainMenu,
    Loading,
    Gameplay
}
```

#### 3. Explicit Integer Assignment
Explicitly define integer values (`None = 0`, `Primary = 1`, `Secondary = 2`). This prevents serialized Unity Inspector data corruption if members are reordered or intermediate values are inserted in the future.

#### 4. Bit Flags Enums (`[Flags]`)
- Always decorate bitmask enums with the `[Flags]` attribute.
- Use bit-shift notation (`1 << n`) for explicit power-of-two assignments.
- Consider specifying smaller underlying types (`byte`, `short`) to minimize memory footprint when stored in large arrays or networked structures:

```csharp
[Flags]
public enum StatusEffects : byte
{
    None     = 0,
    Burned   = 1 << 0, // 1
    Frozen   = 1 << 1, // 2
    Poisoned = 1 << 2, // 4
    Stunned  = 1 << 3, // 8
    All      = Burned | Frozen | Poisoned | Stunned
}
```

#### 5. Avoid Allocations in Hot Paths
- Never call `Enum.ToString()` or `Enum.Parse()` in frame loops (`Update`, `FixedUpdate`). Both invoke reflection and create heap string allocations.
- When using enums as keys in `Dictionary<TKey, TValue>`, Unity 6 / modern .NET provides zero-allocation lookups via default `EqualityComparer<TKey>.Default`. In legacy code or non-generic collections, avoid boxing enums to `object`.

---

## 2. Field Encapsulation & Serialization

### Rule 1: Always Favor `[SerializeField] private`
Never expose public fields solely for Unity Inspector access. Use serialized private fields with public getter properties.

```csharp
// CORRECT
[Header("Movement Settings")]
[SerializeField] private float _moveSpeed = 8.5f;
[SerializeField] private float _jumpForce = 12f;

public float MoveSpeed => _moveSpeed;
public float JumpForce => _jumpForce;

// FORBIDDEN
public float moveSpeed = 8.5f;
public float jumpForce = 12f;
```

### Rule 2: Inspector Organization Attributes
Group and document fields using Inspector attributes for designer clarity:
- `[Header("Title")]`: Categorizes serialized fields.
- `[Tooltip("Description")]`: Explains the variable purpose in the Inspector tooltip.
- `[Range(min, max)]`: Constrains numeric variables with a slider.
- `[Min(0)]`: Prevents invalid negative values.

```csharp
[Header("Combat Attributes")]
[Tooltip("Maximum amount of damage this entity can withstand.")]
[SerializeField, Min(1)] private int _maxHealth = 100;

[Tooltip("Rate of fire in rounds per minute.")]
[SerializeField, Range(60f, 1200f)] private float _fireRateRpm = 450f;
```

---

## 3. MonoBehaviour Script Structure & Member Ordering

Organize members consistently in every MonoBehaviour class:

1. **Constants & Static Readonly Fields**
2. **Serialized Fields** (grouped with `[Header]`)
3. **Private Non-Serialized Fields** (runtime state)
4. **Properties & Events**
5. **Unity Lifecycle Methods** (`Awake`, `OnEnable`, `Start`, `FixedUpdate`, `Update`, `LateUpdate`, `OnDisable`, `OnDestroy`)
6. **Public API Methods**
7. **Private Helper & Subroutine Methods**

---

## 4. MonoBehaviour Lifecycle Best Practices

### A. Lifecycle Method Division
- **`Awake()`**: Dedicated strictly to self-initialization. Cache local components (`GetComponent<T>()`) and initialize local collections. Do not query external game objects here.
- **`OnEnable()`**: Register event listeners, activate reactive subscriptions, reset per-activation state.
- **`Start()`**: Establish cross-object references, query scene-level services, or trigger initial gameplay actions.
- **`Update()`**: Frame-dependent gameplay logic (input polling, camera updates, timer decrements). Use `Time.deltaTime`.
- **`FixedUpdate()`**: Physics simulation only (Rigidbody movement, force calculations, raycasts). Use `Time.fixedDeltaTime`.
- **`LateUpdate()`**: Dependent updates, such as camera positioning following a moved target or procedural bone animation.
- **`OnDisable()`**: Unregister event listeners, stop running timers, disable reactive bindings.
- **`OnDestroy()`**: Clean up unmanaged resources, native buffers, or external handles.

### B. Eliminate Empty Lifecycle Methods
Never leave empty Unity message methods (`Update() {}`, `Start() {}`). Unity's native C++ engine dispatches calls to all declared message methods regardless of their body, causing unnecessary overhead.

```csharp
// FORBIDDEN: Incurs overhead every single frame
private void Update()
{
}
```

### C. Safe Component Caching
Never call `GetComponent<T>()` in `Update()` or hot paths. Cache references during `Awake()` or use `TryGetComponent<T>()`:

```csharp
// CORRECT
private Rigidbody _rigidbody;

private void Awake()
{
    _rigidbody = GetComponent<Rigidbody>();
}

private void OnTriggerEnter(Collider other)
{
    // TryGetComponent avoids allocations and checks safely
    if (other.TryGetComponent<IDamageable>(out var damageable))
    {
        damageable.TakeDamage(_damageAmount);
    }
}
```

---

## 5. Unity Object Lifetime & The "Fake Null" Trap

`UnityEngine.Object` implements custom C++ lifetime management and overloads equality operators (`==` and `!=`). When an object is destroyed in Unity, its underlying C++ entity is released, while the C# wrapper still exists in memory until garbage collected.

### Critical Rules:
1. **Never use `?.` (Null-Conditional) on `UnityEngine.Object`**:
   The `?.` operator bypasses Unity's overloaded `==` operator and only checks the managed C# reference. If the object was destroyed by `Destroy(obj)`, `obj?.Method()` will invoke on a destroyed native object, throwing a `MissingReferenceException`.

2. **Never use `??` (Null-Coalescing) on `UnityEngine.Object`**:
   The `??` operator evaluates the managed reference directly, failing to recognize destroyed Unity objects as null.

```csharp
// CORRECT: Evaluates Unity's custom equality check
if (_targetObject != null)
{
    _targetObject.PerformAction();
}

// FORBIDDEN: Bypasses native lifetime validation
_targetObject?.PerformAction();
var activeTarget = _targetObject ?? _defaultTarget;
```

---

## 6. Events & Delegates Discipline

- **Prefer C# `Action` / `Func` / `event` for code-to-code communication**: High performance, zero allocation, type-safe.
- **Reserve `UnityEvent` for designer-facing Inspector bindings**: Only use `UnityEvent` when level designers or technical artists must hook up objects in the Inspector without writing code.
- **Always unsubscribe from events**: Unsubscribe in `OnDisable()` or `OnDestroy()` to avoid memory leaks.

```csharp
[SerializeField] private HealthComponent _healthComponent;

private void OnEnable()
{
    if (_healthComponent != null)
    {
        _healthComponent.OnHealthChanged += HandleHealthChanged;
    }
}

private void OnDisable()
{
    if (_healthComponent != null)
    {
        _healthComponent.OnHealthChanged -= HandleHealthChanged;
    }
}

private void HandleHealthChanged(int currentHealth, int maxHealth)
{
    // Handle state update
}
```

---

## 7. Modern C# 9+ Features in Unity 6

Unity 6 supports modern C# language features. Utilize them where readability and safety are improved:
- **Target-Typed `new`**: When the type is explicitly declared on the left side:
  ```csharp
  private readonly List<EnemySpawnPoint> _spawnPoints = new();
  ```
- **Pattern Matching & Switch Expressions**:
  ```csharp
  public float GetSpeedMultiplier(MovementState state) => state switch
  {
      MovementState.Walking => 1.0f,
      MovementState.Sprinting => 1.8f,
      MovementState.Crouching => 0.5f,
      _ => 1.0f
  };
  ```
- **Null Safety**: Use `#nullable enable` on new scripts to catch null reference issues at compile time.

---

## 8. Script Execution Order Discipline

Never use `[DefaultExecutionOrder]` as a band-aid to patch initialization race conditions between arbitrary game scripts.
- **The Problem**: Shifting execution orders manually creates invisible coupling and fragile order-of-operation bugs that are difficult to reproduce or debug.
- **The Standard Solution**:
  1. Rely on the natural lifecycle separation: cache local components in `Awake()`, resolve cross-object dependencies in `Start()`.
  2. For complex startup sequences, use an explicit **Bootstrapper** or initialization method (`Construct()` / `Initialize()`) driven by an architectural entry point rather than racing `Awake()` calls.
- **Valid Exception**: Only assign `[DefaultExecutionOrder]` to fundamental engine bootstrappers (e.g., `-1000` for an application service container or input pre-poll manager).

---

## 9. Asynchronous Method Conventions & Signatures

Maintain consistent conventions for asynchronous operations across all C# scripts:

### A. Naming & Method Suffix
- Every method returning `Awaitable`, `Awaitable<T>`, `UniTask`, or `UniTask<T>` must end with the `Async` suffix:
  - ✅ `FadeScreenAsync()`, `FetchLeaderboardAsync()`
  - ❌ `FadeScreen()`, `FetchLeaderboard()`

### B. Parameter Ordering & Cancellation Tokens
- The last parameter of an async method should be a `CancellationToken` (prefer assigning `= default` for optional callers when appropriate, or explicitly requiring it for critical lifecycle routines):
  ```csharp
  public async Awaitable PlayIntroSequenceAsync(float delay, CancellationToken cancellationToken = default)
  ```

### C. Return Types & Prohibitions
- **Public API Return Types**: Always declare public API methods as returning `Awaitable` or `Awaitable<T>`.
- **Private / Internal Utility Types**: `UniTask` / `UniTask<T>` is permitted for private/internal methods when utilizing multi-task tuple combinators or advanced timing.
- **Prohibit `async void`**: Never use `async void` for standard game logic or internal routines. An unhandled exception in an `async void` method can crash the process or escape Unity's error handling.
  - *Exception*: Top-level UI event callbacks (e.g., UI Toolkit button click or UGUI `onClick.AddListener`) may use `async void`, provided the method body is enclosed in a `try/catch` block:
  ```csharp
  // PERMITTED ONLY FOR ROOT UI EVENT HANDLERS
  private async void HandlePlayButtonClicked()
  {
      try
      {
          await StartGameAsync(destroyCancellationToken);
      }
      catch (OperationCanceledException)
      {
          // Normal cancellation on destroy/disable
      }
      catch (Exception ex)
      {
          Debug.LogException(ex);
      }
  }
  ```
