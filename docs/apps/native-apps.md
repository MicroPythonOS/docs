# Native C/C++ Apps

Most MicroPythonOS apps are written in MicroPython, but apps can also include native extension modules written in C or C++. These modules are compiled to MicroPython `.native.mpy` files and bundled inside the app's `.mpk` like any other module.

## When to use native code

Use a native module when:

- You need more CPU performance than pure MicroPython can provide (for example, a game loop or signal processing).
- You need to call C libraries that have no MicroPython equivalent.
- You want to reuse existing C/C++ code.

For everything else, prefer plain MicroPython — native modules add build complexity and are architecture-specific.

## How native modules work

MicroPython supports [native machine code modules](https://docs.micropython.org/en/latest/develop/natmod.html) (often called *natmods*). A natmod is a self-contained `.mpy` file that contains compiled machine code and a MicroPython module descriptor. At runtime it is imported just like a `.py` file:

```python
import breakout
breakout.start()
```

MicroPythonOS places the `.mpy` file in the app folder when the `.mpk` is installed. The AppManager's import path then finds it automatically.

## Example: `com.micropythonos.breakout`

The built-in **Breakout** game uses a native C module for the game logic. Its source lives in `c_mpos/breakout/` in the main repository.

Project layout:

```
c_mpos/breakout/
├── Makefile
├── build.sh
└── breakout.c
```

`Makefile`:

```makefile
MPY_DIR = ../../lvgl_micropython/lib/micropython/

MOD = breakout
SRC = breakout.c
LINK_RUNTIME = 1
ARCHES = x64 xtensawin

ifeq ($(ARCH),)
.PHONY: all $(ARCHES)

all: $(ARCHES)

$(ARCHES):
	$(MAKE) -f $(lastword $(MAKEFILE_LIST)) ARCH=$@ MOD=$(MOD)_$@
else
include $(MPY_DIR)/py/dynruntime.mk
endif
```

This builds two artifacts:

- `breakout_x64.native.mpy` — for the desktop simulator.
- `breakout_xtensawin.native.mpy` — for ESP32-S3 devices.

The `build.sh` script sets up the Espressif toolchain and then copies the resulting `.mpy` files into the app's folder at `internal_filesystem/apps/com.micropythonos.breakout/`.

## Bundling a native module

1. Write your C/C++ source and a `Makefile` based on the MicroPython `dynruntime.mk` rules.
2. Build the module for every architecture you want to support.
3. Rename or place the output `.mpy` file in the app folder so its import name matches what your MicroPython code imports. For Breakout, the `xtensawin` build is named `breakout.mpy` inside the app folder.
4. Build the `.mpk` normally. The native `.mpy` is packaged alongside the Python files.

```
com.micropythonos.breakout/
├── MANIFEST.JSON
├── icon_64x64.png
├── main.py
└── breakout.mpy      # native module, imported by main.py
```

## Limitations

- Native modules must be compiled separately for each target architecture (for example, `x64` for desktop and `xtensawin` for ESP32).
- The MicroPython natmod ABI can change between releases, so rebuild modules when the `lvgl_micropython` submodule is updated.
- Native code has the same filesystem and memory constraints as the rest of the app.

## Drawing: give LVGL a surface, do not fight it for the panel

A native renderer can beat any widget-based path, but where it draws matters more
than how fast it draws. The supported way is
`mpos.ui.native_display.NativeDisplay`: it owns one RGB565 buffer and shows it
with a full-screen `lv_image`, so LVGL stays the only writer of the panel and
only the bands of rows that changed are blitted and flushed.

```python
from mpos.ui.native_display import NativeDisplay

surface = NativeDisplay(self.screen)      # allocates width*height*2 bytes
surface.begin()
native.init(surface.buffer, surface.width, surface.height)
native.set_band_rows(surface.rows_per_band)

# per frame, from the task handler's finished callback:
native.tick(now_ms)                       # advance, mark the rows that moved
surface.present(dirty_bands(), native.render_band)   # blit + flush those rows
```

`present()` calls `lv.refr_now()` itself. Waiting for LVGL's 33 ms refresh
period would quantize the frame rate to 30 fps; the video player does the same
(see [Video Playback Performance](video-performance.md)).

### Rules for the C side

1. **Draw into `surface.buffer`, never into `main_display._frame_buffer1`.**
   The display's buffers are LVGL's draw buffers. Writing them from the app and
   flushing them with `_data_bus.tx_color()` does save the copy — and costs the
   picture: LVGL keeps refreshing its own widget tree into the same buffers and
   flushing them in between, so the screen alternates between game content and
   LVGL's (mostly empty) screen. That is visible flicker, and the interleaved
   flushes take most of the frame rate with them: the old path needed one
   task-handler pass per 45-row chunk (about 13 ms each) and reached 7.6 fps.
   Copying the frame into LVGL's draw buffer instead costs nothing measurable,
   because the blit of the next band overlaps the SPI transfer of the current
   one.
2. **Do not try to stop LVGL from refreshing.** LVGL 9 pauses the display
   refresh timer itself at the start of every pass and resumes it when something
   is invalidated, so an external `timer.pause()` is undone by the next
   invalidate. Replacing the bus callback to take over the transfers leaves LVGL
   waiting for a `flush_ready` that never comes: the next refresh spins forever
   in `wait_for_flushing` and the whole UI hangs.
3. **Report dirty rows, not the whole screen.** A band is `rows_per_band` rows,
   which is exactly what one display draw buffer holds, so one band is one blit
   and one SPI transfer. At 320x240 a full screen is 153,600 bytes (about 25 ms
   of SPI at 6.2 MB/s); one band is about 4.6 ms. A game that moves a ball and a
   paddle should present two bands, not six.
4. **Never import on the hot path.** `mp_import_name("time")` costs **32 ms** on
   the badge, because it walks the import machinery even when the module is
   already loaded. An early version of Breakout called `ticks_ms()` — which
   imported `time` on every call — once per frame step, and cost 32 ms of every
   frame no matter what else was fixed. Pass the clock in from Python instead.
5. **Only pay for the clear you need.** Re-rendering a band means clearing it
   (`memset`) and redrawing whatever reaches it; at 320x240 a band clear is
   about 2 ms, so the number of bands is what decides the frame cost.

### What it costs

Measured on an ESP32-S3 with a 320x240 ST7789 over an 80 MHz SPI bus (28800-byte
double draw buffers, so six bands of 45 rows), running Breakout, which moves a
ball and a paddle:

| | old: render into the display buffers | new: `NativeDisplay` |
|---|---|---|
| Frame rate | 7.6-13 fps across runs | **54-57 fps** |
| Rows sent per frame | 240 (the whole screen) | 2.1 bands (≈95 rows) |
| Writers of the panel | LVGL and the app | LVGL only |
| Time in the frame pump | 6 chunk round-trips, one per task-handler pass | 10 ms per frame (blit + flush) |

The frame pump is now at the panel's own limit: the app's frame interval is
50 fps (20 ms) and the pump spends 10 ms of it moving pixels, of which the SPI
transfer of two bands is 9.2 ms. Asking for more does get more, because the
transfer is what it is:

| Interval asked for | 33 ms (30 fps) | 20 ms (50 fps) | 16 ms (62 fps) | 12 ms (83 fps) |
|---|---|---|---|---|
| Measured | 33 fps | 54-57 fps | 68 fps | 84 fps |

Two more things a native renderer has to get right, both learned here:

* **A frame that is due is presented by the *nearest* pass, not the next one.**
  The frame loop runs in a task-handler pass every few milliseconds, so a
  deadline that is always checked *after* it expires slips a whole pass each
  frame: asking for 30 fps got 25. Presenting up to 5 ms early puts the pacing
  back on its grid.
* **A band is drawn at its own rows of the surface.** The surface is the whole
  screen even though one band is being drawn, so the clear and every row write
  must be offset by `band * rows_per_band`. Getting this wrong still shows
  bricks (they live in the first bands) but puts the paddle at the top of the
  screen. `tests/test_graphical_breakout.py` checks the rows of the drawn
  pixels for exactly this.

### Sound

A game that emits a beep per brick needs a sound path that costs microseconds
and never blocks the pump. On a board with a piezo on a PWM pin that is a
channel whose frequency and duty are set per event and cleared by a deadline
check in the frame loop, which is what Breakout's `Beep` does. It gets the pin
from `AudioManager.find_output_by_kind("buzzer")` and does nothing at all on a
board with no buzzer configured, so the game never depends on sound.

## See also

- [Bundling Apps](bundling-apps.md) — packaging apps as `.mpk` files
- [MicroPython Native Module Documentation](https://docs.micropython.org/en/latest/develop/natmod.html)
- [Creating Apps](creating-apps.md) — writing an app from scratch
