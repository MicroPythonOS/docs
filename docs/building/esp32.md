# Building for ESP32

Build MicroPythonOS for ESP32 microcontrollers, such as the Waveshare ESP32-S3-Touch-LCD-2.

## Prerequisites

- Clone the required repositories:
  ```bash
  mkdir ~/MicroPythonOS
  cd ~/MicroPythonOS
  git clone https://github.com/MicroPythonOS/MicroPythonOS.git
  git clone https://github.com/MicroPythonOS/freezeFS
  git clone https://github.com/cnadler86/micropython-camera-API
  echo 'include("~/MicroPythonOS/lvgl_micropython/build/manifest.py")' >> micropython-camera-API/src/manifest.py
  git clone https://github.com/MicroPythonOS/lvgl_micropython
  git clone https://github.com/MicroPythonOS/secp256k1-embedded-ecdh
