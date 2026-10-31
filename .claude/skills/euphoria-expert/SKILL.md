---
name: euphoria-expert
description: "Euphoria/NaturalMotion ragdoll physics expert for the godototype repo (assets/ragdoll/). Use this skill EVERY TIME you examine ragdoll debug logs, analyze locomotion behaviour, or discuss balance/stepping/posture controller logic. This skill is mandatory whenever the user pastes log output from RagdollDebugOverlay, FootStepper, BalanceController, SpinePostureController, or StanceController — do not wait to be asked. It defines the Euphoria reference patterns that all controller logic must follow, the deviation classification process, and how discovered deviations get split into board cards."
---

# Euphoria Expert

This skill governs how to analyse ragdoll locomotion logs and translate findings into board cards, in the **godototype** repo — a sibling of `kaizend` under the same repo root.
That root is machine-specific (`~/repo` on some machines, `~/Documents/repo` on
others); resolve it rather than assuming a path.

## Where the code and the logs are

- **Controllers:** `assets/ragdoll/` — `RagdollCharacter.cs`, `balance/`
  (`FootStepper`, `BalanceController`, `StanceController`,
  `SpinePostureController`), `limbs/`, `types/` (`RagdollSettings`,
  `StancePose`, `ForwardLocomotionPose`), `debug/RagdollDebugOverlay.cs`.
  (This code used to live at `objects/player/ragdoll/`; that path is dead.)
- **Diagnostic output** goes through a per-instance `DiagChannel` handed down by
  `RagdollCharacter`, **not** the removed static `RagdollDiag` and not
  `PluginLogger` directly — see the `pluginlogger-expert` skill.
- **Scenario logs are complete whatever the log settings say.** Run
  `./testing/run_scenario.sh --scenario=<id> --log=<path>`; `ScenarioRunner`
  captures every message before any level filter. Never change log groups,
  `logs/*` settings, `persisted_settings.cfg` or `InstanceLogLevel` to get
  output — a run that changes project settings fails (exit 3). A line missing
  from a scenario log means the code didn't emit it.
- **Prose reference:** the repo's `wiki/` has settled pages for most of this —
  `balance-controller.md`, `foot-stepper.md`, `stance-controller.md`,
  `spine-posture-controller.md`, `forward-locomotion-pose.md`, `limbs.md`,
  `ragdoll-character.md`, `ragdoll-diag.md`, `ragdoll-debug-overlay.md`. Read the
  page before re-deriving a mechanism from source.
- **Plans:** there is no live plan file. `dev/` exists but is gitignored and local-only;
  it holds reference docs, not plans. `dev/RAGDOLL_RIG_REFERENCE.md` is live — the
  authored rig spec `RagdollCharacter.cs` points to; diff a `[RigDump]` log against it.
  `dev/LOCOMOTION_PLAN.md` and `dev/DEBUG_TOOLING_PLAN.md` are deleted — do not recreate them.
  The 837 KB `.claude/LOCOMOTION_PLAN.md` is the frozen historical copy: read it
  for the rationale behind a lettered item (C.28, G.20, E.2 …), never write there.
  **Findings now become board cards** — see the `story-discipline` skill.

## Locomotion Type / Stance Data Architecture

The posture/gait system is a small resource hierarchy under `assets/ragdoll/types/`:

- **`RagdollSettings.WalkType` / `.CrouchType`** (`RagdollSettings.cs`) are the
  top-level posture selectors — each holds one `LocomotionType` resource
  (`types/locomotion/Walk.tres`, `Crouch.tres`). `RagdollCharacter` picks
  between them via `_isCrouching` (`type = _isCrouching ? _crouchType : _walkType`
  in `RefreshStance()`).
- **`LocomotionType`** (`types/LocomotionType.cs`) bundles the four directional
  poses for one posture:
  - `ForwardType` (a `ForwardLocomotionPose`) — the **speed-continuous** gait:
    curves sampled by `SampleAt(speed)` across walk→sprint. Only meaningful for
    the walk posture.
  - `Forward`, `Backward`, `Strafe` (`StancePose` — fixed stiffness/damping/step
    scalars, no curves).
- **`RefreshStance()`** (in `RagdollCharacter.cs`) picks per-frame: moving
  backward/strafing always uses the discrete `type.Backward`/`type.Strafe`;
  moving forward while not crouching uses `type.ForwardType` (continuous) when
  it's set, otherwise falls back to discrete `type.Forward`. Crouch has no
  `ForwardType`, so crouch forward motion always uses discrete `type.Forward`.

**`type.Forward` is not dead weight even when `ForwardType` is set.** A second,
independent path — `WalkForwardDefaults()` (in `RagdollCharacter.cs`,
`_walkType?.Forward ?? _standDefaults`) — reads `type.Forward` directly, with
no `ForwardType` gate, and four call sites in the same file use it purely as a
scalar-defaults source (neck/spine stiffness,
etc.), independent of which pose drives the actual per-frame stance. So
`Walk.tres`'s `Forward` field (`types/stances/walk/Forward.tres`) is live and required even
though `RefreshStance()` never selects it for walking. Don't propose removing
a `LocomotionType.Forward`/`walk/Forward.tres` reference as "unused fallback"
without grepping `WalkForwardDefaults()` callers first.

**Adding a new gait (e.g. an injured/stupor/weird-walk variant):**
- If it should affect forward/backward/strafe as a whole new posture: add a
  new `LocomotionType` resource (parallel to `Walk.tres`/`Crouch.tres`), a new
  `RagdollSettings` slot next to `WalkType`/`CrouchType`, and a state flag in
  `RagdollCharacter` (paralleling `_isCrouching`) so `RefreshStance()` selects
  it.
- If it should only alter the continuous forward gait: define a new
  `ForwardLocomotionPose` variant and swap it onto the active
  `LocomotionType.ForwardType` at runtime — no new `RagdollSettings` slot
  needed.

## When This Skill Applies

Activate immediately (without waiting for explicit request) when:
- The user pastes debug log lines containing any of these tags: `[Step]`, `[Balance]`, `[Posture]`, `[Stance]`, `[IvM]`, `[Jitter]`, `[Fight]`, `[Knee]`, `[Sprint]`
- The user describes a visible locomotion symptom (bounce, drift, lean, stumble, collapse, oscillation)
- The user asks why a ragdoll behaviour occurs
- The user asks whether something is a Euphoria deviation

## Euphoria Reference Model

Euphoria (NaturalMotion) is the gold standard. Every controller in this codebase must follow these patterns. Deviations are bugs, not stylistic choices.

### Core Euphoria Principles

**1. Velocity-proportional postural lean**
The forward lean target must scale continuously with current horizontal speed:
`leanAngle = BLO_radians × clamp(speed / referenceSpeed, 0, 1)`
Binary snap (full lean while moving, zero at stop) is a deviation. Lean must fade as speed drops.

**2. Fixed-world upright reference**
All upright-restoring torques must use a fixed world reference (e.g., `Vector3.Up`), never a live body basis. Driving toward `lTorso.GlobalBasis.X` (or any body-attached frame) creates resonant feedback because as the body tilts, the target tilts with it. See `ApplyUprightTorque` convention.

**3. Newton's 3rd law on every torque — and no reactionless forces**
Every `ApplyTorque` on a body must have an equal and opposite counter-torque on the parent body — unless the parent is the world anchor. Missing counters inject net angular momentum into the chain.

The same rule binds linear force. `ApplyCentralForce` / `ApplyForce` on a body has **no reaction pair anywhere** — it is net momentum injected into the system from nothing. A comment claiming "the reaction comes from the ground via the foot spring chain" (as C.28b's does) is not a reaction: the constraint chain transmits the resulting *motion*, but no equal-and-opposite force is ever applied to the ground or another body. In Euphoria, support forces arise from a compliant leg pressing on the ground, never from a controller pushing the pelvis upward. Treat any `ApplyCentralForce` on `HipBody`/`lTorso` as E-DEVIATE unless it is modelling a genuine external contact.

**4. Step placement uses CoM frame at stop**
When decelerating or stopped, step origins must be derived from `comXZ` (current centre of mass), not from `captureXZ` (velocity-projected capture point). Using captureXZ during decel overshoots step targets and creates the large pop at walk stop. This is the C.8 pattern.

**5. Deceleration step height modulation**
As horizontal speed decreases toward zero, the effective leg lift arc should flatten. LegLiftForce (or equivalent) should be reduced proportional to speed during the braking phase. Constant LegLiftForce during decel creates visible vertical bobs as each foot lift-plant cycle remains full amplitude.

**6. Foot stiffness ramp at contact**
Foot stiffness should be lower at initial contact and ramp up through stance. Hard instantaneous stiffness at plant (our current stiffness=30 from tick 0) creates an abrupt impulse that decelerates the CoM sharply, producing a ~2:1 speed oscillation ratio per step cycle. Euphoria uses a contact softener (see E.2 LandingRampTicks, already partially applied).

**7. Lateral step correction is position-based, not velocity-based**
Velocity feedback in step target computation is positive-feedback unstable under LIPM. Step correction for lateral drift must be positional: shift step origin by `lateralError × gain` where gain < 1 to maintain stability. Velocity terms in step placement must be removed from all but the forward capture direction.

**8. Spine decouple is world-space**
The mid-torso (mTorso) decoupling target must be expressed in world space, not relative to lTorso. Relative-frame targets couple yaw lag into pitch, producing the compounding decouple seen in right-turn slump.

**9. Hip brace is HipBody-only**
Hip brace torque (stiffness AND damping) must apply only to HipBody. Counter on lTorso drives forward pitch via the CoM-ahead geometry. No lTorso counter for ankle torques or hip brace — both are in the W.1/W.2/W.4 positive-feedback cascade.

**10. LiftAxis is world-right invariant**
The axis used for leg swing torque must be derived from the world anchor (`rightFlatN`, yaw-invariant) not from the body basis (`lTorso.Basis.Y`), which rotates with yaw and introduces roll torque at non-zero yaw → lateral velocity positive feedback.

**11. A joint's configured axis is not evidence of its physical axis**
`Generic6DofJoint3D` axes are the joint node's *local* frame, which need not match the anatomical axis the rig actually flexes around. Before tuning any joint spring, **verify the axis empirically** — log the per-tick dot product between the joint's candidate local axis and the body axis the flex angle is measured around, during live motion. A reading of exactly `0.000` regardless of gait phase means you are driving an axis orthogonal to the real one.

This is not hypothetical: `ORC.KNEETORQUE` (commit `28fde54`) found the knee's flex range and equilibrium spring configured on local **X** while flexion physically happens around **Y** — and `Cfg` had Y limited to `(0,0)`, so the knee was *mechanically locked straight* while a spring drove an unconstrained twist axis. Symptom was "stiff-legged" gait that no amount of curve tuning would move.

Two corollaries, both of which cost real sessions:

- **`Cfg` couples locked-ness to stiffness.** An axis with `lo == hi` gets stiffness **and damping** forced to 0 (`RagdollCharacter.cs`, the `xLocked`/`yLocked`/`zLocked` ternaries). So swapping which axis is free silently relocates the entire spring, not just the limit.
- **An axis fix is never gait-scoped.** The `Cfg` calls run once at rig setup and apply to standing, walking, crouching, and falling alike. A change made to fix sprint changes the mechanical reality of *every* gait at once. Any parameter tuned before the fix — leg stiffness, contact ramp, step distance — was calibrated against the wrong mechanism and must be re-derived, not patched.

**Compensators written for a broken joint become E-DEVIATE the moment the joint is fixed.** The C.28 family (`HeightCompGain`, `HeightDriveGain`) exists only because the knee could not extend to support the body; with a working knee they are redundant with, and actively fight, the real mechanism. When an axis fix lands, hunt down and retire its prosthetics rather than re-tuning them.

### Jitter Score Interpretation

`[Jitter]` reports the **peak** Seg2 angVel seen in its 1 s accumulation window
(`peak=`, `BalanceController.cs`), not a single instantaneous sample — a torso
legitimately rotates several rad/s during normal gait, so a point sample mostly
measures where in the stride the report happened to land. The band that applies
depends on which context produced the reading; do not apply one band across gaits.

**Segment2 is the pelvis, not the upper torso.** `Torso.cs` sets `Top = Bodies[0]`
and `Bottom = Bodies[^1]`; `RagdollCharacter` sets `_lTorso = torsoNode.Bottom`.
With 3 segments, Segment2 = `_lTorso` (the pelvis body), not the head-end segment.

**Never compare `[Jitter]` as a whole-run mean — compare it per phase, with
matching line counts.** `BalanceController` emits a line only when the 1 s sum
of |Δω| passes `JitterLogThreshold` (15). A high-frequency low-amplitude flip
(e.g. a 60 Hz period-2 brace-damper flip) passes that gate even at rest, so a
flipping build logs several extra low-value idle lines a quiet build doesn't —
and its whole-run mean drops relative to a build with fewer, larger-magnitude
lines. Measured on `sprint-keeps-its-jitter-down-without-the-brace-damper-flipping-every-tick`
(logs `dev/euphoria/logs/sprint-jitter/`, commit `01896b4`): the same sprint
runs read whole-run `jitAvg` 2.39/2.22 vs 3.79/3.34 (looked like a real
regression) but per-phase (`summarize_log.sh`'s `jitAvg` column is already
per-phase) read world 4.91 vs avg 5.05 — no regression. Always compare per
phase and check the line counts match before drawing a conclusion from a
`[Jitter]` delta.

| Context | Seg2 peak | Interpretation | Measured under |
|---|---|---|---|
| At rest / settling | < 0.20 rad/s | Acceptable — below perceptual threshold | peak (1 s window) — carried forward from the instantaneous-signal era, not re-measured against a live anchor this pass |
| At rest / settling | 0.20–0.40 rad/s | Mild jitter — investigate if trending | peak (1 s window) — carried forward, unverified |
| At rest / settling | 0.40–0.60 rad/s | Moderate — likely energy injection source present | peak (1 s window) — carried forward, unverified |
| At rest / settling | > 0.60 rad/s | Severe — fix before next feature step | peak (1 s window) — carried forward, unverified |
| Active walk | < 4.0 rad/s | Acceptable — anchored to the known-good pre-`28fde54` walk baseline (3.21 rad/s mean, `walk_forward --p2=30`, window 10–33 s) | peak (1 s window), `a0b307e`+ reporting |
| Active walk | 4.0–5.0 rad/s | Elevated — investigate if trending | peak (1 s window) |
| Active walk | > 5.0 rad/s | Severe — `a0b307e` HEAD reads 5.20 rad/s mean on the same scenario/window; treat as regression, not noise | peak (1 s window), `a0b307e`+ reporting |
| Sustained sprint | < 5.0 rad/s | Acceptable (matches the HF.2/HF.3 sprint acceptance gate) | peak (1 s window) — carried forward, unverified against a live sprint anchor this pass |
| Sustained sprint | > 5.0 rad/s | Severe — fix before next feature step | peak (1 s window) — carried forward, unverified |

The active-walk band was wrong from `a0b307e` (2026-09-03's peak-window change)
until this pass: `[Jitter]` started reporting the windowed peak instead of an
instantaneous sample, which reads ~2x higher for the same underlying motion,
but the active-walk row still cited the old instantaneous-signal anchors
(1.59 rad/s known-good, 3.04 rad/s known-degraded) — a bare copy-forward that
was never re-run against the new signal. Re-measured on the current diagnostic
(`walk_forward --p2=30`, window 10–33 s): pre-`28fde54` (known-good) now reads
3.21 rad/s mean peak, and `a0b307e` HEAD (known-degraded) reads 5.20 rad/s mean
peak — both roughly double the old instantaneous figures, consistent with a
peak-of-window metric. The rest/settling and sustained-sprint rows were written
against the instantaneous signal too and have not been re-measured against the
peak signal; treat their absolute values as unverified until someone re-anchors
them the same way, but do not assume they are wrong by the same 2x factor
without measuring — the peak/instantaneous gap is context-dependent (an active
walk has much more headroom for a windowed peak to diverge from a point sample
than a settling body or a sprint's already-large baseline does).

Do not cite a bare `Seg2 < N rad/s` acceptance criterion without naming which
context it applies to **and** which `[Jitter]` reporting mode (instantaneous or
1 s-peak) the cited number was measured under — the signal itself has already
changed once (`a0b307e`) without every anchor being re-measured, and it can
again.

Jitter scores that increase monotonically over time (not plateau) indicate slow energy accumulation — a positive feedback loop, not an equilibrium oscillation. These must be diagnosed differently from steady-state jitter.

### Speed Oscillation Ratio

Measure `peak_speed / trough_speed` per step cycle. In the IvM logs, trough is just after plant, peak is mid-swing.

| Ratio | Interpretation |
|---|---|
| < 1.5 | Smooth — good foot compliance |
| 1.5–2.0 | Acceptable for current LandingRampTicks |
| > 2.0 | Harsh vault — foot contact too rigid or ramp too short |

### Log Signal Reference

**A contact impulse readback is an estimate, not a measurement.** `PhysicsDirectBodyState3D.GetContactImpulse` on this engine (Godot 4.7 + Jolt) comes from `JPH::EstimateCollisionResponse` computed at contact time, not from the solver's actual delivered impulse — the same "readback ≠ delivered" gap principle 11 already calls out for joint springs (a `K·(flex−eq)` readback is what the spring *demands*, not what reaches the body). The per-point share follows each point's approach velocity, friction is split evenly across points, and speculative contact points 1–2 cm off the floor carry impulse alongside touching ones — a heel-riding foot can read as toe-loaded purely from this. Never trust a raw per-point `GetContactImpulse` reading on its own:

- **Gate on load, not separation alone.** `FootStepper.FloorPointTouching` counts a point as touching either when it's within `SoleTouchSep`, or when the solver loaded it — post-solve closing (from the body's velocity this step) reaches ≥ 0.9 of its gap — because Jolt acts on a speculative contact whenever the approach would close its gap within the step, so a heel a few mm up can already be carrying load. The contact list's own velocity (`GetContactLocalVelocityAtPosition`) is the pre-solve one and can't show this, so it isn't enough on its own to decide touching. This is what `[SoleAnkle]` does, and `[SoleAnkle] specShare` / `[FloorToe] loadedSpec` are the checks that report how much of the reading came from loaded-but-not-yet-separation-close points.
- **Prefer geometry or a momentum-balance load over the raw readback** where possible — `sup heel/flat/toe` support-phase geometry, or a load derived from the body's own momentum balance, same as the spring-torque case.
- See `a-soled-foot-s-floor-load-and-centre-of-pressure-come-from-solved-forces-not-the-engine-s-impact-estimate` for the follow-on card deriving sole load from solved forces instead of the impact estimate.

| Signal | What it tells you |
|---|---|
| `leanAngle` in Posture snap | Whether postural lean is velocity-proportional (should decrease with speed) |
| `lTilt` growing after stop | Residual forward pitch not being corrected — check leanAngle fade and IBF |
| `brace` value at step trigger | + = leaning back (braking), - = leaning forward (driving). Should approach 0 at steady state |
| `targetFwd` growing over walk | Step placement not reaching equilibrium — drive force exceeds MoveDamping cap |
| `drift` > 2× StepTriggerDistance | Active walk; acceptable. Rising indefinitely = no equilibrium |
| Jitter Seg2 growing over time | Slow energy injection — not plateau oscillation |
| `swingYaw` > ±3° | Yaw contamination in leg swing — check liftAxis derivation |
| speed oscillation ratio > 2.0 | Foot impact too hard — LandingRampTicks too short |
| `[SoleAnkle] specShare` > 0 | Some of the impulse readback came from speculative (non-touching) contact points — treat that fraction as unreliable |
| `[SoleAnkle] cop` | The engine's impact estimate weighted over touching points, **not** the solved centre of pressure. Read +0.140–0.146 m in every arm and variant (lean ×0.5 included) while `sCopSs` moved +0.034 → +0.115 m. Never gate on it; gate on `sCopSs` (solved single-support CoP). Logs `dev/euphoria/logs/toeroom/` |
| `[SoleAnkle] capTk` | Counts the **heel** cap (`SOLE_HEEL_CAP`: toe-up spring demand past `HeelCapBudget`), not a toe limit. Upright walking's budget is `SoleCapLever` = 0, so any toe-up demand counts. Moving the CoM back over the foot *raises* it (0 → 57–83 capped stances), so "cop low" and "capped ≤ control + 5" pull in opposite directions — never write both as gates |
| `[Posture] window lTiltAvg` | The **total** angle of lTorso's up axis from vertical (`acos(lTorso.X · Up)`), so it counts **roll as well as pitch**. Never read it as "the pelvis pitched forward" — a turn's extra lTilt was roll (turn phases, offsets 15–29, sole: forward pitch 5.4° vs control 6.7°, |roll| 6.6° vs 2.6°; card `a-crouched-turn-keeps-its-pelvis-over-its-feet-once-the-stance-pairs-react-through-the-leg`). Any lean gate or card says whether it means pitch, roll or the total |
| `[PostureTilt] window` | lTilt split about the pelvis's own flattened axes: `pitchAvg` (signed, **negative = forward**), `rollAvg` (signed), `rollAbsAvg`. Ticks-weighted per phase like `[Posture] window`. `floorpost/cph.py` and `gates.py` print `pit` and `|rol|` beside `lT`. Cite pitch for "leans forward", `|rol|` for "leans sideways" |
| `[Posture] snap` lTilt mean | Change-weighted: the line fires on a 4° ladder crossing, so a steady walk logs 2 snaps and a wobbling one 150. Never take an lTilt mean from `snap`. The per-tick figure is `[Posture] window lTiltAvg` (walk 12.84° → 15.09° where the snap mean read 11.3/12.5° → 15.4°) |

## Analysis Workflow

When given log output, always do these steps **in order**:

### Step 1: Extract the core metrics

From the IvM lines, compute:
- Speed oscillation ratio: `max_speed / min_speed` per step cycle
- Drift trend: is max drift growing, plateauing, or shrinking?
- targetFwd trend: is step placement length converging?
- brace trend: is it converging toward 0?

From Posture snap lines, extract:
- leanAngle at each snap
- lTilt trajectory (growing, stable, or decaying)
- decouple sign pattern

From Jitter lines, extract:
- Seg2 angVel trajectory over time (growing vs plateau)
- Peak score and when it occurs

### Step 2: Classify each observed symptom

For every visible symptom, classify as one of:
- **E-DEVIATE**: Clear Euphoria deviation (references a specific principle above)
- **PARAM**: Parameter mis-tuning, not an architectural deviation
- **EXPECTED**: Normal LIPM physics — acceptable
- **UNKNOWN**: Symptom present but cause not determinable from logs alone

### Step 3: Write up E-DEVIATE items

For each E-DEVIATE, produce a structured entry:

```
**[X.N] <short title>**
Symptom: what the user sees / what the logs show
Euphoria principle violated: which numbered principle above
Root cause: mechanistic explanation (one paragraph)
Fix: specific code change needed
Expected outcome: what the logs should show after fix
```

### Step 4: File each E-DEVIATE as a board card

There is no live plan file to write into — `dev/LOCOMOTION_PLAN.md` is gone. Each
E-DEVIATE becomes a board card, created through the API per the
`story-discipline` skill, with the structured entry from Step 3 as the body.

Split cards **by mechanism, not by symptom or by gait.** "Fix walk" and "fix
sprint" is the wrong seam when both ride the same speed-continuous curve — it
produces two cards that each invalidate the other's verification. Correct seams:

- A **removal** (retiring a compensator) is its own card. It is verifiable alone
  and needs none of the tuning that follows.
- A **tuning pass** over one curve family is one card, even if it is large. Two
  parameters that jointly determine the same observable must be tuned together
  or you ping-pong.
- A **measurement that gates the rest** (principle 11's axis readback) is its own
  card and lands first. It either passes and the plan proceeds, or it changes the
  plan — worthless as a sub-bullet of the work it gates.

### Step 5: Report to user

Tell the user:
- Which symptoms are EXPECTED (no action needed)
- Which are PARAM (suggest parameter adjustment without filing a card)
- Which are E-DEVIATE (summarize each, name the card filed)
- Which are UNKNOWN (describe what additional logging would disambiguate)

Always be specific: quote the log values that support each classification.

## Card Conventions

- Keep the lettered-code habit (`C.28`, `G.20`, `ORC.KNEETORQUE`) in the card
  title or body. It is the only key back into `.claude/LOCOMOTION_PLAN.md` and
  into the source comments, which cite these codes heavily.
- One-sentence description + an explicit "Expected:" outcome, so the card can be
  verified by someone who did not diagnose it.
- Do NOT write "Phase N work" deferrals — fix order is: diagnose → fix → verify → next.
- If the fix changes a `.tres` parameter, say which file and which field.
- If the fix changes a joint axis or limit, say so prominently: per principle 11
  that is a whole-rig change, and every reviewer needs to know it is not scoped
  to the gait in the title.
- **An absolute gait number in an acceptance criterion must name the commit it
  was measured on.** A bare `76 steps / 1.87 oscillation ratio` baseline goes
  stale the moment a later commit changes gait behaviour — and nothing in the
  criterion records whether that has happened. Prefer the *relative* form for
  no-behaviour-change criteria: measure the scenario **on the same commit, with
  and without the change**, and compare those two numbers to each other, not to
  a number pinned earlier. Worked example (card
  `a-gait-number-in-an-acceptance-criterion-says-which-commit-it-came-from`):
  `one-stance-tells-us-why-a-knee-that-usually-holds-sometimes-folds` carried
  the criterion "`walk_forward --p2=30` metrics after the trace is reverted
  match the current 76 steps / 1.87 oscillation ratio / 0.038 m excursion" —
  measured at `3d9638c`, unattributed. Four gait-fixing commits later
  (`404d600`, `60fbf51`, `5c3480e`, `a02702d`) the same scenario legitimately
  read 127 steps / 1.37 oscillation ratio, so the criterion now asserted that
  four merged improvements had not happened. The corrected criterion measured
  127 steps / 0.9° max **both with and without** the trace, on the same commit
  — that comparison is what actually verifies "no behaviour change," and it
  survives every future commit that legitimately moves the absolute numbers.
  If an absolute number must appear anyway (e.g. a fixed regression gate), name
  the commit it was measured on right next to it.
- **A card that removes a reactionless force is a special case of the rule
  above, and gets it wrong in a specific way if written carelessly.** The
  baseline it would naturally reach for was measured on a tree that still had
  the source running — so "stays within baseline" quietly asserts that
  removing the source changed nothing about what the source was doing, which
  is false by construction whenever the source was contributing something real
  (even if illegitimately delivered). Found in T4.1 (`dev/EUPHORIA_COMPLIANCE_PLAN.md`
  section 5): the card's "`lTilt` within baseline" clause was unmeetable by any
  build that also passed its own `[ExtForce] = 0` gate, because the pre-removal
  baseline (`base`) included the ~6° the reactionless torque was buying just by
  existing. **Write a removal card's gate against a same-tree control instead:**
  build the removal, then A/B it against the identical commit with the source
  reverted (a mode switch, not a different commit) — same mechanism as the
  general rule, but the "without the change" run for a removal card means
  "with the old reactionless source put back," not "an earlier commit." That
  isolates the two things a removal always mixes together: the unavoidable
  loss of whatever the source was actually doing (not a regression — no
  removal-only build can avoid it) from any new cost the *replacement*
  mechanism adds on top (a real, attributable regression). Only the second one
  belongs on the removal card's ledger.
  **A gate against an uncapped control is held to what the control's reactionless
  sources do, source by source.** Read `[LongSwing]`'s `src` tail before writing
  one. Measured on `a-capped-crouch-s-long-swing-gate-does-not-ask-for-the-drift-rate-lateraldamp-buys-the-control`
  (parent `a-capped-crouched-body-stays-over-its-stance-foot-through-a-long-swing`,
  logs `dev/euphoria/logs/floorpost/ls2`, local only): the control holds the sideways
  moment in a long swing with `LateralDamp` (−c·v_lat on the spine, no reaction),
  +4.5–5.0 N·m against the capped arm's +2.1–2.7, and the capped arm makes up the
  gap by falling. Stance drift rate read 0.26 vs 0.08 m/s (sole) and 0.34 vs 0.22
  (sphere). "Within 25 % of the control's rate" asked a floor-reacted build to
  reproduce a reactionless source, so no build could meet it. Report the rate
  against the capped arm without the fix.
  **The same holds for falls and timeouts: before gating them against the
  control, check whether the control's count depends on a reactionless source.**
  On the control, lift each cap in turn and turn each `EXT_OFF` source off in
  turn. If any of those changes the count, the control's number is bought by
  that source, and the gate is the capped arm without the fix, as for the rate.
  Only a count that holds with every source accounted for is a fair gate.
  **A correlation does not clear the control.** That a count does not follow
  the controlled error (timeouts as common with the heading inside 10° as
  outside it) does not show the count is unbought. Only the removal (`EXT_OFF`)
  does: on `d1053cd` the heading control read 8 / 0 / 176 timeouts (wta / wft /
  cta, 15 draws) and 383 / 330 / 353 with `EXT_OFF=YawTorque`. Card
  `a-floor-turned-walker-s-heading-gate-does-not-ask-for-the-timeouts-the-reactionless-yaw-torque-buys-the-control`.
  **Heading lags and turn jitter: read the lag's size, not its count, and never
  take a reference that wins by not doing the task.** Card
  `a-floor-turned-walker-s-pelvis-jitters-no-more-in-turns-than-the-reactionless-control`
  (crouch_turn_alternate, sole, offsets 0–29, logs `dev/euphoria/logs/T4.4/ship/gk1`,
  `fin3`, local only). `[Yaw] lag` counts: control 73, control `EXT_OFF=YawTorque`
  214, the uncapped floor yaw pair (k3) 58, the pair capped at the sole's grip
  (`T44_YAW_BUDGET=grip`) 155. `[Yaw] caught up peak=` per episode, median / p90 /
  max: control 11.7 / 16.2 / 18.9°, no-torque control 30.9 / 169 / 180°, k3 12.9 /
  23 / 40.7°, grip-capped 62.5 / 145 / 180°. A count of 86 against the no-torque
  control's 123 looked like a pass while the capped arm was ending turns facing
  backwards. Gate heading on the per-episode peak beside the count. Turn-phase
  `[Jitter] Segment2` is confounded the same way: the no-torque control reads at or
  below the control in crouched turns (2.84–3.96 vs 3.31–3.78) because it barely
  turns, so "removing the source does not raise the jitter" does not show the
  control's steadiness is unbought. At matched heading every floor-reacted lever
  tried (grip and numeric caps, rate-capped heading, double-support push, step
  lead, stance-leg chain, slower turns) traded jitter against heading one-for-one;
  the control's steadiness *with* heading is what the reactionless torque buys.
  Gate such a fix against the arm without its switch, as above. A floor pair left
  uncapped is still a pair (`[ExtForce] YawTorque` 0): a slipping sole returning
  what friction gives is physics, and a self-imposed cap below it only discards
  heading.
  Measured on card `a-capped-crouch-s-long-swing-fall-gate-does-not-ask-for-the-falls-the-clamp-s-reactionless-upright-and-brace-share-buys-the-control`
  (`crouch_turn_alternate`, offsets 45–74, falls sole / sphere, logs
  `dev/euphoria/logs/floorpost/ra1` and `ra3`, local only). The control (`ctrl`)
  reads 0 / 0, and still 0 / 0 without `LateralDamp` (`ra1` `ctnl`), so the
  drift-rate source is not what keeps it up. The capped arm `bf1` reads 2 / 2.
  With one cap lifted (`ra3`, key `ra3u.key`): `UprightPaired` (`u1`) 0 / 0,
  `HipBracePaired` (`u2`) 0 / 0, `SupportDrivePaired` (`u3`) 0 / 3; the other
  five (`u4`–`u8`) 2–3 / 2–3, the same as `bf1`. All caps lifted (`ra1` `xcap`)
  0 / 0. Uncapped, Jolt's clamp deletes those pairs' thigh half, which leaves a
  reactionless pelvis torque, and that torque is what holds the control up. 13
  floor-reacted levers left `bf1` at 1–12 falls per foot over 30 draws, none at
  0. So "COLLAPSED ≤ control (0)" asked a capped build to reproduce the clamp's
  reactionless share; no build could meet it.
  **A fix that rides a switch is gated against the capped arm without that
  switch, not the control.** Same cause, walk-start case: card
  `a-crouched-sole-s-walk-in-fall-gate-asks-for-the-falls-the-capped-arm-without-walk-start-makes-not-the-clamp-bought-control-s-0`
  (parent `a-crouched-sole-stays-up-through-the-turns-it-walks-into-under-the-arm-it-ships-with`,
  `crouch_turn_alternate`, sole, binary `167f268`, logs `floorpost/sb2`, `sb3`,
  `tt1`, `sw1`, local only). Falls per 15-draw set, `c13` (`bf1` +
  `CROUCH_WALK_START=0.13`) vs `bf1`: 0–14 0 vs 0, 45–59 3 vs 1, 75–89 1 vs 0
  (pooled 4 vs 1 over 45 draws); the control reads 0 everywhere. `bf1` alone
  falls 0–3 per set (0, 1, 1, 0, 3; sd ≈ 1.1). Every fall read is the capped
  stance foot losing the body through a long swing, so "≤ control (0)" is the
  clamp-bought gate again. Gate the fix as `c13 + fix` falls ≤ `bf1`'s falls
  on the same 60 fresh draws + 4 (a 4-set pool's sd is ≈ 1.1 × 2 = 2.2, so 4
  is ≈ 2 sigma), and read each arm's falls per 15-draw set beside the total.
  **A speed gate after a reactionless brake comes off is split at the
  release, and only the part after it is gated.** IdleBrake (−90 N·s/m on the
  crouched spine, no reaction) holds the CoM within about a tick of the stop, so
  the control's crouched-idle peak speed (0.06–0.08 m/s, `crouch_stop` sole) is
  bought. Without it the body leaves its 7° walk lean in the first 0.5 s at
  0.23–0.35 m/s in every arm, floor-reacted levers included (a floor-reacted
  horizontal force is bounded near load × CoP reach / CoM height ≈ 24.6 N ×
  0.1 m / 0.5 m ≈ 5 N; IdleBrake applies ≈ 22 N). Card
  `a-crouched-idle-sway-gate-does-not-ask-for-the-stop-rebound-idlebrake-s-reactionless-hold-buys-the-control`
  (logs `floorpost/ci1`–`ci4`, local only). The check is
  `floorpost/idlesplit.sh <dir>`: `rebound` (peak up to the end of the
  rebound, reported, not gated) and `late` (peak after it). The cut follows each
  draw's own rebound, not a fixed 0.5 s: the rebound peak is the highest `[IvM]`
  speed in the first 0.5 s, and the cut is the first later sample below half that
  peak (a fixed 0.5 s left the rebound tail in `late` on quiet draws, and a
  local-minimum cut fails on rebounds with a second hump). Measured on
  `floorpost/ik0` lf0 (card
  `a-crouched-idle-late-sway-split-starts-where-the-stop-rebound-ends`): o0 / o2 /
  o12 / o13 read `late` 0.11 / 0.11 / 0.09 / 0.11, against 0.17–0.18 at a fixed
  0.5 s. 0.11 is those draws' own sway, so no cut reads lower. Gate `late` of the arm without the
  brake + fix against the same arm without the fix (`EXT_OFF=IdleBrake` +
  `IDLE_LEAN_FADE=0`, "lf0"), on two fresh 15-draw sets, never against the
  control.
- **A "no step while the other foot's last `[Contact]` is a lift" check is scoped
  to the no-input (idle) phases, never read over a whole run.** Under move input a
  swing's stance foot is a bounced planted foot 1-2 cm up, and the control itself
  triggers there: whole-run `offl` per 15-draw set (offsets 0-14 / 15-29, binary
  `f30500e` + `3b7834e`'s switch, counter `floorpost/ios.py`, logs
  `floorpost/cg1`, local only), control: `crouch_turn_alternate` sole 30 / 24,
  sphere 125 / 136; `crouch_stop` sole 19 / 12, sphere 56 / 87; `crouch_forward`
  sole 17 / 15, sphere 50 / 73, all with 0 COLLAPSED. Holding those steps buys
  nothing (`STANCE_GROUND_GATE`: NOT A FIX, falls rise). In idle phases the control
  reads 0, so gate `offl` there only. Card
  `a-bounced-stance-foot-step-check-is-scoped-to-the-no-input-phases-where-the-control-reads-zero`.
- **Every same-binary gate assumes a run repeats line for line — and that
  holds only while game code reads contact lists in a fixed order.** Godot
  4.7's Jolt module hands a body its contacts in thread-timing order, and past
  `MaxContactsReported` keeps whichever arrived first; a float sum over that
  list rounds differently run to run and grows into a different gait. Until
  `4145566` the same binary, scenario and offset gave 2–3 distinct logs (card
  `a-crouched-scenario-run-repeats-line-for-line-on-the-same-binary`, logs
  `dev/euphoria/logs/floorpost/dt`, local only: 4 of 22 offsets split across 3
  batches; with the fix 0). Any new loop over `GetContactCount()` walks
  `FootStepper.ContactOrder(state)` instead of `0..n`, and any new body that
  reports contacts gets enough slots for every one. When two runs of one arm
  disagree, suspect that first: diff two batches of the same arm before
  reading a ±1 fall between arms. `CONTACT_ORDER=0` / `CONTACT_SLOTS=4`
  rebuild the old behaviour.
- **Jolt solves constraints in list order, and left-first is not a mirror.**
  Found on card `the-rig-strafes-the-same-whichever-side-jolt-builds-first`
  (commits `e336597`, `ad57e18`, `c074a6d`; logs `dev/euphoria/logs/solveorder/`,
  local only).
  1. **Jolt ignores `Joint3D.SolverPriority`.** Godot prints `Joint solver
     priority is not supported when using Jolt Physics`, and the run repeats the
     unset run line for line (`solveorder/s1`, k2sR vs k2: 0 differing lines). The
     order is the constraint list order (`ConstraintManager::sSortConstraints`:
     priority, then list index), which every rebuild changes: a limit,
     limit-flag or pin-point set, and every body shape commit. `RIG_JOINT_ORDER`
     (`RagdollCharacter.SetupJointOrder`) is the tool that sets it.
  2. **`RIG_LEFT_FIRST=1` does not mirror the rig's solve order.** `Cfg` in
     `_Ready` rebuilds every 6DOF joint after the build, the left leg before the
     right either way. So "right-first X vs left-first mirror-X" is not a
     handedness test: if build order were the only handedness it would read 0
     with no fix. A handedness A/B reads (a) mirrored directions within one build
     (strafe_l X vs strafe_r mirror-X, same phase index) and (b) the same
     direction across build orders. Reader: `solveorder/hand.py`.
  3. **`--override` evidence from before `e336597` is unverified** when more than
     one override run shared a work dir: the second run usually ran without its
     override (350 of 359 in one batch), including the `velocity_steps=40`
     reading on `a-back-right-strafe-pays-the-hip-brace-pair-what-a-back-left-does-not`
     (`hipbrace-br/r6`).
  4. **More solver iterations do not remove the handedness, and a converged
     solver regresses today's gaits** (`c074a6d`, `solveorder/tl1`–`tl4`). Worst
     handedness gap, k2 / control: velocity_steps 10 0.66 / 0.39, 80 0.17 / 0.21,
     160 0.18 / 0.38, 320 0.26 / 0.13 — not monotonic, sign follows build order.
     At 320 velocity + 8 position steps build order leaves the control (0.054
     across orders) and a fixed side shows: the L foot in back-left strafe
     cascades ~10x as often as the R foot in back-right, in both builds. At 80
     steps vs shipped 10: walk_forward sole pitch −15.8° vs −12.5°,
     sprint_forward sphere 4 falls vs 2, crouch_turn_alternate sole L timeouts 79
     vs 39. A solver setting is a whole-rig change (principle 11's corollary);
     re-derive its gains before it ships.
- **A count-type gate (timeouts, swing drags, step triggers, cascades) is
  compared over 15 draws — `--phase-offset-ticks` 0–14 — on the same binary,
  never over a handful of hand-picked offsets. 15 is the count, not a floor:
  do not sweep more offsets "to be safe". Going past 0–14 needs a stated reason
  (one of the pooled cases below) and at most one scenario at a time; a sweep
  over many offsets on several scenarios queues hours of serialized jobs for
  no extra evidence (2026-10-30: asks of 1200 / 960 / 270 / 150 jobs).** Per-draw counts are noisy:
  strafe_r timeouts run 0–8 per draw in every arm, so a 5-draw total moves
  about ±4 from draw noise alone. Measured on one binary over 15 draws (card
  `a-strafe-timeout-gate-compares-builds-over-enough-draws-to-tell-them-apart`,
  logs `dev/euphoria/logs/sole/stop/d/`): lever 0 read 26 timeouts on the
  named five offsets (o0/o3/o7/o10/o13) against the control's 16 — a
  "10-timeout regression" — and 55 against the control's 59 over all 15. A
  whole card was spent fixing that artefact. **Noise floor:** a difference of
  ~±5 per phase, or ~±5 on a 15-draw total, between arms whose mechanism is
  identical is draw noise (both caps inactive in the backward diagonals read 7
  vs 14 stop drags). A gate must clear that floor to claim one build differs
  from another; a smaller gap is "not distinguishable," not a pass or a fail.
  Write the criterion as "over 15 draws (offsets 0–14), same binary, X ≤
  control + 5", and never as a total over a chosen subset of offsets. Fall
  gates (`COLLAPSED`) are the exception: a fall is an event, not a count, so
  0 falls on every draw run still applies.
  **The ±5 floor holds only while the arms share one mechanism from the first
  tick.** A multi-phase scenario whose early phase differs between arms (a switch
  that acts in phase 0) hands every later phase a different start state, and
  there the same arm against itself spreads far wider. Measured on card
  `a-crouched-turn-per-phase-timeout-gate-clears-its-own-draw-noise`
  (`crouch_turn_alternate`, binary `edd36af`, logs `dev/euphoria/logs/onset/all`,
  local only, arms `ctrl` and `K4Lnd`, four 15-draw sets each): per-phase 15-draw
  totals of one arm span p2 16–43, p5 18–41, p6 12–29 (sd 6.5–10.6) on `ctrl`, and
  p3 28–43, p4 22–41 (sd 5.8–7.2) on `K4Lnd` — two to three times ±5, and no
  trend with offset. For those phases (a stated reason, one scenario at a time) pool 60 draws (offsets 0–59) and gate the
  pooled per-phase total at control + 4·sd_set (sd_set = the larger of the two
  arms' sd across the four 15-draw sets; 4 = 2 sigma on a 4× pool). **The bounds
  are per pair of arms, not constants.** For `ctrl` against `K4Lnd`
  (`onset/all`): p2 +42, p3 +23, p4 +29, p5 +34, p6 +26, p7 +14. For `u90ns`
  (`FOOT_SOLE=1 T44_YAW=4 ARM_YAW=20 T44_YAW_STEP=0`) against `K4LndL6`
  (`heading/p8`, binary `b6e91b3`): p1 +48, p2 +34, p3 +36, p4 +43, p5 +44,
  p6 +28, p7 +36. Pooled p4 reads 150 against 119 (+31) there: it passes at +43
  and would fail at the printed +29, so reusing the first pair's numbers gives
  the wrong verdict. Recompute sd_set from the two arms being compared, every
  gate. Phases the switch acts in (phase 0) keep their own exact gate, and a
  phase whose control spread is 0 is not widened. Phase 1 is gated whenever the
  comparison arm's p1 is nonzero in at least two of the four sets (`u90ns` reads
  35 / 17 / 10 / 9); only a phase 1 that is zero in three or more sets is left
  ungated.
  **The spread behind a pooled bound comes from more than one pool, and from the
  control alone.** Decided on card
  `a-pooled-per-phase-timeout-gate-says-whether-it-allows-for-the-control-s-own-spread-across-same-role-phases`
  (sphere crouch_turn_alternate, binary `34f7429`, logs `dev/euphoria/logs/tick240/s6`,
  local only, `hy` = 240 Hz `TIMEOUT_LAND=1 TIMEOUT_HIP_YIELD=0.15`). Four sets of
  one pool are too few to see how far the control itself drifts: its p2 R reads
  11 10 13 8 over offsets 0-59 (sd 2.1) and 14 19 20 26 over 60-119, 121 in all
  (sd 6.1 over 8 sets). `hy` failed p2 R at 63 against 53.0 on 0-59 and passed on
  60-119 (74 vs 98.7); the matched-offset difference `hy - ctrl` per set is
  +6 +3 +1 +11 +3 -4 +2 -6, mean +2.0, standard error 1.9, no different from zero.
  A per-cell bound that stays inside the control's own pool-to-pool spread fails
  an arm for the pool it landed on. The opposite error also showed:
  `TIMEOUT_HIP_YIELD_AT=0.30` passed every cell on both pools by widening the
  bound with its own set spread, while its 120-draw p2 R (151) was the highest of
  any arm. The rule that card set (ctrl pooled + 2 * sqrt(8) * sd_set over offsets
  0-119, sd_set from the control's 8 sets) is **retired**: it fails the control
  against itself (next paragraph). What still holds from it: an arm's own spread
  never widens its bound, and the control's spread across same-role phases (p2 R vs
  p4 R) is a different quantity, never a bound.
  **Timeout gates compare arm and control at matched offsets, never pooled totals
  from different offsets: before `56f40c7`, `--phase-offset-ticks` stretched every
  phase.** `ScenarioRunner` ended a phase at `Duration + offset / 60` s, so offset 119 ran
  each phase ~2 s longer than offset 0, and offset 239 ~4 s longer — a 2 s
  `crouch_turn_alternate` turn phase triples and turns three times as far. A pool
  of later offsets is a pool of longer phases, not a fresh draw. Found on card
  `a-pooled-per-phase-timeout-gate-passes-the-control-against-its-own-fresh-pool`
  (sole, binary `e0b6ec1` + `ea91576`'s flip, logs `dev/euphoria/logs/floorpost/sfe`
  0-119, `sfh` 120-239, `sfn`, `sfz`, local only). The control's
  `crouch_turn_alternate` moving-phase timeouts climb +0.5 to +0.8 per draw for each
  extra second of phase (p3 L 15-sets 15 26 19 20 25 35 39 34 | 32 48 37 38 35 59 59
  62); `strafe_l` / `strafe_r` show no trend (|slope| ≤ 0.1). The pooled 8-set bound
  failed the control's 120-239 against its 0-119 in 19 cells (11 crouch_turn, 5
  strafe_l, 3 strafe_r), and still failed 4 cells on a matched no-change pair
  (control at offset 2i vs 2i+1, 1/60 s apart): on the strafes its 8-set sd is too
  tight for bursty counts (p3 L sd 0, bound exactly the control's total).
  **The gate:** per 15-draw set of matched offsets, d = arm − control; a cell fails
  when mean d > +5 **and** t = mean / se ≥ 3 (se = sd(d) / √NSETS). Phase 0 keeps
  ≤ control + 5. Calibrated on the sign-flip null of the matched no-change pair
  (2000 draws, 8 sets): P(any cell fails per run) is 4.8 % on `crouch_turn_alternate`
  sole and 0 % on `strafe_l` / `strafe_r`; t ≥ 2 gave 18 % on crouch_turn, t ≥ 3
  alone 8-11 % on every gait. No-change check: `ctrlo` vs `ctrle` and back reads 0
  failed cells on all three gaits. `SWING_FIRST=-1` vs control reads 0 failed cells
  on 0-119 and on 120-239 (largest excess `crouch_turn` p6 L +3.1 ± 1.3), where the
  pooled gate failed it 5 and 2. A comparison across offsets — control 120-239
  paired with control 0-119 — correctly fails crouch_turn's moving phases (+12.8 to
  +22.8 per 15 draws, t 3.3-10.4), because those draws ran 2 s longer per phase;
  it is not a no-change check, so never use one as one. Report the per-set d list
  beside the verdict. Reader: `PAIRED=1 NSETS=8 python3 -I tick240/phasegate.py
  <dir> <scen> ctrl <arm> sole` (`OFF0=120` for the 120-239 pool; both arms must
  have run the same offsets).
  **From `56f40c7` on, offsets no longer stretch phases.** The offset is a draw index:
  each phase runs 0–14 extra 60 Hz ticks from a fixed hash of (offset, phase index),
  0 at offset 0, logged as `jitterTicks=` on `[Scenario] phase_start`. No phase is ever
  more than 0.233 s long, whatever the offset, so a pool at offsets 120-239 is a fresh
  draw of the same scenario. Logs from before `56f40c7` are not comparable with logs after
  it at any non-zero offset: re-run both arms. The matched-offset gate above still holds
  and is still the gate. Delaying only phase 0's start was tried and does not vary the
  draw: the body settles to the same rest state, and offsets 0-239 gave 2 distinct
  `strafe_l` runs and 5 `strafe_r` (240 / 82 / 74 distinct for crouch_turn / strafe_l /
  strafe_r before, 239 / 52 / 71 after). Measured on card
  `a-phase-offset-varies-the-draw-without-stretching-every-phase` (control, sole,
  offsets 0-239, logs `dev/euphoria/logs/floorpost/po1` start delay, `po2` the shipped
  form, `po2z` even/odd pairs, local only). Worst per-phase |slope| of per-draw timeouts
  against offset, per 60 offsets: crouch_turn 0.116 (was 0.775), strafe_l 0.034,
  strafe_r 0.052; paired no-change check `ctrle` vs `ctrlo` 0 failed cells on all three,
  both directions. **Gate a slope against its own shuffle null, never a fixed
  |slope| ≤ 0.1.** With the draws' offsets shuffled (no trend by construction, 2000
  shuffles), crouch_turn's worst cell reads above 0.1 in 52 % of shuffles (median 0.102,
  95th percentile 0.168); the strafes' 95th percentiles are 0.067 / 0.079. A trend shows
  as a worst |slope| past the null's 95th percentile.
  **A side gate (back-left vs back-right `cascade-replan`, and any L-vs-R count) at
  low counts pools enough draws to clear its own noise; "within ±15 %" per 15-draw
  set is retired there.** Measured on card
  `a-low-count-side-gate-pools-enough-draws-to-clear-its-own-noise-instead-of-asking-for-within-15-percent`
  (binary `690cd22` + `STEP_SNAPSHOT=2`, 320 + 8, logs `dev/euphoria/logs/solveorder/sn3`,
  local only). Once a fix leaves ~5–15 per side per set, one arm against itself moves more
  than 15 % set to set: control left-first back-left reads 5 7 4 7 5 2, back-right
  8 8 3 2 3 5, pooled 30 / 29 (z +0.1) and still 5 of 6 sets fail ±15 %. For an even
  Poisson build at ~10 per side one set passes ~25 % of the time, so four cells all pass
  ~0.5 % of the time. A per-set noise form, |BL − BR| ≤ max(15 % of mean, 2√(BL + BR)),
  is the opposite error: it passed all 12 cells on offsets 0–29, yet pooled over 0–89 the
  same arms read 57 / 22 (z +3.9) and 86 / 39 (z +4.2). Two sets cannot see a ~2:1 side.
  **The gate** (a low side count is the stated reason to go past 0–14; 90 is the ceiling too, one scenario at a time): pool 6 sets of 15 draws (offsets 0–89, 90 draws per scenario) per arm,
  same binary, and require z = (BL − BR) / √(BL + BR) with |z| ≤ 2. Keep ±15 % of the
  mean only when the pooled count is above ~200 per side, where 2 sigma is under 15 %.
  A pool under 90 draws is "not enough to gate", not a pass. Reader:
  `python3 -I dev/euphoria/logs/solveorder/gate1.py <dir> <steps> <arm>...`, which prints
  the pooled z and `GATE[...]` verdict beside the per-set rows.
  **An arm picked as the best of many on one draw set is confirmed on a fresh
  set, not on the one it was picked on.** Screening ~10 arms on offsets 0–14 and
  then gating the winner on those same 15 draws is selection, not evidence.
  Screen on one set; the gate is the chosen arm on offsets 15–29 (same binary),
  or on both sets with each passing on its own. Measured on card
  `a-strafing-or-turning-swing-still-plants-once-the-stance-pairs-react-through-the-leg`
  (logs `dev/euphoria/logs/floorpost/q2`, `q3`, local only): arm `P5Y` (roll PD
  50/4 + `SWING_LAND_FADE=0.1` + `STANCE_HIP_YIELD=0.1`) passed all three gates on
  offsets 0–14 (`strafe_l` 7/5 vs control 8/7, `strafe_r` 6/6 vs 1/15,
  `walk_turn_alternate` 9/0 vs 4/4), then failed two on offsets 15–29: `strafe_l`
  R 17 vs control 5, `strafe_r` R 18 vs 5. The runner is
  `dev/euphoria/logs/floorpost/arms.sh`; the switch is `OFF0=<n>`
  (`OFF0=15 ./arms.sh`). **Report per-draw counts beside the total whenever a
  gate is within ±5:** each failure above was one draw with one long stall (11
  and 16 timeouts), so a 15-draw total is decided by whether a stall lands in
  the set, and the per-draw list shows that where the total hides it.

## What NOT to classify as E-DEVIATE

- PostureDamping (locked at 0.5, never change)
- Speed oscillation ratio < 2.0 (expected from current LandingRampTicks)
- Small jitter (Seg2 < 0.25 rad/s) at steady state (contact solver chatter, acceptable)
- lTilt matching BLO within 1.5° during walk (architectural drive force delta, accepted in C.9/C.11)
- Drift threshold crossings during normal walk (expected behaviour)
