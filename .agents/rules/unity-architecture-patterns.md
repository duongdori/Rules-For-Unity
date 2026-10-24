---
trigger: model_decision
description: "Game architecture patterns for Unity 6: ScriptableObject architecture, decoupled event-driven systems, state machines, and dependency injection."
---

# Unity 6 Game Architecture & Design Patterns

This rule outlines architectural patterns, decoupling strategies, and structuring paradigms for scalable, testable, and maintainable Unity 6 game systems.

---

## 1. Architectural Philosophy

- **Modular Composition over Monolithic Inheritance**: Prefer assembling focused components rather than creating deep inheritance hierarchies (`Player` : `Character` : `Entity` : `Destructible` : `MonoBehaviour`).
- **Decoupled Communication**: Components should communicate horizontally via events or interfaces, not by tightly coupling to specific concrete implementations.
- **Data vs Logic Separation**: Game data and configurations belong in ScriptableObjects or pure C# records; logic belongs in systems and components.

---

## 2. ScriptableObject-Driven Architecture

ScriptableObjects (SOs) act as first-class architectural building blocks in Unity. Leverage them for three primary purposes:

### A. Configuration & Balance Data Containers
Store stats, weapon data, loot tables, and enemy configurations in asset files.
```csharp
[CreateAssetMenu(fileName = "NewWeaponConfig", menuName = "Game/Combat/Weapon Config")]
public class WeaponConfig : ScriptableObject
{
    [SerializeField] private string _weaponName;
    [SerializeField, Min(1)] private int _baseDamage = 25;
    [SerializeField, Range(0.1f, 5f)] private float _attackRateSeconds = 0.5f;
    [SerializeField] private GameObject _projectilePrefab;

    public string WeaponName => _weaponName;
    public int BaseDamage => _baseDamage;
    public float AttackRateSeconds => _attackRateSeconds;
    public GameObject ProjectilePrefab => _projectilePrefab;
}
```

### B. ScriptableObject Game Events (Zero-Coupling Messaging)
Use ScriptableObjects as event channels. Publishers raise the event without knowing who is listening; listeners subscribe via Inspector or code without referencing the publisher. Protect against subscriber exceptions interrupting remaining callbacks:

```csharp
[CreateAssetMenu(fileName = "NewGameEvent", menuName = "Game/Events/Game Event")]
public class GameEvent : ScriptableObject
{
    private readonly List<Action> _listeners = new();

    public void Raise()
    {
        for (int i = _listeners.Count - 1; i >= 0; i--)
        {
            try
            {
                _listeners[i]?.Invoke();
            }
            catch (Exception ex)
            {
                Debug.LogException(ex);
            }
        }
    }

    public void RegisterListener(Action listener)
    {
        if (!_listeners.Contains(listener)) _listeners.Add(listener);
    }

    public void UnregisterListener(Action listener)
    {
        _listeners.Remove(listener);
    }
}
```

### C. Shared Variables & The Persistence Trap
Centralize shared gameplay values (such as `PlayerHealthVariable` or `ScoreVariable`) into ScriptableObjects to allow UI, gameplay, and audio systems to react without a monolithic `GameManager`.

> [!CAUTION]
> **The ScriptableObject Persistence Trap**:
> While ScriptableObject field modifications persist across Play Mode sessions **inside the Unity Editor**, they **never persist to disk in standalone production builds** (all changes revert when the application closes).
> - **ScriptableObjects**: Strictly for **Design-Time Authoring & Read-Only Configuration** (e.g. baseline stats, weapon definitions, drop tables).
> - **Save Game State**: Must always use pure C# DTO / POCO data classes serialized to disk via JSON, Binary, or dedicated save systems. Never attempt to use ScriptableObjects as your persistent save-game files.

---

## 3. Decoupled Communication & The Anti-Singleton Principle

### The God Singleton Anti-Pattern
Avoid chains of direct singleton lookups:
```csharp
// ANTI-PATTERN: High coupling, brittle, hard to test or refactor
GameManager.Instance.Player.Inventory.Gold += 50;
UIManager.Instance.HudPanel.UpdateGoldText();
```

### Preferred Decoupled Alternatives:
1. **Direct C# Events / Action Callbacks**:
   ```csharp
   public class Inventory : MonoBehaviour
   {
       public event Action<int> OnGoldChanged;

       public void AddGold(int amount)
       {
           _gold += amount;
           OnGoldChanged?.Invoke(_gold);
       }
   }
   ```
2. **Interface Segregation**:
   Depend on small, dedicated interfaces (`IDamageable`, `IInteractable`, `ISavable`) rather than concrete classes.

---

## 4. Finite State Machine (FSM) Pattern

Implement hierarchical or interface-driven finite state machines for characters, AI, and game state flow instead of massive `switch-case` statements in `Update()`.

### Standard State Pattern Interface:
```csharp
public interface IState
{
    void Enter();
    void Update();
    void FixedUpdate();
    void Exit();
}

public class StateMachine
{
    public IState CurrentState { get; private set; }

    public void Initialize(IState startingState)
    {
        CurrentState = startingState;
        CurrentState.Enter();
    }

    public void TransitionTo(IState nextState)
    {
        if (CurrentState == nextState || nextState == null) return;

        CurrentState.Exit();
        CurrentState = nextState;
        CurrentState.Enter();
    }

    public void Update() => CurrentState?.Update();
    public void FixedUpdate() => CurrentState?.FixedUpdate();
}
```

---

## 5. Dependency Injection (DI) & Service Resolution

Avoid scattering `FindObjectOfType` or hardcoded static access. Use Dependency Injection (e.g. VContainer) or a lightweight Service Locator:

### Explicit Initialization Pattern (Constructor / Init Injection):
```csharp
public class EnemySpawner : MonoBehaviour
{
    private IObjectPool<Enemy> _enemyPool;
    private IAudioService _audioService;

    // Explicit dependency injection method
    public void Construct(IObjectPool<Enemy> enemyPool, IAudioService audioService)
    {
        _enemyPool = enemyPool;
        _audioService = audioService;
    }

    public void Spawn()
    {
        var enemy = _enemyPool.Get();
        _audioService.PlaySfx("EnemySpawned");
    }
}
```

---

## 6. Single Responsibility Principle (SRP) for MonoBehaviours

Deconstruct gameplay entities into focused, cooperative components:

| Component | Responsibility |
| :--- | :--- |
| `PlayerInputReader` | Reads from `UnityEngine.InputSystem`, emits pure input events |
| `PlayerMotor` | Applies forces and calculates velocities in `FixedUpdate` |
| `PlayerAnimator` | Listens to movement/action events and drives Animator parameters |
| `PlayerHealth` | Manages damage, invulnerability frames, and death events |
| `PlayerAudio` | Plays sound effects corresponding to entity actions |

No single MonoBehaviour should exceed **300 lines of code**. If it does, extract sub-systems.
