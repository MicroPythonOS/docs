# BatteryManager

The `BatteryManager` provides an Android-inspired API for querying battery and power information. It handles ADC readings, voltage calculations, percentage calculations, and intelligent caching with WiFi coordination on ESP32-S3.

## Overview

`BatteryManager` is a static class that provides direct access to battery voltage, charge percentage, and raw ADC values. It automatically handles:

- **ADC1/ADC2 Detection** - Identifies which ADC bank the pin uses
- **Adaptive Caching** - Different cache durations based on WiFi impact
- **WiFi Coordination** - Temporarily disables WiFi for ADC2 reads on ESP32-S3
- **Desktop Simulation** - Returns random values in typical range when ADC unavailable

## Import

```python
from mpos import BatteryManager
```

## Initialization

Before using battery readings, initialize the ADC with a conversion function:

```python
def adc_to_voltage(adc_value):
    """Convert raw ADC value (0-4095) to battery voltage in volts."""
    return adc_value * 0.001651 + 0.08709

BatteryManager.init_adc(pin=13, adc_to_voltage_func=adc_to_voltage)
```

### Parameters

- **pin** (int): GPIO pin number for battery voltage measurement
  - ADC1 pins: GPIO1-10 (no WiFi interference)
  - ADC2 pins: GPIO11-20 (requires WiFi disable on ESP32-S3)
- **adc_to_voltage_func** (callable): Function that converts raw ADC value (0-4095) to voltage in volts

### Important Notes

- **ADC2 on ESP32-S3**: WiFi will be temporarily disabled during readings to avoid conflicts
- **Desktop Mode**: If ADC is unavailable, returns simulated random values
- **Calibration**: The conversion function should be calibrated for your specific hardware

## API Methods

### read_battery_voltage()

Get the current battery voltage in volts.

```python
voltage = BatteryManager.read_battery_voltage()
print(f"Battery: {voltage:.2f}V")
```

**Parameters:**
- `force_refresh` (bool, optional): Bypass cache and force fresh reading. Default: `False`
- `raw_adc_value` (float, optional): Pre-computed raw ADC value (for testing). Default: `None`

**Returns:** float - Battery voltage in volts

**Raises:** RuntimeError - If ADC2 is used and WiFi is already busy

### get_battery_percentage()

Get the current battery charge as a percentage (0-100).

```python
percentage = BatteryManager.get_battery_percentage()
print(f"Battery: {percentage:.1f}%")
```

**Parameters:**
- `raw_adc_value` (float, optional): Pre-computed raw ADC value (for testing). Default: `None`

**Returns:** float - Battery percentage (0-100), clamped to valid range

### read_raw_adc()

Get the raw ADC value (0-4095) with averaging.

```python
raw = BatteryManager.read_raw_adc()
print(f"Raw ADC: {raw}")
```

**Parameters:**
- `force_refresh` (bool, optional): Bypass cache and force fresh reading. Default: `False`

**Returns:** float - Raw ADC value (average of 10 samples)

**Raises:** RuntimeError - If ADC2 is used and WiFi is already busy

### clear_cache()

Clear the cached battery reading to force a fresh read on the next call.

```python
BatteryManager.clear_cache()
voltage = BatteryManager.read_battery_voltage()  # Fresh read
```

## Caching Strategy

BatteryManager uses adaptive caching to balance accuracy with performance:

| ADC Bank | Cache Duration | Reason |
|----------|----------------|--------|
| ADC1 (GPIO1-10) | 30 seconds | No WiFi interference |
| ADC2 (GPIO11-20) | 10 minutes | Expensive WiFi disable/enable |

To override cache duration at runtime:

```python
import mpos.battery_manager
mpos.battery_manager.CACHE_DURATION_ADC1_MS = 60000  # 60 seconds
mpos.battery_manager.CACHE_DURATION_ADC2_MS = 300000  # 5 minutes
```

## WiFi Coordination (ESP32-S3)

On ESP32-S3, ADC2 (GPIO11-20) cannot be read while WiFi is active. BatteryManager automatically:

1. Checks if WiFi is busy (connecting/scanning)
2. Temporarily disables WiFi if needed
3. Reads the ADC value
4. Restores WiFi to its previous state

**Error Handling:**

```python
try:
    voltage = BatteryManager.read_battery_voltage(force_refresh=True)
except RuntimeError as e:
    print(f"Cannot read battery: {e}")
    # WiFi is busy, use cached value instead
    voltage = BatteryManager.read_battery_voltage()
```

## Constants

```python
from mpos.battery_manager import MIN_VOLTAGE, MAX_VOLTAGE

MIN_VOLTAGE = 3.15  # Minimum battery voltage (0%)
MAX_VOLTAGE = 4.15  # Maximum battery voltage (100%)
```

These constants define the voltage range used for percentage calculation. Adjust based on your battery chemistry.

## Example: Battery Status Display

```python
from mpos import BatteryManager
import time

# Initialize with calibration for your hardware
def adc_to_voltage(adc_value):
    return adc_value * 0.001651 + 0.08709

BatteryManager.init_adc(pin=13, adc_to_voltage_func=adc_to_voltage)

# Display battery status
while True:
    voltage = BatteryManager.read_battery_voltage()
    percentage = BatteryManager.get_battery_percentage()
    
    if percentage > 80:
        status = "🔋 Full"
    elif percentage > 50:
        status = "🔋 Good"
    elif percentage > 20:
        status = "⚠️ Low"
    else:
        status = "🔴 Critical"
    
    print(f"{status}: {voltage:.2f}V ({percentage:.0f}%)")
    time.sleep(5)
```

## Example: Real-time Battery Monitoring in UI

```python
from mpos import BatteryManager, Activity
import lvgl as lv

class BatteryMonitor(Activity):
    def onCreate(self):
        screen = lv.obj()
        self.battery_label = lv.label(screen)
        self.battery_label.set_text("Battery: --V")
        self.setContentView(screen)
    
    def onResume(self, screen):
        super().onResume(screen)
        
        def update_battery(timer):
            voltage = BatteryManager.read_battery_voltage()
            percentage = BatteryManager.get_battery_percentage()
            text = f"Battery: {voltage:.2f}V ({percentage:.0f}%)"
            self.update_ui_threadsafe_if_foreground(
                self.battery_label.set_text, text
            )
        
        # Update every 15 seconds
        lv.timer_create(update_battery, 15000, None)
```

## Board-Specific Configuration

### Fri3d 2024

```python
# GPIO13 (ADC2) with calibrated conversion function
def adc_to_voltage(adc_value):
    return adc_value * 0.001651 + 0.08709

BatteryManager.init_adc(13, adc_to_voltage)
```

### Waveshare ESP32-S3 Touch LCD

```python
# GPIO5 (ADC1) - no WiFi interference
def adc_to_voltage(adc_value):
    return adc_value * 0.001651 + 0.08709

BatteryManager.init_adc(5, adc_to_voltage)
```

### Desktop (Linux)

```python
# Simulated ADC (returns random values)
def adc_to_voltage(adc_value):
    return adc_value * 0.001651 + 0.08709

BatteryManager.init_adc(999, adc_to_voltage)
```

## Testing

BatteryManager includes comprehensive unit tests:

```bash
./MicroPythonOS/tests/unittest.sh MicroPythonOS/tests/test_battery_voltage.py
```

Tests cover:
- ADC1/ADC2 pin detection
- Caching behavior
- WiFi coordination
- Voltage calculations
- Desktop mode simulation

## Troubleshooting

### "RuntimeError: WifiService is already busy"

This occurs when trying to read ADC2 while WiFi is connecting or scanning. Solutions:

1. Use ADC1 pins (GPIO1-10) instead if possible
2. Wait for WiFi to finish connecting before reading
3. Use cached value: `BatteryManager.read_battery_voltage()` (without force_refresh)

### Inaccurate voltage readings

1. Verify your ADC conversion function is calibrated correctly
2. Check that the voltage divider is properly connected
3. Ensure ADC attenuation is set to 11dB (0-3.3V range)
4. Use `force_refresh=True` to bypass cache and get fresh reading

### Battery percentage always 0% or 100%

1. Check MIN_VOLTAGE and MAX_VOLTAGE constants match your battery
2. Verify ADC conversion function is correct
3. Test with known voltage: `BatteryManager.get_battery_percentage(raw_adc_value=2048)`

## See Also

- [`SensorManager`](sensor-manager.md) - For temperature and IMU sensors
- [`WifiService`](wifi-service.md) - For WiFi connectivity management
- [`DeviceInfo`](device-info.md) - For device information
