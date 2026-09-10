DOTFLICKO -- PROJECT SUMMARY
================================================================================

Status: working single-file HTML/canvas toy -- mouse-follow hand entity,
click-or-proximity-triggered flick animation, ball physics/collision -- as
of 2026-09-10. See PROJECT_PROGRESS.md for what's actively being worked on
right now, and CHANGELOG.txt for the full historical log -- this file stays
a mid-altitude snapshot, not a duplicate of either.

--------------------------------------------------------------------------------
OBJECTIVE

A single-file HTML/canvas interactive toy. A hand entity is anchored to the
mouse and continuously rotates so its pre-drawn "up" orientation always
points at the viewport's centerpoint. The hand switches between 8
directional animation sets based on the mouse's angle from center (8 exact
45-degree sectors), and plays a forward-then-reverse flick animation either
on click or automatically when a gravity-driven ball's trajectory enters the
hand's own reachable collision zone. No backend, no build step, no accounts
-- a static page, same category as this workspace's other small interactive
toys (Clicko, DickoClicko).

--------------------------------------------------------------------------------
SCOPE

In scope: the mouse-follow/rotation/direction-bucket mechanic; the 8-
direction FLICK animation system (frame-set preloading, forward+reverse
playback, per-direction rotation-offset tuning); the dev panel (built from
the workspace's TEMPLATE_DEV_PANEL.html per CLAUDE.md Section 12); a
gravity-driven ball with hand-collision physics (3-capsule finger collision
against an annotated skeleton) and wall bouncing (left/right always, top
except during the ball's initial fall into frame, never bottom); the ball-
proximity auto-trigger for the flick sequence; a fixed-timestep physics loop
for reproducible motion.

Out of scope / dormant: this repo started as a repurposed DickoClicko
checkout -- that project's own rope-physics/circle-punch mechanic and its
full git history were deliberately NOT carried over (a fresh git history was
chosen explicitly). SCISS/SNAP animation variant asset sets exist on disk
but aren't yet wired into direction selection. Touch/mobile input has no
equivalent to the mouse-driven interaction model yet.

Audience / how it's used: single-user interactive toy/demo, deployed as a
static page.

--------------------------------------------------------------------------------
CURRENT STATE

Initial build complete and iterated on through 2026-09-10, pushed to
github.com/LeisHo/DotFlicko (commit 3517a52). Working: mouse-follow entity
with center-pointing rotation (verified via 2 independent geometric test
cases -- mouse directly above/right of center rotates the entity to point
down/left respectively); 8-direction angle bucketing; click-to-play
forward+reverse animation with a configurable peak-frame hold (0-10 extra
frames, dev-tunable); a dev panel exposing Entity (scale, per-direction
Hand Rotation Offset via a dropdown + slider -- 8 independent values that
persist across dropdown switches without needing Save --, anim speed, peak
hold, rotation/position smoothing, angle offset), Background, Ball,
Collision, and Debug groups. A concurrent Claude session built a full ball-
physics/hand-collision system (gravity, wall bounce, 3-capsule finger
collision using a per-frame-annotated skeleton) and a skeleton-annotation
tool (scripts/active/flick-skeleton-annotator.html, source of truth synced
to data/processed/flick-skeleton.json). This session added on top of that:
top-edge ball collision (except during the ball's initial fall into frame);
a ball-proximity auto-trigger for the flick sequence, checked against the
swept collision geometry across the full animation sequence (all 20 frames
per direction), not just one pose; and converted the physics loop to a
fixed 1/60s timestep, verified deterministic via direct testing (2
differently-jittered real-frame-timing sequences produced bit-for-bit
identical ball state when compared at matching fixed-step count).

--------------------------------------------------------------------------------
DECISIONS

- Fresh git history, not the original DickoClicko-derived one -- this repo
  began as a repurposed DickoClicko checkout with hundreds of unrelated
  rope-physics commits attached; the user chose to start DotFlicko's public
  history clean (a single initial commit) rather than carry that history
  into a repo named for a different project.
- Reused DickoClicko's own FLICK animation mechanism directly (per-
  direction frame-set preloading, forward+reverse playback shape) per
  explicit instruction ("we will be using the same animation mechanics as
  DickoClicko") -- not reinvented from scratch.
- Fixed-timestep (1/60s accumulator) physics instead of raw per-real-frame
  delta -- the raw-delta version made 2 runs with an identical, unmoving
  mouse position produce visibly different ball trajectories, since real
  requestAnimationFrame timing is never bit-for-bit identical between runs
  and this sim's collision response is sensitive to that jitter.
- Dev panel settings are localStorage-only, no git-tracked settings log --
  simpler default for a project with no deployment server; see the parent
  workspace CLAUDE.md Sections 12l/12m for when that upgrade is warranted.
- Hand Rotation Offset is per-direction (8 independent values behind one
  dropdown+slider UI, all in the DOM simultaneously with only the selected
  one visible) rather than a single global value or 8 separate visible
  sliders -- lets each of the 8 directions be tuned independently without
  cluttering the panel, and without losing an edit when switching the
  dropdown before hitting Save.

--------------------------------------------------------------------------------
DATA SOURCES

`data/FLICK/2TONED/<direction>/A/` -- 20 hand-pose PNG frames per direction
(the 8 base directions -- Behind, Behind Thumb, Side Thumb, Front Thumb,
Front, Front Pinky, Side Pinky, Behind Pinky -- plus SCISS/SNAP variant sets
not yet wired into the app), authored externally, each direction's own
filename prefix (no shared formula across directions). `data/processed/
flick-skeleton.json` -- per-direction, per-keyframe hand-joint annotations
(knuckle/joint1/joint2/tip, normalized 0-1), produced by
`scripts/active/flick-skeleton-annotator.html` and embedded as a literal in
`index.html` (not fetched at runtime, since `fetch()` of a local relative
resource is blocked under plain `file://`).

--------------------------------------------------------------------------------
KNOWN LIMITATIONS

- No automated test suite -- validation has been manual/live: screenshots,
  and direct function-level testing via the browser console (driving the
  real production functions with controlled inputs, e.g. for the fixed-
  timestep determinism claim and the ball-proximity trigger).
- Mobile/touch input untested -- the interaction model (continuous mouse
  position, click) has no touch equivalent implemented yet.
- Dev panel's Mobile/Landscape tabs show only the built-in "Dev Panel"
  appearance group -- every project-specific setting is shared/rendered
  into the Desktop tab only, by convention (this project's mouse-driven
  interaction has no meaningful Mobile/Landscape equivalent yet).
- SCISS/SNAP animation variant asset sets exist on disk but aren't yet
  wired into direction-selection code -- only the base 8 directions are
  actually used by `DIRECTIONS`/`FLICK_SKELETON`.

--------------------------------------------------------------------------------
NEXT ACTION

See PROJECT_PROGRESS.md's "What's next" section.
