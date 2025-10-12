# Building for ESP32

Build MicroPythonOS for ESP32 microcontrollers, such as [supported hardware](../getting-started/supported-hardware.md).

## Prerequisites

Clone the required repositories:

```bash
mkdir ~/MicroPythonOS
cd ~/MicroPythonOS
git clone https://github.com/MicroPythonOS/MicroPythonOS.git
git clone https://github.com/MicroPythonOS/freezeFS
git clone https://github.com/MicroPythonOS/secp256k1-embedded-ecdh
git clone https://github.com/MicroPythonOS/lvgl_micropython
cd lvgl_micropython/
echo "  espressif/esp32-camera:
    git: https://github.com/cnadler86/esp32-camera.git" > lib/micropython/ports/esp32/main/idf_component.yml

git clone https://github.com/cnadler86/micropython-camera-API
cd micropython-camera-API/
git checkout v0.4.0
echo 'include("~/MicroPythonOS/lvgl_micropython/build/manifest.py")' >> src/manifest.py

```

## Build Process

1. **Navigate to the Repository**:

    ```
    cd ~/MicroPythonOS/MicroPythonOS
    ```

2. **Build for Production** (includes preinstalled files):

    ```bash
    ./scripts/build_lvgl_micropython.sh esp32 prod waveshare-esp32-s3-touch-lcd-2
    ```

    or, depending on the device you're building for:

    ```bash
    ./scripts/build_lvgl_micropython.sh esp32 prod fri3d-2024
    ```

3. **Build for Development** (no preinstalled files):

    ```bash
    ./scripts/build_lvgl_micropython.sh esp32 dev fri3d-2024
    # or
    ./scripts/build_lvgl_micropython.sh esp32 dev waveshare-esp32-s3-touch-lcd-2
    ```

## Flashing to ESP32

1. Put your ESP32 in bootloader mode (long-press the BOOT button if running MicroPythonOS).

2. Flash the firmware:

    ```bash
    ./scripts/flash_over_usb.sh
    ```

3. For a development build, fill the filesystem manually:

    ```bash
    ./scripts/install.sh
    ```

## Notes

- Ensure your ESP32 is compatible (see [Supported Hardware](../getting-started/supported-hardware.md)).
- Refer to [Release Checklist](release-checklist.md) for creating a production release.
