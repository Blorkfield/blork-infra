---
name: kaizend-expert
description: "Expert for the kaizend monorepo: the orchestrator's passes (Work, Unblock, garbage time, PR flow), the wind-down engine, post-completion obligations, how to add a pass, and the surrounding structure — the plugin split, Projects/Areas, card attribution, and the Skill Vault. Activate whenever working anywhere in ~/repo/kaizend (packages/kaizend-server, kaizend-app, kaizend-shared, kaizend-plugin-api, packages/plugins/*), touching board.yaml / projects.yaml / app.yaml, or reasoning about what the runner picks up and why."
---

# Kaizend Expert

Governs the kaizend monorepo. Its centre is the orchestrator: a server-backed
runner that drives AI passes over folder-as-column boards. kaizend is the port
target of godototype's `manager.mjs` system, and it has now fully replaced it —
godototype's own board is a kaizend server board, and that repo's
`.claude/locomotion/orchestrator/` is dormant history. Story-authoring rules live
in the **story-discipline** skill, not here.

## When this applies

Activate without being asked when the user:
- works anywhere in the `kaizend` checkout — server, app, shared, plugin API,
  plugins. The repo root above it is machine-specific (`~/repo` on some
  machines, `~/Documents/repo` on others); resolve it, never assume it
- works in `packages/kaizend-server/lib/orchestrator/` (runner, decision,
  status_writer, runner_state, worker, listener, notify, pr, priming, jobs)
- touches the runner/board API (`packages/kaizend-server/lib/api.dart`,
  `POST …/runner/*`, `GET …/status`, `…/orchestrator/capabilities`)
- edits `OrchestratorConfig` / `board.yaml` knobs, `projects.yaml`, `app.yaml`,
  or the Unblock/garbage-time UI
- asks what the runner picks up, why a card was/wasn't worked, or about
  rate-limit / wind-down / garbage-time behaviour (preempt was removed — S10)
- adds or designs a new pass, or a new app plugin

## Where things live

```
packages/
  kaizend-server/      Dart server: board store, API, orchestrator, skill vault,
                       users/auth, projects, migrations
  kaizend-app/         Flutter app (Linux/iOS); ui/, plugins/registry.dart
  kaizend-shared/      DTOs + models both sides share: Story, api_dto, app_config,
                       project, search_query
  kaizend-plugin-api/  the plugin boundary: KaizendPlugin, PluginHost,
                       extension points, plugin settings/fields/services
  plugins/             one folder per plugin package, each named
                       `kaizend-plugin-<id>`: ai, analytics, spellcheck,
                       terminal, user
dev-boards/            the live boards (blorkfield, kaizend, godototype) plus
                       app.yaml, projects.yaml, users.yaml, .skills/ (the vault)
docs/                  the as-built port plans — see the pass table below
```

**Plugins are compile-time modules, not runtime drop-ins** (Flutter cannot load
Dart at runtime). Adding one is exactly two edits — a path dependency in
`kaizend-app/pubspec.yaml` (plus the root `workspace:` list) and one entry in
`kaizend-app/lib/plugins/registry.dart`. **Nothing else in the app may name a
specific plugin**; everything a plugin contributes goes through a typed
extension point declared in `kaizend-plugin-api`. Compiling a plugin in does not
start it — the user enables it per install in App Settings → Plugins, which
writes the `plugins:` map in `app.yaml`. Full contract: `packages/plugins/README.md`.

## The pass model

A **pass** is a distinct AI-worker mode the runner drives over a board. Each
spends tokens on a different job.

| Pass | Status | Core files |
|---|---|---|
| **Work** | live | `runner.dart` (the one resident run), `decision.dart` (pick), `status_writer.dart` (apply) |
| **Unblock** | live (S9), **button only** since 2026-09-05 | `unblock.dart` (pure) + `unblock_executor.dart` (real AI re-attempt over the `WorkerDriver` seam, routed through `applyWorkerResult`). Reached only through `POST …/runner/unblock` and the chip that calls it — the runner never starts one on its own (`the-runner-stops-unblocking-on-its-own`): a person reads the card's `## Blockers` questions, writes into the `**A:**` slots, and presses the button |
| **Garbage time (General, S10a)** | live (S10 Commit C) | `garbage_time.dart` (pure picker/scope/promotion) + `garbage_time_executor.dart` (button pass over the `WorkerDriver` seam → `applyWorkerResult`). Subsumes the old "Audit pass" idea (wiki/audit work is configured here) |
| **Garbage time (Wind-down, S10b)** | live (S10 Commit B) | a runner-loop step (`runner.dart _windDownGarbageTime`), not a pass — drains small gc on wind-down + the small→large promotion counter |
| **Post-completion obligations** | live (S11) | `obligations.dart` + the `applyWorkerResult` hook; config-driven `completionObligations` auto-create follow-up cards via the shared `BoardStore.createCard` primitive |
| **Stop-time + token buffer** | live (S12) | a RUN CAPS section (`run_caps_menu.dart` `RunCapsSection`) in the `⋮` of every surface that starts a run, holding an optional force-stop + token-buffer % + aggressive-idle → per-run `RunnerState.shutdownAt`/`tokenBufferPct` (`withRunCaps`); deadline enforced in the runner loop (finish round → stop via `pastShutdown`); `bufferScale` (`runner_state.dart`) scales the wind-down threshold in the final session; `resolveDeadline` in `package:kaizend_shared`. No new header UI. |
| **PR publish (part of Work)** | live (P4) | `runner.dart` Work completion, when `config.createPrs`: publishes `automation/<id>` + PR instead of applyBack, via `pr.dart` (pure) + `pr_git.dart` (`PrGitOps` seam) |
| **Update PRs** | live (P5) | `pr_update_executor.dart` (merge-check / sync-with-main / PR-comment rework / branch-gc) over the `PrGitOps` seam → `applyWorkerResult` |
| **Groom** | **REMOVED 2026-09-05** | Was `groom.dart` + `groom_executor.dart`: an ambiguity-audit pass that read a card, asked what it could not answer, and wrote the questions back. Deleted with the epic "Deliberate Unblocking" (`the-groom-pass-comes-out-root-to-stem`) — it spent tokens producing questions no human was answering, and the answering is now the *human's* act on a `## Blockers` form (see the Unblock row). Nothing rebinds its columns: `pass_descriptor.dart` keeps `groom_input_columns` / `groom_failure_columns` / `groom_output_columns` / `groom_when` only as retired keys that warn and drop on save. **Do not re-add it** — write the criterion properly when you author the card (**story-discipline**) instead |
| **Auto-split** | **REMOVED 2026-08-11** | Was `split.dart` + `split_executor.dart`: a non-converging story got a haiku analysis worker that proposed 2–4 children and the parent was marked abandoned. Deleted because both halves misfired — the trigger trusted a single round-timeout as non-convergence (retiring a story that had just applied a productive diff, with rounds still on its budget), and the analyzer read the card text rather than the diff the round had just landed, so it proposed children describing work already committed. **Do not re-add it.** The spend guard, max-rounds and round-timeout all simply park the story `blocked`, edits kept, waiting for its human |
| **Deferred work (jobs)** | live | `job_queue.dart` (pure state machine, bounded retries, per-repo serialization, tolerant persistence) + `job_executor.dart`. One engine, two completion modes: **recording** is fire-and-forget and keys off a sha, **scene tests** are awaited and hand results back to the next round (`docs/SCENE_TEST_DECOUPLE.md`). A worker never runs a scene test itself — it returns `RESULT status=awaiting-test` naming scenarios |

**There is one run mode (2026-09-06, epic "Single Run Mode").** `work` (a
bounded session) and `loop` (run-and-stay) were two ideas of what a run is,
inherited from the Obsidian automator and the prototype board. They converged
and the pair is gone: `OrchestratorRunner.start` takes no `mode`,
`RunnerSnapshot` carries none, and `POST …/runner/{start,loop,work}` are three
spellings of one verb (`loop`/`work` kept as thin aliases for old clients). A
run is defined by its **scope**, not its mode — clicking Work on a column, or a
card, starts *the* resident run over that scope (`AiPassController._startRun` →
`runner/start`), and the pill's Start is the same call with no scope. It works
until the scope runs dry, the force-stop passes, or a human stops it; the
end-of-run line names only those three. **Wind-down and the rate wall throttle a
run, they never end it.** An empty cycle first waits on any in-scope scene-test
park, then bursts garbage time (`_emptyCycleGarbageTime`, when the run's
`idleGarbageTime` cap is aggressive — the default), and only a cycle with
nothing left to do either way ends the run. Both gc drains are fenced by the
run's own scope (`_outOfScope`), so a column-scoped run never keeps busy in a
column nobody pointed at.

**Runner startup priming (kaizend-only, Track R, `docs/ORCHESTRATOR_GAPS_PORT.md`):**
before its first work pick (and again on every rate-window wake), the resident
runner primes the board with **Update PRs** — and nothing else (`priming.dart`'s
`planStartupPriming` + `runner.dart`'s `_runStartupPriming`), gated on its own
config-issues check plus a `primingCandidateCount` pre-check, zero tokens when
unbound or when no card carries an open PR. It reuses `OrchestratorRunner.
prUpdate()` in-process, so it can't self-409 and inherits the
ledger/obligations/lock behavior for free.

**The sequence used to be three steps and is now one** (2026-09-05, epic
"Deliberate Unblocking"). Groom was deleted outright
(`the-groom-pass-comes-out-root-to-stem`) and Unblock left the plan
(`the-runner-stops-unblocking-on-its-own`): re-attempting a blocked card against
blank `**A:**` answer slots only spends tokens re-asking the question a human has
not answered yet. Unblocking is a button a person presses, not a step a run
takes. `PrimingStepKind` has one value, and a new priming step registers here —
but only if it can pay for itself without a human in the loop.

Full design: **`docs/ORCHESTRATOR_PASSES_PORT.md`** (the as-built spec; the whole
S9–S12 pass port is COMPLETE) + **`docs/PR_PASS_PORT.md`** (the PR-flow port, P1–P8,
landed 2026-07-02 — reverses the S9 "PR-rework does not map" verdict). Roadmap
context: `docs/PHASE2_ORCHESTRATOR_PLAN.md` (S1–S8 core + S9–S12 pass ports all
done).

## How to add a pass (mirror Garbage time — the template)

0. **Describe it once** in `kaizend-shared/lib/orchestrator/pass_descriptor.dart`:
   a `PassDescriptor` in `PassRegistry` naming the pass, its label, its icon key,
   its ordered `PassRole`s (type, published key, optional?, YAML + wire spelling
   with any legacy aliases, and how to read it off the config) and its validator.
   **This is the only place a pass's shape is written.** The capabilities
   endpoint, `passColumnRoles`/the lock, `board.yaml` parse+emit, the
   `BoardConfigJson` wire mapping and the client's `AiPass` all derive from it —
   the hand-written `<Pass>Requirement` tables and `missing…Roles` functions are
   gone (2026-08-09 `pass-descriptor-registry`).
1. **Pure core** in `orchestrator/<pass>.dart`: `scope<Pass>Targets`
   (scope→cards), `decide<Pass>` (verdict→action) — role types from the
   descriptor, NEVER hardcoded column names — the AI JSON/RESULT contract, a
   `<Pass>Report`/`<Pass>CardOutcome`. Unit-tested, zero tokens.
2. **Role binding**: add the column fields to `OrchestratorConfig` (each one
   read by a role on the descriptor) + a `<pass>ConfigIssues()` validator +
   getter, which the descriptor points at. **Boards bind columns to role types**
   via config. **Write NO binding UI** — the pipeline surface derives every
   binding field from `PassRegistry` (see "The workflow surface" below), so a
   new role appears on the canvas and in the pass inspector for free. Server-side
   consumers still read requirements/bindings off the capabilities endpoint.
   (2026-06-23 role-binding decision.)
3. **Executor** in `orchestrator/<pass>_executor.dart` over the `WorkerDriver`
   seam (`runner.dart`) — the same one Garbage time, Unblock and Update PRs all
   take, so the pass gets the worktree + resumed session + diff-back for free and
   is testable against a fake driver with zero tokens. Route every completion
   through `applyWorkerResult`, never by moving a card yourself. **Gate before
   spend:** validate config + scope and return an error report with ZERO tokens
   on misconfig. Copy `garbage_time_executor.dart`'s `runGarbageTimePass` — it is
   the shortest complete example of all four.
4. **API**: `POST /v1/boards/{board}/runner/<pass>` (body = `ActionScope` JSON;
   reuse `ActionScope` — all/column/story — verbatim). Return the report; **400,
   not 500**, on unbound roles / invalid scope. Add a `<pass>` entry to
   `GET …/orchestrator/capabilities` — one `_passCapability(PassRegistry.<pass>, o)`
   call, since requirements/bindings/missing all come off the descriptor.
5. **UI** — all of it inside the AI plugin
   (`packages/plugins/kaizend-plugin-ai/lib/src/`), none of it in the host. One
   `AiPass` constant in `passes.dart` (mirror `AiPass.garbageTime`) carries the
   pass's whole surface: its `PassDescriptor`, its busy/enabled/`notConfigured`
   labels, its `ColumnRule`s and their `emptyReason`s, its blast-radius text and
   confirm strings, and `perColumn` — garbage time sets it `false` because it is
   a whole-board "spend the spare budget" verb launched from the pill, while a
   column-scoped pass leaves it true and grows a header chip via `aiColumnChips`
   (`ui/column_chips.dart`). Add the pass to `AiPass.all`, and to
   `AiPass.columnOrder` only if it claims a column. Then one arm in
   `scopeGroupsFor` (`ui/scope_menus.dart`) beside `garbageTimeScopeGroups` for
   the All / column / card menu. Everything else is already generic:
   `ScopedActionChip` renders it, and `AiPassController` (`pass_controller.dart`)
   supplies `boardReason`, `columnReasons` (the per-column gray-out text) and the
   `maxContexts` budget gate, then `run` fires the confirm dialog and the POST.
   Nothing to write for the **story view**: `aiCardActions` (the
   `cardActionSlots` contribution in `ui/card_actions.dart`) walks
   `AiPass.columnOrder`, so a pass that claims a column also offers itself on
   every card in it, scoped to that card. It grays only on `boardReason` — the
   column's empty reason is not this chip's question.

## The workflow surface — where `orchestrator:` is edited

**Two editors over the block, never three** (`docs/WORKFLOW_SURFACE.md` is the
as-built audit). Board Settings → Orchestrator has a **Pipeline / Form** toggle:

- **Pipeline** (`workflow_view.dart`) — text and graph at once, plus a per-pass
  pane. `orchestrator_yaml_pane.dart` (YAML + marker gutter),
  `workflow_canvas.dart` (columns are nodes, passes are edges — drag an outcome
  chip or edge label onto a column), `pass_inspector.dart` (that pass's
  **routes**, framing, model/effort ladder, skills, scope). Owns **all topology**
  plus `framings`, `gt_tags`, `gt_areas`, `gt_promote_after`, and `exclude_tags`
  (board-wide, but edited on every pass's Scope panel — the form has no second
  door onto it).
- **Form** (`orchestrator_settings.dart`) — only what a graph has no shape for:
  General (project), Runner limits (max workers/contexts,
  `apply_fail_after`, the three wind-down thresholds — a runner-level gate over
  every pass that *engages* garbage time, which is why it is not a pass knob),
  Garbage time (tag vocabulary + idle), Completion obligations, Skills.
  **Retired 2026-08-10**: `mode` (nothing ever read it); `pr_branch_prefix`,
  `pr_base_branch`, `video_cleanup_wait_days` and `scene_job_timeout_min` moved
  to the **Project** (`projects.yaml`, Project Connections) — repo facts that
  sit beside the delivery/recording/scene-test toggles gating them.
  **Retired 2026-08-11**: `enabled` — see below. A board.yaml still carrying any
  of them warns in the code pane and drops it on save.

**There is no per-board on/off switch.** A board never had a reason to hide its
AI controls: enabling the plugin is the off switch, and an unwired board explains
itself. `AiPassController.shows` is `aiEnabled && board.path.isEmpty` (server-
backed), so every server board renders the upper-right pass controls immediately
— grayed, each hovering `pass.notConfigured` plus its first `configIssues`
sentence. Per-column chips still appear only over role-bound columns, so the
board fills in as the pipeline is wired. The server half is the same sentence:
`POST …/runner/{start,loop,work,tick}` 400s on unbound work roles **at every
scope** (`validateWork` in `api.dart`), which is what keeps an unwired board from
going resident against `OrchestratorPolicy.canonical`'s label vocabulary and
spending tokens on columns nobody bound.

Both are views of one `OrchestratorConfig`; switching is a round trip through
`_collect()`/`_applyConfig()`. `_collect()` is written field by field and takes
everything the form no longer renders straight off `_carried` — **not** a
`copyWith` (nullable knobs must be clearable). The regression that guards this is
`orchestrator_yaml_pane_test.dart` → *"nothing is lost when the form gives way"*.

Two roles the canvas structurally cannot bind, so the inspector's Routes is
editable and the legend lists **every** pass (unbound ones dimmed):
**`input`** (where an edge *starts* — a board with nothing bound has no port to
drag) and **`inflight`** (a board-wide conflict scope no card travels).

**Adding a knob:** a column role is a `PassRole` on the descriptor and nothing
else — no UI edit. Any other knob needs a form (or inspector) field **and** its
`BoardConfigJson` mapping both ways, in the same commit.

## Garbage time = TWO things (S10 — PORT from godototype, not new)

Mostly a packaging port of `manager.mjs` logic; line refs in
`docs/ORCHESTRATOR_PASSES_PORT.md`. Don't re-derive — port.

1. **General garbage time (S10a)** — **LIVE (S10 Commit C).** A **button**
   (manual pass — the runner never starts one):
   `garbage_time_executor.runGarbageTimePass` (mirrors `runUnblockPass`) over `POST …/runner/garbage-time`; `OrchestratorRunner.
   garbageTime` + the `garbageTiming` snapshot flag; the `garbage-time-header`
   `ScopedActionChip` (All/column/card scope). Configured on columns: bound
   **input column(s)** + **output column**, plus the **`gtTags` + `gtAreas`**
   filters. Works the **high-effort** background tasks (wiki/audit-type work is
   configured here — there is no separate "Audit pass"). Done → output column, routed through
   `applyWorkerResult` (so S11 obligations fire); 400 + zero spend on misconfig.
2. **Wind-down garbage time (S10b)** — **LIVE (S10 Commit B).** NOT a pass; a
   **shared runner-loop step** (`OrchestratorRunner._windDownGarbageTime`) the
   resident run inherits — wired once in `_loop`'s `applied` wind-down branch. Arms on **any** wind-down: the session lever (0.95/0.90) OR
   the weekly lever (0.98). (Weekly is a different monitor with a higher trip
   point, not an exemption — user reversed the doc's "session only" on 2026-06-29;
   `applyRateEvent` preserves the window kind through warnings so the weekly 0.98
   threshold actually governs.) The run reaches a stopping point (the triggering round already
   routed through `applyWorkerResult`), then spends remaining tokens on
   **low-effort** gc pickups (`pickGarbageTimeStory` restricted to `effort: low`,
   done → `gtOutputColumns` via the choke point) until a round rate-limits or no
   low-effort story remains. A low-effort story that fails to finish in-window
   `gtPromoteAfter` (default 2) times is **promoted** low→high: just bump
   `windDownAttempts` (persisted in frontmatter) and, at the threshold, flip
   `effort:` low→high + a `promoted` marker — **mark it and stop**. Do NOT route,
   hand back, or resume promoted/large tasks here; whatever claims that work picks
   them up later. The promotion counter is the **one genuinely new** piece (not in
   godototype; model on the DLQ/split-on-timeout flow). Everything else ports.

**Shared mechanics (port verbatim):**
- **Effort is a field, not a tag.** `storyEffortRank` (`kaizend_shared/story.dart`)
  reads the card's **`effort:`** field — `low`=0, `high`=2, anything else
  (including unset) 1. **Lower = fewer tokens.** The old `size: small|large` tag
  is gone root-to-stem; so is the `model:` tag, replaced by the **`context:`**
  field (`haiku`/`sonnet`/`opus`). Never reintroduce either as a tag.
- **Tags that remain structural:** `garbage-time` and `endless` (perpetual;
  **`depends_on: []`** always; picker bypasses dep checks for it) — and both are
  **configurable per board** (`OrchestratorConfig.garbageTimeTag` / `endlessTag`,
  read via `gcTagOrDefault` / `endlessTagOrDefault`). Don't hardcode the strings.
- **`pickGarbageTimeStory`** sort: `effortRank ASC → endless ASC → priority ASC`.
  Used two ways: S10a with the tag/area filter (large work); S10b restricted to
  `effort: low`.
- **`pickWorkableStory`** (normal work) EXCLUDES `garbage-time` AND `endless`.
- **Thresholds** (`runner_state.dart`): rate-limit-type-aware (weekly→0.98;
  session→0.95 single/0.90 multi) + **buffer-scaled** in the final session before a
  deadline (`bufferScale`). **Both** branches arm the S10b drain — the type only
  picks the threshold (2026-06-29 reversal of the doc's "session only"); the window
  kind survives warning events so weekly's 0.98 actually governs a weekly window.
- **Wind-down is the SOLE token-survival lever. Preempt was deliberately REMOVED**
  (S10 Commit A, 2026-06-29): it sat *below* wind-down and was checked before the
  pick, silently masking every wind-down tweak. Preempt IS in the godototype
  source but kaizend does **not** port it — do NOT re-add it, and ignore the port
  doc's preempt / haiku-gc-round text (godototype-source-only). Memory
  `preempt-removed-single-winddown-lever`.

## Post-completion obligations (S11) — DO NOT LOSE THIS

After a card completes, config-driven `completionObligations` auto-create
follow-up cards (e.g. docs/audit follow-ups). **Fired from the single choke point
`status_writer.applyWorkerResult` (S2)** so EVERY pass that completes a card
inherits obligations for free.

**Rule for any new mode:** route completions through `applyWorkerResult`, NEVER
move a card to a done column directly — that's how a new pass inherits obligations
and (eventually) other completion-time behaviour. Obligations and Audit findings
share one `BoardStore.createCard` primitive. Follow-up cards use non-self-blocking
deps (mirror the endless rule).

## Stop-time + token buffer (S12) — LIVE

Per-run controls with **no menu surface of their own** (user constraint — no new
header menus/buttons): a **force-stop date/time** (caps the run), a **token
buffer %** (port of godototype `start -t -b`), and the **aggressive-idle**
override. They live in ONE `RunCapsSection` (`run_caps_menu.dart`) rendered as
the RUN CAPS section of the `⋮` beside every chip that can start a run — the
pill, a column's Work chip, a card's Work chip — so the body click **engages
what is set** rather than stopping to ask. `RunnerStartDialog`
(`runner_start_dialog.dart`) still exists as `AiPassController._startRun`'s
defensive fallback for a caller that hands over no caps; no surface routinely
reaches it. Pre-filled from the board's `deadline`/`tokenBuffer` config; leaving
both off → an open-ended run. The one-shot pass confirms are NOT touched —
force-stop is deliberately not threaded through their executors (enforcement
lives on the session-eating runner loop only).

How it works (the math + state already came over in S10 Commit B — S12 feeds +
applies them):
- **Plumbing:** the section's values ride `POST …/runner/start` (absolute
  ISO `shutdownAt` + int `tokenBufferPct`) → folded into `RunnerState` via
  `withRunCaps` at run-start (survive a restart). Absent → board config defaults.
- **Deadline:** the loop checks `RunnerState.pastShutdown` **between** rounds — never
  a mid-round kill — then stops cleanly. The pending stop rides the snapshot
  (`shutdownAt`); the chip shows "stopping HH:mm".
- **Token buffer:** `bufferScale` (in `winddownThreshold`) now has its inputs fed
  per-run, scaling the wind-down threshold in the final <5h window before
  `shutdownAt`; clamped 0–100; no-op without an end time.
- **`resolveDeadline`** (a `deadline` string → absolute instant: ISO / `HH:mm`
  today-else-tomorrow / `5h`,`90m`) lives in **`package:kaizend_shared`** so the
  server default-application and the dialog pre-fill share one parse. No new config
  knobs — `deadline`/`tokenBuffer` already round-trip; only their Settings labels
  were sharpened (token buffer = **% 0–100**).
- The rate-window sleep buffer is a fixed 60s `_resetBuffer` — it must NOT read
  `config.tokenBuffer` (which is now the token-buffer **percentage**, not seconds).

## Projects own the worker knobs; areas bind a card to a project

A **board** carries only board policy: column roles, tag scoping, obligations,
context caps, wind-down. Every **worker/repo** knob — `repo_path`,
`build_command`, `rules`, default `model`/`effort`, round timeout, the per-card
spend budget (`round_budget_usd`) and its `max_rounds` backstop,
ntfy topics, `create_prs`, `record_videos`, `scene_tests`, deadline/token buffer
— lives on a **Project** in `dev-boards/projects.yaml`, written by the server
(App Settings → Project Connections).

A card does **not** carry `project:`. It carries `area:`, and the **area** names
the project (`areas:` in `app.yaml`). So a card's project is resolved
area → project at pick time; changing an area's project re-points every card in
it. The machine-local repo override is out-of-band in
`.orchestrator/projects.local.json` (gitignored) — never in `projects.yaml`.

**There are three repo paths and they describe two machines** (`ProjectEditorView`,
`plugins/kaizend-plugin-ai/lib/src/ui/project_connections.dart`). The shared
`repo_path` in `projects.yaml` is the fallback every machine falls back to. The
**server host's** override is the out-of-band one above, reached over
`GET/PUT /v1/projects/{id}/local-config` and applied by
`ProjectStore.effectiveRepoPath` (`KAIZEND_REPO_PATH` in the *server's*
environment beats both) — that is the path the runner actually opens. **This
device's** checkout is `ClientPrefs.projectRepoPaths` in the host's
`client.json`: never sent to the server, never read by the runner. Server and
client are roles a machine takes, not hosts, which is why the middle field is
labelled "on the server host" rather than "this machine" — edited from a laptop,
the old label silently re-pointed the runner at a directory that exists only on
the laptop. Where a given project's work runs is a third per-device fact,
`ClientPrefs.executionSites`, remembered at launch rather than written on the
card: the same card worked from two machines answers "where" differently.

`area:` and `epic:` are **different fields with different owners**: `area` is the
2026-07-11 rename of the old epic-as-grouping, `epic:` is a real cross-board
entity in `app.yaml` with a `completed:`/`archived:` lifecycle. Never cross-read
one for the other.

**An area names the package that owns the code** (`area-taxonomy-2026-08`). The
old `Kaizend` / `Kaizend Plugin` / `Context Service` catch-alls are gone — they
had absorbed the whole backlog with the same epic spanning two of them, so epics
were doing the area's job and a subsystem lost its home the moment its build-out
epic finished. The registry now mirrors the repo one-for-one: **Kaizend Server**
(board store, API, auth/users, projects, annotations, attachments, skill vault),
**Kaizend Orchestrator** (`lib/orchestrator` — runner, passes, token economics,
PR flow, jobs), **Kaizend App**, **Kaizend Shared**, **Kaizend Plugin API**, one
area per plugin package (**AI / Terminal / Spellcheck / Analytics / User / Poker
Plugin**), and **Kaizend Housekeeping**. Godototype keeps **Event** (the 2026
rename of `Effect`, matching the code), Ragdoll, Object, Camera, Input,
Diagnostics, Godototype Plugin, Godototype Housekeeping.

Two consequences worth knowing before touching a board:

- **`Homelab` is an area the runner is told to skip.** Physical machines and
  wiring stay on `blorkfield` like everything else; what keeps them off the
  runner is `exclude_areas: [Homelab]` in its `orchestrator:` block, alongside
  the `homelab` tag in `exclude_tags`. There is deliberately no separate board:
  one was tried and removed once `exclude_areas` existed to do the same job
  without a board switch. The word "Server" no longer names an area at all:
  backend work is `Kaizend Server`. The old `server` tag meant both, and
  `blorkfield`'s `exclude_tags: [server, …]` — written to protect the hardware
  chores — was hiding every kaizend-server card from the work pass.
- **There are two standing negative filters, `exclude_tags` and
  `exclude_areas`.** Both sit on `orchestrator:`, and both reach the work pass
  only. `blorkfield` carries `exclude_tags: [homelab]` beside
  `exclude_areas: [Homelab]` — the pair that holds the hardware chores off the
  runner. `exclude_areas` (`OrchestratorConfig.excludeAreas`,
  `orchestrator/decision.dart`) landed with the card that used to be named here
  as its tracker, `a-board-can-hold-a-pass-off-a-whole-area`, and it is what let
  the tag list shrink: `godototype` came off entirely once
  `blorkfield-s-runner-stops-fencing-off-godototype-work` decided that work is
  the runner's to pick up. The marker tag is still applied **from the area**
  (`godototypeMarkerTag`, `migrations/area_taxonomy_map.dart`) rather than left
  to whoever typed the tags — which is what closed the hole a card like
  `Tie Effect to Event` used to walk through — but it is now a label, not a
  fence.

There is **one** `app.yaml`, server-owned and served over `GET/PUT
/v1/app-config` (areas, epics, plugin enablement, `skill_sync_targets`, AI master
switch). The area textures it references are served from it too —
`GET /v1/area-textures/{name}` over `<boardsDir>/areas/`, with a strong
ETag so a client caches them and revalidates for a 304; App Settings
uploads a picked texture to the server rather than importing it into a
device folder a remote client does not have.

## A round's agent — card, then project, then install

Which agent a round launches through resolves in this order: the card's `agent` annotation
(`annotations/agent`, an ordered `AgentProfile.id` list — first runnable wins, see `chooseCardAgent` in
`orchestrator/project_config.dart`) -> the project's `agent:` -> `default_agent:` in app.yaml
(`AppConfig.agentFor`) -> `claude`. The card step is the only one that can refuse: if every id on its list is
unknown, `driver: unsupported`, or rate-exhausted, the card parks blocked (`noRunnableAgentQuestion`) instead of
falling through to the project agent. An empty list skips the step.

## Attribution — who wrote a card, and what wrote it

`Attribution` (`kaizend_shared/story.dart`) parses `created_by:` as
`id (context)` — e.g. `owner (opus)`. The two halves have different sources and
only one is derivable by the server:

- **who** comes from the token → an identity in `dev-boards/users.yaml` (server-
  owned, written via `/v1/users`; an identity a card references is **disabled**,
  never deleted, so old attribution still resolves).
- **what wrote it** cannot be derived — an agent holding a user's token looks
  exactly like that user. It is self-declared on the **`X-Kaizend-Context`**
  request header, and must be one of `cardContexts`
  (`haiku`/`sonnet`/`opus`, `card_vocabulary.dart`); anything else is a
  deliberate 400 so a typo can't split one writer into two. Unsent, the card
  reads as if a human typed it.

Send it on every card you create. See the **story-discipline** skill.

## The Skill Vault

The vault is app-level, not board-level: `dev-boards/.skills/<id>/SKILL.md`
(human-editable content) plus `vault.yaml` (**app-managed** — `enabled`, source,
timestamps; change it through `/v1/skills*`, not by hand). `SkillStore` has no
cache, so editing a `SKILL.md` on disk is safe; flipping `enabled` is not.

Three consumers, one writer (`writeSkillFolder`, now in
`kaizend_shared/skill_delivery.dart`):

- **automated** — `materializeSkills` copies the effective set into each fresh
  worktree's `.claude/skills/` (`WorktreeSession.create`).
- **interactive, on the server's host** — `SkillSync.sync` writes the
  globally-enabled set into every target in `app.yaml`'s `skill_sync_targets:`,
  removes what it previously wrote and is no longer enabled (scoped to
  `<boardsDir>/.skills/synced.yaml`, keyed by root *and* format, so hand-written
  skills — and a second format's — survive), and rewrites the **skill mandate
  block** in that target's instruction file. Runs on server start, after every
  `/v1/skills*` write, and after an app-config save that adds a target.
- **interactive, on a client device** — `DeviceSkillSync`
  (`kaizend-app/lib/data/device_skill_sync.dart`) does the identical thing over
  **`ClientPrefs.skillSyncTargets`**, this device's own roots in `client.json`,
  with its own manifest at `<configDir>/kaizend/skills-synced.yaml`. Runs on
  connect and after any vault write the app observes at the plugin request seam
  (any non-GET under `/v1/skills*`, read off the route so the host names no
  plugin). It reads `GET /v1/skills?body=1` — the plain listing blanks bodies,
  and a folder written from that would be a convincing-looking empty `SKILL.md`,
  so a server that can't serve bodies has the whole sync refused rather than
  half-applied.

**`app.yaml`'s roots are the server host's; a device's are its own, and neither
writes the other's.** `SkillSync` runs inside the server process, so its paths
only mean something on that machine; a remote client writing them would at best
manufacture stubs. That is the whole reason there are two lists rather than one.

Both are the same engine: **`SkillDelivery`** in `kaizend_shared` — folder
writes, the mandate block, the child-checkout expansion, the stub guard, removal
scoped to `synced.yaml`. A deliverer supplies only its `manifestPath` and its
`relativeRootBase` (the boards dir on the server, `$HOME` on a device).
`SkillSync` is a subclass of it, not a copy of it, and the format adapters
(`kaizend_shared/skill_format.dart`) are shared for the same reason — the app
cannot depend on `kaizend-server`, and a near-copy of "where does
`.claude/skills` go" is exactly the vendor drift the adapters exist to prevent.

A target is a **root plus a format** (`{root: ~/repo, format: claude}`), never a
skills directory: the format adapter owns every path under the root — `claude` →
`.claude/skills` + `CLAUDE.md` (at `$HOME`, `~/.claude/CLAUDE.md`), `agents` →
`.agents/skills` + `AGENTS.md`. A new assistant is a new adapter, nothing else;
an unknown format is skipped and logged. The pre-adapter bare-string form is not
read, on either side.

**Where kaizend owns the launch, it stops asking.** Sync and materialization are
both *persuasion* — folders on disk plus a mandate block saying they are house
rules — and a context that ignored them looks exactly like one that complied.
`SkillLauncher` (`kaizend-server/lib/skill_launch.dart`) instead resolves the
effective set (globally enabled ∩ the board's `enabled_skill_ids`), writes it
into a vault-owned bundle, and hands back the argv that loads *that* bundle and
makes every other vault skill unreachable (`SkillFormat.launchPlan`). Every
orchestrator pass launches this way, and so does the terminal plugin's card
launch, over **`GET /v1/skills/launch`** (`?board=`, `?format=`) — the one launch
whose command line is the user's own `cardOpenCommand`, so the server answers
with flags to splice in (`composeSkillLaunch`) rather than composing it.

**The bundle travels.** The launched context runs on the *client's* machine,
which may share no filesystem with the server, so the response carries the
bundle's `files` plus a `bundleToken` marking where in `args` the client
substitutes the directory it wrote them to (`resolveSkillLaunch`,
`terminal_skills.dart`). Answering with a server-side path was the bug: on a
client that path resolved to nothing while the accompanying suppression flags
still switched off every skill the client's own checkout had, so a remote card
context ran with **no** vault skills under a prompt calling them house rules. A
client that cannot write the bundle therefore injects **nothing** — never half
the answer. The bundle is rebuilt on every launch, so it can never be stale,
which is the point: stale is exactly how a disabled skill would survive.

The vault also **proposes**: `POST /v1/skills/distill` (`skill_distill.dart`)
reads the board corpus and answers with the skills it says are missing — cards
per area against enabled-skill coverage, tags recurring across unrelated areas,
epics no skill names, repeated `touches` path prefixes — each proposal carrying
a drafted body plus the evidence it came from. It **writes nothing**; creating
the skill stays the user's own act through `POST /v1/skills`.

An enabled skill is therefore **binding**, not a suggestion — the mandate block
names it and says it is consulted before the work. That is why a stale enabled
skill is worse than no skill, and why anything that no longer applies gets
`enabled: false` rather than being left to mislead. Full as-built spec:
`docs/SKILL_VAULT_PORT.md` (V1–V12, closed; distillation, the travelling launch
bundle and the client-side deliverer recorded under "Beyond the port").

## Invariants & gotchas

- **Derived picker roles (2026-07-03 overnight-idle fixes,
  `OrchestratorPolicy.fromConfig`):** the work pick pool is input ∪ progress
  columns (a needs-round continuation must survive a runner restart); unset
  `inflightColumns` defaults to progress ∪ failure ∪ output; `manual` is a
  structural exclusion like `garbage-time`/`endless` (an empty board
  excludeTags must not admit it). **Dep-satisfaction is `depDoneColumns`**
  (work ∪ gt ∪ pr-update outputs, plus any `auto_done` column via
  `depDoneColumnKeys`) — shared by the work AND gc pickers — while RESULT
  routing keeps the singular `workOutputColumns` (PR flow: done = PR published,
  waits in review). Never check deps against the routing column alone: cards
  march review → completed → archive and their dependents must stay workable.
- **A card's escalation budget is denominated in dollars, not in rounds**
  (2026-09-07). `Project.roundBudgetUsd` (`round_budget_usd` in
  `projects.yaml`) is what the runner's guard reads — `max_rounds` survives
  only as the runaway backstop behind it, and both live projects now sit at
  `round_budget_usd: 15.0` / `max_rounds: 25`. A round count was never the
  budget: rounds on blorkfield have cost between $0.62 and $12.21, so a flat
  `max_rounds: 3` bought anywhere from ~$2 to ~$37 of attempts depending on
  which three rounds a card drew. The spend is read off the **cost ledger**
  (`CostLedger.storySpendUsd`) and scoped to the story's current *attempt* —
  any terminal round (`done`/`done-verify`/`blocked`/error) zeroes it, so a
  card a human puts back in play starts on a fresh budget, and the guard itself
  appends a zero-cost `blocked` record to close the attempt it refused. Checked
  **between** rounds, never mid-round. A card overrides the project default
  through the **`budget` annotation namespace** (`PUT …/annotations/budget/{id}`,
  a dollar amount), exactly the way `context`/`effort` overlay model/effort;
  nonsense or a non-positive value reads as no override, never as a zero budget.
- **`done-verify` means "finished, but look at it"; `blocked` means "I cannot
  proceed".** These were one status until 2026-08-25, and the overload parked
  completed work in the failure column beside genuinely stuck cards. A worker
  whose work is COMPLETE and merely wants eyes returns `done-verify` — a
  completion: it keeps its diff, fires obligations, records its run, and carries
  an optional `reason:` written onto the card as `## Needs your eyes`. `blocked`
  is now only for a question a human alone can answer. Neither is `needs-round`:
  waiting on a human that way burns the card's spend budget until the guard
  force-blocks it anyway.
- **The dial decides where a completion lands**, not the worker alone.
  `PassAutonomy` (`autonomy:` under `orchestrator:`; four stops —
  `always_verify` / `worker_decides` / `trusted` / `never_verify`) crossed with
  the worker's verdict, in `routeCompletion` at the `applyWorkerResult` seam.
  Defaults are computed, never written to disk: verify column unbound →
  `never_verify` whatever the file says; bound with `autonomy:` unset →
  `always_verify` (binding the column is the act of asking for the gate).
  Two board-owned rule lists sit either side of the dial, both lists of
  `CardFilter` matched at the same seam: **`work_verify_when:`** upgrades a
  plain `done` to verify — and only under `worker_decides`, which is the whole
  difference between that stop and `trusted` — while **`work_verify_never:`**
  routes a match straight to the output column at **every** stop, overriding
  even the worker's own `done-verify` (logged, the way `never_verify`'s
  override already is). Read the pair as sign and strength: a `when` rule can
  only ADD a gate and only sometimes; a `never` rule REMOVES one, always. That
  asymmetry is deliberate — a class of card with nothing to review (a `spike`,
  whose deliverable is the cards it filed) cannot be expressed by a list that
  only ever adds. Both are edited in the pass inspector, and an empty
  `CardFilter` in either list is an unfinished draft, skipped, never a wildcard.
- **A worker is expected to decide, not to escalate.** The contract in
  `worker.dart` names the cases that look like human decisions and are not:
  blessing a golden/fixture whose diff IS the card's stated deliverable; a
  pre-existing red the diff did not cause; visible board bookkeeping; a choice
  between defensible implementations. Blocking on one of those is a defect, not
  caution. Which work deserves eyes IS tunable — `FramingRegistry.workVerify`
  (`framings: work-verify:`), editable in the pass inspector.
- **Rounds share one worktree and one resumed session** per story, not a cold
  start each round; apply-back is deferred to disposal, and a `blocked` card
  **keeps** its edits. Don't regress either.
- **ntfy is detached, and its ASCII rule still binds whoever reattaches it.**
  `notify.dart` and `listener.dart` are intact and still correct — Dart's
  HttpClient throws on non-ASCII header values, so `NtfyClient.notify` sanitizes
  the Title (the "·" default title silently killed every alert on 2026-07-03) —
  but nothing calls them: the runner logs where it used to push
  (`ntfy-comes-off-the-wire-until-there-is-a-phone-to-put-it-back-on`, 2026-09).
  The topic was going to a phone nobody was carrying. Reattaching it is the other
  half of the **Mobile Notifications** epic, so treat both files as a live
  contract with no consumer — don't delete them, and don't describe an alert as
  something the runner sends today.
- **A running pass locks ALL of its configured columns** (input, progress,
  output, failure, …) via `OrchestratorConfig.passColumnRoles` — the per-pass
  role registry `ProcessingLock.from` derives from (2026-07-03; Work previously
  locked only its input columns). **Adding a column knob to a pass? Add it to
  that pass's `passColumnRoles` set in the same commit as the field** — the
  lock (server 409s + client graying, both via the one shared factory) then
  inherits it without revisiting the lock code. A new pass adds a registry
  entry + its snapshot flag branch in `ProcessingLock.from`.
- **Machine-local repo path — per *project*, not per board.** The orchestrator's
  effective `repoPath` is `ProjectStore.effectiveRepoPath`:
  `KAIZEND_REPO_PATH` in the server's env > the server host's override in
  `.orchestrator/projects.local.json` (gitignored; edited through the project
  editor's "Repository path — on the server host" field →
  `GET/PUT /v1/projects/{id}/local-config`) > the project's shared `repo_path`.
  The **board**-level form this invariant used to name
  (`.orchestrator/<board>.local.json` and `/v1/boards/{board}/local-config`,
  2026-07-03) came off in the legacy AI config teardown — the repo moved onto
  the referenced Project with every other worker knob. `BoardStore.readEffectiveConfig`
  survives as the documented seam every orchestrator consumer reads, now a
  passthrough to `readConfig`; keep reading it rather than `readConfig` so a
  future board-level effective layer has one place to land, and so the raw
  `readConfig` stays reserved for the Settings round-trip.
- **Config knobs need the wire DTO both ways** or server boards silently drop them
  on save — add the `BoardConfigJson` mapping in the same commit (memory
  `server-board-config-knob-wire-dto`).
- **Empty columns need `.gitkeep`** (columns are folders); server auto-drops it on
  move/delete (memory `empty-board-columns-need-gitkeep`).
- **Verify locally with zero tokens:** point `KAIZEND_BOARDS_DIR` at a throwaway
  boards dir and put a harmless `claude` shim first on `PATH` — a pass then runs
  its whole gate/lock/route path against a real server and spends nothing.
- **Never do git writes without explicit per-action permission** (memory
  `ask-before-git-writes`) — this governs Claude-the-assistant acting interactively
  in a session. It does **not** govern the configured PR-flow pass: when a board
  has `createPrs` enabled, **the orchestrator owns git on the target repo's
  `automation/*` branch namespace**; humans own git everywhere else
  (`docs/PR_PASS_PORT.md`, reversing the earlier S9 "git is human-owned" verdict).
- Gate every token-spending pass behind config + scope validation: misconfig →
  error report, no worker spawned.
- **PR-flow invariants** (`docs/PR_PASS_PORT.md`): never `applyBack` a `pr_url`
  card — it's Update-PRs rework territory, not Unblock's (`unblock.dart` still
  excludes `pr_url` cards from its scope, and that exclusion now guards a
  human-pressed button rather than an automatic priming step). Videos and `REVIEW.md` are **overwritten in
  place** on every branch change (sync or rework), never accumulated — they live
  only in the target repo's local `dev/<storyid>/` and are never committed or
  uploaded, so the target repo's `dev/` **must be gitignored** (`PrGitOps.preflight`
  warns when it isn't).
