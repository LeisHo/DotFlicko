DOTFLICKO -- CODE SUMMARY
================================================================================

Status: see ../README.md for the project-structure layout and how to run it;
this file is a quick orientation pointer into the actual code, not a
duplicate of the README's tree. Reflects `main` post-merge of the
`spawn-placement-mode` rework (a real architecture change from the
original mouse-anchored single-entity version) plus the git-tracked Save
Settings port; see ../CHANGELOG.txt for the full history of how it got here.

--------------------------------------------------------------------------------
FILE MAP

- `index.html` -- the entire app (~3300 lines as of 2026-09-10). One
  `<canvas id="scene">` for the animation, always-visible Start/Stop/Delete
  buttons (`.sim-controls`, top-left, independent of the dev panel), one
  `<div id="devPanel">` (built from the workspace's TEMPLATE_DEV_PANEL.html
  engine, §12) for the dev panel, one inline `<script>` for everything
  else. No build step, no dependencies (see `api/` below for the one
  small exception).
- `api/save-settings.js` -- Vercel serverless function backing the dev
  panel's Save Settings button (git-tracked settings log, CLAUDE.md §12l);
  ported from Clicko/DickoClicko's own established implementation. See
  README.md for the required Vercel env var setup.
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
   against this asset set under concurrent load). `getScaledIdleFrame()`
   (mobile perf, see its own GOTCHAS entry) caches a pre-scaled canvas of
   each direction's frame 1, separate from `ensureDirectionLoaded()`'s own
   raw-Image cache.
5. **Angle/rotation math** (`mouseAngleFromCenter()`,
   `directionIndexForAngle()`, `angleLerp()`) -- 0=up, clockwise, verified
   against concrete geometric cases before trusting it.
   `mouseAngleFromCenter(end, start)` gives "the angle of `end` as seen
   from `start`" -- reused as-is for the spawn-placement line's own angle
   (treating the 1st click as the pivot and the 2nd click/current mouse/
   drag target as the point being aimed at), not just the original mouse-
   to-viewport-center case it was built for.
6. **Entities** (`entities` array, `spawnEntityFromLine()`,
   `hitTestEntityDot()`, `hitTestEntitySprite()`, `entityRotationRad()`) --
   the animation is invisible until placed. Placement supports 2
   interchangeable gestures ending in the same `spawnEntityFromLine()`
   call: 2 SEPARATE taps (canvas `pointerdown` sets `pendingSpawnStart` on
   the 1st, commits on the 2nd), OR a single press-drag-release (the
   `pointerup` handler commits using the release position IF it's the
   same `pointerId` that started the pending spawn AND moved past
   `MIN_DRAG_PLACEMENT_DISTANCE` -- below that, an ordinary tap's own
   release intentionally does nothing, preserving the 2-tap gesture).
   Both gestures are normally Stop-mode only, but `cfg.allowPlacementWhileRunning`
   (a debug checkbox) bypasses that specifically for NEW placement --
   dot-dragging an existing entity and Delete mode stay Stop-only
   regardless of the checkbox (see that checkbox's own GOTCHAS entry). 1st
   point sets the fixed anchor (`x`,`y`); 2nd point (however it arrived)
   sets the aim point (`endX`,`endY`, kept around
   permanently -- Stop mode's adjustment dots render/drag a real handle at
   it, AND it's the live source `entityRotationRad(entity)` recomputes
   rotation from every time it's needed, NOT a value frozen at spawn time)
   and derives `directionKey` from the line's angle via #5's math.
   `entityRotationRad(entity)` = `mouseAngleFromCenter(endX,endY,x,y) +
   cfg.entityRotationOffsetByDirection[directionKey]`, recomputed on every
   render/collision call rather than stored -- this is what makes moving
   the Hand Rotation Offset dev-panel slider immediately rotate every
   already-placed entity of that direction, per explicit request; no
   entity ever stores its own `rotationRad`. Each entity then tracks its
   own play/collision/trigger state completely independently (`playing`,
   `playFrameIndex`, `wasInFlickTriggerZone`, `jointWorld`, etc.) -- no
   entity's state is ever shared with another's. Manual click-to-flick is
   retired entirely; the ball-proximity auto-trigger (#7) is the only way
   any entity's flick sequence starts. `hitTestEntitySprite()` (Delete
   mode, #9) is a separate hit-test checking BOTH an entity's own drawn
   bounding box (the same rectangle Show Hitbox draws) AND actual per-pixel
   opacity at the click point (`getDirectionAlphaCanvas()`/
   `isOpaqueAtLocalPoint()`, a per-direction cached full-res canvas read
   once per click, not per-frame) — a bounding-box-only test was found to
   delete the wrong entity whenever 2 overlapping entities' boxes
   overlapped and a click landed in one's transparent padding that
   happened to cover a neighbor's actual visible hand. Distinct from
   `hitTestEntityDot()`'s tiny anchor-point radius used for dragging.
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
   trigger A without touching B. `cfg.ballMaxSpeed` (%vmin/s) hard-caps the
   ball's own speed once per tick, AFTER every velocity change that tick
   (gravity, wall bounce, the segment-reflection fix, the collision kick)
   -- checked last so it consistently caps the result regardless of which
   source pushed it over, not just one of them.
8. **Start/Stop** (`running`, `setRunning()`) -- gates ball spawning only
   (`updateBallAndCollision`'s respawn cycle runs `if (running)`; Stop's
   own handler also clears any ball on screen immediately). Boots stopped.
9. **Stop-mode entity adjustment** -- while `!running`, `render()` draws 2
   dots per entity (base = `x,y`; end = `endX,endY`) plus a dashed
   connector; `hitTestEntityDot()` (checked in `pointerdown`, before the
   2-click spawn gesture, only when no spawn is already pending) starts a
   drag; the shared `pointermove` listener does the actual dragging:
   dragging the base dot translates `x,y` AND `endX,endY` by the same
   delta (rotation/direction untouched, since `entityRotationRad()` only
   depends on the RELATIVE endX/endY-to-x/y geometry -- "move"); dragging
   the end dot holds `x,y` fixed and only updates `directionKey` if the
   drag crosses a 45-degree sector boundary (and preloads that direction's
   frames) -- rotation itself needs no explicit update at all, since
   `entityRotationRad()` recomputes it live from the now-updated
   `endX,endY` on the very next read ("rotate"). Gated to Stop mode the
   same way spawn-placement is (`pointerdown` returns immediately `if
   (running)`).
10. **Delete mode** (`deleteMode`, `setDeleteMode()`, `hitTestEntitySprite()`)
    -- a 3rd always-visible sim-control button, mutually exclusive with
    placement/dragging (turning it on clears any in-progress
    `pendingSpawnStart`/`draggingEntity`), gated to Stop mode the same way.
    `pointerdown` checks `deleteMode` FIRST, before the spawn/drag logic:
    a hit removes that entity from `entities` via `.filter()`; a miss does
    nothing (stays in delete mode, doesn't fall through to start a spawn).
11. **Fixed-timestep loop** (`step(dt)`, `update(now)`) -- `step()` always
    advances by exactly `FIXED_DT` (1/60s); `update()` runs once per real
    animation frame, accumulates real elapsed time, and runs `step()` 0-N
    times to catch up. `render()` runs once per real frame using whichever
    step's output was most recently produced.
12. **Migration** (`pruneStaleDevPanelGroups()`) -- runs once after
    `initDevPanelEngine()`, cleans up dev-panel groups from earlier code
    revisions that a browser's saved localStorage might still reference.

Data flow: a 2-click gesture (or, in Stop mode, a dot drag) creates/adjusts
an entity's own x/y/endX/endY/directionKey -> `step()` advances each
entity's own play-sequence clock and calls `updateBallAndCollision()` ->
ball physics reads every entity's current LIVE rotation (`entityRotationRad()`,
recomputed from endX/endY + the CURRENT cfg offset, never a stored/frozen
value) + a skeleton pose, independently -> `render()` draws everything
(entities, live spawn/drag preview, Stop-mode dots, ball) from the latest
step's output. The dev panel writes directly into `cfg`/the rotation-offset
map via its own 'input'/'change' listeners, read by `step()`/`render()`
every tick -- no separate "apply settings" step, and (per `entityRotationRad()`)
no separate "re-apply to existing entities" step either. Start/Stop, Delete
mode, and entity placement/dragging are plain DOM event handlers updating
`running`/`deleteMode`/`entities` directly, same "no apply step" pattern.

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
  `VISIBLE_BOUNDS_BY_DIRECTION` (a STATIC, precomputed-offline table, one
  `{centerX,bottomY}` entry per direction) supplies the numbers;
  `visibleBoundsForDirection()` is a plain synchronous lookup into it (with
  a 0.5/1 fallback for an unlisted direction key). `localOffsetFromNormalized()`
  takes that box's `centerX`/`bottomY` and must stay threaded through all 4
  of its call sites in lockstep (`render()`'s main draw, `render()`'s live
  spawn/drag preview draw, `ballInFlickTriggerZone`, `updateBallAndCollision`'s
  per-entity joint transform) -- editing the anchor formula in only some of
  them desyncs the drawn sprite from its own collision geometry.
- **This anchor used to be computed with a LIVE runtime canvas scan
  (`computeVisibleContentBounds()`/`visibleBoundsCache`/`MAX_SCAN_DIM`) --
  removed entirely, don't reintroduce it.** That approach went through 2
  full rounds of real, reported bugs: a full-resolution scan (2400x1181px,
  ~37-83ms) run from the live spawn/drag preview froze the page on first
  placement (sweeping across several newly-encountered directions in one
  gesture chained multiple such scans past a frame's budget); downscaling
  the scan to fix that (200px, later retuned to 400px) fixed the freeze but
  permanently traded away real accuracy (downscaling blends the anti-
  aliased edge into a wider apparent footprint -- measured up to 0.0065 of
  image height off from a full-resolution scan on the worst direction); and
  a defensive try/catch (added as hardening against a theoretical canvas-
  taint SecurityError for a locally-opened file:// page) could silently
  swallow ANY scan failure with only a `console.warn` and fall back to the
  OLD raw-PNG default (0.5/1) -- meaning a real failure in that exact class
  of environment would make the first 2 fixes completely irrelevant, since
  the scan would never even run. A user report ("still not at the base of
  the hand... regardless of scale") is what surfaced this 3rd, deeper
  problem. The static table eliminates all 3 failure modes at once: no
  runtime scan, so no freeze risk, no downscale accuracy tradeoff, and no
  environment-dependent canvas/getImageData failure mode.
- **To regenerate `VISIBLE_BOUNDS_BY_DIRECTION`** (e.g. after replacing a
  direction's frame 1 art): run the Node+`sharp` one-liner in the comment
  directly above the table in `index.html` against the new PNG, at FULL
  resolution (no downscaling needed -- this is now an offline, one-time
  computation, not a runtime cost) -- `sharp` is already present in this
  project's own `node_modules` (used for this exact purpose), no install
  needed. Don't hand-guess a replacement value.
- Entities keep their own `endX`/`endY` (the original 2nd click) as real,
  permanent fields -- NOT just used once to derive a rotation and then
  discarded. They serve 2 purposes: Stop mode's 2 draggable adjustment
  dots need an actual point to render/hit-test/drag the "end" handle at,
  AND `entityRotationRad(entity)` recomputes rotation from them fresh on
  every call (see the ARCHITECTURE section's #6) -- an entity has NO
  `rotationRad` field at all any more. Any code that spawns or clones an
  entity must set `endX`/`endY` alongside `x`/`y`/`directionKey`, or
  `entityRotationRad()` will throw/misbehave on `undefined` coordinates.
- `entityRotationRad(entity)` is a plain function call, not a cached/
  stored value -- called fresh every time rotation is needed (render's
  rotate, updateBallAndCollision's cos/sin, the trigger-zone check). This
  is deliberate: it's what makes moving the Hand Rotation Offset dev-panel
  slider immediately rotate every already-placed entity of that direction
  (a real, explicit request) with zero extra bookkeeping -- don't
  "optimize" this back into a cached field without re-solving that
  requirement some other way.
- Dragging an entity's END dot (Stop-mode adjustment) intentionally does
  NOT need any velocity-discontinuity handling for a direction-key switch,
  even though `updateBallAndCollision`'s joint-velocity smoothing normally
  cares about exactly that kind of jump. Confirmed by reading that
  function's own joint computation first: `jointLocal` (what velocity is
  actually differenced from) depends only on `directionKey`/`entityScale`,
  never on rotation/`x`/`y` -- so a BASE-dot drag (pure translation)
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
- (SUPERSEDED -- kept as a pointer, not a live gotcha) `visibleBoundsForDirection()`
  used to wrap a runtime `computeVisibleContentBounds()` scan in try/catch
  as a defensive measure against a possible canvas-taint `SecurityError`.
  That whole mechanism (scan, cache, try/catch) is now REMOVED -- see the
  "used to be computed with a LIVE runtime canvas scan" gotcha above.
  `visibleBoundsForDirection()` is now a plain table lookup that cannot
  throw at all, so this specific concern no longer applies.
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
- `hitTestEntitySprite()` (Delete mode) transforms a click point into an
  entity's own LOCAL unrotated space (rotate by `-entityRotationRad(entity)`,
  the inverse of the render/collision transform), then requires BOTH a hit
  against the bounding rectangle Show Hitbox draws AND actual pixel opacity
  at that point (`isOpaqueAtLocalPoint()`) -- a bounding-box-only version
  was a real, reported bug: entities placed close together have heavily
  overlapping boxes (sprites carry a lot of transparent padding), so a
  click meant for one entity's visible hand could land inside a
  DIFFERENT, closer-in-array (topmost-rendered) entity's box at a
  transparent point and delete the wrong one. A box hit on a transparent
  pixel now falls through to check entities further underneath rather
  than stopping there. Deliberately NOT the same hit-test as
  `hitTestEntityDot()` (a small fixed-radius circle around just the
  base/end points, used for dragging) -- Delete mode needs "click
  anywhere VISIBLE on the entity," dragging needs "click exactly on one
  of its 2 handles" -- don't merge these into one hit-test function, they
  answer genuinely different questions.
- `getDirectionAlphaCanvas()` draws a direction's frame-1 image to an
  offscreen canvas ONCE per direction (cached in
  `directionAlphaCanvasCache`), at FULL natural resolution -- this is safe
  (unlike the old, removed anchor-point scan) because it only ever runs on
  a rare, human-paced `pointerdown` click while Delete mode is active,
  never per-frame/per-render, and `isOpaqueAtLocalPoint()`'s own
  `getImageData()` call reads exactly 1 pixel, not the whole image. Don't
  call `getDirectionAlphaCanvas()`/`isOpaqueAtLocalPoint()` from any
  per-frame code path (render/collision) without re-deriving this safety
  reasoning first.
- **Save Settings writes to a git-tracked settings log (CLAUDE.md §12l),
  ported from Clicko/DickoClicko's own established 3-tier implementation**
  (`writeSettingsViaApi()`/`readSettingsViaApi()` → `api/save-settings.js`
  → GitHub Contents API; then `getGitSettingsFileHandle()`/
  `writeGitSettingsLog()`/`readGitSettingsLog()` → File System Access API;
  then `downloadSettingsAsFile()` + sessionStorage + the old
  `DEV_PANEL_LOCALSTORAGE_KEY` as a final fallback) -- see the big comment
  above `saveDevPanelSettings()` in `index.html`, and `README.md`, for the
  full picture and the required Vercel env vars (`GITHUB_TOKEN`,
  `DEV_PANEL_SAVE_SECRET`). Unlike Clicko/DickoClicko, this project's
  snapshot IS `captureFullDevPanelState()`'s own existing return value
  used as-is (not a hand-split values/order/panelGeometry/textOverrides
  object) -- this project's `DEV_PANEL_LAYOUT` is a single shared object,
  not split per device tab the way Clicko/DickoClicko's panel geometry is,
  so there's no "other tab's geometry" to separately merge in. Don't
  reintroduce that manual splitting without first confirming
  `DEV_PANEL_LAYOUT` has actually become per-tab in the meantime.
- `resetDevPanelSettings()` is called fire-and-forget at boot (not
  awaited) -- its own Tier 1 read is a network fetch that can resolve well
  after the page's synchronous boot sequence finishes. The page runs on
  its own hardcoded `cfg` defaults until/unless that resolves with
  something actually saved, same pattern as Clicko/DickoClicko's own boot
  sequence. `DEV_PANEL_SAVE_SECRET`'s value (`PkrbMti03M6xm3FEThYXa8gGW_08BOGj`)
  is the same workspace-wide shared token Clicko/DickoClicko already use --
  don't generate a new one without also updating `api/save-settings.js`'s
  own expected value AND the Vercel project's `DEV_PANEL_SAVE_SECRET` env
  var to match.
- Toggling Delete mode on (`setDeleteMode(true)`) clears any in-progress
  `pendingSpawnStart`/`draggingEntity`/`draggingPoint`, and `setRunning(true)`
  (Start) forces Delete mode off -- the 3 interaction modes (placing,
  dragging, deleting) are mutually exclusive by construction, not just by
  convention. Adding a 4th stopped-mode interaction should reset the other
  3 the same way, or 2 modes' state can end up active at once.
- **`effectiveDpr()` (capped at 2) MUST be used everywhere this file reads
  the canvas's own pixel ratio -- never `window.devicePixelRatio` directly
  a 2nd time.** A real, reported mobile performance complaint ("placing,
  rotating, moving... very very laggy") traced to the canvas backing store
  being sized off the RAW devicePixelRatio (up to 3 or higher on many
  phones), which is 9x more pixels than DPR 1 for the same on-screen CSS
  size (3x width * 3x height) -- every fillRect/drawImage this file does
  fills into that whole buffer. `resizeCanvas()` and `update()`'s own
  self-healing size-mismatch check BOTH call `effectiveDpr()` -- they must
  agree, or the mismatch check would see a permanent "wrong size" against
  an uncapped comparison and call `resizeCanvas()` (reallocating the
  entire backing store) every single frame, a far worse perf problem than
  the one this fixes.
- **`getScaledIdleFrame(directionKey, size)` caches a pre-scaled canvas of
  a direction's frame 1, keyed on `size`** (the one value shared by every
  direction, since `cfg.entityScale` is global -- NOT drawH, which varies
  per direction via that direction's own image aspect ratio). Used by
  BOTH the main render loop's idle-entity draw and the live spawn/drag
  preview draw -- exactly the 2 draw calls active while placing, rotating,
  or moving an entity (Stop mode, where nothing is ever mid-flick), and
  the other real half of the same reported mobile lag: `ctx.drawImage()`
  scaling a full-resolution source (up to 2400x1181px) down to on-screen
  size, every single frame, for every idle entity, is a real per-call
  cost on weaker mobile GPUs. Deliberately NOT used for a mid-flick
  PLAYING entity's own current frame -- those change every tick and are
  individually short-lived, so caching all of them isn't worth the
  memory; only frame 1 (the idle pose) is cached. If a future change adds
  a 3rd place that draws a direction's idle frame at `size`, route it
  through this same cache rather than a fresh `ctx.drawImage(img, ...)`
  call, or that 3rd site reintroduces the exact cost this exists to
  avoid.
- **Tap-and-drag placement (mobile) and the original 2-separate-taps
  gesture share the SAME `pendingSpawnStart`/`spawnEntityFromLine()` path
  -- they're 2 ways to reach the same commit, not 2 separate systems.**
  `spawnPointerId` (set alongside `pendingSpawnStart` in `pointerdown`)
  is what lets the `pointerup` handler tell "this release belongs to the
  press that started the pending spawn" apart from an unrelated
  pointerup (a 2nd finger, or a stray event) -- don't remove it as
  "unused" just because `pendingSpawnStart` alone looks sufficient.
  `MIN_DRAG_PLACEMENT_DISTANCE` (24px) is a deliberate, real distinction:
  a plain tap's own release (nearly 0px of movement, including ordinary
  touch jitter) must NOT auto-complete a placement, or the 1st tap of the
  ORIGINAL 2-tap gesture would immediately finish a degenerate,
  near-zero-length placement on its own release instead of waiting for a
  real 2nd tap. Don't lower this threshold without re-confirming ordinary
  tap jitter still stays under it.
- **`cfg.allowPlacementWhileRunning` (Debug group checkbox) is scoped to
  NEW placement ONLY -- it does NOT make dot-dragging or Delete mode
  available while running.** The `pointerdown` handler checks it in 2
  separate places rather than one shared top-level gate: `if (running &&
  !cfg.allowPlacementWhileRunning) return;` guards the placement path
  specifically (AFTER the `deleteMode` branch, which has its own
  unconditional `if (running) return;` that this checkbox never touches),
  and the dot-hit-test itself is forced to `null` whenever `running` is
  true (`const hit = running ? null : hitTestEntityDot(x, y);`) so a
  click landing exactly on an existing entity's own anchor point starts a
  NEW placement instead of grabbing that entity's dot for adjustment.
  Don't collapse these back into one shared `if (running &&
  !cfg.allowPlacementWhileRunning) return;` at the very top of the
  handler -- that would also let Delete mode and dot-dragging bypass
  Stop mode, which was never asked for and is explicitly out of scope
  for this checkbox. The `pointerup` handler's own drag-release-commit
  check mirrors the same condition (`(!running ||
  cfg.allowPlacementWhileRunning)`) for the press-drag-release gesture.
- **`#scene` (the canvas) MUST keep `touch-action: none` in its CSS rule.**
  Without it, a real touchscreen's native gesture recognizer treats a
  press-drag on the canvas as a pan attempt and hijacks the touch mid-
  gesture, firing `pointercancel` instead of letting `pointermove`/
  `pointerup` continue -- this is what broke the tap-and-drag placement
  gesture (§ above) on real mobile hardware even though it worked
  correctly in this environment's own synthetic-event testing (a
  dispatched `PointerEvent` never goes through native gesture
  recognition, so this exact class of bug is invisible to that kind of
  test -- only code inspection + matching the convention already used by
  every OTHER draggable element in this file, e.g. `.dev-header`/the
  resize handles, actually catches it). A `pointercancel` listener on
  `window` clears `draggingEntity`/`draggingPoint` and, when it matches
  `spawnPointerId`, `pendingSpawnStart`/`spawnPointerId` too -- defensive
  cleanup for a genuine cancel (OS interruption, an edge-swipe nav
  gesture) that `touch-action: none` doesn't fully rule out; without it a
  cancelled press-drag would leave `pendingSpawnStart` stuck, silently
  blocking every future placement.
- **`entityFlickSpeedMultiplier(entity)` (Flick Speed Multiplier) follows
  the EXACT SAME live-recompute pattern as `entityRotationRad()` -- NOT a
  value frozen at spawn time.** Per explicit request, a placement line's
  own length (base to aim point) relative to that direction's own
  VISIBLE content height (`VISIBLE_BOUNDS_BY_DIRECTION`'s `heightFrac`
  field, at the CURRENT Entity Scale) scales the flick's OWN animation
  playback speed: <=100% of that height -> 1x, 100%-200% -> linear ramp
  1x-2x, 200%+ -> capped at 2x (the 1x-2x multiplier range maps 1:1 onto
  the 1x-2x distance-ratio range, so the formula is just the clamped
  ratio itself, `Math.min(2, Math.max(1, dist/visibleHeightPx))` -- no
  separate interpolation curve). **This originally scaled the
  ball-collision-kick strength instead ("Flick Intensity Multiplier"),
  then was explicitly repurposed** ("instead of the intensity, make it
  control the hand animation speed") -- the collision-kick site
  (`ball.vx/vy += nx/ny * speed * cfg.collisionIntensity`) is back to
  using ONLY the global slider, no per-entity factor; the SAME
  formula/function (only renamed, logic untouched) now instead
  multiplies `step()`'s own `entity.playFrameAccum` advancement
  (`dt * FLICK_BASE_FPS * cfg.clickAnimSpeed * entityFlickSpeedMultiplier(entity)`),
  ON TOP OF the shared Hand Anim Speed slider, not a replacement for it
  -- verified precisely via 2 entities (short vs. maximally-long line)
  ticked in lockstep: the 2x entity's own frame index tracked almost
  exactly double the 1x entity's at every sampled tick. Reads the
  entity's own permanent `endX`/`endY` fresh every call (via
  `flickSpeedMultiplierForLine()`, the underlying formula shared with
  the live spawn/drag preview's own debug-text readout), so it
  automatically tracks a live Entity Scale slider change AND a
  Stop-mode end-dot drag, exactly like rotation already does.
- **`clampLineEndpoint(directionKey, startX, startY, rawEndX, rawEndY)` is
  a SEPARATE, additional cap from the Flick Speed Multiplier's own
  saturation -- it constrains the actual point, not just a derived
  value.** Per explicit request ("cap the actual length of the visible
  dotted line to 2x the length of the visible part of the animation
  frame" / "IE 2 hand lengths"), pulls a raw end point in along the SAME
  angle to at most `MAX_PLACEMENT_LINE_RATIO` (2) times
  `visibleHeightPxForDirection()` (factored out of
  `flickSpeedMultiplierForLine()` for this reuse) -- direction/angle
  is always resolved from the RAW, uncapped point FIRST (bucketing
  depends only on angle, never distance), and the clamp applied only
  after, so capping never changes which direction an entity resolves
  to. Applied at all 3 places an aim/end point is set or displayed, kept
  in sync deliberately: `spawnEntityFromLine()` (the committed entity's
  own endX/endY), the Stop-mode end-dot `pointermove` handler
  (`draggingEntity.endX/endY`), and `render()`'s own live placement
  preview (the dashed line's drawn endpoint only -- the sprite preview
  itself was already angle-only, unaffected either way). If a 4th place
  ever sets an aim/end point, it needs this same clamp too, or that one
  path would silently let the line exceed the cap while every other path
  enforces it.
- **`VISIBLE_BOUNDS_BY_DIRECTION`'s `heightFrac` field measures frame 1's
  (idle) visible content only** -- deliberately the SAME frame the
  `centerX`/`bottomY` anchor fields already derive from, not the
  peak/most-extended flick frame, so a placement's own Flick Intensity
  reading always measures against the entity's idle silhouette
  regardless of which frame the animation later plays. If a direction's
  frame 1 art is ever replaced, regenerate `heightFrac` (and
  `centerX`/`bottomY`) together via the one Node+`sharp` one-liner in the
  comment directly above the table -- don't hand-guess it, and don't
  regenerate only 2 of the 3 fields from a stale computation.
- **Win/Lose (`WIN_LOSE_ASSETS`, `triggerWinLoseSequence()`,
  `advanceWinLoseSequences()`) is a wholly SEPARATE asset family and
  state machine from the normal flick system -- do not try to unify
  them.** Different frame count (`WIN_LOSE_FRAME_COUNT = 24`, vs.
  `DIRECTION_FRAME_COUNT = 20`), different source raster
  (1920x912, not 2400x1181 -- confirmed via `sharp` metadata, not
  assumed), its own independently-computed `centerX`/`bottomY` anchor
  per direction (NOT `visibleBoundsForDirection()` -- reusing that table
  for this different art would misplace it), and a genuinely different
  playback shape (2 real-TIME holds via `cfg.winLoseFistHoldDuration`/
  `winLoseFinalHoldDuration`, not the normal flick's single frame-COUNT
  hold via `peakHoldFrames`, and an asymmetric forward-holds/reverse-no-
  holds shape the normal forward+reverse `playSequence` array doesn't
  have at all). `entity.winLoseType` being non-null takes over that
  entity's ENTIRE draw dispatch in `render()` ahead of both the
  `playing`/idle branches (mutually exclusive with a normal flick --
  `triggerWinLoseSequence()` forces `entity.playing = false` first).
- **Win/Lose assets were copied in from a SIBLING project**
  (`J:\CLAUDE\PROJECTS\DICKOCLICKO\data\FLICK\2TONED\<FOLDER>_MF|_TU\`),
  only the `FIST` subfolder of each (NOT the sibling `ARCH`/`NOFIST`
  variants also present there, which this feature never uses) --
  DOTFLICKO is a static single-file app with no cross-project asset
  serving, so the files had to be physically copied into this project's
  OWN `data/FLICK/2TONED/<FOLDER>_MF|_TU/FIST/` (384 files, ~15MB), not
  just referenced by a DICKOCLICKO-relative path. `_MF` = Lose, `_TU` =
  Win, per explicit instruction -- don't swap these.
- **Frame count is PER-ASSET (`WIN_LOSE_ASSETS[key][type].frameCount`),
  not a shared global** -- a real asset replacement report ("i replaced
  the animation frames for BehindThumb_TU. There are now 48 frames, so
  instead of pausing on frame 13, you will pause on frame 25") proved a
  single global count can't survive a real per-direction frame-count
  change. `triggerWinLoseSequence()` derives and stores
  `entity.winLoseFistIndex`/`.winLoseLastIndex` PER ENTITY at trigger
  time (`Math.floor(frameCount / 2)` 0-based / `frameCount - 1`) from
  THAT entity's own asset's own `frameCount` -- don't reintroduce a
  shared top-level `WIN_LOSE_FIST_INDEX`/`WIN_LOSE_LAST_INDEX` constant,
  2 different directions can now genuinely have 2 different frame counts
  (e.g. several `_TU` sequences sit at 48 frames while every `_MF`
  sequence and the unrevised `_TU` ones stay at 24) at the same time.
- **`winLoseVisibleBounds(directionKey, type)` reads a PER-TYPE
  `centerX`/`bottomY` override on the `lose`/`win` sub-object first,
  falling back to the shared direction-level value** -- render()'s own
  Win/Lose draw dispatch calls this (never reads
  `WIN_LOSE_ASSETS[key].centerX/bottomY` directly). Every direction
  originally shared ONE anchor between lose/win (both start from the
  same "pre-flick" pose) -- 'front-pinky' broke that assumption when its
  own Win/TU frame 1 was replaced with genuinely different art (measured
  centerX/bottomY 0.4846/0.9189 vs. the shared 0.4943/0.9221 every other
  still-unrevised entry uses) -- confirmed by an actual recompute, not
  assumed. Only add a per-type override when a real recompute shows a
  genuine difference (as with front-pinky) -- don't add one
  speculatively "just in case" for an entry that still measures
  identically.
- **A Win/Lose entity's draw dispatch in `render()` MUST fall back to
  its own idle frame whenever the target frame isn't loaded yet -- never
  leave `drawSource` as `null` in that branch.** `ensureWinLoseLoaded()`
  is lazy (same as the normal flick set) and can genuinely still be
  mid-fetch the first time a direction is actually used; a real,
  reported bug ("on a hard refresh, the first time i hit Win and
  Lose... they each flash in succession") traced to exactly this --
  with no fallback, an entity whose current frame hadn't finished
  downloading drew NOTHING that tick, so several entities (each
  fetching a different direction's own 24 images at different network
  speeds) blinked into existence one at a time as their own fetches
  happened to complete. The fallback reuses `getScaledIdleFrame()` +
  `visibleBoundsForDirection()` -- the SAME source/anchor the plain
  idle branch just below it uses, not a half-composed mix of the 2
  asset families. `spawnEntityFromLine()` also now calls
  `ensureWinLoseLoaded()` for BOTH types the moment an entity is
  placed (not only once Win/Lose is actually clicked) as a genuine,
  directly-relevant reduction in how often this fallback path is even
  needed -- not a substitute for it, since a fast-enough click after
  placement can still race the network either way.
- **`cfg.collisionOnlyWhilePlaying` gates ONLY the physical joint-capsule
  collision block in `updateBallAndCollision()` -- it does NOT touch
  the ball-proximity auto-trigger (`ballInFlickTriggerZone`).** These
  are 2 separate checks in the same `entities.forEach` iteration (the
  trigger check is a sibling statement AFTER the collision block, not
  nested inside it) -- gating the trigger check too would mean an idle
  entity could never detect the ball closely enough to ever start
  playing in the first place, permanently bricking every entity. Real,
  live-verified behavior (manually-driven fixed-timestep ticks, since
  the browser pane's own rAF loop is fully suspended while hidden, not
  just throttled): a ball approaching an idle entity crosses the
  (unaffected) trigger-zone radius FIRST, flipping `entity.playing`
  true via the normal auto-trigger, and only several ticks later
  reaches the tighter physical-capsule collision distance -- so with
  this checkbox on, an idle entity still starts its own flick normally,
  it just doesn't get shoved by the ball's geometry until it's actually
  mid-flick.
- **The ball-proximity auto-trigger MUST also check `!entity.winLoseType`,
  not just `!entity.playing`, before calling `triggerFlick()`.** A real,
  reported bug ("the ball still gets stuck sometimes" -- screenshot of a
  dense ring of Win-animating entities) traced to exactly this gap:
  `entity.playing` is deliberately `false` throughout an entire Win/Lose
  sequence (so the 2 systems don't fight over `render()`'s draw
  dispatch), but with no `winLoseType` check, a ball drifting near a
  Win/Lose-active entity would ALSO silently `triggerFlick()` a normal
  flick underneath it -- invisible on screen (`winLoseType` still wins
  the draw dispatch), but `entityRealFrameNumber()` (which drives the
  ACTUAL collision geometry, independent of what's rendered) would
  start tracking that accidental flick's own animating frames instead
  of the static idle pose, desyncing collision geometry from what's
  shown. With many entities doing this independently and
  asynchronously in a tight cluster, the ball can get caught in an
  unpredictable, constantly-shifting collision field. Verified via
  manually-driven fixed-timestep ticks (same technique as the
  `collisionOnlyWhilePlaying` entry above): a ball parked continuously
  in a Win-active entity's own trigger zone across a full sequence
  (forward1/holdFist/forward2/holdFinal) never flipped `entity.playing`
  true, while the identical setup against a normal (non-Win/Lose)
  entity still triggered correctly at the expected tick -- confirms the
  fix without a regression to the original mechanism.
- **The ball-vs-finger-capsule collision resolver still resolves
  VELOCITY sequentially (one capsule at a time, each correcting the
  ball's velocity in place before the next segment's sweep runs) -- but
  POSITION is now relaxed iteratively across every capsule hit that
  tick, fixing a real "ball still gets stuck sometimes" report.**
  `updateBallAndCollision()` collects every capsule hit that tick into
  `hitCapsules` as it resolves each one's velocity effect (reflection +
  the collision kick, applied exactly once per capsule, unchanged).
  After that loop, a position-only relaxation pass
  (`POSITION_RELAXATION_ITERATIONS = 4`) re-checks the ball's CURRENT
  position against every collected capsule each iteration (via a
  degenerate zero-length "swept" call to `closestPointsBetweenSegments`
  -- confirmed safe against a zero-length segment via that function's
  own `a <= EPS` branch) and pushes out remaining overlap, breaking
  early once clear. This is what a single sequential pass couldn't do:
  in a densely-packed cluster (e.g. the reported screenshot's ~16
  entities in a tight ring), correcting the ball's position against one
  capsule could silently reintroduce overlap with an earlier one that
  the single pass never rechecked. Mirrors the iterative
  constraint-relaxation technique this workspace's own DickoClicko
  project already uses for its Verlet rope-constraint solving. Skipped
  entirely when `hitCapsules.length <= 1` -- zero cost/risk for the
  common sparse-placement case. Fixed SEPARATE from and in addition to
  the `winLoseType`-auto-trigger bug above (both were real, independent
  contributors to the same reported symptom). Verified via an isolated
  Node.js simulation reproducing the exact bug (old single-pass
  approach left the ball at 0.00 distance from -- i.e. fully re-stuck
  against -- a second capsule after "resolving" the first) and the
  exact fix (new iterative approach converges to a position clear of
  both capsules, e.g. distances of 28.28 and 20.00 in one measured run,
  within 3 iterations). Committed as `32668e4`.
