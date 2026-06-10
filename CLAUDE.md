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
- **Verification** is driving the real app via Playwright MCP: navigate **with a cache-buster** (`?v=...` — a stale cached script once made a fresh fix look broken), assert on internals and dispatch events via `browser_evaluate`, screenshot for visual checks. Internals are deliberately script-scope globals (`state`, `render`, `renderSidebar`, `setMode`, `activeApt`, `localToWorld`, `worldToLocal`, `setWallLength`, `partDraft`, `editWall`, and from iteration 4: `partSnap`, `altHeld`, `lenInputDirty`, `undoStack`/`redoStack`/`undo`/`redo`/`pushUndo`, `zoomBy`/`fitAll`/`spaceHeld`, `hover`/`ruler`/`toggleCheatsheet`, `openCanvasInput`/`wallInput`, …) so `browser_evaluate` can reach them. Note: `let`/`const` globals are reachable by bare name in `browser_evaluate` but are **not** `window` properties (so test `typeof state`, not `window.state`).
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
- Wall hit-testing (`wallHitTest`) prefers the **active** apartment, and in `mousedown` a wall hit wins over polygon reselection. Overlapping plans are the whole point of the app — breaking this makes walls unclickable in overlap zones. Clicking a wall in `select` mode (a click without drag-movement) opens an on-canvas length input (`openCanvasInput`) pinned to that wall's dimension label: Enter applies via `setWallLength`, Esc cancels, blur applies; the closing wall (index n−1) stays «авто» read-only (no input). The sidebar «Размеры стен» accordion remains a synced read-through overview (`syncWalls`) — the old focus-into-sidebar transition is gone.
- The on-canvas input (`openCanvasInput`) is one shared `#wallInput` element; its Enter handler calls `e.stopPropagation()` (same double-action guard as `#lenInput`) and each open auto-closes any prior editor. `repositionCanvasInput()` re-pins it to its label every `render()`; `setMode` cancels it. Reuse this opener for every on-canvas dimension edit (wall length, partition segment, offset) — don't build a second input.
- Partition vertices snap (one-shot) to the **active** apartment's own host lines — its footprint walls (closed loop) and its existing partitions — with priority **corner > line > grid**; never to other apartments' geometry and never to the footprint draft. Snap and the «слева»/«справа» offsets are computed in the apartment's **local** space (so they hold under rotation); thresholds are screen-px converted via `state.scale`. Holding **Alt** suppresses line+grid snapping (ortho still applies). The capture is **one-shot**: the point→host-line link is **not** stored (only `partSnap` transient display state), so partitions still do **not** follow later footprint-wall edits. Workflow assumption: size the footprint first, then add partitions.
- Undo/redo (`undoStack`/`redoStack`, memory-only, depth ~50) snapshots apartment **data** + selection. Never hook `commit()` for undo — it also fires on pan/zoom/mode. Call `pushUndo()` immediately *before* a discrete data mutation; wrap continuous gestures (drag, sliders, name typing) in `beginGesture()`/`commitGesture()` so they coalesce to one step. During a draft, Ctrl+Z is point-wise (`undoPoint`/`undoPartPoint`); in a text field it's native (the handler returns without `preventDefault`). Pan/zoom/mode never enter the stack.
- Active-preference extends past walls: in `select` `mousedown` the priority is **vertex (`vertexHitTest`) > wall (`wallHitTest`) > partition segment (`partSegHitTest`) > polygon body**, all ACTIVE-apartment-first, so in an overlap zone you grab the active plan's vertex/wall/partition rather than the topmost. A partition segment label opens the same `openCanvasInput` (rigid tail-shift of the polyline via `setPartSegLength`; no centroid pin — the footprint is untouched).
- Vertex dragging (`drag.type==="vertex"`, select mode) grabs a footprint or partition vertex and wins over the plan-move drag; it reuses `findPartSnap(a, p, skip)` with `vertexSkip` excluding the vertex's own incident segments (no self-snap), then grid. A footprint-vertex drag on a rotated apartment re-pins the world centroid (same `tx,ty` adjust as `setWallLength`); one drag = one undo step (`beginGesture` on mousedown, `commitGesture` on mouseup).
- Offset editing (#9) slides **only** the edited vertex along its host line: clicking an «слева»/«справа» offset label (rebuilt into `offsetLabels[]` each render, hit-tested in `click` before mode logic) opens `openCanvasInput`; applying sets the vertex to `A + dir*L` (or `B − dir*L`). Labels appear for the draft's last point and the hovered partition vertex (`hover.kind==="pvertex"`, which outranks the wall it sits on). One-shot per ADR 0001 — the host link is recomputed at edit time, never stored.
- The tape measure (`ruler`) and the cheatsheet are transient view state, **not** a `state.mode`. The ruler intercepts `svg` clicks before mode logic and creates/saves nothing; Esc priority is cheatsheet → ruler → mode action (each conditional branch returns).
- An unfinished partition draft auto-commits on mode switch (`setMode`) and on apartment switch (`selectApt`) — drawing work is never silently dropped.
