# Hijacking the Boot Sequence

You can intercept the boot sequence to drop into a raw REPL shell instead of starting MicroPythonOS — useful for debugging, running custom startup code, or recovering from a broken install.

## How to Hijack the Boot Sequence

Essentially, you have to create a file `/lib/mpos/main.py` on the filesystem.

Code you place there will be executed. If an exception is thrown, it will be printed on the serial port and a REPL shell will be started.

## The one-liner

```python
__import__("os").mkdir("/lib") or True; __import__("os").mkdir("/lib/mpos") or True; open("/lib/mpos/main.py","w").write('raise RuntimeError("/lib/mpos/main.py: starting REPL shell. To resume boot, do: import mpos.main")\n')
```

Copy-paste this into your device's REPL (e.g. over serial) and hit enter. The `or True` handles the case where the directories already exist.

There are other ways to create this file:

- use mpremote.py to mkdir and then cp the new main.py or
- use a Web IDE like https://fri3dcamp.github.io/Fri3d-IDE/ which have file managers that can create and edit files and folders

## How this works

When the firmware boots, the built-in `main.py` (from `internal_filesystem/main.py` in the repo) adds `/lib` to the start of sys.path and then executes `import mpos.main`.

Since the internal filesystem's `/lib` is on the sys.path _before_ the .frozen library folder, placing `mpos/main.py` in `/lib` will take precedence and will get executed.

The one-liner above writes a stub that raises an exception, which means:

1. The device boots into a clean MicroPython REPL.
2. No MicroPythonOS code (LVGL, apps, frameworks) is loaded.
3. You have full access to the filesystem and MicroPython stdlib.

## Resuming normal boot

From that REPL, you can start MicroPythonOS manually:

```python
import mpos.main
```

This runs the full boot sequence (LVGL init, app loading, launcher) as if nothing happened.

## Undoing the hijack

To restore normal boot on every power-on, delete the override and its containing folder:

```python
import shutil ; shutil.rmtree("/lib/mpos")
```

## Overriding Board Detection

If `DeviceInfo.hardware_id` is already set before `import mpos.main`, the normal board detection is skipped entirely. This lets you force a specific board file or do custom initialization without modifying the firmware.

### Force a known board

```python
from mpos import DeviceInfo
DeviceInfo.set_hardware_id("linux")
import mpos.main
```

The matching `mpos/board/linux.py` will be imported as usual.

### Custom/new device (no board file)

Set a hardware ID that has no corresponding `mpos/board/*.py` file. Put any init code (display, touch, etc.) directly in `/lib/mpos/main.py` before the import:

```python
from mpos import DeviceInfo
from mpos import DisplayMetrics
import lvgl as lv

# Custom init for a new board
DisplayMetrics.set_resolution(480, 320)
DisplayMetrics.set_dpi(160)
# ... more init as needed ...

DeviceInfo.set_hardware_id("my_new_board")
import mpos.main
```

`mpos.main` will log a warning about the missing board file and continue booting. This is useful when adding support for a new device using a prebuilt image — you can prototype the board init in `/lib/mpos/main.py` without rebuilding the firmware. Once stable, create `mpos/board/my_new_board.py` and include it in the next firmware build.
