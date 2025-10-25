# System Components

MicroPythonOS consists of several core components that initialize and manage the system.

- **boot.py**: Initializes hardware on the [Waveshare ESP32-S3-Touch-LCD-2](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-2)
- **boot_fri3d2024.py**: Initializes hardware on the [Fri3d Camp 2024 Badge](https://fri3d.be/badge/2024/)
- **boot_unix.py**: Initializes hardware on Linux and MacOS systems

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

