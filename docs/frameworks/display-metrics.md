# DisplayMetrics

DisplayMetrics is an Android-inspired singleton class that provides a unified, clean API for accessing display properties like width, height, DPI, and pointer coordinates.

## Overview

Instead of scattered module-level functions, DisplayMetrics centralizes all display-related metrics in a single class with class methods. This provides:

- **Unified API** - Single class for all display metrics
- **Clean Namespace** - No scattered functions cluttering imports
- **Testable** - DisplayMetrics can be tested independently
- **Android-Inspired** - Follows Android's DisplayMetrics pattern
- **Elegant Initialization** - Setters called from `init_rootscreen()`

## Architecture

DisplayMetrics is implemented as a singleton using class variables and class methods:

```python
class DisplayMetrics:
    _width = None
    _height = None
    _dpi = None
    
    @classmethod
    def set_resolution(cls, width, height):
        """Set the display resolution (called by init_rootscreen)."""
        cls._width = width
        cls._height = height
    
    @classmethod
    def width(cls):
        """Get display width in pixels."""
        return cls._width
    
    # ... other methods
```

No instance creation is needed - all methods are class methods that operate on class variables.

## Usage

### Basic Display Metrics

```python
from mpos import DisplayMetrics

# Get display dimensions
width = DisplayMetrics.width()      # e.g., 320
height = DisplayMetrics.height()    # e.g., 240
dpi = DisplayMetrics.dpi()          # e.g., 130
```

### Percentage-Based Calculations

```python
from mpos import DisplayMetrics

# Get percentage of display width/height
half_width = DisplayMetrics.pct_of_width(50)    # 50% of width
quarter_height = DisplayMetrics.pct_of_height(25)  # 25% of height
```

### Dimension Queries

```python
from mpos import DisplayMetrics

# Get min/max dimensions
min_dim = DisplayMetrics.min_dimension()  # min(width, height)
max_dim = DisplayMetrics.max_dimension()  # max(width, height)
```

### Pointer/Touch Coordinates

```python
from mpos import DisplayMetrics

# Get current pointer/touch position
x, y = DisplayMetrics.pointer_xy()
if x == -1 and y == -1:
    print("No active input device")
else:
    print(f"Pointer at ({x}, {y})")
```

## API Reference

### Class Methods

#### `set_resolution(width, height)`
Set the display resolution. Called by `init_rootscreen()` during initialization.

**Parameters:**
- `width` (int): Display width in pixels
- `height` (int): Display height in pixels

**Example:**
```python
DisplayMetrics.set_resolution(320, 240)
```

#### `set_dpi(dpi)`
Set the display DPI. Called by `init_rootscreen()` during initialization.

**Parameters:**
- `dpi` (int): Display DPI (dots per inch)

**Example:**
```python
DisplayMetrics.set_dpi(130)
```

#### `width()`
Get display width in pixels.

**Returns:** int - Display width

**Example:**
```python
w = DisplayMetrics.width()  # 320
```

#### `height()`
Get display height in pixels.

**Returns:** int - Display height

**Example:**
```python
h = DisplayMetrics.height()  # 240
```

#### `dpi()`
Get display DPI (dots per inch).

**Returns:** int - Display DPI

**Example:**
```python
dpi = DisplayMetrics.dpi()  # 130
```

#### `pct_of_width(pct)`
Get percentage of display width.

**Parameters:**
- `pct` (int): Percentage (0-100)

**Returns:** int - Pixel value

**Example:**
```python
half_width = DisplayMetrics.pct_of_width(50)  # 160 (50% of 320)
```

#### `pct_of_height(pct)`
Get percentage of display height.

**Parameters:**
- `pct` (int): Percentage (0-100)

**Returns:** int - Pixel value

**Example:**
```python
quarter_height = DisplayMetrics.pct_of_height(25)  # 60 (25% of 240)
```

#### `min_dimension()`
Get minimum dimension (width or height).

**Returns:** int - Minimum of width and height

**Example:**
```python
min_dim = DisplayMetrics.min_dimension()  # 240
```

#### `max_dimension()`
Get maximum dimension (width or height).

**Returns:** int - Maximum of width and height

**Example:**
```python
max_dim = DisplayMetrics.max_dimension()  # 320
```

#### `pointer_xy()`
Get current pointer/touch coordinates.

**Returns:** tuple - (x, y) coordinates, or (-1, -1) if no active input device

**Example:**
```python
x, y = DisplayMetrics.pointer_xy()
if x != -1:
    print(f"Touch at ({x}, {y})")
```

## Practical Examples

### Responsive Layout

```python
from mpos import Activity, DisplayMetrics
import lvgl as lv

class MyApp(Activity):
    def onCreate(self):
        screen = lv.obj()
        
        # Create a button that's 50% of display width
        button = lv.button(screen)
        button.set_width(DisplayMetrics.pct_of_width(50))
        button.set_height(DisplayMetrics.pct_of_height(20))
        button.center()
        
        self.setContentView(screen)
```

### Adaptive UI Based on Screen Size

```python
from mpos import DisplayMetrics

def get_font_size():
    """Return appropriate font size based on display size."""
    min_dim = DisplayMetrics.min_dimension()
    
    if min_dim < 200:
        return lv.font_montserrat_12
    elif min_dim < 300:
        return lv.font_montserrat_16
    else:
        return lv.font_montserrat_20
```

### Touch Event Handling

```python
from mpos import DisplayMetrics
import lvgl as lv

def handle_scroll(event):
    """Handle scroll event with pointer coordinates."""
    x, y = DisplayMetrics.pointer_xy()
    
    if x == -1:
        print("No active input device")
        return
    
    # Check if touch is in specific region
    if x < DisplayMetrics.pct_of_width(50):
        print("Touch in left half")
    else:
        print("Touch in right half")
```

## Implementation Details

### File Structure

```
mpos/ui/
├── display_metrics.py    # Core DisplayMetrics class
├── display.py            # Initialization (init_rootscreen)
└── __init__.py           # Exports DisplayMetrics
```

### Initialization Flow

1. **System Startup** - `init_rootscreen()` is called
2. **Query Display** - Gets resolution and DPI from LVGL
3. **Set Metrics** - Calls `DisplayMetrics.set_resolution()` and `DisplayMetrics.set_dpi()`
4. **Ready to Use** - Apps can now call DisplayMetrics methods

```python
# In display.py
def init_rootscreen():
    screen = lv.screen_active()
    disp = screen.get_display()
    width = disp.get_horizontal_resolution()
    height = disp.get_vertical_resolution()
    dpi = disp.get_dpi()
    
    # Initialize DisplayMetrics
    DisplayMetrics.set_resolution(width, height)
    DisplayMetrics.set_dpi(dpi)
```

## Design Patterns

### Singleton Pattern

DisplayMetrics uses the singleton pattern with class variables and class methods:

```python
class DisplayMetrics:
    _width = None  # Shared across all "instances"
    
    @classmethod
    def width(cls):
        return cls._width
```

This avoids the need for instance creation while maintaining a single source of truth.

### Class Method Delegation

All methods are class methods, so no instance is needed:

```python
# No need for DisplayMetrics() or DisplayMetrics.get()
width = DisplayMetrics.width()  # Direct class method call
```

## Related Frameworks

- **[AudioFlinger](audioflinger.md)** - Audio management (similar singleton pattern)
- **[ConnectivityManager](connectivity-manager.md)** - Network connectivity
- **[SensorManager](sensor-manager.md)** - Sensor access

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
