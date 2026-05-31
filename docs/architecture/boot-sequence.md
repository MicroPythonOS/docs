# Boot sequence

MicroPythonOS consists of several core components that initialize and manage the system.

- **lvgl_micropython/lib/micropython/ports/esp32/modules/_boot.py**: attempts to mount the internal storage partition to / and if it fails, formats it

- **internal_filesystem/main.py**: Hands off execution to /lib/mpos/main.py by importing it

- **/lib/mpos/main.py**:
    - Detects the hardware board
    - Initializes the filesystem driver
    - Mounts the freezefs into /builtin/
    - Loads the com.micropythonos.settings
    - Initializes the user interface.
    - Launches the `launcher` app that shows the icons
    - Runs auto-start apps (`auto_start_app_early`, `auto_start_app`)
    - Starts **boot services** — all services that subscribe to the `boot_completed` intent are instantiated and receive `onStart()`:
        - `WifiBootService` — auto-connects WiFi in a background thread
        - `WebServerBootService` — starts the HTTP web server
        - `AIOReplService` — starts the asyncio REPL task
        - App-specific services (e.g., `OSUpdateService`)
    - Marks the current boot as successful (cancel rollback)
    - Starts the TaskManager (asyncio event loop)

See [Filesystem Layout](filesystem.md) for where apps and data are stored.

See [Service](../frameworks/service.md) for details on writing boot services.
