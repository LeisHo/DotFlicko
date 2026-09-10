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
