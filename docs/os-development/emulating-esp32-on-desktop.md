## Running in a QEMU ESP32 emulator on desktop

For development and automated testing, it can be very useful to run MicroPythonOS on an emulated ESP32 device.

Compared to running on desktop, the emulated ESP32 device offers more capabitlities:

- emulated WiFi scanning (finds a hard-coded list of access points)
- emulated WiFi connection (doesn't support encryption, so leave password blank)
- emulated WiFi Access Point (AP) mode
- emulated storage, which enables LittleFS2 filesystem and Over-The-Air update testing
- emulated Ultra Low Processor and DeepSleep
- emulated GPIO buttons
- emulated capacitive touch buttons
- emulated ST7789V display

In the future, more features might land:

- [emulated Bluetooth](https://github.com/Ebiroll/qemu_esp32) - currently only on ESP32 but maybe [someday on ESP32S3](https://github.com/Ebiroll/calib/issues/1).

To this end, we're using the amazing a159x36 QEMU emulator, which can emulate the LilyGo T-Display (ESP32) and LilyGo T-Display-S3 (ESP32S3), with [a few improvements](https://github.com/a159x36/qemu/pulls?q=is%3Apr) waiting to be merged upstream.

To get it running, follow these steps:

1. Compile the ESP32 QEMU
2. Pad a MicroPythonOS release image to 16MB (otherwise QEMU refuses it)
3. Start QEMU with the padded MicroPython release image

There are also [GitHub workflows](https://github.com/MicroPythonOS/qemu/tree/esp-develop-9.2/.github/workflows) that compile the ESP32 QEMU for Linux, Windows, MacOS (intel) and MacOS (arm) so you might be able to download a prebuilt version from the [GitHub actions](https://github.com/MicroPythonOS/qemu/actions) artifacts. You need to be logged in to download them.

## 1. Compile QEMU

Get the prerequisites:

```
sudo apt-get install -y -q --no-install-recommends build-essential libgcrypt-dev libglib2.0-dev libpixman-1-dev libsdl2-dev libslirp-dev ninja-build python3-pip libvte-2.91-dev wget zlib1g-dev
```

Clone it:

```
git clone https://github.com/MicroPythonOS/qemu/
cd qemu
```

Configure it:

```
./configure --target-list=xtensa-softmmu --enable-gcrypt --enable-slirp --enable-debug --enable-vte --enable-stack-protector
```

Compile it:

`make -j4 # 4 is the number of cores` 

## 2. Pad a MicroPythonOS build

Download the [latest release](https://github.com/MicroPythonOS/MicroPythonOS/releases) or use one you built yourself.

The ESP32 QEMU refuses it if it's not a supported size (2, 4, 8 or 16MB) so just pad it with zeroes.

A simple way to do this on Linux is:

```
dd if=MicroPythonOS_esp32_0.8.0.bin of=MicroPythonOS_esp32_0.8.0.bin.padded bs=1M count=16 oflag=append conv=notrunc
```

## 3. Start QEMU

```
./build/qemu-system-xtensa -machine esp32s3 -m 8M -drive file=MicroPythonOS_esp32_0.8.0.bin.padded,if=mtd,format=raw -nic user,model=esp32_wifi,hostfwd=tcp:127.0.0.1:10080-192.168.4.15:80,net=192.168.4.0/24,host=192.168.4.2,hostfwd=tcp:127.0.0.1:1081-192.168.4.1:80,hostfwd=tcp:127.0.0.1:7890-192.168.4.15:7890,hostfwd=tcp:127.0.0.1:7891-192.168.4.1:7890 -display default,show-cursor=on -parallel none -monitor none -serial stdio
```

The hostfwd lines forward:

- `127.0.0.1:7890 to 192.168.4.15:7890` for webrepl webserver on port 7890 of the device in Station (WiFi client) mode
- `127.0.0.1:7891 to 192.168.4.1:7890` for webrepl webrepl on port 7890 of the ESP32 in Access Point (Hotspot) mode

Also these are forwarded, although currently unused:

- `127.0.0.1:10080 to 192.168.4.15:80` when the ESP32 is in Station (WiFi client) mode
- `127.0.0.1:10081 to 192.168.4.1:80` when the ESP32 is in Access Point (Hotspot) mode

**Note**: QEMU will only log to stdout if you start it from a normal shell because then stdin is be set. If you're starting it from a script or other tool that doesn't set stdin, consider adding the `unbuffer` tool as a prefix at the start of the command to make sure stdin is set, otherwise you won't see much logging.

## Controls

To navigate around in the emulated T-Display S3:

- press `r` for the reset button
- press `1` for the GPIO0 button - mapped to the "previous" action
- press `2` for the GPIO14 button - mapped to the "next" action
- press both for the "enter" action
- long press `1` for the "back" action

You can also click the buttons on the picture of the board if you like.

Also, currently unused:

- 7, 8, 9 and 0 are bound to capacitive touch input pins 0, 1, 11 and 12 which correspond to GPIO01, GPIO02, GPIO12 and GPIO13

The code for these controls is in QEMU's `hw/xtensa/esp32s3.c` and `hw/display/st7789v.c`.

## Building an image with a filesystem

For quick development, rather than doing a full rebuild, you could also just rebuild the filesystem based on `internal_filesystem/` and bundle that as a LittleFS2 filesystem in the image.

Run `./scripts/make_image.sh` from the [MicroPythonOS repo](https://github.com/MicroPythonOS/MicroPythonOS) to do so.

It will call `./scripts/mklittlefs.sh` to build the filesystem and bundle it with all the other requisites, giving you a bootable image in one second.

