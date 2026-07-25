# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **apps/**: Directory for downloaded and installed apps.
    - **com.micropythonos.helloworld/**: Installation directory for HelloWorld App. See [Creating Apps](../apps/creating-apps.md).
- **builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
    - **apps/**: See [Built-in Apps](../apps/built-in-apps.md).
    - **res/**: Resources (icons, fonts, etc.).
- **lib/**: Libraries and frameworks
    - **mpos/**: MicroPythonOS libraries and frameworks
        - **ui/**: MicroPythonOS User Interface libraries and frameworks
- **prefs/**: Storage for app data.
    - **com.micropythonos.helloworld/**: App-specific storage (e.g., `config.json`)
    - **com.micropythonos.settings/**: Storage used by the built-in Settings App
    - **com.micropythonos.wifi/**: Storage used by the built-in WiFi App

This structure ensures a clear separation between system resources, apps, and user data.

When creating packages, use `_` instead of `.` in names (e.g., `com_micropythonos_nostr`), since `.` has special meaning in Python module imports.
