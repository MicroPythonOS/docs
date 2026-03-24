# SensorManager

MicroPythonOS provides a unified sensor framework called **SensorManager**, inspired by Android's SensorManager API. It provides easy access to motion sensors (accelerometer, gyroscope, magnetometer) and temperature sensors across different hardware platforms.

## Overview

SensorManager automatically detects available sensors on your device:

- **QMI8658 IMU** (Waveshare ESP32-S3-Touch-LCD-2)
- **WSEN_ISDS / LSM6DSO IMU** (Fri3d Camp 2024/2026)
- **MPU6886 IMU** (M5Stack Fire)
- **Linux IIO** (Desktop builds with IIO sensors)
- **ESP32 MCU Temperature** (All ESP32 boards)

The framework handles:

- **Auto-detection** - Identifies which IMU is present
- **Unit normalization** - Returns standard SI units (m/s², deg/s, °C, μT)
- **Persistent calibration** - Calibrate once, saved across reboots
- **Thread-safe** - Safe for concurrent access
- **Hardware-agnostic** - Apps work on all platforms without changes

## Sensor Types

```python
from mpos import SensorManager

# Motion sensors
SensorManager.TYPE_ACCELEROMETER    # m/s² (meters per second squared)
SensorManager.TYPE_GYROSCOPE        # deg/s (degrees per second)
SensorManager.TYPE_MAGNETIC_FIELD   # μT (micro teslas)

# Temperature sensors
SensorManager.TYPE_SOC_TEMPERATURE  # °C (MCU internal temperature)
SensorManager.TYPE_IMU_TEMPERATURE  # °C (IMU chip temperature)
```

## Quick Start

### Basic Usage

```python
from mpos import Activity, SensorManager

class MyActivity(Activity):
    def onCreate(self):
        if not SensorManager.is_available():
            print("No sensors available")
            return

        self.accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
        self.gyro = SensorManager.get_default_sensor(SensorManager.TYPE_GYROSCOPE)

        accel_data = SensorManager.read_sensor(self.accel)
        gyro_data = SensorManager.read_sensor(self.gyro)
```

### Reading Temperature

```python
# Get MCU internal temperature (most stable)
temp_sensor = SensorManager.get_default_sensor(SensorManager.TYPE_SOC_TEMPERATURE)
temperature = SensorManager.read_sensor(temp_sensor)

# Get IMU chip temperature
imu_temp_sensor = SensorManager.get_default_sensor(SensorManager.TYPE_IMU_TEMPERATURE)
imu_temperature = SensorManager.read_sensor(imu_temp_sensor)
```

### Desktop/Linux IIO Support

On desktop builds, SensorManager can use Linux IIO sensors:

```python
from mpos import SensorManager

SensorManager.init_iio()
mag = SensorManager.get_default_sensor(SensorManager.TYPE_MAGNETIC_FIELD)
print(SensorManager.read_sensor(mag))
```

## Calibration

Calibration removes sensor drift and improves accuracy. The device must be **stationary on a flat surface** during calibration.

### Manual Calibration (Programmatic)

```python
accel = SensorManager.get_default_sensor(SensorManager.TYPE_ACCELEROMETER)
gyro = SensorManager.get_default_sensor(SensorManager.TYPE_GYROSCOPE)

accel_offsets = SensorManager.calibrate_sensor(accel, samples=100)
gyro_offsets = SensorManager.calibrate_sensor(gyro, samples=100)
```

Calibration is saved to `imu_calibration.json` under `com.micropythonos.settings`.

## List Available Sensors

```python
sensors = SensorManager.get_sensor_list()
for sensor in sensors:
    print(sensor.name, sensor.type, sensor.vendor)
```

## API Reference

### Functions

**`init(i2c_bus, address=0x6B, mounted_position=FACING_SKY)`**

Initialize SensorManager. Returns `True` if initialization succeeded.

**`init_iio()`**

Initialize Linux IIO sensors (desktop builds).

**`is_available()`**

Returns `True` if sensors are available.

**`get_sensor_list()`**

Returns list of all available `Sensor` objects.

**`get_default_sensor(sensor_type)`**

Returns the default `Sensor` for the given type, or `None` if not available.

**`read_sensor(sensor)`**

Reads sensor data. Returns `(x, y, z)` tuple for motion sensors, single value for temperature, or `None` on error.

**`read_sensor_once(sensor)`**

One-shot sensor read with automatic initialization (used internally by calibration helpers).

**`calibrate_sensor(sensor, samples=100)`**

Calibrates the sensor (device must be stationary). Returns calibration offsets and saves them to disk.

**`check_calibration_quality(samples=50)`**

Return a dict containing mean/variance statistics and a quality score.

**`check_stationarity(samples=30, variance_threshold_accel=0.5, variance_threshold_gyro=5.0)`**

Return `True` if the device appears stationary based on variance thresholds.

### Sensor Object

Properties:

- `name` - Human-readable sensor name
- `type` - Sensor type constant
- `vendor` - Manufacturer name
- `version` - Driver version
- `max_range` - Maximum measurement range
- `resolution` - Measurement resolution
- `power` - Power consumption in mA

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [SharedPreferences](preferences.md) - Persist app data
