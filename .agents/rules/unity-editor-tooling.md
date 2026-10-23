---
trigger: glob
globs: "*Editor*.cs, Assets/Editor/**/*.cs, *.Editor.cs"
description: "Standards for Unity Editor scripting, Custom Inspectors, PropertyDrawers, EditorWindows, and undo management."
---

# Unity 6 Editor Scripting & Tooling Standards

This rule governs the creation of Custom Inspectors, PropertyDrawers, EditorWindows, and custom build pipelines in Unity 6.

---

## 1. Assembly Definition Isolation & Preprocessor Directives

### Non-Negotiable Rule: Zero Editor Code in Runtime Builds
Any reference to the `UnityEditor` namespace inside a runtime assembly will fail build compilation.

### Standards:
1. **Prefer Dedicated Editor Assemblies**: Place all editor scripts under an `Editor/` folder with a dedicated `*.Editor.asmdef` configured with **Platform: Editor** only.
2. **Preprocessor Guards**: If an editor-specific method or attribute must exist in a runtime script, wrap it strictly in `#if UNITY_EDITOR`:
   ```csharp
   #if UNITY_EDITOR
   private void OnDrawGizmosSelected()
   {
       Gizmos.color = Color.yellow;
       Gizmos.DrawWireSphere(transform.position, _detectionRadius);
   }
   #endif
   ```

---

## 2. SerializedObject & SerializedProperty Discipline

When writing Custom Inspectors (`Editor`), always interact through `SerializedObject` and `SerializedProperty`. This automatically grants undo/redo support, multi-object editing, and prefab override tracking.

### Standard Inspector Template:
```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;

[CustomEditor(typeof(PlayerController))]
[CanEditMultipleObjects]
public class PlayerControllerEditor : Editor
{
    private SerializedProperty _moveSpeedProp;
    private SerializedProperty _jumpForceProp;
    private SerializedProperty _weaponConfigProp;

    private void OnEnable()
    {
        // Cache serialized properties
        _moveSpeedProp = serializedObject.FindProperty("_moveSpeed");
        _jumpForceProp = serializedObject.FindProperty("_jumpForce");
        _weaponConfigProp = serializedObject.FindProperty("_weaponConfig");
    }

    public override void OnInspectorGUI()
    {
        // Must start with Update()
        serializedObject.Update();

        EditorGUILayout.PropertyField(_moveSpeedProp);
        EditorGUILayout.PropertyField(_jumpForceProp);

        EditorGUILayout.Space(8);
        EditorGUILayout.PropertyField(_weaponConfigProp);

        // Must end with ApplyModifiedProperties()
        serializedObject.ApplyModifiedProperties();
    }
}
#endif
```

---

## 3. Undo System & Dirty Tracking

If your editor code modifies raw object fields outside of `SerializedProperty`:

1. **Record Undo**: Always call `Undo.RecordObject()` before mutating fields so the user can press `Ctrl+Z`.
2. **Mark Dirty**: Call `EditorUtility.SetDirty(target)` to ensure Unity serializes changes to disk.
3. **Mark Scene Dirty**: If modifying an object in the active scene, call `EditorSceneManager.MarkSceneDirty()`.

```csharp
if (GUILayout.Button("Reset To Defaults"))
{
    var controller = (PlayerController)target;
    Undo.RecordObject(controller, "Reset Controller Defaults");
    controller.ResetParameters();
    EditorUtility.SetDirty(controller);
}
```

---

## 4. Modern UI Toolkit for Editor Windows in Unity 6

In Unity 6, UI Toolkit (`UnityEngine.UIElements`) is the standard for custom Editor Windows, offering high-performance retained-mode rendering and clean separation of UI (UXML), styling (USS), and C# logic.

### Editor Window UI Toolkit Pattern:
```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

public class GameToolsWindow : EditorWindow
{
    [MenuItem("Tools/Game/Tools Dashboard")]
    public static void ShowWindow()
    {
        var window = GetWindow<GameToolsWindow>();
        window.titleContent = new GUIContent("Game Tools");
        window.minSize = new Vector2(350, 400);
    }

    public void CreateGUI()
    {
        var root = rootVisualElement;

        var header = new Label("Game Tools & Debug Dashboard");
        header.style.fontSize = 18;
        header.style.unityFontStyleAndWeight = FontStyle.Bold;
        header.style.marginBottom = 10;
        root.Add(header);

        var actionButton = new Button(OnExecuteAction)
        {
            text = "Generate Game Balance Data"
        };
        root.Add(actionButton);
    }

    private void OnExecuteAction()
    {
        // Tool execution logic
    }
}
#endif
```

---

## 5. Menu Items & Tool Validation

- Always namespace menu items under project-specific categories (e.g., `Tools/ProjectName/...`).
- Use validation functions to disable menu items when conditions are not met:
```csharp
[MenuItem("Tools/Combat/Quick Hurt Selected", true)]
private static bool ValidateQuickHurt()
{
    // Only enabled if selected object has IDamageable
    return Selection.activeGameObject != null &&
           Selection.activeGameObject.TryGetComponent<IDamageable>(out _);
}

[MenuItem("Tools/Combat/Quick Hurt Selected")]
private static void QuickHurt()
{
    if (Selection.activeGameObject.TryGetComponent<IDamageable>(out var damageable))
    {
        damageable.TakeDamage(10);
    }
}
```
