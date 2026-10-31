# Compose Web without Skia

A fork of [Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform-core) whose web targets (`wasmJs` and `js`) draw with the browser's Canvas 2D API, plus one shared WebGL 2 context where Canvas 2D can't, instead of Skia compiled to WebAssembly. Your Compose code stays the same, and `skiko.wasm` is gone.

It is an independent project, not affiliated with JetBrains. It is based on Compose `1.13.0-beta01`, and it isn't published to a Maven repository yet: you build it yourself.

## What it changes

The same Material 3 app, built twice, as production bundles. Medians in Chrome, on `wasmJs` unless stated. [Safari](#safari) is its own story.

| | upstream (Skia) | Canvas 2D | |
|---|---|---|---|
| Download of the first screen (brotli) | 3077 KiB | 545 KiB | 5.6× smaller |
| The same on `js` | 3074 KiB | 627 KiB | 4.9× smaller |
| First frame | 239 ms | 95 ms | 2.5× faster |
| The same on `js` | 469 ms | 237 ms | 2.0× faster |
| First frame, 4× slower CPU, 10 Mbit/s, 100 ms RTT | 3583 ms | 1165 ms | 3.1× faster |
| Fonts downloaded for 11 scripts and emoji | 28 files, 1213 KiB | none | |

Fonts need no download because the browser draws text, with the fonts of the system.

## Animations in Chrome

The frame rate is uncapped, so this is the throughput of the whole pipeline rather than the refresh rate of a display, and both builds draw the same frames. All the scenarios of the benchmark, on `wasmJs`. CPU is per frame, of the page and the GPU process together. On a 60 Hz display, CPU per frame is the column that matters.

| scenario | frames/s, upstream | frames/s, Canvas 2D | CPU, upstream | CPU, Canvas 2D |
|---|---|---|---|---|
| `scroll`: a list of 100 000 messages | 1679 | 2958 | 1.30 ms | 0.90 ms |
| `layers`: 400 layers of a single shape | 228 | 795 | 16 ms | 1.97 ms |
| `fade`: 60 cards fading | 765 | 2731 | 3.76 ms | 1.02 ms |
| `churn`: the cards moving, with text that changes | 349 | 581 | 4.99 ms | 2.43 ms |
| `filters`: color matrices | 754 | 984 | 3.93 ms | 3.95 ms |
| `blur`: 24 blurred cards | 579 | 740 | 5.26 ms | 4.53 ms |
| `nested`: layers in layers | 663 | 2177 | 4.75 ms | 1.43 ms |
| `perspective`: 24 turning cards | 1657 | 1776 | 1.25 ms | 1.93 ms |
| `elevation`: shadows that change | 850 | 956 | 2.16 ms | 3.06 ms |
| `shadows`: shadows that stay | 972 | 2398 | 1.93 ms | 1.32 ms |
| `cutshadows`: shadows of paths | 787 | 2485 | 2.55 ms | 1.28 ms |
| `cardscroll`: shadows that move | 770 | 1619 | 2.59 ms | 1.68 ms |
| `blend`: a layer blend mode | 320 | 2680 | 8.79 ms | 1.14 ms |
| `canvas`: 1600 shapes | 665 | 1315 | 2.29 ms | 1.79 ms |
| `text`: 12 paragraphs laid out again | 37 | 2146 | 9.86 ms | 1.05 ms |
| `effects`: shadows under a moving blur | 1097 | 3312 | 1.83 ms | 1.11 ms |

**Where it loses.** `perspective` and `elevation` cost about 50% more CPU per frame than Skia, and `filters` is level. On `js`, `elevation` also draws fewer frames (1010 → 937). These are still to be fixed.

**How to read `text`.** The two builds don't lay out the same thing: upstream uses Roboto and the fork the system font, so lines break differently. The gap is real but it isn't a like-for-like 58×.

**How to read `layers`.** Fading a layer of a single shape needs no offscreen surface here, so it's the best case. `fade` and `nested` are the same work without that shortcut.

Measured in Chrome 156 on an Apple M4 Pro, 1280×800 at 2×: [full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/chrome.md), including `js` and memory. The [benchmark](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark) has the app, the runners and how to measure it yourself.

## Safari

Safari 27.2 on the same machine. It draws at 60 Hz, so frame rates are capped, and CPU per frame is what compares the builds. The first frame is faster here too: 267 → 147 ms on `wasmJs` (1.8×), 383 → 256 ms on `js` (1.5×).

**Animations are where the fork doesn't win in Safari.** Both builds hold 60 frames per second in most scenarios, and the fork uses more CPU in most of them: close to twice as much in `filters`, `blur`, `perspective` and `cutshadows`. It uses less in `layers`, `blend` and `text`. This is still to be fixed.

**Upstream's `wasmJs` build breaks in three scenarios here:** `layers` runs at 26 frames per second, `canvas` at 8, and `blur` stops with an error (`Can't wrap nullptr`). Its `js` build holds 60 in all three, and the fork holds 60 in every scenario.

CPU per frame, of the busiest page and GPU processes together, upstream and Canvas 2D:

| scenario | `wasmJs`, upstream | `wasmJs`, Canvas 2D | `js`, upstream | `js`, Canvas 2D |
|---|---|---|---|---|
| `scroll` | 10 ms | 11 ms | 10 ms | 11 ms |
| `layers` | 34 ms, at 26 fps | 12 ms | 20 ms | 12 ms |
| `fade` | 10 ms | 12 ms | 11 ms | 11 ms |
| `churn` | 9.8 ms | 11 ms | 17 ms | 19 ms |
| `filters` | 11 ms | 18 ms | 11 ms | 20 ms |
| `blur` | fails | 20 ms | 12 ms | 20 ms |
| `nested` | 10 ms | 12 ms | 12 ms | 12 ms |
| `perspective` | 9.9 ms | 16 ms | 9.9 ms | 18 ms |
| `elevation` | 9.7 ms | 12 ms | 10 ms | 13 ms |
| `shadows` | 9.8 ms | 12 ms | 9.8 ms | 12 ms |
| `cutshadows` | 10 ms | 20 ms | 9.9 ms | 19 ms |
| `cardscroll` | 9.9 ms | 13 ms | 11 ms | 13 ms |
| `blend` | 15 ms | 11 ms | 15 ms | 12 ms |
| `canvas` | 183 ms, at 8 fps | 11 ms | 10 ms | 11 ms |
| `text` | 14 ms | 11 ms | 19 ms | 12 ms |
| `effects` | 9.8 ms | 13 ms | 10 ms | 13 ms |

[Full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/safari.md).

## Limitations

- **It is not pixel-identical.** Text differs, because the browser draws it. Scenes of shapes match almost exactly (0.1–0.3% of pixels differ visibly in `canvas` and `layers`), scenes with some text differ in 1–5% of their pixels, and `text` in 15%. Scrolling scenes can't be compared this way, since the faster build has scrolled further.
- **Libraries that import Skia don't link.** Anything using `org.jetbrains.skia` needs a fork: [`components-resources`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13/components/resources) has one, `coil` and `haze` don't yet.
- **Skia-only features are gone**, such as Skottie for Lottie animations.
- **An SVG resource goes through an `<img>`:** an SVG with `<foreignObject>` isn't shown, and one without a size of its own gets the browser's natural size rather than none.
- **A `ResourceReader` of your own** serves images, SVG, XML and fonts through `getUri`, which must be a URL the browser can fetch.
- **Measured on one machine.** `wasmJs` needs WasmGC (Safari 18.2+), so older browsers need the `js` build. Firefox, Windows and Linux are not measured.

## Repositories

- [`compose-multiplatform-core`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform-core/tree/web-canvas/1.13), branch `web-canvas/1.13`: Compose, with the renderer in `compose/ui/ui-web-canvas`.
- [`compose-multiplatform`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13), branch `web-canvas/1.13`: `components-resources` without Skia.
- [`compose-web-benchmark`](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark): the numbers above.
