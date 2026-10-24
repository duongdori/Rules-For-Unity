# Google Antigravity Rules for Unity 6 C# Game Development

A production-ready, highly optimized ruleset for AI agents operating on **Unity 6+ (6000.x.x)** game projects powered by the **Google Antigravity** ecosystem.

This repository strictly complies with the official [Google Antigravity Rules Specification](file:///docs/Hu%CC%9B%CC%81ng%20da%CC%83n%20chi%20tie%CC%82%CC%81t%20Google%20Antigravity%20Rules.pdf), utilizing all four trigger modes to stay well within token budgets and the 24 KB per-file constraint while maximizing agent reasoning accuracy.

---

## 1. Ruleset Architecture & Trigger Modes

To conserve context tokens and adhere to Antigravity's 20,000-token rules budget, this ruleset uses **Progressive Disclosure** across four trigger tiers:

| Rule File | Trigger Mode | Activation Condition | Focus Area |
| :--- | :--- | :--- | :--- |
| **`AGENTS.md`** | `always_on` *(inherent)* | Root of workspace; injected unconditionally into all chat turns | Project core invariants, directory layout, `.asmdef` boundaries |
| **`.agents/rules/unity-csharp-conventions.md`** | `glob` | `*.cs` | Naming standards, `[SerializeField] private`, Unity Object lifetime ("fake null") |
| **`.agents/rules/unity-performance-gc.md`** | `glob` | `*.cs` | Zero-allocation hot loops, Unity 6 `Awaitable`, `UnityEngine.Pool`, NonAlloc physics |
| **`.agents/rules/unity-ui-toolkit.md`** | `glob` | `*.uxml, *.uss, *UI*.cs, *Presenter*.cs, *View*.cs` | UI Toolkit standards, silent traps (fonts, kebab-case, flex-shrink), anti click-through, theming |
| **`.agents/rules/unity-editor-tooling.md`** | `glob` | `*Editor*.cs, Assets/Editor/**/*.cs, *.Editor.cs` | Custom Inspectors, `SerializedObject`, Undo recording, UI Toolkit for Editor |
| **`.agents/rules/unity-architecture-patterns.md`** | `model_decision` | Injected progressively when architectural design/refactoring is detected | ScriptableObject architecture, Event Bus, State Machines, Dependency Injection |
| **`.agents/rules/unity-ui-physics-audio.md`** | `model_decision` | Injected progressively when UGUI, physics, or audio tasks are detected | Canvas splitting, TextMeshPro, `linearVelocity`, AudioSource pooling |
| **`.agents/rules/unity-audit-release.md`** | `manual` | Triggered only when the user types `@unity-audit-release` in chat | Pre-flight audit: missing scripts, profiler GC check, IL2CPP, ASTC compression |

---

## 2. Key Highlights for Unity 6

1. **Unity 6 Native Concurrency**: Emphasizes `UnityEngine.Awaitable` over legacy Coroutines for zero-allocation async programming with `CancellationToken` support.
2. **Modern Input & Physics APIs**: Adopts modern Unity 6 physics conventions (such as `Rigidbody.linearVelocity`) and modern Input System event paradigms.
3. **Memory Safety & Profiler Compliance**: Strictly prohibits heap allocations (`new`, closures, LINQ, string concatenation) in hot loops (`Update`, `FixedUpdate`, `LateUpdate`).
4. **Unity Object Lifetime**: Prevents common traps with null-coalescing (`??`) and null-conditional (`?.`) operators on `UnityEngine.Object` subclasses.

---

## 3. Installation & Usage Guide

### Option A: Use Directly in a Unity Project (Workspace Scope)
Copy the `AGENTS.md` file and `.agents/` directory directly into the root folder of your Unity project (where the `Assets/` and `ProjectSettings/` folders reside):

```powershell
# From your Unity project root:
Copy-Item -Recurse -Force "d:\AI_Agents\RulesForUnity\.agents" .
Copy-Item -Force "d:\AI_Agents\RulesForUnity\AGENTS.md" .
```

### Option B: Install as Global Rules (Machine-Wide Scope)
To have these rules automatically active across **all** projects on your development machine:

```powershell
# Create global rules directory if it does not exist:
New-Item -ItemType Directory -Force "$HOME\.gemini\config\rules"

# Copy modular rules to global configuration:
Copy-Item -Force "d:\AI_Agents\RulesForUnity\.agents\rules\*.md" "$HOME\.gemini\config\rules\"
```

### Option C: Share via Project Inheritance (`rules.json`)
If you keep this repository as a shared submodule or sibling repository, inherit from it in your target project's `.agents/rules.json`:

```json
{
  "inherits": [
    "../RulesForUnity/.agents/rules.json"
  ]
}
```

---

## 4. Manual Trigger Reference

Whenever you are preparing a build or performing a quality milestone review, mention the audit rule directly in your chat:

```text
@unity-audit-release please audit the current project settings and scene references
```

Antigravity will load the full release checklist into the conversation to inspect missing scripts, memory budgets, IL2CPP stripping, and texture compression settings.
