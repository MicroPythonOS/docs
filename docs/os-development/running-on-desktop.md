## Running on Linux or MacOS

1. Make sure you have the `local_filesystem/` folder

    Make a local clone that contains the [internal_filesystem](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem) with:

    <pre>
    ```
    git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
    cd MicroPythonOS/
    ```
    </pre>

2. Make sure you have the software

    Either you built your own [on MacOS](../os-development/macos.md) or [Linux](../os-development/linux.md) or you can download a pre-built executable binary (e.g., `MicroPythonOS_amd64_Linux_0.7.1.elf`, `MicroPythonOS_amd64_MacOS_0.7.1.bin`) from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases).

    If you downloaded a pre-built binary (for example to /Users/yourname/MicroPythonOS_amd64_MacOS_0.7.1.bin) then put it in the right location, with execution rights:

    <pre>
    ```
    mkdir -p lvgl_micropython/build
    cp /Users/yourname/MicroPythonOS_0.7.1.bin lvgl_micropython/build/lvgl_micropy_macOS
    ./scripts/run_desktop.sh
    ``` 
    </pre>

    On MacOS, if you get an error about a missing /opt/homebrew/opt/libffi/lib/libffi.8.dylib then fix that with: `brew install libffi`

## Development on Desktop

This offers a quick coding cycle (modify, deploy, test) so it's the recommending starting method.

When you run `./scripts/run_desktop.sh`, the OS runs **directly from `internal_filesystem/`**. This means:

- **All changes to Python files are immediately active** - no build or install needed
- **Instant testing** - edit a file, restart the app, see the changes
- **Fast iteration cycle** - the recommended way to develop and test

There's no need to run `./scripts/install.sh` when testing on desktop - that script is only for deploying everything from internal_filesystem/ to physical hardware.

### Modifying Files

You'll notice that whenever you change a file in `internal_filesystem/`, the changes are immediately visible on desktop when you reload the file or restart the app.

This results in a very quick coding cycle - no compilation or installation needed for Python code changes.

**Try it yourself:**

1. Edit `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py`
2. Run `./scripts/run_desktop.sh`
3. Open the About app
4. See your changes immediately!


## Development on Hardware

Once you've tested your changes on desktop and they work correctly, or you're doing things you can't test on desktop, then you can deploy to physical hardware.

This assumes your device was already flashed with MicroPythonOS or at least MicroPython, using something like https://install.MicroPythonOS.com/

The `install.sh` script copies files from `internal_filesystem/` to the device's storage partition over USB/serial:

```bash
./scripts/install.sh
```

On MacOS, you need this for the install.sh script: `brew install --cask serial`

**When you DO need to rebuild and flash the MicroPythonOS firmware:**

You only need to run `./scripts/build_mpos.sh` when:

- Modifying C extension modules (`c_mpos/`, `secp256k1-embedded-ecdh/`)
- Changing MicroPython core or LVGL bindings
- Testing the frozen filesystem for production releases
- Creating firmware for distribution

For **all Python code development**, just edit files in `internal_filesystem/` and run `./scripts/run_desktop.sh` (to run on desktop) or `./scripts/install.sh` to copy your files to a device.
