# Doom

Doom (the shareware episode, *Knee-Deep in the Dead*) runs on ESP32-S3 badges
and on the desktop from one package, `com.micropythonos.doom`. It is not
preinstalled: install it from the App Store or [BadgeHub](badgehub.md).

One package carries both engines and the installer keeps only the one the
device can run:

| Host | Engine file | How it runs |
|------|-------------|-------------|
| ESP32-S3 badge | `esp32s3.xip` | flash-resident core, mapped from a littlefs2e extent and executed from flash through the cache |
| x86-64 desktop | `module_x64.mpy` | MicroPython native module |

The badge engine is too large to fit in executable RAM, so it never does: the
App Store writes it into a contiguous extent at install time, a preloaded
factory image already carries it as one, and the loader maps it where it was
linked. There is no first-launch conversion in either case; only a copy that
was sideloaded as an ordinary file is converted once (with a "Preparing
engine" screen, after defragmenting room for it).

The shareware WAD ships deflated as `doom1.wad.zip` and is unpacked into RAM at
each launch (PSRAM is volatile), so a first launch costs a few seconds with an
"Unpacking WAD" screen. A plain `doom1.wad` — next to the app, at `/doom1.wad`,
or on the SD card under `/sdcard/doom1.wad` — is used as a file instead and
skips the unpack.

## Requirements

- A badge needs a MicroPythonOS with littlefs2e extent support: the app checks
  for it and says "Doom needs a newer MicroPythonOS" instead of failing
  silently. The package is listed for ESP32-S3 and x64; there is no engine for
  the P4, classic ESP32, ARM64 or the web build yet.
- Sound needs an i2s output. On a piezo-only board Doom is silent on purpose:
  11 kHz broadband effects through a resonator are a screech.

## Controls

Hold the badge sideways while playing.

| Touch | Action |
|-------|--------|
| left half | joystick: drag left/right to turn, up/down to walk |
| right, bottom | fire (hold to keep shooting) |
| right, middle | use / open |
| right, top | menu / back; **hold it** for the licenses |

The touch controls are outlined for a few seconds at start and whenever the
menu opens, so the outline never costs a frame while playing.

| Button | Action |
|--------|--------|
| arrows / joystick | turn and walk |
| Enter / A | use, open doors, select in menus |
| B / space | fire, and select in menus |
| Y | Doom's menu |
| X / Esc | back to the launcher |

Saving works with the touch controls alone: an empty save slot is prefilled
with `DOOMPERSON`, and fire also sends Enter, the only key the save-name prompt
accepts. Saves and settings live in `prefs/com.micropythonos.doom/`.

## Licenses

The engine is GPL v2 or later; the complete corresponding source is
<https://github.com/MicroPythonOS/MicroPythonOS/tree/main/c_mpos/doom>.
`DOOM1.WAD` is id Software's shareware episode, distributed unmodified and
free of charge. The app shows both notices (hold the menu touch area), and
[`doom_legal.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/apps/com.micropythonos.doom/doom_legal.py)
in the app carries the text.

## Notes for developers

- The package is flat (eight files; no directories inside the app) because
  every littlefs directory costs about 8 KiB of metadata.
- The engines are built with `./scripts/build_doom_app.sh`; see
  [Native C/C++ Apps](native-apps.md) for the core/native-module split.
- The installer's `"native"` selection and the install-time extent are
  described in [Publishing Apps](publishing-apps.md#native-engines-in-one-package).
