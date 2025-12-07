# SensorManager

MicroPythonOS provides a unified sensor framework called **SensorManager**, inspired by Android's SensorManager API. It provides easy access to motion sensors (accelerometer, gyroscope) and temperature sensors across different hardware platforms.

## Overview

SensorManager automatically detects available sensors on your device:

- **QMI8658 IMU** (Waveshare ESP32-S3-Touch-LCD-2)
- **WSEN_ISDS IMU** (Fri3d Camp 2024 Badge)
- **ESP32 MCU Temperature** (All ESP32 boards)

The framework handles:

✅ **Auto-detection** - Identifies which IMU is present
✅ **Unit normalization** - Returns standard SI units (m/s², deg/s, °C)
✅ **Persistent calibration** - Calibrate once, saved across reboots
✅ **Thread-safe** - Safe for concurrent access
✅ **Hardware-agnostic** - Apps work on all platforms without changes

## Sensor Types

```python
import mpos.sensor_manager as SensorManager

# Motion sensors
SensorManager.TYPE_ACCELEROMETER    # m/s² (meters per second squared)
SensorManager.TYPE_GYROSCOPE        # deg/s (degrees per second)

# Temperature sensors
SensorManager.TYPE_SOC_TEMPERATURE  # °C (MCU internal temperature)
SensorManager.TYPE_IMU_TEMPERATURE  # °C (IMU chip temperature)
```

## Quick Start

### Basic Usage

```python
from mpos.app.activity import Activity
import mpos.sensor_manager as SensorManager

class MyActivity(Activity):
    def onCreate(self):
        # Check if sensors are available
        if not SensorManager.is_available():
            print("No sensors available")
            return

        # Get sensors
        self.accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
        self.gyro = SensorManager.get_default_sensor(SensorManager.TYPE_GYROSCOPE)

        # Read data
        accel_data = SensorManager.read_sensor(self.accel)
        gyro_data = SensorManager.read_sensor(self.gyro)

        if accel_data:
            ax, ay, az = accel_data  # In m/s²
            print(f"Acceleration: X={ax:.2f}, Y={ay:.2f}, Z={az:.2f} m/s²")

        if gyro_data:
            gx, gy, gz = gyro_data  # In deg/s
            print(f"Gyroscope: X={gx:.2f}, Y={gy:.2f}, Z={gz:.2f} deg/s")
```

### Reading Temperature

```python
# Get MCU internal temperature (most stable)
temp_sensor = SensorManager.get_default_sensor(SensorManager.TYPE_SOC_TEMPERATURE)
temperature = SensorManager.read_sensor(temp_sensor)
print(f"MCU Temperature: {temperature:.1f}°C")

# Get IMU chip temperature
imu_temp_sensor = SensorManager.get_default_sensor(SensorManager.TYPE_IMU_TEMPERATURE)
imu_temperature = SensorManager.read_sensor(imu_temp_sensor)
if imu_temperature:
    print(f"IMU Temperature: {imu_temperature:.1f}°C")
```

## Tilt-Controlled Game Example

This example shows how to create a simple tilt-controlled ball game:

```python
from mpos.app.activity import Activity
import mpos.sensor_manager as SensorManager
import mpos.ui
import lvgl as lv
import time

class TiltBallActivity(Activity):
    def onCreate(self):
        # Create screen
        self.screen = lv.obj()

        # Check sensors
        if not SensorManager.is_available():
            label = lv.label(self.screen)
            label.set_text("No accelerometer available")
            label.center()
            self.setContentView(self.screen)
            return

        # Get accelerometer
        self.accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)

        # Create ball
        self.ball = lv.obj(self.screen)
        self.ball.set_size(20, 20)
        self.ball.set_style_radius(10, 0)  # Make it circular
        self.ball.set_style_bg_color(lv.color_hex(0xFF0000), 0)

        # Ball physics
        self.ball_x = 160.0  # Center X
        self.ball_y = 120.0  # Center Y
        self.ball_vx = 0.0   # Velocity X
        self.ball_vy = 0.0   # Velocity Y
        self.last_time = time.ticks_ms()

        self.setContentView(self.screen)

    def onResume(self, screen):
        # Start physics updates
        self.last_time = time.ticks_ms()
        mpos.ui.task_handler.add_event_cb(self.update_physics, 1)

    def onPause(self, screen):
        # Stop physics updates
        mpos.ui.task_handler.remove_event_cb(self.update_physics)

    def update_physics(self, a, b):
        # Calculate delta time
        current_time = time.ticks_ms()
        delta_time = time.ticks_diff(current_time, self.last_time) / 1000.0
        self.last_time = current_time

        # Read accelerometer (returns m/s²)
        accel = SensorManager.read_sensor(self.accel)
        if not accel:
            return

        ax, ay, az = accel

        # Apply acceleration to velocity (scale down for gameplay)
        # Tilt right (positive X) → ball moves right
        # Tilt forward (positive Y) → ball moves down (flip Y)
        self.ball_vx += (ax * 5.0) * delta_time  # Scale factor for gameplay
        self.ball_vy -= (ay * 5.0) * delta_time  # Negative to flip Y

        # Apply friction
        self.ball_vx *= 0.98
        self.ball_vy *= 0.98

        # Update position
        self.ball_x += self.ball_vx
        self.ball_y += self.ball_vy

        # Bounce off walls
        if self.ball_x < 10 or self.ball_x > 310:
            self.ball_vx *= -0.8  # Bounce with energy loss
            self.ball_x = max(10, min(310, self.ball_x))

        if self.ball_y < 10 or self.ball_y > 230:
            self.ball_vy *= -0.8
            self.ball_y = max(10, min(230, self.ball_y))

        # Update ball position
        self.ball.set_pos(int(self.ball_x) - 10, int(self.ball_y) - 10)
```

## Gesture Detection Example

Detect device shake and rotation:

```python
import mpos.sensor_manager as SensorManager
import math

class GestureDetector(Activity):
    def onCreate(self):
        self.accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
        self.gyro = SensorManager.get_default_sensor(SensorManager.TYPE_GYROSCOPE)

        # Shake detection
        self.shake_threshold = 15.0  # m/s²

        # Rotation detection
        self.rotation_threshold = 100.0  # deg/s

    def onResume(self, screen):
        mpos.ui.task_handler.add_event_cb(self.detect_gestures, 1)

    def detect_gestures(self, a, b):
        # Detect shake (sudden acceleration)
        accel = SensorManager.read_sensor(self.accel)
        if accel:
            ax, ay, az = accel
            magnitude = math.sqrt(ax*ax + ay*ay + az*az)
            gravity = 9.80665

            # Shake = acceleration magnitude significantly different from gravity
            if abs(magnitude - gravity) > self.shake_threshold:
                self.on_shake()

        # Detect rotation (spinning device)
        gyro = SensorManager.read_sensor(self.gyro)
        if gyro:
            gx, gy, gz = gyro
            rotation_speed = math.sqrt(gx*gx + gy*gy + gz*gz)

            if rotation_speed > self.rotation_threshold:
                self.on_rotate(gx, gy, gz)

    def on_shake(self):
        print("Device shaken!")
        # Trigger action (shuffle playlist, undo, etc.)

    def on_rotate(self, gx, gy, gz):
        print(f"Device rotating: {gx:.1f}, {gy:.1f}, {gz:.1f} deg/s")
        # Trigger action (rotate view, spin wheel, etc.)
```

## Calibration

Calibration removes sensor drift and improves accuracy. The device must be **stationary on a flat surface** during calibration.

### Using the Built-in Calibration Tool

The easiest way to calibrate your IMU is through the Settings app:

1. Open **Settings** → **IMU** → **Calibrate IMU**
2. Place your device on a flat, stable surface
3. Tap **Calibrate Now**
4. Keep the device still for ~2 seconds
5. Done! Calibration is saved automatically

The built-in tool performs stationarity checks and calibrates both the accelerometer and gyroscope with 100 samples each for optimal accuracy.

### Manual Calibration (Programmatic)

```python
class SettingsActivity(Activity):
    def calibrate_clicked(self, event):
        # Get sensors
        accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
        gyro = SensorManager.get_default_sensor(SensorManager.TYPE_GYROSCOPE)

        # Show instructions
        self.status_label.set_text("Place device flat and still...")
        wait_for_render()  # Let UI update

        # Calibrate accelerometer (100 samples)
        self.status_label.set_text("Calibrating accelerometer...")
        accel_offsets = SensorManager.calibrate_sensor(accel, samples=100)

        # Calibrate gyroscope
        self.status_label.set_text("Calibrating gyroscope...")
        gyro_offsets = SensorManager.calibrate_sensor(gyro, samples=100)

        # Done - calibration is automatically saved
        self.status_label.set_text(f"Calibration complete!\n"
                                    f"Accel: {accel_offsets}\n"
                                    f"Gyro: {gyro_offsets}")
```

### Persistent Calibration

Calibration data is automatically saved to `data/com.micropythonos.settings/sensors.json` and loaded on boot. You only need to calibrate once (unless the device is physically relocated or significantly re-oriented).

## List Available Sensors

```python
# Get all sensors
sensors = SensorManager.get_sensor_list()

for sensor in sensors:
    print(f"Name: {sensor.name}")
    print(f"Type: {sensor.type}")
    print(f"Vendor: {sensor.vendor}")
    print(f"Max Range: {sensor.max_range}")
    print(f"Resolution: {sensor.resolution}")
    print(f"Power: {sensor.power} mA")
    print("---")

# Example output on Waveshare ESP32-S3:
# Name: QMI8658 Accelerometer
# Type: 1
# Vendor: QST Corporation
# Max Range: ±8G (78.4 m/s²)
# Resolution: 0.0024 m/s²
# Power: 0.2 mA
# ---
# Name: QMI8658 Gyroscope
# Type: 4
# Vendor: QST Corporation
# Max Range: ±256 deg/s
# Resolution: 0.002 deg/s
# Power: 0.7 mA
# ---
# Name: QMI8658 Temperature
# Type: 14
# Vendor: QST Corporation
# Max Range: -40°C to +85°C
# Resolution: 0.004°C
# Power: 0 mA
# ---
# Name: ESP32 MCU Temperature
# Type: 15
# Vendor: Espressif
# Max Range: -40°C to +125°C
# Resolution: 0.5°C
# Power: 0 mA
```

## Performance Tips

### Polling Rate

IMU sensors can be read very quickly (~1-2ms per read), but polling every frame is unnecessary:

```python
# ❌ BAD: Poll every frame (60 Hz = 60 reads/sec)
def update_frame(self, a, b):
    accel = SensorManager.read_sensor(self.accel)  # Too frequent!

# ✅ GOOD: Poll every other frame (30 Hz)
def update_frame(self, a, b):
    self.frame_count += 1
    if self.frame_count % 2 == 0:
        accel = SensorManager.read_sensor(self.accel)

# ✅ BETTER: Use timer instead of frame updates
def onStart(self, screen):
    # Poll at 20 Hz (every 50ms)
    self.sensor_timer = lv.timer_create(self.read_sensors, 50, None)

def read_sensors(self, timer):
    accel = SensorManager.read_sensor(self.accel)
```

**Recommended rates:**
- **Games**: 20-30 Hz (responsive but not excessive)
- **UI feedback**: 10-15 Hz (smooth enough for tilt UI)
- **Background monitoring**: 1-5 Hz (screen rotation, pedometer)

### Thread Safety

SensorManager is thread-safe and can be read from multiple threads:

```python
import _thread
import mpos.apps

def background_monitoring():
    accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
    while True:
        accel_data = SensorManager.read_sensor(accel)  # Thread-safe
        # Process data...
        time.sleep(1)

# Start background thread
_thread.stack_size(mpos.apps.good_stack_size())
_thread.start_new_thread(background_monitoring, ())
```

## Platform Differences

### Hardware Support

| Platform | Accelerometer | Gyroscope | IMU Temp | MCU Temp |
|----------|---------------|-----------|----------|----------|
| Waveshare ESP32-S3 | ✅ QMI8658 | ✅ QMI8658 | ✅ QMI8658 | ✅ ESP32 |
| Fri3d 2024 Badge | ✅ WSEN_ISDS | ✅ WSEN_ISDS | ❌ | ✅ ESP32 |
| Desktop/Linux | ❌ | ❌ | ❌ | ❌ |

### Graceful Degradation

Always check if sensors are available:

```python
if SensorManager.is_available():
    # Use real sensor data
    accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
    data = SensorManager.read_sensor(accel)
else:
    # Fallback for desktop/testing
    data = (0.0, 0.0, 9.8)  # Simulate device at rest
```

## Unit Conversions

SensorManager returns standard SI units. Here are common conversions:

### Acceleration

```python
# SensorManager returns m/s²
accel = SensorManager.read_sensor(accel_sensor)
ax, ay, az = accel

# Convert to G-forces (1 G = 9.80665 m/s²)
ax_g = ax / 9.80665
ay_g = ay / 9.80665
az_g = az / 9.80665
print(f"Acceleration: {az_g:.2f} G")  # At rest, Z ≈ 1.0 G
```

### Gyroscope

```python
# SensorManager returns deg/s
gyro = SensorManager.read_sensor(gyro_sensor)
gx, gy, gz = gyro

# Convert to rad/s
import math
gx_rad = math.radians(gx)
gy_rad = math.radians(gy)
gz_rad = math.radians(gz)

# Convert to RPM (rotations per minute)
gz_rpm = gz / 6.0  # 360 deg/s = 60 RPM
```

## API Reference

### Functions

**`init(i2c_bus, address=0x6B)`**
Initialize SensorManager. Called automatically in board init files. Returns `True` if any sensors detected.

**`is_available()`**
Returns `True` if sensors are available.

**`get_sensor_list()`**
Returns list of all available `Sensor` objects.

**`get_default_sensor(sensor_type)`**
Returns the default `Sensor` for the given type, or `None` if not available.

**`read_sensor(sensor)`**
Reads sensor data. Returns `(x, y, z)` tuple for motion sensors, single value for temperature, or `None` on error.

**`calibrate_sensor(sensor, samples=100)`**
Calibrates the sensor (device must be stationary). Returns calibration offsets. Saves to SharedPreferences automatically.

### Sensor Object

Properties:

- `name` - Human-readable sensor name
- `type` - Sensor type constant
- `vendor` - Manufacturer name
- `version` - Driver version
- `max_range` - Maximum measurement range
- `resolution` - Measurement resolution
- `power` - Power consumption in mA

## Troubleshooting

### Sensor Returns None

```python
data = SensorManager.read_sensor(accel)
if data is None:
    # Possible causes:
    # 1. Sensor not available (check is_available())
    # 2. I2C communication error
    # 3. Sensor not initialized
    print("Sensor read failed")
```

### Inaccurate Readings

- **Calibrate the sensors** - Run calibration with device stationary
- **Check mounting** - Ensure device is flat during calibration
- **Wait for warmup** - Sensors stabilize after 1-2 seconds

### High Drift

- **Re-calibrate** - Temperature changes can cause drift
- **Check for interference** - Keep away from magnets, motors
- **Use filtered data** - Apply low-pass filter for smoother readings

```python
# Simple low-pass filter
class LowPassFilter:
    def __init__(self, alpha=0.1):
        self.alpha = alpha
        self.value = None

    def filter(self, new_value):
        if self.value is None:
            self.value = new_value
        else:
            self.value = self.alpha * new_value + (1 - self.alpha) * self.value
        return self.value

# Usage
accel_filter_x = LowPassFilter(alpha=0.2)
accel = SensorManager.read_sensor(accel_sensor)
ax, ay, az = accel
filtered_ax = accel_filter_x.filter(ax)
```

### ImportError: can't import name _CONSTANT

If you try to directly import driver constants, you'll see this error:

```python
from mpos.hardware.drivers.qmi8658 import _QMI8685_PARTID  # ERROR!
# ImportError: can't import name _QMI8685_PARTID
```

**Cause**: Driver constants are defined with MicroPython's `const()` function, which makes them compile-time constants. They're inlined during compilation and the names aren't available for import at runtime.

**Solution**: Use hardcoded values instead:

```python
from mpos.hardware.drivers.qmi8658 import QMI8658
# Define constants locally
_QMI8685_PARTID = 0x05
_REG_PARTID = 0x00
```

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [SharedPreferences](preferences.md) - Persist app data
- [System Components](../architecture/system-components.md) - OS architecture overview
