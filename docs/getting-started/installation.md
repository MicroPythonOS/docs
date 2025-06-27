# Installation

MicroPythonOS can be installed on supported microcontrollers (e.g., ESP32) or desktop systems (Linux, potentially MacOS). For detailed instructions, visit [install.micropythonos.com](https://install.micropythonos.com).

## Installing on ESP32

1. **Prepare the Environment**:
```bash
mkdir ~/MicroPythonOS
cd ~/MicroPythonOS
git clone https://github.com/MicroPythonOS/MicroPythonOS.git
git clone https://github.com/MicroPythonOS/freezeFS
git clone https://github.com/cnadler86/micropython-camera-API
echo 'include("~/MicroPythonOS/lvgl_micropython/build/manifest.py")' >> micropython-camera-API/src/manifest.py
git clone https://github.com/MicroPythonOS/lvgl_micropython
git clone https://github.com/MicroPythonOS/secp256k1-embedded-ecdh
```

2. **Build for ESP32**:
```bash
cd ~/MicroPythonOS/MicroPythonOS
./scripts/build_lvgl_micropython.sh esp32 prod
```
   For a development build (no preinstalled files):
```bash
./scripts/build_lvgl_micropython.sh esp32 dev
```

3. **Flash to ESP32**:
   - Put your ESP32 in bootloader mode (long-press the BOOT button if running MicroPythonOS).
   - Flash the firmware:
```bash
./scripts/flash_over_usb.sh
```
   - For a development build, install files manually:
```bash
./scripts/install.sh
```

## Installing on Desktop (Linux/MacOS)

1. **Install Dependencies** (Linux):
```bash
sudo apt install libv4l-dev  # For webcam support
```
   See [lvgl-micropython](https://github.com/MicroPythonOS/lvgl-micropython) for additional dependencies.

2. **Build for Desktop**:
```bash
cd ~/MicroPythonOS/MicroPythonOS
./scripts/build_lvgl_micropython.sh unix dev
```
   For MacOS (untested):
```bash
./scripts/build_lvgl_micropython.sh macOS dev
```

3. **Run on Desktop**:
   - Download a release (e.g., `MicroPythonOS_amd64_Linux_0.0.6`) or use your build.
   - Run:
```bash
cd internal_filesystem/
/path/to/MicroPythonOS_amd64_Linux_0.0.6 -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
```
   - See `scripts/run_on_desktop.sh` for options like fullscreen or direct app launch.

## Next Steps

- Check [Supported Hardware](supported-hardware.md) for compatible devices.
- Explore [Built-in Apps](../apps/built-in-apps.md) to get started with the system.
