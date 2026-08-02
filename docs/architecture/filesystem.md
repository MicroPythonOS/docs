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
- **data/**: Content files independent of specific apps (images, audio, recordings, etc.). Used by apps like Camera, ImageView, Sound Recorder, and Music Player.
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

