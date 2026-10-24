---
trigger: model_decision
description: "Unit testing with UTF, zero-cost conditional logging, and engine-safe assertions for Unity 6."
---

# Unity 6 Testing, Logging & Diagnostics Standards

This rule governs test automation using the Unity Test Framework (UTF), production logging discipline, and runtime assertions.

---

## 1. Test Architecture: EditMode vs PlayMode

Organize tests strictly within dedicated test assemblies (`Game.Tests.asmdef`) referencing `UnityEngine.TestRunner`:

### A. EditMode Tests (Preferred for Pure Logic)
- **Fast Execution**: Runs in milliseconds inside the Editor without launching game loop or rendering.
- **Scope**: Test damage formulas, inventory math, state transitions, save/load DTO serialization, and pure algorithm logic.
- **Requirement**: Structure gameplay systems into POCO C# classes separated from `MonoBehaviour` so they can be unit-tested without scene overhead.

```csharp
using NUnit.Framework;

[TestFixture]
public class CombatCalculationTests
{
    [Test]
    public void CalculateDamage_ArmorMitigatesDamageCorrectly()
    {
        // Pure C# test - executes instantaneously
        int rawDamage = 50;
        int armor = 20;

        int finalDamage = CombatUtility.CalculateMitigatedDamage(rawDamage, armor);

        Assert.AreEqual(30, finalDamage);
    }
}
```

### B. PlayMode Tests (Integration & Engine Loops)
- **Scope**: Required when testing `UnityEngine.Awaitable`, physics collisions (`OnTriggerEnter`), Prefab instantiations, or Canvas UI interactions.
- Use `[UnityTest]` attribute returning `IEnumerator` or `async Awaitable`:

```csharp
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;

public class ProjectileTests
{
    [UnityTest]
    public IEnumerator Projectile_DeactivatesAfterLifetime()
    {
        var projectileGo = new GameObject();
        var projectile = projectileGo.AddComponent<Projectile>();
        projectile.Initialize(lifetimeSeconds: 0.1f);

        yield return new WaitForSeconds(0.15f);

        Assert.IsFalse(projectileGo.activeSelf);
        Object.Destroy(projectileGo);
    }
}
```

---

## 2. Zero-Cost Conditional Logging

Raw calls to `Debug.Log($"Enemy {name} damaged for {dmg}")` incur string formatting and heap allocations even when console logging is disabled.

### The `[Conditional]` Logger Standard:
Wrap all debug logging inside a centralized utility using `[Conditional("DEVELOPMENT_BUILD")]` or `[Conditional("UNITY_EDITOR")]`. The C# compiler strips the invocation and argument evaluations entirely from production release binaries:

```csharp
using System.Diagnostics;
using Debug = UnityEngine.Debug;

public static class GameLogger
{
    [Conditional("UNITY_EDITOR")]
    [Conditional("DEVELOPMENT_BUILD")]
    public static void Log(string message)
    {
        Debug.Log($"[Game] {message}");
    }

    [Conditional("UNITY_EDITOR")]
    [Conditional("DEVELOPMENT_BUILD")]
    public static void LogWarning(string message)
    {
        Debug.LogWarning($"[Game] {message}");
    }

    // Errors remain visible in production release builds for crash reporting
    public static void LogError(string message)
    {
        Debug.LogError($"[Game ERROR] {message}");
    }
}
```

---

## 3. Engine-Safe Assertions

Use `UnityEngine.Assertions.Assert` rather than throwing exceptions or using `System.Diagnostics.Debug.Assert`.
- `UnityEngine.Assertions.Assert` is actively integrated with Unity's engine loop and profiler.
- Assertions automatically compile away in release builds when `UNITY_ASSERTIONS` is not defined.

```csharp
using UnityEngine;
using UnityEngine.Assertions;

public class HealthComponent : MonoBehaviour
{
    public void TakeDamage(int amount)
    {
        // Enforce invariants cleanly during development
        Assert.IsTrue(amount >= 0, "Damage amount cannot be negative!");

        _currentHealth -= amount;
    }
}
```
