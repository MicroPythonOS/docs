# DeviceInfo

DeviceInfo is a singleton class that provides access to device hardware information. It stores and retrieves the hardware identifier that uniquely identifies the device.

## Overview

DeviceInfo centralizes device hardware information in a single class with class methods. This provides:

- **Hardware Identification** - Access the unique device/hardware identifier
- **Unified API** - Single class for all device information
- **Boot Integration** - Set during system initialization
- **Android-Inspired** - Follows Android's DeviceInfo pattern
- **Singleton Pattern** - Single source of truth for device identity

## Architecture

DeviceInfo is implemented as a singleton using class variables and class methods:

```python
class DeviceInfo:
    hardware_id = "missing-hardware-info"
    
    @classmethod
    def set_hardware_id(cls, device_id):
        """Set the device/hardware identifier (called during boot)."""
        cls.hardware_id = device_id
    
    @classmethod
    def get_hardware_id(cls):
        """Get the hardware identifier."""
        return cls.hardware_id
```

No instance creation is needed - all methods are class methods that operate on class variables.

## Usage

### Getting Device Hardware ID

```python
from mpos import DeviceInfo

# Get the hardware identifier
hw_id = DeviceInfo.get_hardware_id()
# Returns: e.g., "esp32s3-12345678" or "desktop-build"
```

### Checking Device Identity

```python
from mpos import DeviceInfo

# Check if running on specific hardware
hw_id = DeviceInfo.get_hardware_id()

if hw_id.startswith("esp32s3"):
    print("Running on ESP32-S3 hardware")
elif hw_id.startswith("desktop"):
    print("Running on desktop")
else:
    print(f"Unknown hardware: {hw_id}")
```

### Device-Specific Configuration

```python
from mpos import DeviceInfo

def get_device_config():
    """Get configuration based on device hardware."""
    hw_id = DeviceInfo.get_hardware_id()
    
    if "esp32s3" in hw_id:
        return {
            "max_memory": 8 * 1024 * 1024,  # 8MB PSRAM
            "has_camera": True,
            "has_battery": True
        }
    else:
        return {
            "max_memory": 512 * 1024 * 1024,  # 512MB
            "has_camera": False,
            "has_battery": False
        }
```

## API Reference

### Class Methods

#### `set_hardware_id(device_id)`
Set the device/hardware identifier. Called during system boot.

**Parameters:**
- `device_id` (str): The hardware identifier string (e.g., "esp32s3-12345678")

**Example:**
```python
DeviceInfo.set_hardware_id("esp32s3-abcdef123456")
```

#### `get_hardware_id()`
Get the hardware identifier.

**Returns:** str - The hardware identifier

**Example:**
```python
hw_id = DeviceInfo.get_hardware_id()
# Returns: "esp32s3-abcdef123456"
```

## Practical Examples

### Device Detection in Apps

```python
from mpos import Activity, DeviceInfo
import lvgl as lv

class MyApp(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Show device info
        hw_id = DeviceInfo.get_hardware_id()
        label = lv.label(screen)
        label.set_text(f"Device: {hw_id}")
        
        self.setContentView(screen)
```

### Hardware-Specific Feature Detection

```python
from mpos import DeviceInfo

class FeatureDetector:
    @staticmethod
    def has_camera():
        """Check if device has camera."""
        hw_id = DeviceInfo.get_hardware_id()
        return "esp32s3" in hw_id
    
    @staticmethod
    def has_battery():
        """Check if device has battery."""
        hw_id = DeviceInfo.get_hardware_id()
        return "esp32s3" in hw_id
    
    @staticmethod
    def is_desktop():
        """Check if running on desktop."""
        hw_id = DeviceInfo.get_hardware_id()
        return "desktop" in hw_id

# Usage
if FeatureDetector.has_camera():
    # Enable camera features
    pass
```

### Device Logging

```python
from mpos import DeviceInfo
import logging

def setup_logging():
    """Setup logging with device information."""
    hw_id = DeviceInfo.get_hardware_id()
    logger = logging.getLogger(__name__)
    
    # Include device ID in log messages
    logger.info(f"App started on device: {hw_id}")
    
    return logger
```

### Device-Specific Debugging

```python
from mpos import DeviceInfo

def debug_info():
    """Print device debug information."""
    hw_id = DeviceInfo.get_hardware_id()
    
    print(f"Hardware ID: {hw_id}")
    print(f"Device Type: {'ESP32-S3' if 'esp32s3' in hw_id else 'Desktop'}")
    
    # Can be used for conditional debugging
    if "esp32s3" in hw_id:
        print("Running on embedded hardware - memory constrained")
    else:
        print("Running on desktop - full resources available")
```

### Device Telemetry

```python
from mpos import DeviceInfo

class Telemetry:
    @staticmethod
    def get_device_context():
        """Get device context for telemetry."""
        hw_id = DeviceInfo.get_hardware_id()
        
        return {
            "device_id": hw_id,
            "device_type": "esp32s3" if "esp32s3" in hw_id else "desktop",
            "timestamp": time.time()
        }
```

## Implementation Details

### File Structure

```
mpos/
├── device_info.py     # Core DeviceInfo class
└── __init__.py        # Exports DeviceInfo
```

### Initialization Flow

1. **System Boot** - MicroPythonOS initializes
2. **Hardware Detection** - System detects hardware type
3. **Set Device ID** - Calls `DeviceInfo.set_hardware_id()` with detected hardware ID
4. **Ready to Use** - Apps can now call `DeviceInfo.get_hardware_id()`

```python
# During system initialization
hw_id = detect_hardware()  # e.g., "esp32s3-12345678"
DeviceInfo.set_hardware_id(hw_id)
```

## Design Patterns

### Singleton Pattern

DeviceInfo uses the singleton pattern with class variables and class methods:

```python
class DeviceInfo:
    hardware_id = "missing-hardware-info"  # Shared across all "instances"
    
    @classmethod
    def get_hardware_id(cls):
        return cls.hardware_id
```

This ensures a single source of truth for device identity.

### Class Method Delegation

All methods are class methods, so no instance is needed:

```python
# No need for DeviceInfo() or DeviceInfo.get()
hw_id = DeviceInfo.get_hardware_id()  # Direct class method call
```

## Related Frameworks

- **[BuildInfo](build-info.md)** - OS version and build information
- **[TimeZone](time-zone.md)** - Timezone conversion and management
- **[DisplayMetrics](display-metrics.md)** - Display properties and metrics

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
