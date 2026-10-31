---
name: story-discipline
description: "How this shop turns a request into work: every task the user gives you becomes a board card BEFORE the work starts, written with title/area/tags and always both context: and effort:, created through POST /v1/boards/{board}/cards with your own X-Kaizend-Context header, and edited through the API rather than the file on disk. Also covers the way out: every card a session works lands in a column and carries a ## Handoff log entry before you report back. A card whose done-line came true goes to 05_TESTING/06_DONE; one that did not has exactly three exits, and you pick it, never the user — (1) the rest of the work lives in other cards, so depends_on them and back to 01_TODO; (2) 08_FAILED with a concrete retry plan a fresh context can run; (3) it truly cannot be done or only the user can decide, so tell them the real reason (08_FAILED or 03_STUCK with question-shaped blockers). Never close an unmet card as 'answered' or 'premise refuted', and never hand the column choice back. A finding that names work becomes its own card, and an unmeetable acceptance criterion is itself a finding to file. Not a keyword skill — it applies to every task, in every repo."
---

# Story Discipline

The board is the record. A conversation is not — it scrolls away, and nothing about it survives into the next context. So anything the user asks for becomes a card **before** the work starts, in every repo, on every board.

This is not a keyword skill. It applies to every task you are given.

## Server connection

The kaizend server is accessed through the `$KAIZEND_SERVER_URL` environment variable. **Never type a literal port or host** (`localhost:8080`, `localhost:8420`, etc.) in any command, curl example, or script. If you encounter a "connection refused" error, the first diagnostic step is to verify that `$KAIZEND_SERVER_URL` is set correctly in your environment:

```bash
echo $KAIZEND_SERVER_URL
```

If `$KAIZEND_SERVER_URL` is empty or wrong, that is the root cause — not a server outage. Set it correctly before retrying any API call. Every example in this skill uses `$KAIZEND_SERVER_URL`; none of them name a port.

## Work becomes a story first

1. The user asks for something.
2. You file it as a card on the board that owns that repo.
3. Then you do the work, and move the card as it moves.

Small enough to be "just a quick fix" is not an exemption — that is exactly the work that disappears. The only things that skip the board are questions you are answering in place and changes the user is actively watching you make on an existing card.

## A pasted story is a work order

When the user drops a card's contents into the chat with no other instruction,
that **is** the instruction: work it. Do not ask what they want done with it, do
not summarise it back, and do not treat it as context for a conversation. The
card already says what done looks like; the paste says "start".

The rule this replaces is the one that keeps costing a round: reading the story,
answering "what would you like me to do?", and waiting. Pasting a story is the
user handing over a work order, and the only correct first move is
`terminal-cost.py start` against that card followed by the work itself.

## Every card you work lands in a column, with a handoff log

**This binds every card a session touches, however the session got hold of it** — pasted into the
chat, named by id ("begin T4.2"), picked off the top of a column, or carried over from earlier work.
The entry rule at the top of this skill has an exit half, and this is it. When the work stops — for
any reason, including the window closing on you — the card moves through the API and carries a
handoff log, **before** you report back in chat:

**Did the card's done-line come true?** Answer that first, against the card's own Description and
Tasks, and only then pick a column.

If yes:

| Outcome | Column |
|---|---|
| Every task done and the evidence is in hand (suite green, behaviour verified) | `05_TESTING` — the human verification gate is the user's, not yours to skip |
| Verified by the user in this session, or the card carries no gate | `06_DONE` |

If no, the card has **exactly three exits**. Decide which one it is yourself; it is never the
user's job to sort it:

| The card could not finish because… | What you do |
|---|---|
| **1. The rest of the work is somewhere else.** Another piece of work has to land first, or the work is bigger than the card. | File a card for each piece (or find the one that already exists), set this card's `depends_on` to them, rewrite its Tasks as what a fresh context does once they land, and put it back in `01_TODO`. It waits; it does not close. |
| **2. What was tried did not work, and there is a next thing to try.** | `08_FAILED`, with what was tried, what it measured, what is refuted, and **the concrete retry plan a fresh context can run**: which lever next, on which switch, gated how. A failed card with no retry plan is a dead end nobody can pick up. |
| **3. It cannot be done, or only the user can decide.** | Tell the user the real reason — the measured physical, structural or product fact that makes it impossible — in plain words, as the headline of your report. The card goes to `08_FAILED` carrying that reason (impossible) or to `03_STUCK` with question-shaped `## Blockers` (the user's decision). This is the only exit that spends the user's attention, so it has to be a reason, not a shrug. |

Never:

- **Close a card whose done-line did not come true.** "Answered, not fixed", "premise refuted",
  "closed at the user's request", "nothing to fix" — none is a way into `05_TESTING` or `06_DONE`.
  A refuted premise means you learned *why* the work is elsewhere (exit 1), what to try next
  (exit 2), or that it can't be done (exit 3). Pick one.
- **Hand the column decision back** — "move it back if you'd rather", "you can put it in TESTING
  instead". The user asked you to do the work, not to choose between your options for it.
- **Use `08_FAILED` as a parking lot** for a card whose next step is another card. That is exit 1:
  set `depends_on` so the card comes back on its own when the work it waits on lands.

Worked example: `a-walking-sole-keeps-room-under-its-toe-for-the-trunk-s-lean-moment` (2026-09-30).
Its first pass refuted the premise and filed the card that owned the real work, then parked itself
in `08_FAILED` with "next step: that card". A second session moved it to `06_DONE` "as answered, not
fixed" and told the user to move it back if they disagreed. Both were wrong. It was exit 1: its
question — room under the toe *once the trunk's lean moment reaches the floor* — could not be
measured until the pitch-budget card landed. Fixed by `depends_on` on that card, Tasks rewritten as
the rerun, back to `01_TODO`.

`03_STUCK` is a question, never a status. "Needs more investigation" is not a
blocker; it is you not having finished. The same bar `## Blockers` sets for a
runner round sets it for you — see the blockers rules above.

Never leave a worked card sitting in `02_DOING` while you report in chat. The
chat scrolls away; the column is the record of where the work actually landed.

```bash
curl -sS -X POST "$KAIZEND_SERVER_URL/v1/boards/blorkfield/cards/<id>/move" \
  -H "Authorization: Bearer $KAIZEND_TOKEN" -H 'Content-Type: application/json' \
  -d '{"toColumn": "05_TESTING"}'
```

### The handoff log

Write it in the same breath as the move — not as a separate chore you might get to, and never only
in the chat reply. **A card you did not finish is the case where it matters most**, because
somebody else is going to pick up exactly where you stopped, and everything they cannot recover from
the card they will pay to re-derive. A `03_STUCK` card with blockers but no handoff log asks a
question without saying what was already learned.

The heading is literally `## Handoff log`, at the bottom of the card body — the boards recognise it
as a section (`config.aiSections`), so an ad-hoc name like "Notes" or "Session summary" does not
render as one. Under it, one `###` entry per session, dated and attributed, **newest last: append,
never rewrite.** A previous session's entry is a record of what was true then; editing it destroys
the only copy.

```markdown
## Handoff log

Newest last. Append, don't rewrite.

### 2026-09-20 — first pass (opus, terminal session)
```

An entry earns its place by holding what the next session cannot get anywhere else:

- **What landed** — the commit sha and branch, and the files in it. A sha is worth a paragraph of
  description, and it is the thing chat scrollback loses first.
- **What in the working tree is *not* yours.** Uncommitted files you found and deliberately left
  alone, said plainly, so the next session does not assume they belong to this card or commit them
  by accident.
- **How to reproduce the measurement** — the exact command, with flags, and where the logs are. Not
  "ran the scenarios".
- **What was refuted, and by what numbers.** The most expensive thing a successor can do is retry an
  approach you already disproved. Say it was measured, not that it "didn't seem to work".
- **The ids of every card you filed**, so the trail reads in both directions (see "A finding that
  names work is a card, not a note").
- **What is carried out unanswered** — open questions that now belong to another card, each naming
  that card.
- **What the next step is, branched on each blocker's answer**, when you are parking in `03_STUCK`.
  The person answering a blocker should not also have to work out what their answer implies.

What does **not** go in it: a narrative of your session, a restatement of the card's Description, or
an explanation of why an acceptance criterion you missed did not really matter (that is a finding —
see "An acceptance criterion that cannot be met is a finding").

## How a story is written

- **Title** — a sentence about the finished state, not a task name. "An Attached File Lands Where You Can See It", not "fix attachment insert".
- **`area`** — the area that owns it. Required, never blank, and always a name that already exists (see below). The project rides on the area; cards carry no `project:`.
- **`epic`** — the epic it joins, when one applies. Also always an existing name.
- **`tags`** — what it touches.
- **`context:`** — `haiku` | `sonnet` | `opus`. The context the work needs.
- **`effort:`** — `low` | `medium` | `high`.
- **`depends_on:`** — ids that must land first.

`area`, `context:` and `effort:` are never omitted. Every card sets all three.

**A card whose acceptance requires calling the kaizend server API directly is
`manual`.** Skill-vault edits (`PUT /v1/skills/{id}`), `app-config` writes, or
anything else only reachable with a real `KAIZEND_TOKEN` in an interactive
session cannot be done by a headless orchestrator worker — workers carry no
standing board-API credentials by design. Without the tag, such a card still
gets picked up: the worker burns a full round discovering it has no network
access, then correctly parks itself STUCK with a Response block asking a human
to finish it — a round and its tokens spent to learn something the tag would
have said for free. `manual` is picker-keyed (`decision.dart` excludes it from
Work, `garbage_time.dart` skips it), so a `manual` card is never picked up
headlessly at all; it waits for a human or an interactive session with real
credentials, same as `garbage-time` and `endless` are picker-keyed for their
own reasons. Card
`a-gait-number-in-an-acceptance-criterion-says-which-commit-it-came-from` is
the worked example.

### Area and epic are looked up, not guessed

Areas and epics are two different registries and they do not share names. An
**epic** is a body of work with a start and a finish ("AI Enablement", "Human
Verification Gate"); an **area** is the standing part of the system that owns the
code ("Kaizend Server", "Camera", "Terminal Plugin"). Writing an epic's name into
`area:` is the failure mode this section exists to stop — it produces a card with
no valid area *and* no epic, which the pickers, the textures, and the project
resolution all silently drop.

The subtler failure is the one that actually happened, and it is why the areas
now mirror the repo one-for-one. `Kaizend` and `Kaizend Plugin` were allowed to
mean "somewhere in the kaizend monorepo", so they absorbed 366 cards between
them, with the same epic spanning both — nothing at the card level said which was
which. Epics ended up doing the area's job, so a subsystem lost its home the
moment its build-out epic finished. **An area names the package that owns the
code. If you can name the package, you can name the area:**

| Area | Owns |
|---|---|
| `Kaizend Server` | `packages/kaizend-server`: board store, API, auth/users, projects, annotations, attachments, skill vault |
| `Kaizend Orchestrator` | `kaizend-server/lib/orchestrator`: the runner, its passes, token economics, PR flow, jobs |
| `Kaizend App` | `packages/kaizend-app`: board view, columns, story view and modals, colorways, header chrome, search UI |
| `Kaizend Shared` | `packages/kaizend-shared`: DTOs, models, `board_yaml`, `search_query`, the pass descriptors |
| `Kaizend Plugin API` | `packages/kaizend-plugin-api`: the boundary, extension points, the registry |
| `AI Plugin` · `Terminal Plugin` · `Spellcheck Plugin` · `Analytics Plugin` · `User Plugin` · `Poker Plugin` | one per package under `packages/plugins/` |
| `Kaizend Housekeeping` | repo chores, board maintenance, the wiki cards |
| `Event` · `Ragdoll` · `Object` · `Camera` · `Input` · `Diagnostics` · `Godototype Plugin` · `Godototype Housekeeping` | the godototype repo |
| `Homelab` | physical machines, wiring, hardware — on `blorkfield` like everything else, held off the runner by `exclude_areas: [Homelab]` |

Two rules that fall out of that table:

- **"Server" is not an area.** Backend work is `Kaizend Server`; the machines in
  the basement are `Homelab`. The bare word used to mean both, and the tag of the
  same name — sitting in `blorkfield`'s `exclude_tags` to keep the runner off
  hardware chores — was hiding every kaizend-server card from the work pass. Do
  not reintroduce it in either field.
- **A tag must say something the area does not.** An area-shaped tag —
  `kaizend`, `orchestrator`, `diagnostics`, `ragdoll`, and the rest — repeats
  the area and carries nothing. The canonical list of these is code, not prose:
  `retiredTags` in
  `packages/kaizend-server/lib/migrations/area_taxonomy_map.dart` — grep it
  before typing a tag that might be area-shaped, since it is the one place this
  gets amended when a new area or epic makes another word redundant:

  ```bash
  grep -A15 '^const retiredTags' packages/kaizend-server/lib/migrations/area_taxonomy_map.dart
  ```

  What survives that check is the cross-cutting facet a tag is actually for —
  `bug`, `refactor`, `testing`, `docs`, `ux`, `spike`, `audit`, `wiki`,
  `automation`, `config`, `teardown`, `flutter`, `dart`, and the like. Four are
  structural and deliberately excluded from `retiredTags` even though they look
  area-shaped — never drop them on the same logic: `garbage-time`, `endless`,
  and `manual` are picker-keyed, and `godototype` is a label applied from the
  area via `godototypeMarkerTag`, not typed by hand (no longer a fence:
  `blorkfield`'s `exclude_tags` is `[homelab]` and the runner picks up
  godototype work).

So before you write either field, read the registries — never write a name from
memory:

```bash
curl -sS "$KAIZEND_SERVER_URL/v1/app-config" \
  -H "Authorization: Bearer $KAIZEND_TOKEN" | jq '{areas: [.areas[].name], epics: [.epics[] | select(.completed == null) | .name]}'
```

Then:

- Pick `area` from the **areas** list, copied character-for-character. If nothing
  in the list fits, the right move is to ask the user which area owns it — not to
  invent one, and not to leave it blank.
- Pick `epic` from the **epics** list only if the card genuinely belongs to that
  body of work. No epic is a normal, correct state; a wrong one is not. Skip
  epics carrying a `completed:` date.
- A card almost always has both a real area and, when it's part of a push,
  an epic. Check the sibling cards already in that column for the shape.

**An optional fourth field: `agent`.** Not frontmatter — an annotation namespace like `context:`/`effort:`, set with
`PUT $KAIZEND_SERVER_URL/v1/boards/<board>/annotations/agent/<id>` (`annotations/agent`). Its value is an ordered list of
`AgentProfile.id`s the card's rounds may run through; order is preference, the first runnable one wins. Empty or unset means
inherit the project's agent. Set it only when a card needs a specific agent — most cards leave it empty.

The failure mode: a card whose list names **only** agents that cannot run a round (unknown id, `driver: unsupported`, or
an exhausted rate window) **parks blocked** with a `## Blockers` question naming each id and why it was passed over. It never
falls back to the project default. Pick ids from the install's agent catalog (`agents:` in `/v1/app-config`), and list a
ready fallback after a preferred one if the card must run regardless.

Body is two sections, in this order:

```markdown
## Description

Why this exists and what "done" looks like. Prose, not a restatement of the title.

## Tasks
- [ ] One verifiable step
- [ ] Another
```

A new card goes to the **top** of its column — the server does this for you on create; don't fight it by writing an order yourself.

## Acceptance criteria nobody has to come back and ask about

Nothing checks this for you. A machine pass used to read a card and flag what it
could not answer without the user; it was deleted on 2026-09-05 because producing
questions is cheap and answering them is the part that was never happening. The
gate is now the moment you write the card. A criterion earns its place only if it
is:

- **Specific** — names the exact thing checked, not "it works" or "looks correct".
- **Measurable** — a number, a threshold, a log tag, or an explicit before/after.
- **Readable by automation, or explicitly a human check** — a purely visual
  criterion with no signal to read is a question for the user, not a criterion.
- **Not padding** — no criterion that restates another.

Good: `"dotnet build exits 0 with no errors after the rename."` ·
`"sf2- in the [SwingTarget] diag reads 0 on every entry."`
Bad: `"The fix works correctly."` · `"Performance is acceptable."` ·
`"Code is clean and readable."`

Read the card back before you create it and hunt for the things that would have
been flagged: an open scope boundary ("is X in or out?"), a missing criterion, a
visual-only criterion with no signal to read, `depends_on` pointing at an id that
does not exist, an ambiguous "done". Resolve them yourself — anything answerable
by reading the code, the project's RULES, or a decision already recorded is not a
question for the user and never was. State the problem, name a file or function,
say what done looks like at the code level, and set `context:`/`effort:`.

**What is genuinely left over is a question, not a caveat.** A card whose work
stops on something only a person can decide parks `blocked`, and a `blocked`
round writes a `## Blockers` section onto the card: one numbered `###` heading
per question, each followed by an `**A:**` slot the person fills in
(`status_writer.blockersSection`). That shape is the contract — it is what makes
a blocker answerable in one pass instead of a paragraph someone has to decode.
Write your own open questions the same way. Two things follow from it:

- **A blocker must be phrased as a question with an answer slot.** "Needs
  clarification on the API" is not one. "Should a 404 from the upstream retry or
  fail the round?" is.
- **Nothing re-attempts the card on its own.** Unblock is a button a person
  presses after writing the answers, never a step the runner takes
  (`the-runner-stops-unblocking-on-its-own`). An unanswered question sits there
  costing nothing — which is the point — so a question nobody can answer is a
  card that will never move.

## A finding that names work is a card, not a note

The same rule that opens this skill applies on the way out. Work you discover
while finishing a card is still work, and a card parked in a review column is not
where the board keeps it. **Before you move a card out of your hands, every
finding on it that names work someone should do becomes its own card.** You do
not wait to be asked, and you do not ask whether it is wanted — file it, and say
in your report which cards you filed.

The tell is grammatical. If a sentence in your findings, handoff log, or closing
report says what *should* happen next — "the next card should…", "worth a
follow-up", "someone will need to…", "out of scope here", "left alone for now",
"this ought to be split" — that sentence is a card that has not been written yet.
Prose describing future work is the one thing the board exists to replace.

Three kinds come up constantly, and all three are cards:

- **A blocked or rejected approach.** You tried something, measured it, and it
  did not work. The disproof belongs on the card you were working; the *next*
  thing to try belongs on a new one.
- **A prerequisite you hit and worked around.** "X and Y cannot both be satisfied
  until Z is split" is a card for Z, and usually the highest-value one you found,
  because it is what unblocked work is waiting on.
- **A defect you found while looking at something else.** It does not become
  yours to fix by having been noticed, but it does become yours to file.

What is *not* a card: an observation with no action behind it, a measurement that
only supports the card you are already on, or a restatement of a card that exists.
Check for that last one before creating — a duplicate is worse than a note.

Link the new card back with `depends_on:` when the parent genuinely blocks it,
and name the ids you created in the parent's handoff log so the trail reads in
both directions.

## An acceptance criterion that cannot be met is a finding

A criterion that turns out to be unreachable, to measure the wrong signal, or to
have never been true of any build is **not** a reason to quietly fail the card,
and not a line to explain away in the handoff log. It means the criterion is
wrong, and a wrong criterion outlives the card — it gets copied onto the next one
and fails that too.

So say so explicitly, in three places:

1. **On the card**, state which criterion, what it actually measures, and the
   evidence — including what a known-good build reads against it. "Never met by
   any build" is the strongest form of this and settles the question.
2. **As a new card**, to fix the criterion and whatever generated it. That is
   usually prose rather than code: a threshold table in a vault skill, a
   boilerplate acceptance block, a wiki page. Route it through the
   **skill-refinement** skill when the source is a skill.
3. **In your report to the user**, as a flag, not a footnote — this is a
   judgement call they may want to make differently.

Do not silently substitute a criterion you *can* meet. Failing a stated bar and
saying why is honest; moving the bar and reporting a pass is not.

## Endless stories

A card tagged `endless` never completes — it exists to make and keep making a
full pass at its scope (a running audit, a wiki that tracks the code). Two rules,
and they are the whole contract:

- **`depends_on:` is always `[]`.** An endless story that found work and added it
  to its own deps would block itself on its own findings. The picker bypasses dep
  checks for endless cards precisely so a mistake here can't wedge one, but the
  correct state is still `[]`.
- **Findings become new cards, never deps.** Record the finding in the report;
  if it deserves a fix, create a card for it and keep going.

Pair it with `garbage-time` (both tag names are per-board configurable) so it is
only ever worked with tokens that would otherwise go unspent, and give it
`effort: low` so wind-down can drain it.

## Repo-specific fields

A board's cards inherit whatever fields that repo's stories have always carried —
read a neighbouring card before inventing a shape. On **godototype** in
particular: `priority` (1–100 active, 100–899 backlog, 900+ idle-fallback only),
`touches`, `acceptance`, `test_scenarios`, and `euphoria_principle` (the specific
Euphoria principle a locomotion story serves, or `"N/A — infrastructure"` — never
blank). Its ids are prefixed by kind: `ORC` for orchestration/infra/tooling
(the default when unsure), `SF` for locomotion/ragdoll code, `HF` for work that
genuinely needs the user's hands or eyes.

## Sign your own work

Create through the API, with your own context on the header:

```bash
curl -sS -X POST "$KAIZEND_SERVER_URL/v1/boards/blorkfield/cards" \
  -H "Authorization: Bearer $KAIZEND_TOKEN" \
  -H "X-Kaizend-Context: opus" \
  -H 'Content-Type: application/json' \
  -d '{
    "column": "01_TODO",
    "title": "An Attached File Lands Where You Can See It",
    "fieldInputs": {
      "area": "Kaizend App",
      "epic": "Kaizend Search",
      "tags": "flutter, ux"
    },
    "body": "## Description\n\n...\n\n## Tasks\n- [ ] ...\n"
  }'
```

**`context:` and `effort:` do not go in `fieldInputs`.** They stopped being card
frontmatter when they moved into annotation namespaces, and `fieldInputs` drops
an unknown key silently — a card created with them there comes out unsized, which
is exactly what the pickers and the wind-down drain read. Set each one on the
card the create returned:

```bash
for ns in context effort; do
  curl -sS -X PUT "$KAIZEND_SERVER_URL/v1/boards/blorkfield/annotations/$ns/<id>" \
    -H "Authorization: Bearer $KAIZEND_TOKEN" -H 'Content-Type: application/json' \
    -d '{"set": "sonnet"}'   # effort takes low | medium | high
done
```

The stamp has two halves and the server can only derive one. Your token says *who* — and a worker holding an agent token looks exactly like the person who minted it. It cannot know *what wrote this*. That half is `X-Kaizend-Context`, self-declared, and unsent it defaults to nothing: the card then reads as if a person typed it by hand. That is how `story-attachment-insert-point` and `story-open-attachment-launcher` landed attributed to a bare `owner`.

Send it on every create. `haiku`, `sonnet`, or `opus` — your own model, not the card's `context:`. A value outside that vocabulary is a 400, on purpose, so a typo can't quietly split one writer into two.

## Edit through the API, never the file

A running server holds cards in cache and writes them back. A direct edit to the `.md` on disk gets clobbered, and frontmatter gets flattened on the way out.

```bash
curl -sS -X PATCH "$KAIZEND_SERVER_URL/v1/boards/blorkfield/cards/<id>" \
  -H "Authorization: Bearer $KAIZEND_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"fields": {"effort": "high"}}'
```

Move with `POST .../cards/<id>/move` (`{"toColumn": "05_TESTING"}`). Same rule: the API, not the folder.
