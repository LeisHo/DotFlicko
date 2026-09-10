DOTFLICKO -- CODE SUMMARY
================================================================================

Status: see ../README.md for the project-structure layout and how to run it;
this file is a quick orientation pointer into the actual code, not a
duplicate of the README's tree.

--------------------------------------------------------------------------------
FILE MAP

- `index.html` -- the entire app (~2400 lines as of 2026-09-10). One
  `<canvas id="scene">` for the animation, one `<div id="devPanel">` (built
  from the workspace's TEMPLATE_DEV_PANEL.html engine, §12) for the dev
  panel, one inline `<script>` for everything else. No build step, no
  dependencies.
- `scripts/active/flick-skeleton-annotator.html` -- a separate single-file
  tool (another concurrent session's own work) for hand-annotating the
  knuckle/joint1/joint2/tip keyframes that live in `data/processed/
  flick-skeleton.json` and are embedded as a literal (`FLICK_SKELETON`) in
  `index.html` itself.
- `data/FLICK/2TONED/<direction>/A/` -- the 8 base directions' 20-frame PNG
  sequences, plus SCISS/SNAP variant sets not yet wired into the app.
- `data/processed/flick-skeleton.json` -- source of truth for the skeleton
  data embedded in `index.html`'s `FLICK_SKELETON` constant; re-sync that
  literal by hand any time the annotator tool changes this file.

--------------------------------------------------------------------------------
ARCHITECTURE

Inside `index.html`'s single `<script>`, in roughly this order:

1. **Dev panel engine** (copied verbatim from TEMPLATE_DEV_PANEL.html) --
   fully generic, no project-specific names. Project code registers its own
   control arrays into it via `registerDevControlArray()` and renders rows
   via `findGroupContent()` + `buildUniformControlRow()`.
2. **Project config** (`cfg`) -- the single source of truth for every
   dev-tunable value. `ENTITY_CONTROLS`/`MOUSE_CONTROLS`/`BACKGROUND_CONTROLS`/
   `BALL_CONTROLS`/`COLLISION_CONTROLS`/`DEBUG_CONTROLS` arrays declare the
   dev-panel rows; `renderProjectControls()`/`wireProjectControls()` build
   and wire them. `ROTATION_OFFSET_CONTROLS` + `renderRotationOffsetRow()`/
   `wireRotationOffsetRow()` are a hand-built exception (a compound
   dropdown+slider row the generic per-control row builder can't express)
   still registered into the same generic save/load machinery.
3. **Direction data** (`DIRECTIONS`, `FLICK_SKELETON`, `DIRECTION_FRAME_COUNT`)
   -- 8 directions, each with a frame-URL prefix (no shared formula across
   directions) and per-keyframe skeleton joint positions.
   `interpolatedLocalPose()` interpolates the skeleton for any frame
   number; `currentRealFrameNumber()` maps the live playSequence index back
   to a real 1-20 frame number for that lookup.
4. **Frame loading** (`ensureDirectionLoaded()`) -- lazy per-direction
   Image preloading (only directions the mouse actually visits get
   fetched), each frame wrapped in `loadImageWithRetry()` (up to 3 retries
   -- a real, reproduced `net::ERR_CONNECTION_RESET` class of failure
   against this asset set under concurrent load).
5. **Angle/rotation math** (`mouseAngleFromCenter()`,
   `directionIndexForAngle()`, `angleLerp()`) -- 0=up, clockwise, verified
   against concrete geometric cases before trusting it.
6. **Ball + collision** (`spawnBall()`, `updateBallAndCollision()`,
   `ballInFlickTriggerZone()`) -- gravity-driven ball, capsule collision
   against the hand's 3 finger segments (knuckle-joint1, joint1-joint2,
   joint2-tip) transformed into world space via the SAME rotate+translate
   the sprite itself is drawn with. `ballInFlickTriggerZone()` reuses that
   exact transform but evaluates it across every frame (1..
   DIRECTION_FRAME_COUNT) of the CURRENT direction, not just the live pose,
   to drive the proximity auto-trigger.
7. **Fixed-timestep loop** (`step(dt)`, `update(now)`) -- `step()` always
   advances by exactly `FIXED_DT` (1/60s); `update()` runs once per real
   animation frame, accumulates real elapsed time, and runs `step()` 0-N
   times to catch up. `render()` runs once per real frame using whichever
   step's output was most recently produced.
8. **Migration** (`pruneStaleDevPanelGroups()`) -- runs once after
   `initDevPanelEngine()`, cleans up dev-panel groups from earlier code
   revisions that a browser's saved localStorage might still reference.

Data flow: mouse position (async `pointermove` listener) -> `step()` reads
it each fixed tick -> entity position/rotation/direction -> ball physics
reads the entity's current rotation + a skeleton pose -> `render()` draws
everything from the latest step's output. The dev panel writes directly
into `cfg`/the rotation-offset map via its own 'input'/'change' listeners,
read by `step()`/`render()` every tick -- no separate "apply settings" step.

--------------------------------------------------------------------------------
UNTOUCHABLE SYSTEMS

None formally designated yet.

--------------------------------------------------------------------------------
GOTCHAS

- `window.innerWidth`/`innerHeight` can read 0 at script-parse time or even
  persist at 0 on a long-lived/backgrounded tab in some environments (a
  real, reproduced-in-session quirk, not just a theoretical mobile-Safari
  edge case) -- `mouseX`/`entityX`/etc. defer their real initialization
  into the physics loop's own first live tick rather than a one-time
  top-level snapshot, and the canvas size is self-healed every frame
  (`update()`'s own size-mismatch check) rather than only at boot/resize.
  A 0x0 canvas isn't just a rendering bug -- `cursor:none` (Hide OS Cursor)
  has no real screen area to apply to when this happens.
- The physics loop is fixed-timestep (`step(dt)` always called with exactly
  `FIXED_DT = 1/60`), not raw per-real-frame delta -- this is what makes 2
  runs with identical inputs (a static mouse position included) produce
  numerically identical ball motion. Any NEW per-frame logic that needs to
  scale with elapsed time must go inside `step()` (or a function it calls
  with `dt`), never read `now`/real elapsed time directly, or it will
  reintroduce the exact "different every time" symptom this was built to
  fix.
- `ballInFlickTriggerZone()` deliberately checks EVERY frame (1..
  DIRECTION_FRAME_COUNT) of the current direction's animation, not just
  the peak/fully-extended frame -- an initial, narrower spec ("just the
  midpoint/peak") was explicitly corrected mid-task to "any point in the
  animation sequence." Don't narrow this back to a single frame without
  re-confirming that's actually wanted.
- The dev panel's "Hand Rotation Offset" row is NOT a normal generic
  per-control row -- it's 8 real slider/value DOM element pairs (one per
  direction, `sliderRotOffset_<key>`/`valueRotOffset_<key>`), only the one
  matching the dropdown's current selection visible (`display:none` on the
  rest). This is what makes each direction's value survive switching the
  dropdown without hitting Save. `pruneStaleDevPanelGroups()` exists
  specifically because 2 earlier group names ("Hand Animation", "Mouse /
  Rotation") got renamed/consolidated away mid-project -- any FUTURE group
  rename should add its old name to that function's `STALE_GROUP_NAMES`
  list, or anyone with settings already saved under the old name will see
  an empty ghost group reappear every load.
- `data/FLICK/2TONED/<direction>/A/` filenames use a DIFFERENT prefix
  convention per direction (confirmed via `ls`, no shared formula) --
  `DIRECTIONS` lists each direction's real prefix explicitly; don't assume
  a new direction/variant follows an existing one's naming pattern without
  checking the actual files on disk first (this exact assumption has
  already caused real bugs in the sibling DickoClicko project's own
  history for this same asset family).
- This project's dev panel renders every project-specific control ONLY
  into the Desktop tab (never Mobile/Landscape) -- a deliberate
  simplification, not an oversight, since the whole interaction model is
  mouse-driven with no touch equivalent yet. Don't "fix" this by adding
  Mobile/Landscape rows without first building real touch support.
