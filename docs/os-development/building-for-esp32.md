# Building for ESP32

Build MicroPythonOS for ESP32 microcontrollers, such as [supported hardware](../getting-started/supported-hardware.md).

## Get the prerequisites

Clone the repositories:

```
git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
```

That will take a while, because it recursively clones MicroPython, LVGL, ESP-IDF and all their dependencies.

While that's going on, make sure you have everything installed to compile code:

```
sudo apt update
sudo apt-get install -y build-essential libffi-dev pkg-config cmake ninja-build gnome-desktop-testing libasound2-dev libpulse-dev libaudio-dev libjack-dev libsndio-dev libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxfixes-dev libxi-dev libxss-dev libxkbcommon-dev libdrm-dev libgbm-dev libgl1-mesa-dev libgles2-mesa-dev libegl1-mesa-dev libdbus-1-dev libibus-1.0-dev libudev-dev fcitx-libs-dev libpipewire-0.3-dev libwayland-dev libdecor-0-dev libv4l-dev
```

## Compile the code

1. **Navigate to the main repository**:

    ```
    cd ~/MicroPythonOS
    ```

2. **Start the Compilation**

    ```
    ./scripts/build_lvgl_micropython.sh esp32 dev fri3d-2024
    ```

    or, depending on the device you're building for:

    ```
    ./scripts/build_lvgl_micropython.sh esp32 dev waveshare-esp32-s3-touch-lcd-2
    ```

## Install it on an ESP32

1. Put your ESP32 in bootloader

If you're already in MicroPythonOS: go to Settings - Restart to Bootloader - Bootloader - Save.

Otherwise, physically keep the "BOOT" (sometimes labeled "START") button pressed while briefly pressing the "RESET" button.

2. Flash the firmware:

    ```
    ./scripts/flash_over_usb.sh
    ```

    After reset, you should find a MicroPython REPL shell on the serial line.

3. Run this script to copy the files to the device manually:

    ```
    ./scripts/install.sh
    ```

## Notes

- A "dev" build without frozen files is quite a bit slower when starting apps because all the libraries need to be compiled at runtime.
- Ensure your ESP32 is compatible (see [Supported Hardware](../getting-started/supported-hardware.md)). If it's not, then you might need the [Porting Guide](../os-development/porting-guide.md).
- Refer to [Release Checklist](release-checklist.md) for creating a production release.
