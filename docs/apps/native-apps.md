# Native C/C++ Apps

Most MicroPythonOS apps are written in MicroPython, but apps can also include native extension modules written in C or C++. These modules are compiled to MicroPython natmod `.mpy` files (one per architecture) and bundled inside the app's `.mpk` like any other module.

## When to use native code

Use a native module when:

- You need more CPU performance than pure MicroPython can provide (for example, a game loop or signal processing).
- You need to call C libraries that have no MicroPython equivalent.
- You want to reuse existing C/C++ code.

For everything else, prefer plain MicroPython — native modules add build complexity and are architecture-specific.

Native is also the only way past the LVGL refresh ceiling: a widget-based game is
capped at ~30 fps (`LV_DEF_REFR_PERIOD`), while a native renderer presenting its
own surface reached 56 fps at 320x240 on the same badge. See
[Drawing](#drawing-give-lvgl-a-surface-do-not-fight-it-for-the-panel).

## How native modules work

MicroPython supports [native machine code modules](https://docs.micropython.org/en/latest/develop/natmod.html) (often called *natmods*). A natmod is a self-contained `.mpy` file that contains compiled machine code and a MicroPython module descriptor. At runtime it is imported just like a `.py` file:

```python
import sys
if sys.platform == "esp32":
    import breakout_xtensawin as breakout   # the badge
else:
    import breakout_x64 as breakout        # desktop simulator
```

MicroPythonOS places the `.mpy` file in the app folder when the `.mpk` is installed, and the app's own directory is on `sys.path` while it runs, so a bare import finds it.

## Example: `com.micropythonos.breakout`

The **Breakout** game (preinstalled on the badge and published on the AppStore) uses a native C module for both the game logic and the drawing. Its source lives in `c_mpos/breakout/` in the main repository.

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

# libgcc/libm are linked (the xtensawin module needs memset); libc is not.
LINK_RUNTIME = 1

ARCHES = x64 xtensawin

ifeq ($(ARCH),)
.PHONY: all $(ARCHES)

all: $(ARCHES)

# One object dir per architecture: dynruntime.mk defaults to BUILD=build, and
# sharing it makes the second architecture link the first one's objects.
$(ARCHES):
	$(MAKE) -f $(lastword $(MAKEFILE_LIST)) ARCH=$@ MOD=$(MOD)_$@ BUILD=build_$@
else
include $(MPY_DIR)/py/dynruntime.mk
endif
```

This builds `breakout_x64.mpy` (desktop simulator) and
`breakout_xtensawin.mpy` (ESP32-S3). `build.sh` puts the Espressif toolchain on
`PATH`, then copies both into `internal_filesystem/apps/com.micropythonos.breakout/`.

Two things the build needs that are easy to trip over:

* the **xtensa toolchain** (`xtensa-esp32-elf-gcc`) — `build.sh` points at
  `~/.espressif/tools/xtensa-esp-elf/*/xtensa-esp-elf/bin`;
* `ar` and `pyelftools` **importable by the python that runs `mpy_ld.py`** — the
  system python refuses (`pip install` is blocked by PEP 668), so point the
  build at a venv: `PATH="$PWD/tmp/venv/bin:$PATH" make`.

`sys.platform` is `"esp32"` for every ESP32 variant, so pick the module by
platform, and keep the name the app imports stable:

```python
if sys.platform == "esp32":
    import breakout_xtensawin as breakout
else:
    import breakout_x64 as breakout
```

## Bundling a native module

1. Write your C/C++ source and a `Makefile` based on the MicroPython
   `dynruntime.mk` rules.
2. Build the module for every architecture you want to support.
3. Put the output `.mpy` files in the app folder under the names the entrypoint
   imports (Breakout ships `breakout_x64.mpy` and `breakout_xtensawin.mpy`).
4. Build the `.mpk` normally. The native `.mpy` files are packaged alongside the
   Python files.

```
com.micropythonos.breakout/
├── MANIFEST.JSON
├── icon_64x64.png
├── breakout.py                 # the Activity
├── native_display.py           # bundled copy, see "Packaging" below
├── breakout_x64.mpy            # native module, imported by breakout.py
└── breakout_xtensawin.mpy
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

### A surface the engine owns, and letterboxing

An engine port usually owns its framebuffer already. Hand the surface the
buffer instead of letting it allocate one — nothing is copied, and the image
source is the engine's own pixels:

```python
buffer = core.alloc_framebuffer(320, 200)             # m_malloc in the module
surface = NativeDisplay(self.screen, buffer=buffer, width=320, height=200)
core.init(buffer, 320, 200, 0)
```

`width`/`height` default to the whole panel; a smaller surface is centred, so a
320x200 engine is letterboxed on a 320x240 badge. `stride` (bytes per row) and
`bytes_per_pixel` are exposed for C code that computes offsets itself.

`present_area(x1, y1, x2, y2, render)` renders the bands that intersect a
rectangle and invalidates only that rectangle, so the letterbox rows are never
flushed. Coordinates are surface-local; the surface position is added for you.

### Converting a legacy format, band by band

An engine that hands you 8-bit paletted or 32-bit RGBX pixels needs a conversion
on the way to RGB565, and where that conversion happens decides the frame rate.
Converting the whole frame and then presenting it puts the conversion *in front
of* the SPI transfer. Converting one band inside the render callback overlaps
them, because LVGL is already pushing the previous band:

```python
# whole frame first: 33 ms at 320x200
core.expand8_lut(src8, dst)
surface.present_all(noop)

# per band instead: 26 ms — as fast as native RGB565 (the SPI is the limit)
surface.present_all(lambda band: core.expand8_band(src8, dst, band, rows_per_band))
```

Measured on a 320x200 surface with 45-row bands: palette→RGB565 through a
prepared LUT costs 6.2 ms per frame and 32-bit RGBX 9.5 ms, while a full-frame
present alone is 27.2 ms. Converting the whole frame first and then presenting
it therefore costs 33 ms (palette) to 38 ms (RGBX), i.e. 26-30 fps; the banded
form is bounded by whichever of the two is slower and is the only one that can
stay at the present's own ~27 ms (37 fps). The conversion functions take the
band as `rows` plus the frame `width`: a "row count" treated as a pixel count
converts 200 pixels instead of 64,000 and flatters every measurement.

### How big a module's *code* can be

There is a hard limit that has nothing to do with RAM: the code section of a
native module is committed with `MP_PLAT_COMMIT_EXEC`, which on ESP32 is
`heap_caps_malloc(..., MALLOC_CAP_EXEC)`. It has to fit in **executable memory
(IRAM)**, and a booted ESP32-S3 with WiFi/BT/IDF running has only a few KB of
that free — on the lab badge the largest contiguous executable block was
**~7 KB**. Data is different: `rodata` and BSS are ordinary heap allocations and
can live in PSRAM.

So a module with 3 KB of code and 500 KB of tables loads fine, while a
Doom-class engine (140 KB of code) fails with
`MemoryError: memory allocation failed, allocating 142528 bytes` — the size of
its code section. Options if the code is big:

* put the engine in the **firmware** (`c_mpos/src` style): its code is executed
  from flash through the instruction cache, so IRAM and the Python heap are not
  involved, and it needs no writable-data rewrite;
* keep it out of MicroPython entirely (the retro-go partition path).

### Bigger engines: a core in its own flash partition

When the code simply cannot fit in executable RAM, the engine can still run
*inside* MicroPythonOS from flash. `c_mpos/core_abi.h` defines a small ABI
(`init`/`tick`/`stop` plus host callbacks for frame bands, palette, keys, clock,
file reads, allocation and logging) and `mpos_core` loads a blob from a
partition:

1. the core is linked for **fixed addresses** and flashed into a partition (the
   badge reserves `retro-core` for exactly this);
2. the loader reads the blob's entry table through a temporary data mapping,
   maps its flash pages at those addresses and calls `init`;
3. the app's frame loop calls `tick(now)` and the core reports pixels through
   `present_band`, so MPOS keeps the board, the display, input and the activity.

A core needs three regions, all at addresses it was linked for:

* **text** in the instruction window, executed through the cache;
* **read-only data** in the data window, mapped from the *same* flash pages a
  second time: instruction memory cannot be read as data, so a string has to
  live on the data bus even though it is read-only in both places (the firmware
  splits itself the same way: `.flash.text` in IROM, `.flash.rodata` in DROM);
* a writable **arena** for `.bss`: the host allocates page-aligned RAM, zeroes it
  and maps it at the address the blob declares, so the engine's globals are
  ordinary variables.

The engine's code runs XIP through the cache like the firmware itself: no IRAM,
no executable-RAM limit, no reboot and no OTA bookkeeping. The activity keeps
the panel (`NativeDisplay`), the input and the file system; the core asks for all
three through the ABI.

The Doom port runs this way: `com.micropythonos.doomcore` starts the engine from
the `retro-core` partition, reads the WAD through MicroPython's own file objects,
rotates and converts each frame to RGB565 in C (320x200 becomes a 200x320
surface that fits the portrait panel) and draws straight into the display
surface, one band of rows at a time - the panel transfer of a band overlaps the
conversion of the next one. On a fri3d-class S3 that is 18-19 fps, split as
15 ms of engine, 5 ms of conversion and 29 ms of panel transfer: the frame is
now shared between the renderer and the SPI bus.

Controls are chosen for the hardware that exists: on a touch panel the left and
right thirds turn, the middle walks and a tap fires (the top edge opens the
menu); on a badge the joystick turns and walks, B fires, A and Y use/open and X
opens the menu. The core translates those same keys while a menu is up (fire
selects, the turn keys move the cursor), so a device without a keyboard can
still start a game.

Two things that are easy to get wrong: the host must keep the Python callbacks
alive (the collector does not trace C statics — a registered root pointer does
it), and code that runs with the flash cache disabled must be in IRAM, which is
why the mapping lets `esp_mmu_map()` own IDF's cache/MMU sequence.

On ESP32-S3 there is no PSRAM escape: `CONFIG_SPIRAM_XIP_FROM_PSRAM` only copies
flash code/rodata into PSRAM at boot, while IDF keeps `MALLOC_CAP_EXEC` on
internal DIRAM/RTC RAM only (`components/heap/port/esp32s3/memory_layout.c`), so
an executable allocation is never served from PSRAM. Big code means firmware.

`gc.mem_free()`, `gc.collect()` and heap pre-growth do not help: the failing
allocation is never a GC allocation.

### How big a module can be

A native module is loaded into the MicroPython heap at import, so a
Doom-class module is a question of RAM and import time, not of the toolchain.
Measured on the badge: a 267 KB `.mpy` (256 KB `const` tables in rodata plus
256 KB of BSS) imports in 443 ms and costs 537 KB of heap, on a boot with ~7 MB
free. Budget for the import: it is a one-off per boot, but it is a visible
pause if the app does it on launch.

### Porting an engine: writable data is the real work

A natmod is loaded with text, rodata and BSS only. There is no data section,
and `static` BSS variables are not supported either: writable file-scope data
has to be a **non-static global without an initializer**, written to only inside
functions. A Doom-class engine has thousands of file-scope variables, so this —
not the renderer — is the bulk of an engine port:

- `static int thing;` → `int thing;` (global BSS)
- `int thing = 5;` → `int thing;` plus `thing = 5;` in an init function
- `static const uint8_t table[] = {...}` → unchanged (rodata)

The probe confirms the rule at scale: it imports with a 256 KB global BSS array,
so the limit is RAM, not the linker. Keeping the engine's own globals (with the
mechanical rewrite above) is much less invasive than moving every module's state
into Python-heap bytearrays behind getters, which is what an earlier attempt did
and why it stalled.

### Loading game data: the filesystem is the slow part

Reading game assets through the normal file API is far slower than the panel:

| Read | Cost |
|---|---|
| 16 KB from LittleFS (internal, `open`/`seek`/`readinto`) | **12.5 ms** (≈1.3 MB/s) |
| 256 KB from LittleFS, one read | 257 ms (≈1.0 MB/s) |
| 16 KB from a raw flash partition (`esp32.Partition.readblocks`) | **1.6 ms** (≈8 MB/s) |
| Seek only | 0.08 ms |

So an engine that reads a lump per frame starves at ~80 lump reads per second:
a WAD has to be read once into RAM, or read from a raw partition, and a small
cache in front of the filesystem is not optional. Reads are also much cheaper in
one big call than in 4 KB chunks (4 KB × 4 costs 24.8 ms against 12.2 ms for one
16 KB call).

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

### Packaging: an `.mpk` must run on older firmware

An app installed from the AppStore runs on whatever firmware the user has, which
is older than your working tree. A frozen module you added to `lib/mpos/` is
therefore **not** available to users yet: import it and the app dies with
`ImportError: no module named ...` for everyone who is not on your build.

That is why Breakout ships `native_display.py` in its own folder and imports it
by bare name (the app's directory is on `sys.path` while it runs), while the
framework copy stays in `lib/mpos/ui/` for the apps that ship with a firmware
that has it. Keep the two identical — `tests/test_graphical_native_display.py`
compares them byte for byte.

Same rule for helpers generally: anything the app needs that is newer than the
oldest firmware you support belongs in the app folder.

## Checklist: a new native game, end to end

1. **Loop first, pixels second.** Get `tick(now)` → `present(bands, render)` →
   sound polling working with solid rectangles, on desktop (`backend='process'`)
   and on the badge, before writing game content.
2. **Clock from Python.** No `mp_import_name`, no `ticks_ms()` in C. (32 ms.)
3. **Dirty bands, not full frames.** Mark the old *and* new rectangle of
   everything that moves; clear and redraw only the bands those touch.
4. **Present from `TASK_HANDLER_FINISHED`**, up to ~5 ms early, and let
   `present()` call `lv.refr_now()`.
5. **Budget:** one band (45 rows at 320x240) ≈ 4.6 ms of SPI; two bands per
   frame ≈ 10 ms end to end, i.e. 56 fps at a 20 ms interval. A full-screen
   redraw is ~27 ms of present, so a game that changes every pixel (Doom) is a
   ~30 fps proposition; converting per band rather than per frame is what keeps
   an 8-bit engine there (26.2 ms against 33.1 ms).
6. **Tests before the badge:** band geometry, per-band row offsets (the
   offset bug that still shows bricks but moves the paddle to the top), the
   tick budget, pause/resume, sound mapping. They run on desktop and on the
   device (`MPOSController.run_test_file`).
7. **Deploy and measure:** build the module, copy the `.mpy` into the app
   folder, install, then read the app's own counters (`frames`, `bands`,
   `rows_per_band`) — do not trust a screenshot for a frame rate.
8. **Package:** bump `MANIFEST.JSON`, `make check-app-versions ARGS='--changed'`,
   `./scripts/bundle_apps.sh --app <fullname>`, install the extracted `.mpk` on
   the badge and re-measure. Then
   [Publishing Apps](publishing-apps.md).

## See also

- [Bundling Apps](bundling-apps.md) — packaging apps as `.mpk` files
- [MicroPython Native Module Documentation](https://docs.micropython.org/en/latest/develop/natmod.html)
- [Creating Apps](creating-apps.md) — writing an app from scratch
