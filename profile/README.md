# Compose Web without Skia

A fork of [Compose Multiplatform](https://github.com/JetBrains/compose-multiplatform-core) whose web targets (`wasmJs` and `js`) draw with the browser's Canvas 2D API, plus one shared WebGL 2 context where Canvas 2D can't, instead of Skia compiled to WebAssembly. Your Compose code stays the same, and `skiko.wasm` is gone.

It is an independent project, not affiliated with JetBrains, based on Compose `1.13.0-beta01`. You don't need to build Compose yourself: [it is published](#using-it).

## Demo

[emotes.redlance.org](https://emotes.redlance.org/) moves to the fork in version 2.4.0, and its footer shows the version. It is at 2.3.7 for now, which still draws with Skia.

## What it changes

The same Material 3 app, built twice, as production bundles. Medians in Chrome, on `wasmJs` unless stated. [Safari](#safari) is its own story.

| | upstream (Skia) | Canvas 2D | |
|---|---|---|---|
| Download of the first screen (brotli) | 3076 KiB | 545 KiB | 5.6× smaller |
| The same on `js` | 3074 KiB | 629 KiB | 4.9× smaller |
| First frame | 235 ms | 83 ms | 2.8× faster |
| The same on `js` | 459 ms | 236 ms | 1.9× faster |
| First frame, 4× slower CPU, 10 Mbit/s, 100 ms RTT | 3537 ms | 1045 ms | 3.4× faster |
| Fonts downloaded for 11 scripts and emoji | 28 files, 1214 KiB | none | |

Fonts need no download because the browser draws text, with the fonts of the system.

## Animations in Chrome

The frame rate is uncapped, so this is the throughput of the whole pipeline rather than the refresh rate of a display, and both builds draw the same frames. All the scenarios of the benchmark, on `wasmJs`. CPU is per frame, of the page and the GPU process together. On a 60 Hz display, CPU per frame is the column that matters.

| scenario | frames/s, upstream | frames/s, Canvas 2D | CPU, upstream | CPU, Canvas 2D |
|---|---|---|---|---|
| `scroll`: a list of 100 000 messages | 1815 | 2584 | 1.43 ms | 1.01 ms |
| `layers`: 400 layers of a single shape | 220 | 738 | 17 ms | 2.22 ms |
| `fade`: 60 cards fading | 735 | 2467 | 4.00 ms | 1.16 ms |
| `churn`: the cards moving, with text that changes | 311 | 509 | 5.72 ms | 2.86 ms |
| `filters`: color matrices | 714 | 937 | 4.27 ms | 4.18 ms |
| `blur`: 24 blurred cards | 547 | 739 | 5.59 ms | 4.47 ms |
| `nested`: layers in layers | 663 | 2227 | 4.69 ms | 1.40 ms |
| `perspective`: 24 turning cards | 1694 | 1841 | 1.20 ms | 1.83 ms |
| `elevation`: shadows that change | 958 | 959 | 1.96 ms | 3.05 ms |
| `shadows`: shadows that stay | 1016 | 2520 | 1.85 ms | 1.24 ms |
| `cutshadows`: shadows of paths | 767 | 2615 | 2.65 ms | 1.21 ms |
| `cardscroll`: shadows that move | 904 | 1749 | 2.28 ms | 1.54 ms |
| `blend`: a layer blend mode | 328 | 2770 | 8.71 ms | 1.09 ms |
| `canvas`: 1600 shapes | 688 | 1323 | 2.15 ms | 1.75 ms |
| `text`: 12 paragraphs laid out again | 44 | 2225 | 9.56 ms | 1.01 ms |
| `effects`: shadows under a moving blur | 943 | 3223 | 1.88 ms | 1.12 ms |

**Where it loses.** `perspective` and `elevation` cost about 50% more CPU per frame than Skia, and `filters` is level. On `js`, they also draw fewer frames (`perspective` 1727 → 1273, `elevation` 1227 → 951). These are still to be fixed.

**How to read `text`.** The two builds don't lay out the same thing: upstream uses Roboto and the fork the system font, so lines break differently. The gap is real but it isn't a like-for-like 50×.

**How to read `layers`.** Fading a layer of a single shape needs no offscreen surface here, so it's the best case. `fade` and `nested` are the same work without that shortcut.

Measured in Chrome 156 on an Apple M4 Pro, 1280×800 at 2×: [full tables](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/chrome.md), including `js` and memory. The [benchmark](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark) has the app, the runners and how to measure it yourself.

## With Haze

[Haze](https://github.com/chrisbanes/haze) blurs what's behind a panel, in every frame if what's behind it moves, which makes it a hard case for a renderer. [Its fork](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.0) draws without Skia. A small app with panels of 340×340 CSS px over a moving backdrop, measured like the scenarios above, on `wasmJs`:

| | upstream (Skia) | Canvas 2D | |
|---|---|---|---|
| Download (brotli) | 2832 KiB | 346 KiB | 8.2× smaller |
| First frame | 201 ms | 63 ms | 3.2× faster |
| One frosted panel, frames/s | 726 | 4319 | 5.9× more |
| One frosted panel, CPU per frame | 2.03 ms | 0.95 ms | 2.1× less |
| Six panels of different styles, frames/s | 628 | 445 | 1.4× fewer |
| Six panels of different styles, CPU per frame | 3.16 ms | 5.94 ms | 1.9× more |

**One panel wins, six lose.** Each panel costs the fork a blur of its own, so its cost grows with the panels much faster than Skia's does. `js` gives the same picture. This is Chrome only: Safari isn't measured with Haze. [Full table](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark/blob/main/results/haze.md).

## Safari

Safari 27.2 on the same machine, at the fork's commit `3dbb0d27622`, older than the numbers above: effects have changed since. It draws at 60 Hz, so frame rates are capped, and CPU per frame is what compares the builds. The first frame is faster here too: 267 → 147 ms on `wasmJs` (1.8×), 383 → 256 ms on `js` (1.5×).

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
| Compose and Material 3 | | `1.13.0-beta01-webcanvas.1` | `wasmJs`, `js` |
| `components-resources` | | `1.13.0-beta01-webcanvas.1` | `wasmJs` |
| `coil` | `coil`, `coil-core`, `coil-compose`, `coil-compose-core`, `coil-network-core`, `coil-network-ktor3` | `3.6.3-webcanvas.1` | `wasmJs` |
| `haze` | `haze`, `haze-blur`, `haze-utils` | `2.0.1-webcanvas.1` | `wasmJs` |

It needs Kotlin 2.4.20 or newer. Keep declaring upstream's versions, Compose `1.13.0-beta01`, Coil `3.6.3` and Haze `2.0.1`, and the `org.jetbrains.compose` Gradle plugin, as the fork publishes none. Then add this rule to the `build.gradle.kts` of every module with a web target. Other targets stay on upstream:

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
            "org.jetbrains.compose.material3" -> if (isWeb) useVersion("1.13.0-beta01-webcanvas.1")
            "org.jetbrains.compose.components" ->
                if (requested.name == "components-resources-wasmJs") useVersion("1.13.0-beta01-webcanvas.1")
            "io.coil-kt.coil3" -> useVersion(if (isWasmModule) "3.6.3-webcanvas.1" else "3.6.3")
            "dev.chrisbanes.haze" -> useVersion(if (isWasmModule) "2.0.1-webcanvas.1" else "2.0.1")
        }
    }
}
```

A module of these libraries that isn't in the table fails to resolve. On `js` the libraries stay upstream's and fail where they use Skia, unless you build their forks yourself.

## Limitations

- **It is not pixel-identical.** Text differs, because the browser draws it. Scenes of shapes match almost exactly (0.1–0.3% of pixels differ visibly in `canvas` and `layers`), scenes with some text differ in 1–5% of their pixels, and `text` in 15%. Scrolling scenes can't be compared this way, since the faster build has scrolled further.
- **Libraries that use Skia fail where they use it.** The app still builds, and a call into `org.jetbrains.skia` throws `IrLinkageError` in the browser. Forks without Skia: [`components-resources`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13/components/resources), [`coil`](https://github.com/ComposeWebWithoutSkiko/coil/tree/web-canvas/3.6) (without `coil-svg`) and [`haze`](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.0) (without Glass, and its noise is a tile). They are published for `wasmJs`; on `js` they are built from source.
- **Skia-only features are gone**, such as Skottie for Lottie animations.
- **An SVG resource goes through an `<img>`:** an SVG with `<foreignObject>` isn't shown, and one without a size of its own gets the browser's natural size rather than none.
- **A `ResourceReader` of your own** serves images, SVG, XML and fonts through `getUri`, which must be a URL the browser can fetch.
- **Measured on one machine.** `wasmJs` needs WasmGC (Safari 18.2+), so older browsers need the `js` build. Firefox, Windows and Linux are not measured.

## Repositories

- [`compose-multiplatform-core`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform-core/tree/web-canvas/1.13), branch `web-canvas/1.13`: Compose, with the renderer in `compose/ui/ui-web-canvas`.
- [`compose-multiplatform`](https://github.com/ComposeWebWithoutSkiko/compose-multiplatform/tree/web-canvas/1.13), branch `web-canvas/1.13`: `components-resources` without Skia.
- [`coil`](https://github.com/ComposeWebWithoutSkiko/coil/tree/web-canvas/3.6), branch `web-canvas/3.6`: images through the browser's `fetch` and `createImageBitmap`.
- [`haze`](https://github.com/ComposeWebWithoutSkiko/haze/tree/web-canvas/2.0), branch `web-canvas/2.0`: Haze without Skia.
- [`compose-web-benchmark`](https://github.com/ComposeWebWithoutSkiko/compose-web-benchmark): the numbers above.
