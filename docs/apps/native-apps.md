# Native C/C++ Apps

Most MicroPythonOS apps are written in MicroPython, but apps can also include native extension modules written in C or C++. These modules are compiled to MicroPython `.native.mpy` files and bundled inside the app's `.mpk` like any other module.

## When to use native code

Use a native module when:

- You need more CPU performance than pure MicroPython can provide (for example, a game loop or signal processing).
- You need to call C libraries that have no MicroPython equivalent.
- You want to reuse existing C/C++ code.

For everything else, prefer plain MicroPython — native modules add build complexity and are architecture-specific.

## How native modules work

MicroPython supports [native machine code modules](https://docs.micropython.org/en/latest/develop/natmod.html) (often called *natmods*). A natmod is a self-contained `.mpy` file that contains compiled machine code and a MicroPython module descriptor. At runtime it is imported just like a `.py` file:

```python
import breakout
breakout.start()
```

MicroPythonOS places the `.mpy` file in the app folder when the `.mpk` is installed. The AppManager's import path then finds it automatically.

## Example: `com.micropythonos.breakout`

The built-in **Breakout** game uses a native C module for the game logic. Its source lives in `c_mpos/breakout/` in the main repository.

Project layout:

```
c_mpos/breakout/
├── Makefile
├── build.sh
└── breakout.c
```

`Makefile`:

```makefile
MPY_DIR = ../../lvgl_micropython/lib/micropython/

MOD = breakout
SRC = breakout.c
LINK_RUNTIME = 1
ARCHES = x64 xtensawin

ifeq ($(ARCH),)
.PHONY: all $(ARCHES)

all: $(ARCHES)

$(ARCHES):
	$(MAKE) -f $(lastword $(MAKEFILE_LIST)) ARCH=$@ MOD=$(MOD)_$@
else
include $(MPY_DIR)/py/dynruntime.mk
endif
```

This builds two artifacts:

- `breakout_x64.native.mpy` — for the desktop simulator.
- `breakout_xtensawin.native.mpy` — for ESP32-S3 devices.

The `build.sh` script sets up the Espressif toolchain and then copies the resulting `.mpy` files into the app's folder at `internal_filesystem/apps/com.micropythonos.breakout/`.

## Bundling a native module

1. Write your C/C++ source and a `Makefile` based on the MicroPython `dynruntime.mk` rules.
2. Build the module for every architecture you want to support.
3. Rename or place the output `.mpy` file in the app folder so its import name matches what your MicroPython code imports. For Breakout, the `xtensawin` build is named `breakout.mpy` inside the app folder.
4. Build the `.mpk` normally. The native `.mpy` is packaged alongside the Python files.

```
com.micropythonos.breakout/
├── MANIFEST.JSON
├── icon_64x64.png
├── main.py
└── breakout.mpy      # native module, imported by main.py
```

## Limitations

- Native modules must be compiled separately for each target architecture (for example, `x64` for desktop and `xtensawin` for ESP32).
- The MicroPython natmod ABI can change between releases, so rebuild modules when the `lvgl_micropython` submodule is updated.
- Native code has the same filesystem and memory constraints as the rest of the app.

## See also

- [Bundling Apps](bundling-apps.md) — packaging apps as `.mpk` files
- [MicroPython Native Module Documentation](https://docs.micropython.org/en/latest/develop/natmod.html)
- [Creating Apps](creating-apps.md) — writing an app from scratch
