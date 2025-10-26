## Compile the code

1. **Make sure you're in the main repository**:

    ```
    cd MicroPythonOS/
    ```

2. **Start the Compilation**

    Usage:

    <pre>
    ```
    ./scripts/build_mpos.sh <target system> <build type (prod or dev)> [optional target device]
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
    ./scripts/build_mpos.sh esp32 prod fri3d-2024
    ./scripts/build_mpos.sh esp32 dev waveshare-esp32-s3-touch-lcd-2
    ./scripts/build_mpos.sh esp32 unix dev
    ./scripts/build_mpos.sh esp32 macOS dev
    ```
    </pre>

    The resulting build file will be in `lvgl_micropython/build/`, for example:

    - `lvgl_micropython/build/lvgl_micropy_unix`
    - `lvgl_micropython/build/lvgl_micropy_macOS`
    - `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin`

