---
trigger: glob
globs: "*.cs"
description: "C# coding standards, naming conventions, MonoBehaviour lifecycle, and object reference safety for Unity 6."
---

# Unity 6 C# Coding Conventions & Standards

This rule governs coding standards, naming conventions, class structure, and lifecycle discipline for all C# source files within Unity 6 projects.

---

## 1. Naming Conventions

Maintain consistent naming across all scripts:

| Identifier | Convention | Example | Notes |
| :--- | :--- | :--- | :--- |
| **Classes / Structs / Records** | `PascalCase` | `PlayerController`, `InventoryItem` | Clear, noun-based naming |
| **Interfaces** | `IPascalCase` | `IDamageable`, `IInteractable` | Prefix with capital `I` |
| **Methods** | `PascalCase` | `TakeDamage()`, `CalculateTrajectory()` | Verb or verb-phrase |
| **Properties** | `PascalCase` | `Health`, `IsGrounded` | Encapsulate backing fields |
| **Public / Protected Fields** | `PascalCase` | `MaxSpeed` | Avoid public fields; use properties |
| **Private / Internal Fields** | `_camelCase` | `_moveSpeed`, `_rigidbody` | Prefix with underscore `_` |
| **Constants / Readonly Static** | `PascalCase` or `UPPER_SNAKE` | `MaxEnemies`, `DEFAULT_TIMEOUT` | Consistent within the project |
| **Events / Delegates** | `PascalCase` | `OnHealthChanged`, `OnDied` | Prefix event handlers/publishers clearly |
| **Local Variables / Parameters**| `camelCase` | `delta`, `targetPosition` | Meaningful, readable names |

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
