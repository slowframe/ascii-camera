# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

No automated tests — browser-only app. Open `index.html` directly in a browser to run the app. No build step, no server, no dependencies.

## Architecture

Everything lives in a single file: `index.html`. CSS, HTML, and JS are all inline — there is no bundler, no modules, no external scripts.

### Render pipeline (per frame)

```
getUserMedia → sampler canvas (cols×rows px) → MediaPipe (async) → getImageData → per-cell loop → asciiCanvas draw
```

1. **Grid calc** — `calcGrid()` measures character cell dimensions via a hidden `#char-probe` span, then computes how many cols/rows fit the container. The video frame is center-cropped to the grid's aspect ratio.
2. **Sampling** — the video frame is drawn into a tiny `samplerCanvas` (one pixel per cell) with horizontal mirror flip applied via canvas transform.
3. **MediaPipe segmentation** — `samplerCanvas` is sent to `SelfieSegmentation` each frame (model 1, landscape). Results arrive asynchronously via `onSegmentationResults()`, which extracts alpha → `maskData` (Float32Array, one confidence value per cell). The mask is always active once loaded; `getMaskWeight()` returns 1.0 for foreground cells and `bgLevel` (BG slider) for background cells.
4. **Per-cell loop** — for each cell: apply mask weight → compute luminance → apply contrast curve (0.0-guarded to avoid lifting pure black) → select character from ramp → apply gamma lift to RGB → resolve palette color.
5. **Dirty-cell buffer** — `frameBuffer` stores `{ brightQ, colorKey }` per cell. Cells are only redrawn when one of those keys changes. `brightQ` is brightness quantized to 8 steps. `colorKey` for palette modes is the ANSI16 nearest-color index (coarser than the rendered color — intentional stability dead-zone). Both keys encode `maskWeight` so the buffer correctly invalidates when BG slider or mask changes.
6. **Canvas draw** — cells are drawn with `fillRect` (black background) then `fillText` (character). `font` and `textBaseline` are re-set after every potential canvas resize because resizing resets the 2D context state.

### Palette system

- Four palette lookup tables (`paletteLookups`) are precomputed at load time via `buildNearestLookup()` — a 4096-entry `Uint16Array` mapping quantized RGB → nearest palette index.
- `getPaletteInfo(name)` returns `{ pal, cache }` for indexed palettes. Returns `null` for `truecolor`, `green`, and `amber` (handled separately in the loop).
- **If you add a palette**, you must call `buildNearestLookup()` for it and register it in `paletteLookups`.

### Debug panels (8-cell 4×2 grid)

When debug mode is active, `renderDebugPanels()` fills a `#debug-grid` with: **RAW** (mirrored source pixels), **MASK** (segmentation confidence, grayscale), **LUMA** (post-mask brightness), **CONTRAST** (curve applied), **CHAR** (glyph chosen, white on black), **COLOR LIFT** (gamma-compensated RGB), **PALETTE SNAP** (final palette color + glyph), and one empty placeholder cell.

### Key design constraints (do not violate)

- **Gamma lift split is intentional**: `bright` (pre-gamma) drives character selection; `brightLifted` (post-gamma) drives RGB color scaling. Merging them corrupts luminance fidelity.
- **`colorKey` uses ANSI16 index, not the rendered color**: this coarser key absorbs camera noise. Switching to the full rendered color causes visible flicker.
- **`font`/`textBaseline` must be set after canvas resize**: canvas resize resets 2D context state entirely.
- **LUT is static**: `buildNearestLookup()` runs once at load. It is not rebuilt when controls change.
- **Mask is applied before contrast**: `maskWeight` scales raw luminance first. The contrast curve then gets a 0.0-guard so fully-masked cells (bright = 0) are not lifted to the 0.5 midpoint.
- **MediaPipe receives the already-mirrored `samplerCanvas`**: no second mirror is needed in `onSegmentationResults`. `maskData` is already in the same coordinate space as `samplerCanvas` pixels.
