# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file HTML5 Canvas wilderness-survival game prototype (`index.html`). No build step, no package manager, no dependencies — it's plain HTML/CSS/JS and runs by opening the file directly in a browser.

## Running / testing

- Open `index.html` directly in a browser (`open index.html` on macOS), or reload the tab after edits.
- There is no build, lint, or test tooling in this repo. Verify changes by playing the game in a browser and by sanity-checking the script parses, e.g.:
  ```
  node -e "
  const fs = require('fs');
  const script = fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1];
  new Function(script);
  "
  ```
  (This only checks for syntax errors — it does not execute game logic meaningfully without a real DOM/canvas.)

## Architecture

Everything — markup, styles, and game logic — lives in `index.html` as one `<script>` block. There is no module system; all state is top-level `const`/`let` in a single scope.

The script is organized into clearly commented sections, in this order:
1. **config** — tuning constants (rates, ranges, speeds, cooldowns)
2. **state** — mutable game state: `stats` (hunger/thirst/energy/temperature/health), inventory flags (`wood`, `hasWeapon`, `fenceCount`), entity arrays (`trees`, `bushes`, `medkits`, `fencePickups`, `fenceSlots`, `monsters`, `toasts`)
3. **input** — `keydown`/`keyup` listeners populate a `keys` Set for continuous movement; `E` and `F` are edge-triggered actions (`tryInteract()`, `tryAttack()`)
4. **update** — `update(dt)` mutates all state for one frame: resource respawn timers, campfire fuel, cooldowns, monster spawning/movement/collision, player movement, temperature drift, hunger/thirst/energy drain, stat clamping, game-over checks
5. **render** — `render()` draws the canvas frame and syncs DOM HUD elements (bar widths, text labels) each frame; drawing order is back-to-front (background → pond → static resources → fence slots → campfire → items → monsters → player → prompts/toasts → night overlay)
6. **loop** — `requestAnimationFrame` loop computing `dt`, calling `update` then `render`

Key mechanics to know before editing:
- **Interaction is proximity + priority based**: `tryInteract()` and `nearestInteractable()` both walk the same ordered list of interactables (weapon → fence pickups → fence placement → campfire → trees → bushes → medkits → pond) and must be kept in sync — the prompt text shown by `nearestInteractable()` has to match what `tryInteract()` actually does at that distance.
- **Day/night** is a single `time` accumulator; `ambientTemp()` derives a `daylight` value (0..1) from `time % DAY_LENGTH` via a sine wave, which drives ambient temperature, sky color, and monster spawn gating (monsters only spawn when `daylight <= 0.5`).
- **Fences are physical obstacles**: `fenceSlots` are fixed positions around the campfire; monster movement in `update()` computes a candidate new position each frame and pushes it back outside `FENCE_BLOCK_RADIUS` of any filled slot before committing it — this is the only collision logic in the game (player and resources have no collision).
- Resource entities (`trees`, `bushes`, `medkits`) share a `{available, timer}` respawn pattern; one-time collectibles (`weaponItem`, `fencePickups`) use a `{picked}` flag instead and never respawn.
