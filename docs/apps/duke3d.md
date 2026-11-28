# Duke Nukem 3D

Duke Nukem 3D (the compacted shareware episode, E1L1–E1L4) runs on ESP32-S3
badges from the package `com.micropythonos.duke3d`. It is not preinstalled:
install it from the App Store.

The engine is the DukeNano3D port of Chocolate Duke3D, built as a
**flash-resident core** — the same `mpos_core` ABI as [Doom](doom.md). The
~570 KB blob is linked for fixed addresses, stored as a littlefs2e extent in
the app folder and executed from flash through the instruction cache, so a
Duke-sized engine never has to fit in executable RAM.

| Host | Engine file | How it runs |
|------|-------------|-------------|
| ESP32-S3 badge | `esp32s3.xip` | flash-resident core, mapped from a littlefs2e extent (or the `retro-core` partition for a development badge) |
| x86-64 desktop | — | there is no desktop Duke build yet |

The GRP ships deflated (`E1L1-4_compromise.grp.zip`, 1.7 MB) and is inflated
into a buffer outside MicroPython's garbage collector at each launch (2.6 MB);
nothing is written to storage. The core reads it through the ABI. The engine's
own task runs on core 0 while the activity presents its frames on the
MicroPython task, at **17–19 fps** on an ESP32-S3.

The first frame is about 22 s away on an S3. The GRP is inflated, the engine
starts, and then the level's 605 textures are decoded before the first tic —
15 s of that is the texture preload alone. A wait that long has to say what it
is doing, so the loading screen shows the phase and a percentage
("Textures 40%"), and a save or a load does the same over the picture
("Saving 42%").

## Requirements

- A badge needs a MicroPythonOS with ABI 11 (`mpos_core.start_task` with the
  `progress=` callback, the worker task) and the 24 KiB MicroPython task stack.
  There is no compatibility path for older firmware: the framework API is not
  frozen (API level 0), and the loader simply refuses the core on a host that is
  older than it.
- Duke needs about 7 MB of PSRAM for the three big blocks it keeps at once:
  the inflated GRP (2.6 MB), the engine arena (1.9 MB) and its heap (2.5 MB on
  a fresh boot). On a fragmented board the heap shrinks; a start below 768 KB is
  refused with "restart the badge" rather than left to fail in the renderer.
- Sound needs an i2s output (Settings → Audio). The engine's own audiolib mixes
  the effects at 11025 Hz, and the same ring carries the music: the OPL3
  emulator plays the GRP's `.mid` files with the Adlib timbre bank, so the
  five songs of the compacted episode (WATRWLD1, STREETS, STALKER, GRABBAG,
  DETHTOLL) play through the engine's menus and levels. A board with no such
  output is silent and spends no time on audio (the engine refuses every sound,
  and never loads or renders a song, before the host has asked for samples).
  The in-game sound menu's MUSIC ON/OFF and MUSIC VOLUME work as they do on a
  desktop.

## Controls

On a badge, hold it sideways while playing. The touch controls are outlined and
labelled for a few seconds at start and whenever the game resumes.

| Touch | Action |
|-------|--------|
| left half | walk and turn: touch anywhere and drag from there |
| right, top row | X crouch, Y use/open |
| right, middle | A fire (hold), B jump |
| right, bottom | START menu, MENU next weapon |

| Button | Action |
|--------|--------|
| joystick | walk and turn |
| A | fire (hold) |
| B | jump (hold) |
| X | crouch (hold) |
| Y | use/open (hold) |
| START | menu (tap) |
| MENU | next weapon (tap) |

The on-screen pad is labelled with the same button letters, so the two maps are
the same map. In the engine's menus the stick moves the cursor and A or Y
confirm; START goes back (and the engine's Quit closes the game). The OS back
key and the edge swipes are off while Duke is foreground — the level is not
lost to a stray X or a thumb on the edge.

On a desktop there is no pad to label, so the keys are these (and the list is
shown over the picture for a few seconds at launch, with F1 to bring it back):

| Key | Action |
|-----|--------|
| arrows | walk and turn |
| A | fire |
| B | jump |
| X or Z | crouch |
| Y or Space | use / open |
| Home | next weapon |
| Esc | menu |
| F1 | show or hide the key list |

The letters are the action pad's, so a player switching between a badge and a
computer does not have to learn a second map. Esc is the exception: a keyboard
has no X button, and Esc is where everyone looks for the menu. In the engine's
menus the arrows move the cursor, A or Space confirms and Esc goes back.

Settings, saves and the engine's config live in
`prefs/com.micropythonos.duke3d/`:

| Engine path | On the badge |
|---|---|
| `/config/duke3d.cfg` | `duke3d.cfg` |
| `/saves/duke3d/game0.sav` | `duke3d/game0.sav` |

Saving is a menu action (START, then SAVE GAME), and the engine's ten slots are
five hundred kilobytes each, so a handful of saves is what the 7 MB filesystem
holds. The engine runs on its own task and cannot touch the filesystem itself,
so a save is a request the activity answers from its frame loop (see
[Native C/C++ Apps](native-apps.md#the-engines-files-a-blocking-handshake)).
It is flash-bound and takes tens of seconds for a full slot, so the picture
says "Saving 42%" from the blocks as they land; a load says the same, and a
first save into an empty slot shows the bytes instead of a percentage.

## How a frame is made

- The loader creates a FreeRTOS task for the core (`MPOS_CORE_FLAG_WORKER`,
  ABI 9); it never enters MicroPython. Input is a C ring, the GRP is served by
  pointer from the inflated buffer, the log is a ring the activity drains, and
  the clock is `esp_timer`.
- Its start-up cannot touch the GUI, so the phases come back as data: ABI 11's
  `progress(host, permille, text)` is forwarded by `start_task(progress=...)`
  from the host's task while it waits, and the activity paints "Textures 40%".
  See [Native C/C++ Apps](native-apps.md#a-start-up-that-takes-seconds-report-progress-abi-11).
- The core renders into one of two buffers and calls `frame_ready()`; the
  activity shows the published buffer with `NativeDisplay.select_buffer()` +
  `flush()` and hands it back with `task_presented()`. Rendering and the panel
  transfer overlap.
- The worker stack is 48 KiB (peak ~38 KB, reached in play, not at start-up).
  A board whose internal RAM cannot give 48 KiB contiguous, like the fri3d
  2024 and 2026, gets the stack from PSRAM instead.
- A frame is ~47 ms of engine (render ~35 ms + palette convert ~12 ms), and the
  panel transfer runs on the other core. `c_mpos/duke3d/PERFORMANCE.md` has the
  measurements.

## Panels it runs on

The core always renders its 320x200 frame; the host's surface says how to show
it. A panel that holds the frame gets it as it is (320x240 badges letterbox
20 rows), a portrait panel gets it transposed (200x320), and a panel that fits
neither - the fri3d 2024 is **296x240** - gets the **middle** of the picture at
full resolution: 296x200, so 12 pixels are dropped at each edge and nothing is
resampled. Cropping keeps every remaining pixel exactly as the engine drew it;
scaling the same frame to 296x185 would blur all of them to save 24 columns, and
the app did that for one build before the crop replaced it. A panel shorter
than 200 rows loses the same amount top and bottom. Below 160x100 it refuses
with a message.

On the fri3d 2024 that is **19-20 fps** at 296x200, with the repeats of its
keypad switched off while the engine is foreground. Its buttons and joystick
are independent inputs, so holding fire and turning at the same time works -
the badge's driver reports every held key, not just the first one
(`drivers/indev/fri3d_2024_keypad.py`).

## Notes for developers

Duke is the second flash-resident core after Doom, and its renderer keeps
surprising a port. The traps that cost the most time:

- **Off-screen views are a second size domain.** The engine draws a security
  camera's monitor, the screen tilt used while the player is dead, or a save
  thumbnail into a *tile* with `setviewtotile()`, and sets the view from the
  tile's own dimensions. The death view is **320×320** and a GRP can declare a
  larger camera texture — bigger than the screen the row/column tables were
  sized for. The view is now clamped to `MAXYDIM` rows and `MAXXDIM` columns
  while the tile keeps its own dimensions, `ylookup` and `lastx` hold
  `MAXYDIM + 1` entries, and the ESP32 profile sizes both from
  `INTERNAL_RES_W` (320) for exactly that 320x320 view. The clamping follows
  upstream's own patch ([DynaMight1124/retro-go#31](https://github.com/DynaMight1124/retro-go/pull/31))
  rather than a port variant, so the next sync only has to reconcile the
  documented port-only extras. Without that, drawing such a view wrote
  through `globalhoriz`/`horizlookup2` and the next wall faulted in
  `hline`/`hlineasm4` (the four faces of that corruption were
  `LoadProhibited`, a rejected cache write, `LoadStoreError` and a
  divide-by-zero in `setaspect`). A port must audit every `setview` call for a
  size that comes from an asset or a hard-coded tilt. The same overrun was the
  upstream "death glitch", removed by raising `MAXYDIM` to 320 (`9c46dd9e`);
  the remaining bounds are in
  [DynaMight1124/retro-go#31](https://github.com/DynaMight1124/retro-go/pull/31)
  and analyzed in
  [`c_mpos/duke3d/UPSTREAM.md`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/c_mpos/duke3d/UPSTREAM.md).
- **A tile view leaves the renderer's view offset in the tile.** `drawrooms()`
  computes `frameoffset = frameplace + viewoffset`, so the tilted death view
  (a 320×320 tile that is turned back onto the screen) leaves that offset
  pointing into the tile. `setviewback()` restores `frameplace` but not the
  offset, and everything drawn after it — the status bar, the weapon, the menus
  — belongs to the screen. On a badge the result looked like a graphics fault:
  the frame froze at the instant of death (the world kept being drawn into the
  tile, which nothing presented) and the pause menu over the dead view was
  reduced to red fragments and stripes. `setviewback()` recomputes the offset
  now, and the span guards are bounded by `frameplace` and the taller of
  `ydim`/`ydimen` — never by `frameoffset`, which describes a view, not the
  buffer being drawn into. `tests/test_graphical_duke3d_engine.py` starts a
  level, kills the player and checks that the red death view and the menu over
  it reach the screen and that no span is dropped.
- **A `reportandexit`/`Error` macro has to actually stop.** In this port
  `RG_PANIC` was a log call, so `allocache`'s "CACHE SPACE ALL LOCKED UP" path
  continued with an invalid best fit and wrote past `cac[]`. It now aborts
  (which the core turns into a clean quit with the reason logged), and the
  allocator recovers from a full cache: clear stale locks and retry, then give
  the one tile a 1×1 transparent stand-in. A full cache costs an invisible
  texture, not the app.
- **A missing ART file is not a missing tile.** The compacted GRP has no
  `tiles000.art`; the engine's fallback allocated the declared 128×400 (50 KB)
  and handed the renderer a texture whose pixels were never loaded. It is now
  1×1 transparent, dimensions and `picsiz` included. The GRP also carries PNG
  overrides (`TILE0785.PNG`, ...) that `lodepng` decodes.
- **An ART entry that cannot be read is cosmetic, not fatal.** The compacted
  GRP lists tiles it does not hold (tile 3281 is a 320×200 image with neither
  ART bytes nor a PNG override). `loadtile()` used to panic on the short read,
  which ended the launch with `core init failed: -11` — on the badge that looked
  like a hang at "Loading Duke Nukem 3D...", because a failed start never paints
  its message. The tile is zeroed now (the whole tile: keeping the partial bytes
  showed the sky as red/orange stripes) and the log names it.
- **A texture that cannot be decoded must keep its declared size.** The
  compacted GRP has no `TILES000.ART`, so everything below tile 1024 that the
  level uses comes from a PNG override — and an override is decoded with a
  buffer, which has to fit next to the tile cache. The cache is sized from the
  engine heap minus a reserve, and with the reserve too small the decode failed
  and the tile became the "invisible" 1x1 stand-in. That stand-in also *changes
  the tile's dimensions*, and Duke's parallax sky is one image split into five
  slices: `parascan()` derives the horizontal step from the base tile while
  `wallscan()` samples every slice with its own size, so a slice that shrank to
  1x1 turned the sky into vertical bands of garbage (horizontal stripes on the
  panel). Three things came out of it: the reserve leaves room for the largest
  decode, `parascan()` paints a flat sky instead of sampling slices that
  disagree, and palette PNGs are decoded *as* palette (`color_convert = 0`,
  a 256-entry lookup instead of a nearest-colour search per pixel): a 320x200
  decode needs 64 KB rather than 250 KB, which is the difference between
  fitting next to the tile cache and not. It does not shorten the load much -
  the 15 s texture preload is decoding pixels, not hunting colours - so the
  start-up progress display earns its place.
- **Menu keys are sampled once a tic.** Duke reads `KB_KeyDown` in `getinput`;
  `KeyHolds.tap()` presses and releases in one call, which the engine never
  sees. `mpos.ui.native_input.KeyHolds.tap_held(code, 150)` keeps the key down
  for the tic.
- **Diagnostics:** the activity rotates `/duke3d.log` to `/duke3d.prev.log`, so
  a reset cannot erase the run that crashed; the first lines record
  `machine.reset_cause()` and the PSRAM heap, and the core logs
  `engine/convert/present/wait`, tile decodes and heap free/largest every 5 s.
  A UART panic dump plus
  `xtensa-esp32-elf-addr2line -e c_mpos/duke3d/build_core/duke_core.elf <addr>`
  names the source line. The loader prints the core's log ring if start-up
  fails.
- **A sideloaded `.xip` on a nearly full filesystem defragments for minutes** at
  first launch (the AppStore install does it at install time). For development,
  flash the blob into `retro-core` (0x710000), delete the app's `esp32s3.xip`
  and let the app fall back to the partition.

Source and measurements: `c_mpos/duke3d/` in the main repository
(`README.md`, `PERFORMANCE.md`), and
[Native C/C++ Apps](native-apps.md) for the core ABI itself.

## Licenses

The engine is GPL v2 or later; the complete corresponding source is
<https://github.com/MicroPythonOS/MicroPythonOS/tree/main/c_mpos/duke3d>.
The game data is the shareware episode's GRP, distributed unmodified and free
of charge.
