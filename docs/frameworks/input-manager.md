# InputManager

InputManager is a singleton framework that provides a unified API for managing input device interactions, including pointer/touch coordinates and focus management.

## Overview

InputManager centralizes all input-related operations in a single class with class methods. This provides:

- **Unified API** - Single class for all input management
- **Clean Namespace** - No scattered functions cluttering imports
- **Testable** - InputManager can be tested independently
- **Focus Control** - Emulate focus on specific UI objects
- **Pointer Access** - Get current touch/pointer coordinates

## Architecture

InputManager is implemented as a singleton using class variables and class methods:

```python
class InputManager:
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

## Related Frameworks

- **[DisplayMetrics](display-metrics.md)** - Display properties and pointer coordinates (now uses InputManager)
- **[WidgetAnimator](widget-animator.md)** - UI animation framework
- **[MposKeyboard](../ui/keyboard.md)** - Custom keyboard with focus management

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
- [LVGL Documentation](https://docs.lvgl.io/)
