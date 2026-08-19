# Boot sequence

MicroPythonOS consists of several core components that initialize and manage the system.

- **lvgl_micropython/lib/micropython/ports/esp32/modules/_boot.py**: attempts to mount the internal storage partition to / and if it fails, formats it

- **internal_filesystem/main.py**: Hands off execution to /lib/mpos/main.py by importing it

- **/lib/mpos/main.py**:
    - Disables the Ctrl-C interrupt character for the duration of the boot
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

## Connecting over serial during boot

Tools that open the serial port send Ctrl-C to interrupt whatever is running — `mpremote` does this whenever it enters the raw REPL. Arriving mid-boot, that would abort the boot scripts and leave the device at a bare REPL shell with the OS only half-started, which is easily mistaken for a broken build: the prompt looks normal, but apps are missing and files placed in `/lib` appear to have no effect.

To prevent this, `/lib/mpos/main.py` disables the interrupt character (`micropython.kbd_intr(-1)`) as its first action, and the asyncio REPL manages it from then on. It is restored on the paths that fall back to the REPL shell, so a failed boot still gives you an interruptible prompt.

Practical consequences:

- Connecting while the device boots is safe; the command waits for the boot to finish instead of corrupting it.
- Wait for `Starting asyncio REPL...` on the serial console to know the boot completed.
- Ctrl-C can no longer break into a boot that hangs — use the board's reset button in that case.
