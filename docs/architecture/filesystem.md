# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **/apps/**: Directory for downloaded and installed apps.
  - **/apps/com.micropythonos.helloworld/**: Installation directory for HelloWorld App. See [Developing Apps](../apps/developing-apps.md).
- **/builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
  - **/builtin/apps/**: See [Built-in Apps](../apps/built-in-apps.md). 
  - **/builtin/res/mipmap-mdpi/default_icon_64x64.bin**: Default icon for apps without one.
- **/data/**: Storage for app data.
  - **/data/images/**: Images stored by apps (e.g., camera app).
  - **/data/com.example.app1/**: App-specific storage (e.g., `config.json`) for com.example.app1.

This structure ensures a clear separation between system resources, apps, and user data.
