# Built-in Apps

MicroPythonOS includes essential apps to bootstrap the system. The launcher and the system apps live in [`/builtin/apps/`](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem/builtin/apps); the Media Player is preinstalled into `/apps/` instead, so it can be updated through the App Store like any other app.

- **Launcher**: Starts apps and provides the home screen.
- **WiFi**: Configures WiFi connections.
- **AppStore**: Downloads and installs new apps.
- **OSUpdate**: Manages Over-The-Air (OTA) system updates.
- **Settings**: Configuration for MicroPythonOS.
- **File Manager**: Browses the filesystem and opens files with the appropriate viewer.
- **Media Player**: Plays MJPEG video (AVI with the sound inside, or raw `.mjpeg` plus a WAV sidecar) and WAV/MP3 audio or RTTTL ringtones from the SD card. See [Media Player](media-player.md).

## Screenshots

<div class="grid">
  <figure>
    <img src="../../assets/images/launcher.png" alt="Launcher" style="width:100%;max-width:320px;">
    <figcaption>Launcher</figcaption>
  </figure>
  <figure>
    <img src="../../assets/images/wifi.png" alt="WiFi" style="width:100%;max-width:320px;">
    <figcaption>WiFi</figcaption>
  </figure>
  <figure>
    <img src="../../assets/images/mpos_appstore.png" alt="AppStore" style="width:100%;max-width:320px;">
    <figcaption>App Store</figcaption>
  </figure>
  <figure>
    <img src="../../assets/images/osupdate_progress3.png" alt="OSUpdate" style="width:100%;max-width:320px;">
    <figcaption>OSUpdate</figcaption>
  </figure>
</div>

See [App Store](appstore.md) for additional apps.
