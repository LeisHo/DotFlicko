# DOTFLICKO — Project Conventions

<One-line description of what this project is — fill in. See
`docs/PROJECT_SUMMARY.md` for the fuller objective/scope, `docs/CODE_SUMMARY.md`
for how the code is structured, once those are filled in.>

The dev panel follows the workspace-wide standard in the parent `CLAUDE.md`
§12 — this file only covers what's specific to this project, not a
restatement of §12 itself.

## File map

<Fill in once the project's file layout exists — what's in `src/` vs
`scripts/active` vs `scripts/archive` vs `data/raw`, and what's load-bearing
vs safe-to-ignore. State plainly if this project is a deliberate single-file
architecture exception (CLAUDE.md §11), same as Clicko / Quiz Game / the
prior Dicko Clicko build.>

## Untouchable systems

None formally designated yet.

## Dev-panel prompt shorthand (how the user specs dev controls)

Uses the workspace-standard `*DC*`/`*D*` notation (parent `CLAUDE.md` §12g).
A bare `*D*` with no group context goes into whichever existing collapsible
group fits best (per §12g); create a new group only if none fit.

## Dev-panel behavior (project-specific judgment calls under §12)

<Fill in as real decisions get made for this project — e.g. which settings
are shared vs. device-split between the Desktop/Mobile/Landscape tabs (§12f),
what units position/size sliders use (§12a), the shared X/Y origin convention
(§12k), where Save Settings actually writes to (§12l: localStorage baseline,
or the optional git-tracked settings-log upgrade).>

## Gotchas

<Real bugs already hit and fixed, so a later session doesn't rediscover
them. One bullet per gotcha, concrete enough to actually prevent the
mistake.>

- A PostToolUse:Edit hook auto-opens the just-edited `index.html` as a
  `data:` URL preview in the Browser pane. That preview LOOKS like a
  normally-loaded page but its origin can't resolve any relative asset
  path (image `src`/`fetch` calls to `data/...` silently fail —
  `naturalWidth` stays 0, no console error). Before trusting a
  browser-pane verification that touches images/assets, confirm
  `location.href` starts with `http://localhost:<port>`, not `data:` —
  if it's a stale data: preview, `navigate` to the real
  `static`/`static-alt`/`static-alt2` server URL (`.claude/launch.json`)
  first and re-verify there.
- This project's data assets sometimes get saved into the sibling
  `DICKOCLICKO` project by mistake (same `data/FLICK/2TONED/<...>/FIST/`
  folder layout, easy to confuse). If a reported frame-count/asset
  change doesn't match what's on disk here, check
  `J:\CLAUDE\PROJECTS\DICKOCLICKO\data\FLICK\2TONED\` for the real files
  before assuming the user is wrong — confirmed real prior instance:
  BehindThumb_TU/FrontThumb_TU/BehindPinky_TU's 48-frame replacements
  (2026-09-11).
