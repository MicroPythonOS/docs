# Waveshare ESP32-P4-WIFI6-Touch-LCD-4.3

The [Waveshare ESP32-P4-WIFI6-Touch-LCD-4.3](https://docs.waveshare.com/ESP32-P4-WIFI6-Touch-LCD-4.3) is the first ESP32-P4 board that MicroPythonOS supports, starting with release 0.20.0. It has a 4.3" 480x800 touch screen, 32 MB of flash and 32 MB of PSRAM.

**Check the chip revision before you install.** MicroPythonOS runs only on ESP32-P4 chip revisions v0.x and v1.x. Newer boards can have a v3.x chip, which MicroPythonOS does not support. See [Chip revision](#chip-revision).

The ESP32-P4 has no radio of its own. Wi-Fi and Bluetooth come from a second chip on the board, an ESP32-C6, which runs its own firmware and talks to the ESP32-P4 over SDIO using Espressif's [ESP-Hosted](https://github.com/espressif/esp-hosted-mcu). This page calls it the C6.

## Chip revision

MicroPythonOS 0.20.0 runs only on an ESP32-P4 of chip revision v0.x or v1.x. Its `esp32p4` image is built with ESP-IDF 5.5.1, which supports revisions v0.0 to v1.99. The main branch builds it the same way. The board MicroPythonOS support was developed on has revision v1.3.

Boards with a v3.x chip are not supported. Waveshare's [FAQ](https://docs.waveshare.com/ESP32-P4-WIFI6-Touch-LCD-4.3/FAQ) calls v0.x and v1.x chips engineering samples, and one user's board of this model has a v3.2 chip. Espressif's [chip revision v3.x guide](https://documentation.espressif.com/esp32-p4-chip-revision-v3.x_user_guide_en.html) says that v1.x and v3.x chips cannot share the same firmware image, and that v3.x needs ESP-IDF 5.5.3 or later. Support for v3.x is tracked in [MicroPythonOS/MicroPythonOS#344](https://github.com/MicroPythonOS/MicroPythonOS/issues/344).

To check your board, connect it as in step 1 of [Installing](#installing) and run:

```bash
python3 -m esptool --chip esp32p4 chip_id
```

esptool prints the revision when it connects. esptool v4 prints `Chip is ESP32-P4 (revision v1.3)`. esptool v5 prints a `Chip type:` line that ends with `ESP32-P4 (revision v1.3)`. esptool then warns that the ESP32-P4 has no chip ID and prints its MAC address instead. esptool v5 also notes that `chip_id` is deprecated. Both messages are expected.

If your board has a v3.x chip, keep Waveshare's factory firmware on it. Do not get past the revision limit with esptool's `--force`, or by raising the limit when you build MicroPythonOS. The image is then written, but the board restarts over and over and MicroPythonOS never starts. A v3.2 board flashed this way reset in a loop with `rst:0x10 (CHIP_LP_WDT_RESET)`. Writing Waveshare's factory firmware back made it work again.

## What works and what doesn't

| Part | Status |
|---|---|
| ESP32-P4 chip revision | v0.x and v1.x only. v3.x is not supported. See [Chip revision](#chip-revision). |
| Display | Works. The panel is portrait (480x800); MicroPythonOS shows its usual landscape UI at 800x480, rotated by the ESP32-P4's pixel processing accelerator (PPA). |
| Brightness | Works, with the brightness slider in the drawer. |
| Touch | Works (GT911). |
| Speaker | Works: ES8311 codec and the on-board amplifier, on the speaker connector. |
| Wi-Fi | Works, through the C6. Read [the C6 firmware](#the-c6-firmware) and [Wi-Fi on channels 6 to 9](#known-issue-wi-fi-on-channels-6-to-9). |
| OS updates | The update server has an entry for this board, so the OSUpdate app checks for new releases. Not tested yet: 0.20.0 is the board's first release, so there has been nothing newer to install. See [Updating](#updating). |
| Microphones | Not supported. They go through a separate ES7210 chip, which MicroPythonOS does not set up on this board. |
| TF (microSD) card slot | Not supported. The slot needs the ESP32-P4's internal LDO channel 4 switched on, which `machine.SDCard` cannot do. |
| Camera (MIPI-CSI) | Not supported. |
| USB host (USB OTG port) | Not supported. MicroPythonOS builds USB host support for the ESP32-S3 only. |
| Battery | The board has a connector for a lithium battery, but MicroPythonOS does not read the battery level on this board. |
| Bluetooth | Not tested with MicroPythonOS on this board. |

## Installing

The firmware for this board is the `esp32p4` build: the file `MicroPythonOS_esp32p4_<version>.bin` on the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases), first in 0.20.0. It is a full flash image, written at offset 0.

The [web installer](https://install.micropythonos.com) lists an ESP32-P4 build, but that option currently fails to load. The steps below use [esptool](https://docs.espressif.com/projects/esptool/en/latest/esp32p4/). In the commands, `-b 921600` sets a faster serial speed than esptool's default.

1. **Connect the board.** Use the USB-C port that Waveshare's documentation calls USB TO UART (power, programming and debugging). The other USB-C port is the USB OTG port.

2. **Check the chip revision**, as described in [Chip revision](#chip-revision):

        python3 -m esptool --chip esp32p4 chip_id

    If the revision is v3.0 or later, stop here. MicroPythonOS will not start on this board, and the next steps would erase its factory firmware.

    Do not count on esptool to stop you. The release `.bin` starts with padding, not with an image header, so esptool writes it without checking the revision. esptool checks only files that start with an image header, such as the `.ota` file or a bootloader you built yourself. Up to v5.4.0, it refuses any such file built for other revisions. From v5.5.0, it refuses only a bootloader written at offset 0x2000. Either way, `--force` does not help: the image is written, but the board does not start.

3. **Back up the factory firmware (optional).** The board comes with Waveshare's demo firmware. To be able to go back to it, save the whole 32 MB flash first:

        python3 -m esptool --chip esp32p4 -b 921600 read_flash 0 0x2000000 waveshare-p4-factory.bin

    To restore it later, write the file back at offset 0: `python3 -m esptool --chip esp32p4 -b 921600 write_flash 0 waveshare-p4-factory.bin`.

4. **Write MicroPythonOS:**

        python3 -m esptool --chip esp32p4 -b 921600 write_flash --erase-all 0 MicroPythonOS_esp32p4_0.20.0.bin

    `--erase-all` clears the factory firmware's data, which uses a different partition layout. Leave it out when you reinstall over MicroPythonOS, as it erases your apps and settings. Even without it, reinstalling can cost you data: read [Updating](updating.md) first.

    esptool puts the board into download mode by itself. If it cannot connect, hold the BOOT button, press RESET, release BOOT and try again.

5. **Start MicroPythonOS.** esptool resets the board when it is done. If the screen stays dark, press RESET.

## Updating

Use the OSUpdate app, as on the other boards. See [Updating](updating.md). This has not been tested on this board yet: 0.20.0 is its first release, so there has been no newer release to update to.

For a manual offline update, also described there, use the `MicroPythonOS_esp32p4_<version>.ota` file from the releases page. The two OTA slots of the `esp32p4` build are at `0x20000` and `0x420000`.

## The C6 firmware

The C6 firmware should be updated. On the board MicroPythonOS support was developed on, the C6 came with ESP-Hosted firmware older than version 1.4.7. MicroPythonOS's `esp32p4` build uses ESP-Hosted 2.7.0 on the ESP32-P4 side.

Why it matters:

- **Empty scans.** With the original C6 firmware, a Wi-Fi scan made straight after the radio was switched on found nothing, in about 1.2 s. A scan 5 s later found 26 networks. While it is not connected, MicroPythonOS switches the radio off and on before every scan. So after losing Wi-Fi, the board could miss its network and switch Wi-Fi off again ([MicroPythonOS/MicroPythonOS#309](https://github.com/MicroPythonOS/MicroPythonOS/issues/309)).
- **After the update.** On the same board, after its C6 was updated to ESP-Hosted 2.7.0, none of 22 scans made straight after a radio restart came back empty.
- **A fallback, not on 0.20.0.** On the main branch, when the scan after a radio restart finds nothing, MicroPythonOS scans once more 5 s later ([MicroPythonOS/MicroPythonOS#341](https://github.com/MicroPythonOS/MicroPythonOS/pull/341); see [`WifiService.scan_networks()`](../frameworks/wifi-service.md#scanning-functions)). Release 0.20.0 does not have this. It could not be tested on the original C6 firmware, as no copy of that firmware was available.
- **Espressif's advice.** ESP-Hosted's own driver recommends upgrading a co-processor whose firmware is older than the host side's, to avoid timeouts in the requests between the two chips.

### Check the version

At the MicroPython REPL, for example with [mpremote](https://docs.micropython.org/en/latest/reference/mpremote.html) on the USB TO UART port. Connect once MicroPythonOS has finished starting: mpremote interrupts a start-up that is still running, which leaves Wi-Fi off. Press RESET if that happens.

```python
import hosted
hosted.host_version()         # ESP-Hosted on the ESP32-P4: (2, 7, 0)
hosted.coprocessor_version()  # ESP-Hosted on the C6
```

`coprocessor_version()` returns a version such as `(2, 7, 0)`. It raises `OSError` if the C6 does not answer the version request. The original firmware did not answer it. MicroPythonOS was tested with ESP-Hosted 2.7.0 on both chips.

### Update it

MicroPythonOS has no supported way to update the C6 firmware: there is no prebuilt C6 image and no step-by-step guide. Flashing the wrong firmware can leave the board without Wi-Fi.

What exists today:

- **The `hosted` module.** It has the low-level calls that send a new firmware image to the C6 over the link between the two chips: `hosted.ota_begin()`, `hosted.ota_write(data)`, `hosted.ota_end()` and `hosted.ota_activate()`. The original firmware switches to the new image at `ota_end()`; `ota_activate()` is for ESP-Hosted 2.x. The C6 on the board MicroPythonOS was developed on was updated to ESP-Hosted 2.7.0 this way, with an image built from Espressif's ESP-Hosted co-processor sources for the ESP32-C6.
- **Espressif's example.** ESP-Hosted's [co-processor OTA example](https://github.com/espressif/esp-hosted-mcu/tree/main/examples/ota/coprocessor_ota) shows the same kind of update, driven from the host chip.
- **Waveshare's FAQ.** The board's [FAQ](https://docs.waveshare.com/ESP32-P4-WIFI6-Touch-LCD-4.3/FAQ) ("How to flash firmware onto the ESP32-C6?") describes flashing the C6 directly through its own UART pins, with both chips in download mode. Waveshare warns that the firmware has to be compatible.

## Known issue: Wi-Fi on channels 6 to 9

On the board MicroPythonOS support was developed on, the C6 joins a network on 2.4 GHz channels 6 to 9 slowly or not at all. With a test access point 1 m away:

- channels 1 and 11 joined in 3 to 5 s;
- channel 6 joined in 6 of 12 attempts, after 14 to 30 s;
- channels 7 to 9 joined in none of 11 attempts on one day; 7 and 8 had each joined once on another day.

Results varied between sessions. Channel 6 also fails with Waveshare's own ESP-IDF Wi-Fi example and with plain ESP-IDF, so it is not specific to MicroPythonOS. It failed with the original C6 firmware and still fails with the updated one. Another user reported the same problem on the ESP32-P4-WIFI6-Touch-LCD-3.4C. The cause is not known. It is open as [waveshareteam/ESP32-P4-WIFI6-Touch-LCD-4.3#7](https://github.com/waveshareteam/ESP32-P4-WIFI6-Touch-LCD-4.3/issues/7).

MicroPythonOS gives this board 30 s to join a network instead of the usual 13 s (`WifiService.CONNECT_TIMEOUT_S`), so a slow join on channel 6 can still succeed.

If your board does not connect, or keeps losing Wi-Fi, and your router's 2.4 GHz network is on channel 6, 7, 8 or 9, set the router to channel 1 or 11.

## Building from source

```bash
./scripts/build_mpos.sh esp32p4
```

This builds MicroPython's `ESP32_GENERIC_P4` board with the `C6_WIFI` variant, for 32 MB of flash with two 4 MiB app partitions. The full flash image is `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_P4-C6_WIFI-32.bin`, and the app-only image (for the OTA slots) is `lvgl_micropython/lib/micropython/ports/esp32/build-ESP32_GENERIC_P4-C6_WIFI/micropython.bin`. See [OS Development](../os-development/index.md) for the toolchain.

## How MicroPythonOS drives the board

The board file is [`waveshare_esp32_p4_wifi6_touch_lcd_4_3.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/lib/mpos/board/waveshare_esp32_p4_wifi6_touch_lcd_4_3.py). MicroPythonOS recognises the board by its GT911 touch controller, at I2C address 0x5D or 0x14 on SDA 7 / SCL 8.

- **Display:** ST7701 panel over 2-lane MIPI-DSI (`lcd_bus.DSIBus`) at 500 Mbps per lane, RGB565, with the frame buffers in PSRAM. `LCD_ROTATION = 270` shows the landscape UI upright; 90 is upside down and 0 is the panel's native portrait.
- **Backlight:** GPIO 26, PWM, active low.
- **Touch:** GT911 on I2C (SDA 7, SCL 8). It is polled, as its reset and interrupt lines are not connected to the ESP32-P4.
- **Speaker:** ES8311 codec at I2C address 0x18; I2S MCLK 13, BCLK 12, LRCK 10, data out 9; amplifier enable on GPIO 53. The output stays open for 30 s after a clip, so clips played back to back do not click.
- **Wi-Fi:** the C6, over SDIO (ESP-Hosted). The board file sets `WifiService.CONNECT_TIMEOUT_S = 30`.
