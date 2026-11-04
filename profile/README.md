# Compose Web without Skia

A fork of [Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform-core) whose web targets (`wasmJs` and `js`) draw with the browser's Canvas 2D API, plus one shared WebGL 2 context where Canvas 2D can't, instead of Skia compiled to WebAssembly. Your Compose code stays the same, and `skiko.wasm` is gone.

It is an independent project, not affiliated with JetBrains, based on Compose `1.13.0-beta01`. You don't need to build Compose yourself: [it is published](#using-it).

## Demo

[emotes.redlance.org](https://emotes.redlance.org/) runs on the fork since its version 2.4.0, and its footer shows the version.

## What it changes

The same Material 3 app, built twice, as production bundles. Medians in Chrome, on `wasmJs` unless stated. [Firefox](#firefox) and [Safari](#safari) have sections of their own.

| | upstream (Skia) | Canvas 2D | |
|---|---|---|---|
| Download of the first screen (brotli) | 3081 KiB | 552 KiB | 5.6× smaller |
| The same on `js` | 3077 KiB | 634 KiB | 4.9× smaller |
| First frame | 225 ms | 78 ms | 2.9× faster |
| The same on `js` | 448 ms | 230 ms | 1.9× faster |
| First frame, 4× slower CPU, 10 Mbit/s, 100 ms RTT | 3513 ms | 1046 ms | 3.4× faster |
| Fonts downloaded for 11 scripts and emoji | 28 files, 1213 KiB | none | |

Fonts need no download because the browser draws text, with the fonts of the system.

## Animations in Chrome

The frame rate is uncapped, so this is the throughput of the whole pipeline rather than the refresh rate of a display, and both builds draw the same frames. All the scenarios of the benchmark, on `wasmJs`. CPU is per frame, of the page and the GPU process together. On a 60 Hz display, CPU per frame is the column that matters.

| scenario | frames/s, upstream | frames/s, Canvas 2D | CPU, upstream | CPU, Canvas 2D |
|---|---|---|---|---|
| `scroll`: a list of 100 000 messages | 2177 | 2949 | 1.25 ms | 0.91 ms |
| `layers`: 400 layers of a single shape | 232 | 825 | 15 ms | 1.89 ms |
| `fade`: 60 cards fading | 736 | 2500 | 3.77 ms | 1.11 ms |
| `churn`: the cards moving, with text that changes | 361 | 608 | 4.72 ms | 2.31 ms |
| `filters`: color matrices | 757 | 969 | 3.75 ms | 4.03 ms |
| `imagebrush`: a tiled image through color matrices | 771 | 4490 | 1.86 ms | 0.68 ms |
| `gradientbrush`: a gradient through color matrices | 769 | 5659 | 1.90 ms | 0.57 ms |
| `compositebrush`: a blend of two gradients through color matrices | 766 | 33 | 1.95 ms | 93 ms |
| `softlight`: an image drawn in soft-light | 452 | 5756 | 5.84 ms | 0.57 ms |
| `newcontent`: a scroll, after texts new in each of its first 120 frames | 1609 | 5062 | 1.59 ms | 0.69 ms |
| `blur`: 24 blurred cards | 564 | 748 | 5.46 ms | 4.37 ms |
| `nested`: layers in layers | 698 | 2154 | 4.37 ms | 1.33 ms |
| `perspective`: 24 turning cards | 1165 | 1520 | 1.50 ms | 1.84 ms |
| `elevation`: shadows that change | 733 | 976 | 2.34 ms | 3.00 ms |
| `shadows`: shadows that stay | 813 | 2634 | 2.12 ms | 1.20 ms |
| `cutshadows`: shadows of paths | 629 | 2371 | 2.83 ms | 1.22 ms |
| `cardscroll`: shadows that move | 657 | 1839 | 2.72 ms | 1.48 ms |
| `cardlist`: a list clipped by a card with rounded corners | 1255 | 3276 | 1.56 ms | 0.83 ms |
| `blend`: a layer blend mode | 334 | 2607 | 8.12 ms | 1.08 ms |
| `softlightlayers`: layers blended in soft-light | 332 | 2492 | 8.15 ms | 1.15 ms |
| `canvas`: 1600 shapes | 652 | 1389 | 3.30 ms | 1.74 ms |
| `text`: 12 paragraphs laid out again | 45 | 2240 | 9.21 ms | 1.00 ms |
| `effects`: shadows under a moving blur | 788 | 3411 | 2.18 ms | 1.06 ms |

**Where it loses.** `compositebrush` falls from 766 to 33 frames per second: no fill style draws a blend of shaders, so the fork draws an image of it through the filter, made again for every card in every frame. `perspective` and `elevation` cost about 25% more CPU per frame than Skia, in every run (1.83–1.88 ms against 1.48–1.51, and 2.99–3.00 against 2.27–2.52), and `filters` about 7% more (4.03 against 3.75 ms; one slower upstream run reached 4.18). On `js`, `perspective` and `elevation` cost about 20% more, `filters` is level, and `compositebrush` runs at 43. These are still to be fixed.

**How to read `text`.** The two builds don't lay out the same thing: upstream uses Roboto and the fork the system font, so lines break differently. The gap is real but it isn't a like-for-like 50×.

**How to read `layers`.** Fading a layer of a single shape needs no offscreen surface here, so it's the best case. `fade` and `nested` are the same work without that shortcut.

Medians of three runs, measured in Chrome 156 on an Apple M4 Pro, 1280×800 at 2×: [full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/chrome.md), including `js` and memory. The [benchmark](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark) has the app, the runners and how to measure it yourself.

## Firefox

Firefox 159 on the same machine, the same builds, drawing at most 120 frames per second. Where both builds hold 120, CPU per frame, of the tab and GPU processes together, is what compares them. On `wasmJs`:

| scenario | frames/s, upstream | frames/s, Canvas 2D | CPU, upstream | CPU, Canvas 2D |
|---|---|---|---|---|
| `scroll` | 120 | 120 | 3.74 ms | 4.55 ms |
| `layers` | 119 | 120 | 15 ms | 5.95 ms |
| `fade` | 120 | 120 | 4.11 ms | 4.46 ms |
| `churn` | 116 | 120 | 6.90 ms | 6.75 ms |
| `filters` | 120 | 120 | 4.26 ms | 5.23 ms |
| `imagebrush` | 120 | 120 | 3.75 ms | 4.33 ms |
| `gradientbrush` | 120 | 120 | 3.70 ms | 4.56 ms |
| `compositebrush` | 120 | 29 | 3.72 ms | 40 ms |
| `softlight`, drawn by the fork as hard-light | 120 | 120 | 5.89 ms | 4.44 ms |
| `newcontent` | 120 | 120 | 3.92 ms | 4.19 ms |
| `blur` | 120 | 120 | 4.99 ms | 4.32 ms |
| `nested` | 120 | 120 | 4.41 ms | 4.48 ms |
| `perspective` | 120 | 119 | 4.10 ms | 13 ms |
| `elevation` | 120 | 45 | 3.81 ms | 25 ms |
| `shadows` | 120 | 120 | 3.83 ms | 5.64 ms |
| `cutshadows` | 120 | 120 | 3.91 ms | 5.22 ms |
| `cardscroll` | 120 | 120 | 4.16 ms | 6.48 ms |
| `cardlist` | 120 | 120 | 3.92 ms | 4.60 ms |
| `blend` | 120 | 120 | 7.20 ms | 4.52 ms |
| `softlightlayers` | 120 | 46 | 7.22 ms | 22 ms |
| `canvas` | 120 | 120 | 4.20 ms | 5.38 ms |
| `text` | 109 | 120 | 9.23 ms | 4.22 ms |
| `effects` | 120 | 120 | 4.00 ms | 5.22 ms |

**Where it loses in Firefox.** Three scenarios fall well below 120: `compositebrush` (29; 33 in Chrome), `softlightlayers` (46), because Firefox's accelerated Canvas 2D can't draw soft-light and compositing a layer in it falls back to software, and `elevation` (45), for a reason not found yet. `perspective` holds 119–120 in two runs of three and drops to about 90 in the third, at three times the CPU. Where both hold 120, the fork mostly takes more CPU per frame, up to 1.6× in `cardscroll`, and less in `blur` and `blend`; it also takes less in `layers`, `churn` and `text`, where upstream doesn't hold 120. `softlight` costs it less too, but the fork draws it as hard-light in Firefox ([Limitations](#limitations)), so that row compares a different effect. `js` gives the same picture. These are still to be fixed.

[Full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/firefox.md).

## With Haze

[Haze](https://github.com/chrisbanes/haze) blurs what's behind a panel, in every frame if what's behind it moves, which makes it a hard case for a renderer. [Its fork](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.1) draws without Skia. A small app with panels of 340×340 CSS px over a moving backdrop, measured like the scenarios above, on `wasmJs`:

| | upstream (Skia) | Canvas 2D | |
|---|---|---|---|
| Download (brotli) | 2830 KiB | 355 KiB | 8.0× smaller |
| First frame | 202 ms | 63 ms | 3.2× faster |
| One frosted panel, frames/s | 722 | 4372 | 6.1× more |
| One frosted panel, CPU per frame | 2.06 ms | 0.94 ms | 2.2× less |
| Six panels of different styles, frames/s | 628 | 446 | 1.4× fewer |
| Six panels of different styles, CPU per frame | 3.19 ms | 5.80 ms | 1.8× more |
| Firefox: one frosted panel, frames/s | 120 | 120 | |
| Firefox: one frosted panel, CPU per frame | 2.60 ms | 3.06 ms | 1.2× more |
| Firefox: six panels, frames/s | 120 | 14 | 8.6× fewer |
| Firefox: six panels, CPU per frame | 3.14 ms | 69 ms | 22× more |

**In Chrome one panel wins and six lose.** Each panel costs the fork a blur of its own, so its cost grows with the panels much faster than Skia's does. **In Firefox** one panel holds 120 frames per second at 1.2× the CPU, and six drop to 14, for a reason not found yet; the fork draws Haze's noise there as hard-light. These are still to be fixed. `js` gives the same picture. Safari isn't measured with Haze. [Full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/haze.md).

## Safari

Safari 27.2 on the same machine, the same builds. It draws at 60 Hz, so frame rates are capped, and CPU per frame is what compares the builds. The first frame is faster here too: 290 → 138 ms on `wasmJs` (2.1×), 372 → 221 ms on `js` (1.7×).

**Animations are where the fork doesn't win in Safari.** Both builds hold 60 frames per second almost everywhere. On `wasmJs` the fork uses more CPU in 16 scenarios of 22: about twice as much in `filters`, `elevation` and `cutshadows`, 1.6× in `perspective` and `cardlist`. On `js` it uses more in 11 and less in 10: about twice as much in `filters` and `cutshadows`, 1.6–1.7× in `perspective`, `cardlist` and `blur`. On both it uses less in `layers`, `blend`, `softlightlayers` and `text`. `compositebrush` drops to 20 frames per second. This is still to be fixed.

**Upstream's `wasmJs` build breaks in three scenarios here:** `layers` runs at 29 frames per second, `canvas` at 9, and `blur` stops with an error (`Can't wrap nullptr`). Its `js` build holds 60 in all three, and the fork holds 60 in every scenario but `compositebrush`.

CPU per frame, of the busiest page and GPU processes together, upstream and Canvas 2D:

| scenario | `wasmJs`, upstream | `wasmJs`, Canvas 2D | `js`, upstream | `js`, Canvas 2D |
|---|---|---|---|---|
| `scroll` | 7.57 ms | 7.37 ms | 8.33 ms | 8.37 ms |
| `layers` | 16 ms, at 29 fps | 7.80 ms | 16 ms | 8.87 ms |
| `fade` | 6.25 ms | 6.50 ms | 8.37 ms | 8.03 ms |
| `churn` | 6.33 ms | 7.83 ms | 15 ms | 15 ms |
| `filters` | 7.10 ms | 13 ms | 7.94 ms | 16 ms |
| `imagebrush` | 4.23 ms | 5.93 ms | 6.27 ms | 6.10 ms |
| `gradientbrush` | 4.15 ms | 6.17 ms | 6.63 ms | 6.53 ms |
| `compositebrush` | 4.09 ms | 75 ms, at 20 fps | 6.30 ms | 80 ms, at 19 fps |
| `softlight` | 4.17 ms | 6.00 ms | 8.97 ms | 6.97 ms |
| `newcontent` | 6.20 ms | 6.43 ms | 7.99 ms | 7.24 ms |
| `blur` | fails | 12 ms | 9.03 ms | 15 ms |
| `nested` | 6.13 ms | 6.54 ms | 8.00 ms | 7.60 ms |
| `perspective` | 6.25 ms | 10 ms | 7.97 ms | 13 ms |
| `elevation` | 4.63 ms | 9.07 ms | 8.00 ms | 11 ms |
| `shadows` | 5.63 ms | 7.33 ms | 6.69 ms | 8.17 ms |
| `cutshadows` | 5.90 ms | 12 ms | 6.83 ms | 13 ms |
| `cardscroll` | 6.63 ms | 8.20 ms | 8.07 ms | 9.63 ms |
| `cardlist` | 7.03 ms | 11 ms | 8.39 ms | 13 ms |
| `blend` | 8.63 ms | 6.40 ms | 10 ms | 8.50 ms |
| `softlightlayers` | 8.53 ms | 6.68 ms | 11 ms | 8.17 ms |
| `canvas` | 122 ms, at 9 fps | 7.26 ms | 6.77 ms | 7.63 ms |
| `text` | 8.93 ms | 6.67 ms | 13 ms | 9.60 ms |
| `effects` | 6.72 ms | 7.87 ms | 7.10 ms | 8.07 ms |

[Full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/safari.md).

## Using it

The fork is on a Maven repository:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        maven("https://repo.redlance.org/public") {
            content { includeVersionByRegex("(org\\.jetbrains\\.compose|io\\.coil-kt|dev\\.chrisbanes).*", ".*", ".*-webcanvas.*") }
        }
        mavenCentral()
        google()
    }
}
```

| | modules | version | targets |
|---|---|---|---|
| Compose and Material 3 | | `1.13.0-beta01-webcanvas.5` | `wasmJs`, `js` |
| `components-resources` | | `1.13.0-beta01-webcanvas.1` | `wasmJs` |
| `coil` | `coil`, `coil-core`, `coil-compose`, `coil-compose-core`, `coil-network-core`, `coil-network-ktor3` | `3.6.3-webcanvas.1` | `wasmJs` |
| `haze` | `haze`, `haze-blur`, `haze-utils` | `2.1.0-webcanvas.1` | `wasmJs` |

It needs Kotlin 2.4.20 or newer, and the fork of Haze needs Compose `1.13.0-beta01-webcanvas.5` or newer. Keep declaring upstream's versions, Compose `1.13.0-beta01`, Coil `3.6.3` and Haze `2.1.0`, and the `org.jetbrains.compose` Gradle plugin, as the fork publishes none. Then add this rule to the `build.gradle.kts` of every module with a web target. Other targets stay on upstream:

```kotlin
configurations.all {
    val isWeb = name.startsWith("wasmJs") || name.startsWith("js") || name.startsWith("web")
    resolutionStrategy.eachDependency {
        val isWasmModule = "-wasm" in requested.name
        when (requested.group) {
            "org.jetbrains.compose.ui",
            "org.jetbrains.compose.runtime",
            "org.jetbrains.compose.foundation",
            "org.jetbrains.compose.animation",
            "org.jetbrains.compose.material3" -> if (isWeb) useVersion("1.13.0-beta01-webcanvas.5")
            "org.jetbrains.compose.components" ->
                if (requested.name == "components-resources-wasmJs") useVersion("1.13.0-beta01-webcanvas.1")
            "io.coil-kt.coil3" -> useVersion(if (isWasmModule) "3.6.3-webcanvas.1" else "3.6.3")
            "dev.chrisbanes.haze" -> useVersion(if (isWasmModule) "2.1.0-webcanvas.1" else "2.1.0")
        }
    }
}
```

A module of these libraries that isn't in the table fails to resolve. On `js` the libraries stay upstream's and fail where they use Skia, unless you build their forks yourself.

## Limitations

- **It is not pixel-identical.** Text differs, because the browser draws it. In October 2026, before the fork's first release, scenes of shapes matched almost exactly (0.1–0.3% of pixels differed visibly in `canvas` and `layers`), scenes with some text differed in 1–5% of their pixels, and `text` in 15%; this hasn't been measured again since. Scrolling scenes can't be compared this way, since the faster build has scrolled further.
- **In Firefox, soft-light is drawn as hard-light**, and looks different, where it is drawn with a paint or blends a render effect that fills with a color, a gradient or a tiled image, Haze's noise included: Firefox's accelerated Canvas 2D can't draw soft-light, and drawing it in software took the whole canvas off acceleration. These stay in soft-light, and fall back to software in Firefox: layers composited in it (`softlightlayers` above), tints in it, the other blends of render effects, composite shaders in it, text with a shadow, and paints whose color filter is a tint applied as a compositing operation.
- **Libraries that use Skia fail where they use it.** The app still builds, and a call into `org.jetbrains.skia` throws `IrLinkageError` in the browser. Forks without Skia: [`components-resources`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13/components/resources), [`coil`](https://github.com/ComposeWebWithoutSkiko/coil/tree/web-canvas/3.6) (without `coil-svg`) and [`haze`](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.1) (without Glass, and its noise is a tile). They are published for `wasmJs`; on `js` they are built from source.
- **Skia-only features are gone**, such as Skottie for Lottie animations.
- **An SVG resource goes through an `<img>`:** an SVG with `<foreignObject>` isn't shown, and one without a size of its own gets the browser's natural size rather than none.
- **A `ResourceReader` of your own** serves images, SVG, XML and fonts through `getUri`, which must be a URL the browser can fetch.
- **Measured on one machine.** `wasmJs` needs WasmGC (Safari 18.2+), so older browsers need the `js` build. Windows and Linux are not measured.

## Repositories

- [`compose-multiplatform-core`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform-core/tree/web-canvas/1.13), branch `web-canvas/1.13`: Compose, with the renderer in `compose/ui/ui-web-canvas`.
- [`compose-multiplatform`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13), branch `web-canvas/1.13`: `components-resources` without Skia.
- [`coil`](https://github.com/ComposeWebWithoutSkiko/coil/tree/web-canvas/3.6), branch `web-canvas/3.6`: images through the browser's `fetch` and `createImageBitmap`.
- [`haze`](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.1), branch `web-canvas/2.1`: Haze without Skia.
- [`compose-web-benchmark`](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark): the numbers above.
