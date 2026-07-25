# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **apps/**: Directory for downloaded and installed apps. Cleaned up when the app is uninstalled.
    - **com.micropythonos.helloworld/**: Installation directory for HelloWorld App. See [Creating Apps](../apps/creating-apps.md).
    - **org.yourdomain.yourapp/**: Installation directory for the "Your App" app, created by yourdomain.org
- **builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
    - **apps/**: See [Built-in Apps](../apps/built-in-apps.md).
    - **res/**: Resources (icons, fonts, etc.).
- **cache/**: Temporary files for each app. Deleted when the app is uninstalled, and may also be deleted without uninstalling the app to free up space.
- **data/**: Content files independent of specific apps (images, audio, recordings, etc.). Used by apps like Camera, ImageView, Sound Recorder, and Music Player.
- **lib/**: Libraries and frameworks
    - **mpos/**: MicroPythonOS libraries and frameworks
        - **ui/**: MicroPythonOS User Interface libraries and frameworks
- **prefs/**: App preferences and settings. Cleaned up when the app is uninstalled.
    - **com.micropythonos.helloworld/**: App-specific storage (e.g., `config.json`)
    - **com.micropythonos.settings/**: Storage used by the built-in Settings App
    - **com_micropythonos_nostr/**: Storage for the Nostr App

This structure ensures a clear separation between system resources, apps, and user data.

When creating packages, use `_` instead of `.` in names (e.g., `com_micropythonos_nostr`), since `.` has special meaning in Python module imports.
