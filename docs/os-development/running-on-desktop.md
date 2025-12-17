## Running on Linux or MacOS

1. Make sure you have the software

    Either you built your own [on MacOS](../os-development/macos.md) or [Linux](../os-development/linux.md) or you can download a pre-built executable binary (e.g., `MicroPythonOS_amd64_Linux`, `MicroPythonOS_amd64_MacOS`) from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases).

    Give it executable permissions:

    ```
    chmod +x /path/to/MicroPythonOS_executable_binary
    ``` 

2. Make sure you have the `local_filesystem/` folder

    You probably already have a local clone that contains the [internal_filesystem](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem).

    If not, then clone it now:

    <pre>
    ```
    git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
    cd MicroPythonOS/
    ```
    </pre>

3. Start it from the local_filesystem/ folder:

    <pre>
    ```
    cd internal_filesystem/ # make sure you're in the right place to find the filesystem
    /path/to/MicroPythonOS_executable_binary -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
    ```
    </pre>

    There's also a convenient `./scripts/run_desktop.sh` script that will attempt to start the latest build that you compiled yourself.

### Development Workflow: Desktop vs Hardware

**IMPORTANT**: Understanding the difference between desktop testing and hardware deployment is critical for efficient development.

#### Desktop Development (Recommended for Most Development)

When you run `./scripts/run_desktop.sh`, the OS runs **directly from `internal_filesystem/`**. This means:

✅ **All changes to Python files are immediately active** - no build or install needed
✅ **Instant testing** - edit a file, restart the app, see the changes
✅ **Fast iteration cycle** - the recommended way to develop and test

**DO NOT run `./scripts/install.sh` when testing on desktop!** That script is only for deploying to physical hardware.

**Example workflow:**
```bash
# 1. Edit a file
nano internal_filesystem/builtin/apps/com.micropythonos.settings/assets/settings.py

# 2. Run on desktop - changes are immediately active!
./scripts/run_desktop.sh

# That's it! Your changes are live.
```

#### Hardware Deployment (Only After Desktop Testing)

Once you've tested your changes on desktop and they work correctly, you can deploy to physical hardware:

```bash
# Deploy to connected ESP32 device
./scripts/install.sh waveshare-esp32-s3-touch-lcd-2
```

The `install.sh` script copies files from `internal_filesystem/` to the device's storage partition over USB/serial.

### Modifying Files

You'll notice that whenever you change a file in `internal_filesystem/`, the changes are immediately visible on desktop when you reload the file or restart the app.

This results in a very quick coding cycle - no compilation or installation needed for Python code changes.

**Try it yourself:**

1. Edit `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py`
2. Run `./scripts/run_desktop.sh`
3. Open the About app
4. See your changes immediately!

**When you DO need to rebuild:**

You only need to run `./scripts/build_mpos.sh` when:

- Modifying C extension modules (`c_mpos/`, `secp256k1-embedded-ecdh/`)
- Changing MicroPython core or LVGL bindings
- Testing the frozen filesystem for production releases
- Creating firmware for distribution

For **all Python code development**, just edit files in `internal_filesystem/` and run `./scripts/run_desktop.sh`.
