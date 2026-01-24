# Frameworks

MicroPythonOS provides a unified framework architecture for accessing system services. All frameworks follow a consistent, simple pattern that makes them easy to discover, use, and extend.

## Overview

Frameworks are centralized services that provide access to system capabilities like audio, networking, camera, sensors, and task management. They follow a **singleton class pattern with class methods**, ensuring a predictable and discoverable API across the entire system.

### Design Philosophy

- **Simple**: Single pattern, no `.get()` calls, clear imports
- **Functional**: Supports all use cases (state management, async operations, callbacks)
- **Harmonized**: All frameworks work identically
- **Discoverable**: IDE autocomplete shows all available methods

## Unified Framework Pattern

All frameworks follow this standardized structure:

```python
class MyFramework:
    """Centralized service for [purpose]."""
    
    _initialized = False
    _instance_data = {}
    
    @classmethod
    def init(cls, *args, **kwargs):
        """Initialize the framework (call once at startup)."""
        cls._initialized = True
        # initialization logic
        return True
    
    @classmethod
    def is_available(cls):
        """Check if framework is available."""
        return cls._initialized
    
    @classmethod
    def method_name(cls, *args, **kwargs):
        """Framework methods as class methods."""
        # implementation
        return result
```

### Key Characteristics

| Aspect | Details |
|--------|---------|
| **Instantiation** | No instance creation needed - use class methods directly |
| **Initialization** | Call `Framework.init()` once at startup |
| **State Management** | Use class variables (`_instance_data`, `_initialized`) |
| **API Access** | `Framework.method_name(...)` - no `.get()` needed |
| **Async Support** | Class methods can be async (`async def`) |
| **Testing** | Easy to mock class methods and class variables |

## Available Frameworks

### AppManager
Manages app discovery, installation, launching, and version management.

```python
from mpos.content.app_manager import AppManager

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

### AudioFlinger
Manages audio playback and recording.

```python
from mpos import AudioFlinger

# Initialize at startup
AudioFlinger.init()

# Use anywhere
AudioFlinger.play_wav("path/to/audio.wav")
AudioFlinger.stop()
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
from mpos import (
    AppearanceManager,
    AudioFlinger,
    DownloadManager,
    ConnectivityManager,
    CameraManager,
    SensorManager,
    TaskManager
)
from mpos.content.app_manager import AppManager

def init_frameworks():
    """Initialize all frameworks."""
    AppManager.refresh_apps()  # Discover all installed apps
    AppearanceManager.init(prefs)  # Requires SharedPreferences
    AudioFlinger.init()
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
    
    _initialized = False
    _instance_data = {}
    
    @classmethod
    def init(cls, **kwargs):
        """
        Initialize the framework.
        
        Args:
            **kwargs: Configuration options
            
        Returns:
            bool: True if initialization successful
        """
        if cls._initialized:
            return True
        
        # Perform initialization
        cls._instance_data['config'] = kwargs
        cls._initialized = True
        return True
    
    @classmethod
    def is_available(cls):
        """Check if framework is available."""
        return cls._initialized
    
    @classmethod
    def your_method(cls, arg1, arg2):
        """
        Do something.
        
        Args:
            arg1: First argument
            arg2: Second argument
            
        Returns:
            Result of operation
        """
        if not cls.is_available():
            raise RuntimeError("MyFramework not initialized")
        
        # Implementation
        return result
```

### Checklist for New Frameworks

- [ ] Inherit from no base class (use class methods only)
- [ ] Include `_initialized` class variable
- [ ] Include `_instance_data` class variable for state
- [ ] Implement `init()` classmethod for initialization
- [ ] Implement `is_available()` classmethod to check status
- [ ] All public methods are classmethods
- [ ] Add docstrings to all methods
- [ ] Import in `mpos/__init__.py`
- [ ] Add to board initialization files
- [ ] Write tests in `MicroPythonOS/tests/`

## Import Pattern

All frameworks are imported consistently as classes:

```python
from mpos import (
    AppearanceManager,
    AudioFlinger,
    CameraManager,
    ConnectivityManager,
    DownloadManager,
    SensorManager,
    SharedPreferences,
    TaskManager,
    WifiService,
)
from mpos.content.app_manager import AppManager
```

**Note:** AppManager is imported from `mpos.content.app_manager` rather than the main `mpos` module.

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
✅ Use class methods directly without `.get()`  
✅ Check `is_available()` before using framework  
✅ Use class variables for state management  
✅ Document initialization requirements  
✅ Write tests for framework behavior  

### Don'ts

❌ Create instances of framework classes  
❌ Call `.get()` to access frameworks  
❌ Mix module-level functions with class methods  
❌ Store framework state in global variables  
❌ Skip initialization in board files  
❌ Use different patterns for different frameworks  

## Testing Frameworks

Frameworks are easy to test due to their class method design:

```python
import unittest
from mpos import MyFramework

class TestMyFramework(unittest.TestCase):
    def setUp(self):
        """Reset framework state before each test."""
        MyFramework._initialized = False
        MyFramework._instance_data = {}
    
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
```

## See Also

- [AppManager](../frameworks/app-manager.md): App discovery, installation, and launching
- [System Components](system-components.md): Overview of all system components
- [Architecture Overview](overview.md): High-level design principles
- [App Development Guide](../apps/index.md): How to use frameworks in apps
