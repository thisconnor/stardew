# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A self-contained, single-file pixel-art browser game built for a real-world marriage proposal. "Mia" (the player) explores a Stardew Valley-style farm completing tasks, culminating in Connor walking in to ask her to be his girlfriend. The entire game lives in `index.html` (~382KB), which embeds base64-encoded audio, JPEG photos, and all CSS/JS inline. There is no build system, no package manager, and no external dependencies beyond a Google Fonts stylesheet.

## Running Locally

Open `index.html` directly in a browser, or serve it with any static server to avoid potential CORS issues with the embedded `data:` URIs:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

There are no tests, no linting tools, and no CI configuration.

## Architecture

Everything is in one `<script>` block near the bottom of `index.html`. The sections in order are:

1. **Audio** — `AUD` object holds base64-encoded MP3s (`bgm`, `heal`, `water`, `chest`, `throw`). `startBGM()` plays looping background music on first tap. `playSFX(key, vol)` plays one-shot effects.

2. **Canvas setup** — Virtual resolution `GW=288, GH=480` (game pixels). `sc` is the integer/fractional scale factor computed by `resize()` to fill the viewport. `s2g(sx, sy)` converts screen pixels back to game coordinates. All drawing calls multiply coordinates by `sc`.

3. **Drawing primitives** — Five helpers used everywhere:
   - `px(x, y, w, h, color)` / `rect` (alias) — filled rectangle in game coords
   - `circ(x, y, r, color)` — filled circle
   - `ellip(x, y, rx, ry, color)` — filled ellipse
   - `txt(str, x, y, size, color, align)` — "Press Start 2P" pixel font text

4. **Seeded RNG** — `srand(seed)` + `rng()` provide a deterministic LCG. Called before drawing each tile so tile variation is consistent frame-to-frame without storing per-tile state.

5. **Color palette** — `P` object (e.g. `P.grass`, `P.wood`, `P.skin`). Always reference palette keys rather than raw hex strings when drawing new elements, so color changes propagate everywhere.

6. **Tilemap** — `map[y][x]` is a 2D array of tile type integers:
   - `-1` sky (not drawn), `0` dirt, `1` grass, `2` stone path, `3` dry soil, `4` wet soil, `5` water
   - Tile size is `T=16` game pixels; grid is `MW × MH` tiles.
   - `drawTile(tx, ty)` renders one tile with seeded variation.

7. **Game objects** — Each interactable is a plain object (`signs[]`, `crops[]`, `hutch`, `chest`, `mailbox`, `flowerBush`) with a `draw*` function. Highlight pulse animations use `Math.sin(Date.now()/300)` and dashed `strokeRect`.

8. **Characters** — `drawChar(x, y, isCon, frame, bouquet)` renders both Mia (`isCon=false`) and Connor (`isCon=true`) from the same function, differentiated by shirt/hair colors from the palette.

9. **Scene system** — `scene` string drives `render()` and `update()`:
   - `'title'` → `'wake'` → `'explore'` → `'walk'` → `'ask'` → `'celeb'` or `'reject'`
   - `fadeD` / `fade` handle black fade transitions between scenes.

10. **Task progression** — `task` string gates which objects are interactive:
    - `'none'` → `'water'` (read crops sign) → `'feed'` (water all crops) → `'chest'` (feed bunnies) → done
    - HUD hint text is driven by the current task.

11. **UI overlays** — HTML `<div>` panels (`#ui`) float over the canvas at `z-index:20`. `showOv(id)` / `hideOv(id)` toggle `.show` class. The `uiOn` flag blocks canvas tap events while any overlay is visible.

12. **Input** — `handleTap(gx, gy)` (game coordinates) is called from both `click` and `touchstart` listeners. `walkTo(tx, ty, cb)` moves the player then fires a callback when they arrive (polled via `setInterval`).

13. **Game loop** — `loop()` calls `update()` then `render()` via `requestAnimationFrame`. `timer` increments each frame and drives scene transitions.

## Customization Points

- **Photos**: The `data:image/jpeg;base64,...` strings inside overlay `<div>` elements are the embedded couple photos. Replace the base64 to swap images.
- **Signs**: The `signs[]` array near the tilemap section holds all narrative text. Edit `msg` strings to change the story.
- **Proposal text**: The dialog box text is set in `update()` inside the `scene==='walk'` block: `document.getElementById('dlg-text').textContent = '...'`.
- **Celebration screen**: `#ov-celeb` overlay contains the final message. Edit its HTML directly.
- **OG/Twitter meta tags**: Lines 13–23 in `<head>` — update the domain URL when redeploying.
- **Audio**: Replace the base64 strings in `AUD` to swap background music or sound effects.
