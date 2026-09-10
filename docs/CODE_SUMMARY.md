DOTFLICKO -- CODE SUMMARY
================================================================================

Status: see ../README.md for the project-structure layout and how to run it;
this file is a quick orientation pointer into the actual code, not a
duplicate of the README's tree. Reflects the `spawn-placement-mode` branch
(uncommitted) -- a real architecture rework from the original mouse-
anchored single-entity version; see ../CHANGELOG.txt for the full history
of how it got here.

--------------------------------------------------------------------------------
FILE MAP

- `index.html` -- the entire app (~2900 lines as of 2026-09-10). One
  `<canvas id="scene">` for the animation, always-visible Start/Stop
  buttons (`.sim-controls`, top-left, independent of the dev panel), one
  `<div id="devPanel">` (built from the workspace's TEMPLATE_DEV_PANEL.html
  engine, §12) for the dev panel, one inline `<script>` for everything
  else. No build step, no dependencies.
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
   still registered into the same generic save/load machinery. `cfg` itself
   is entirely GLOBAL/shared (entityScale, clickAnimSpeed, per-direction
   rotation offsets, ball/collision tuning, etc.) -- no per-entity value
   lives in `cfg`, only on each entity object itself (see #6).
3. **Direction data** (`DIRECTIONS`, `FLICK_SKELETON`, `DIRECTION_FRAME_COUNT`)
   -- 8 directions, each with a frame-URL prefix (no shared formula across
   directions) and per-keyframe skeleton joint positions.
   `interpolatedLocalPose()` interpolates the skeleton for any frame
   number; `entityRealFrameNumber(entity)` maps a given entity's own live
   playSequence index back to a real 1-20 frame number for that lookup
   (renamed from the old single-entity `currentRealFrameNumber()`).
4. **Frame loading** (`ensureDirectionLoaded()`) -- lazy per-direction
   Image preloading (only directions actually spawned/aimed-at get
   fetched), each frame wrapped in `loadImageWithRetry()` (up to 3 retries
   -- a real, reproduced `net::ERR_CONNECTION_RESET` class of failure
   against this asset set under concurrent load).
5. **Angle/rotation math** (`mouseAngleFromCenter()`,
   `directionIndexForAngle()`, `angleLerp()`) -- 0=up, clockwise, verified
   against concrete geometric cases before trusting it.
   `mouseAngleFromCenter(end, start)` gives "the angle of `end` as seen
   from `start`" -- reused as-is for the spawn-placement line's own angle
   (treating the 1st click as the pivot and the 2nd click/current mouse/
   drag target as the point being aimed at), not just the original mouse-
   to-viewport-center case it was built for.
6. **Entities** (`entities` array, `spawnEntityFromLine()`,
   `hitTestEntityDot()`) -- the animation is invisible until placed. A
   2-click gesture (canvas `pointerdown`) spawns a new entity: 1st click
   sets its fixed anchor (`x`,`y`); 2nd click sets its aim point
   (`endX`,`endY`, kept around permanently, not just baked into
   `rotationRad`, specifically so Stop mode has a real point to render/
   drag a handle at -- see #9) and derives `directionKey`/`rotationRad`
   from the line's angle via #5's math. Each entity then tracks its own
   play/collision/trigger state completely independently (`playing`,
   `playFrameIndex`, `wasInFlickTriggerZone`, `jointWorld`, etc.) -- no
   entity's state is ever shared with another's. Manual click-to-flick is
   retired entirely; the ball-proximity auto-trigger (#7) is the only way
   any entity's flick sequence starts.
7. **Ball + collision** (`spawnBall()`, `updateBallAndCollision()`,
   `ballInFlickTriggerZone()`) -- gravity-driven ball, SWEPT capsule
   collision (`closestPointsBetweenSegments()`, a standard
   closestPtSegmentSegment algorithm) against each entity's 3 finger
   segments (knuckle-joint1, joint1-joint2, joint2-tip), tested against
   the ball's whole PATH this tick (`prevBallX/Y -> ball.x/y`), not just
   its end-of-tick point -- a point-only test lets a fast ball tunnel
   clean through a thin capsule between 2 ticks without ever registering a
   hit. Segments are transformed into world space via the SAME
   rotate+translate the sprite itself is drawn with, computed once per
   entity per tick and stashed on the entity (`entity.jointWorld`/
   `jointSpeed`) for the collision pass, the debug-line render, and the
   trigger check to all read without recomputing.
   `ballInFlickTriggerZone()` reuses that exact transform but evaluates it
   across every frame (1..DIRECTION_FRAME_COUNT) of the entity's own
   direction, not just its live pose, to drive the proximity auto-trigger
   -- looped over every entity independently, so a ball near entity A can
   trigger A without touching B.
8. **Start/Stop** (`running`, `setRunning()`) -- gates ball spawning only
   (`updateBallAndCollision`'s respawn cycle runs `if (running)`; Stop's
   own handler also clears any ball on screen immediately). Boots stopped.
9. **Stop-mode entity adjustment** -- while `!running`, `render()` draws 2
   dots per entity (base = `x,y`; end = `endX,endY`) plus a dashed
   connector; `hitTestEntityDot()` (checked in `pointerdown`, before the
   2-click spawn gesture, only when no spawn is already pending) starts a
   drag; the shared `pointermove` listener does the actual dragging:
   dragging the base dot translates `x,y` AND `endX,endY` by the same
   delta (rotation/direction untouched -- "move"); dragging the end dot
   holds `x,y` fixed and recomputes `rotationRad`/`directionKey` from the
   new `endX,endY` via the exact same formula `spawnEntityFromLine()` uses
   ("rotate"), including a direction-key switch (and preloading its
   frames) if the drag crosses a 45-degree sector boundary. Gated to Stop
   mode the same way spawn-placement is (`pointerdown` returns immediately
   `if (running)`).
10. **Fixed-timestep loop** (`step(dt)`, `update(now)`) -- `step()` always
    advances by exactly `FIXED_DT` (1/60s); `update()` runs once per real
    animation frame, accumulates real elapsed time, and runs `step()` 0-N
    times to catch up. `render()` runs once per real frame using whichever
    step's output was most recently produced.
11. **Migration** (`pruneStaleDevPanelGroups()`) -- runs once after
    `initDevPanelEngine()`, cleans up dev-panel groups from earlier code
    revisions that a browser's saved localStorage might still reference.

Data flow: a 2-click gesture (or, in Stop mode, a dot drag) creates/adjusts
an entity's own x/y/endX/endY/rotationRad/directionKey -> `step()` advances
each entity's own play-sequence clock and calls `updateBallAndCollision()`
-> ball physics reads every entity's current rotation + a skeleton pose,
independently -> `render()` draws everything (entities, live spawn/drag
preview, Stop-mode dots, ball) from the latest step's output. The dev panel
writes directly into `cfg`/the rotation-offset map via its own 'input'/
'change' listeners, read by `step()`/`render()` every tick -- no separate
"apply settings" step. Start/Stop and entity placement/dragging are plain
DOM event handlers updating `running`/`entities` directly, same "no apply
step" pattern.

--------------------------------------------------------------------------------
UNTOUCHABLE SYSTEMS

None formally designated yet.

--------------------------------------------------------------------------------
GOTCHAS

- `window.innerWidth`/`innerHeight` can read 0 at script-parse time or even
  persist at 0 on a long-lived/backgrounded tab in some environments (a
  real, reproduced-in-session quirk, not just a theoretical mobile-Safari
  edge case) -- `mouseX`/etc. defer their real initialization into the
  physics loop's own first live tick rather than a one-time top-level
  snapshot, and the canvas size is self-healed every frame (`update()`'s
  own size-mismatch check) rather than only at boot/resize. A 0x0 canvas
  isn't just a rendering bug -- `cursor:none` (Hide OS Cursor) has no real
  screen area to apply to when this happens.
- The physics loop is fixed-timestep (`step(dt)` always called with exactly
  `FIXED_DT = 1/60`), not raw per-real-frame delta -- this is what makes 2
  runs with identical inputs (a static mouse position included) produce
  numerically identical ball motion. Any NEW per-frame logic that needs to
  scale with elapsed time must go inside `step()` (or a function it calls
  with `dt`), never read `now`/real elapsed time directly, or it will
  reintroduce the exact "different every time" symptom this was built to
  fix.
- `ballInFlickTriggerZone()` deliberately checks EVERY frame (1..
  DIRECTION_FRAME_COUNT) of an entity's own direction's animation, not just
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
- An entity's anchor is the bottom-center of its frame's actual VISIBLE
  (non-transparent) content, NOT the raw PNG's own bottom-center --
  `computeVisibleContentBounds()` (a one-time-per-direction alpha-channel
  scan, alpha>10 counts as visible, cached in `visibleBoundsCache`) finds
  the real box; `localOffsetFromNormalized()` takes that box's
  `centerX`/`bottomY` (defaulting to 0.5/1 if not yet cached) and must stay
  threaded through all 4 of its call sites in lockstep (`render()`'s main
  draw, `render()`'s live spawn/drag preview draw, `ballInFlickTriggerZone`,
  `updateBallAndCollision`'s per-entity joint transform) -- editing the
  anchor formula in only some of them desyncs the drawn sprite from its own
  collision geometry.
- **`computeVisibleContentBounds()` MUST scan a downscaled copy of the
  image, never the full-resolution source.** A full-res scan (up to
  2400x1181px) measured at 36-83ms per direction is fine as a one-time,
  rare cost -- but the live spawn/drag preview (render(), every frame)
  calls `visibleBoundsForDirection()` for whichever direction the mouse
  currently points toward, and sweeping across several direction sectors
  while aiming chains multiple fresh per-direction scans together into a
  real, reported page freeze. Fixed by scanning into an offscreen canvas
  downscaled first -- don't remove this downscale step; the previous
  full-res version is a proven, reproduced freeze.
- **`MAX_SCAN_DIM` is 400, not the original 200 -- don't lower it without
  re-measuring.** 200 fixed the freeze but was later found (a real user
  report: "still isn't in the base of the visible portion") to trade away
  more accuracy than assumed: downscaling blends the anti-aliased edge
  into a wider apparent footprint, measured at up to 0.0065 of image
  height (~2-4px at typical entityScale) off from a full-resolution scan's
  own bottomY on the worst-observed direction. At 400 that same error
  drops to 0.0018 (sub-1px) while the scan itself still measures only
  ~1.2ms (vs 200's ~0.6ms, full-res's own ~33ms, and 800's ~21ms -- 800
  was measured and rejected, most of the original freeze's own budget
  back). Any future retune of this constant should re-measure both the
  error (compare against a temporary full-res reference scan) and the
  timing (`performance.now()` around the scan) before picking a value --
  don't guess.
- Entities keep their own `endX`/`endY` (the original 2nd click) as real,
  permanent fields, not just a value baked once into `rotationRad` and
  discarded -- Stop mode's 2 draggable adjustment dots need an actual
  point to render/hit-test/drag the "end" handle at. Any code that spawns
  or clones an entity must set both `endX`/`endY` alongside `rotationRad`/
  `directionKey`, or the end dot will render at `undefined,undefined`.
- Dragging an entity's END dot (Stop-mode adjustment) intentionally does
  NOT need any velocity-discontinuity handling for a direction-key switch,
  even though `updateBallAndCollision`'s joint-velocity smoothing normally
  cares about exactly that kind of jump. Confirmed by reading that
  function's own joint computation first: `jointLocal` (what velocity is
  actually differenced from) depends only on `directionKey`/`entityScale`,
  never on `rotationRad`/`x`/`y` -- so a BASE-dot drag (pure translation)
  never touches `jointLocal` at all, and an END-dot drag only risks a
  single-tick velocity spike on an actual sector-crossing direction
  change, which is harmless since Stop mode (where dragging happens) never
  has a ball on screen to receive a stray collision kick from it.
- Start/Stop's `running` flag gates 2 different systems in OPPOSITE
  directions on purpose: ball spawning/physics only runs `if (running)`,
  while entity placement AND the Stop-mode dot-drag hit-testing only run
  `if (!running)` (`pointerdown` returns immediately `if (running)` before
  either path). Don't consolidate these into one shared guard -- they're
  deliberately inverted, not the same condition reused.
- `#devPanel` has the `hidden` class ON BY DEFAULT in the static HTML --
  don't remove it. There's no other gating or persisted-visibility
  mechanism for the panel's own open/closed state (only `panel-collapsed`
  persists); without the static `hidden` class, ANY fresh page load (no
  saved localStorage -- true for the user's own first-ever open of the
  page, and for anyone testing after an `localStorage.clear()`) renders
  the panel wide open, covering roughly the entire left/upper portion of a
  typical viewport, including the Start/Stop buttons. This was a real,
  reported bug ("placing the entities still freezes" -- actually every
  placement click landing on the invisible-to-the-user dev panel instead
  of the canvas, not a real hang) that a session's own repeated testing in
  an already-configured browser tab (panel already manually hidden from
  an earlier test) never caught, since that tab's `hidden` state carried
  over between reloads via the live DOM, not localStorage. Test dev-panel-
  adjacent changes against a genuinely fresh load (`localStorage.clear()`
  + reload), not just an already-configured tab.
- `visibleBoundsForDirection()`'s call into `computeVisibleContentBounds()`
  is wrapped in try/catch, falling back to (and caching) the same 0.5/1
  default used elsewhere in that function on any throw. This is
  precautionary, not a confirmed-necessary fix for any specific bug --
  `render()` (the only real caller, via the live preview and the main
  entity draw) runs inside `requestAnimationFrame` with no try/catch of
  its own above this point, so an uncaught throw here (e.g. a possible
  canvas-taint `SecurityError` on some browsers for a `file://`-opened
  page) would silently kill the entire render loop after the first
  placement attempt -- visually indistinguishable from a true freeze.
  Keep this wrapped; don't remove it as "unnecessary" without confirming
  first that no browser/environment this project targets can ever throw
  from that `getImageData()` call.
- `cfg.entityXOffsetByDirection` (Animation X Offset, %vmin, per direction,
  default 0) is a manual correction for sprites that aren't visually
  centered on their own placement line -- it's a LOCAL-x shift added
  BEFORE rotation (so it always reads as perpendicular to the entity's own
  facing direction in world space, never as a raw screen-space nudge), via
  `localOffsetFromNormalized()`'s 3rd optional `xOffsetPx` param. It must
  stay threaded through the SAME 4 call sites `vb.centerX`/`vb.bottomY`
  already has to stay in lockstep across (see that gotcha above) -- adding
  a 5th anchor-consuming call site without also adding its own xOffsetPx
  term reintroduces the same "sprite/collision desync" class of bug.
  Deliberately does NOT touch `entity.x`/`y`/`endX`/`endY` -- those stay
  the real clicked anchor/aim points, so the Stop-mode adjustment dots
  never move because of this setting, only the rendered sprite/collision
  geometry does.
