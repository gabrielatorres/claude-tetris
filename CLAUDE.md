# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the game

There is no build step, no install, no test suite and no linter. Do not add a
`package.json`, a bundler or a dependency unless explicitly asked.

```bash
start index.html            # Windows; `open` on macOS, `xdg-open` on Linux
python3 -m http.server 8000 # or: npx serve .   (recommended, then open localhost:8000)
```

Verification is manual: load the page and play. Since there are no tests, "run a single
test" does not apply -- check changes by exercising the affected mechanic in the browser
and watching the devtools console.

## Architecture

Four files, all loaded directly by the browser: `index.html` (DOM + two canvases),
`style.css` (dark arcade theme), `game.js` (all logic), `README.md` (Spanish docs).

`game.js` is a single top-level script -- no modules, no IIFE, no bundler. Every mutable
value lives in one module-level declaration at `game.js:43` (`board, current, next, score,
lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId`). `init()`
(`game.js:259`) is both the bootstrap and the restart path: it runs once at load and again
from the Reiniciar button, so **any new piece of state must be reset inside `init()`** or
it will leak from one game into the next.

### Pieces: type, color and cell value are the same integer

`COLORS` and `PIECES` (`game.js:7-27`) are parallel arrays indexed 1-7 with `null` at
index 0, and each piece's matrix is filled with *its own index* -- the T piece is made of
`3`s, the L piece of `7`s. `merge()` copies those integers straight into `board`, and
`drawBlock()` treats `0` as empty and skips rendering it.

So adding or recoloring a piece is never a one-line change. It touches `COLORS`, `PIECES`,
the digits *inside* that piece's shape matrix, and the `Math.random() * 7` in
`randomPiece()` (`game.js:50`), which must stay in sync with the array length.

### Canvas sizes are hardcoded and must match the constants

`#board` is `width="300" height="600"` (`index.html:12`) and must remain
`COLS * BLOCK` by `ROWS * BLOCK`. `#next-canvas` is `120x120` (`index.html:30`), which
assumes the 4x4 layout `drawNext()` centers into at `NB = 30`. Changing `COLS`, `ROWS` or
`BLOCK` in `game.js` without editing `index.html` produces a cropped or stretched board
with no error.

### DOM contract

`game.js:31-41` looks up every element by id at the top level, with no `DOMContentLoaded`
guard -- this works only because `<script src="game.js">` sits at the end of `<body>`.
The required ids are: `board`, `next-canvas`, `score`, `lines`, `level`, `overlay`,
`overlay-title`, `overlay-score`, `restart-btn`, `theme-toggle`, `theme-label`. Renaming one in `index.html` yields
`null` at load and throws later, when a handler first touches it.

A single `#overlay` element serves both states: `togglePause()` and `endGame()` swap its
title/score text and toggle the `.hidden` class.

### Game loop and collision

`loop()` is `requestAnimationFrame`-driven, accumulating frame `dt` into `dropAccum` and
dropping one row when it exceeds `dropInterval` (`dropAccum` is reset to `0`, not
decremented by the interval). Pausing cancels the pending frame; `togglePause()` re-primes
`lastTime` before restarting, so time spent paused is not banked as drop time.

`collide(shape, ox, oy)` reads the global `board` but takes the *candidate* position as
arguments. It is the single predicate behind horizontal movement, `tryRotate()` wall kicks
(offsets `[0, -1, 1, -2, 2]`), the `ghostY()` landing projection, and the spawn-time
game-over test in `spawn()`. New movement mechanics should go through it rather than
re-implementing bounds checks.

## Conventions

`'use strict'`, ES6+ browser JavaScript with no transpiler, so only use APIs that run
natively in a modern browser. User-facing strings, code comments and the README are in
Spanish -- keep new UI text in Spanish.
