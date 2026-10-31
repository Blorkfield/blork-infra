---
name: skill-router
description: "The index of this vault: what each enabled skill covers and which repo it belongs to. Read it when you have a task in hand and want to know which domain skill owns it — especially when the task spans two subsystems, or when you are in a repo whose skills you haven't used before. Not a trigger skill; it routes, it does not do work."
---

# Skill Router

The vault's index. Every enabled skill is already binding — the mandate block in
`CLAUDE.md` names them all and says they are consulted before the work, not when
a description happens to catch your eye. This skill exists for the step before
that: deciding **which** of them your task actually lands in.

Two repos are covered. Check which one you are in first — a skill for the other
repo describes code that isn't there.

## `kaizend` — the Dart/Flutter monorepo

| Skill | Owns |
|---|---|
| `kaizend-expert` | The whole monorepo. The orchestrator's passes (Work, Unblock, garbage time, PR flow, deferred jobs), what the runner picks up and why, wind-down and rate limits, obligations, `board.yaml`/`projects.yaml`/`app.yaml`, the plugin split, card attribution, the Skill Vault itself. |

## `godototype` — the Godot 4 / C# game

| Skill | Owns |
|---|---|
| `euphoria-expert` | Ragdoll locomotion: balance, stepping, posture, stance, gait, jitter. **Mandatory** whenever ragdoll debug log output is pasted. Classifies deviations and writes them out as board cards — `dev/LOCOMOTION_PLAN.md` is gone, and with it the plan file that used to hold them (the live record is `dev/EUPHORIA_COMPLIANCE_PLAN.md` plus the board). |
| `event-expert` | The composable event system (`events/`) and everything built on it — powerups, pickups, traps, magnets. `EventType` / `EventTrigger` / `EventTarget` and the layer separation between them. (Formerly "effect".) |
| `camera-expert` | `camera/`: modes (follow/fixed/orbit/chase/freefly), the claim/priority stack, blending, the boom rig, mouse-look and capture, the `camera` console command. |
| `pluginlogger-expert` | `addons/plugin_logger/`: log groups and levels, `CustomGroups.json`, per-instance `DiagChannel`, `GlobalInterceptor`, and the `core/debug/` overlay kit. |
| `blendporter-expert` | `addons/blendporter/`: the `.blend` import pipeline, project-wide vs per-file settings, scene creation and duplicate handling. |

## Everywhere

| Skill | Owns |
|---|---|
| `story-discipline` | How a request becomes a card before the work starts **and how that card leaves your hands**: the fields every card sets, acceptance criteria nobody has to come back and ask about, the column every worked card lands in, the `## Handoff log` entry it carries there, question-shaped blockers on anything parked in `03_STUCK`, endless-story rules, and creating/editing through the API with your own `X-Kaizend-Context`. Applies to **every** task, in every repo — it is not keyword-triggered. |
| `session-accounting` | What a terminal session owes the story it worked: registering the card with `terminal-cost.py`, booking tokens/turns/duration/cost into the story journal and the board's `costs.jsonl` at every checkpoint, and the `SessionEnd` hook that catches the rest. Also **every** task, in every repo — nobody asks for it. |
| `skill-refinement` | Keeping the shop's prose honest after a change lands: auditing the enabled skills, the generated `CLAUDE.md` mandate blocks, and `docs/*.md` for symbols, areas, epics, and tags that no longer exist, and editing skills through `PUT /v1/skills/{id}`. Run it after a rename or removal, or when a card shows up carrying a dead tag. |

## Rules

- **Match on subject matter, not on formatting.** A card, a pasted log, or a
  plain sentence all route the same way. (The old `<details>`-block trigger
  belonged to godototype's retired `manager.mjs` prompts and is gone.)
- **A task can match several.** Something touching the event system *and* its
  logging is both `event-expert` and `pluginlogger-expert`. Take both.
- **A card id says nothing about the skill.** `SF.*` can be locomotion,
  events, or anything else. Read the content.
- **Nothing matched is a real answer.** Proceed — but `story-discipline` still
  applies, because it always does.
- **A new skill in the vault gets a row here** in the same change that adds it.
  A retired one loses its row: `orchestrator-expert` is disabled, superseded by
  `kaizend-expert` and `story-discipline`.
