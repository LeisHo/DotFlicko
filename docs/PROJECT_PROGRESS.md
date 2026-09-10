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

Nothing in progress from this session. Latest round of changes (spawn-
placement architecture rework, anchor-point refinements, swept collision,
live spawn preview, the visible-content-scan freeze fix, and Start/Stop +
draggable adjustment dots) all live on the `spawn-placement-mode` branch,
uncommitted — not merged/pushed to `master` yet.

**Note for a new session:** a *different*, concurrent Claude session has
also been actively developing this same `index.html` (the ball/collision
physics system and the skeleton-annotation tool) throughout this project's
history so far — check `git log`/`git status` before assuming this doc, or
any in-progress understanding of the file, is still current.

## Recently completed

On `spawn-placement-mode` (uncommitted):
- Reworked from a single mouse-anchored entity to a spawn-and-place model:
  the animation is invisible until placed; a 2-click gesture (1st = anchor
  point, 2nd = aim point) spawns a new, independently-tracked entity along
  that line; multiple entities can be placed at any angle; manual click-to-
  flick was retired (the ball-proximity trigger, generalized per-entity, is
  now the only way any entity's flick fires).
- Entity anchor point iterated twice: first to the raw PNG's own bottom-
  center (from the old centroid), then to the bottom-center of the frame's
  actual VISIBLE (non-transparent) content, via a one-time-per-direction
  alpha-channel scan (`computeVisibleContentBounds`) — real, non-trivial
  padding existed in the raw PNGs (e.g. 'front': centerX 0.4765, bottomY
  0.9145, not 0.5/1.0).
- Swept (segment-vs-segment) ball collision, replacing a point-only test —
  fixes a fast ball tunneling clean through the hand between 2 physics
  ticks. Live drag preview: the actual sprite, at the real angle/direction,
  renders translucently while aiming, before the 2nd click commits it.
- Fixed a freeze-on-first-click regression the visible-content scan itself
  introduced: the scan ran at full image resolution (36-83ms/direction)
  and the live preview called it every frame while sweeping across
  direction sectors — fixed by scanning a downscaled copy instead. The
  downscale dimension was later retuned from 200px to 400px after a
  follow-up accuracy report ("still isn't in the base of the visible
  portion") turned out to be a real, measured downscale error (~2-4px at
  200px on the worst direction) — 400px cuts that to sub-1px while still
  costing only ~1.2ms, nowhere near freeze territory.
- Fixed a second, unrelated "still freezes" report: the dev panel had no
  default-hidden state, so a genuinely fresh page load (no saved
  localStorage) rendered it wide open over most of the viewport —
  placement clicks landed on the panel instead of the canvas, looking
  exactly like a hang. Now boots hidden by default. Also added a
  defensive try/catch around the visible-content scan as precautionary
  hardening (not the confirmed cause of this report).
- Start/Stop buttons (always-visible, top-left, independent of the dev
  panel): Start lets balls spawn/fall, Stop halts that and clears the ball
  on screen immediately. Entity placement is gated to Stop mode only. In
  Stop mode, every entity shows 2 draggable dots (base = fixed anchor, end
  = aim point) — dragging the base dot moves the entity, dragging the end
  dot rotates it in place, both reusing the same angle/direction math the
  original 2-click spawn gesture uses.
- Animation X Offset: a 2nd compound dropdown+slider dev-panel row (mirrors
  Hand Rotation Offset's own architecture) letting each direction's sprite/
  collision placement be manually shifted perpendicular to its own
  placement line (%vmin, defaults 0) — a manual fix for directions that
  aren't visually centered. Hand Rotation Offset defaults were also
  re-baked to a newly-tuned set in the same pass.

Earlier, committed and pushed (`3517a52`, pre-spawn-placement-mode):
mouse-follow entity with center-pointing rotation and 8-direction angle
bucketing; per-direction Hand Rotation Offset; top-edge ball collision
(except the ball's initial off-screen fall-in); the original (non-swept)
ball-proximity flick auto-trigger; the fixed-1/60s-timestep physics
conversion (fixes non-deterministic ball trajectories from a static mouse
position); a self-healing canvas-resize check (also the likely fix for an
earlier "Hide OS Cursor doesn't work" report).

## What's next

No specific next action is currently queued by the user. `spawn-placement-
mode` is uncommitted and not yet merged to `master` — ask before merging/
pushing. Candidates not yet requested: wiring the SCISS/SNAP animation
variant sets into direction selection; touch/mobile input support.

## Open questions / blockers

None currently open.
