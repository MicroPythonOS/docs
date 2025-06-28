# Building for Desktop

MicroPythonOS can be built to run as an application on Linux desktops (fully supported) or MacOS (untested).

## Prerequisites

- Install dependencies (Linux):
```bash
sudo apt install libv4l-dev  # For webcam support
```
- See [lvgl-micropython](https://github.com/MicroPythonOS/lvgl-micropython) for additional requirements.
- Clone repositories as described in [Building for ESP32](esp32.md).

## Build Process

1. **Navigate to the Repository**:

    ```bash
    cd ~/MicroPythonOS/MicroPythonOS
    ```

2. **Build for Linux**:

    ```bash
    ./scripts/build_lvgl_micropython.sh unix dev
    ```

3. **Build for MacOS** (untested):

    ```bash
    ./scripts/build_lvgl_micropython.sh macOS dev
    ```

## Running on Desktop

1. Download a release (e.g., `MicroPythonOS_amd64_Linux_0.0.8`) or use your build.
2. Run the application:

    ```bash
    cd internal_filesystem/
    /path/to/MicroPythonOS_amd64_Linux_0.0.8 -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
    ```

3. Check `scripts/run_on_desktop.sh` for options like fullscreen or direct app launch.

## Notes

- Linux is fully supported; MacOS support is experimental.
- See [Supported Hardware](../getting-started/supported-hardware.md) for platform details.
