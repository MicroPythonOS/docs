# Building for ESP32

Build MicroPythonOS for ESP32 microcontrollers, such as [supported hardware](../getting-started/supported-hardware.md).

## Prerequisites

Clone the required repositories:

```
mkdir ~/MicroPythonOS
cd ~/MicroPythonOS

git clone https://github.com/MicroPythonOS/MicroPythonOS.git
git clone https://github.com/MicroPythonOS/freezeFS
git clone https://github.com/MicroPythonOS/secp256k1-embedded-ecdh
git clone https://github.com/MicroPythonOS/lvgl_micropython

echo "  espressif/esp32-camera:
    git: https://github.com/MicroPythonOS/esp32-camera" >> lvgl_micropython/lib/micropython/ports/esp32/main/idf_component.yml

echo 'include("$(MPY_DIR)/extmod/asyncio") # This is needed to have asyncio, which is used by aiohttp, which has easy to use websockets' >> lvgl_micropython/lib/micropython/ports/unix/variants/manifest.py

# Unix builds need this as they don't handle C_
ln -s ../../secp256k1-embedded-ecdh lvgl_micropython/ext_mod/secp256k1-embedded-ecdh
ln -s ../../MicroPythonOS/c_mpos lvgl_micropython/ext_mod/c_mpos

git clone https://github.com/cnadler86/micropython-camera-API
pushd micropython-camera-API/
git checkout v0.4.0
echo 'include("~/MicroPythonOS/lvgl_micropython/build/manifest.py")' >> src/manifest.py
popd
```

## Build Process

1. **Navigate to the main repository**:

    ```
    cd ~/MicroPythonOS/MicroPythonOS
    ```

2. **Build for Production** (includes preinstalled files):

    ```
    ./scripts/build_lvgl_micropython.sh esp32 prod waveshare-esp32-s3-touch-lcd-2
    ```

    or, depending on the device you're building for:

    ```
    ./scripts/build_lvgl_micropython.sh esp32 prod fri3d-2024
    ```

3. **Build for Development** (no preinstalled/frozen files):

    ```
    ./scripts/build_lvgl_micropython.sh esp32 dev fri3d-2024
    ```

    or, depending on the device you're building for:

    ```
    ./scripts/build_lvgl_micropython.sh esp32 dev waveshare-esp32-s3-touch-lcd-2
    ```

## Flashing to ESP32

1. Put your ESP32 in bootloader mode (long-press the BOOT button if running MicroPythonOS).

2. Flash the firmware:

    ```
    ./scripts/flash_over_usb.sh
    ```

3. For a development build, fill the filesystem with files manually:

    ```
    ./scripts/install.sh
    ```

## Notes

- A "dev" build without frozen files is quite a bit slower when starting apps because all the libraries need to be compiled at runtime.
- Ensure your ESP32 is compatible (see [Supported Hardware](../getting-started/supported-hardware.md)).
- Refer to [Release Checklist](release-checklist.md) for creating a production release.
