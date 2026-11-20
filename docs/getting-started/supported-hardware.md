# Supported Hardware

MicroPythonOS runs on a variety of platforms, from microcontrollers to desktops.

## Specifications

Board files are the source of truth for what the OS initializes. Alphabetical.

| Board | MCU | Flash | PSRAM | Display | Size | Driver | Touch | Pts | SD | Audio | IMU | Camera | Battery |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| D-Shell | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.8" | ST7789 | GT911 | 5 | — | Speaker + buzzer | QMI8658 | — | — |
| Desktop / web | host | — | — | window | — | SDL | SDL pointer | 1 | host | host | mock | webcam / mock | — |
| Freenove ESP32-S3 Display | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.8" | ILI9341V | FT6336G | 2 | 4-bit SDMMC | Speaker + mic | — | — | ADC |
| fri3d 2024 | ESP32-S3 | 16 MB | 8 MB | 240×296 | 2.0" | ST7789 | — | 0 | SPI | Speaker + mic + buzzer | LSM6DSO | — | ADC |
| fri3d 2026 | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.0" | ST7789 | CST816S | 1 | SPI | Speaker + mic + buzzer | LSM6DSO | — | Expander ADC |
| LilyGO T-Display S3 | ESP32-S3 | 16 MB | 8 MB | 170×320 | 1.9" | ST7789 | — | 0 | — | — | — | — | — |
| LilyGO T-HMI | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.8" | ST7789 | XPT2046 | 1 | SDMMC | — | — | — | ADC |
| LilyGO T-Watch S3 Plus | ESP32-S3 | 16 MB | 8 MB | 240×240 | 1.3" | ST7789 | FT6x36 | 2 | — | Speaker + mic | BMA423 | — | AXP2101 |
| LilyGO T4 | ESP32 | 4 MB | 8 MB | 240×320 | 2.4" | ILI9341 | — | 0 | SPI | Buzzer | — | — | ADC |
| M5Stack Core2 | ESP32 | 16 MB | 8 MB | 320×240 | 2.0" | ILI9342C | FT6336U | 2 | — | Speaker + mic | MPU6886 | — | AXP192 |
| M5Stack Fire | ESP32 | 16 MB | 8 MB | 320×240 | 2.0" | ILI9342 | — | 0 | — | Buzzer | MPU6886 | — | — |
| Makerfabs MaTouch 2.8 | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.8" | ST7789 | GT911 | 5 | SDIO | — | — | OV3660 | — |
| ODROID-GO | ESP32-PICO-D4 | 4 MB | — | 240×320 | 2.4" | ILI9341 | — | 0 | — | Buzzer | — | — | ADC |
| SQUiXL | ESP32-S3 | 16 MB | 8 MB | 480×480 | 4.0" | ST7701S | GT911 | 5 | SD | Speaker | — | — | MAX1704X |
| UniHiker K10 | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.8" | ILI9341 | — | 0 | — | Speaker + mic (not enabled) | — | GC2145 | — |
| unPhone 9 | ESP32-S3 | 8 MB | 8 MB | 320×480 | 3.5" | HX8357D | XPT2046 | 1 | — | — | — | — | — |
| Waveshare ESP32-P4 4.3 | ESP32-P4 | 32 MB | 32 MB | 480×800 | 4.3" | ST7701S | GT911 | 5 | — | Speaker | — | — | — |
| Waveshare ESP32-S3 1.54 | ESP32-S3 | 16 MB | 8 MB | 240×240 | 1.54" | ST7789 | CST816S | 1 | SD | Speaker + mic | — | — | ADC |
| Waveshare ESP32-S3 2 | ESP32-S3 | 16 MB | 8 MB | 240×320 | 2.0" | ST7789 | CST816S | 1 | — | — | QMI8658 | OV5640 | ADC |
| Waveshare ESP32-S3 3.5 | ESP32-S3 | 16 MB | 8 MB | 320×480 | 3.5" | ST7796 | FT6336 | 2 | — | Speaker + mic | QMI8658 | OV5640/OV2640 | AXP2101 |

Flash and PSRAM are the module fitted on the board; vendors ship variants, and the
value listed is the one MicroPythonOS is tested on. `—` means the board file does
not wire up that peripheral (the hardware may still provide it); the SD column is
what the board file initializes, not what the PCB exposes. `Pts` is the maximum
number of touch points the controller reports; `0` means no touchscreen.

## ESP32 Microcontrollers

- [M5Stack Fire](https://docs.m5stack.com/en/core/fire): Fully Supported
- [M5Stack Core2](https://docs.m5stack.com/en/core/core2): Fully Supported
- [HardKernel ODROID-GO](https://wiki.odroid.com/odroid_go/odroid_go): Fully Supported
- [LilyGo T4 V1.3](https://github.com/Xinyuan-LilyGO/LilyGo_Txx): Fully Supported

## ESP32-S3 Microcontrollers

- [D-Shell ESP32-S3 N16R8](https://github.com/FreakStudioCN/MicroPython_Claude_Assistant_Public): Fully Supported
- [DFRobot UniHiker K10](https://www.dfrobot.com/product-2676.html): Fully Supported
- [Fri3d Camp 2024 Badge](https://fri3d.be/badge/2024/): Fully Supported (including Communicator Add-On)
- [Fri3d Camp 2026 Badge](https://fri3d.be/badge/2026/): Fully Supported (including Communicator Add-On)
- [Freenove ESP32-S3 Display](https://github.com/Freenove/Freenove_ESP32_S3_Display): Fully Supported
- [LilyGo T-Display S3](https://lilygo.cc/products/t-display-s3): Fully Supported
- [LilyGo T-HMI](https://lilygo.cc/en-us/products/t-hmi): Fully Supported
- [LilyGo T-Watch S3 Plus](https://lilygo.cc/products/t-watch-s3-plus): Fully Supported
- [Makerfabs MaTouch ESP32-S3 SPI IPS 2.8" with Camera OV3660](https://www.makerfabs.com/matouch-esp32-s3-spi-ips-2-8-with-camera-ov3660.html): Fully Supported
- [SQUiXL](https://squixl.io): Fully Supported
- [unPhone 9](https://unphone.net/): Fully Supported
- [Waveshare ESP32-S3-Touch-LCD-1.54](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.54): Fully Supported
- [Waveshare ESP32-S3-Touch-LCD-2](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-2): Fully Supported
- [Waveshare ESP32-S3-Touch-LCD-3.5](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-3.5): Fully Supported

## ESP32-P4 Microcontrollers

- [Waveshare ESP32-P4-WIFI6-Touch-LCD-4.3](https://docs.waveshare.com/ESP32-P4-WIFI6-Touch-LCD-4.3): Supported on ESP32-P4 chip revisions v0.x and v1.x (tested on v1.3), not v3.x. The microphones, TF card slot, camera, USB host and battery level are not supported. See [its page](waveshare-esp32-p4-wifi6-touch-lcd-4-3.md), including a note on its ESP32-C6 Wi-Fi firmware.

## WebAssembly (Browser)

- **Latest release**: [web.micropythonos.com](https://web.micropythonos.com/)
- **Main branch CI**: [micropythonos.github.io/MicroPythonOS](https://micropythonos.github.io/MicroPythonOS/)
- Runs entirely in a modern desktop browser — no install, no flash.
- Built from source with `./scripts/build_mpos.sh web` (requires [Emscripten SDK](../web-port/developer.md#prerequisites)).
- See [CI config](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/.github/workflows/web-pages.yml) for automated build details.

## Desktop Computers

- **Linux**: Supported using SDL for display handling.
- **MacOS**: Supported as well.
- **Windows**: [Users report](https://github.com/MicroPythonOS/MicroPythonOS/issues/31) that the Linux desktop version runs fine under WSL2 on Windows 11.

## Raspberry Pi

- **Raspbian and other Linux-based**: Should work!

## Adding a new board

If your device is not listed, see the [Porting Guide](../os-development/porting-guide.md).
