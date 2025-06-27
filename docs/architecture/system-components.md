# System Components

MicroPythonOS consists of several core components that initialize and manage the system.

- **boot.py**: Initializes hardware on ESP32 microcontrollers.
- **boot_unix.py**: Initializes hardware on Linux desktops (and potentially MacOS).
- **main.py**:
    - Sets up the user interface.
    - Provides helper functions for apps.
    - Launches the `launcher` app to start the system.

These components work together to bootstrap the OS and provide a foundation for apps. See [Filesystem Layout](filesystem.md) for where apps and data are stored.

Additionally, the following concepts are also used throughout the code:

- **Activity**
- **ActivityNavigator**
- **Intent**
- **SharedPreferences**
- **WidgetAnimator**
- **WiFiService**

