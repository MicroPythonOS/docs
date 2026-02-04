The easiest way to install on the ESP32 is using the [webinstaller](https://install.MicroPythonOS.com), of course.

But if you need to install a version that's not available there, or you built your own, then you can manually install it on an ESP32 device.

1. **Get the firmware**

    - Download a release binary (e.g., `MicroPythonOS_esp32_0.5.0.bin`)
    - Or build your own [on MacOS](macos.md) or [Linux](linux.md)

2. **Put the ESP32 in Bootloader Mode**

    If you're already in MicroPythonOS: go to Settings - Restart to Bootloader - Bootloader - Save.
    
    Otherwise, physically keep the "BOOT" (sometimes labeled "START") button pressed while powering up the board.
    This is explained in more detail at [the webinstaller](https://install.micropythonos.com/)
    
3. **Flash the firmware**

    ```
    ~/.espressif/python_env/idf5.2_py3.9_env/bin/python -m esptool --chip esp32s3 write_flash 0 firmware_file.bin
    ```

    Add the `--erase-all` option if you want to erase the entire flash memory, so that no old files or apps will remain.
    
    There's also a convenient `./scripts/flash_over_usb.sh` script that will attempt to flash the latest firmware that you compiled yourself.

4. **Access the MicroPython REPL shell**

    After reset, the REPL shell should be available on the serial line.
    
    Any serial client will do, but it's convenient to use the `mpremote.py` tool that's shipped with lvgl_micropython:
    
    ```
    lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py
    ```

5. **Populate the filesystem** (only for development)

    In development, you probably want to override the "frozen" libraries and apps that are compiled in, and replace them with source files, which you can edit.

    There's a convenient script that will do this for you.
    
    Usage:

    <pre>
    ```
    ./scripts/install.sh
    ```
    </pre>

    <pre>
    ```
    ./scripts/install.sh com.micropythonos.about # to install one single app
    ```
    </pre>

    On MacOS, the install.sh script needs: `brew install --cask serial`

    If you need to frequently update a small number of files, you can also update them manually, for example:

    <pre>
    ```
    mpremote.py cp internal_filesystem/lib/mpos/device_info.py :/lib/mpos
    ```
    </pre>

## Notes

- Ensure your ESP32 is compatible (see [Supported Hardware](../getting-started/supported-hardware.md)). If it's not, then you might need the [Porting Guide](../os-development/porting-guide.md).
