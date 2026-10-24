---
trigger: glob
globs: "*Input*.cs, *.inputactions"
description: "Unity 6 Modern Input System standards, Input Action Asset C# wrapper, decoupled InputReader pattern, Action Map lifecycle, and device management."
---

# Unity 6 Input System Architecture & Standards

This rule governs the design, lifecycle management, and decoupling of player input using Unity 6's modern Input System package (`UnityEngine.InputSystem`).

---

## 1. Input Architecture Principles

- **No Legacy Input Manager**: Never use `Input.GetKeyDown`, `Input.GetAxis`, or `Input.mousePosition`.
- **Decoupled Input Reader**: Gameplay logic components (`PlayerMotor`, `CombatSystem`) must never reference `InputAction` or raw hardware directly. All input reads must pass through an abstract or event-driven **`InputReader`**.
- **Generated C# Wrapper**: Always enable **"Generate C# Class"** in the `.inputactions` asset inspector. Never query actions via string lookups (`actions.FindAction("Move")`).

---

## 2. Decoupled InputReader Pattern

Use a centralized ScriptableObject or persistent service to encapsulate generated input actions and expose clean C# events:

```csharp
using System;
using UnityEngine;
using UnityEngine.InputSystem;

[CreateAssetMenu(fileName = "InputReader", menuName = "Game/Input/Input Reader")]
public class InputReader : ScriptableObject, GameInputActions.IPlayerActions, GameInputActions.IUIActions
{
    private GameInputActions _actions;

    // Gameplay Events
    public event Action<Vector2> MoveEvent;
    public event Action JumpEvent;
    public event Action JumpCanceledEvent;
    public event Action AttackEvent;

    // UI Events
    public event Action CancelEvent;

    private void OnEnable()
    {
        if (_actions == null)
        {
            _actions = new GameInputActions();
            _actions.Player.SetCallbacks(this);
            _actions.UI.SetCallbacks(this);
        }
        EnableGameplayInput();
    }

    private void OnDisable()
    {
        DisableAllInput();
    }

    public void EnableGameplayInput()
    {
        _actions.UI.Disable();
        _actions.Player.Enable();
    }

    public void EnableUIInput()
    {
        _actions.Player.Disable();
        _actions.UI.Enable();
    }

    public void DisableAllInput()
    {
        _actions.Player.Disable();
        _actions.UI.Disable();
    }

    // --- IPlayerActions Callbacks ---
    public void OnMove(InputAction.CallbackContext context)
    {
        MoveEvent?.Invoke(context.ReadValue<Vector2>());
    }

    public void OnJump(InputAction.CallbackContext context)
    {
        if (context.performed) JumpEvent?.Invoke();
        else if (context.canceled) JumpCanceledEvent?.Invoke();
    }

    public void OnAttack(InputAction.CallbackContext context)
    {
        if (context.performed) AttackEvent?.Invoke();
    }

    // --- IUIActions Callbacks ---
    public void OnCancel(InputAction.CallbackContext context)
    {
        if (context.performed) CancelEvent?.Invoke();
    }
}
```

---

## 3. Action Map Lifecycle & UI Modal Discipline

Never allow gameplay inputs (e.g. shooting, jumping) to execute while UI menus, dialogues, or inventory screens are open.

### State Transition Rules:
1. **Entering UI**: Call `inputReader.EnableUIInput()` immediately when opening screens.
2. **Exiting UI**: Call `inputReader.EnableGameplayInput()` only when all modal overlays are closed.
3. **Cutscenes / Loading**: Disable both Action Maps via `inputReader.DisableAllInput()`.

---

## 4. Polling vs Callbacks in Gameplay Code

- **Continuous Input (Movement / Aiming)**:
  Consume cached `Vector2` values stored by the `InputReader` during physics updates:
  ```csharp
  private void FixedUpdate()
  {
      _motor.Move(_currentMoveDirection);
  }
  ```
- **Discrete Actions (Jump, Fire, Interact)**:
  Trigger logic exclusively from event subscriptions:
  ```csharp
  private void OnEnable()
  {
      _inputReader.JumpEvent += HandleJump;
  }

  private void OnDisable()
  {
      _inputReader.JumpEvent -= HandleJump;
  }
  ```

---

## 5. Device Disconnection & Input Buffering

### A. Device Disconnection Handling
Listen to `InputSystem.onDeviceChange` to automatically pause gameplay or show an alert when an active gamepad disconnects:
```csharp
private void OnEnable()
{
    InputSystem.onDeviceChange += OnDeviceChange;
}

private void OnDisable()
{
    InputSystem.onDeviceChange -= OnDeviceChange;
}

private void OnDeviceChange(InputDevice device, InputDeviceChange change)
{
    if (change == InputDeviceChange.Disconnected && device is Gamepad)
    {
        PauseGameDueToControllerDisconnect();
    }
}
```

### B. Input Buffering
For responsive gameplay (e.g. jump immediately upon landing if button was pressed 100ms early), store action timestamps rather than dropping inputs:
```csharp
private float _jumpBufferExpiresTime;
private const float JumpBufferDuration = 0.15f;

private void HandleJumpPressed()
{
    _jumpBufferExpiresTime = Time.time + JumpBufferDuration;
}

private void Update()
{
    if (Time.time < _jumpBufferExpiresTime && _isGrounded)
    {
        ExecuteJump();
        _jumpBufferExpiresTime = 0f; // Clear buffer
    }
}
```
