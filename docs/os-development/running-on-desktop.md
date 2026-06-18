## Running on desktop

Desktop builds of MicroPythonOS can run without a local source checkout. The `internal_filesystem/` is frozen into the binary at build time, so a single pre-built executable is enough to try the OS.

### Download a pre-built binary

1. Go to the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases).
2. Download the binary for your platform:
   - Linux / WSL2 on Windows: `lvgl_micropy_unix`
   - macOS (Apple Silicon or Intel): `lvgl_micropy_macOS`
3. Make it executable:

```
chmod +x lvgl_micropy_unix
```

4. Place it where `scripts/run_desktop.sh` expects it. The script looks for:
   - `lvgl_micropython/build/lvgl_micropy_unix` on Linux
   - `lvgl_micropython/build/lvgl_micropy_macOS` on macOS

You can create that folder and copy the binary there:

```
mkdir -p lvgl_micropython/build
cp /path/to/downloaded/lvgl_micropy_unix lvgl_micropython/build/lvgl_micropy_unix
```

5. Run it:

```
./scripts/run_desktop.sh
```

### Build from source

If you want to modify the OS itself or run the very latest code, you can [build it from source](compiling.md). The built binary will already be in `lvgl_micropython/build/lvgl_micropy_XXX` where `XXX` is `unix` or `macOS`.

### Notes on MacOS

If you get an error about a missing `/opt/homebrew/opt/libffi/lib/libffi.8.dylib` then fix that with: `brew install libffi`

If you get an error about the code being unsigned, then allow it like this:

![Allow Anyway on MacOS](/os-development/macos-allow-anyway.png)


## Making Changes on Desktop

If you do have a source checkout, you can still run the OS directly from `internal_filesystem/`. When you run `./scripts/run_desktop.sh`, the OS runs the MicroPythonOS scripts **directly from `internal_filesystem/`**. This means:

- **All changes to Python files are immediately active** - no build or install needed
- **Instant testing** - edit a file, restart the app, see the changes
- **Fast iteration cycle** - the recommended way to develop and test

**Try it yourself:**

1. Edit `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py`
2. Run `./scripts/run_desktop.sh`
3. Open the About app
4. See your changes immediately!


## Making Changes on ESP32

Once you've tested your changes on desktop and they work correctly, or you're doing things you can't test on desktop, then you can deploy to physical hardware.

{!os-development/installing-on-esp32.md!}

