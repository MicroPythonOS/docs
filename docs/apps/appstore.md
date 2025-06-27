# AppStore

The MicroPythonOS App Store allows users to download and install new apps to extend system functionality.

App discovery is currently done by downloading the app list from [apps.micropythonos.com](https://apps.micropythonos.com).

## Example Apps

- **Hello World**: A sample app demonstrating basic functionality.
- **Camera**: Captures images and scans QR codes.
- **Image Viewer**: Displays images stored in `/data/images/`.
- **IMU**: Visualize data from the Intertial Measurement Unit, also known as the accellerometer.

## Screenshots

<div class="grid">
  <figure>
    <img src="../../assets/images/mpos_appstore_camera.png" alt="Camera App Store" style="width:100%;max-width:320px;">
    <figcaption>Camera App in App Store</figcaption>
  </figure>
  <figure>
    <img src="../../assets/images/hello_world_install.png" alt="Hello World Install" style="width:100%;max-width:320px;">
    <figcaption>Hello World Installation</figcaption>
  </figure>
  <figure>
    <img src="../../assets/images/mpos_camera_qr_320x240.png" alt="Camera QR Code" style="width:100%;max-width:320px;">
    <figcaption>Camera QR Code Scanner</figcaption>
  </figure>
</div>

## Developing Apps

Apps are written in MicroPython and installed in `/apps/`. See [Filesystem Layout](../architecture/filesystem.md) for the app directory structure.
