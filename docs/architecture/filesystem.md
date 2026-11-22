# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **apps/**: Directory for downloaded and installed apps. Cleaned up when the app is uninstalled.
    - **com.micropythonos.helloworld/**: Installation directory for HelloWorld App. See [Creating Apps](../apps/creating-apps.md).
    - **org.yourdomain.yourapp/**: Installation directory for the "Your App" app, created by yourdomain.org
- **builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
    - **apps/**: See [Built-in Apps](../apps/built-in-apps.md).
    - **res/**: Resources (icons, fonts, etc.).
- **cache/**: Temporary files for each app. Deleted when the app is uninstalled, but may also be deleted to free up space when the filesystem is almost full, without the app being uninstalled.
    - **org.yourdomain.yourapp/**: Files cached by the `org.yourdomain.yourapp` app, such as large audio files or images that are nice to cache, but can be re-downloaded if needed.
- **data/**: Content files independent of specific apps (images, audio, recordings, etc.). Used by apps like Camera, ImageView, Sound Recorder, Media Player and Music Player.
- **lib/**: Libraries and frameworks
    - **mpos/**: MicroPythonOS libraries and frameworks
        - **ui/**: MicroPythonOS User Interface libraries and frameworks
- **prefs/**: App preferences and settings. Cleaned up when the app is uninstalled.
    - **com.micropythonos.helloworld/**: App-specific storage (e.g., `config.json`)
    - **com.micropythonos.settings/**: Storage used by the built-in Settings App
    - **com_micropythonos_nostr/**: Storage for the Nostr App
- **sdcard/**: Mount point for an optional (micro) SD card that may be inserted.

This structure ensures a clear separation between system resources, apps, and user data.

**Note**: When creating packages, use `_` instead of `.` in names (e.g., `com_micropythonos_nostr`), since `.` has special meaning in Python module imports.

## External storage layout

A similar structure should be used on additional storage devices, such as a micro SD card.

For example, **sdcard/cache/org.yourdomain.yourapp** should be used to cache files which may similarly be deleted to free up space when the filesystem is almost full.

Currently, **sdcard/apps** is not yet used to search for and install apps but it's likely that it will be, in a future release.

## Extents and defragmentation

The internal filesystem is **littlefs2e**, a fork of littlefs v2.11.3 that adds
extent files: an immutable, contiguous, optionally aligned run of blocks that a
payload is committed into. An extent is what lets a native engine run straight
out of flash (XIP) instead of being read through littlefs's CTZ skip-list, and
the same fork's defragmenter is what makes room for one by relocating the files
(and the directory metadata pairs) inside a candidate window.

The fork lives at
[MicroPythonOS/littlefs_with_extents_and_defrag](https://github.com/MicroPythonOS/littlefs_with_extents_and_defrag)
and is consumed as the `third_party/littlefs2e` git submodule;
`scripts/sync_littlefs2e.sh` prefixes its sources into the MicroPython tree at
build time. Its [EXTENTS.md](https://github.com/MicroPythonOS/littlefs_with_extents_and_defrag/blob/extents/EXTENTS.md)
has the on-disk format, the API and the defragmenter's limits (a split
directory's tail pair still cannot be moved).

Extents keep the standard littlefs disk version 2.1, but stock littlefs does not
know the extent tag: it treats an extent's blocks as free space and reports the
file as empty. Treat any image that may contain extents as read-only for stock
tools (littlefs-python, `mklittlefs`, an older firmware); each extent carries a
CRC so a consumer can detect that before using the payload.

