The easiest way to install on the ESP32 is using the [webinstaller](https://install.MicroPythonOS.com), of course.

But if you need to install a version that's not available there, or you built your own, then you can manually install it on an ESP32 device.

1. **Get the firmware**

    - Download a release binary (e.g., `MicroPythonOS_fri3d-2024_prod_0.2.1.bin`, `MicroPythonOS_waveshare-esp32-s3-touch-lcd-2_prod_0.2.1.bin`, etc.)
    - Or build your own [on MacOS](macos.md) or [Linux](linux.md)

2. **Put the ESP32 in Bootloader Mode**

    If you're already in MicroPythonOS: go to Settings - Restart to Bootloader - Bootloader - Save.
    
    Otherwise, physically keep the "BOOT" (sometimes labeled "START") button pressed while briefly pressing the "RESET" button.
    
3. **Flash the firmware**

    ```
    ~/.espressif/python_env/idf5.2_py3.9_env/bin/python -m esptool --chip esp32s3 0x0 firmware_file.bin
    ```

    Add --erase-all if you want to erase the entire flash memory, so that no old files or apps will remain.
    
    There's also a convenient `./scripts/flash_over_usb.sh` script that will attempt to flash the latest firmware that you compiled yourself.

4. **Access the MicroPython REPL shell**

    After reset, the REPL shell should be available on the serial line.
    
    Any serial client will do, but it's convenient to use the `mpremote.py` tool that's shipped with lvgl_micropython:
    
    ```
    lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py
    ```

5. **Populate the filesystem** (only for "dev" builds)

    The "dev" builds come without a filesystem so you probably want to copy:

    - the whole internal_filesystem/ folder, including main.py
    - the appropriate device-specific internal_filesystem/boot*.py file to /boot.py on the device
    
    There's a convenient script that will do this for you.
    
    Usage:

    <pre>
    ```
    ./scripts/install.sh <target device>
    ```
    </pre>
    
    **Target devices**: waveshare-esp32-s3-touch-lcd-2 or fri3d-2024

    Examples:

    <pre>
    ```
    ./scripts/install.sh fri3d-2024
    ./scripts/install.sh waveshare-esp32-s3-touch-lcd-2
    ```
    </pre>


## Notes

- A "dev" build without frozen files is quite a bit slower when starting apps because all the libraries need to be compiled at runtime.
- Ensure your ESP32 is compatible (see [Supported Hardware](../getting-started/supported-hardware.md)). If it's not, then you might need the [Porting Guide](../os-development/porting-guide.md).
