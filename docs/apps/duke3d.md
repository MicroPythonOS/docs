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
MicroPython task, at **20–22 fps** on an ESP32-S3.

## Requirements

- A badge needs a MicroPythonOS with ABI 9 (`mpos_core.start_task`, the worker
  task) and the 24 KiB MicroPython task stack. The app checks the API and says
  "needs a newer MicroPythonOS" instead of failing silently.
- Duke needs about 7 MB of PSRAM for the three big blocks it keeps at once:
  the inflated GRP (2.6 MB), the engine arena (1.9 MB) and its heap (2.5 MB on
  a fresh boot). On a fragmented board the heap shrinks; a start below 768 KB is
  refused with "restart the badge" rather than left to fail in the renderer.
- No audio: the panels this targets have none, and the core's mixer is silent.
  The engine's sound effects are stubbed.

## Controls

Hold the badge sideways while playing. The touch controls are outlined and
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

Settings, saves and the engine's config live in
`prefs/com.micropythonos.duke3d/`.

## How a frame is made

- The loader creates a FreeRTOS task for the core (`MPOS_CORE_FLAG_WORKER`,
  ABI 9); it never enters MicroPython. Input is a C ring, the GRP is served by
  pointer from the inflated buffer, the log is a ring the activity drains, and
  the clock is `esp_timer`.
- The core renders into one of two buffers and calls `frame_ready()`; the
  activity shows the published buffer with `NativeDisplay.select_buffer()` +
  `flush()` and hands it back with `task_presented()`. Rendering and the panel
  transfer overlap.
- The worker stack is 48 KiB (peak ~38 KB, reached in play, not at start-up).
  A board whose internal RAM cannot give 48 KiB contiguous, like the fri3d
  2026, gets the stack from PSRAM instead.
- A frame is ~47 ms of engine (render ~35 ms + palette convert ~12 ms), and the
  panel transfer runs on the other core. `c_mpos/duke3d/PERFORMANCE.md` has the
  measurements.

## Notes for developers

Duke is the second flash-resident core after Doom, and its renderer keeps
surprising a port. The traps that cost the most time:

- **Off-screen views are a second size domain.** The engine draws a mirror,
  screen tilt or save thumbnail into a *tile* with `setviewtotile()`, and sets
  the view from the tile's own dimensions. The shareware GRP declares a
  **128×400** mirror and the tilt/thumbnail views ask for **320** — larger than
  the screen the row/column tables were sized for. The view is now clamped to
  `MAXYDIM` rows and `MAXXDIM` columns, `ylookup` holds 1024 entries and the
  `lastx` stores are bounded. Without that, drawing the mirror wrote through
  `globalhoriz`/`horizlookup2` and the next wall faulted in `hline`/`hlineasm4`
  (the four faces of that corruption were `LoadProhibited`, a rejected cache
  write, `LoadStoreError` and a divide-by-zero in `setaspect`). A port must
  audit every `setview` call for a size that comes from an asset. The same
  overflow exists in the upstream ESP32 profile; the reproduction and proposed
  patch are in
  [`c_mpos/duke3d/UPSTREAM.md`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/c_mpos/duke3d/UPSTREAM.md).
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
