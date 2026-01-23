# AppearanceManager

AppearanceManager is an Android-inspired singleton framework that provides a unified, clean API for managing all aspects of the app's visual appearance: light/dark mode, theme colors, UI dimensions, and LVGL styling.

## Overview

Instead of scattered functions and constants, AppearanceManager centralizes all appearance-related settings in a single class with class methods. This provides:

- **Unified API** - Single class for all appearance settings
- **Clean Namespace** - No scattered functions or constants cluttering imports
- **Testable** - AppearanceManager can be tested independently
- **Android-Inspired** - Follows Android's Configuration and Resources pattern
- **Elegant Initialization** - Single `init()` call from `main.py`

## Architecture

AppearanceManager is implemented as a singleton using class variables and class methods:

```python
class AppearanceManager:
    """Android-inspired appearance management singleton."""
    
    # UI Dimensions (constants)
    NOTIFICATION_BAR_HEIGHT = 24
    
    # Private state (class variables)
    _is_light_mode = True
    _primary_color = None
    
    @classmethod
    def init(cls, prefs):
        """Initialize from SharedPreferences."""
        pass
    
    @classmethod
    def is_light_mode(cls):
        """Check if light mode is enabled."""
        return cls._is_light_mode
```

No instance creation is needed - all methods are class methods that operate on class variables.

## Quick Start

### Basic Usage

```python
from mpos import AppearanceManager

# Check light/dark mode
if AppearanceManager.is_light_mode():
    print("Light mode enabled")
else:
    print("Dark mode enabled")

# Get UI dimensions
bar_height = AppearanceManager.get_notification_bar_height()

# Get theme colors
primary_color = AppearanceManager.get_primary_color()
```

### Initialization

AppearanceManager is automatically initialized during system startup in `main.py`:

```python
from mpos.ui.appearance_manager import AppearanceManager
import mpos.config

prefs = mpos.config.SharedPreferences("com.micropythonos.settings")
AppearanceManager.init(prefs)  # Called once at boot
```

## API Reference

### Class Constants

#### `NOTIFICATION_BAR_HEIGHT`
Height of the notification bar in pixels.

**Value:** 24 (pixels)

**Example:**
```python
from mpos import AppearanceManager

bar_height = AppearanceManager.NOTIFICATION_BAR_HEIGHT  # 24
```

### Class Methods

#### `init(prefs)`
Initialize AppearanceManager from SharedPreferences.

Called once during system startup to load theme settings and initialize the LVGL theme.

**Parameters:**
- `prefs` (SharedPreferences): Preferences object containing theme settings
  - `"theme_light_dark"`: "light" or "dark" (default: "light")
  - `"theme_primary_color"`: hex color string like "0xFF5722" or "#FF5722"

**Example:**
```python
from mpos.ui.appearance_manager import AppearanceManager
import mpos.config

prefs = mpos.config.SharedPreferences("com.micropythonos.settings")
AppearanceManager.init(prefs)
```

#### `is_light_mode()`
Check if light mode is currently enabled.

**Returns:** bool - True if light mode, False if dark mode

**Example:**
```python
from mpos import AppearanceManager

if AppearanceManager.is_light_mode():
    print("Using light theme")
else:
    print("Using dark theme")
```

#### `set_light_mode(is_light, prefs=None)`
Set light/dark mode and update the theme.

**Parameters:**
- `is_light` (bool): True for light mode, False for dark mode
- `prefs` (SharedPreferences, optional): If provided, saves the setting

**Example:**
```python
from mpos import AppearanceManager

# Switch to dark mode
AppearanceManager.set_light_mode(False)

# Switch to light mode and save preference
AppearanceManager.set_light_mode(True, prefs)
```

#### `get_primary_color()`
Get the primary theme color.

**Returns:** lv.color_t or None - The primary color

**Example:**
```python
from mpos import AppearanceManager
import lvgl as lv

color = AppearanceManager.get_primary_color()
if color:
    button.set_style_bg_color(color, 0)
```

#### `set_primary_color(color, prefs=None)`
Set the primary theme color.

**Parameters:**
- `color` (lv.color_t or int): The new primary color
- `prefs` (SharedPreferences, optional): If provided, saves the setting

**Example:**
```python
from mpos import AppearanceManager
import lvgl as lv

AppearanceManager.set_primary_color(lv.color_hex(0xFF5722))
```

#### `get_notification_bar_height()`
Get the height of the notification bar.

The notification bar is the top bar that displays system information (time, battery, signal, etc.).

**Returns:** int - Height in pixels (default: 24)

**Example:**
```python
from mpos import AppearanceManager

bar_height = AppearanceManager.get_notification_bar_height()
content_y = bar_height  # Position content below the bar
```

#### `get_keyboard_button_fix_style()`
Get the keyboard button fix style for light mode.

The LVGL default theme applies white background to keyboard buttons, which makes them invisible in light mode. This method returns a custom style to override that.

**Returns:** lv.style_t or None - Style to apply, or None if not needed

**Note:** This is a workaround for an LVGL/MicroPython issue. Only applies in light mode.

**Example:**
```python
from mpos import AppearanceManager
import lvgl as lv

style = AppearanceManager.get_keyboard_button_fix_style()
if style:
    keyboard.add_style(style, lv.PART.ITEMS)
```

#### `apply_keyboard_fix(keyboard)`
Apply keyboard button visibility fix to a keyboard instance.

Call this after creating a keyboard to ensure buttons are visible in light mode.

**Parameters:**
- `keyboard`: The lv.keyboard instance to fix

**Example:**
```python
from mpos import AppearanceManager
import lvgl as lv

keyboard = lv.keyboard(screen)
AppearanceManager.apply_keyboard_fix(keyboard)
```

## Practical Examples

### Responsive Layout Based on Theme

```python
from mpos import Activity, AppearanceManager, DisplayMetrics
import lvgl as lv

class MyApp(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Create a button with theme-aware colors
        button = lv.button(screen)
        button.set_width(DisplayMetrics.pct_of_width(50))
        button.set_height(DisplayMetrics.pct_of_height(20))
        
        # Use primary color from theme
        primary_color = AppearanceManager.get_primary_color()
        if primary_color:
            button.set_style_bg_color(primary_color, 0)
        
        button.center()
        self.setContentView(screen)
```

### Positioning Below Notification Bar

```python
from mpos import AppearanceManager, DisplayMetrics
import lvgl as lv

def create_content_area(parent):
    """Create content area positioned below notification bar."""
    content = lv.obj(parent)
    
    # Position below notification bar
    bar_height = AppearanceManager.get_notification_bar_height()
    content.set_pos(0, bar_height)
    
    # Fill remaining space
    content.set_size(
        DisplayMetrics.width(),
        DisplayMetrics.height() - bar_height
    )
    
    return content
```

### Theme-Aware Keyboard

```python
from mpos import AppearanceManager
import lvgl as lv

def create_keyboard(parent):
    """Create a keyboard with proper styling in light mode."""
    keyboard = lv.keyboard(parent)
    
    # Apply light mode fix if needed
    AppearanceManager.apply_keyboard_fix(keyboard)
    
    return keyboard
```

### Checking Theme at Runtime

```python
from mpos import AppearanceManager
import lvgl as lv

def update_ui_for_theme():
    """Update UI colors based on current theme."""
    if AppearanceManager.is_light_mode():
        # Light mode: use light colors
        bg_color = lv.color_hex(0xFFFFFF)
        text_color = lv.color_hex(0x000000)
    else:
        # Dark mode: use dark colors
        bg_color = lv.color_hex(0x1E1E1E)
        text_color = lv.color_hex(0xFFFFFF)
    
    screen = lv.screen_active()
    screen.set_style_bg_color(bg_color, 0)
    screen.set_style_text_color(text_color, 0)
```

## Implementation Details

### File Structure

```
mpos/ui/
├── appearance_manager.py    # Core AppearanceManager class
├── topmenu.py               # Uses AppearanceManager
├── gesture_navigation.py     # Uses AppearanceManager
└── __init__.py              # Exports AppearanceManager
```

### Initialization Flow

1. **System Startup** - `main.py` is executed
2. **Create Preferences** - `SharedPreferences("com.micropythonos.settings")` is created
3. **Initialize AppearanceManager** - `AppearanceManager.init(prefs)` is called
4. **Load Settings** - Theme mode and colors are loaded from preferences
5. **Initialize LVGL** - `lv.theme_default_init()` is called with loaded settings
6. **Ready to Use** - Apps can now call AppearanceManager methods

```python
# In main.py
prefs = mpos.config.SharedPreferences("com.micropythonos.settings")
AppearanceManager.init(prefs)  # Initialize appearance
init_rootscreen()              # Initialize display
```

## Design Patterns

### Singleton Pattern

AppearanceManager uses the singleton pattern with class variables and class methods:

```python
class AppearanceManager:
    _is_light_mode = True  # Shared across all "instances"
    
    @classmethod
    def is_light_mode(cls):
        return cls._is_light_mode
```

This avoids the need for instance creation while maintaining a single source of truth.

### Class Method Delegation

All methods are class methods, so no instance is needed:

```python
# No need for AppearanceManager() or AppearanceManager.get()
is_light = AppearanceManager.is_light_mode()  # Direct class method call
```

## Preferences Storage

AppearanceManager reads and writes to SharedPreferences:

**Preference Keys:**
- `"theme_light_dark"` - "light" or "dark" (default: "light")
- `"theme_primary_color"` - hex color string like "0xFF5722"

**Example:**
```python
import mpos.config

prefs = mpos.config.SharedPreferences("com.micropythonos.settings")

# Read current theme
theme = prefs.get_string("theme_light_dark", "light")

# Write new theme
editor = prefs.edit()
editor.put_string("theme_light_dark", "dark")
editor.commit()
```

## Related Frameworks

- **[DisplayMetrics](display-metrics.md)** - Display properties (width, height, DPI)
- **[SensorManager](sensor-manager.md)** - Sensor access
- **[SharedPreferences](preferences.md)** - Persistent app data

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
