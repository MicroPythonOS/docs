# Build-time Patches

MicroPythonOS builds from pinned upstream sources (MicroPython, LVGL, the
`lvgl_micropython` binding) and applies a small set of patches at build time
instead of forking those projects. This page is the single reference for
where those patches live, how they are applied, and how to add or update one.

## Two families

| Family | Where the `.patch` files live | What they patch | Applied by |
|---|---|---|---|
| **lvgl_micropython patches** | root of the [`lvgl_micropython`](https://github.com/MicroPythonOS/lvgl_micropython) repo (`integration` branch) | files under its `lib/micropython`, `lib/lvgl`, or the repo itself (e.g. `builder/`) | `scripts/build_mpos.sh` (`apply_patch`) |
| **Web-port patches** | `scripts/web_port/` in the [`MicroPythonOS`](https://github.com/MicroPythonOS/MicroPythonOS) repo | `lvgl_micropython` files needed only by the Emscripten target | `scripts/build_mpos.sh web`, see [Web Port Developer Info](../web-port/developer.md#submodule-patches-applied-automatically) |

Everything else about the two families is the same: a unified diff created
with `git diff`, applied with `patch -p1 --forward`, idempotent on rebuilds.

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
against older submodule pins; copy that pattern for new patches.

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
  `network_wlan_country_japan.patch`, `network_wlan_config_country.patch`.
- **Desktop (unix/macOS)** builds only: `unix_autoimport_main.patch`,
  `unix_native_decorator_fallback.patch` (guarded), and
  `unix_sdl_release_2_32_8.patch` (guarded; patches `builder/unix.py` in the
  `lvgl_micropython` repo itself, applied in that directory).

## Adding a patch

1. Make the change directly in the checked-out submodule tree and verify the
   build.
2. Generate the diff **from the directory the patch will be applied in**, so
   the paths are right for `patch -p1`:

    ```bash
    # a MicroPython change:
    cd lvgl_micropython/lib/micropython
    git diff -- ports/unix/main.c > ../../my_change.patch

    # a change to lvgl_micropython itself (e.g. builder/):
    cd lvgl_micropython
    git diff -- builder/unix.py > my_change.patch
    ```

3. Commit the `.patch` file to `lvgl_micropython` (branch off `integration`,
   see its [CONTRIBUTING.md](https://github.com/MicroPythonOS/lvgl_micropython/blob/integration/CONTRIBUTING.md))
   and open a **companion PR** in MicroPythonOS adding an `apply_patch` call
   to `scripts/build_mpos.sh` in the right target section, existence-guarded.
4. Revert your direct edit in the submodule and rebuild: the build must
   re-apply the patch cleanly (no `.rej` files).

Web-port patches follow the same steps but live in `scripts/web_port/` and
need no companion PR; the update commands are in the
[Web Port Developer Info](../web-port/developer.md#updating-a-submodule-patch).

## Updating a patch

Regenerate it the same way (edit the file in the submodule, `git diff` from
the apply directory, overwrite the `.patch`), then rebuild from a clean
submodule to prove it applies. If a submodule bump makes a patch stop
applying, the build's `FATAL` message tells you which one.
