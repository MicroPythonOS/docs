# Frameworks

MicroPythonOS provides a unified framework architecture for accessing system services. All frameworks follow a consistent, simple pattern that makes them easy to discover, use, and extend.

## Importing from mpos

**Apps should only import from the `mpos` module directly.** Importing from submodules (such as `mpos.ui`, `mpos.content`, etc.) is not necessary and should be avoided.

All frameworks and utilities you need are available through the main `mpos` module. If you find yourself needing to import from a submodule, or if you would like a new framework to be created, please [create a GitHub issue](https://github.com/MicroPythonOS/MicroPythonOS/issues) describing your use case.

### Correct Import Style

Import directly from `mpos`:

```python
from mpos import DisplayMetrics

DisplayMetrics.width()
```

Or import the module and use it:

```python
import mpos

mpos.DisplayMetrics.width()
```

### Avoid Submodule Imports

Do not import from submodules:

```python
# ❌ Don't do this
from mpos.ui import DisplayMetrics
from mpos.content import AppManager
```

Instead, use the main `mpos` module which re-exports everything you need:

```python
# ✅ Do this instead
from mpos import DisplayMetrics, AppManager
```

## Overview

Frameworks are centralized services that provide access to system capabilities like audio, networking, camera, sensors, and task management. They follow a **singleton class pattern with class methods**, ensuring a predictable and discoverable API across the entire system.

### Design Philosophy

- **Simple**: Single pattern, no `.get()` calls, clear imports
- **Functional**: Supports all use cases (state management, async operations, callbacks)
- **Harmonized**: All frameworks work identically
- **Discoverable**: IDE autocomplete shows all available methods

## Unified Framework Pattern

All frameworks follow a **singleton pattern with class method delegation**. This provides a clean API where you call class methods directly without needing to instantiate or call `.get()`.

```python
class MyFramework:
    """Centralized service for [purpose]."""
    
    _instance = None  # Singleton instance
    
    def __init__(self):
        """Initialize singleton instance (called once)."""
        if MyFramework._instance:
            return
        MyFramework._instance = self
        # initialization logic
    
    @classmethod
    def get(cls):
        """Get or create the singleton instance."""
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
    
    def method_name(self, *args, **kwargs):
        """Instance methods (implementation)."""
        # implementation
        return result

# Class method delegation (at module level)
_original_methods = {}
_methods_to_delegate = ['method_name']

for method_name in _methods_to_delegate:
    _original_methods[method_name] = getattr(MyFramework, method_name)

def _make_class_method(method_name):
    """Create a class method that delegates to the singleton instance."""
    original_method = _original_methods[method_name]
    
    @classmethod
    def class_method(cls, *args, **kwargs):
        instance = cls.get()
        return original_method(instance, *args, **kwargs)
    
    return class_method

for method_name in _methods_to_delegate:
    setattr(MyFramework, method_name, _make_class_method(method_name))
```

### Key Characteristics

| Aspect | Details |
|--------|---------|
| **Instantiation** | Singleton pattern - single instance created on first use |
| **Initialization** | Call `Framework.init()` once at startup (or auto-initialize on first use) |
| **State Management** | Instance variables stored in singleton instance |
| **API Access** | `Framework.method_name(...)` - class methods delegate to singleton |
| **Async Support** | Instance methods can be async (`async def`) |
| **Testing** | Easy to mock by replacing `_instance` |

## Available Frameworks

### AppManager
Manages app discovery, installation, launching, and version management.

```python
from mpos import AppManager

# Get all installed apps
apps = AppManager.get_app_list()

# Start an app
AppManager.start_app("com.example.myapp")

# Install from .mpk file
AppManager.install_mpk("/tmp/app.mpk", "apps/com.example.newapp")

# Check for updates
if AppManager.is_update_available("com.example.myapp", "2.0.0"):
    print("Update available")
```

### AppearanceManager
Manages visual appearance: light/dark mode, theme colors, UI dimensions, and LVGL styling.

```python
from mpos import AppearanceManager

# Initialize at startup
AppearanceManager.init(prefs)

# Use anywhere
if AppearanceManager.is_light_mode():
    print("Light mode enabled")

bar_height = AppearanceManager.get_notification_bar_height()
primary_color = AppearanceManager.get_primary_color()
```

### AudioManager
Manages audio playback and recording.

```python
from mpos import AudioManager

# Initialize at startup
AudioManager.init()

# Use anywhere
AudioManager.play_wav("path/to/audio.wav")
AudioManager.stop()
```

### DownloadManager
Handles HTTP downloads with session management.

```python
from mpos import DownloadManager

# Initialize at startup
DownloadManager.init()

# Use anywhere
await DownloadManager.download_url("https://example.com/file.bin", "local_path")
```

### ConnectivityManager
Monitors network connectivity status.

```python
from mpos import ConnectivityManager

# Initialize at startup
ConnectivityManager.init()

# Check connectivity
if ConnectivityManager.is_online():
    print("Connected to network")
```

### CameraManager
Provides access to camera hardware.

```python
from mpos import CameraManager

# Initialize at startup
CameraManager.init()

# Capture image
image_data = CameraManager.capture_photo()
```

### SensorManager
Manages sensor access (accelerometer, gyroscope, etc.).

```python
from mpos import SensorManager

# Initialize at startup
SensorManager.init()

# Read sensor data
accel_data = SensorManager.read_accelerometer()
```

### TaskManager
Manages async task creation and lifecycle.

```python
from mpos import TaskManager

# Create and track tasks
task = TaskManager.create_task(my_coroutine())
TaskManager.sleep(seconds)
```

### SharedPreferences
Per-app configuration storage (exception to the pattern - instance-based).

```python
from mpos import SharedPreferences

# Create instance per app
prefs = SharedPreferences("com.example.myapp")

# Store and retrieve values
prefs.set_string("key", "value")
value = prefs.get_string("key", default="default")
```

## Framework Initialization

Frameworks should be initialized once at system startup in the board initialization file:

```python
# In board/your_board.py
from mpos import AppearanceManager, AudioManager, DownloadManager, ConnectivityManager, CameraManager, SensorManager, TaskManager, AppManager

def init_frameworks():
    """Initialize all frameworks."""
    AppManager.refresh_apps()  # Discover all installed apps
    AppearanceManager.init(prefs)  # Requires SharedPreferences
    AudioManager.init()
    DownloadManager.init()
    ConnectivityManager.init()
    CameraManager.init()
    SensorManager.init()
    TaskManager.init()
```

**Note:** AppManager doesn't require explicit initialization - call `refresh_apps()` to discover apps at startup.

## Creating New Frameworks

When creating a new framework, follow this template:

```python
"""
MyFramework - [Brief description of what this framework does]
"""

class MyFramework:
    """Centralized service for [purpose]."""
    
    _instance = None  # Singleton instance
    
    def __init__(self):
        """Initialize singleton instance."""
        if MyFramework._instance:
            return
        MyFramework._instance = self
        # initialization logic
    
    @classmethod
    def get(cls):
        """Get or create the singleton instance."""
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
    
    def init(self, **kwargs):
        """
        Initialize the framework.
        
        Args:
            **kwargs: Configuration options
            
        Returns:
            bool: True if initialization successful
        """
        # Perform initialization
        self._config = kwargs
        return True
    
    def is_available(self):
        """Check if framework is available."""
        return hasattr(self, '_config')
    
    def your_method(self, arg1, arg2):
        """
        Do something.
        
        Args:
            arg1: First argument
            arg2: Second argument
            
        Returns:
            Result of operation
        """
        if not self.is_available():
            raise RuntimeError("MyFramework not initialized")
        
        # Implementation
        return result

# Class method delegation (at module level)
_original_methods = {}
_methods_to_delegate = ['init', 'is_available', 'your_method']

for method_name in _methods_to_delegate:
    _original_methods[method_name] = getattr(MyFramework, method_name)

def _make_class_method(method_name):
    """Create a class method that delegates to the singleton instance."""
    original_method = _original_methods[method_name]
    
    @classmethod
    def class_method(cls, *args, **kwargs):
        instance = cls.get()
        return original_method(instance, *args, **kwargs)
    
    return class_method

for method_name in _methods_to_delegate:
    setattr(MyFramework, method_name, _make_class_method(method_name))
```

### Checklist for New Frameworks

- [ ] Implement singleton pattern with `_instance` class variable
- [ ] Implement `__init__()` to initialize singleton
- [ ] Implement `get()` classmethod to get/create singleton
- [ ] Implement `init()` instance method for initialization
- [ ] Implement `is_available()` instance method to check status
- [ ] All public methods are instance methods
- [ ] Add class method delegation at module level
- [ ] Add docstrings to all methods
- [ ] Import in `mpos/__init__.py`
- [ ] Add to board initialization files
- [ ] Write tests in `MicroPythonOS/tests/`

## Import Pattern

All frameworks are imported consistently as classes from the main `mpos` module:

```python
from mpos import AppearanceManager, AudioManager, CameraManager, ConnectivityManager, DownloadManager, SensorManager, SharedPreferences, TaskManager, WifiService, AppManager

# Then use class methods directly (no .get() needed)
AppearanceManager.init(prefs)
AudioManager.play_wav("music.wav")
SensorManager.read_sensor(accel)
```

**Note:** Some frameworks like `AudioManager` and `SensorManager` use singleton patterns internally, but the API is the same - call class methods directly without needing to call `.get()`.

## Benefits of Harmonization

| Aspect | Before | After |
|--------|--------|-------|
| **API Consistency** | 5 different patterns | 1 unified pattern |
| **Learning Curve** | High (multiple patterns) | Low (single pattern) |
| **Import Clarity** | Mixed class/module imports | All consistent class imports |
| **IDE Support** | Partial autocomplete | Full autocomplete |
| **Testing** | Varies by framework | Consistent mocking approach |
| **Documentation** | Per-framework docs | Single pattern reference |
| **Onboarding** | Confusing for new developers | Clear and straightforward |

## Exception: SharedPreferences

`SharedPreferences` is intentionally instance-based rather than a singleton class method framework because:

1. **Per-app isolation**: Each app needs its own configuration namespace
2. **Multiple instances**: Different apps may need different preference stores
3. **Flexibility**: Apps can create multiple preference instances if needed

```python
# Each app creates its own instance
prefs = SharedPreferences("com.example.myapp")
prefs.set_string("theme", "dark")
```

## Best Practices

### Do's

✅ Call `Framework.init()` once at system startup
✅ Use class methods directly (they delegate to singleton)
✅ Check `is_available()` before using framework
✅ Store state in singleton instance variables
✅ Document initialization requirements
✅ Write tests for framework behavior
✅ Use class method delegation pattern for clean API

### Don'ts

❌ Create multiple instances of framework classes
❌ Call `.get()` directly in app code (it's internal)
❌ Mix instance methods with class methods in public API
❌ Store framework state in global variables
❌ Skip initialization in board files
❌ Use different patterns for different frameworks

## Testing Frameworks

Frameworks are easy to test due to their singleton pattern:

```python
import unittest
from mpos import MyFramework

class TestMyFramework(unittest.TestCase):
    def setUp(self):
        """Reset framework state before each test."""
        MyFramework._instance = None  # Reset singleton
    
    def test_initialization(self):
        """Test framework initialization."""
        self.assertFalse(MyFramework.is_available())
        MyFramework.init()
        self.assertTrue(MyFramework.is_available())
    
    def test_method(self):
        """Test framework method."""
        MyFramework.init()
        result = MyFramework.your_method("arg1", "arg2")
        self.assertEqual(result, expected_value)
    
    def tearDown(self):
        """Clean up after each test."""
        MyFramework._instance = None  # Reset singleton
```

## See Also

- [AppManager](../frameworks/app-manager.md): App discovery, installation, and launching
- [System Components](system-components.md): Overview of all system components
- [Architecture Overview](overview.md): High-level design principles
- [App Development Guide](../apps/index.md): How to use frameworks in apps
