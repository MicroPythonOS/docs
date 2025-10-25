## Compile the code

1. **Make sure you're in the main repository**:

    ```
    cd MicroPythonOS
    ```

2. **Start the Compilation**

    Usage:

    <pre>
    ```
    ./scripts/build_lvgl_micropython.sh <target system> <build type (prod or dev)> [optional target device]
    ```
    </pre>

    **Target systems**: esp32, unix (= Linux) and macOS
    
    **Build types**:

    - A "prod" build includes the complete filesystem that's "frozen" into the build, so it's fast and all ready to go but the files in /lib and /builtin will be read-only.
    - A "dev" build comes without a filesystem, so it's perfect for power users that want to work on MicroPythonOS internals. There's a simple script that will copy all the necessary files over later, and these will be writeable.

    _Note_: for unix and macOS systems, only "dev" has been tested. The "prod" builds might have issues but should be made to work soon.

    **Target devices**: waveshare-esp32-s3-touch-lcd-2 or fri3d-2024
    
    **Examples**:

    <pre>
    ```
    ./scripts/build_lvgl_micropython.sh esp32 prod fri3d-2024
    ./scripts/build_lvgl_micropython.sh esp32 dev waveshare-esp32-s3-touch-lcd-2
    ./scripts/build_lvgl_micropython.sh esp32 unix dev
    ./scripts/build_lvgl_micropython.sh esp32 macOS dev
    ```
    </pre>

    The resulting build file will be in `lvgl_micropython/build/`, for example:

    - lvgl_micropython/build/lvgl_micropy_unix
    - lvgl_micropython/build/lvgl_micropy_macOS
    - lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin

## Running on Linux or MacOS

1. Download a release binary (e.g., `MicroPythonOS_amd64_Linux`, `MicroPythonOS_amd64_MacOS`) or build your own [on MacOS](macos.md) or [Linux](linux.md).
2. Run the application:

    <pre>
    ```
    cd internal_filesystem/ # make sure you're in the right place to find the filesystem
    /path/to/release_binary -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
    ```
    </pre>

    There's also a convenient `./scripts/run_desktop.sh` script that will attempt to start the latest build that you compiled yourself.

### Modifying files

You'll notice that, whenever you change a file on your local system, the changes are immediately visible whenever you reload the file.

This results in a very quick coding cycle.

Give this a try by editing `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py` and then restarting the "About" app. Powerful stuff!
