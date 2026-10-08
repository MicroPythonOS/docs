# App Performance

How to make MicroPythonOS apps hit smooth framerates on resource-constrained hardware. The rules below come from a real optimization pass on QuasiBird (Fri3d badge, ST7789 over SPI): **~17 → ~24.5 fps**, measured in-game.

## 1. Measure first, guess never

- Read the `sysmon:` line over serial with the FPS overlay **off**: `sysmon: 25 FPS (refr_cnt: 8 | ...)`, `refr ... (render ... | flush ...)`. The `render Xms | flush Yms` split tells you whether you're CPU-bound or bus-bound. Optimize the bigger one.
- `FPS` counts refresh *cycles* per ~300 ms window — including cycles where nothing was dirty. It is not "frames your game drew".
- FPS is hard-capped at ~30 by `LV_DEF_REFR_PERIOD=33ms`. A 16 ms game timer can't beat it; invalidates just coalesce until the next refresh tick.
- `change_task_handler(period_ms)` sets the outer pump that evaluates LVGL timers, not the refresh rate. Smaller values reduce latency but don't raise the ceiling.
- In-game FPS overlays perturb what they measure: every `set_text()` is a `malloc` + invalidate + re-blend, plus serial logging inside the measurement window. Trust the serial `sysmon` line over the on-screen number.

## 2. Know the display path

- ESP32-S3 SPI divides 80 MHz by an integer, so 60 MHz silently rounds down to 40 — 80 MHz is the only step up from 40. It only speeds up the *flush* portion of the frame; if your frame is 50 ms render + 8 ms flush, doubling the bus saves ~4 ms, not half.
- Display `freq` is per-device on a shared SPI bus. An SD card (20 MHz) or radio (16 MHz) on the same bus keeps its own clock; each transfer just waits its turn.
- Flushing is DMA-async with double buffering — LVGL renders the next chunk while the previous one transfers. The 33 ms refresh cap usually binds long before the bus does.
- `rgb565_byte_swap=True` runs an in-place CPU swap loop over every flushed chunk before DMA. Required for correct colors on most panels — just know it's part of `flush` time, not free.

## 3. No floats in hot loops

MicroPython floats are heap objects. Float physics at 60 Hz easily means ~20 short-lived allocations per frame — constant GC pressure that shows up as *jitter* (drifting fps), not just a lower average.

Use integer fixed-point instead: positions in milli-pixels, velocities in milli-pixels/s, `delta_ms` kept as `int`. LVGL takes ints anyway, so convert with `//` at the display call.

Two traps:

- **Don't truncate per frame.** `SPEED * dt // 1000` in whole pixels freezes slow movers (30 px/s × 16 ms = 0.48 px → 0). Accumulate in milli-pixels so sub-pixel motion is preserved.
- **Bound your counters.** An ever-growing scroll accumulator eventually outgrows MicroPython small ints and becomes a heap-allocated bigint on every op. Wrap with modulo (e.g. one screen width).

Verify with a float-vs-fixed simulation before flashing: identical pixel outputs (within ±1 px at wrap edges) and identical game-logic timing.

## 4. Redraw hygiene

Every invalidate costs render + flush. LVGL redraws only dirty areas, so shrink what you dirty:

- `label.set_text()` **always** mallocs and invalidates, even for an identical string. Only call it when the text actually changed. Same for any label you refresh "just in case" every frame.
- `image.set_offset_x()` invalidates the whole object unconditionally. Guard it behind a "did the pixel change?" check.
- `set_x()` / `set_y()` / `set_pos()` are no-ops when the integer pixel didn't change — call them freely.
- A permanent `set_rotation()` on an image means *every frame* any overlapping dirty rect takes the slow per-pixel transform path, plus a larger invalidated area. Ship a pre-rotated asset instead (verify pixel-identity once with PIL).
- `TILE` mode issues one draw call **per tile**. A 240 px strip from a 20 px tile = 12 draw calls per frame. Pre-compose the strip offline (tiny PNG) and scroll two leapfrogging images via `set_x` — 1 draw call each, identical pixels. Keep `TILE` as fallback if you target multiple screen widths.
- Translucent (`bg_opa` < 255) + rounded + bordered panels force read-modify-write blending, corner masking, and repainting of everything beneath them. Opaque rectangles take the fast fill path — a visual tradeoff, budget it deliberately.
- Images with transparency force the background underneath to repaint (overdraw). Opaque sprites compose cheaper.
- Slow movers (parallax clouds at ~0.5 px/frame) can push to LVGL at half rate while physics keeps integrating every frame — visually identical, half the invalidates.

## 5. Verify like-for-like

- Simulate old-vs-new logic in CPython where possible (trajectories, scroll phases, wrap behavior) before spending flash cycles.
- `ruff` and `mpy-cross` must run on the **real** file path: apps symlinked from external repos are not covered by the main repo's `make lint` / `make syntax-tests`.
- Bump the app version with every user-visible change so testers can tell builds apart.
