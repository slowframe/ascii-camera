# ASCII Camera

A single-file browser app that converts live webcam input into colored ASCII art in real time. No build step, no dependencies, no server — open the HTML file and go.

---

## Demo

Open `index.html` in a modern browser and click **Start**.

Camera access is required. The app runs entirely client-side.

---

## Features

- Live webcam → ASCII art rendered to canvas
- 8 character sets including classic ASCII ramps, CP437 block characters, Braille dot patterns, and custom Cyborg and Occult glyph sets
- 7 color palettes: xterm 256, ANSI 16, CGA, Green monochrome, Amber monochrome, Grayscale 24, and True Color
- Adjustable font size (8–40px), which controls ASCII grid resolution
- Adjustable FPS (1–20)
- Brightness slider (gamma lift) to compensate for ASCII coverage loss
- Contrast slider
- MediaPipe SelfieSegmentation — identifies the person in frame; background cells are attenuated by the BG slider before the render pipeline runs
- BG slider (0–0.4) controls how much background luminance leaks through; 0 = hard black, higher = faint ghost
- Dirty-cell frame buffer — only redraws cells that actually changed
- Inline stats display showing % character changes, % color changes, and total cell count per frame
- Mobile-friendly collapsible controls drawer

---

## How It Works

### Pipeline Overview

```
Webcam → sampler canvas → MediaPipe (async) → mask weight per cell → brightness/color → charset lookup → palette lookup → canvas draw
```

### 1. Video Capture

The browser's `getUserMedia` API provides webcam frames. Each frame is drawn into a small **sampler canvas** scaled to the current ASCII grid resolution, giving fast pixel access via `getImageData`.

### 2. Grid Calculation

The renderer computes how many character columns and rows fit the container at the current font size. The video frame is center-cropped to match the grid's aspect ratio, avoiding distortion.

### 3. Person Segmentation

The sampler canvas is sent to **MediaPipe SelfieSegmentation** (landscape model) each frame. Because it runs asynchronously, results arrive on the following frame — adding a single frame of latency that is not perceptible.

`onSegmentationResults` extracts the segmentation mask's alpha channel into a `Float32Array` (`maskData`) at cell resolution. `getMaskWeight()` returns `1.0` for foreground cells (confidence > 0.5) and the BG slider value for background cells. This weight is the first thing applied in the per-cell loop, before luminance, contrast, or character selection.

When MediaPipe has not yet loaded or no mask is available, `maskWeight` defaults to `1.0` so the app works normally without segmentation.

### 4. Brightness and Contrast

Per-cell luminance is computed using the standard weighted formula:

```
brightness = 0.299R + 0.587G + 0.114B
```

Contrast is applied around the midpoint, then brightness is quantized into discrete steps to reduce flicker from camera noise.

### 5. Gamma Lift

ASCII characters cover less than 100% of their cell area — even the densest characters like `@` or `█` top out around 60–70% coverage. This causes the output to read darker than the source image.

The brightness slider applies a power-curve gamma lift **before character selection**, pushing midtones up while preserving black at 0 and white at 1. This compensates for coverage loss without distorting the character mapping.

```
lifted = brightness ^ gamma   (gamma < 1.0 brightens midtones)
```

### 6. Character Selection

The lifted brightness value indexes into the chosen character ramp. Darker cells map to sparse characters (spaces, dots); brighter cells map to denser characters (`@`, `#`, `█`).

Braille mode dynamically generates a 256-character ramp sorted by dot density, giving sub-character resolution.

### 7. Color Mapping

The original pixel RGB is used for color output — independent of the brightness used for character selection. A second gamma lift is applied to the RGB values to match the visual brightness of the character layer.

For palette modes (xterm 256, ANSI 16, CGA, Grayscale 24), a 4096-entry precomputed lookup table accelerates nearest-color matching.

### 8. Dirty-Cell Rendering

The renderer stores the previous frame's character and color for every cell. A cell is only redrawn when:
- The brightness has changed enough to select a different character
- The color mapping has changed

This eliminates the cost of redrawing the full canvas every frame, which is significant at high resolutions.

---

## Character Sets

| Name | Description |
|---|---|
| Bourke 70 | Paul Bourke's classic 70-character luminance ramp — the historical standard for ASCII art renderers |
| Bourke 10 | Trimmed 10-character version of the same ramp — cleaner, more stylized |
| AAlib | Ramp from the AAlib C library (1997), used in terminal video renderers and demoscene tools |
| CP437 Blocks | IBM PC block characters (`░▒▓█`) from the BBS/ANSI art era |
| Hatching | Constructed ramp using stroke and cross characters — engraving aesthetic |
| Braille | Unicode braille patterns used as 2×4 pixel grids, popular in modern terminal image viewers |
| Cyborg | Custom high-contrast ramp built for stark, punchy output — sharp transitions between light and dark |
| Occult | Custom glyph set drawn from religious and occult symbols — moons, stars, sigils, and sacred geometry |

---

## Requirements

Any modern browser with webcam access and support for:
- `getUserMedia`
- HTML5 Canvas
- `getImageData`

MediaPipe SelfieSegmentation is loaded from `cdn.jsdelivr.net` at runtime when the camera starts. No other external dependencies.

---

## License

MIT

---

## Notes for Modification

### The gamma lift split is intentional — do not merge the two pipelines

The brightness slider controls gamma lift, but it is applied in two separate places for two different purposes. `bright` (pre-gamma) is used for character selection. `brightLifted` (post-gamma) is used only to scale the RGB color output. If you apply gamma before the charset index lookup, you change which character gets picked — the output reads correctly brighter but you've corrupted the source luminance fidelity that makes photographic charsets work well. The split is load-bearing. Keep it.

### The dirty buffer uses two different comparison keys for a reason

Character redraw is gated on `brightQ` (brightness quantized to 8 steps). Color redraw is gated on `colorKey`, which for palette modes is the ANSI16 nearest-color index — not the actual rendered palette color. Using a coarser key for color comparison than for color rendering is intentional: it creates a stability dead-zone that absorbs per-frame camera noise without making the output feel unresponsive. If you switch to using the full rendered color as the comparison key, expect visible flicker in static areas.

### The palette LUT must be rebuilt if you change a palette

`buildNearestLookup()` precomputes a 4096-entry lookup table for each palette at load time. If you add a new palette or modify an existing one's colors, you must add a corresponding `buildNearestLookup()` call and register the result in `paletteLookups`. The LUT is not rebuilt dynamically.

### The mask is applied before contrast — do not reorder

`maskWeight` scales raw luminance before the contrast curve runs. If you apply contrast first and mask after, the contrast curve lifts fully-masked cells (brightness = 0) to the midpoint (0.5), making invisible background cells reappear as gray. The per-cell loop guards against this with an explicit `bright === 0.0` check that bypasses the curve — that guard is also load-bearing.

### Canvas resize resets the 2D drawing state

`font` and `textBaseline` are reassigned after every potential canvas resize. This is intentional — resizing a canvas element in the DOM resets its entire 2D context state including font. If you move the font assignment earlier in the render loop, characters will stop drawing correctly after any resize event.
