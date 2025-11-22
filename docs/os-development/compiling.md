## Compile the code

1. **Make sure you're in the main repository**:

    ```
    cd MicroPythonOS/
    ```

2. **Start the Compilation**

    Usage:

    <pre>
    ```
    ./scripts/build_mpos.sh <target system>
    ```
    </pre>

    **Target systems**: `esp32`, `unix` (= Linux) and `macOS`
    
    **Examples**:

    <pre>
    ```
    ./scripts/build_mpos.sh esp32
    ./scripts/build_mpos.sh unix
    ./scripts/build_mpos.sh macOS
    ```
    </pre>

    The resulting build file will be in `lvgl_micropython/build/`, for example:

    - `lvgl_micropython/build/lvgl_micropy_unix`
    - `lvgl_micropython/build/lvgl_micropy_macOS`
    - `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin`

