---
name: camera-expert
description: "Expert for the godototype camera system (camera/). Use this skill whenever working on camera modes, the priority/blend claim stack, the CameraManager singleton, VirtualCamera rig behaviour, mouse-look/orbit/free-fly controls, or the `camera` console command. Activate automatically when the user mentions the camera, a camera mode (follow/fixed/orbit/chase/freefly), camera framing/aiming, mouse-look, camera blending, or a CameraClaim."
---

# Camera Expert

Full architecture, rig model, mode contracts, and gotchas for the camera system at `camera/` in the **godototype** repo — a sibling of `kaizend` under the same repo root, which is machine-specific (`~/repo` on some machines, `~/Documents/repo` on others), so resolve it rather than assuming a path.

## When This Skill Applies

Activate immediately (without waiting to be asked) when:
- User is working on any file in `camera/`
- User describes a camera symptom — wrong framing, not tracking the target, looking the wrong way, jitter on blend, mouse capture stuck, orbit not orbiting
- User wants to add or change a `CameraMode`, or change how a mode positions/aims
- User is touching the claim/priority stack, blending, or the `camera` console command
- User mentions mouse-look, freelook, zoom, or free-fly movement

---

## System Overview

The viewport is driven by **one real `Camera3D`** owned by `CameraManager` (an autoload singleton). Every camera *source* in the scene is a virtual camera that only ever proposes a transform; it never renders. Each `_Process`, `CameraManager` asks the active source for its desired transform + FOV and copies (or blends) it onto the real camera.

```
CameraManager (autoload)
  owns  PrimaryCamera : Camera3D        ← the only rendering camera
  holds _suitors : List<CameraClaim>    ← priority-sorted, highest on top = active
  each frame:  active.GetDesiredTransform() → PrimaryCamera.GlobalTransform (blended)

IVirtualCamera  (contract every source implements)
  GetDesiredTransform() : Transform3D   ← "where the real camera should be this frame"
  GetDesiredFov()       : float
  SetRig / SetFocus / ClearFocus

VirtualCamera : Node3D, IVirtualCamera  ← the concrete implementation used in scenes
```

## File Map

```
camera/
  CameraManager.cs     ← autoload; claim stack, blending, `camera` console command
  IVirtualCamera.cs    ← the source contract
  CameraClaim.cs       ← record(Source, Priority, BlendIn)
  VirtualCamera.cs     ← [Tool] Node3D; CameraMode enum + all per-mode behaviour
```

There is **no `VirtualCamera.tscn`** — the rig is code-owned. `_Ready` looks for
`CameraContainer/Camera` and, if a scene hasn't authored those children, creates
them itself. So a `VirtualCamera` node dropped anywhere in a scene is a complete
rig; authoring the children by hand is optional and only worth it to tweak the
container's authored framing. Reference page: `wiki/camera-manager.md`.

---

## The Rig (critical — most bugs come from misunderstanding this)

The rig is a **three-node boom**:

```
VirtualCamera   (Node3D, rig root)      ← positioned at/near the focus; yaw lives here
  CameraContainer (Node3D)              ← local offset CameraOffset (0,3,5); pitch tilt lives here
    Camera        (Camera3D)            ← the source Camera3D; looks down its local -Z
```

`ApplyToContainer()` writes:
- `container.Position = CameraOffset * _zoom`   → the boom arm (5 back, 3 up by default)
- `container.Rotation.X = deg(CameraPitchDeg + _pitchOffset)`  → in-place pitch tilt

**Key consequence:** the container offset is a *lever arm* off the root. Rotating the **root** swings the camera *around* whatever the root sits on (an orbit). Rotating the **container** tilts the camera *in place* (a swivel). Yaw is on the root; the authored pitch tilt is on the container. This is why, historically, yaw orbited but pitch only swivelled — see the Orbit note below.

`GetDesiredTransform()` normally returns `_camera.GlobalTransform` (whatever the rig geometry produces). Two modes **override** it and compute a transform directly with `LookingAt`: `Fixed` and `Orbit`.

---

## Camera Modes (`CameraMode` enum)

| Mode | Position each frame | Aim | Mouse | Notes |
|---|---|---|---|---|
| `Follow` | rig root lerps to focus (`FollowPos`) | yaw from `_yawDeterminate` + `_yawOffset`, held (no auto-face) | freelook overlay only (hold middle-mouse) | Default. Original behaviour. |
| `Fixed` | never repositions — rides the scene graph (parent it under a mover) | `GetDesiredTransform` `LookingAt` the focus each frame | none | Security-cam that swivels to watch the target. |
| `Orbit` | boom arcs around focus from yaw **and** pitch | `LookingAt` the focus (always framed) | **captures** mouse; look drives yaw/pitch; wheel zooms | 3rd-person orbit. See below. |
| `VelocityChase` | rig root lerps to focus | yaw derived from focus's direction of travel, `LerpAngle`-smoothed | none | For projectiles with no intrinsic facing. |
| `FreeFly` | WASD + Jump/Crouch move the root | yaw on root, pitch on container, from mouse | **captures** mouse | Detached free cam; ignores focus. |

### Follow-mode freelook overlay
`Follow` supports a temporary orbit: `camera freelook on`, then **hold middle-mouse** to drag yaw/pitch (`_middleMouseHeld`). It does not capture the OS mouse. Distinct from `Orbit`, which captures.

### Orbit — how it must work (and the fixed bug)
Orbit is a **spherical boom around the focus**: yaw swings the camera position around world-up, pitch swings it around the yawed right axis, and the camera always `LookingAt`s the target. This is computed directly in `GetDesiredTransform()`'s Orbit branch — **not** via the container tilt.

**The classic bug:** if pitch is applied as the container's in-place `Rotation.X` (the `ApplyToContainer` path), mouse-pitch makes the camera *look straight up/down while staying in the same position* — it swivels instead of orbiting. Only yaw (on the root) orbited. The fix is to treat pitch as part of the boom (rotate the offset arm about the right axis) so the camera *position* arcs over the target. Clamp boom elevation to ±85° so `LookingAt(…, Vector3.Up)` never hits the poles.

---

## CameraManager — Claim Stack & Blending

- Sources register with `Request(source, priority, blendIn)` → returns a `CameraClaim`. Suitors are kept sorted; **highest priority on top is active** (`_suitors[^1]`).
- `Release(claim, blendOut)` removes it; if the active source changed, a blend starts.
- **Channel override:** `FocusNode(node)` pins the active source to the first claim inside `node`'s subtree, bypassing priority. Pass `null` to clear. Use only for deliberate control (e.g. the `control` console command) — never as normal camera logic. `GetActiveSource` honours the override only while its claim is still in `_suitors`.
- **View override (`camera view`):** `FocusSource(source)` pins the viewport to a specific virtual camera chosen from the console, **taking precedence over the channel override**. `ClearViewOverride()` drops it. Crucially, any subsequent `FocusNode` (i.e. a `control` channel change) **clears the view override** so channel/claim logic resumes — this is the intended "changing the control channel re-asserts the camera claim logic" behaviour. Precedence in `GetActiveSource`: `_viewOverride` → `_channelOverride` → top of `_suitors`.
- **Blending:** `StartBlend(duration)` captures the current real-camera transform/FOV, then `_Process` smoothsteps (`t*t*(3-2t)`) from it to the target with `InterpolateWith`. Duration floored at 0.01s.
- `IsActiveSource(cam)` is what a `VirtualCamera` checks to decide whether it owns input/mouse this frame.
- `UpdateFocusedEntity()` feeds `StatsManager.FocusedEntity` (the active source's parent) for the debug overlay.

## Mouse Capture
`UpdateMouseCapture(active)` captures the OS mouse **only** when this camera is the active source AND mode is `Orbit` or `FreeFly`. `_ExitTree` releases capture if held — always restore `Input.MouseModeEnum.Visible` on teardown so a removed camera can't strand a hidden cursor.

## Offsets & Determinates
- `_yawOffset` — accumulated yaw from mouse / `camera yaw` / presets. `OffsetYaw` wraps mod `Tau`; `SetYawOffset` normalises to `[0, Tau)`.
- `_pitchOffset` — mouse pitch, clamped in `OffsetPitch` to `[-80 - CameraPitchDeg, 80 - CameraPitchDeg]` (so container total pitch stays within ±80°). Orbit additionally clamps its **boom** elevation locally to ±85°.
- `_yawDeterminate : Func<float>` — a live yaw source (e.g. `RagdollCharacter` pushes its facing every frame). `_yawOffset` accumulates *on top* of it, so player look survives the character's per-frame `SetYaw`.
- `Snap()` — teleport to the focus after a scene reset so the camera doesn't lerp across the level. `Fixed` never snaps onto the focus (it stays authored).

## The `camera` Console Command (registered in `CameraManager._Ready`)
```
camera list                            ← list registered virtual cameras (* = active)
camera view <n|name|priority>          ← look through a specific camera; 'priority'/'clear' drops the pin (FocusSource / ClearViewOverride)
camera front|back|left|right|side      ← yaw presets (SetYawOffset)
camera mode <follow|fixed|orbit|chase|freefly>
camera yaw <deg>|<val>rad              ← OffsetYaw (relative)
camera pitch <deg>|<val>rad            ← OffsetPitch (relative)
camera zoom <delta>                    ← OffsetZoom (e.g. -0.2 pulls in)
camera reset                           ← ResetToScene (scene-authored Mode + cleared yaw/pitch/zoom)
camera freelook [on|off|toggle]        ← Follow-mode middle-mouse orbit
```
`list`/`view` operate at the manager level (they don't need the active source to be a `VirtualCamera`). Presets and `reset` route through `ApplyCameraPreset`; the remaining subcommands require the active source to be a `VirtualCamera` (else "Active camera does not support rotation."). `camera reset` restores the mode captured in `VirtualCamera._Ready` (`_authoredMode`), not just the offsets.

---

## What NOT to Do

- **Don't put orbit/aim pitch on the container `Rotation.X`.** That swivels in place. Position-arc pitch (yaw + pitch as a boom) belongs in the `GetDesiredTransform` Orbit branch. Container tilt is only for the non-orbiting modes' authored framing.
- **Don't let a source drive the real `Camera3D` directly.** Everything goes through `GetDesiredTransform` → `CameraManager`. Bypassing it breaks blending and priority.
- **Don't drive an orbit/LookAt toward the poles.** `LookingAt(target, Vector3.Up)` degenerates when the view is near-vertical — clamp boom elevation (±85°) before building the transform.
- **Don't forget to release mouse capture on teardown** (`_ExitTree`) — a stranded captured mouse is a hard-to-diagnose "cursor disappeared" bug.
- **Don't add auto-facing to `Follow`.** Follow deliberately holds yaw (determinate + offset) and does not rotate toward the character's travel/facing. `VelocityChase` is the auto-facing mode.
- **`CameraOffset` is uniform-scaled by `_zoom`.** Its Y/Z ratio (rest elevation `atan2(Y, hyp(X,Z))`) is preserved under zoom — rely on that when computing orbit boom clamps.
