# InputManager

InputManager is a singleton framework that provides a unified API for managing input device interactions, including pointer/touch coordinates, focus management, and input device registration.

## Overview

InputManager centralizes all input-related operations in a single class with class methods. This provides:

- **Unified API** - Single class for all input management
- **Clean Namespace** - No scattered functions cluttering imports
- **Testable** - InputManager can be tested independently
- **Focus Control** - Emulate focus on specific UI objects
- **Pointer Access** - Get current touch/pointer coordinates
- **Device Registration** - Register and query available input devices by type

## Architecture

InputManager is implemented as a singleton using class variables and class methods:

```python
class InputManager:
    _registered_indevs = []  # List of registered input devices

    @classmethod
    def pointer_xy(cls):
        """Get current pointer/touch coordinates."""
        import lvgl as lv
        indev = lv.indev_active()
        if indev:
            p = lv.point_t()
            indev.get_point(p)
            return p.x, p.y
        return -1, -1

    @classmethod
    def has_pointer(cls):
        """Check if a pointer device is registered."""
        return cls.has_indev_type(lv.INDEV_TYPE.POINTER)
```

No instance creation is needed - all methods are class methods that operate on class variables.

## Usage

### Getting Pointer Coordinates

```python
from mpos import InputManager

x, y = InputManager.pointer_xy()
if x == -1 and y == -1:
    print("No active input device")
else:
    print(f"Pointer at ({x}, {y})")
```

### Checking for Pointer Devices

```python
from mpos import InputManager

if InputManager.has_pointer():
    print("Touch or pointer input is available")
```

### Managing Focus

```python
from mpos import InputManager
import lvgl as lv

focusgroup = lv.group_get_default()
if focusgroup:
    InputManager.emulate_focus_obj(focusgroup, my_button)
```

## API Reference

### Class Methods

#### `pointer_xy()`
Get current pointer/touch coordinates from the active input device.

**Returns:** tuple - (x, y) coordinates, or (-1, -1) if no active input device

#### `has_pointer()`
Check if any registered input device is a pointer/touch device.

**Returns:** bool - True if a pointer device is registered

#### `emulate_focus_obj(focusgroup, target)`
Emulate setting focus to a specific object in the focus group.

#### `register_indev(indev)`
Register an input device for later querying by type.

#### `list_indevs()`
Get list of all registered input devices.

#### `has_indev_type(indev_type)`
Check if any registered input device has the specified type.

## Related Frameworks

- **[DisplayMetrics](display-metrics.md)** - Display properties and pointer coordinates (uses InputManager)
- **[WidgetAnimator](widget-animator.md)** - UI animation framework

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
- [LVGL Documentation](https://docs.lvgl.io/)
