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

Nothing in progress from this session. The spawn-placement rework (started
on a `spawn-placement-mode` branch) is complete, merged into `main`, and
pushed to `origin/main` (commit `3be0ac4`). Save Settings' git-tracked
settings log (ported from Clicko/DickoClicko) is **confirmed working
end-to-end in production** — the user's own live Save clicks on the
deployed Vercel site committed `data/processed/dev-panel-settings.json`
straight to GitHub multiple times, pulled into this repo via several
merges.

**Open item:** a real mobile-lag report ("placing, rotating, moving...
very very laggy") was addressed with 2 well-established canvas
performance fixes (DPR capped at 2; idle-frame sprites now drawn from a
pre-scaled cache instead of rescaling the full-resolution source every
frame) — but this session could only verify the fixes are correct
(cache hits/invalidates properly, rendering/rotation/dragging all still
work), NOT that they actually resolve the reported lag on real mobile
hardware, since this environment has no way to profile real mobile
CPU/GPU performance. **The user should re-test on their actual phone and
report back** — if still laggy, the next real lead is the source PNGs
themselves (up to 2400x1181px per frame, likely oversized for this
project's own on-screen scale — DickoClicko's own history flagged this
same "oversized PNGs" issue before) or lowering `DIRECTION_FRAME_COUNT`'s
own asset footprint, neither of which this session attempted. Also added
(same mobile push, per explicit request): a press-drag-release placement
gesture as an alternative to the original 2-separate-taps flow — verified
logically correct (via mouse-drag simulation and the browser tool's own
touch emulation), but real on-device touch-feel confirmation is the
user's own to make too.

**Note for a new session:** a *different*, concurrent Claude session has
also been actively developing this same `index.html` (the ball/collision
physics system and the skeleton-annotation tool) throughout this project's
history so far — check `git log`/`git status` before assuming this doc, or
any in-progress understanding of the file, is still current. As of this
update, that other session also has ~552 uncommitted working-tree changes
(a rename/re-export of the SCISS/SNAP variant PNG asset sets) sitting
alongside this session's own commits — untouched by this session's commits
(verified: staged and committed only this session's own 4 files), but a
new session should be aware they're there before touching `data/FLICK/`.

## Recently completed

- Reworked from a single mouse-anchored entity to a spawn-and-place model:
  the animation is invisible until placed; a 2-click gesture (1st = anchor
  point, 2nd = aim point) spawns a new, independently-tracked entity along
  that line; multiple entities can be placed at any angle; manual click-to-
  flick was retired (the ball-proximity trigger, generalized per-entity, is
  now the only way any entity's flick fires). A live drag preview shows the
  actual sprite, at the real angle/direction, while aiming.
- Entity anchor point (bottom-center of each direction's actual VISIBLE,
  non-transparent content, not the raw PNG's own bottom-center) is now a
  precomputed static table (`VISIBLE_BOUNDS_BY_DIRECTION`), not a runtime
  scan — the runtime-scan approach went through 2 rounds of real bugs (a
  freeze, then an accuracy tradeoff from the freeze's own downscale fix,
  then a 3rd report tracing back to a defensive try/catch silently masking
  scan failures) before landing here; see `CHANGELOG.txt` for the full
  history. The table is regenerated offline with Node + `sharp` (already
  in this project's `node_modules`) directly against the source PNGs.
- Swept (segment-vs-segment) ball collision, replacing a point-only test —
  fixes a fast ball tunneling clean through the hand between 2 physics
  ticks.
- Start/Stop/Delete buttons (always-visible, top-left, independent of the
  dev panel): Start lets balls spawn/fall, Stop halts that and clears the
  ball on screen immediately. Entity placement, dragging, and deletion are
  all gated to Stop mode, and mutually exclusive with each other. In Stop
  mode, every entity shows 2 draggable dots (base = fixed anchor, end =
  aim point) — dragging the base dot moves the entity, dragging the end
  dot rotates it in place; Delete mode lets a click on any placed entity's
  own sprite remove it.
- Hand Rotation Offset is now LIVE: moving its dev-panel slider immediately
  rotates every already-placed entity of that direction (rotation is
  recomputed from each entity's own fixed aim point + the current offset
  every frame, never stored/frozen).
- Animation X Offset: a 2nd compound dropdown+slider dev-panel row (mirrors
  Hand Rotation Offset's own architecture) letting each direction's sprite/
  collision placement be manually shifted perpendicular to its own
  placement line (%vmin, defaults 0) — a manual fix for directions that
  aren't visually centered.
- Hand Rotation Offset and Animation X Offset defaults re-baked to tuned
  sets from pasted Copy Settings dumps (several times, as tuning
  continued); `angleOffset` default set to 180.
- Delete mode's hit-test now checks actual pixel opacity, not just an
  entity's bounding box — fixes a real reported bug where clicking one
  entity could delete a different, nearby one whenever their (heavily
  padded) bounding boxes overlapped.
- Save Settings now writes to a git-tracked settings log
  (`data/processed/dev-panel-settings.json`), ported from Clicko/
  DickoClicko's own established 3-tier implementation (Vercel/GitHub API
  → File System Access API → local download/session fallback) rather than
  localStorage only — see README.md for the Vercel env var setup.
- Ball Max Speed (%vmin/s) dev-panel slider — a hard cap on the ball's own
  speed, applied once per tick after every velocity change that tick
  (gravity, wall bounce, the segment-reflection fix, the collision kick).
- Small dev-panel polish: the "+ Add Group" button now matches Copy/
  Reset/Save's own styling (it was an unstyled bare `<button>` before);
  the `showAngleDebug` status text moved from y=12 to y=50 so it no
  longer renders behind the Start/Stop/Delete buttons.
- Skeleton re-annotated (the other concurrent session's own
  flick-skeleton-annotator.html work): denser keyframes for 6 of 8
  directions (`behind`, `behind-thumb`, `side-thumb`, `front-thumb`,
  `front`, `front-pinky` each gained 2-3 new keyframes; `side-pinky`/
  `behind-pinky` unchanged), purely additive — no keyframes removed, no
  direction/point keys missing. Synced to both
  `data/processed/flick-skeleton.json` and the embedded `FLICK_SKELETON`
  literal in `index.html` (confirmed byte-identical after the sync).
- Mobile performance: canvas DPR capped at 2, and idle-entity/live-preview
  sprites now draw from a pre-scaled cache instead of rescaling the
  full-resolution source every frame (see Open item above).
- Tap-and-drag placement: press for the 1st point, drag, release for the
  2nd — an alternative to the original 2-separate-taps gesture, not a
  replacement (a plain tap's own release still leaves the pending spawn
  active, waiting for a real 2nd tap, exactly as before).
- Debug checkbox "Allow Placement While Running" — lets entities be
  newly placed regardless of Start/Stop state. Scoped to new placement
  only; dot-dragging an already-placed entity and Delete mode stay
  Stop-only regardless of the checkbox.

Earlier (pre-spawn-placement-mode, also on `main`): mouse-follow entity
with center-pointing rotation and 8-direction angle bucketing; per-
direction Hand Rotation Offset (the original, non-live version); top-edge
ball collision (except the ball's initial off-screen fall-in); the
original (non-swept) ball-proximity flick auto-trigger; the fixed-1/60s-
timestep physics conversion; a self-healing canvas-resize check.

## What's next

No specific next action is currently queued by the user. Candidates not
yet requested: wiring the SCISS/SNAP animation variant sets into direction
selection (note: another session appears to be actively re-exporting those
exact assets right now, per the note above — coordinate before starting
this); touch/mobile input support.

## Open questions / blockers

None currently open.
