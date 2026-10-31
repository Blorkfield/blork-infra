---
name: pluginlogger-expert
description: "Expert for the godototype PluginLogger system (addons/plugin_logger/) — the single structured logging channel for all game code in that repo, including per-instance DiagChannel output, the CustomGroups.json group/level model, and the GlobalInterceptor capture hook. Activate whenever adding or retuning a log group, adding diagnostic output to a component, debugging missing/flooding log lines, or working in addons/plugin_logger/."
---

# PluginLogger Expert

This skill documents the PluginLogger system (`addons/plugin_logger/`) — the single structured logging channel for all game code in the **godototype** repo — a sibling of `kaizend` under the same repo root, which is machine-specific (`~/repo` on some machines, `~/Documents/repo` on others), so resolve it rather than assuming a path.

## When This Skill Applies

Activate immediately (without waiting) when:
- User wants to add a new log group or change a group's log level
- User is debugging why log messages are missing, appearing at wrong levels, or flooding the console
- User is working on `addons/plugin_logger/` source files
- User asks how to configure per-subsystem log levels
- User needs to understand what `CustomGroups.json` tags do
- User is adding diagnostic output to a component, or wiring up a `DiagChannel`
- User needs to capture every log line regardless of editor settings (scenario recording, tests)

---

## API Surface

```csharp
PluginLogger.Debug("msg");    // LogLevel.Debug  (3)
PluginLogger.Info("msg");     // LogLevel.Info   (2)
PluginLogger.Warning("msg");  // LogLevel.Warning(1) → GD.PushWarning
PluginLogger.Error("msg");    // LogLevel.Error  (0) → GD.PushError
```

`[CallerFilePath]` and `[CallerLineNumber]` are injected automatically — always call the methods without file/line args.

The log string format is: `{level} - [{FileName}:{line}]: {msg}`

---

## LogLevel Enum

```
Disabled = -1   (suppress all output)
Error    =  0
Warning  =  1
Info     =  2
Debug    =  3   (most verbose)
```

Level comparison: `effectiveLevel < level → skip`. So a group set to `Warning (1)` suppresses `Info` and `Debug` but emits `Error` and `Warning`.

---

## Level Resolution (priority order)

Resolved in `PluginLogger._log`:

1. **Test override** — `_testLevelOverride`, when a test has set one.
2. **Group match** — `GroupRegister.TryGetLevelForFilePath(callerFilePath)` scans the caller's file path for any registered tag (directory segment). First match wins.
3. **Global fallback** — `blendporter/project/log_level` project setting (default: `Warning`, `Settings.DefaultLogLevel`).

Then, if the call came through a `DiagChannel`, the **instance level narrows it**:
`effectiveLevel = min(effectiveLevel, instanceLevel)`. An instance can only ever
be quieter than the project settings allow, never louder.

The group lookup is O(1) via a tag index. Results are cached per file path and invalidated on `ProjectSettings.SettingsChanged`.

## `GlobalInterceptor` — capture before filtering

`PluginLogger.GlobalInterceptor` (`Action<LogLevel, string, string, int>`) fires
for **every** message that reaches `_log()`, *before* any level gating. It is the
hook for cross-cutting capture — scenario file logging, replay recorders — so a
capture is independent of whatever the developer happens to have configured in
their editor session. Don't reach for group levels to solve a capture problem.

---

## Group Configuration — `CustomGroups.json`

Location: `addons/plugin_logger/CustomGroups.json`

Current groups (as on disk):

```json
[
  { "Details": { "Name": "logs/ragdoll",   "Tags": ["ragdoll"]   }, "Value": 2 },
  { "Details": { "Name": "logs/camera",    "Tags": ["camera"]    }, "Value": 2 },
  { "Details": { "Name": "logs/event",     "Tags": ["event"]     }, "Value": 1 },
  { "Details": { "Name": "logs/dodgeball", "Tags": ["dodgeball"] }, "Value": 3 }
]
```

(`logs/effect` became `logs/event` with the Effect → Event rename — see the
`event-expert` skill.)

- **`Name`** — project setting key (appears in Project Settings UI under the group name).
- **`Tags`** — directory/folder name fragments matched against the caller's file path. Matching is **exact segment equality** (`ComputeFileLevelFromTags` splits the path and looks each segment up in the tag index), so a file at `events/scene/type/GravityEvent.cs` matches a tag of `"events"` — and does **not** match `"event"`.
- **`Value`** — initial LogLevel integer (`3` = Debug, `2` = Info, `1` = Warning, `-1` = Disabled).

### Adding a new group

1. Add an entry to `CustomGroups.json`:
   ```json
   { "Details": { "Name": "logs/mysubsystem", "Tags": ["mysubsystem"] }, "Value": 2 }
   ```
2. The tag should match a directory or file name segment in the subsystem's file paths.
3. Restart the Godot editor (or toggle the plugin off/on) so `LoggerEntry._EnterTree()` re-calls `GroupRegister.RegisterSettings()`.
4. The new setting appears in **Project → Settings → logs/mysubsystem** and can be changed at runtime without code changes.

### Changing a group's level at runtime

Open **Project → Settings**, filter by `logs/`, and change the value. The change takes effect immediately — `GroupRegister.Refresh()` clears the file-path cache on `ProjectSettings.SettingsChanged`.

---

## Component Logging — use a `DiagChannel`, not the static API

**A component with instances does not call `PluginLogger` directly.** The old
static `RagdollDiag` is **gone** — a single static sink cannot filter per
instance (last writer wins), and two ragdolls in one scene both wanted their own
verbosity. It was replaced by `DiagChannel` (`addons/plugin_logger/DiagChannel.cs`),
which is per-instance:

```csharp
[Export] public LogLevel InstanceLogLevel { get; set; } = LogLevel.Debug;

_diag = new DiagChannel(InstanceLogLevel);   // in _Ready
_diag.Debug("...");                          // → PluginLogger.Debug
_diag.Warn("...");                           // → PluginLogger.Warning
_diag.Log(LogLevel.Info, "...");
```

**The owner creates the channel and hands it down.** `RagdollCharacter` and
`DodgeballController` each build one from their exported `InstanceLogLevel` and
assign it to every subsystem they own — `FootStepper`, `BalanceController`,
`StanceController`, `SpinePostureController`, `RagdollDebugOverlay`, `Dodgeball`
all expose `public DiagChannel Diag { get; set; }`. A subsystem never makes its
own.

Set an instance's `Sink` to redirect just that instance (file, test double,
replay recorder); set it to `null` to silence it. For cross-cutting capture use
`PluginLogger.GlobalInterceptor` instead — see above.

The `logs/ragdoll` group (`Tags: ["ragdoll"]`) controls the project-settings
level for files under `assets/ragdoll/`; `logs/dodgeball` covers
`assets/dodgeball/`. (The ragdoll code moved from `objects/player/ragdoll/` to
`assets/ragdoll/` — the tag still matches because `ragdoll` is a path segment.)

## Diagnostic Overlays

Sampling and on-screen diagnostics are a **separate** kit from logging, in
`core/debug/`: `DebugOverlay3D` (the base overlay),
`DiagnosticRegistry` (registers overlays and serves the `diagnostic` console
command), `ThresholdWatch`, `ConditionalSampler`. Per-tick sampling belongs
there, not in a log group — see the level table below.

---

## When to Use Each Level

| Level | Use for |
|---|---|
| `Error` | Unrecoverable misconfiguration — node paths missing, required children absent |
| `Warning` | Unexpected but recoverable: type mismatch in ApplyTo, signal wiring fallbacks |
| `Info` | Lifecycle events: _Ready, scene loaded, trigger wired/torn down |
| `Debug` | Per-detection, per-frame (event-gated only — never on a timer) |

**Log only on events, never on a timer or every frame.** Threshold crossings, state transitions, and discrete events are appropriate. Per-tick sampling belongs in a visual debug overlay only. See the `[Paradigm 2]` rule in CLAUDE.md.

---

## File Map

```
addons/plugin_logger/
  PluginLogger.cs          ← static API; level resolution entry point; GlobalInterceptor
  DiagChannel.cs           ← per-instance channel (instance level + swappable Sink)
  LoggerEntry.cs           ← EditorPlugin; calls RegisterSettings/_UnregisterSettings
  CustomGroups.json        ← group definitions; edit to add/remove groups
  config/
    LogLevel.cs            ← enum + LogLevels helper (string for PropertyHint.Enum)
    GroupDetails.cs        ← record(Name, Tags)
    GroupSetting.cs        ← GroupDetails + current Value
    GroupRegister.cs       ← registration, tag index, file-path cache, refresh

core/debug/                ← the overlay/sampling kit (NOT logging)
  DebugOverlay3D.cs  DiagnosticRegistry.cs  ThresholdWatch.cs  ConditionalSampler.cs
```

Reference pages: `wiki/ragdoll-diag.md` and `wiki/ragdoll-debug-overlay.md` in
the godototype repo. There is no console/plugin-logger page there — this skill is
the reference for the logger itself.

---

## Gotchas

- **Tag matching is exact segment equality, not a substring or prefix** — a tag matches only a whole path segment spelled the same way. `"event"` does not match `.../events/...`, `.../event_types/...`, or anything else; only a literal `event/` directory. Spell the tag as the directory is spelled. (`logs/event` currently carries `Tags: ["event"]` against an `events/` directory, so it matches nothing — see `the-event-log-group-s-tag-matches-the-directory-it-is-meant-to-cover`.)
- **Global default is `Warning`** — without a matching group, Info and Debug are suppressed. Add a group entry if a new subsystem's Debug logs are needed.
- **Plugin must be active** — `LoggerEntry._EnterTree()` calls `RegisterSettings()`. If the plugin is disabled, group settings are never registered and the global fallback applies.
- **`GD.PushWarning` / `GD.PushError`** — Warning and Error go through Godot's push channels, which appear in the Errors panel and count as editor errors. Keep warnings meaningful; don't use Warning for expected-at-startup log spam.
