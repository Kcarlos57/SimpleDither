# Simple Dither

A single-file, zero-dependency browser tool for dithering images. Upload a photo, adjust the controls, and the processed result becomes the full-screen background. Everything runs locally in the browser.

---

## Usage

Download `dither.html` and open it in any modern browser. That's it.

```
open dither.html
```

Link to site: https://kcarlos57.github.io/SimpleDither/

---

## Features

### Algorithms
| Algorithm | Description |
|---|---|
| **Ordered (Bayer)** | Compares each pixel against a tiled threshold matrix. Produces a regular halftone dot pattern. Grid size is configurable (2×2 → 16×16). |
| **Floyd-Steinberg** | Error diffusion — rounding errors are propagated to neighbouring pixels. Smooth, organic output with no visible grid. |
| **Atkinson** | Apple's 1984 Mac algorithm. Diffuses 6/8 of the error across 6 neighbours; absorbs the rest. Crisper and slightly lighter than Floyd-Steinberg. |

### Core controls
| Control | Range | Effect |
|---|---|---|
| Threshold | 0 – 255 | Brightness cut-off between black and white pixels |
| Contrast | −100 – +100 | Applied before dithering via a photographic S-curve |
| Scale | 1× – 8× | Downsamples the image before dithering — the primary coarseness control |
| Noise | 0 – 80 | Adds seeded random jitter to break up repetitive patterns |

### Visual popup
- **Invert** — swap black and white output pixels
- **Dot size** — Bayer matrix size (2, 4, 8, 16) for ordered dithering
- **Dark / light colour** — replace black and white pixels with custom colours
- **Gradient tint** — diagonal two-colour gradient composited over the result with adjustable opacity

### Pre-process popup
- **Brightness** — offset applied before dithering
- **Blur** — Gaussian blur via CSS canvas filter, softens edges before the algorithm runs
- **Zoom / crop** — values above 1 crop into the centre of the image; values below 1 zoom out, padding with black

### Output popup
- **Export resolution** — 1×, 2×, or 4× upscale of the saved PNG
- **Transparent background** — exports with black pixels as fully transparent alpha
- **Copy to clipboard** — writes the canvas as a PNG blob via the Clipboard API

### Creative popup
- **Scanlines** — draws semi-transparent horizontal lines every 2px (CRT effect) with adjustable intensity
- **Tile mode** — repeats the dithered image as a tiling pattern
- **Animate threshold** — continuously sweeps the threshold value between ~38–218 using `requestAnimationFrame`
- **Reseed noise** — picks a new seed for the Mulberry32 PRNG, changing the noise pattern

### Panel controls
- **⛶ drag handle** (top-left) — click and drag to reposition the panel anywhere on screen; touch supported
- **⊠ hide button** (top-right of panel) — fades the entire panel out for a clean screenshot; a small floating button appears in the screen corner to restore it
- **? info button** (top-right of panel) — opens a side popup with plain-language descriptions of each algorithm and core slider
- **fill / fit toggle** (bottom row) — switches the background display between two modes:
  - **fill** — stretches the dithered result to cover the full screen
  - **fit** — scales the image down to fit entirely within the screen with black bars on the short axis, preserving aspect ratio; useful for checking what the saved file will look like before downloading

---

## Implementation notes

### Single-file architecture

All HTML, CSS, and JavaScript lives in one `.html` file. This is an intentional choice: the tool has no build pipeline, no dependencies, and no server requirements. Opening it via `file://` in any browser works without restriction because there are no external asset imports.

### Pixel processing

Dithering is performed on a hidden off-screen `<canvas>` element (`#work`). The pipeline on each render is:

1. Draw the source image to the scratch canvas at working resolution (original size ÷ scale, adjusted for zoom)
2. Read pixel data with `getImageData()`
3. Convert each pixel to greyscale using the standard luminance weights (`0.299R + 0.587G + 0.114B`)
4. Apply brightness offset and contrast curve
5. Run the selected dithering algorithm over the greyscale float array
6. Write the result back via `putImageData()`
7. Draw the scratch canvas to the full-screen `#bg-canvas` (stretched in fill mode, letterboxed in fit mode) with `imageSmoothingEnabled = false`
8. Composite tint gradient and scanlines on top

All pixel data is handled as typed arrays (`Float32Array` for greyscale, `Uint8ClampedArray` for output RGBA) for performance.

### Debouncing

Re-rendering on every slider input event would cause jank. All slider changes are debounced with an 80ms timeout — the render only fires once the user pauses.

### Seeded noise

JavaScript's `Math.random()` cannot be seeded, so the noise pattern would change on every render, making other slider adjustments visually unstable. A [Mulberry32](https://gist.github.com/tommyettinger/46a874533244883189143505d203312c) PRNG is used instead, initialised from a stored seed value that only changes when the user clicks "reseed noise".

### Save / export

The save function re-runs the full dithering pipeline at the output resolution rather than upscaling the existing screen preview. This ensures the saved PNG faithfully matches the on-screen result regardless of monitor size or display mode. Output dimensions always match the original input image (× export multiplier) — the scale slider affects dot coarseness, not file dimensions.

### Fit mode

In fit mode, the background canvas is filled with solid black before drawing, then the dithered result is drawn centred and scaled to fit within the screen bounds. Tint and scanlines are clipped to the image rectangle so the black letterbox bars stay clean. This mode is primarily useful for previewing exact dot density before saving.

---
