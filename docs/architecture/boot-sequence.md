# Boot sequence

MicroPythonOS consists of several core components that initialize and manage the system.

- **MicroPythonOS/lvgl_micropython/lib/micropython/ports/esp32/modules/_boot.py**: attempts to mount the internal storage partition to / and if it fails, formats it

- **internal_filesystem/main.py**: Hands off execution to /lib/mpos/main.py by importing it

- **/lib/mpos/main.py**:
    - Detects the hardware board
    - Initializes the filesystem driver
    - Mounts the freezefs into /builtin/
    - Loads the com.micropythonos.settings
    - Initializes the user interface.
    - Starts the WiFi autoconnect thread
    - Launches the `launcher` app that shows the icons
    - Starts the asyncio REPL
    - Marks the current boot as successful (cancel rollback)
    - Starts the TaskManager

See [Filesystem Layout](filesystem.md) for where apps and data are stored.

