---
name: session-accounting
description: "How a terminal session leaves its bill on the story it worked: register the card with terminal-cost.py at the start, book the spend (tokens, turns, duration, cost) into the story's journal and the board's costs.jsonl at every checkpoint, and let the SessionEnd hook catch whatever is left. Proactive by contract — the user closes the window when the work is done and never asks for this. Applies to every terminal session that works a kaizend board card."
---

# Session Accounting

A headless round leaves its bill on the board without being asked. It writes a
`CostRecord` into `<boardsDir>/.orchestrator/<board>.costs.jsonl` — tokens,
turns, duration, cost, status — and a line into that story's journal at
`<boardsDir>/.orchestrator/journal/<board>/<story>.log`, which is what the app's
story-journal view reads.

A terminal session working the same card used to write nothing, so the board's
record of what a story cost was only ever the half a worker did. **You close that
half yourself, and you do it without being asked** — the user closes the window
when the work is done and is never the one who remembers.

Everything here rides one script:

```
packages/kaizend-server/scripts/terminal-cost.py
```

It reads this session's own transcript (`~/.claude/projects/<slug>/<session>.jsonl`),
sums the usage the API actually reported, and appends the same two lines a
headless round writes — with `type: "terminal"` so the two sources stay tellable
apart. Every run is incremental against a cursor, so booking twice never counts a
token twice.

**It books the same bill from either side of the wire**, and works out which on
its own, so none of the commands below change with where you are:

- on the **server's own host**, with `KAIZEND_BOARDS_DIR` naming a real boards
  directory, it writes those two files directly — the original path, unchanged;
- on a **client**, with `KAIZEND_SERVER_URL` + `KAIZEND_TOKEN` set and no boards
  directory to reach, it books over `/v1/terminal-sessions/…` and the **server**
  appends the same two lines. The cursor lives wherever the marker does, which in
  this mode is the server — so a re-run books nothing there either.

A relative `KAIZEND_BOARDS_DIR` (what `dev.sh` exports) is deliberately not
trusted as a boards directory: it used to resolve to one that was not the
server's. With neither variable set it falls back to the script's built-in
default, `~/repo/kaizend/dev-boards` (`DEFAULT_BOARDS_DIR` in its header) — which
is a guess about one machine's layout, so set one of the two variables rather
than trusting it.

## Register the story before you work it

`story-discipline` already says the work becomes a card first. The moment that
card exists — or the moment you pick up an existing one — bind this session to it:

```bash
packages/kaizend-server/scripts/terminal-cost.py \
  start --session <this session id> --story <story-id>
```

Your session id is the last path segment of your scratchpad directory. The board
is worked out from whichever board holds the card — by the script over the folders,
or by the server over its own — so `--board` is only needed when the card isn't
there yet.

**This is the step that makes the rest work.** The end-of-session hook can only
book spend against a story it was told about; an unregistered session leaves no
record at all. Do it early — not at the end, when the window may already be gone.

Moving to a second story is a fresh `start` with the new id; the first story is
booked automatically on the way out.

## Book the stretch at every checkpoint

```bash
packages/kaizend-server/scripts/terminal-cost.py \
  record --session <id> --status done --note "wired the transport, suite green"
```

Run it when a story reaches a real boundary, without being prompted:

- the card moves column (into testing, into done, into failed),
- a major stretch of work lands — a feature wired end to end, a suite going green,
- you hand the story off, or park it for the user,
- you are about to say "done" in the conversation. Say it *after* the record.

`--status` takes the worker vocabulary: `done`, `wip`, `blocked`. `--note` is one
clause about what the stretch bought — it lands on the journal line beside the
numbers, which is the difference between a bill and a history.

## The hook is the backstop, not the plan

The kaizend checkout's `.claude/settings.json` wires two hooks. Both reach the script
through `$CLAUDE_PROJECT_DIR` and pin no boards directory, which is what lets one
checked-in settings file serve the server's host and every client alike:

- **`SessionEnd`** books whatever is unbooked, whatever the reason the session
  ended. This is what survives a closed window.
- **`Stop`** books at a turn boundary only once ~400k tokens have piled up
  unbooked, so a long session leaves a few spend lines instead of one per turn.

Both are best-effort and can only fire against a registered story. Never treat
them as a reason to skip a checkpoint `record` — the hook writes numbers, you
write the note that explains them.

## Reading it back

```bash
# what this session is bound to, and what is not yet booked — either mode
terminal-cost.py status --session <id>

# the story's history, terminal and headless interleaved. Reads from anywhere:
curl -sH "Authorization: Bearer $KAIZEND_TOKEN" \
  "$KAIZEND_SERVER_URL/v1/boards/blorkfield/cards/<story>/journal" | jq -r '.lines[]'

# on the server's own host the same two files read straight off disk
tail dev-boards/.orchestrator/journal/blorkfield/<story>.log
grep '<story>' dev-boards/.orchestrator/blorkfield.costs.jsonl | jq .
```

That journal endpoint is what the story-journal view in the app renders, so a
line written here is a line the user sees on the card.

## What the numbers are

Tokens, turns, and duration are measured — they come from the `usage` object the
API returned on each request, deduped by `requestId` (one request can land as
several transcript lines carrying the same usage; counting lines instead of
requests inflates the bill severalfold).

The dollar figure is **estimated**: the transcript records tokens, never dollars,
so the script prices them from a table in its own header, with the standard cache
multipliers. When model pricing changes, that table is the one place to fix — and
an unknown model is billed at the opus rate rather than as free, so a stale table
under-reports, never silently zeroes.
