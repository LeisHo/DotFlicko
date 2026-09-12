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

**Open item:** the reported "click and drag to place entities doesn't work
on mobile" bug was root-caused to a missing `touch-action: none` on
`#scene` (the canvas) — every other draggable element in this file already
had it, the canvas didn't. Without it, a real touchscreen's native gesture
recognizer hijacks a press-drag on the canvas as a pan attempt, firing
`pointercancel` instead of letting the drag complete — plain taps still
worked (not enough movement to trigger the browser's pan detection), which
matches exactly what was reported. Fixed by adding `touch-action: none` to
`#scene`'s CSS, plus a `window` `pointercancel` handler that resets
`pendingSpawnStart`/`spawnPointerId`/`draggingEntity`/`draggingPoint` so a
genuine cancel (OS interruption, edge-swipe nav gesture) can't leave
placement permanently stuck. **Verified:** the full gesture logic (press →
drag → release → `spawnEntityFromLine`) completes correctly end-to-end via
synthetic touch-type `PointerEvent`s in this environment's own browser tool
— but a synthetic dispatch never goes through a real browser's native
touch-gesture recognizer, so this environment cannot reproduce the actual
hijack-and-cancel failure mode itself. **The user should re-test on their
actual phone to confirm the real fix**, the same open-verification caveat
as the mobile-lag item below.

**Note for a new session:** a *different*, concurrent Claude session has
also been actively developing this same `index.html` throughout this
project's history so far — check `git log`/`git status` before assuming
this doc, or any in-progress understanding of the file, is still current.
The Level Maker/Sandbox/Play feature mentioned in earlier versions of
this note is now complete, committed, and pushed (`751f84d`) — see
"Recently completed" below for what it actually covers.

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
- Fixed mobile touch-drag entity placement (missing `touch-action: none`
  on the canvas — see CHANGELOG for the full root cause). Still needs
  the user's own real-hardware retest to confirm.
- Flick Speed Multiplier — per explicit request, the placement line's
  own length (base to aim point) scales each entity's OWN flick
  animation playback speed, multiplicatively on top of the shared Hand
  Anim Speed slider: up to 100% of that direction's own idle-frame
  visible content height (at the current Entity Scale) is 1x, 100%-200%
  ramps linearly to 2x, 200%+ caps at 2x. (This originally scaled the
  ball-collision-kick strength instead — "Flick Intensity" — before
  being explicitly repurposed to animation speed; the collision kick is
  back to using only the global Collision/Flick Intensity slider, no
  per-entity variance.) Recomputed live from each entity's own permanent
  endX/endY (same pattern as its rotation), so a Stop-mode end-dot drag
  updates it immediately. The live spawn/drag preview and Stop-mode
  end-dot adjustment both show a running `speed X.XXx` readout next to
  the existing debug status text. The placement/adjustment line's own
  VISIBLE length is also capped at 2 hand-lengths
  (`MAX_PLACEMENT_LINE_RATIO`) — dragging further just stops the dashed
  line (and the committed entity's own endX/endY, and a Stop-mode
  end-dot drag) from extending any further. **Verified precisely**: 2
  entities placed with a short (1x) and a maximally-long (capped 2x)
  line, both flicks triggered and manually ticked in lockstep — the
  2x entity's own frame index tracked almost exactly double the 1x
  entity's at every sampled tick, and it finished its entire play
  sequence and returned to idle well before the 1x entity did.
- Win/Lose test buttons — clicking Win or Lose transitions EVERY
  currently-placed entity into that direction's own Win (TU) or Lose (MF)
  24-frame sequence: play 1→13, hold (Win/Lose Fist Hold Duration, new
  dev-panel slider), continue 13→24, hold again (Win/Lose Final Hold
  Duration), then play the whole thing in reverse with no holding, back
  to idle. Assets (384 PNGs, 16 direction/type folders' own `FIST`
  subfolder only) copied in from `DICKOCLICKO`'s own
  `data/FLICK/2TONED/<FOLDER>_MF|_TU/FIST` — a genuinely different
  source asset family (1920×912, not 2400×1181), so its own
  centerX/bottomY anchor was independently computed via the same offline
  Node+sharp technique, not reused from the normal flick set. **Verified
  the state-machine timing/frame-boundary logic exactly correct via an
  isolated Node simulation of the extracted formula** (forward1 stops at
  exactly frame index 12 = `_013`, forward2 at exactly index 23 = `_024`,
  each hold lasts exactly its configured duration, reverse has no
  intermediate hold) — real-browser confirmation only got a partial,
  qualitative check (3 distinct, internally-stable image plateaus in the
  right order) because this session's Browser pane was reporting
  `document.hidden === true` throughout testing, which throttles this
  project's own `requestAnimationFrame`/fixed-timestep accumulator (a
  known, previously-documented environment limitation, not a code
  issue) — the user's own click-through in a real, focused browser is
  the first real-time-accurate check.
- Win/Lose Anim Speed — a dedicated dev-panel slider (default 11.7,
  same default/range as Hand Anim Speed) controlling only the Win/Lose
  sequence's forward/reverse frame-advance rate, independent of the
  normal flick's own Hand Anim Speed slider. The 2 hold durations are
  unaffected (real-time, not playback speed).
- Fixed a real reported bug: on a cold page load, the first Win/Lose
  click showed entities flashing in one at a time as each direction's 24
  frames finished loading (Win/Lose assets are lazy-loaded, same as the
  normal flick set) — an entity whose target frame wasn't ready yet drew
  NOTHING at all that tick. Now falls back to the entity's own
  already-cached idle frame while a Win/Lose frame is still loading, so
  it never goes blank. Also now prefetches an entity's own Win AND Lose
  assets the moment it's PLACED, not only once the button is actually
  clicked, giving the network a head start for the common case (place,
  look around, then click Win/Lose). Verified live: an entity remained
  visible through a Win click on a fresh page load, and a newly-placed
  entity's own Lose assets were confirmed fetching over the network with
  zero Win/Lose button ever clicked.
- Win/Lose asset re-syncing from DICKOCLICKO is an ongoing, recurring
  task as the user keeps revising individual directions' own frame art
  there (see `CHANGELOG.txt` for the full history of each round) — each
  round: `diff -rq` the full 16 folders against DOTFLICKO's own copy
  (never assume only the folder(s) the user explicitly names are the
  only ones that changed — one round found 3 extra already-broken
  directions this way, where the OTHER concurrent session had already
  updated the CODE to expect new 48-frame assets but the actual PNG
  files were never copied in; another found 2 directions the user's own
  message hadn't named at all), copy in whatever differs, recompute (not
  assume) that direction's own visible-content anchor. Frame count is
  now genuinely PER-ASSET (most `_TU` Win sequences sit at 48 frames,
  `_MF` Lose sequences and a couple of still-unrevised `_TU` ones stay
  at 24) via `WIN_LOSE_ASSETS[key][type].frameCount`, and win/lose can
  now have genuinely DIFFERENT anchors within the same direction
  (`winLoseVisibleBounds()`'s own per-type override, added when
  front-pinky's own Win art was replaced with visibly different-anchored
  art) — see `CODE_SUMMARY.md`'s own GOTCHAS for both mechanisms.
  (side-pinky's own Win/TU source briefly had a real gap, frames
  028-032 missing from DICKOCLICKO itself — since fixed at the source
  and re-synced; the render()-side idle-frame fallback that would have
  covered a permanently-missing frame is still in place regardless.)
- "Collision Only While Playing" (new Collision-group checkbox, default
  off/unchanged behavior) — when on, an entity's own finger-joint
  capsules stop physically colliding with the ball (position
  correction, bounce, the Flick Intensity kick) whenever that entity
  isn't currently mid-flick. Deliberately leaves the ball-proximity
  auto-trigger check untouched (a separate, independent check in the
  same loop) — an idle entity still needs to detect the ball to ever
  start playing in the first place. Verified via manually-driven,
  fixed-timestep physics ticks (bypassing the browser pane's own
  suspended-while-hidden rAF loop): with the checkbox on, zero collision
  response occurred while `entity.playing` was false, and the first real
  collision kick only landed 3 ticks after `entity.playing` had already
  flipped true via the (unaffected) proximity trigger — confirmed with
  real logged tick numbers, not just a visual check.
- Fixed a real reported bug ("the ball still gets stuck sometimes",
  screenshot of a dense ring of ~16 Win-animating entities): the
  ball-proximity auto-trigger only checked `!entity.playing` before
  starting a normal flick, never whether the entity was already
  mid-Win/Lose — since `entity.playing` is deliberately false
  throughout a Win/Lose sequence, a ball near a Win/Lose-active entity
  could silently ALSO trigger a normal flick underneath it, invisible
  on screen but desyncing the entity's actual collision geometry
  (which follows the accidental flick's own animating frames) from
  what's rendered (which stays on the Win/Lose pose) — with many
  entities doing this independently in a tight cluster, the ball could
  get caught in an unpredictable, shifting collision field. Fixed by
  also checking `!entity.winLoseType`. **Not a settings issue** — no
  slider controlled this; it was a real gap between the auto-trigger
  and the (later-added) Win/Lose system. Verified via manually-driven
  fixed-timestep ticks: a ball parked continuously in a Win-active
  entity's own trigger zone across its entire sequence never flipped
  `entity.playing`, while the same setup against a normal entity still
  triggered correctly — confirms the fix without breaking the original
  mechanism.
- Fixed the deeper "ball still gets stuck sometimes" cause left open by
  the fix above: `updateBallAndCollision()`'s capsule-collision loop
  resolved each hit one at a time, immediately mutating `ball.x/y` — in
  a densely-packed cluster (the reported screenshot: ~16 entities in a
  tight ring), correcting against a later capsule could silently
  reintroduce overlap with an earlier one that the single pass never
  rechecked. Fixed with an iterative position-only relaxation pass
  (`POSITION_RELAXATION_ITERATIONS = 4`) that runs after the existing
  per-capsule loop: it collects every capsule hit that tick
  (`hitCapsules`), then re-checks the ball's current position against
  all of them each pass, pushing out remaining overlap and breaking
  early once clear. Velocity effects (reflection + the collision kick)
  still apply exactly once per capsule, unchanged — only position gets
  the extra passes. Mirrors DickoClicko's own iterative Verlet
  rope-constraint relaxation. Skipped entirely when 0-1 capsules were
  hit, so the common case is unaffected. **Verified via an isolated
  Node.js simulation** reproducing the exact bug (old approach left the
  ball at 0.00 distance from a second capsule after "resolving" the
  first) and the exact fix (new approach converges to clear of both
  capsules within 3 iterations). Committed in isolation (`32668e4`) via
  a scoped `git apply --cached` patch built against HEAD, since another
  concurrent session's own uncommitted Level Maker feature work was
  interleaved in the same working-tree `index.html` at the time —
  confirmed that work was left untouched on disk and unstaged.

- Level System: Level Maker and Sandbox (same placement tools;
  Sandbox never shows Target/the save UI), both available to everyone
  (Level Maker briefly gated its own button on DEV_MODE, reverted per
  direct request), and Play (a Levels dropdown + Play button that loads
  a saved level's geometry read-only). Ball/Rectangle/Target live in
  their own top-right button group, separate from the left-side Start/
  Stop/Delete/Hand/Sandbox/Level Maker/Win/Lose group (also per direct
  request). Place/resize/rotate wall-floor and target rectangles via 6
  handles (move/4-corner-resize/rotate), same rotation-aware local-space
  hit-testing `hitTestEntitySprite` already used. New circle-vs-rotated-rect
  collision reuses the
  existing finger-capsule reflection formula unchanged. A Target rect
  is Instant Hit or Settle Duration (per-target choice) and auto-
  triggers Win; every ball falling off the bottom (no respawn during an
  active level) auto-triggers Lose once ALL balls are gone. Also added:
  a Ball-to-Ball Collision toggle (default off) and a per-level Max
  Hands placement budget with a live "Hands: N/infinity" HUD. Levels
  persist through the same 3-tier pipeline Scenes already uses. See
  `CHANGELOG.txt`'s own entry for the full verification list and one
  real bug caught+fixed during implementation (2 dev-panel labels
  sharing a row corrupted each other's text).

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
this); touch/mobile input support; actually authoring real levels with
the new Level Maker (the system itself is built and verified, but no
levels have been designed/saved for real players yet); mobile touch
support specifically for the new rectangle handles (verified via
synthetic PointerEvents only, same open-verification caveat as the
mobile items above).

## Open questions / blockers

None currently open.
