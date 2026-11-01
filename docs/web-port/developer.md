# Web Port Developer Info

This section describes how MicroPythonOS runs in the browser, how to build and serve it, and everything you need to know to make changes. **The entire web port is self-contained in the main MicroPythonOS repository** — no edits to the `lvgl_micropython` submodule (or its nested `micropython`/`lvgl`) need to be committed. The web-only submodule modifications are stored under [`scripts/web_port/`](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/scripts/web_port/) and are applied automatically at build time.

## TL;DR

```bash
# 1. One-time: install & activate the Emscripten SDK (see "Prerequisites").
#    The build auto-activates ../emsdk or ../../emsdk if emcc is not on PATH.

# 2. Build the web target.
scripts/build_mpos.sh web

# 3. Serve web/ locally at http://localhost:8080/
scripts/run_web.sh --no-build
#   PORT=9000 scripts/run_web.sh --no-build   # serve on a different port
```

Output artifacts land in `web/`: `micropython.{html,js,wasm,data}` (and `micropython.wasm.map` if the build makes one), plus copies `index.html` and `mpos.html`.

## Prerequisites

- **Emscripten SDK** (tested with **6.0.0**). Either have `emcc` on `PATH`, or place an activated `emsdk` checkout one or two directories above this repo (`../emsdk` or `../../emsdk`). `scripts/build_mpos.sh web` sources `emsdk_env.sh` automatically when `emcc` is missing.
- Standard host toolchain to build `mpy-cross` (built with the host compiler, not emcc) and the usual MicroPython build dependencies (`python3`, `make`).
- The git submodules must be checked out (`git submodule update --init --recursive`). A **clean** submodule checkout is fine — the build re-applies the web changes every time.

## Build from a fresh clone

The entire web port lives in the main repository, so a fork is self-contained: a fresh clone plus the submodules plus an Emscripten SDK is all that is needed. The submodule C changes are re-applied automatically by the build, so **nothing needs to be committed into any submodule.**

```bash
# 1. Clone your fork with submodules.
git clone --recursive https://github.com/<you>/MicroPythonOS
cd MicroPythonOS
#   (if you forgot --recursive:)
#   git submodule update --init --recursive

# 2. Install + activate Emscripten 6.0.0 somewhere the build can find it
#    (emcc on PATH, or an activated emsdk at ../emsdk or ../../emsdk).

# 3. Build, then serve.
scripts/build_mpos.sh web
scripts/run_web.sh --no-build
```

### Pinned submodule commits (known-good)

The web patches apply to specific upstream revisions. If a submodule moves past these, a patch can stop applying. The build then stops with `FATAL: patch ... does not apply` (see [Build-time Patches](../os-development/build-patches.md#how-apply_patch-behaves)). These are the commits pinned at MicroPythonOS `main` commit `0d71bd2b` (2026-10-28). The CI web build passed with them:

| Submodule | Commit | Tag/branch |
| --- | --- | --- |
| `lvgl_micropython` | `1ab63df` | `integration` |
| `lvgl_micropython/lib/micropython` | `78ff170` | `v1.27.0` |
| `lvgl_micropython/lib/lvgl` | `c016f72` | `v9.4.0` |
| `lvgl_micropython/lib/SDL` | `6ad390fc` | 4202 commits after `release-2.26.0` |
| `lvgl_micropython/lib/esp-idf` | `fcae328` | `v5.5.1` |
| `freezeFS` | `5f211e3` | `main` |
| `secp256k1-embedded-ecdh` | `ca1c146` | `micropython_1.25.0` |
| `micropython-camera-API` | `db42861` | `master` |
| `micropython-nostr` | `04c0a2e` | `main` |

The web patches change files in `lvgl_micropython` and `lvgl_micropython/lib/micropython`, so those two pins matter most. A fork should pin at least those. The web build also applies the MicroPythonOS patches to `lib/esp-idf`, so that submodule must be checked out too. If you bump a pin and a patch stops applying, regenerate it (see *Updating a submodule patch*) and check for a clean boot.

## How it works (architecture)

The web target reuses the **unix port** of MicroPython (LVGL + SDL display/input drivers, the frozen manifest, the `ext_mod` C modules) but compiles it with the Emscripten toolchain (`emcc`/`em++`/`emar`) and links Emscripten's bundled SDL2 port (`-sUSE_SDL=2`) instead of a natively built `libSDL2.a`. The result renders into an HTML `<canvas>`.

```
┌─────────────────────────────────────────────────────────────┐
│ Browser tab (web/mpos.html → micropython.js + .wasm + .data) │
│                                                              │
│   <canvas>  ◀── SDL2 (Emscripten port) ◀── LVGL ◀── MPOS     │
│                                                              │
│   asyncio event loop (TaskManager.start → asyncio.run)       │
│     ├─ task_handler  (drives lv.task_handler + lv.tick_inc)  │
│     └─ machine.Timer  (asyncio-backed periodic/one-shot)     │
└─────────────────────────────────────────────────────────────┘
```

Key design points:

- **`REAL_PORT = 'unix'`** — the web build reuses the unix port, its builder patches and `machine_sdl`. `web.py` overrides three of the unix builder's patch hooks to drop `machine_timer` and the bluetooth, termios, socket and ffi sources.
- **No native threads / sockets / ffi / termios / bluetooth.** The web build sets `MICROPY_PY_THREAD/SOCKET/FFI/TERMIOS/BLUETOOTH/BTREE=0`. `_thread` and `socket` get small web-only shims (see below). Importing `ffi`, `termios`, `bluetooth` or `btree` raises `ImportError`.
- **No native `machine.Timer`.** `machine_timer.c` is dropped from the web Makefile; an asyncio-backed `Timer` is injected at boot.
- **The preloaded filesystem is mounted at `/`** (root), matching the on-device layout, because `main.py` does `sys.path.insert(0, "lib")` and apps use root-relative paths like `/apps` and `/builtin`.

## Where everything lives (all in this repo)

| Path | Purpose |
| --- | --- |
| [`scripts/build_mpos.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/build_mpos.sh) | Central build orchestration. The `web` target branch does all web-specific work (patching the submodule, staging the FS, copying in shims, invoking `make.py web`, collecting artifacts). |
| [`scripts/run_web.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/run_web.sh) | Build (optional) + serve `web/` with `python3 -m http.server`. |
| [`scripts/web_port/web.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/web_port/web.py) | The Emscripten build backend. Copied into `lvgl_micropython/builder/web.py` at build time; consumed by `make.py web`. |
| [`scripts/web_port/*.patch`](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/scripts/web_port/) | The four web-port patches: `make.py.patch`, `lcd_bus_micropython.mk.patch`, `sdl_bus.h.patch`, `gccollect.c.patch` (see "Submodule patches"). |
| [`scripts/web_port/ext_mod/`](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/scripts/web_port/ext_mod/) | The web-only C modules `_webnet`, `_webterm` and `_webio`. Copied into `lvgl_micropython/ext_mod/` at build time. |
| [`scripts/web_port/staged_lib/`](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/scripts/web_port/staged_lib/) | Web-only Python shims. Copied into the staged `lib/` at build time (see "Web-only Python shims"). |
| [`scripts/web_port/inject_web_machine_timer.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/web_port/inject_web_machine_timer.py) | Adds the `machine.Timer` injection to the staged `main.py` (see "`machine.Timer` injection"). |
| [`web/shell.html`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/web/shell.html) | The HTML shell template (`--shell-file`). Build copies the produced `micropython.html` to `index.html` and `mpos.html`. |
| `web/.preload_internal_filesystem/` | Auto-generated staging copy of `internal_filesystem/` (with web shims added). Recreated on every build; do not edit by hand. |
| `web/.preload_bundled_apps/`, `web/.preload_bundled_data/` | Auto-generated. The bundled apps and the contents of `internal_filesystem_data/`, packaged apart from the main tree (see "Persistence"). |

`build_mpos.sh` copies the web-only Python shims from `scripts/web_port/staged_lib/` into the staged filesystem. `inject_web_machine_timer.py` adds the boot-time `machine.Timer` injection to the staged `main.py`. None of this is in `internal_filesystem/`, so device builds are unaffected.

## Submodule patches (applied automatically)

> **Note:** The general mechanism (how `apply_patch` works, adding or regenerating a patch, and the `lvgl_micropython` patch family) is documented once in [Build-time Patches](../os-development/build-patches.md). This section covers only what is specific to the web target.

The web target makes eight changes inside the `lvgl_micropython` submodule. They are stored in the MicroPythonOS repo, not committed to the submodule. The `web` build applies the four patches with `apply_patch` and copies in the other files on every build:

1. **`builder/web.py`** (full file) — the Emscripten build backend. Copied from `scripts/web_port/web.py`.
2. **`make.py`** — `make.py.patch` adds `web` to the build targets and loads `builder/web.py` for it.
3. **`ext_mod/lcd_bus/micropython.mk`** — `lcd_bus_micropython.mk.patch` adds an `MPOS_WEB=1` branch. It builds `lcd_bus` against Emscripten's SDL2 port (`-sUSE_SDL=2`) instead of a natively built SDL2.
4. **`ext_mod/lcd_bus/sdl_bus/sdl_bus.h`** — `sdl_bus.h.patch` adds a missing `uint32_t buffer_flags;` field to `mp_lcd_sdl_bus_obj_t` so its layout matches the generic `mp_lcd_bus_obj_t`. `lcd_panel_io_init()` casts the SDL object to the generic type and calls `panel_io_handle.init(...)`; without this field the offset of `panel_io_handle` differs on 32-bit/wasm and the indirect call reads the wrong function pointer → `function signature mismatch` trap. (On 64-bit native, struct padding hid the bug.)
5. **`lib/micropython/ports/unix/gccollect.c`** — `gccollect.c.patch` adds an `__EMSCRIPTEN__` branch to `gc_collect()` that uses `emscripten_scan_stack()` + `emscripten_scan_registers()` instead of the setjmp-based `gc_helper_collect_regs_and_stack()`. On wasm, live object pointers live in wasm locals/registers (not linear memory) and are invisible to a memory scan, so the stock collector freed in-use objects → `memory access out of bounds`. This matches the upstream MicroPython `ports/webassembly` approach and **requires ASYNCIFY** (so registers can be spilled), enabled via `-sASYNCIFY=1` in `web.py`.
6. **`ext_mod/_webnet/`** (new files, not a patch) — the browser `fetch()` and `WebSocket` bridge for networking. `webnet.c` + `micropython.mk` are copied from `scripts/web_port/ext_mod/_webnet/`; the `.mk` only compiles it when `MPOS_WEB=1`. See *Networking* below.
7. **`ext_mod/_webterm/`** (new files, not a patch) — a byte bridge between the browser and MicroPython's stdio. An external host such as ViperIDE can use it to drive the REPL like a serial device. Copied from `scripts/web_port/ext_mod/_webterm/`; built only when `MPOS_WEB=1`.
8. **`ext_mod/_webio/`** (new files, not a patch) — the badge peripheral bridge. NeoPixel output goes to the LED dots on the page, and the on-page joystick and X/Y/A/B/MENU/START buttons act as input. It also plays sound through the browser's Web Audio API: WAV files for `mpos.audio`, and tones for the `PWM` shim. Copied from `scripts/web_port/ext_mod/_webio/`; built only when `MPOS_WEB=1`.

The web build also applies `unix_autoimport_main.patch` and `unix_native_decorator_fallback.patch`. These are lvgl_micropython patches shared with the desktop builds, so [Build-time Patches](../os-development/build-patches.md#current-lvgl_micropython-patches) covers them.

Before its own build, every other target runs `reset_web_port_changes` in `build_mpos.sh`. It deletes `builder/web.py` and the three `ext_mod/_web*` directories, and checks out the four patched files. So these changes do not leak into device or desktop builds. A new web-port patch to another file must be added to that function too.

There is one edit that the function does not undo. During a web build, `web.py` removes lines from `lvgl_micropython/lib/micropython/ports/unix/Makefile`, among them `modtermios.c` and `modffi.c`. A later desktop build in the same checkout does not put all of them back, so that binary lacks some modules, such as `termios` and `ffi`. To avoid this, run `git -C lvgl_micropython/lib/micropython checkout -- ports/unix/Makefile` before the desktop build.

### Updating a submodule patch

To change a patched file, start from the patched version, or the regenerated patch holds only your new edit. A `web` build leaves the four files patched. A build for any other target checks them out again. In that case, run `scripts/build_mpos.sh web` first, or apply the patch by hand, for example `patch -p1 -d lvgl_micropython < scripts/web_port/make.py.patch` (use `-d lvgl_micropython/lib/micropython` for `gccollect.c.patch`). Then edit the file in the submodule, and regenerate its patch from the directory the patch is applied in. Run only the line for the file you changed: a line for a file that is not patched at that moment writes an empty `.patch`.

```bash
cd lvgl_micropython
git diff -- make.py > ../scripts/web_port/make.py.patch
git diff -- ext_mod/lcd_bus/micropython.mk \
    > ../scripts/web_port/lcd_bus_micropython.mk.patch
git diff -- ext_mod/lcd_bus/sdl_bus/sdl_bus.h \
    > ../scripts/web_port/sdl_bus.h.patch

cd lib/micropython
git diff -- ports/unix/gccollect.c \
    > ../../../scripts/web_port/gccollect.c.patch
```

`web.py` and the `ext_mod/_web*` modules are copied, not patched. Edit them in `scripts/web_port/`: every web build copies them over the submodule copies.

To check that the build reproduces everything, undo the web changes in the submodule and rebuild. These commands do what `reset_web_port_changes` does:

```bash
( cd lvgl_micropython \
    && git checkout -- make.py ext_mod/lcd_bus/micropython.mk \
        ext_mod/lcd_bus/sdl_bus/sdl_bus.h \
    && rm -f builder/web.py make.py.rej ext_mod/lcd_bus/micropython.mk.rej \
        ext_mod/lcd_bus/sdl_bus/sdl_bus.h.rej \
    && rm -rf ext_mod/_webnet ext_mod/_webterm ext_mod/_webio )
( cd lvgl_micropython/lib/micropython && git checkout -- ports/unix/gccollect.c )
scripts/build_mpos.sh web
```

The build log must show each web patch being applied (`patching file ...`). CI builds the web target on every push and pull request to `main`, so a patch that stops applying fails CI.

## Build flags (in `scripts/web_port/web.py`)

The Emscripten link flags (`web_ldflags`) are the main tuning surface:

| Flag | Why |
| --- | --- |
| `-sUSE_SDL=2` | Use Emscripten's bundled SDL2 port. |
| `-sALLOW_MEMORY_GROWTH=1`, `-sINITIAL_MEMORY=...`, `-sMAXIMUM_MEMORY=...` | Size of the wasm memory: 256 MiB at start, can grow to 2 GiB. The MicroPython heap is a fixed 16 MiB block inside that memory, set by `-X heapsize=16M` in `web/shell.html`. |
| `-sSTACK_SIZE=8388608 -sSTACK_OVERFLOW_CHECK=2` | Larger stack for deep LVGL/MPOS call chains, with stack overflow checks. |
| `-lidbfs.js` | IndexedDB-backed filesystem for `/data` and `/apps` (see "Persistence"). |
| `-sASYNCIFY=1 -sASYNCIFY_STACK_SIZE=32768` | **Required** for `emscripten_scan_registers()` in the GC fix. |
| `-sFORCE_FILESYSTEM=1 -sEXIT_RUNTIME=0` | Link Emscripten's full `FS` API, which `shell.html` and `_webio` call from JavaScript. Keep the runtime alive after `main()` returns. |
| `-Wl,--allow-multiple-definition` | MicroPython's libc `printf` vs Emscripten libc `printf` duplicate. |
| `-sASSERTIONS=2 --profiling-funcs` | **Debug only.** Symbolized stacks. Remove for an optimized production build. |

Module disabling and SDL include/link selection happen via `MPOS_WEB=1` and the `MICROPY_PY_*=0` make variables, also set in `web.py`.

## Web-only Python shims (staged into `lib/`)

`build_mpos.sh` copies these from `scripts/web_port/staged_lib/` into `web/.preload_internal_filesystem/lib/`. There they shadow frozen/native modules at runtime without affecting device builds:

| Shim | Replaces | Behavior |
| --- | --- | --- |
| `_thread.py` | native `_thread` | Cooperative: runs thread bodies as asyncio tasks; locks are no-ops; `get_ident()==1`. |
| `socket.py` | native `socket` | Stub; raises `OSError` on use (no browser raw sockets). |
| `_webrepl.py` / `websocket.py` | native modules | Stubs; raise `OSError` on use. |
| `aiorepl.py` | `aiorepl` | Reads REPL input from the `_webterm` bridge instead of `sys.stdin`, which the browser cannot read. |
| `aiohttp/__init__.py` | `aiohttp` | HTTP and WebSocket client on top of `_webnet` (see "Networking"). |
| `neopixel.py` | `neopixel` | Sends the LED buffer to the dots on the page via `_webio`. |
| `web_expander.py` | Fri3d 2026 expander | Reads the on-page joystick and buttons via `_webio`, so the stock expander input driver works unchanged. |
| `task_handler.py` | frozen `task_handler` | Drives `lv.task_handler()` + `lv.tick_inc()` from an asyncio task instead of `machine.Timer`. Same public API (`TaskHandler`, `TASK_HANDLER_STARTED/FINISHED`, `add_event_cb`, `disable/enable`, `deinit`). |
| `_web_machine_timer.py` | `machine.Timer` | asyncio-backed periodic/one-shot timer. |
| `_web_machine_pin.py` | `machine.Pin` | In-memory pin that does no hardware I/O. |
| `_web_machine_hw.py` | other `machine` names | Stubs such as `ADC`, `I2C`, `SPI`, `UART`, `RTC`, `WDT` and `reset()`. `PWM` is a stub too, except when its pin is a `Pin` object with id -1: then it plays tones through `_webio`. |

### `machine.Timer` injection

The native `machine` module dict is read-only, so `machine.Timer = ...` fails. Instead, `scripts/web_port/inject_web_machine_timer.py` patches the **staged** copy of `main.py` (never the device source) right after `sys.path.insert(0, "lib")`. The added code replaces `sys.modules["machine"]` with a thin wrapper. The wrapper exposes `Timer`, and `Pin` if the native module has none. Other names come from the native module, or from `_web_machine_hw` if the native module lacks them.

## Networking (HTTP via browser `fetch()`)

The browser has no raw TCP/UDP sockets, so `socket` is stubbed and `MICROPY_PY_SOCKET=0`. HTTP instead goes through the browser's `fetch()` API via a small native module plus an `aiohttp` shim:

| Piece | Location | Role |
| --- | --- | --- |
| `_webnet` native module | `scripts/web_port/ext_mod/_webnet/webnet.c` (+ `micropython.mk`) | C/`EM_JS` bridge to the browser `fetch()` and `WebSocket` APIs. Non-blocking, poll-based so the asyncio/UI loop keeps running. Built only for web (`MPOS_WEB=1`); copied into `lvgl_micropython/ext_mod/_webnet/` by `build_mpos.sh`. |
| `aiohttp` shim | `scripts/web_port/staged_lib/aiohttp/__init__.py`, copied to the staged `lib/aiohttp/__init__.py` | Re-implements `ClientSession.get/post/put/...` on top of `_webnet`'s fetch bridge, and `ClientSession.ws_connect()` on top of `_webnet`'s WebSocket bridge, polling with `await asyncio.sleep_ms(...)`. Imports `WSMsgType` from the device `aiohttp_ws.py`. |

`_webnet` HTTP API: `fetch_start(method, url, headers_json, body)` returns an int handle; then `poll(h)` (0 pending / 1 done / -1 error), `status(h)`, `headers(h)`, `body(h)`, `error(h)`, and `free(h)`.

`_webnet` WebSocket API: `ws_open(url, protocols_json)` returns a handle; `ws_state(h)` (0 connecting / 1 open / 2 closing / 3 closed), `ws_peek_type(h)` (0 none / 1 text / 2 binary), `ws_peek_len(h)`, `ws_read(h)` (pops the front message as bytes), `ws_send_text(h, str)`, `ws_send_bytes(h, bytes)`, `ws_close(h)`, `ws_error(h)`, and `ws_free(h)`.

**Limitations:**

- **CORS applies.** Cross-origin requests fail unless the server sends `Access-Control-Allow-Origin`. The default app-store/OTA hosts do not, so those downloads fail in the browser with `fetch failed: TypeError: Failed to fetch` — this is a server-side policy, not a port bug. Same-origin or CORS-enabled endpoints work.
- **WebSockets work**, via the browser `WebSocket` API (`ClientSession.ws_connect`). Unlike `fetch()`, cross-origin WebSockets are not blocked by CORS (the server decides via the `Origin` header), so e.g. Nostr relays connect. Limitation: the browser WebSocket API cannot set custom request headers, so any `headers` passed to `ws_connect`/`ClientSession` are ignored for the WS handshake.

## Persistence (writable FS via IndexedDB / IDBFS)

The bulk of the filesystem (`/lib`, `/builtin`, `main.py`) is baked into `micropython.data` at link time and loaded into memory at boot. Writes there work, but are lost when the page reloads. Two paths are instead mounted from the browser's IndexedDB so writes survive a page reload:

| Path | Backing | Contents |
| --- | --- | --- |
| `/data` | IDBFS (IndexedDB) | User data such as images and audio files, seeded from `internal_filesystem_data/` (see below). |
| `/apps` | IDBFS (IndexedDB) | User-installed apps (`AppManager` installs to `apps/<fullname>`) and a one-time copy of the bundled demo apps. |

How it is wired up:

- **Link flag:** `-lidbfs.js` is added to `web_ldflags` in `scripts/web_port/web.py` so the IDBFS backend is available.
- **Mount + load (boot):** `web/shell.html`'s `Module.preRun` mounts IDBFS at `/data` and `/apps`, then gates the runtime start on `FS.syncfs(true, …)` (wrapped in `addRunDependency`/`removeRunDependency`) so Python only starts once the persisted contents are loaded from IndexedDB.
- **Why those two paths are excluded from the preload:** `FS.syncfs(true)` reconciles the in-memory FS to match the IndexedDB store, so a mount point must not also be a `--preload-file` target — otherwise the first boot (empty store) would wipe the preloaded files. `build_mpos.sh` therefore stages the tree **without** `apps/` and `data/`. The bundled demo apps are packaged separately at `/.bundled_apps` (`--preload-file …@/.bundled_apps`). `data/` is dropped. Instead, the contents of `internal_filesystem_data/` are packaged at `/.bundled_data`.
- **Seeding bundled apps:** on first run `seedBundledApps()` copies `/.bundled_apps` into `/apps` once and writes a `/apps/.seeded` marker, so the bundled apps appear via the normal `AppManager` scan. The marker means a user uninstalling a bundled app makes the removal stick across reloads instead of being re-seeded every boot.
- **Seeding bundled data:** on every boot `seedBundledData()` copies `/.bundled_data` into `/data`. It only copies files that are not there yet, so it never overwrites a file in `/data`.
- **Flushing writes:** `startPersistFlush()` periodically calls `FS.syncfs(false, …)` (every few seconds, plus on `pagehide` and when the tab is hidden) to push writes back to IndexedDB.

Notes / limitations:

- Persistence is per-origin and subject to the browser's IndexedDB storage quota and eviction policy (clearing site data wipes it). The page's **Reset storage** button deletes the `/data` and `/apps` stores and reloads.
- No device-side code changed: `AppManager` install/uninstall uses its normal relative path (`apps/…`), which resolves to the IDBFS mount because the working directory is `/`.
- `SharedPreferences` writes `prefs/<app>/config.json`. `/prefs` is not an IDBFS mount, so app settings are lost when the page reloads.

## Making changes — common scenarios

- **Change MPOS Python code / apps:** edit under `internal_filesystem/` as usual, then rebuild (`scripts/build_mpos.sh web`). Most of the FS is baked into `micropython.data` at link time, so a rebuild is required; there is no live file mount. Apps under `internal_filesystem/apps/` are copied into the persistent `/apps` store only on first boot, and files from `internal_filesystem_data/` are copied into `/data` only when missing (see "Persistence"). To see a new or changed bundled app, or a changed data file, press **Reset storage**. This also deletes installed apps and everything else in `/data` and `/apps`.
- **Change the HTML/JS shell:** edit `web/shell.html`, rebuild. `index.html` and `mpos.html` are regenerated from it.
- **Change build/link flags:** edit `scripts/web_port/web.py`, rebuild.
- **Change a patched submodule C file:** see "Updating a submodule patch".
- **Change a web-only C module (`_webnet`, `_webterm`, `_webio`):** edit it under `scripts/web_port/ext_mod/`, rebuild.
- **Add another web-only shim:** add the file under `scripts/web_port/staged_lib/`, and a `cp` line next to the others in the `web` branch of `scripts/build_mpos.sh`. If the shim goes in a new subdirectory of `lib/`, add a `mkdir -p` line before the `cp`.

## Testing in a browser

After a build, serve with `scripts/run_web.sh --no-build` and open the page. When iterating, note that `micropython.data` (the preloaded FS) is cached **separately** from the wasm, so when testing filesystem changes, hard-reload, or turn on Disable cache in the browser's developer tools. A query such as `?v=2` on the page URL does not help: the page still loads `micropython.js` and `micropython.data` by their plain names.

The page's **Log** button shows the boot output. MicroPython's stdout goes only to that log. Its stderr, where MicroPythonOS writes its warnings, also goes to the browser console.

A clean boot prints `sys.version=...`, `sys.implementation=...` and the `RAM: ... free` line, then `Passing execution over to mpos.main`, and renders the launcher. The following messages are **expected and non-fatal** in the browser:

- `__debug__ is set so code compiled with opt_level 0`
- `could not import/run freezefs_mount_builtin: [Errno 20] EEXIST`
- `could not mark this update as valid: no module named 'esp32'`
- `download error` / `fetch failed: TypeError: Failed to fetch` for the default app-store/OTA hosts (CORS-blocked cross-origin requests — see *Networking*).

A **fatal** problem would instead show `memory access out of bounds`, `function signature mismatch`, or `MicroPythonOS exiting`.

## Production build checklist

- Remove the debug flags `-sASSERTIONS=2 --profiling-funcs` from `web_ldflags` in `scripts/web_port/web.py`.
- Rebuild and serve the contents of `web/` from any static host.

## Deploying to GitHub Pages

CI deploys the MicroPythonOS site itself. [`.github/workflows/web-pages.yml`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/.github/workflows/web-pages.yml) builds the web target on every push and pull request to `main`. It boots the build headless and checks the `_webterm` REPL bridge (`tests/web_bridge_smoke.mjs`). A push to `main` is then deployed to the `gh-pages` branch. A pull request from a branch in the same repo gets a Netlify preview.

To deploy a fork by hand, use [`scripts/deploy_web_pages.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/deploy_web_pages.sh). It builds the web target, then force-pushes the contents of `web/` as a single commit to a `gh-pages` branch. Run it whenever you want to update your site.

The build leaves out the git-ignored files that desktop runs create in `internal_filesystem/`, such as the app settings in `prefs/`. Other untracked files there, such as apps you added, are included.

```bash
scripts/deploy_web_pages.sh                   # build, then deploy to remote `origin`, branch `gh-pages`
scripts/deploy_web_pages.sh --no-build        # deploy the existing web/ as-is
REMOTE=myfork scripts/deploy_web_pages.sh     # deploy to a different remote
BRANCH=pages scripts/deploy_web_pages.sh      # use a different branch
```

One-time setup on GitHub: **Settings → Pages → Build and deployment → Source: "Deploy from a branch", Branch: `gh-pages` / `(root)`**. The site is then served at `https://<owner>.github.io/<repo>/`.

The script adds a `.nojekyll` file so Pages serves `web/` as-is, without running Jekyll. The `gh-pages` branch holds only generated artifacts and is overwritten on every deploy.
