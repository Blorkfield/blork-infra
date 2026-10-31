---
name: blendporter-expert
description: "Expert for the BlendPorter Godot editor plugin (addons/blendporter/) in the godototype repo. Use this skill whenever adding settings, extending the import pipeline, debugging import behaviour, or understanding how project-level defaults and per-file import options interact."
---

# BlendPorter Expert

This skill captures the full architecture, settings model, import pipeline, and extension contracts for the BlendPorter plugin at `addons/blendporter/` in the **godototype** repo — a sibling of `kaizend` under the same repo root, which is machine-specific (`~/repo` on some machines, `~/Documents/repo` on others), so resolve it rather than assuming a path. There is no BlendPorter page in `wiki/`; this skill is the reference.

> `addons/blendporter/config/Settings.cs` also owns the project-wide
> `blendporter/project/log_level` setting that `PluginLogger` falls back to —
> see the `pluginlogger-expert` skill before changing it.

## When This Skill Applies

Activate immediately (without waiting for explicit request) when:
- User wants to add a new import setting (project-wide or per-file)
- User is debugging why an import option isn't showing expected values
- User asks how auto-import, first-import, or manual reimport differ in behaviour
- User wants to extend `SceneCreator`, `SceneService`, or `DuplicateHandling`
- User asks how project settings relate to per-file `.blend.import` values
- User is modifying `BlendFileProcessor`, `FileSettingRegistry`, or `PersistentSettingRegistry`

---

## Plugin Overview

BlendPorter is a Godot `EditorPlugin` / `EditorScenePostImportPlugin` that post-processes `.blend` files after Godot imports them. The entry point is `BlendImporter.cs`; the processing logic is in `BlendFileProcessor.cs`.

**Pipeline:**
```
Blender saves .blend
  → Godot filesystem monitor detects change (or file dropped / Reimport clicked)
  → BlendFileProcessor._GetImportOptions(path)   ← registers options, sets defaults
  → Godot runs scene import
  → BlendFileProcessor._PostProcess(scene)        ← applies metadata, creates scenes
```

---

## File Map

```
addons/blendporter/
  BlendImporter.cs              ← EditorPlugin entry; registers plugin + settings on load
  BlendFileProcessor.cs         ← EditorScenePostImportPlugin; _GetImportOptions + _PostProcess
  config/
    Settings.cs                 ← all StringNames, SettingDefinitions, PersistentSettings[], FileSettings[], GetFileSettingDefault()
    Applicators.cs              ← applicator definitions
    Converters.cs               ← type converters
    Properties.cs               ← property definitions
  domain/
    DuplicateHandling.cs        ← enum: Sequential | Skip | Override
    PropertyDefinition.cs
    PropertyRestriction.cs
    SettingDefinition.cs        ← record(Name, PropertyType, Hint, HintString, SettingValue)
  service/
    DirectoryService.cs         ← output directory creation
    MetaPropertyService.cs      ← applies Blender metadata to scene nodes
    SceneService.cs             ← orchestrates SceneCreator calls; threads DuplicateHandling
    SettingService.cs           ← coordinates Register/Unregister of both setting tiers
    component/
      DynamicPropertyApplicator.cs
      FileSettingRegistry.cs    ← reads/writes [params] in .blend.import files
      ImportFileLocator.cs      ← scans project for .blend.import files; caches by name
      NodePropertyAggregator.cs
      NodePropertyScanner.cs    ← reads custom Blender properties from .blend files
      ObjectValidator.cs
      PersistentSettingRegistry.cs ← reads/writes ProjectSettings
      PropertyApplicator.cs
      SceneCreator.cs           ← packs and saves nodes as .tscn; implements DuplicateHandling
```

---

## Settings Model — Two Tiers

This is the most important thing to understand. Settings exist at two levels and interact through `GetFileSettingDefault`.

### PersistentSettings — Project-wide defaults
- Registered into `project.godot` via `ProjectSettings.SetSetting`
- Visible in **Project → Project Settings → blendporter/**
- Only written if the key doesn't already exist (idempotent)
- Current: `log_level` (under `blendporter/project/`), `create_scenes` and `duplicate_handling` (under `blendporter/import_defaults/`)

### FileSettings — Per-file import options
- Appear in Godot's **Import panel** when a `.blend` is selected
- Values stored in the `.blend.import` file under `[params]` section
- Read at import time via `GetOptionValue(setting.Name)` in `_PostProcess`
- Current: `create_scenes`, `duplicate_handling`

### How they interact: `GetFileSettingDefault`

```csharp
public static Variant GetFileSettingDefault(SettingDefinition fileSetting)
{
    var persistent = PersistentSettings.FirstOrDefault(s => s.Name == fileSetting.Name);
    if (persistent != null && ProjectSettings.HasSetting(persistent.Name))
        return ProjectSettings.GetSetting(persistent.Name);
    return fileSetting.SettingValue;
}
```

- Called by `BlendFileProcessor._GetImportOptions` when registering each file setting option
- Called by `FileSettingRegistry.RegisterSettingForFile` when writing missing keys to existing `.import` files
- If a file setting has a matching persistent setting (same `StringName`), the project setting value is used as the default
- This means: new files inherit whatever is in Project Settings at first import

### When each tier is consulted

| Scenario | What drives the import |
|---|---|
| First import of a new file | `_GetImportOptions` default → `GetFileSettingDefault` → project setting |
| Auto-reimport (file changed on disk) | `[params]` in `.blend.import` file — project setting NOT consulted |
| Manual reimport (Reimport button) | `[params]` in `.blend.import` file — project setting NOT consulted |
| User changes value in Import panel + reimports | New value saved to `[params]`, used from then on |

**Key rule:** project settings are defaults for first import only. After a file has been imported once, its `[params]` values are authoritative for all future imports.

### FileSettingRegistry — The backstop

`FileSettingRegistry.Register()` runs at plugin load and ensures all `.blend.import` files in the project have the expected `[params]` keys. It only writes missing keys — it does not overwrite existing values. Uses `GetFileSettingDefault` so new keys get the project setting value, not the hardcoded default.

**Known limitation:** `ProjectSettings.Connect("settings_changed", ...)` does not exist in Godot 4's C# API. Do not attempt to wire up live propagation of project settings to `.import` files via signals — it won't compile.

---

## DuplicateHandling

Controls what `SceneCreator` does when a `.tscn` with the target filename already exists.

| Value | Behaviour |
|---|---|
| `Sequential` | Appends `-1`, `-2`, etc. until a free name is found (original behaviour) |
| `Skip` | Returns `false` without writing; logs at Debug level |
| `Override` | Writes directly to the base filename, overwriting whatever is there |

Threaded through: `BlendFileProcessor._PostProcess` → `SceneService.CreateScenes` → `SceneService.CreateNodeScene` → `SceneCreator.Create`.

---

## Adding a New Setting

Follow this checklist exactly — missing any step causes the setting to appear in one place but not the other.

### Project-wide default only (no per-file override)

1. Add a `StringName` constant to `Settings.cs`
2. Add a `SettingDefinition` to `Settings.cs`
3. Add the definition to `PersistentSettings[]`
4. Read it via `PersistentSettingRegistry.GetSetting(definition)` wherever needed

### Per-file only (no project default, Import panel only)

1. Add a `StringName` constant to `Settings.cs`
2. Add a `SettingDefinition` to `Settings.cs`
3. Add the definition to `FileSettings[]`
4. Read it via `GetOptionValue(Settings.XxxSetting)` in `BlendFileProcessor._PostProcess`

### Both tiers (project default + per-file override — the standard pattern)

1. Add a `StringName` constant to `Settings.cs` — **use the same name for both tiers**
2. Add a `SettingDefinition` to `Settings.cs`
3. Add the definition to **both** `PersistentSettings[]` AND `FileSettings[]`
4. `GetFileSettingDefault` will automatically bridge them — no extra wiring needed
5. Read the effective value at import time via `GetOptionValue(Settings.XxxSetting)` in `_PostProcess`

**Naming convention for setting paths:**
- `blendporter/project/xxx` — genuinely project-scoped (log level)
- `blendporter/import_defaults/xxx` — defaults for new file imports (create_scenes, duplicate_handling)

---

## Import Pipeline Detail

### `_GetImportOptions(path)`

Fires when Godot is about to process a file. Responsibilities:
1. Stores `_currentBlendPath` for use in `_PostProcess`
2. Calls `ImportFileLocator.Add(path)` to register the file in the cache
3. Registers each `FileSettings` entry as an import option via `AddImportOptionAdvanced`, using `GetFileSettingDefault` for the default value
4. Scans the `.blend` file for custom Blender node properties and registers those as import options too

### `_PostProcess(scene)`

Fires after Godot has built the scene graph. Responsibilities:
1. Applies Blender metadata via `MetaPropertyService`
2. Applies dynamic node properties via `DynamicPropertyApplicator` (reads from scanned blend data)
3. Reads `DuplicateHandling` and `CreateScenes` via `GetOptionValue`
4. Calls `SceneService.CreateScenes` if enabled
5. Resets `DirectoryService` and `SceneService` state

---

## ImportFileLocator

Lazy-loaded, process-lifetime cache. Maps stem filename → list of `.blend.import` paths.

- `Add(sourcePath)` — registers a path during `_GetImportOptions`; allows `FileSettingRegistry` to find newly-encountered files
- `Valid()` — returns only entries with exactly one match (disambiguated)
- `Find(nodeName)` — lookup by stem name
- Cache is not invalidated during a session; `_loaded` flag prevents duplicate scans

---

## What NOT to Do

- **Do not call `ProjectSettings.Connect` or `ProjectSettings.Disconnect`** — these methods do not exist on `ProjectSettings` in Godot 4 C#. Attempting to propagate settings changes via signals will not compile.
- **Do not write new settings using hardcoded `setting.SettingValue` as the default** when writing to `.import` files — always use `Settings.GetFileSettingDefault(setting)` so the project setting is respected.
- **Do not put modification logic in `SceneService` or `BlendFileProcessor`** — `SceneCreator` owns the file-writing decision; `SceneService` only orchestrates which nodes to write.
- **Do not import `.blend` files directly in code** — the pipeline is driven entirely by Godot's importer; BlendPorter only hooks in via `EditorScenePostImportPlugin`.
