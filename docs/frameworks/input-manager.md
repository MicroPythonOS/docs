# InputManager

InputManager is a singleton framework that provides a unified API for managing input device interactions, including pointer/touch coordinates, focus management, and input device registration.

## Overview

InputManager centralizes all input-related operations in a single class with class methods. This provides:

- **Unified API** - Single class for all input management
- **Clean Namespace** - No scattered functions cluttering imports
- **Testable** - InputManager can be tested independently
- **Pointer Access** - Get current touch/pointer coordinates
- **Device Registration** - Register and query available input devices by type
- **Navigation Gating** - Disable system-level back and drawer-open actions globally
- **Touch Feedback** - Attach haptic/touch-feedback callbacks to pointer devices

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
import lvgl as lv

focusgroup = lv.group_get_default()
if focusgroup:
    lv.group_focus_obj(my_button)
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
**Deprecated.** Compatibility shim. Use `lv.group_focus_obj(target)` directly.

#### `register_indev(indev)`
Register an input device for later querying by type. Called by board initialization code.

#### `unregister_indev(indev)`
Unregister an input device. Disables the device (`indev.enable(False)`) and removes it from the registry.

#### `list_indevs()`
Get list of all registered input devices.

#### `has_indev_type(indev_type)`
Check if any registered input device has the specified type (e.g., `lv.INDEV_TYPE.KEYPAD`, `lv.INDEV_TYPE.POINTER`).

#### `has_haptic_feedback()`
Check whether touch feedback (haptic) has been set up on pointer devices.

**Returns:** bool

#### `set_touch_feedback_cb(cb)`
Attach a callback `cb(event)` to every registered pointer input device on `LV_EVENT.CLICKED`. LVGL sends `CLICKED` on touch release only when the press did not scroll a scrollable parent, so the callback fires on taps but not on swipes. Call once at boot; calling again re-registers the callback.

## Navigation Gating

Apps that need full control over the back and menu buttons can disable system-level navigation actions globally via InputManager:

```python
from mpos.ui.input_manager import InputManager
import mpos

InputManager.set_back_screen_disabled(True)
InputManager.set_drawer_open_disabled(True)

if not InputManager.is_back_screen_disabled():
    InputManager.set_back_screen_disabled(True)
```

The same functions are also available as top-level `mpos` imports:

```python
import mpos
mpos.set_back_screen_disabled(True)
mpos.set_drawer_open_disabled(True)
```

- `close_drawer()` still works when drawer opening is disabled — you can close an already-open drawer.
- `finish_current_activity()` (called directly) is not gated, only `back_screen()`.
- All paths — hardware key handlers, gesture navigation, and programmatic calls — funnel through `back_screen()` and `open_drawer()`/`toggle_drawer()`, so a single call gates every trigger.
- When disabled, the back-screen and drawer-open actions invoke optional callbacks passed to `set_back_screen_disabled(True, cb=...)` and `set_drawer_open_disabled(True, cb=...)`. The callbacks receive no arguments. If no callback is set, the action is silently consumed.

### Navigation Gating API

#### `set_back_screen_disabled(disabled, cb=None)`
Disable or enable the back-screen navigation action. When `disabled=True`, back-screen actions are consumed (optionally invoking `cb`).

#### `is_back_screen_disabled()`
**Returns:** bool — True if back-screen navigation is currently disabled.

#### `set_drawer_open_disabled(disabled, cb=None)`
Disable or enable the top-menu drawer open action. When `disabled=True`, drawer-open actions are consumed (optionally invoking `cb`).

#### `is_drawer_open_disabled()`
**Returns:** bool — True if drawer opening is currently disabled.

## Related Frameworks

- **[DisplayMetrics](display-metrics.md)** - Display properties and pointer coordinates (uses InputManager)
- **[WidgetAnimator](widget-animator.md)** - UI animation framework

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
- [LVGL Documentation](https://docs.lvgl.io/)
