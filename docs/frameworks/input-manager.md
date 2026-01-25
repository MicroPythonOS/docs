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
    def emulate_focus_obj(cls, focusgroup, target):
        """Emulate setting focus to a specific object."""
        # ... implementation
    
    @classmethod
    def register_indev(cls, indev):
        """Register an input device for later querying."""
        if indev and indev not in cls._registered_indevs:
            cls._registered_indevs.append(indev)
    
    @classmethod
    def list_indevs(cls):
        """Get list of all registered input devices."""
        return cls._registered_indevs
    
    @classmethod
    def has_indev_type(cls, indev_type):
        """Check if any registered input device has the specified type."""
        for indev in cls._registered_indevs:
            if indev.get_type() == indev_type:
                return True
        return False
```

No instance creation is needed - all methods are class methods that operate on class variables.

## Usage

### Getting Pointer Coordinates

```python
from mpos import InputManager

# Get current pointer/touch position
x, y = InputManager.pointer_xy()
if x == -1 and y == -1:
    print("No active input device")
else:
    print(f"Pointer at ({x}, {y})")
```

### Managing Focus

```python
from mpos import InputManager
import lvgl as lv

# Get the default focus group
focusgroup = lv.group_get_default()

# Set focus to a specific button
if focusgroup:
    InputManager.emulate_focus_obj(focusgroup, my_button)
```

## API Reference

### Class Methods

#### `pointer_xy()`
Get current pointer/touch coordinates from the active input device.

**Returns:** tuple - (x, y) coordinates, or (-1, -1) if no active input device

**Example:**
```python
x, y = InputManager.pointer_xy()
if x != -1:
    print(f"Touch at ({x}, {y})")
```

#### `emulate_focus_obj(focusgroup, target)`
Emulate setting focus to a specific object in the focus group.

This method is needed because LVGL doesn't have a direct `set_focus()` method. It cycles through the focus group until it finds the target object.

**Parameters:**
- `focusgroup` (lv.group): The focus group to operate on
- `target` (lv.obj): The object to focus on

**Returns:** None

**Example:**
```python
focusgroup = lv.group_get_default()
if focusgroup:
    InputManager.emulate_focus_obj(focusgroup, my_button)
```

#### `register_indev(indev)`
Register an input device for later querying by type.

This method is called by board initialization code to register available input devices. Apps can then query these devices without hardcoding hardware IDs.

**Parameters:**
- `indev` (lv.indev): The LVGL input device to register

**Returns:** None

**Example:**
```python
# In board initialization (e.g., fri3d_2024.py)
indev = lv.indev_create()
indev.set_type(lv.INDEV_TYPE.KEYPAD)
indev.set_read_cb(keypad_read_cb)
InputManager.register_indev(indev)
```

#### `list_indevs()`
Get list of all registered input devices.

**Returns:** list - List of registered LVGL input devices

**Example:**
```python
devices = InputManager.list_indevs()
for indev in devices:
    print(f"Device type: {indev.get_type()}")
```

#### `has_indev_type(indev_type)`
Check if any registered input device has the specified type.

This is a convenience method for checking device capabilities without iterating through the device list.

**Parameters:**
- `indev_type` (int): LVGL input device type (e.g., `lv.INDEV_TYPE.KEYPAD`, `lv.INDEV_TYPE.POINTER`)

**Returns:** bool - True if a device with the specified type is registered, False otherwise

**Example:**
```python
if InputManager.has_indev_type(lv.INDEV_TYPE.KEYPAD):
    print("Device has physical buttons")
else:
    print("Device is touch-only")
```

## Practical Examples

### Touch Event Handling

```python
from mpos import InputManager, DisplayMetrics
import lvgl as lv

def handle_scroll(event):
    """Handle scroll event with pointer coordinates."""
    x, y = InputManager.pointer_xy()
    
    if x == -1:
        print("No active input device")
        return
    
    # Check if touch is in specific region
    if x < DisplayMetrics.pct_of_width(50):
        print("Touch in left half")
    else:
        print("Touch in right half")
```

### Focus Management in Dialogs

```python
from mpos import InputManager, Activity
import lvgl as lv

class MyDialog(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Create buttons
        yes_btn = lv.button(screen)
        no_btn = lv.button(screen)
        
        # ... setup buttons ...
        
        # Set focus to "No" button by default
        focusgroup = lv.group_get_default()
        if focusgroup:
            InputManager.emulate_focus_obj(focusgroup, no_btn)
        
        self.setContentView(screen)
```

### Keyboard Navigation

```python
from mpos import InputManager
import lvgl as lv

def navigate_to_button(button):
    """Navigate keyboard focus to a specific button."""
    focusgroup = lv.group_get_default()
    if focusgroup:
        InputManager.emulate_focus_obj(focusgroup, button)
        print(f"Focus moved to {button}")
```

### Input Device Registration (Board Initialization)

```python
# In board initialization file (e.g., fri3d_2024.py)
from mpos import InputManager
import lvgl as lv

# Create and configure input device
indev = lv.indev_create()
indev.set_type(lv.INDEV_TYPE.KEYPAD)
indev.set_read_cb(keypad_read_cb)

# Register with InputManager for app access
InputManager.register_indev(indev)
```

### Querying Input Devices (App Code)

```python
from mpos import InputManager, Activity
import lvgl as lv

class MyGame(Activity):
    def onCreate(self):
        # Check if device has physical buttons
        if InputManager.has_indev_type(lv.INDEV_TYPE.KEYPAD):
            # Show button-based controls
            self.show_button_controls()
        else:
            # Show touch-based controls
            self.show_touch_controls()
    
    def show_button_controls(self):
        """Display UI optimized for physical buttons."""
        helptext = "Press A to start!\nY to reset, B to show FPS"
        # ... setup button-based UI ...
    
    def show_touch_controls(self):
        """Display UI optimized for touch."""
        helptext = "Tap to start!\nTop left to reset, bottom left for FPS"
        # ... setup touch-based UI ...
```

## Implementation Details

### File Structure

```
mpos/ui/
├── input_manager.py    # Core InputManager class
└── __init__.py         # Exports InputManager
```

### Focus Emulation Algorithm

The `emulate_focus_obj()` method works by:

1. Checking if focusgroup and target are valid
2. Iterating through the focus group using `focus_next()`
3. Comparing the currently focused object with the target
4. Stopping when the target is found
5. Printing a warning if the target is not found

```python
def emulate_focus_obj(cls, focusgroup, target):
    if not focusgroup or not target:
        return
    
    for objnr in range(focusgroup.get_obj_count()):
        currently_focused = focusgroup.get_focused()
        if currently_focused is target:
            return  # Found!
        else:
            focusgroup.focus_next()  # Try next
    
    print("WARNING: emulate_focus_obj failed to find target")
```

## Design Patterns

### Singleton Pattern

InputManager uses the singleton pattern with class methods:

```python
class InputManager:
    @classmethod
    def pointer_xy(cls):
        # Direct class method call, no instance needed
        return x, y
```

This avoids the need for instance creation while maintaining a single source of truth.

### LVGL Integration

InputManager integrates directly with LVGL's input device and focus group APIs:

```python
# Pointer access
indev = lv.indev_active()
p = lv.point_t()
indev.get_point(p)

# Focus management
focusgroup = lv.group_get_default()
focusgroup.focus_next()
```

### Device Registration Pattern

InputManager uses a registration pattern to decouple hardware initialization from app code:

**Board Initialization (Hardware Layer):**
```python
# fri3d_2024.py - knows about hardware
InputManager.register_indev(keypad_indev)
InputManager.register_indev(touch_indev)
```

**App Code (Application Layer):**
```python
# quasibird.py - doesn't know about hardware
if InputManager.has_indev_type(lv.INDEV_TYPE.KEYPAD):
    # Adapt UI for buttons
else:
    # Adapt UI for touch
```

This pattern provides:
- **Hardware Abstraction** - Apps don't hardcode hardware IDs
- **Flexibility** - New boards can register different devices
- **Testability** - Apps can be tested with different device configurations
- **Scalability** - Easy to add new input device types

## Related Frameworks

- **[DisplayMetrics](display-metrics.md)** - Display properties and pointer coordinates (now uses InputManager)
- **[WidgetAnimator](widget-animator.md)** - UI animation framework
- **[MposKeyboard](../ui/keyboard.md)** - Custom keyboard with focus management

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
- [LVGL Documentation](https://docs.lvgl.io/)
