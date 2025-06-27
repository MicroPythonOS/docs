# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **/apps/**: Directory for downloaded and installed apps.
    - **com.micropythonos.helloworld/**: Installation directory for HelloWorld App. See [Developing Apps](../apps/developing-apps.md).
- **/builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
    - **apps/**: See [Built-in Apps](../apps/built-in-apps.md). 
    - **res/mipmap-mdpi/default_icon_64x64.png**: Default icon for apps without one.
- **lib/**: Libraries and frameworks
    - **mpos/**: MicroPythonOS libraries and frameworks
        - **ui/**: MicroPythonOS User Interface libraries and frameworks
- **/data/**: Storage for app data.
    - **com.micropythonos.helloworld/**: App-specific storage (e.g., `config.json`)
    - **com.micropythonos.settings/**: Storage used by the built-in Settings App
    - **com.micropythonos.wifi/**: Storage used by the built-in WiFi App
    - **images/**: Generic storage for images (use by e.g. the Camera App).

This structure ensures a clear separation between system resources, apps, and user data.
