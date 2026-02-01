# Architecture Overview

MicroPythonOS is designed as a lightweight, app-centric operating system inspired by Android. Written entirely in MicroPython, it provides a minimal core with facilities for apps, making it easy to develop and deploy applications.

## Architecture

- Apps (AppStore, Launcher, OSUpdate, Settings, WiFi)
- Frameworks: MicroPython (AppManager, AudioManager, SensorManager) and C (QRDecoder)
- Drivers: Micropython (Display, TouchInput) and C (Camera)
- Kernel: FreeRTOS and Linux (low level bus drivers)

## Design Principles

- **Thin OS**: The core OS handles hardware initialization, multitasking, and UI, leaving most functionality to apps.
- **Everything is an App**: System features like WiFi configuration and updates are implemented as apps.
- **Developer-Friendly**: MicroPython simplifies app development with Python-based APIs.

## Key Components

- **Boot Process**: Initializes hardware and mounts the filesystem.
- **User Interface**: Android-like touch screen UI with gestures.
- **App Ecosystem**: Built-in apps and an App Store for extensibility.
- **OTA Updates**: Seamless system and app updates.

See [System Components](system-components.md) for details on key files and [Filesystem Layout](filesystem.md) for the directory structure.
