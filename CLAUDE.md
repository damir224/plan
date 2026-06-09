# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file web app for comparing 2–3 apartment floor plans by overlaying them at real (metre) scale. The entire app — HTML, CSS, JS — is `index.html` (one inline `<script>`, sectioned with `// ====` banners). No build step, no dependencies, no package.json, no test suite. UI text is Russian — keep new UI strings Russian.

The project follows the local `prototype` skill philosophy (`.claude/skills/prototype/`, gitignored): minimal, runnable, state visible on screen, no premature abstractions. Two deliberate deviations: localStorage persistence and real-browser verification.

`spec/` holds the user's reference floor-plan images. It is local-only and gitignored — never commit it (the repo is public).

## Commands

- **Run locally:** `python3 -m http.server 8731` in the repo root, then open `http://localhost:8731/index.html`. Don't test over `file://` — the Playwright MCP blocks it and Safari blocks localStorage on it.
- **Syntax check** (closest thing to a test runner here):
  `python3 -c "import re;open('/tmp/app.js','w').write(re.search(r'<script>\n(.*)</script>',open('index.html').read(),re.S).group(1))" && node --check /tmp/app.js`
- **Verification** is driving the real app via Playwright MCP: navigate **with a cache-buster** (`?v=...` — a stale cached script once made a fresh fix look broken), assert on internals and dispatch events via `browser_evaluate`, screenshot for visual checks. Internals are deliberately script-scope globals (`state`, `render`, `renderSidebar`, `setMode`, `activeApt`, `localToWorld`, `worldToLocal`, `setWallLength`, `partDraft`, `editWall`, …) so `browser_evaluate` can reach them.
- **Test through real dispatched events, not just model calls.** Every serious bug so far (overlap wall-pick hijack, Enter double-action) passed model-level tests and only reproduced through full `mousedown→mousemove→mouseup→click` sequences or a `keydown` on the actually-focused input. Build the discriminating case (overlapping apartments, rotated apartment, focused field).
- **Before committing:** `rm -rf .playwright-mcp *.png` (Playwright drops artifacts into the repo root).

## Deploy

Push to `main` = deploy. GitHub Pages serves the repo root of `damir224/plan`; live at https://damir224.github.io/plan/.

- **Commit identity:** always `-c user.name="damir224" -c user.email="47707728+damir224@users.noreply.github.com"`. The repo is public; a personal email leaked into history once and had to be amended out.
- **Verify a deploy by commit SHA, not status alone:** poll `gh api repos/damir224/plan/pages/builds/latest` until `.commit` equals the pushed HEAD **and** `.status == "built"`, then curl the live URL for 200 (or grep it for a string unique to the new version). `status=built` can coexist with a stale CDN for 1–2 min.
- Pages serves `index.html` with ~10 min cache — after deploying, tell the user to hard-reload (Cmd+Shift+R).

## Architecture

Data model — `state.apartments[]`, persisted (debounced 250 ms) to `localStorage['plann_overlay_v1']`:

```
{ id, name, color, visible, opacity,
  points: [{x,y}],                 // outer footprint: closed polygon, LOCAL metres
  closed,                          // footprint finished?
  tx, ty, rot,                     // overlay transform: translation (m) + rotation (deg)
  partitions: [{points:[{x,y}]}] } // interior walls: open polylines, LOCAL metres
```

**Coordinate pipeline** (keep centralized — never inline ad-hoc transform math):
`local (m) → rotate about centroid(points) by rot, translate (tx,ty) → world (m) → ×scale, +pan → screen (px)`, inverses `worldToLocal` / `screenToWorld`. Footprints are drawn at identity transform (world == local). Partition input goes screen→world→**local**, and snap/ortho run in local space so partitions follow a rotated apartment's own axes. Lengths computed from local points are transform-invariant.

**Modes:** `state.mode` ∈ `draw` (footprint) / `partition` (interior walls) / `select` (move, rotate, click-wall-to-edit). The shared svg `mousedown`/`click`/`keydown` handlers branch on mode; a new interaction gets its own explicit branch. Mind cross-talk on shared inputs: arrow keys, Enter, and the one `#lenInput` length field.

**Rendering:** full SVG rebuild per frame (`render()`, rAF-batched via `scheduleRender()`). Sidebar cards are fully rebuilt by `renderSidebar()`; updates that must survive without losing focus/accordion state go through in-place sync (`syncWalls()`, the `openWalls` set).

**Persistence compat:** read newer fields defensively (`a.partitions || []`) — returning users load older saved shapes, and this upgrade path is expected to keep working. localStorage is per-origin: localhost, the deployed site, and `file://` are three separate stores; Export/Import JSON is the bridge.

## Invariants (each one guards a fixed bug)

- `#lenInput`'s Enter handler calls `e.stopPropagation()` before `blur()`: blur changes `document.activeElement`, and without stopPropagation the window keydown handler would *also* fire `closePoly()`/`finishPartition()` — a double action. Don't remove it.
- The rotation pivot is `centroid(points)`, recomputed every frame. Any mutation of `points` on a rotated apartment must re-pin the world centroid by adjusting `tx,ty` (see `setWallLength`) or the whole plan visibly lurches.
- The closing wall of a footprint is **derived** (rendered dashed + «авто», read-only in the accordion): a closed polygon has n−1 independently settable walls; `setWallLength(k)` rigid-shifts the tail `k+1..n−1`.
- Wall hit-testing (`wallHitTest`) prefers the **active** apartment, and in `mousedown` a wall hit wins over polygon reselection. Overlapping plans are the whole point of the app — breaking this makes walls unclickable in overlap zones.
- Partition endpoints snap to the grid, not to walls, and partitions do **not** follow later footprint-wall edits (they keep their local coords). Workflow assumption: size the footprint first, then add partitions.
- An unfinished partition draft auto-commits on mode switch (`setMode`) and on apartment switch (`selectApt`) — drawing work is never silently dropped.
