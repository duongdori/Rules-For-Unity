---
trigger: model_decision
description: "Game data persistence standards for Unity 6: DTO serialization, atomic file writes against save corruption, PlayerPrefs limits, and version migration."
---

# Unity 6 Data Persistence & Save/Load Standards

This rule governs save game architecture, file serialization, state persistence, and data corruption prevention for Unity 6.

---

## 1. Architectural Boundaries: DTO vs MonoBehaviour

Never attempt to directly serialize `MonoBehaviour` or `ScriptableObject` instances to disk.

### Separation of Concerns:
- **Game Entities (`MonoBehaviour`)**: Responsible for simulation, physics, and rendering.
- **Game Configurations (`ScriptableObject`)**: Read-only baseline designer values.
- **Save Data DTO (`Data Transfer Object`)**: Pure C# `[Serializable]` records or classes containing only primitive fields, arrays, and standard collections:

```csharp
using System;
using System.Collections.Generic;

[Serializable]
public class GameSaveData
{
    public int SaveVersion = 1;
    public long Timestamp;
    public int PlayerLevel;
    public int Gold;
    public List<string> UnlockedItemIds = new();
    public PlayerTransformData TransformData;
}

[Serializable]
public struct PlayerTransformData
{
    public float PositionX;
    public float PositionY;
    public float PositionZ;
}
```

---

## 2. Anti-Corruption: Atomic File Writing Pattern

Writing directly to `save.json` risks corrupting the user's progress if the game crashes, OS kills the process, or power is lost mid-write.

### The Atomic Writing Standard:
1. Write serialized data to a temporary file (`save.json.tmp`).
2. Verify temp file write integrity.
3. Atomically replace the existing save file (`File.Replace` or move), creating a backup (`save.json.bak`):

```csharp
using System;
using System.IO;
using UnityEngine;

public static class SafeSaveUtility
{
    public static void SaveAtomic(string saveFileName, string serializedData)
    {
        string directory = Application.persistentDataPath;
        string targetPath = Path.Combine(directory, saveFileName);
        string tempPath = Path.Combine(directory, $"{saveFileName}.tmp");
        string backupPath = Path.Combine(directory, $"{saveFileName}.bak");

        try
        {
            // 1. Write full content to temporary file
            File.WriteAllText(tempPath, serializedData);

            // 2. Perform atomic replace
            if (File.Exists(targetPath))
            {
                File.Replace(tempPath, targetPath, backupPath);
            }
            else
            {
                File.Move(tempPath, targetPath);
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"[Persistence] Atomic save failed: {ex.Message}");
            if (File.Exists(tempPath)) File.Delete(tempPath);
            throw;
        }
    }

    public static string LoadSafe(string saveFileName)
    {
        string directory = Application.persistentDataPath;
        string targetPath = Path.Combine(directory, saveFileName);
        string backupPath = Path.Combine(directory, $"{saveFileName}.bak");

        if (File.Exists(targetPath))
        {
            return File.ReadAllText(targetPath);
        }

        // Fallback to backup if primary save file was lost
        if (File.Exists(backupPath))
        {
            Debug.LogWarning("[Persistence] Primary save missing; recovering from backup.");
            return File.ReadAllText(backupPath);
        }

        return null;
    }
}
```

---

## 3. Strict Limitations on `PlayerPrefs`

- **FORBIDDEN**: Never store player inventories, progression, unlocks, or high scores in `PlayerPrefs`. On Windows, `PlayerPrefs` writes to the Registry; on mobile, it stores an unencrypted plist/XML file that easily corrupts or gets manipulated.
- **PERMITTED ONLY**: Use `PlayerPrefs` exclusively for non-critical user hardware preferences:
  - Master, Music, and SFX volume sliders
  - Selected language code (`"en"`, `"vi"`)
  - Screen resolution and Fullscreen toggle

---

## 4. Schema Versioning & Data Migration

Every save data model must include a `SaveVersion` integer. When modifying save schemas across game updates:
1. Compare `data.SaveVersion` against `CurrentVersion`.
2. Apply sequential migration steps before feeding data to game systems:

```csharp
public static void Migrate(GameSaveData data)
{
    if (data.SaveVersion < 2)
    {
        // Version 2 added default Starter Sword to unlocked items
        if (!data.UnlockedItemIds.Contains("item_starter_sword"))
        {
            data.UnlockedItemIds.Add("item_starter_sword");
        }
        data.SaveVersion = 2;
    }
}
```
