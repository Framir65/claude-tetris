# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-page Tetris implementation in vanilla JavaScript (HTML5 Canvas + CSS). No dependencies, no build step, no package.json — just `index.html`, `style.css`, and `game.js`.

## Running the game

There is no build/lint/test tooling. To run:

```bash
open index.html                 # macOS, works directly (file://)
python3 -m http.server 8000     # or serve locally, then visit http://localhost:8000
```

There are no automated tests. Verify changes by opening the game in a browser and playing it (movement, rotation, soft/hard drop, pause, line clears, game over/restart).

## Architecture

Everything lives in `game.js` as top-level module-scope state and functions (no classes, no build-time modules). The whole game is driven by a single `requestAnimationFrame` loop plus a `keydown` listener.

- **Board model**: `board` is a `ROWS × COLS` (20×10) matrix of numbers — `0` for empty, `1–8` as a color index identifying which piece type occupies that cell (see `COLORS`/`PIECES`). Index `8` is the Nut piece.
- **Pieces**: 7 standard tetrominoes plus a challenge piece, the **Nut** (`NUT = 8`, `PIECES[8] = [[8,8,8],[8,0,8],[8,8,8]]`) — a 3×3 ring with an empty center that no other piece can ever fill, so each Nut placed permanently wastes a row (it still shifts down as lower rows clear, it just never completes). As compensation, clearing a line that contains any Nut cells scores that clear at ×2 (see `clearLines`). `current` and `next` are piece objects `{ type, shape, x, y }`. Rotation (`rotateCW`) transposes + reverses rows; `tryRotate` applies `rotateCW` then attempts a sequence of wall-kick offsets (`[0, -1, 1, -2, 2]`) until one doesn't collide — the Nut's ring is rotationally symmetric so rotation is a visual no-op for it.
- **Collision** (`collide`): the single source of truth for whether a shape at a given offset is legal (out of bounds or overlapping locked cells). Used by movement, rotation, ghost-piece projection, and spawn (to detect game over).
- **Nut hole rendering**: the Nut's center cell is `0` in `PIECES[8]`, so it's already empty by construction — `drawBlock` skips falsy cells (`if (!colorIndex) return;`), which renders as a plain square hole with no extra logic. `drawMatrix(context, matrix, ox, oy, size, alpha)` wraps the per-cell loop + `drawBlock` call and is the single call used for the board, the ghost, the current piece, and the next-piece preview.
- **Game loop** (`loop`): accumulates elapsed time each animation frame; once `dropAccum` exceeds `dropInterval`, the piece drops one row (or locks if it can't). `draw()` renders grid → locked board → ghost piece → current piece, in that order, every frame.
- **Locking/scoring flow**: `lockPiece()` → `merge()` (bake piece into `board`) → `clearLines()` (scan bottom-up, splice full rows, unshift empty rows at top, score via `LINE_SCORES` table × `level`, doubled if any cleared row contained a Nut cell) → `spawn()` (promote `next` to `current`, generate new `next`, check game-over collision).
- **Level/speed**: level increments every 10 cleared lines; `dropInterval = max(100, 1000 - (level-1) * 90)`.
- **Ghost piece**: `ghostY()` projects `current` straight down via repeated `collide` checks; drawn at low alpha before the real piece.
- All DOM/canvas elements are grabbed once at top of file by ID (`board`, `next-canvas`, `score`, `lines`, `level`, `overlay`, etc.) — `index.html` and `game.js` are tightly coupled by these element IDs.

## Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK` (cell px size), `COLORS`, `PIECES`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `<canvas id="board">` `width`/`height` in `index.html` to match (`COLS×BLOCK` by `ROWS×BLOCK`).

Primary docs and gameplay/control reference is in `README.md` (in Spanish).
