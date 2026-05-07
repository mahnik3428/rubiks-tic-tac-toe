# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file browser game that fuses a 3x3 Rubik's Cube with Tic-Tac-Toe. The entire game lives in `index.html`: HTML + inline CSS + an inline ES module that imports Three.js from CDN via an `importmap`. There is no build, no `package.json`, no tests, no lint config.

GitHub repo: `https://github.com/mahnik3428/rubiks-tic-tac-toe`

## Game rules (drives the architecture)

- Two players alternate. On each round:
  1. The **active player places** an X or O on any empty sticker.
  2. The **opponent rotates** — and the rotation must include the cubie that the mark was just placed on. This restricts the rotator to **6 of the 18 moves**.
- Win check runs **after the rotation animation completes**, never after placement. A placement that creates 3-in-a-row but is then disrupted by the rotator's chosen move does not win.
- After place + rotate, the rotator becomes the next placer (so both players take turns at both roles).

If you change rule semantics, the same change usually needs to land in three places: `placeMark`, `doMove`, and the AI helpers (`aiPlace`, `aiRotate`).

## Architecture (the parts you can't tell from a quick scan)

**Source-of-truth is the Three.js scene, not a parallel logical model.** Cubie positions and quaternions are mutated in place; `checkWin` and the AI both read world transforms back out of the scene. There is no separate sticker-permutation array.

- **27 cubies**, each a `THREE.Group` at integer `(x,y,z) ∈ {-1,0,1}³`. Each cubie owns its own `MeshStandardMaterial` (per-cubie, *not* shared) so emissive can be set individually for highlighting.
- **54 stickers** are `PlaneGeometry` meshes added as children of their parent cubie. They carry the mark in `userData.mark`. Marks (X/O) are rendered as a small child plane added to the sticker, textured from a shared `CanvasTexture`.
- **Move animation** (`performMove`): reparent the 9 layer cubies into a temporary `pivot` group with `pivot.attach(c)`, animate `pivot.rotation[axis]` over ~300 ms, then on completion `cubeRoot.attach(c)` to bake the rotation back into each cubie's own transform and snap `position` to integers (cleans up float drift).
- **Layer detection** (`inLayer`) compares cubie's *current* `position` against the layer value. After bakes, positions are integer-clean, so equality-with-tolerance works.
- **AI simulation** uses `applyMoveInstantAndSnap` / `restoreSnap` — same rotation logic as the animator but instant, with a per-call snapshot of the affected cubies' position+quaternion so it can be reverted. **After every mutate/restore, `cubeRoot.updateMatrixWorld(true)` is called** so subsequent `getWorldQuaternion` / `getWorldPosition` reads see the new state. Forgetting this is the easiest way to introduce subtle bugs.
- **Win detection** (`checkWinForSymbol`): for each of 6 face directions, find the 9 stickers whose world-space normal matches (`dot > 0.99`), project world position onto the face's two tangent axes (the `uAxis` / `vAxis` of `FACE_DIRS`) to recover a 3×3 grid, check the 8 lines in `WIN_LINES`.

**State machine** lives in `STATES`: `PLACING → ROTATING → ANIMATING → (PLACING with placer flipped, or GAME_OVER)`. `currentPlayer` is the **placer**; the **rotator** is `3 - currentPlayer`. `activeCubie` and `validMoves` are populated by `placeMark` and cleared by `doMove`.

**Move conventions** (`MOVES` table): 12 face turns + 6 slice quarter turns. Sign of `angle` follows the right-hand rule around `+axis` (e.g. `R = -π/2 around +X`, `D = +π/2 around +Y`). Slices `M/E/S` follow `L/D/F` directions respectively.

**AI** has two roles, dispatched by `aiTurn`:
- `aiPlace()` — when AI is the placer. Evaluates each empty sticker by considering the rotator (opponent)'s best response: simulates each of the 6 valid rotations and asks "after the worst rotation, who wins?" Prefers placements where every valid rotation wins for AI; then placements that block the same threat from the human; then any non-losing placement.
- `aiRotate()` — when AI is the rotator. Picks a winning rotation if available, else any rotation that doesn't let the opponent win, else random.

AI is auto-triggered from `placeMark` (when the *rotator* is AI) and from `doMove` (when the next *placer* is AI). `aiBusy` blocks human input during AI turns and is reset before each `await doMove(...)` so the next AI hook can fire.

**Highlighting layers** uses emissive on the per-cubie body + per-sticker materials. Three highlight states stack: hover (cyan, while pointer is on a move button), active cubie (orange, while in `ROTATING`), neither. `clearHover` re-applies the active highlight when it removes the hover one — easy to forget when adding new highlight types.

## Three.js gotchas to avoid regressing

- Per-cubie / per-sticker materials are required. Don't refactor them back to a shared material; emissive highlighting will break.
- Use `Object3D.attach()` (not `.add()`) when reparenting cubies during rotation — `.add()` discards world transform.
- After programmatically setting `position` / `quaternion`, call `cubeRoot.updateMatrixWorld(true)` before any `getWorld*` reads. The animator and AI sim both rely on this.
- Hover hints (`arrowMeshes`) use `depthTest: false` and `renderOrder: 999` so they're visible from any camera angle. Don't "fix" this.

## Mobile responsiveness

**`fitCamera()`** runs on startup and on every `resize` event. It computes how much of the viewport is actually visible between the top bar and the bottom button tray, then sets camera distance and `controls.target.y` so the cube fills and centers that region:

```
topUI    = height of the top overlay in px  (varies by breakpoint)
bottomUI = height of the bottom overlay in px
visibleH = innerHeight - topUI - bottomUI   (clamped to ≥ 120)
cubeRadius = 2.6
dV = cubeRadius * innerHeight / (visibleH * tan(fov/2))   // distance to fit vertically
dH = cubeRadius * innerHeight / (innerWidth * tan(fov/2))  // distance to fit horizontally
targetD = max(dV, dH) * 1.18
controls.target.y = (pxOffset / innerHeight) * worldH      // shift orbit target up/down
```

If you change the top/bottom UI heights (e.g. add a new row of buttons), update the `topUI`/`bottomUI` constants inside `fitCamera` for each relevant breakpoint or the cube will drift out of center.

**CSS breakpoints:**
- `@media (max-width: 600px)` — portrait mobile: stacks the top-bar content vertically, reduces button size, compacts control strip.
- `@media (max-height: 500px)` — landscape phones: makes the top bar a single compact row, hides the "Player" label text, compacts the button panel.

**Touch:** All interactive elements (`button`, `.toggle`, `#new-game`) carry `touch-action: manipulation` to eliminate the 300 ms iOS double-tap delay. The canvas itself does not need this because OrbitControls handles touch natively.

## Local dev server

`.claude/launch.json` configures a local static server:

```json
{ "name": "rubiks-ttt", "runtimeExecutable": "npx",
  "runtimeArgs": ["--yes", "http-server", "-p", "8765", "-c-1", "--silent"],
  "port": 8765 }
```

This is used by the Claude Preview MCP (`preview_start("rubiks-ttt")`) for screenshot-based UI testing. You can also run `npx --yes http-server -p 8765 -c-1` manually and open `http://localhost:8765`.

## Run / verify

- Open `index.html` in any modern browser, or use the Launch preview panel / local dev server (see above).
- No automated tests. Verification is manual:
  - **Desktop**: place a mark, confirm the active cubie glows orange and only 6 buttons are enabled, hover a button to see the cyan layer + arrow hint, trigger the rotation, watch win check fire after animation.
  - **Mobile portrait** (`max-width: 600px`): cube should fill the space between the top bar and button strip; no scrolling needed.
  - **Mobile landscape** (`max-height: 500px`): top bar compacts to a single row; buttons remain reachable; cube stays centered in the remaining vertical space.
  - **Touch**: a finger tap should place a mark; a finger drag should orbit the cube, not place.
- Toggling **AI opponent (Player 2)** in the top-right makes the AI play both roles for Player 2 (rotates after the human places, then places, then waits for the human to rotate).
