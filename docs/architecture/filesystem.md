# Filesystem Layout

MicroPythonOS uses a structured filesystem to organize apps, data, and resources.

- **/apps/**: Directory for downloaded and installed apps.
  - **/apps/com.example.app1/**: Installation directory for an example app.
    - **MANIFEST.MF**: Metadata (e.g., app name, start script).
    - **mipmap-mdpi/**: Medium DPI images (e.g., `icon_64x64.bin` for a 64x64 pixel icon).
- **/builtin/**: Read-only filesystem compiled into the OS, mounted at boot by `main.py`.
  - **/builtin/apps/**: Built-in apps (e.g., `launcher`, `wificonf`, `appstore`, `osupdate`).
  - **/builtin/res/mipmap-mdpi/default_icon_64x64.bin**: Default icon for apps without one.
- **/data/**: Storage for app data.
  - **/data/images/**: Images stored by apps (e.g., camera app).
  - **/data/com.example.app1/**: App-specific storage (e.g., `config.json`).

This structure ensures a clear separation between system resources, apps, and user data.
