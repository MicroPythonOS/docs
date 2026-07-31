# CameraManager

MicroPythonOS provides a centralized camera framework called **CameraManager**, inspired by Android's Camera2 API. It manages camera devices through a registry of `Camera` descriptors that encapsulate hardware-specific init, capture, deinit, and settings functions.

## Overview

CameraManager provides:

- **Camera registry** - Board init files register camera devices with their driver functions
- **Lens facing constants** - `LENS_FACING_BACK`, `LENS_FACING_FRONT`, `LENS_FACING_EXTERNAL`
- **Resolution mapping** - Map pixel dimensions to ESP32 `FrameSize` enum
- **OV camera settings** - Apply brightness, contrast, saturation, white balance, exposure, etc. from preferences
- **Singleton pattern** - Classmethod delegation: `CameraManager.add_camera(...)`, `CameraManager.get_cameras()`
- **Hardware-agnostic** - Apps use the same API across all platforms

## Quick Start

### Registering a Camera

Register cameras once at startup in your board init file:

```python
from mpos import CameraManager

# ESP32 with OV5640
CameraManager.add_camera(CameraManager.Camera(
    lens_facing=CameraManager.CameraCharacteristics.LENS_FACING_BACK,
    name="OV5640",
    vendor="OmniVision",
    init=my_ov5640_init_function,
    deinit=my_ov5640_deinit_function,
    capture=my_ov5640_capture_function,
    apply_settings=my_ov5640_apply_settings,
    rotation_degrees=90,
))
```

### Using Cameras in Apps

```python
from mpos import Activity, CameraManager

class MyCameraActivity(Activity):
    def onCreate(self):
        if not CameraManager.has_camera():
            print("No camera available")
            return

        self.cam = CameraManager.get_cameras()[0]
        self.cam_obj = self.cam.init(320, 240, "RGB565")

    def take_photo(self):
        image_data = self.cam.capture(self.cam_obj)
        # … process image_data …

    def onDestroy(self):
        if self.cam_obj:
            self.cam.deinit(self.cam_obj)
```

### Checking Availability

```python
from mpos import CameraManager

if CameraManager.has_camera():
    print(CameraManager.get_camera_count(), "camera(s) available")
else:
    print("No camera on this device")
```

## Desktop / V4L2

On Linux desktop builds, the board init file registers a V4L2 camera:

```python
CameraManager.add_camera(CameraManager.Camera(
    lens_facing=CameraManager.CameraCharacteristics.LENS_FACING_BACK,
    name="V4L2 Camera",
    vendor="Linux",
    init=cam_init,
    deinit=cam_deinit,
    capture=cam_capture,
    apply_settings=cam_apply_settings,
))
```

## Applying OV Camera Settings

The static helper `ov_apply_camera_settings` reads from a `SharedPreferences` object and applies all supported settings to an OV series camera:

```python
from mpos import CameraManager
from mpos import SharedPreferences

prefs = SharedPreferences("com.micropythonos.camera")
prefs.edit()
    .put_int("brightness", 0)
    .put_int("contrast", 1)
    .put_int("saturation", 0)
    .put_bool("hmirror", False)
    .put_bool("vflip", False)
    .put_bool("whitebal", True)
    .put_int("wb_mode", 0)
    .apply()

# After init:
CameraManager.ov_apply_camera_settings(cam_obj, prefs)
```

### Supported Settings

| Preference key | Type | Description |
|----------------|------|-------------|
| `brightness` | `int` | -2 to 2 |
| `contrast` | `int` | -2 to 2 |
| `saturation` | `int` | -2 to 2 |
| `hmirror` | `bool` | Horizontal mirror |
| `vflip` | `bool` | Vertical flip |
| `special_effect` | `int` | Effect index |
| `exposure_ctrl` | `bool` | Auto exposure control master switch |
| `aec_value` | `int` | Manual exposure value |
| `ae_level` | `int` | Auto exposure level |
| `aec2` | `bool` | AEC2 mode |
| `gain_ctrl` | `bool` | Auto gain control master switch |
| `agc_gain` | `int` | Manual gain value |
| `gainceiling` | `int` | Gain ceiling |
| `whitebal` | `bool` | Auto white balance master switch |
| `wb_mode` | `int` | White balance mode |
| `awb_gain` | `bool` | AWB gain |
| `sharpness` | `int` | Sharpness (OV5640+) |
| `denoise` | `int` | Denoise level (OV5640+) |
| `colorbar` | `bool` | Color bar test pattern |
| `dcw` | `bool` | Digital crop window |
| `bpc` | `bool` | Black pixel correction |
| `wpc` | `bool` | White pixel correction |
| `raw_gma` | `bool` | Raw gamma correction |
| `lenc` | `bool` | Lens correction |

## Resolution Mapping

Map pixel dimensions to ESP32 `FrameSize` enum values:

```python
from mpos import CameraManager

framesize = CameraManager.resolution_to_framesize(320, 240)
# Returns FrameSize.QVGA
```

Supported resolutions: `(96,96)` through `(1920,1080)`. Falls back to `R240X240` for unknown sizes. Returns `None` if the `camera` module is unavailable (e.g., desktop builds).

## API Reference

### Camera Class

Represents a camera device. Created via `CameraManager.Camera(...)`.

#### `Camera(lens_facing, name=None, vendor=None, version=None, init=None, deinit=None, capture=None, apply_settings=None, rotation_degrees=0)`

- **Parameters:**
  - `lens_facing` — `CameraCharacteristics.LENS_FACING_BACK`, `_FRONT`, or `_EXTERNAL`
  - `name` — Human-readable name (e.g., `"OV5640"`)
  - `vendor` — Manufacturer name
  - `version` — Driver version (default `1`)
  - `init` — `(width, height, colormode) -> camera_object`
  - `deinit` — `(camera_object) -> None`
  - `capture` — `(camera_object, colormode=None) -> image_data`
  - `apply_settings` — `(camera_object, prefs) -> None`
  - `rotation_degrees` — Clockwise rotation of the camera sensor

#### `camera.init(width, height, colormode)`

Initialize the camera hardware. Delegates to the registered `init` function.

- **Returns:** Camera object (hardware-specific handle)

#### `camera.deinit(cam_obj=None)`

Release camera hardware. Delegates to the registered `deinit` function.

#### `camera.capture(cam_obj, colormode=None)`

Capture a frame. Delegates to the registered `capture` function.

- **Returns:** Image data bytes

#### `camera.apply_settings(cam_obj, prefs)`

Apply settings from a `SharedPreferences` object. Delegates to the registered `apply_settings` function.

#### `camera.get_rotation_degrees()`

Returns `rotation_degrees` set at construction time.

### CameraCharacteristics

Constants matching Android Camera2 API:

- `LENS_FACING_BACK = 0` — Back-facing camera (primary)
- `LENS_FACING_FRONT = 1` — Front-facing camera (selfie)
- `LENS_FACING_EXTERNAL = 2` — External USB camera

### CameraManager Class

Singleton with classmethod delegation. All methods below are callable as classmethods (`CameraManager.get_cameras()`, etc.).

#### `CameraManager.init()`

Initialize CameraManager. Called automatically on module import.

- **Returns:** `True`

#### `CameraManager.is_available()`

Check if CameraManager is initialized.

- **Returns:** `bool`

#### `CameraManager.add_camera(camera)`

Register a `Camera` object.

- **Returns:** `bool`

#### `CameraManager.get_cameras()`

List all registered cameras.

- **Returns:** `list` of `Camera` objects (copy)

#### `CameraManager.get_camera_by_facing(lens_facing)`

Find first camera with the specified lens facing.

- **Returns:** `Camera` or `None`

#### `CameraManager.has_camera()`

Check if any camera is registered.

- **Returns:** `bool`

#### `CameraManager.get_camera_count()`

Number of registered cameras.

- **Returns:** `int`

#### `CameraManager.resolution_to_framesize(width, height)` *(static)*

Map pixel dimensions to ESP32 `FrameSize` enum.

- **Returns:** `FrameSize` enum value, or `None` if `camera` module unavailable

#### `CameraManager.ov_apply_camera_settings(cam, prefs)` *(static)*

Apply OV camera settings from a `SharedPreferences` object. Safe to call on non-OV cameras — unsupported settings are silently skipped.

- **Parameters:**
  - `cam` — Camera object returned by `camera.init()`
  - `prefs` — `SharedPreferences` instance

## See Also

- [Creating Apps](../apps/creating-apps.md)
- [SharedPreferences](preferences.md)
- [DeviceInfo](device-info.md) — `FeatureDetector.has_camera()`
- [BuildInfo](build-info.md) — `"camera"` feature flag
