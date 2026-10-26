# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A deep-zoom Mandelbrot explorer in one file, `index.html`, with all the HTML, CSS, JS and shaders inline. There is no build step, no dependencies and no test suite. Rendering uses WebGPU and falls back to WebGL2.

## Running and checking

- Serve the directory with `python3 -m http.server 8931` and open `http://localhost:8931/index.html`. Don't use port 5000 (macOS AirPlay) or 8765 (another project's service worker is cached on that origin). `file://` also works, since the worker is loaded from a Blob.
- `?gl` in the query string forces the WebGL2 backend.
- The URL hash sets the view, e.g. `#re=-0.7436438870371587&im=0.131825904205312&h=1e-12`. `h` is the visible height in world units; magnification = 3/h.
- Syntax-check the script:
  `sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/m.js && node --check /tmp/m.js`
- Headless check of the WebGL path:
  `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --use-angle=metal --virtual-time-budget=15000 --window-size=900,500 --screenshot=out.png "http://localhost:8931/index.html?gl#re=...&im=...&h=..."`
  Headless WebGPU does not work on this machine, so test WebGPU in real Chrome.
- If a GPU hang crashes Chrome's GPU process, Chrome disables WebGPU and WebGL until the browser is restarted.

## Architecture

**Precision scheme (perturbation theory with Zhuoran rebasing).**
- **View center:** `view.cx` and `view.cy` are BigInt fixed-point values with `PREC = 320` fractional bits. Helpers: `fromNum`, `toNum`, `toDec`, `parseDec`.
- **Zoom:** `view.h` is a plain Number, clamped to `[MIN_H = 1e-30, MAX_H]`. The lower bound is the float32 limit: per-pixel deltas must stay above the f32 normal range.
- **Reference orbit:** an inline Web Worker (`workerSrc`) computes it with BigInt at `zoomBits + 64` bits and returns it as a Float32Array of `Z_n`. Requests are serialized: one in flight at a time, with `wantRef` queuing the next. `scheduleRef` debounces requests but forces one at least every 400 ms during long gestures.
- **Per-pixel iteration (GPU):** each pixel iterates in f32 only `dz' = 2·Z·dz + dz² + dc`. When `|z| < |dz|` or the orbit runs out, it rebases (`dz = z`, `m = 0`). The reference may lag behind the view; the shader adds `off = view − ref` to `dc`.
- **Placeholder reference:** before the first worker reply, the reference is `c = 0` with orbit `[0, 0]`, which is exact.

**Two-pass rendering.** Both backends implement the same interface: `setOrbit`, `iterTiles(rects)`, `colorize()`, `finish()`, `destroy()`.
1. **Iterate pass:** writes an `rgba32float` texture holding the smooth iteration count `mu`, the surface normal from the scaled derivative `dd = dz/dc · pixelSize`, and an escaped flag in `.w`.
2. **Colorize pass:** applies the cosine palette, color cycling and relief lighting. Palette, cycling and relief changes set `colorDirty` and re-run only this pass. View changes set `dirty`, which re-iterates.

**Shaders and uniforms.**
- The shaders exist twice: `WGSL_ITER`/`WGSL_COLOR` and `GLSL_ITER`/`GLSL_COLOR`. Any math change must be made in both.
- One 112-byte uniform block (`UB`, viewed as `F`/`U32`) feeds both backends. Its layout must match WGSL `struct U`. Some `.w` slots carry the light direction, light height and mapping mode. `org` (F[26..27]) is the chunk origin for export.
- WebGL reads the same values with individual `gl.uniform*` calls. In GL, `org.y` is converted to a bottom-up origin, and scissor rects are bottom-up too.

**Tiled, progressive rendering (`frame`, `makeJob`).**
- Work is split into 128 px tiles, center first, with one GPU submit per tile, so no command buffer trips the macOS GPU watchdog. Don't go back to full-screen draws.
- The number of tiles per frame adapts from the measured throughput `rate` (pixel·iterations per ms) to a ~25 ms frame budget.

**PNG export (`exportPNG`).**
- Renders `screen × mult` (1/2/4×) in 2048² chunks through the same pipeline: the canvas is resized to 2048, and `fillUniforms({W, H, ox, oy})` points it at each chunk.
- After each chunk it calls `colorize()` and, in the same task, `drawImage`s the result into a 2D canvas. The WebGPU current texture and the WebGL buffer (no `preserveDrawingBuffer`) are only valid within that task.
- Output is capped to the browser canvas limits (~250 MP, 32767 px per side).
- The main loop pauses while `exporting` is set.

**Iteration count.** Auto mode uses `250 + 100 · log10(zoom)^1.6`. Lower values made deep views render black: near the Seahorse-valley preset at 1e-22, pixels need ~10k iterations.
