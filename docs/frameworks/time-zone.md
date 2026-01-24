# TimeZone

TimeZone is a utility class that provides timezone conversion and management functionality. It allows apps to convert between timezone names and POSIX timezone strings, and access the complete list of supported timezones.

## Overview

TimeZone centralizes timezone-related operations in a single class with static methods. This provides:

- **Timezone Conversion** - Convert timezone names to POSIX timezone strings
- **Timezone Listing** - Access all available timezones
- **Simple API** - Static methods for easy access without instantiation
- **Comprehensive Support** - Supports all major world timezones

## Architecture

TimeZone is implemented as a utility class using static methods:

```python
class TimeZone:
    @staticmethod
    def timezone_to_posix_time_zone(timezone):
        """Convert timezone name to POSIX timezone string."""
        # Implementation
    
    @staticmethod
    def get_timezones():
        """Get list of all available timezone names."""
        # Implementation
```

No instance creation is needed - all methods are static and operate independently.

## Usage

### Basic Timezone Conversion

```python
from mpos import TimeZone

# Convert timezone name to POSIX string
posix_tz = TimeZone.timezone_to_posix_time_zone('Europe/Berlin')
# Returns: 'CET-1CEST,M3.5.0,M10.5.0'

# Handle unknown timezone
posix_tz = TimeZone.timezone_to_posix_time_zone('Invalid/Zone')
# Returns: 'GMT0' (default fallback)
```

### Getting Available Timezones

```python
from mpos import TimeZone

# Get sorted list of all available timezones
timezones = TimeZone.get_timezones()
# Returns: ['Africa/Abidjan', 'Africa/Accra', ..., 'UTC', ...]

# Use in a timezone selector
for tz in timezones:
    print(tz)
```

### Handling None Values

```python
from mpos import TimeZone

# None timezone defaults to GMT0
posix_tz = TimeZone.timezone_to_posix_time_zone(None)
# Returns: 'GMT0'
```

## API Reference

### Static Methods

#### `timezone_to_posix_time_zone(timezone)`
Convert a timezone name to its POSIX timezone string.

**Parameters:**
- `timezone` (str or None): Timezone name (e.g., 'Africa/Abidjan', 'Europe/Berlin') or None

**Returns:** str - POSIX timezone string (e.g., 'GMT0', 'CET-1CEST,M3.5.0,M10.5.0'). Returns 'GMT0' if timezone is None or not found.

**Example:**
```python
posix_tz = TimeZone.timezone_to_posix_time_zone('Europe/Berlin')
# Returns: 'CET-1CEST,M3.5.0,M10.5.0'
```

#### `get_timezones()`
Get a sorted list of all available timezone names.

**Returns:** list - Sorted list of timezone names (e.g., ['Africa/Abidjan', 'Africa/Accra', ...])

**Example:**
```python
all_timezones = TimeZone.get_timezones()
print(f"Available timezones: {len(all_timezones)}")
```

## Practical Examples

### Settings Screen with Timezone Selector

```python
from mpos import Activity, TimeZone
import lvgl as lv

class SettingsActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Create dropdown with all timezones
        dropdown = lv.dropdown(screen)
        dropdown.set_options('\n'.join(TimeZone.get_timezones()))
        dropdown.set_width(300)
        
        self.setContentView(screen)
```

### Timezone Validation

```python
from mpos import TimeZone

def validate_timezone(tz_name):
    """Check if a timezone is valid."""
    available = TimeZone.get_timezones()
    return tz_name in available

# Usage
if validate_timezone('Europe/Berlin'):
    posix_tz = TimeZone.timezone_to_posix_time_zone('Europe/Berlin')
else:
    posix_tz = TimeZone.timezone_to_posix_time_zone(None)  # Fallback to GMT0
```

### Timezone Search

```python
from mpos import TimeZone

def find_timezones(region):
    """Find all timezones in a region."""
    all_tz = TimeZone.get_timezones()
    return [tz for tz in all_tz if tz.startswith(region)]

# Usage
european_timezones = find_timezones('Europe')
# Returns: ['Europe/Amsterdam', 'Europe/Andorra', ..., 'Europe/Zurich']
```

### Safe Timezone Conversion

```python
from mpos import TimeZone

def get_posix_timezone(tz_name):
    """Get POSIX timezone with validation."""
    available = TimeZone.get_timezones()
    
    if tz_name not in available:
        print(f"Unknown timezone: {tz_name}, using GMT0")
        return TimeZone.timezone_to_posix_time_zone(None)
    
    return TimeZone.timezone_to_posix_time_zone(tz_name)
```

## Implementation Details

### File Structure

```
mpos/
├── time_zone.py       # Core TimeZone class
├── time_zones.py      # TIME_ZONE_MAP constant
└── __init__.py        # Exports TimeZone
```

### Timezone Map

The `TIME_ZONE_MAP` is a dictionary mapping timezone names to POSIX timezone strings:

```python
TIME_ZONE_MAP = {
    'Africa/Abidjan': 'GMT0',
    'Africa/Accra': 'GMT0',
    'Europe/Berlin': 'CET-1CEST,M3.5.0,M10.5.0',
    'Europe/London': 'GMT0BST,M3.5.0/1,M10.5.0',
    # ... many more timezones
}
```

## Design Patterns

### Static Method Utility Class

TimeZone uses static methods for a simple, functional API:

```python
class TimeZone:
    @staticmethod
    def timezone_to_posix_time_zone(timezone):
        # No instance state needed
        return TIME_ZONE_MAP.get(timezone, "GMT0")
```

This pattern is ideal for stateless utility functions that don't require instance data.

## Related Frameworks

- **[DeviceInfo](device-info.md)** - Device hardware information
- **[BuildInfo](build-info.md)** - OS version and build information
- **[Preferences](preferences.md)** - Persistent user preferences (for storing timezone selection)

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
