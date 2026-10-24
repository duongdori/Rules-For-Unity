---
trigger: glob
globs: "*.uxml, *.uss, *UI*.cs, *Presenter*.cs, *View*.cs"
description: "Technical standards, silent traps, layout constraints, anti click-through patterns, and dynamic theming for Unity 6 UI Toolkit."
---

# Unity 6 UI Toolkit Standards & Production Invariants

This rule governs the architecture, layout discipline, data binding, and silent failure traps when building runtime game user interfaces with **UI Toolkit** (`UnityEngine.UIElements`) in Unity 6+.

---

## 1. Panel Component Architecture & Hot Reloading

### A. PanelRenderer vs UIDocument
In Unity 6, **`PanelRenderer`** is the modern component for rendering UXML through `PanelSettings`.
- **The Internal Root Trap**: `PanelRenderer.rootVisualElement` and `IPanelComponent.GetRootVisualElement()` are `internal`. User code cannot read them directly.
- **The Public Standard**: Always access the visual tree via `RegisterUIReloadCallback`:

```csharp
public class ScreenPresenter : MonoBehaviour
{
    [SerializeField] private PanelRenderer _panelRenderer;
    private VisualElement _screenRoot;

    // Use VersionedUIReloadCallback (unversioned overload is obsolete)
    private PanelRenderer.VersionedUIReloadCallback _reloadCallback;

    private void Awake()
    {
        _reloadCallback = OnUIReloaded;
    }

    private void OnEnable()
    {
        if (_panelRenderer != null)
        {
            _panelRenderer.RegisterUIReloadCallback(_reloadCallback);
        }
    }

    private void OnDisable()
    {
        if (_panelRenderer != null)
        {
            _panelRenderer.UnregisterUIReloadCallback(_reloadCallback);
        }
    }

    private void OnUIReloaded(PanelRenderer panel, VisualElement root, int version)
    {
        // root is the PanelRendererRootElement wrapper; authored root is child[0]
        _screenRoot = root.childCount > 0 ? root[0] : null;
        BindViews();
    }
}
```

> [!IMPORTANT]
> **Live Hot-Reload in Play Mode**: Keeping `RegisterUIReloadCallback` active throughout `OnEnable` / `OnDisable` enables **hot-reloading of UXML and USS while Play Mode is running**. Modifying stylesheets or layouts on disk immediately updates the live game view without restarting play mode.

### B. PanelSettings Scaling Invariants
For screen-space overlay interfaces, configure `PanelSettings` once and author strictly against the design reference resolution:
- **Render Mode**: `PanelRenderMode.ScreenSpaceOverlay`
- **Scale Mode**: `PanelScaleMode.ScaleWithScreenSize`
- **Reference Resolution**: Target design resolution (e.g., `1080 x 1920` for portrait)
- **Screen Match Mode**: `MatchWidthOrHeight` with `match = 1.0f` (matches height, preventing portrait clipping)
- **Clear Color**: `false` (lets camera clear the background; overlay renders on top)

---

## 2. Silent Traps & Failure Invariants (Zero Console Errors)

These pitfalls produce no compiler errors, warnings, or console exceptions, but silently break the interface.

### Trap 1: The Missing Font Trap (Invisible Text)
- TextCore requires an SDF **`FontAsset`** (not legacy `Font`).
- **Dual Declaration Requirement**: Assigning a font asset to `PanelSettings.textSettings` only affects runtime. **UI Builder does not read `PanelSettings`**.
- **Mandatory Rule**: You must also define the font asset in the root class of your USS:
  ```css
  .screen-root {
      -unity-font-definition: url("project://database/Assets/UI/GameFont.asset");
  }
  ```
  Without this USS line, opening screens in UI Builder will render blank text with zero errors.

### Trap 2: UXML Attributes Must Be Strictly Kebab-Case
- C# `[UxmlAttribute]` properties must be referenced in UXML in **lowercase kebab-case**.
- Unity silently ignores unknown or uppercase attributes without any warning, leaving values at their defaults:
  ```xml
  <!-- WRONG: Silently ignored; corner radius stays 0 -->
  <g:CustomBadge CornerCut="12" cornerRadius="12" />

  <!-- CORRECT: Lowercase kebab-case -->
  <g:CustomBadge corner-cut="12" corner-radius="12" />
  ```
- **Rule**: No attribute name in UXML may contain uppercase characters.

### Trap 3: `flex-shrink` Defaults to 1 (Layout Squeezing)
- In UI Toolkit's layout engine (Yoga), `flex-shrink` defaults to `1`.
- Any element with an authored `width` or `height` (buttons, icons, avatars) in a crowded row or column will be squeezed into slivers unless explicitly protected:
  ```css
  .icon-button {
      width: 48px;
      height: 48px;
      flex-shrink: 0; /* MANDATORY for fixed-size elements */
  }
  ```

### Trap 4: No Sibling Combinator in USS
- USS does **not** support the general sibling combinator (`.a ~ .b`) or adjacent sibling combinator (`.a + .b`).
- If an active state on one element must restyle a sibling, mirror the state class onto their common parent container from C#.

### Trap 5: Linear Color Space Low-Alpha Wash
- In Linear Color Space, subtle alpha washes (e.g. 10% white on dark backdrops) composite significantly brighter than in sRGB.
- For subtle tints, use alphas around **0.01 – 0.05** rather than 0.1 – 0.2.
- For precise brand colors, prefer opaque color tokens over alpha overlays.

### Trap 6: Author Resting State in UXML, Not Animation Start State
- **Never author `.screen--enter { opacity: 0; }` directly on root elements in UXML**.
- Because UI Builder executes no C# scripts, opening a screen with an authored entrance class displays a completely blank canvas.
- **Rule**: Author the resting, fully-visible layout in UXML. Let C# apply the hidden class at runtime and release it after arming:
  ```css
  .reveal {
      opacity: 1;
      transition-property: opacity;
      transition-duration: 400ms;
  }
  .screen--enter .reveal {
      opacity: 0;
      transition-duration: 0s; /* Snap instantly into hidden state */
  }
  ```

---

## 3. Runtime Data Binding with ScriptableObjects

Never imperatively assign text in presenter code (`label.text = data.name`). This creates two conflicting sources of truth. Use Unity 6 Runtime Data Binding.

### A. Reactive ScriptableObject Model
Data assets must implement `INotifyBindablePropertyChanged` and tag properties with `[CreateProperty]`. Call `OnValidate()` so Inspector edits propagate live:

```csharp
using System;
using System.ComponentModel;
using System.Runtime.CompilerServices;
using Unity.Properties;
using UnityEngine;

[CreateAssetMenu(fileName = "CharacterData", menuName = "Game/UI/Character Data")]
public class CharacterData : ScriptableObject, INotifyBindablePropertyChanged
{
    [SerializeField] private string _characterName = "HERO";
    [SerializeField] private int _baseHealth = 100;

    [CreateProperty]
    public string CharacterName
    {
        get => _characterName;
        set => SetField(ref _characterName, value);
    }

    [CreateProperty]
    public string HealthDisplay => $"{_baseHealth} HP";

    public event EventHandler<BindablePropertyChangedEventArgs> propertyChanged;

    private void SetField<T>(ref T field, T value, [CallerMemberName] string propertyName = "")
    {
        if (Equals(field, value)) return;
        field = value;
        propertyChanged?.Invoke(this, new BindablePropertyChangedEventArgs(propertyName));
    }

    // Live propagation of Inspector edits during authoring & play mode
    private void OnValidate()
    {
        propertyChanged?.Invoke(this, new BindablePropertyChangedEventArgs(nameof(CharacterName)));
        propertyChanged?.Invoke(this, new BindablePropertyChangedEventArgs(nameof(HealthDisplay)));
    }
}
```

### B. "Bind the Data, Push the Paint" Principle
- **Bind Data**: Bind string, numeric, and enum values through `dataSource` and UXML bindings.
- **Push Paint**: Custom visual elements requiring procedural `Painter2D` or `MeshGenerationContext` (radial progress rings, health bars, custom charts) receive state through C# properties that call `MarkDirtyRepaint()`.

---

## 4. Input & Anti Click-Through Discipline

### A. Independent Event System in Unity 6
- UI Toolkit in Unity 6 runs its own internal event pipeline for pointer hits and `:hover`.
- A uGUI `EventSystem` GameObject is **not required** unless the project mixes uGUI Canvas and UI Toolkit in the same scene.

### B. Click-Through Between Screens (Critical Guard)
When transitioning between screens, an incoming panel that sets `display: Flex` immediately becomes hit-testable while the player's pointer interaction is still in flight. A single click can cascade into multiple screens.

**Mandatory Screen Transition Pattern**:
```csharp
public static class UITransitionUtility
{
    public static void HideScreen(VisualElement screenRoot, int fadeDurationMs = 300)
    {
        // IMMEDIATELY disable hits before beginning visual fade
        screenRoot.pickingMode = PickingMode.Ignore;
        screenRoot.AddToClassList("screen--hidden");

        screenRoot.schedule.Execute(() =>
        {
            screenRoot.style.display = DisplayStyle.None;
        }).StartingIn(fadeDurationMs);
    }

    public static void ShowScreen(VisualElement screenRoot, int enterDelayMs = 20, int settleMs = 320)
    {
        // Keep deaf to inputs while fading in
        screenRoot.pickingMode = PickingMode.Ignore;
        screenRoot.style.display = DisplayStyle.Flex;

        screenRoot.schedule.Execute(() =>
        {
            screenRoot.RemoveFromClassList("screen--hidden");
        }).StartingIn(enterDelayMs);

        // Only restore hit-testing after animation fully settles
        screenRoot.schedule.Execute(() =>
        {
            screenRoot.pickingMode = PickingMode.Position;
        }).StartingIn(settleMs);
    }
}
```

---

## 5. Dynamic Theming Contract

### The Hard Limitation of UI Toolkit
There is **no C# runtime API to mutate USS custom variables (`--accent-color`) directly**.

### The Class-Swapping Solution
1. Define theme colors in a `ThemePalette` ScriptableObject.
2. Generate class tokens (`Tokens.uss`) defining variants:
   ```css
   .accent--crimson {
       --accent-primary: rgb(220, 20, 60);
       --accent-soft: rgba(220, 20, 60, 0.15);
   }
   .accent--azure {
       --accent-primary: rgb(0, 127, 255);
       --accent-soft: rgba(0, 127, 255, 0.15);
   }
   ```
3. Swap theme classes on the root `VisualElement`. All child elements, borders, and transitions re-skin automatically through CSS cascade.
4. Custom elements that need theme colors must read them via `CustomStyleProperty<Color>`:
   ```csharp
   private static readonly CustomStyleProperty<Color> AccentColorProp = new("--accent-primary");

   private void OnCustomStyleResolved(CustomStyleResolvedEvent evt)
   {
       if (evt.customStyle.TryGetValue(AccentColorProp, out var color))
       {
           _frameColor = color;
           MarkDirtyRepaint();
       }
   }
   ```

---

## 6. Vector Icons & Procedural Mesh Generation

- **Static Icons**: Use SVGs imported as `VectorImage` (ensure importer `SvgType` is `VectorImage`, not `VectorSprite`). Author SVGs in white/grey so `-unity-background-image-tint-color` can dynamically tint them.
- **Procedural Glows (Painter2D)**: Never stack concentric disc arcs (`Arc` + `Fill`), which creates stepping artifacts. Use a triangle fan (`MeshGenerationContext.Allocate`) with per-vertex alpha interpolation to achieve smooth GPU falloff.
