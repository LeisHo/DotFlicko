# DOTFLICKO — Project Progress

**This is a live document, not a log.** It holds only the current picture —
what's being worked on right now, what's recently done, and what's next. It
does **not** accumulate a running history of every past session; that
history already lives in `CHANGELOG.txt` (the append-only, authoritative
record — see CLAUDE.md §4a/§4). When something here is finished and no
longer relevant to understand what's current, remove it from this file
rather than leaving it to pile up. Rewrite the sections below in place at
each real update — don't append a new dated block underneath the old one.

This doc functionally doubles as a handoff document (CLAUDE.md §4c): a
brand-new AI chat with no prior context should be able to read this file
alone and know exactly where the project currently stands, and pick up the
work seamlessly from there.

--------------------------------------------------------------------------------

## Currently working on

Nothing in progress from this session — the latest round of changes (per-
direction Hand Rotation Offset, top-edge ball collision, the ball-proximity
flick auto-trigger, and the fixed-timestep physics conversion) is committed
and pushed (`3517a52`).

**Note for a new session:** a *different*, concurrent Claude session has
also been actively developing this same `index.html` (the ball/collision
physics system and the skeleton-annotation tool) throughout this project's
history so far — check `git log`/`git status` before assuming this doc, or
any in-progress understanding of the file, is still current.

## Recently completed

- Mouse-follow entity with center-pointing rotation and 8-direction angle
  bucketing (verified via geometric test cases); click-to-play forward+
  reverse flick animation with a configurable peak-frame hold.
- Dev panel built from the workspace's `TEMPLATE_DEV_PANEL.html`, settings
  decomposed per CLAUDE.md §12n into Entity/Background/Debug groups (Ball
  and Collision groups are the other concurrent session's own).
- Hand Rotation Offset made per-direction: a dropdown + slider compound row
  (8 real, independently-persisted values) consolidated into one "Entity"
  group (an earlier brief "Hand Animation" group name, and a separate
  "Mouse / Rotation" group, were both folded back into Entity per explicit
  request — a migration also handles anyone whose browser had already
  saved settings referencing the old names).
- Top-edge ball collision, except during the ball's initial fall into frame
  from its off-screen spawn point.
- Ball-proximity auto-trigger for the flick sequence: checked against the
  swept collision geometry across the full animation sequence (all 20
  frames per direction), not just the current or peak pose; rising-edge
  triggered so a ball sitting in the zone doesn't spam-retrigger.
- Physics loop converted from raw per-real-frame delta to a fixed 1/60s
  timestep accumulator — verified deterministic (2 differently-jittered
  real-frame-timing sequences produce bit-for-bit identical ball state when
  compared at matching fixed-step count), fixing a reported "same mouse
  position, different trajectory every time" symptom.
- A self-healing canvas-resize check (fixes a real, reproduced-in-session
  "canvas stuck at 0×0" class of bug, also the likely cause of a "Hide OS
  Cursor doesn't work" report).

## What's next

No specific next action is currently queued by the user. Candidates not yet
requested: wiring the SCISS/SNAP animation variant sets into direction
selection; touch/mobile input support.

## Open questions / blockers

None currently open.
