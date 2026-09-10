# DOTFLICKO — <one-line tagline>

<1-3 sentence description: what this project is, what it does. If it's a
prototype/research project rather than a production app, say so plainly here
(see DOGGO's README for an example: "This is not a production app.").>

See `docs/PROJECT_SUMMARY.md` for the current state (objective, scope,
current state, recent decisions, known limitations, next action — the part
that changes often) and `docs/PROJECT_PROGRESS.md` for what's being worked
on right now. This README stays a short pointer, not a duplicate of either —
don't let real content drift into this file instead of those.

## How to run it

Single-file, no build step: open `index.html` directly, or serve the folder
with any static file server. Append `?dev=1` (or run on localhost/127.0.0.1,
or open as a bare local `file://`) to see the dev panel.

### Dev panel Save Settings (git-tracked, ported from Clicko/DickoClicko)

Save Settings/Reset try 3 tiers in order (see the big comment above
`saveDevPanelSettings()`/`loadDevPanelSettingsAsync()` in `index.html` for
the full picture):
1. **This Vercel API** (`api/save-settings.js`) — works from any device,
   including a phone with no filesystem access of its own. **Requires the
   one-time setup below before it does anything.**
2. **File System Access API** — a native save-file picker, direct local
   disk write. Desktop Chromium only, and only meaningful against a real
   local checkout (not a random visitor's disk on the deployed site).
3. **Local save prompt + this tab's own session cache, and the old
   localStorage key** — last resort when opened as a bare local file with
   no server at all, or neither tier above is available.

Until the two env vars below are set on the Vercel project, tier 1 does
nothing (Save silently falls through to tier 2 or 3 instead) — that's
expected until this is set up, not a sign of a bug:

1. **`GITHUB_TOKEN`** — a GitHub fine-grained personal access token,
   scoped to only this repo (`LeisHo/DotFlicko`), with **Contents: Read
   and write** permission and nothing else. Create one at
   github.com → Settings → Developer settings → Personal access tokens →
   Fine-grained tokens.
2. **`DEV_PANEL_SAVE_SECRET`** — an anti-abuse shared token (not a real
   secret — it also lives in the page's own client-side source, same as
   any other value there). Set it to `PkrbMti03M6xm3FEThYXa8gGW_08BOGj`
   (the value already embedded in `index.html`'s `DEV_PANEL_SAVE_SECRET`
   constant — the same value Clicko/DickoClicko's own Vercel projects use,
   since this is one workspace-wide shared token, not a per-project one)
   — or change both to a new value together if you'd rather generate your
   own.

Add both under the Vercel project → Settings → Environment Variables,
then redeploy. `GITHUB_REPO`, `GITHUB_BRANCH`, and `SETTINGS_FILE_PATH`
are optional overrides (see `api/save-settings.js`) — the defaults
already match this repo (`LeisHo/DotFlicko`, branch `main`).

## Project structure

```
DOTFLICKO/
├── index.html               <the entire app — markup, styles, and JS inline>
├── api/                     <save-settings.js — Vercel serverless function, see above>
├── data/
│   ├── FLICK/2TONED/         <the 8 directions' frame PNG sequences>
│   └── processed/            <flick-skeleton.json, and the git-tracked
│                              settings log Save Settings writes to>
├── scripts/active/           <flick-skeleton-annotator.html>
├── docs/                    <README (pointer only — this file), PROJECT_SUMMARY.md,
│                             CODE_SUMMARY.md, PROJECT_PROGRESS.md, CHANGELOG.txt>
```

This is otherwise a single-file project (see project `CLAUDE.md` for why)
— `api/` is the one small, necessary exception (a Vercel serverless
function can't live inside `index.html`); the rest of the workspace-standard
scaffold (`src/`, `scripts/archive/`, `logs/`, `results/`, `tests/`) mostly
stays unused for this project.

## Known limitations

See `docs/PROJECT_SUMMARY.md`'s Known Limitations section for the current,
maintained list — not duplicated here to avoid drift between two copies of
the same information.

## Roadmap

No formal roadmap is tracked separately. See `docs/PROJECT_PROGRESS.md` for
what's currently being worked on and what's next, and `docs/CHANGELOG.txt`
for the full history of what's been built.
