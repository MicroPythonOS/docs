# Build-time Patches

MicroPythonOS builds from pinned upstream sources (MicroPython, LVGL, ESP-IDF,
the `lvgl_micropython` binding) and applies a small set of patches at build
time instead of forking those projects. This page is the single reference for
where those patches live, how they are applied, and how to add or update one.

## Three families

| Family | Where the `.patch` files live | What they patch | Applied by |
|---|---|---|---|
| **lvgl_micropython patches** | root of the [`lvgl_micropython`](https://github.com/MicroPythonOS/lvgl_micropython) repo (`integration` branch) | files under its `lib/micropython`, `lib/lvgl`, or the repo itself (e.g. `builder/`) | `scripts/build_mpos.sh` (`apply_patch`) |
| **MicroPythonOS patches** | `patches/` in the [`MicroPythonOS`](https://github.com/MicroPythonOS/MicroPythonOS) repo | files under `lvgl_micropython`'s `lib/esp-idf` and `lib/micropython`, and in the `espressif__tinyusb` component that ESP32-S3 and ESP32-P4 builds download | `scripts/build_mpos.sh` (`apply_patch`) |
| **Web-port patches** | `scripts/web_port/` in the [`MicroPythonOS`](https://github.com/MicroPythonOS/MicroPythonOS) repo | `lvgl_micropython` files needed only by the Emscripten target | `scripts/build_mpos.sh web`, see [Web Port Developer Info](../web-port/developer.md#submodule-patches-applied-automatically) |

All three are unified diffs, applied with `patch -p1 --forward` by the same
helper, and idempotent on rebuilds. The sections below say where they differ.

## How `apply_patch` behaves

`scripts/build_mpos.sh` applies every patch through one helper:

```bash
apply_patch <directory to patch in> <patch file>
```

1. If the patch applies forward, it is applied.
2. Else if it applies in reverse — meaning it is already present — it is
   skipped with `Patch ... already applied, skipping.`
3. Otherwise the build **fails** (`FATAL: patch ... does not apply`). A patch
   that silently stops applying once shipped a broken build, so this is on
   purpose: fix or regenerate the patch instead of ignoring it.

Patches that were added after some pinned `lvgl_micropython` commits are
**existence-guarded** (`if [ -f "$patch" ]`) so MicroPythonOS can still build
against older submodule pins; copy that pattern for new `lvgl_micropython`
patches.

## Current lvgl_micropython patches

Read `scripts/build_mpos.sh` for the authoritative list and the target of
each; at the time of writing:

- Applied for **every** target, in `lvgl_micropython/lib/micropython` or
  `lvgl_micropython/lib/lvgl`: `esp32_uart_repl_runtime.patch`,
  `mpremote_no_auto_soft_reset.patch`, `lib_lvgl_lv_bmp.c.patch`,
  `lib_lvgl_src_libs_tjpgd_fix_scaling.patch`, `imgfont_set_range.patch`
  (guarded).
- **ESP32** builds only: `esp32_inisetup_warn_and_format.patch`,
  `esp32_inisetup_readsize_progsize.patch`,
  `network_wlan_country_japan.patch`, `network_wlan_config_country.patch`,
  `esp32_user_c_modules_abspath.patch` (guarded; patches `builder/esp32.py`
  in the `lvgl_micropython` repo itself, applied in that directory), and
  `esp32_riscv_frozen_native_align.patch` (guarded).
- **Desktop (unix/macOS) and web** builds: `unix_autoimport_main.patch` and
  `unix_native_decorator_fallback.patch` (guarded).
- **Desktop (unix/macOS)** builds only: `unix_sdl_release_2_32_8.patch`
  (guarded; patches `builder/unix.py` in the `lvgl_micropython` repo itself,
  applied in that directory).
- **macOS** builds only: `macos_brew_prefix.patch` (guarded; patches
  `builder/macOS.py` in the `lvgl_micropython` repo itself, applied in that
  directory after the `sdl2-compat` edit described under
  [Adding a patch](#adding-a-patch)).

## Current MicroPythonOS patches

These are in `patches/` and are applied for every target. Only ESP32 builds
compile the files they change, but desktop and web builds apply them too, so
they also need the `lib/esp-idf` submodule checked out.

- In `lvgl_micropython/lib/esp-idf`: `usb_ext_port_settle.patch`,
  `usb_ext_port_retries.patch`, `usb_enum_no_abort.patch`.
- In `lvgl_micropython/lib/micropython`: `usb_phy_deinit.patch`.
- In the `espressif__tinyusb` component under
  `lvgl_micropython/lib/micropython/ports/esp32/managed_components/`:
  `tinyusb_isr_double_free.patch`. The ESP-IDF component manager downloads
  this component during ESP32-S3 and ESP32-P4 builds, so it isn't there on a
  fresh checkout. If the component is there before the build,
  `build_mpos.sh` applies the patch then. If the build downloads the
  component, or replaces a patched copy, `build_mpos.sh` applies the patch
  after the build and builds again. Other targets don't use the component,
  so the patch doesn't change their firmware.

## Current web-port patches

These are in `scripts/web_port/` and are applied only for the `web` target.
Every other build undoes them by checking out the four files they change.

- In `lvgl_micropython`: `make.py.patch`, `lcd_bus_micropython.mk.patch`
  (patches `ext_mod/lcd_bus/micropython.mk`) and `sdl_bus.h.patch` (patches
  `ext_mod/lcd_bus/sdl_bus/sdl_bus.h`).
- In `lvgl_micropython/lib/micropython`: `gccollect.c.patch` (patches
  `ports/unix/gccollect.c`).

## Adding a patch

1. Make the change directly in the checked-out submodule tree and verify the
   build.
2. Generate the diff **from the directory the patch will be applied in**, so
   the paths are right for `patch -p1`:

        # a MicroPython change:
        cd lvgl_micropython/lib/micropython
        git diff -- ports/unix/main.c > ../../my_change.patch

        # a change to lvgl_micropython itself (e.g. builder/):
        cd lvgl_micropython
        git diff -- builder/unix.py > my_change.patch

        # a MicroPythonOS patch to ESP-IDF:
        cd lvgl_micropython/lib/esp-idf
        git diff -- components/usb/hub.c > ../../../patches/my_change.patch

    After a build, the file also holds the changes made by the other
    patches and the in-place edits described below, and `git diff` shows
    those too. Keep only your own change in the patch.

3. Commit the `.patch` file to `lvgl_micropython` (branch off `integration`,
   see its [CONTRIBUTING.md](https://github.com/MicroPythonOS/lvgl_micropython/blob/integration/CONTRIBUTING.md))
   and open a **companion PR** in MicroPythonOS adding an `apply_patch` call
   to `scripts/build_mpos.sh` in the right target section, existence-guarded.
   Put the call after every other `apply_patch` call that changes the same
   file. The guard skips the patch until MicroPythonOS's `lvgl_micropython`
   submodule is bumped to a commit that has the file.
4. Revert your direct edit in the submodule and rebuild. The build log must
   show your patch being applied (`patching file ...`). Build once more: it
   must now say `Patch ... already applied, skipping.`

MicroPythonOS patches and web-port patches live in the MicroPythonOS repo
(`patches/` and `scripts/web_port/`), so the patch file and its `apply_patch`
call go in one PR, with no existence guard on the patch file. A new
web-port patch to another file also needs that file added to
`reset_web_port_changes` in `build_mpos.sh`, or other builds keep the change.
Git doesn't track the downloaded tinyusb component, so make a patch for it
with `diff -u` against an unpatched copy, with `a/` and `b/` in front of the
paths. Put its `apply_patch` call inside the same check for the component's
files that the current tinyusb call uses. Without it, every build that hasn't
downloaded the component stops with `FATAL`. Also add the call right after
the second `apply_patch` call for `tinyusb_isr_double_free.patch`, the one
that runs after the ESP32 build. Without that, the first ESP32-S3 or ESP32-P4
build after a fresh checkout is built without your patch. The web-port update
commands are
in the
[Web Port Developer Info](../web-port/developer.md#updating-a-submodule-patch).

`build_mpos.sh` and lvgl_micropython's `make.py` also edit some files in
place, without a `.patch` file, and most of these edits stay after the build.
For example, for the macOS target `build_mpos.sh` changes `'sdl2'` to
`'sdl2-compat'` in the `brew info` call in `builder/macOS.py`. On desktop
builds `make.py` edits `ports/unix/main.c` after the patches are applied, so
a fresh checkout doesn't have those edits when the patches go in. If your
patch changes such a file, keep the edited lines out of the patch and out of
its context (for example with `git diff -U1`), so it applies whether or not
the edit has been made.

## Updating a patch

Regenerate it the same way (edit the file in the submodule, `git diff` from
the apply directory, overwrite the `.patch`), then rebuild from a clean
submodule to prove it applies. If a submodule bump makes a patch stop
applying, the build's `FATAL` message tells you which one.

`tinyusb_isr_double_free.patch` is applied to the downloaded component
instead. After you change it, delete
`lvgl_micropython/lib/micropython/ports/esp32/managed_components/espressif__tinyusb`
and build an ESP32-S3 or ESP32-P4 target. The build downloads the component
again, applies the patch and builds a second time. A tree that still has the
old version applied needs the same delete, or the build can stop with
`FATAL`.
