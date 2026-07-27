# Hijacking the Boot Sequence

You can intercept the boot sequence to drop into a raw REPL shell instead of starting MicroPythonOS — useful for debugging, running custom startup code, or recovering from a broken install.

## The one-liner

```python
__import__("os").mkdir("/lib") or True; __import__("os").mkdir("/lib/mpos") or True; open("/lib/mpos/main.py","w").write('raise RuntimeError("/lib/mpos/main.py: dropping to REPL shell. To resume boot, do: import mpos.main")\n')
```

Copy-paste this into your device's REPL (e.g. over serial) and hit enter. The `or True` handles the case where the directories already exist.

## What it does

`/lib/mpos/main.py` is MicroPythonOS's entry point. When the firmware boots, it looks for `main.py` on the filesystem — if found, it runs it first. This one-liner writes a stub that raises an error instead, which means:

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

To restore normal boot on every power-on, delete the stub:

```python
__import__("os").remove("/lib/mpos/main.py")
```
