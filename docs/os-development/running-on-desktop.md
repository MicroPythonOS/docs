## Running on desktop

MicroPythonOS runs on Linux, macOS and Raspberry Pi desktops. The desktop build is a single executable that contains the whole OS, so you usually do not need to compile anything.

Pick the level that matches what you want to do.

---

### Level 1: Just want to run it?

Grab a pre-built binary and launch it.

1. Download the binary for your platform from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases):
   - **Linux, Raspberry Pi, WSL2 on Windows:** `lvgl_micropy_unix`
   - **macOS (Apple Silicon or Intel):** `lvgl_micropy_macOS`
2. Make it executable:

```bash
chmod +x lvgl_micropy_unix
```

3. Run it. From a terminal:

```bash
./lvgl_micropy_unix
```

Or double-click it in your file manager. You may want to rename it to `MicroPythonOS` first.

---

### Level 2: Want to develop an app but use a prebuilt OS?

You can use a downloaded OS binary with a full source checkout so you can edit apps and see changes immediately.

1. Clone the repository:

```bash
git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
cd MicroPythonOS
```

2. Download the binary for your platform from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases). Rename it and put it where the runner script expects it:
   - Linux / Raspberry Pi / WSL2 → `lvgl_micropython/build/lvgl_micropy_unix`
   - macOS → `lvgl_micropython/build/lvgl_micropy_macOS`

```bash
mkdir -p lvgl_micropython/build
cp /path/to/downloaded/binary lvgl_micropython/build/lvgl_micropy_unix
chmod +x lvgl_micropython/build/lvgl_micropy_unix
```

3. Run it with the helper script:

```bash
./scripts/run_desktop.sh
```

`scripts/run_desktop.sh` launches the OS using the Python files in `internal_filesystem/` directly, so any edit you make there appears the next time you restart the app. No rebuild is needed.

**Try it:**

1. Edit `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py`
2. Run `./scripts/run_desktop.sh`
3. Open the About app
4. See your change immediately

Once your app works on desktop, deploy it to a physical device with [Installing on ESP32](installing-on-esp32.md).

---

### Level 3: Want to build the OS yourself?

If you need to change the OS itself, add C extensions, modify MicroPython/LVGL bindings, or run the very latest code, build from source. The binary will already land in `lvgl_micropython/build/lvgl_micropy_XXX` where `XXX` is `unix` or `macOS`.

```bash
./scripts/build_mpos.sh unix     # Linux, Raspberry Pi, WSL2
./scripts/build_mpos.sh macOS    # macOS
```

See [Compiling](compiling.md) for the full instructions, including cloning and dependencies.

---

## Known issues and fixes

### Linux: "No such file or directory" when running the binary

Make sure it is executable:

```bash
chmod +x lvgl_micropy_unix
```

### macOS: missing `libffi.8.dylib`

Install libffi:

```bash
brew install libffi
```

### macOS: "cannot be opened because the developer cannot be verified"

The prebuilt binary is not signed. Open **System Settings → Privacy & Security** and click **Allow Anyway** next to the blocked item, then run it again.

![Allow Anyway on MacOS](/os-development/macos-allow-anyway.png)

### Windows

Native Windows builds are not supported. [Users report](https://github.com/MicroPythonOS/MicroPythonOS/issues/31) that the Linux desktop binary works under WSL2 on Windows 11. Alternatively, you can try the [web port](../web-port/using.md), which runs a desktop build in the browser.

---

## Deploying to hardware

Once your app works on desktop, install it on a supported ESP32 device.

{!os-development/installing-on-esp32.md!}

