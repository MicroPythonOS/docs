# Installation

MicroPythonOS can be installed on supported microcontrollers (e.g., ESP32) and on desktop systems (Linux, Raspberry Pi, MacOS, etc).

If you're a developer, you can [build it yourself and install from source](../building/index.md)

To simply install prebuilt software, read on!

## Installing on ESP32

Just use the [WebSerial installer at install.micropythonos.com](https://install.micropythonos.com).

## Installing on Desktop (Linux/MacOS)

Download the [latest release for desktop](https://github.com/MicroPythonOS/MicroPythonOS/releases).

Here we'll assume you saved it in /tmp/MicroPythonOS_amd64_Linux_0.0.8

Get the internal_filesystem files:

```
git clone https://github.com/MicroPythonOS/MicroPythonOS.git
cd MicroPythonOS/
cd internal_filesystem/
```

Now run it by starting the entry points, boot_unix.py and main.py:

```
/tmp/MicroPythonOS_amd64_Linux_0.0.8 -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
```

You can also check out `scripts/run_desktop.sh` for more examples, such as immediately starting an app or starting fullscreen.

## Next Steps

- Check [Supported Hardware](supported-hardware.md) for compatible devices.
- Explore [Built-in Apps](../apps/built-in-apps.md) to get started with the system.
