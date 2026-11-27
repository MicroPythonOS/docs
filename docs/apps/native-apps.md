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

### A panel smaller than the frame: crop the middle, don't resample

An engine with a fixed frame size (Doom and Duke both render 320x200 whatever
the host says) cannot simply be given the panel's surface and expected to fill
it: its conversion loop ends at the surface's width, so the picture loses its
right edge - on Duke that is the right side of the status bar. Handing it a
*scaled* surface works but resamples every pixel to save a few columns.

Both cores now present the **middle** of the frame instead: a 296-wide panel
(the fri3d 2024's 296x240) gets 296x200, with 12 columns dropped at each edge
and every surviving pixel untouched. A panel shorter than the frame loses rows
the same way. Only a panel that cannot show a whole row falls back to nearest
neighbour sampling (Doom keeps that path for a 240x150 quarter scale).

The app chooses the surface and reports where it sits in the frame:

```python
class Engine:
    frame_size = (320, 200)          # the engine's own frame
    frame_window = (12, 0, 296, 200) # what the surface shows of it
```

`NativeActivity` uses `frame_window` in `to_game()`, `game_to_panel()` and
`game_rect_to_panel()`, after the letterbox and before the rotation, so a
finger still lands on the pixel it is over and an overlay lands where the
picture is. Leaving it `None` means the surface is the whole frame, which is
what an engine that scales (Doom's native module) wants.

Duke runs at **19-20 fps** on the fri3d 2024 that way (engine 52 ms, convert
8 ms, present on the other core), so it is the engine's render cost, not the
crop, that sets the pace.

### A view into a tile is a second render target

An engine that renders a room into a texture (a mirror, a tilted death view, a
thumbnail) reprograms its own state for that view: the framebuffer pointer, the
bytes per line, the view window, and the *view offset* used by the span
renderer. What it does not necessarily restore is the offset. In Duke3D
`drawrooms()` recomputes `frameoffset = frameplace + viewoffset` every time, so
after `setviewback()` the framebuffer is the screen again while the offset still
points into the tile — and anything drawn afterwards (HUD, menus) is written to
the wrong place. Check every "draw into a tile" path for the restore, and when
guarding a span write, bound it by the *current target buffer* and its height,
not by a view-relative pointer.

### The engine's files: a blocking handshake

A core that runs on its own task must never enter MicroPython, and the host's
filesystem *is* MicroPython's — so an engine's own files (its setup, its
savegames) cannot be a plain `fopen` from the core. ABI 10 adds a request the
host serves:

```python
# the core: publishes (op, path, offset, length), then blocks
count = abi->file_request(host, MPOS_CORE_FILE_WRITE, "/saves/duke3d/game0.sav",
                          offset, len, data)

# the activity, once a frame:
op, path, offset, length, buffer = mpos_core.file_poll()
with open(real_path(path), "wb" if offset == 0 else "r+b") as handle:
    handle.seek(offset)
    count = handle.write(buffer)      # a read uses handle.readinto(buffer)
mpos_core.file_reply(count)           # wakes the engine
```

Two rules make it work:

- **The buffer is the core's own memory, wrapped by reference.** `file_poll()`
  returns a `bytearray` over it (`mp_obj_new_bytearray_by_ref`), so `write()`
  and `readinto()` move the bytes without a copy. Copying a 100 KB block into a
  Python `bytes` instead grows the GC heap by an area that is never returned:
  1 MB of PSRAM per save, which is exactly the memory the *next* engine needs.
- **Answer during `init()` at once, not with the handshake.** The frame loop
  does not exist while the engine starts, so a request would sit out its timeout;
  answer -1 (file not found) and preload anything the start-up really needs.
- **Reply before doing anything that can fail.** The core is blocked for at most
  ten seconds; painting a progress label first turns an out-of-memory error or a
  torn-down screen into a lost savegame. `file_reply(count)` goes first, and the
  label is best-effort afterwards.

The engine's paths follow rg_system's layout (`/config/...`, `/saves/...`); map
them under the app's own prefs directory so an uninstall takes them along.

### A start-up that takes seconds: report progress (ABI 11)

A worker core's start-up runs before its first frame and can be long. Duke
spends 22 s getting to its first frame, and 15 s of that is decoding the level's
605 textures. The core's task must not touch the GUI, so it reports instead:

```c
if (abi->version >= 11) {
    abi->progress(abi->host, 400, "Textures");   /* permille, phase name */
}
```

`progress()` only copies the permille and the short text — it never calls back
into MicroPython from the core's task. `mpos_core.start_task(progress=cb)`
picks the newest report up from the host's task while it waits for the core to
come up, so `cb` may create and paint a label. Gate the call on the ABI version
so an older host still runs the core without progress. Report permille rather
than whole percent: a long phase that asks for percent can sit on one number
for half a second, which reads as a hang.

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

### Where that RAM lives

An asset read into RAM is several megabytes, and *which heap* holds it decides
whether the engine can even start twice:

- A Python `bytearray` of that size lives in the GC heap, which on ESP32 is a
  split heap: the collector adds areas taken from the IDF heap, and it can only
  give an area back once the area is completely empty. As soon as any other
  object lands in it, `heap_caps_malloc()` can never use that memory again -
  while `gc.mem_free()` still reports megabytes free. The engine's zone (one
  contiguous block, often 2-4 MiB, see `-mb` in the port) then has no room on
  the second launch. Keep the asset out of the GC heap: allocate it with
  `mpos_core.buffer(size)`, fill it in place with `zipasset.load(path,
  buffer=buf)`, and hand it to the engine from there.
- Ownership is explicit: `mpos_core.buffer_free(buf)` when the engine has
  stopped. Freeing returns the block to the IDF heap immediately. Holding it for
  the life of the boot is not a fix on a 7 MB device.
- Watch the **largest** free block from `esp32.idf_heap_info(esp32.HEAP_DATA)`,
  not the total: the zone is a single `malloc`, so fragmentation matters as much
  as free bytes. Refuse a launch with a message when the block cannot fit rather
  than letting the engine's allocator `I_Error` into a reset.

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

### Immersive input and touch hints

A full-screen game owns the picture, but two OS zones still sit above every app
in `lv.layer_top()`: a bar-height strip across the top (swipe down opens the
drawer) and a bar-height strip down the left edge (swipe right goes back).
Because they are above the activity's screen they win LVGL's hit test, so a
joystick that starts at the edge or a menu band at the top never receives the
touch. The activity declares the intent, and the view applies it whenever the
foreground changes:

```python
class Doom(NativeActivity):
    system_gestures_enabled = False
```

`gesture_navigation.set_system_gestures_enabled()` hides both strips **and
removes their `CLICKABLE` flag**; both are needed, because a hidden object that
is still clickable keeps winning the hit test. The hardware back key and
`Activity.onBackPressed()` keep working - only the edge swipes stop. A settings
or dialog activity pushed over the game is a normal activity, so the swipes come
back when it becomes the foreground and stop again when the game is resumed.

An app that wants to show where its zones are draws them itself; the framework
only supplies the inverse of `to_game()`:

```python
x, y = activity.game_to_panel(game_x, game_y)
panel_x, panel_y, panel_width, panel_height = activity.game_rect_to_panel(
    game_x, game_y, game_w, game_h)
```

The result includes the letterbox and swaps the sides on a rotated portrait
surface, so an overlay can be written in the game's own coordinates next to the
input zones. Doom's `doom_overlay.py` is the example: 1 px outlines and LVGL
symbols, no fills, added to the activity's screen with `CLICKABLE` removed so
the pointer still reaches the game.

Draw it as a hint, not as permanent chrome. A static overlay is repainted every
frame whether it is invalidated or not: every band the engine updates is
redrawn with the objects that intersect it. Its own cost is small - a few
milliseconds for outlines and a handful of glyphs - but Doom already sits at
the LVGL refresh budget on a busy map. Measured on the MaTouch, an overlay that
adds about 3 ms took a heavy E1M1 view from 17.9 to 11.4 fps (88 ms/frame),
while the engine's own 17-18 ms and the 6 ms conversion were unchanged: the
draw pass crossed the 33 ms refresh period, so every frame waited for one more
slot. On a light scene (engine ~7 ms) the same overlay fit and cost nothing.
Doom therefore shows the map for six seconds at start and whenever its menu
opens, then hides it; hiding is what keeps the steady-state frame rate intact.
Filling the zones would be worse, and drawing the overlay every N frames saves
nothing (the band repaint happens anyway) and only flickers.

### Key repeat is a typing feature, not a gamepad one

A keypad reports a *press*, and LVGL sends no key-release event for anything but
ENTER. An engine that wants "hold the joystick to keep turning" therefore cannot
wait for a release. `KeyHolds` used a 180 ms deadline that a repeat had to
refresh, and that fell apart on a keypad whose first repeat comes after 400 ms:
the key turned for 200 ms, stopped, and only resumed 230 ms later when the
repeat arrived. That gap is what a player feels as a repeat delay.

The clean split is the one the badge hardware already makes. Repeats stay for
the UI, where scrolling a list and moving a text cursor want them. A game sets
`key_state = True` on its activity:

```python
class Doom(NativeActivity):
    key_state = True
```

`NativeActivity` then asks `InputManager.held_keys()` every frame which keys the
hardware says are down, and `KeyHolds.hold(code, key)` is refreshed for as long
as its physical key is in that set. Pressing starts at once, releasing stops
within a frame, and the keypad's synthesised repeats are switched off while the
game is in the foreground - a repeated menu-button press would otherwise open
and close the menu ten times a second. A driver that cannot report held state
keeps the timed behaviour, and repeats with it, so the same app still works
everywhere.

`InputManager.held_keys()` returns `None` when nothing can answer. A driver that
already tracks the physical state (the fri3d 2026 expander, the communicator
keyboard) exposes `held_keys()`; a keypad that keeps reporting `PRESSED` while
a button is down needs nothing, because LVGL's last read already says so. A
driver that synthesises press/release pairs is trusted over that last read,
since LVGL has seen the release by the time the next frame runs.

`KeyHolds.tap()` is only for an engine that acts on the key **event** (Doom's
menus). An engine that **samples** its keys once a tic - Duke's menu and
next-weapon actions read `KB_KeyDown` in `getinput` - drains a press and its
release from the same frame and sees neither. `KeyHolds.tap_held(code,
hold_ms=MIN_HOLD_MS)` presses now and releases through the deadline the frame
loop already runs, long enough for one tic; Duke passes 150 ms because a heavy
frame can stretch the presenter past the default. See
[Duke Nukem 3D](duke3d.md#notes-for-developers).

### Two thumbs at once

A virtual joystick and a fire button are only useful together: a player walks
and fires at the same time. LVGL's pointer device cannot deliver that, and not
by accident - when a second finger lands, the driver reports the whole panel as
released so that a pinch cannot click the widget under it. That rule is what
keeps two-finger gestures from pressing buttons, so a game does not fight it;
it reads the driver's own points instead:

```python
class Doom(NativeActivity):
    touch_fingers = True

    def on_touch_finger(self, finger, event_code, x, y):
        ...
```

`touch_fingers` is off by default. When the panel reports more than one point,
`NativeActivity` polls it every frame and sends every finger to
`on_touch_finger()` with the same LVGL event codes a single pointer produces;
`finger` stays the same for as long as that finger is down. `on_pointer()` is
not called on such a panel at all, because the poll already covers the first
finger. A board with a single-touch controller keeps using `on_pointer()`
unchanged, so an app that sets `touch_fingers` works on both.

`mpos.ui.native_input.ThumbZones` keeps one control state per finger: the left
thumb's direction survives the right thumb landing on fire, and lifting one
finger releases only its own keys. A zone has a single owner - a second finger
that lands in the same band (or on the joystick half) is ignored rather than
sharing the key, because otherwise the first finger to lift would release it
for both.

`mpos.TouchMonitor` is the piece both apps share: one poll of the panel feeds
the per-finger events (`monitor.fingers`) and the pinch/rotation/pan view
(`monitor.gesture`), so an app that ever wants both does not run two pollers at
different rates. `NativeActivity` drives it from the frame loop; Image View,
which has no frame loop, drives it from a 30 ms LVGL timer.

The GT911 and FT6x36 drivers report their *slot* number with each point, and
that number is the finger identity. A driver that reports no number at all
still works: `TouchFingers` then matches a point to the nearest finger that is
already down, which is also how a pinch pair is selected.

Cost on the MaTouch, measured with the panel idle: LVGL's input-device timer
reads the controller 29.6 times a second (34 ms between reads, 37 ms worst) and
one `_get_coords()` costs at most 2.3 ms - 0.5 ms for the status register and
1.7 ms for five 8-byte records at 400 kHz. A press therefore reaches a frame
loop within about one read plus one frame. Reading five records even when the
status nibble claims fewer points is deliberate: on this panel the nibble can
count empty slots, and the parser drops them.

### Packaging: framework code lives in `lib/mpos`

`BuildInfo.version.api_level` is 0, so the framework API is not frozen yet: an
app is allowed to require the latest OS release, and that release may break
apps built against an older one. There is no `ImportError` compatibility
promise to keep.

Shared framework code therefore belongs in `lib/mpos/` (for example
`NativeDisplay` in `mpos.ui.native_display` and gestures in
`mpos.ui.multitouch`) and apps import it from there. Never ship a copy of a
framework module in an app folder: the copies drift, and
`tests/test_graphical_native_display.py` fails when a `native_display.py`
reappears.

The release order still matters: publish an app that needs a new framework
module only **after** the OS release that carries the module is out, so an
AppStore user who installs the app is not sent to an `ImportError` on the
current firmware. Once API level 1 exists, the compatibility policy changes.

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
