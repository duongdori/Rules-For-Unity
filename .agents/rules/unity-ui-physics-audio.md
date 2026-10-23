---
trigger: model_decision
description: "Subsystem best practices for Unity 6: UI Toolkit and UGUI optimization, 2D/3D physics simulation, and audio management."
---

# Unity 6 UI, Physics & Audio Subsystems

This rule specifies performance, architecture, and configuration standards for User Interfaces, Physics simulations, and Audio systems in Unity 6.

---

## 1. User Interface (UI) Architecture & Optimization

### A. UI Framework Selection
- **UI Toolkit (`UnityEngine.UIElements`)**: Preferred for new projects, scalable menus, inventory screens, and HUDs in Unity 6. Offers retained-mode rendering with minimal draw-call overhead.
- **UGUI (`UnityEngine.UI`)**: Supported for existing or 3D world-space interfaces. Requires strict Canvas partitioning.

### B. UGUI Performance Rules: Canvas Splitting
When an element inside a Canvas moves, changes size, or changes text, Unity must regenerate the entire Canvas vertex mesh.

1. **Partition Static vs Dynamic Elements**:
   - **Static Canvas**: Background panels, borders, static icons, labels. (Batch once, never regenerate).
   - **Dynamic Canvas**: Health bars, minimap markers, damage numbers, timers. (Regenerates frequently without invalidating static geometry).
2. **Disable `Raycast Target`**:
   - By default, `Image` and `TextMeshProUGUI` enable `Raycast Target`. Disable this on all decorative UI elements to eliminate raycast traversal overhead during input processing.
3. **TextMeshPro Allocation Discipline**:
   - Always use TextMeshPro (`TextMeshProUGUI`) instead of legacy `Text`.
   - Use non-allocating formatting overloads:
     ```csharp
     // ZERO ALLOCATION
     _ammoText.SetText("AMMO: {0} / {1}", currentAmmo, maxAmmo);
     ```

---

## 2. Physics Simulation (2D & 3D)

### A. Lifecycle & Movement Discipline
- **Execute physics code strictly in `FixedUpdate()`** using `Time.fixedDeltaTime`.
- **Never modify `transform.position` or `transform.rotation` directly on dynamic Rigidbodies**. Doing so teleports the body, resets velocity, and breaks continuous collision detection.

### B. Movement APIs in Unity 6
In Unity 6, `Rigidbody.velocity` is updated to `linearVelocity` and `angularVelocity`:
```csharp
// CORRECT in Unity 6
private void FixedUpdate()
{
    // Directional velocity control
    _rigidbody.linearVelocity = new Vector3(_moveInput.x * _speed, _rigidbody.linearVelocity.y, _moveInput.y * _speed);

    // Or kinematic continuous movement
    _kinematicRb.MovePosition(_kinematicRb.position + moveDelta);
}
```

### C. Jitter Prevention with Interpolation
Enable **Interpolate** on the player and prominent moving rigidbodies. This blends transform positions between physics ticks, delivering smooth visual motion at high refresh rates (120Hz/144Hz+) even if `fixedDeltaTime` runs at 50Hz (0.02s).

### D. Layer Collision Matrix Optimization
- Configure the **Physics Layer Collision Matrix** (Project Settings -> Physics).
- Disable collisions between layers that never interact (e.g. `Debris` vs `Debris`, `PlayerBullet` vs `Player`, `EnemyTrigger` vs `Environment`).

### E. Collider Primitives over Mesh Colliders
- Use primitive colliders (Box, Sphere, Capsule) or compound primitive colliders.
- Avoid dynamic (non-convex) `MeshCollider` on moving objects.

---

## 3. Audio Subsystem Best Practices

### A. AudioClip Import Settings
Configure AudioClips based on their duration and usage pattern:

| Category | Typical Duration | Load Type | Compression Format |
| :--- | :--- | :--- | :--- |
| **Short SFX (Footsteps, UI, Gunshots)** | < 2 seconds | **Decompress On Load** | PCM or ADPCM |
| **Medium SFX (Dialogue, Explosions)** | 2 – 10 seconds | **Compressed In Memory** | Vorbis |
| **BGM & Long Ambience** | > 10 seconds | **Streaming** | Vorbis |

### B. AudioSource Pooling
Never instantiate game objects for one-shot audio effects. Pool preconfigured `AudioSource` components:
```csharp
public class AudioManager : MonoBehaviour
{
    [SerializeField] private AudioMixerGroup _sfxGroup;
    private IObjectPool<AudioSource> _sfxPool;

    public void PlaySound(AudioClip clip, Vector3 position, float volume = 1f)
    {
        var source = _sfxPool.Get();
        source.transform.position = position;
        source.clip = clip;
        source.volume = volume;
        source.outputAudioMixerGroup = _sfxGroup;
        source.Play();

        // Release back to pool when playback finishes using Awaitable
        ReleaseAfterPlaybackAsync(source, clip.length);
    }

    private async void ReleaseAfterPlaybackAsync(AudioSource source, float delay)
    {
        await Awaitable.WaitForSecondsAsync(delay, destroyCancellationToken);
        _sfxPool.Release(source);
    }
}
```

### C. Audio Mixer Routing
Route all sound sources through designated `AudioMixer` groups (`Master`, `Music`, `SFX`, `Voice`, `UI`) to allow volume control, ducking, and DSP effects without hardcoded volume modifications.
