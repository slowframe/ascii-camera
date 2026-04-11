# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
./test.sh        # no automated tests — browser-only app, exits 0
```

Open `index.html` directly in a browser to run the app. No build step, no server, no dependencies.

## Architecture

Everything lives in a single file: `index.html`. CSS, HTML, and JS are all inline — there is no bundler, no modules, no external scripts.

### Render pipeline (per frame)

```
getUserMedia → sampler canvas (cols×rows px) → getImageData → per-cell loop → asciiCanvas draw
```

1. **Grid calc** — `calcGrid()` measures character cell dimensions via a hidden `#char-probe` span, then computes how many cols/rows fit the container. The video frame is center-cropped to the grid's aspect ratio.
2. **Sampling** — the video frame is drawn into a tiny `samplerCanvas` (one pixel per cell) with horizontal mirror flip applied via canvas transform.
3. **Per-cell loop** — for each cell: compute luminance → apply contrast → select character from ramp → apply gamma lift to RGB → resolve palette color.
4. **Dirty-cell buffer** — `frameBuffer` stores `{ brightQ, colorKey }` per cell. Cells are only redrawn when one of those keys changes. `brightQ` is brightness quantized to 8 steps. `colorKey` for palette modes is the ANSI16 nearest-color index (coarser than the rendered color — intentional stability dead-zone).
5. **Canvas draw** — cells are drawn with `fillRect` (black background) then `fillText` (character). `font` and `textBaseline` are re-set after every potential canvas resize because resizing resets the 2D context state.

### Palette system

- Four palette lookup tables (`paletteLookups`) are precomputed at load time via `buildNearestLookup()` — a 4096-entry `Uint16Array` mapping quantized RGB → nearest palette index.
- `getPaletteInfo(name)` returns `{ pal, cache }` for indexed palettes. Returns `null` for `truecolor`, `green`, and `amber` (handled separately in the loop).
- **If you add a palette**, you must call `buildNearestLookup()` for it and register it in `paletteLookups`.

### Key design constraints (do not violate)

- **Gamma lift split is intentional**: `bright` (pre-gamma) drives character selection; `brightLifted` (post-gamma) drives RGB color scaling. Merging them corrupts luminance fidelity.
- **`colorKey` uses ANSI16 index, not the rendered color**: this coarser key absorbs camera noise. Switching to the full rendered color causes visible flicker.
- **`font`/`textBaseline` must be set after canvas resize**: canvas resize resets 2D context state entirely.
- **LUT is static**: `buildNearestLookup()` runs once at load. It is not rebuilt when controls change.
