# BuildInfo

BuildInfo is a singleton class that provides access to OS version and build information. It stores the release version and API level (SDK version) that identifies the MicroPythonOS build.

## Overview

BuildInfo centralizes OS version and build information in a single class with nested version attributes. This provides:

- **Version Information** - Access the human-readable release version
- **API Level** - Get the SDK API level for compatibility checks
- **Build Identification** - Identify the exact OS build
- **Android-Inspired** - Follows Android's Build.VERSION pattern
- **Singleton Pattern** - Single source of truth for OS version

## Architecture

BuildInfo is implemented as a singleton with a nested `version` class containing version attributes:

```python
class BuildInfo:
    class version:
        """Version information."""
        release = "0.7.0"  # Human-readable version
        sdk_int = 0        # API level
```

All version information is accessed through class attributes - no instance creation is needed.

## Usage

### Getting OS Version

```python
from mpos import BuildInfo

# Get human-readable version
version = BuildInfo.version.release
# Returns: "0.7.0"

# Get API level
api_level = BuildInfo.version.sdk_int
# Returns: 0
```

### Version Checking

```python
from mpos import BuildInfo

# Check if running specific version
if BuildInfo.version.release == "0.7.0":
    print("Running MicroPythonOS 0.7.0")

# Check API level
if BuildInfo.version.sdk_int >= 1:
    print("API level 1 or higher")
```

### Version Comparison

```python
from mpos import BuildInfo

def parse_version(version_str):
    """Parse version string into tuple."""
    return tuple(map(int, version_str.split('.')))

# Compare versions
current = parse_version(BuildInfo.version.release)
required = parse_version("0.7.0")

if current >= required:
    print("OS version meets requirements")
else:
    print("OS version too old")
```

## API Reference

### Class Attributes

#### `BuildInfo.version.release`
Human-readable OS version string.

**Type:** str

**Example:**
```python
version = BuildInfo.version.release
# Returns: "0.7.0"
```

#### `BuildInfo.version.sdk_int`
API level (SDK version) as an integer.

**Type:** int

**Example:**
```python
api_level = BuildInfo.version.sdk_int
# Returns: 0
```

## Practical Examples

### Version Display in About Screen

```python
from mpos import Activity, BuildInfo
import lvgl as lv

class AboutActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Display version information
        version_label = lv.label(screen)
        version_label.set_text(f"Version: {BuildInfo.version.release}")
        
        api_label = lv.label(screen)
        api_label.set_text(f"API Level: {BuildInfo.version.sdk_int}")
        
        self.setContentView(screen)
```

### Feature Availability Based on API Level

```python
from mpos import BuildInfo

class FeatureManager:
    @staticmethod
    def supports_feature(feature_name):
        """Check if feature is available in current API level."""
        api_level = BuildInfo.version.sdk_int
        
        features = {
            "camera": 1,
            "wifi": 0,
            "bluetooth": 2,
            "nfc": 3
        }
        
        required_api = features.get(feature_name, float('inf'))
        return api_level >= required_api

# Usage
if FeatureManager.supports_feature("camera"):
    # Enable camera features
    pass
```

### Version-Specific Behavior

```python
from mpos import BuildInfo

def get_ui_style():
    """Get UI style based on OS version."""
    version = BuildInfo.version.release
    
    if version.startswith("0.7"):
        return "modern"
    elif version.startswith("0.6"):
        return "classic"
    else:
        return "legacy"
```

### Compatibility Checking

```python
from mpos import BuildInfo

class AppCompatibility:
    MIN_VERSION = "0.7.0"
    MIN_API_LEVEL = 0
    
    @classmethod
    def is_compatible(cls):
        """Check if app is compatible with current OS."""
        version = BuildInfo.version.release
        api_level = BuildInfo.version.sdk_int
        
        # Check API level
        if api_level < cls.MIN_API_LEVEL:
            return False
        
        # Check version
        current = tuple(map(int, version.split('.')))
        required = tuple(map(int, cls.MIN_VERSION.split('.')))
        
        return current >= required

# Usage
if AppCompatibility.is_compatible():
    print("App is compatible with this OS")
else:
    print("App requires newer OS version")
```

### Build Information Logging

```python
from mpos import BuildInfo
import logging

def setup_logging():
    """Setup logging with build information."""
    logger = logging.getLogger(__name__)
    
    # Log build information
    logger.info(f"MicroPythonOS {BuildInfo.version.release} (API {BuildInfo.version.sdk_int})")
    
    return logger
```

### Version-Specific Workarounds

```python
from mpos import BuildInfo

def apply_workarounds():
    """Apply version-specific workarounds."""
    version = BuildInfo.version.release
    
    if version == "0.7.0":
        # Workaround for known issue in 0.7.0
        import gc
        gc.collect()
    
    if BuildInfo.version.sdk_int < 1:
        # Workaround for older API
        print("Using legacy API compatibility mode")
```

### Telemetry with Version Info

```python
from mpos import BuildInfo
import time

class Telemetry:
    @staticmethod
    def get_system_context():
        """Get system context for telemetry."""
        return {
            "os_version": BuildInfo.version.release,
            "api_level": BuildInfo.version.sdk_int,
            "timestamp": time.time()
        }
```

## Implementation Details

### File Structure

```
mpos/
├── build_info.py      # Core BuildInfo class
└── __init__.py        # Exports BuildInfo
```

### Version Attributes

The `BuildInfo.version` class contains:

```python
class BuildInfo:
    class version:
        release = "0.7.0"  # Human-readable version: "major.minor.patch"
        sdk_int = 0        # API level: integer for compatibility checks
```

### Version Numbering

- **release**: Semantic versioning (e.g., "0.7.0", "1.0.0")
  - Major: Breaking changes
  - Minor: New features
  - Patch: Bug fixes

- **sdk_int**: API level for feature availability
  - Incremented when new APIs are added
  - Used for compatibility checks

## Design Patterns

### Nested Class Pattern

BuildInfo uses a nested `version` class to organize version-related attributes:

```python
class BuildInfo:
    class version:
        release = "0.7.0"
        sdk_int = 0
```

This pattern groups related information and provides a clear namespace.

### Class Attribute Access

All attributes are class attributes, accessed directly without instantiation:

```python
# No need for BuildInfo() or BuildInfo.get()
version = BuildInfo.version.release  # Direct class attribute access
```

## Related Frameworks

- **[DeviceInfo](device-info.md)** - Device hardware information
- **[TimeZone](time-zone.md)** - Timezone conversion and management
- **[DisplayMetrics](display-metrics.md)** - Display properties and metrics

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
