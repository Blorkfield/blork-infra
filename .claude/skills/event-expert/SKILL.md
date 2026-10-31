---
name: event-expert
description: "Expert for the godototype composable event system (formerly \"effect\"). Use this skill whenever designing, extending, or debugging the event system (events/) or any consumer of it (powerups, traps, pickups, magnets, etc.). Activate automatically when the user asks about adding a new EventType, EventTrigger, EventTarget, or any design question about the event layer separation."
---

# Event System Expert

This skill captures the architecture, design canon, extension contracts, and invariants for the composable **event** system at `events/` in the **godototype** repo — a sibling of `kaizend` under the same repo root, which is machine-specific (`~/repo` on some machines, `~/Documents/repo` on others), so resolve it rather than assuming a path. Its log group is `logs/event` (`pluginlogger-expert`).

> **Rename history:** this system was formerly "effect" (`objects/effect/`, `EffectType`, `EffectTrigger`, `EffectSettings`). It was renamed `Effect → Event` to make room for capabilities beyond temporary modification (transitive arming, Start/Stop routing, etc.). **`EffectSettings` no longer exists** — `EventTrigger` is now the composition root and orchestrator.

## When This Skill Applies

Activate immediately (without waiting for explicit request) when:
- User wants to add a new `EventType` (force, scale, shader, …)
- User wants to add a new `EventTrigger` (timer, proximity, interact, …)
- User wants to add a new `EventTarget` (group, children, sensor, location, …)
- User asks where to put new domain logic for an event
- User is designing or debugging a powerup, pickup, trap, magnet, or any node that uses the event system
- User asks whether logic belongs in the trigger, target, or type layer
- User asks how blending, charges, cooldowns, Start/Stop routing, or whole-structure targeting work

---

## System Overview

The event system is a **composable, node-agnostic framework** for applying modifications to any Godot node. Events are assembled in the scene tree from three orthogonal concerns, all rooted at the trigger:

```
EventTrigger        ← COMPOSITION ROOT + ORCHESTRATOR
                       (detection + lifecycle: duration, charges, cooldown, trigger-count)
  ├─ EventType...    ← WHAT IS MODIFIED (gravity, mass, particles, magnet, …) — one or more
  ├─ NodeTarget...   ← WHICH NODES IT HITS (triggerer, explicit, owner, sensor) — zero or more
  └─ LocationTarget… ← WHAT WORLD-SPACE FRAME (for spatial types like magnet) — zero or more
```

**Core principle:** every layer does exactly one job; logic that belongs in one layer must never leak into another. All design decisions flow from this.

**Why EventTrigger is the root:** one trigger fires all sibling `EventType` children. A pickup that changes gravity AND starts particles has one trigger with two `EventType` children — no detection duplication. Each `EventType` filters the resolved target list via its own `SupportedTargetTypes`; types that don't match a target silently skip it.

---

## File Map

```
events/
  EventComponent.cs              ← abstract base [GlobalClass] : Node for all event nodes
  EventStructureUtil.cs          ← FindSiblingRoot(from, siblingParent): structure-root walk
  components/                     ← reusable, scene-tree-free (or lifecycle-only) helpers
    FilterTypeCache.cs           ← reflection cache: SupportedTargetTypes → assignable concretes
    DetectionFilterResolver.cs   ← owns DetectionFilter string[]; resolve + editor dropdown hints
    DetectionWiring.cs           ← wires BodyEntered/AreaEntered on a single Area3D
    DetectionSelector.cs         ← dwell-time tracking + All/Nearest/First/Last selection (SHARED)
    ShapeVolumeSlots.cs          ← Shape3D[] → owned detection-volume subtrees lifecycle (SHARED)
    CollisionShapeSyncer.cs      ← one owned detection node + CollisionShape3D, two-way shape sync
    CollisionNodeWatcher.cs      ← watches a NodePath collision node for TreeExited
  domain/                        ← pure data; no scene-tree references
    IBlendable.cs                ← NewValue, BlendDuration, Evaluate(original, t)
    BlendableFloat.cs            ← IBlendable<float>; optional AnimationCurve; default 0.2s linear
    BlendablePath.cs             ← IBlendable<Curve3D>; snaps at t≥0.5; default 0.5s
    AxisLockConfig.cs            ← snapshot/restore of RigidBody3D axis locks (+ Freeze)
    RecoveryType.cs              ← enum: None | Reset | Bounce (post-event path behavior)
  scene/
    trigger/
      EventTrigger.cs            ← abstract root: detection, self-guard, filter, FULL lifecycle,
                                    target resolution, ApplyBatch orchestration, NotifyStop
      TargetingTrigger.cs        ← abstract marker: produces a detected Node3D source
                                    (required by TriggererTarget)
      MultiTriggerBase.cs        ← abstract: multi-slot (managed + NodeRefs), RequireTriggerAll,
                                    Start/Stop-on-enter/exit routing, cooldown refire
      MultiShapeTrigger.cs       ← abstract: inline Shape3D[] → owned volumes via ShapeVolumeSlots
      AreaShapeTrigger.cs        ← Area3D volumes; TargetSelection + TriggerDelay; Start/StopEvent
      CollisionShapeTrigger.cs   ← contact-monitoring RigidBody3D volumes; Begin/EndContact events
      RaycastTrigger.cs          ← polls RayCast3D[] (NodeRefs only) each physics frame
    type/
      EventType.cs               ← abstract: ApplyTo, Restore, SupportedTargetTypes/OwnerTypes,
                                    ApplyBatch (claim/apply/release), claim guard
      GravityEvent.cs            ← RigidBody3D.GravityScale (BlendableFloat)
      MassEvent.cs               ← RigidBody3D.Mass (BlendableFloat)
      AxisEvent.cs               ← RigidBody3D axis locks via AxisLockConfig snapshot
      ParticleEvent.cs           ← GpuParticles3D.Emitting
      PathEvent.cs               ← PathFollow3D parent Path3D.Curve + rate (BlendablePath/Float)
      MagnetEvent.cs             ← continuous central force toward a LocationTarget frame
      TransitiveEvent.cs         ← duplicates its EventTrigger children onto resolved structure roots
    target/
      EventTarget.cs             ← abstract shared base: must-be-under-EventTrigger warning only
      NodeTarget.cs              ← abstract: "which nodes get modified" → IReadOnlyList<Node>;
                                    WholeStructure; CollectMatchingDescendants; GetSupportedTypes
        TriggererTarget.cs       ←   the entering node (WholeStructure → its structure tree)
        ExplicitTarget.cs        ←   hand-assigned TargetNodes array
        OwnerTarget.cs           ←   the event owner (trigger's parent)
        SensorTarget.cs          ←   abstract: owns a detection volume; Selection via DetectionSelector
          AreaSensorTarget.cs    ←     Area3D volume(s): inline Shapes + optional AreaPath ref
          RaycastSensorTarget.cs ←     RayCast3D[] poll; tracked while ANY ray hits
      LocationTarget.cs          ← abstract: "what world-space frame" → Transform3D?;
                                    valid under an EventTrigger OR a spatial EventType
        NodeLocationTarget.cs    ←   a Node3D's GlobalTransform (falls back to event owner)
```

> **Residual drift (note, don't trust the namespaces):** files moved to `events/` but the C# namespaces did **not** fully follow. They are scattered — `objects.effect`, `objects.effect.type`, `objects.effect.target`, `objects.common`, `objects.common.settings`, and even `objects.particle.type` / `objects.behavior.type` (ParticleEvent / PathEvent). Comments and field names still say "effect" throughout. Treat the *folder* layout above as canonical; the namespaces are an unfinished part of the rename.

---

## Layer Separation Rules (never violate)

**1. EventType owns modification only.** `ApplyTo(Node)` / `Restore(Node)` read and write the target node's properties. They must not touch trigger timing, charge counts, or target resolution. No side-channels to the trigger.

**2. EventTrigger owns detection AND lifecycle/orchestration.** Charges, cooldown, duration, trigger-count thresholds, whole-object counting, target resolution, and `ApplyBatch` orchestration all live here (this is the role old docs gave to `EffectSettings`). On threshold crossing it resolves targets and calls `ApplyBatch` on each sibling `EventType`; it starts the Duration timer and restores applied nodes when it fires. No property-modification logic lives here.

**3. EventTarget owns resolution only.** `NodeTarget` resolves *which* nodes; `LocationTarget` resolves *what frame*. No modification logic, no detection logic.

**4. Domain objects are pure data.** `IBlendable`, `AxisLockConfig`, `RecoveryType`, etc. carry no scene-tree references. Never put scene-tree queries in `domain/`.

**5. Components are mechanism, not policy.** `DetectionSelector` and `ShapeVolumeSlots` are shared by both triggers and targets precisely because they encode no policy — they own dwell-tracking/selection and owned-volume lifecycle respectively. Reuse them; never reimplement them in a new trigger/target.

---

## Trigger Hierarchy

```
EventTrigger (abstract)              detection + lifecycle + orchestration
  └─ TargetingTrigger (abstract)     marker: produces a detected Node3D source
       └─ MultiTriggerBase (abstract) multi-slot, RequireTriggerAll, Start/Stop routing
            ├─ MultiShapeTrigger (abstract)  inline Shape3D[] → owned volumes
            │    ├─ AreaShapeTrigger         Area3D (pass-through) sensing
            │    └─ CollisionShapeTrigger    RigidBody3D contact sensing
            └─ RaycastTrigger                RayCast3D polling (refs only, no managed shapes)
```

- **Slots:** every multi-trigger accepts both **managed slots** (inline `Shape3D[]` it instantiates and cleans up) and **ref slots** (`NodeRefs` `NodePath[]` to existing nodes it only wires). Both feed the same gate.
- **`RequireTriggerAll`:** fire only when *all* slots are simultaneously occupied (uses `NotifyDetectionMultiple`). Warns if <2 slots.
- **Start/Stop routing** (`StartEvent`/`StopEvent` enums per concrete trigger): an enter or exit can *start* an event (apply) or *stop* it (`NotifyStop` → immediate restore of all active applications).
- **AreaShapeTrigger extras:** `TargetSelection` (`ImmediateOnEntry` | `LongestPresent` | `ShortestPresent` | `AllPresent`) and `TriggerDelay` (one pending window; later entrants join the pool without restarting the timer).

---

## Extension Contracts

### Adding a new EventType
1. Create `scene/type/XxxEvent.cs` extending `EventType`.
2. Override `SupportedTargetTypes` (the `Type[]` it can modify — `ApplyBatch` filters by this) and `SupportedOwnerTypes`.
3. Override `ApplyTo(Node)`: cast (safe — already filtered), store original per node keyed by `GetInstanceId()`, apply, return `true`/`false`.
4. Override `Restore(Node)`: look up stored original, revert, remove from the dictionary.
5. **Never start a Duration timer in `ApplyTo`** — the trigger owns that. **Do not call `ClaimNode`/`ReleaseNode` from `ApplyTo`** unless you also override `ApplyBatch` (the base `ApplyBatch` claims/releases for you; note `MassEvent`/`AxisEvent`/`PathEvent` *do* claim inside ApplyTo/Restore as a per-node guard — match the pattern of the type you're modeling on).
6. Blendable values: add an optional exported `BlendableXxx`; in `ApplyTo`, `Evaluate` over a tween if non-null, else snap.
7. **Spatial type?** Read a frame from a child/sibling `LocationTarget` via `GetParent()` trigger's `GetLocationTargets()` (or your own child `LocationTarget`, as `MagnetEvent` does). Apply transient forces in `_PhysicsProcess`; enroll/unenroll in `ApplyTo`/`Restore`.

`ApplyBatch` is the entry point the trigger calls — not `ApplyTo` directly. Override it only for batch-level collapsing (see `TransitiveEvent`, which groups candidates by structure root and arms once per root).

### Adding a new EventTrigger
- **Most cases:** extend `AreaShapeTrigger` / `CollisionShapeTrigger` behavior by configuration, or subclass `MultiShapeTrigger` (shape-based) / `MultiTriggerBase` (ref-based like `RaycastTrigger`). Implement `CreateManagedNode`, `WireDetectionSignals`, `ResolveRefNode`, `ManagedNodeName`/`ManagedColliderName`, and the `StartOnEnter/Exit`/`StopOnEnter/Exit` overrides.
- **Produces a detected node?** Extend `TargetingTrigger` (so `TriggererTarget` is valid under it). Triggers that fire without a detected node (interact/keypress) extend `EventTrigger`/`MultiTriggerBase` directly and `TriggererTarget` will warn under them.
- On detection: base `OnDetection` runs cooldown guard → `IsSelf` → `PassesFilter` → `NotifyDetection`. **Do not decide whether to apply** — `NotifyDetection`/the lifecycle decides.

### Adding a new EventTarget
Fork by **output contract** — never mix:
- **`NodeTarget`** → nodes to modify (`ResolveTargets → IReadOnlyList<Node>`). Return only nodes assignable to a sibling `EventType.SupportedTargetTypes` (use `GetSupportedTypes()` / `CollectMatchingDescendants`); honor `WholeStructure`.
- **`LocationTarget`** → a frame (`ResolveLocation → Transform3D?`). Modifies nothing; read by spatial `EventType`s. Valid as a child of a trigger OR of a spatial EventType.
- **Sensor (AoE)?** Extend `SensorTarget` (a `NodeTarget`). Own the Area3D/RayCast, feed `TrackCandidate`/`UntrackCandidate`, supply `ReferencePosition` for `Nearest`. Dwell-tracking + selection is shared via `DetectionSelector` — do NOT reimplement. Reuse `ShapeVolumeSlots` for inline-shape volumes (see `AreaSensorTarget`).

### Adding a new domain object
- Blendable value: implement `IBlendable<T>`; wire as an optional exported property (null = snap).
- State snapshot: pattern of `AxisLockConfig` (`new(node)` to capture, `ApplyTo(node)` to restore).
- Post-event behavior enum: add to `RecoveryType` or a new enum in `domain/`.

---

## Detection & Lifecycle Flow (canonical)

```
Node3D enters volume (Area/contact/ray)
  → trigger slot enter → OnDetection(node)
      → cooldown gate → IsSelf (skip self/ancestor/descendant) → PassesFilter
      → NotifyDetection(node):
          → if ChargesExhausted: drop
          → counterKey = CountWholeObject ? structureRoot(node) : node
            (CountWholeObject: first part of a structure claims the root; siblings
             ignored until ALL parts of that root exit → NotifyExit clears it)
          → CountMode == PerSource ? per-root counter : single global counter
          → if counter < TriggerCount: return false
          → reset counter; if Charges>0 decrement (disable when 0)
          → ApplyToTargets:
              → ResolveTargets = merge all NodeTarget children (fallback: the triggerer)
              → for each EventType: ApplyBatch(targets)
                  → filter by SupportedTargetTypes → claim → ApplyTo → release
                  → record applied nodes in _activelyApplied
          → return true
      → if Cooldown>0 and fired: block detections; on expire re-fire OnDetection
        for still-overlapping nodes
  → if Duration>0: timer restores the applied nodes for each EventType on expiry
  → Stop routing / NotifyStop(): immediately Restore all _activelyApplied + cancel pending
```

`RequireTriggerAll` substitutes a "all slots occupied" gate that calls `NotifyDetectionMultiple` (resolves targets across all occupants, applies once).

---

## Existing EventTypes — Reference

| Type | Target Node | Modifies | Blendable? | Notes |
|---|---|---|---|---|
| `GravityEvent` | `RigidBody3D` | `GravityScale` | `BlendableFloat` | per-node original by instance id |
| `MassEvent` | `RigidBody3D` | `Mass` | `BlendableFloat` | claims inside ApplyTo |
| `AxisEvent` | `RigidBody3D` | axis locks (+Freeze) | no (snap) | `AxisLockConfig` snapshot |
| `ParticleEvent` | `GpuParticles3D` | `Emitting` | no (snap) | original emitting bool |
| `PathEvent` | `PathFollow3D` | parent `Path3D.Curve` + rate | `BlendablePath`/`Float` | `RecoveryType` |
| `MagnetEvent` | `RigidBody3D` | transient central force | n/a | pulls toward a `LocationTarget` frame in `_PhysicsProcess`; `Strength`/`StopRadius`/`MaxRange` |
| `TransitiveEvent` | `RigidBody3D` (→ root) | spawns its `EventTrigger` children onto the structure root | n/a | "arms" a target; fits detection shape to the whole structure; one spawn per root |

## Existing Targets — When to Use Which

| Target | Use when |
|---|---|
| `TriggererTarget` | Hit whoever entered. `WholeStructure=true` collects the triggerer's whole structure tree. Requires a `TargetingTrigger`. |
| `ExplicitTarget` | Always hit a fixed `TargetNodes` set regardless of who triggered. |
| `OwnerTarget` | Hit the event owner (the trigger's parent) — e.g. self-destroying powerup. |
| `AreaSensorTarget` | AoE decoupled from what fired: bodies in a separate Area3D region (inline `Shapes` and/or `AreaPath`). `Selection` = All/Nearest/First/Last. |
| `RaycastSensorTarget` | Like AreaSensor but region = one or more `RayCast3D` (line-of-sight / beam). Tracked while ANY ray hits. |
| `NodeLocationTarget` | A spatial type (magnet) needs a *frame* — a Node3D's `GlobalTransform` (falls back to the event owner). |

Multiple `NodeTarget`s on one trigger are additive (merged + deduped). `LocationTarget`s are read by spatial types, not merged into the node list.

---

## Scene Tree Constraints (editor warnings enforce)

- `EventTrigger` must be a **direct child of a Node3D** (RigidBody3D, StaticBody3D, …) — it finds the nearest Node3D ancestor as its detection anchor.
- `EventTrigger` should have **≥1 `EventType`** child, else it modifies nothing.
- `EventTrigger` should have **≥1 `EventTarget`** child, else only the triggering node is affected.
- `EventType` / `EventTarget` must be **direct children of an `EventTrigger`** (LocationTarget may also be a child of a spatial `EventType`).
- Multi-triggers need ≥1 configured slot (managed Shape or NodeRef); `RequireTriggerAll` needs ≥2.
- `TriggererTarget` requires a `TargetingTrigger` parent.

---

## Powerup Pattern

A powerup is a `RigidBody3D` with one or more `EventTrigger` nodes as direct children. The trigger is the composition root.

```
PowerupRoot (RigidBody3D)
  ├─ Mesh + CollisionShape
  └─ AreaShapeTrigger (Charges=1, Shapes=[Sphere])
       ├─ OwnerTarget                       ← resolves PowerupRoot + its GpuParticles3D
       ├─ TriggererTarget (WholeStructure)  ← resolves the character's RigidBody3D parts
       ├─ GravityEvent                      ← acts on RigidBody3D from the merged list
       └─ ParticleEvent                     ← acts on GpuParticles3D from the merged list
```

Each `EventType` filters the merged target list by its own `SupportedTargetTypes` — no coordination between types. For "one event, multiple trigger configs," use multiple `EventTrigger` siblings. To remove the powerup on use, react to charge exhaustion in a signal callback, not inside `ApplyTo`.

---

## What Does NOT Belong Here

- **Locomotion physics** (spring torques, balance, stepping) → `euphoria-expert`
- **Story board / orchestrator** → `orchestrator-expert`
- **GravityManager registration** → separate system; gravity events write `GravityScale` on the body directly
- **InputManager / camera** → unrelated; events are physics/property modifications only
