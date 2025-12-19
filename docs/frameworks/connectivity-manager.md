# ConnectivityManager

MicroPythonOS provides a connectivity monitoring service called **ConnectivityManager** that tracks network status and notifies apps when connectivity changes. It's inspired by Android's ConnectivityManager API.

## Overview

ConnectivityManager provides:

- **Connectivity monitoring** - Periodic checks of network status
- **Callback notifications** - Get notified when connectivity changes
- **Singleton pattern** - Single instance shared across all apps
- **Platform-agnostic** - Works on ESP32 and desktop
- **Android-like API** - Familiar patterns for Android developers

## Quick Start

### Checking Connectivity

```python
from mpos.net.connectivity_manager import ConnectivityManager

# Get the singleton instance
cm = ConnectivityManager.get()

# Check if online (has internet)
if cm.is_online():
    print("Connected to internet")
else:
    print("No internet connection")

# Check if WiFi is connected (local network)
if cm.is_wifi_connected():
    print("WiFi connected")
```

### Registering for Connectivity Changes

```python
from mpos.app.activity import Activity
from mpos.net.connectivity_manager import ConnectivityManager

class MyActivity(Activity):
    def onCreate(self):
        self.cm = ConnectivityManager.get()
        
    def onResume(self, screen):
        super().onResume(screen)
        # Register for connectivity changes
        self.cm.register_callback(self.on_connectivity_changed)
        
        # Update UI with current status
        self.update_status(self.cm.is_online())
    
    def onPause(self, screen):
        super().onPause(screen)
        # Unregister when activity is paused
        self.cm.unregister_callback(self.on_connectivity_changed)
    
    def on_connectivity_changed(self, is_online):
        """Called when connectivity status changes."""
        if is_online:
            print("Now online!")
            self.fetch_data()
        else:
            print("Gone offline")
            self.show_offline_message()
    
    def update_status(self, is_online):
        status = "Online" if is_online else "Offline"
        self.status_label.set_text(status)
```

### Waiting for Connectivity

```python
from mpos.net.connectivity_manager import ConnectivityManager

def download_with_wait():
    cm = ConnectivityManager.get()
    
    # Wait up to 30 seconds for connectivity
    if cm.wait_until_online(timeout=30):
        print("Connected! Starting download...")
        # Proceed with download
    else:
        print("Timeout waiting for connection")
```

## Common Patterns

### Network-Aware App

```python
from mpos.app.activity import Activity
from mpos.net.connectivity_manager import ConnectivityManager
from mpos import TaskManager, DownloadManager
import lvgl as lv

class NewsReaderActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        self.cm = ConnectivityManager.get()
        
        # Status indicator
        self.status_icon = lv.label(self.screen)
        self.status_icon.align(lv.ALIGN.TOP_RIGHT, -10, 10)
        
        # Content area
        self.content = lv.label(self.screen)
        self.content.set_width(280)
        self.content.align(lv.ALIGN.CENTER, 0, 0)
        
        self.setContentView(self.screen)
    
    def onResume(self, screen):
        super().onResume(screen)
        self.cm.register_callback(self.on_connectivity_changed)
        self.update_ui()
    
    def onPause(self, screen):
        super().onPause(screen)
        self.cm.unregister_callback(self.on_connectivity_changed)
    
    def on_connectivity_changed(self, is_online):
        self.update_ui()
        if is_online:
            TaskManager.create_task(self.fetch_news())
    
    def update_ui(self):
        if self.cm.is_online():
            self.status_icon.set_text(lv.SYMBOL.WIFI)
        else:
            self.status_icon.set_text(lv.SYMBOL.WARNING)
            self.content.set_text("No internet connection")
    
    async def fetch_news(self):
        data = await DownloadManager.download_url(
            "https://api.example.com/news"
        )
        if data:
            self.content.set_text(data.decode())
```

### Retry on Reconnect

```python
from mpos.net.connectivity_manager import ConnectivityManager
from mpos import TaskManager, DownloadManager

class SyncManager:
    def __init__(self):
        self.cm = ConnectivityManager.get()
        self.pending_sync = False
        self.cm.register_callback(self.on_connectivity_changed)
    
    def sync_data(self):
        if self.cm.is_online():
            TaskManager.create_task(self._do_sync())
        else:
            # Mark for retry when online
            self.pending_sync = True
            print("Offline - sync queued")
    
    def on_connectivity_changed(self, is_online):
        if is_online and self.pending_sync:
            print("Back online - retrying sync")
            self.pending_sync = False
            TaskManager.create_task(self._do_sync())
    
    async def _do_sync(self):
        success = await DownloadManager.download_url(
            "https://api.example.com/sync",
            # ... sync data
        )
        if not success:
            self.pending_sync = True
```

### Conditional Feature Loading

```python
from mpos.net.connectivity_manager import ConnectivityManager

class AppStoreActivity(Activity):
    def onCreate(self):
        self.cm = ConnectivityManager.get()
        
        if self.cm.is_online():
            # Load full app store with remote data
            self.load_remote_apps()
        else:
            # Show only installed apps
            self.load_local_apps()
            self.show_offline_banner()
```

## API Reference

### Getting the Instance

#### `ConnectivityManager.get()`

Get the singleton ConnectivityManager instance.

**Returns:**
- `ConnectivityManager` - The singleton instance

**Example:**
```python
cm = ConnectivityManager.get()
```

**Note:** The first call initializes the manager and starts periodic connectivity checks.

### Connectivity Status

#### `is_online()`

Check if the device has internet connectivity.

**Returns:**
- `bool` - `True` if online, `False` otherwise

**Example:**
```python
if cm.is_online():
    print("Internet available")
```

---

#### `is_wifi_connected()`

Check if WiFi is connected (local network).

**Returns:**
- `bool` - `True` if WiFi connected, `False` otherwise

**Note:** A device can be WiFi-connected but not online (e.g., no internet on the network).

**Example:**
```python
if cm.is_wifi_connected():
    print("WiFi connected")
```

---

#### `wait_until_online(timeout=60)`

Block until the device is online or timeout expires.

**Parameters:**
- `timeout` (int) - Maximum seconds to wait (default: 60)

**Returns:**
- `bool` - `True` if online, `False` if timeout expired

**Example:**
```python
if cm.wait_until_online(timeout=30):
    print("Connected!")
else:
    print("Timeout - still offline")
```

### Callback Management

#### `register_callback(callback)`

Register a callback to be notified of connectivity changes.

**Parameters:**
- `callback` (callable) - Function that takes one boolean parameter (`is_online`)

**Example:**
```python
def on_change(is_online):
    print(f"Connectivity: {'online' if is_online else 'offline'}")

cm.register_callback(on_change)
```

---

#### `unregister_callback(callback)`

Unregister a previously registered callback.

**Parameters:**
- `callback` (callable) - The callback to remove

**Example:**
```python
cm.unregister_callback(on_change)
```

**Important:** Always unregister callbacks when your activity is paused or destroyed to prevent memory leaks and stale references.

## Monitoring Behavior

ConnectivityManager uses a periodic timer to check connectivity:

| Setting | Value |
|---------|-------|
| Check interval | 8 seconds |
| Timer ID | 1 (Timer 0 is used by task_handler) |
| Check method | `wlan.isconnected()` |

### How Connectivity is Determined

1. **ESP32/Hardware:**
   - Uses `network.WLAN(network.STA_IF).isconnected()`
   - Returns `True` if WiFi is connected to an access point

2. **Desktop/Linux:**
   - No network module available
   - Always reports as "connected" for testing

### Callback Timing

- Callbacks are only called when status **changes**
- Initial status is checked at startup (no callback)
- Callbacks receive `True` when going online, `False` when going offline

```python
# Example callback sequence:
# Boot: offline (no callback - initial state)
# WiFi connects: callback(True)
# WiFi disconnects: callback(False)
# WiFi reconnects: callback(True)
```

## Platform Differences

| Platform | Behavior |
|----------|----------|
| **ESP32** | Real WiFi monitoring via `network` module |
| **Desktop** | Always reports online (no network module) |

### Desktop Testing

On desktop, ConnectivityManager always reports as connected:

```python
cm = ConnectivityManager.get()
print(cm.is_online())  # Always True on desktop
print(cm.is_wifi_connected())  # Always True on desktop
```

This allows apps to be tested without actual network connectivity.

## Troubleshooting

### Callbacks Not Being Called

**Symptom:** Registered callback never fires

**Possible causes:**
1. Connectivity never changes
2. Callback was unregistered
3. Callback raises exception (silently caught)

**Solution:**
```python
# Check current status
print(f"Current status: {cm.is_online()}")

# Verify callback is registered
print(f"Callbacks: {len(cm.callbacks)}")

# Ensure callback doesn't raise exceptions
def safe_callback(is_online):
    try:
        # Your logic here
        pass
    except Exception as e:
        print(f"Callback error: {e}")

cm.register_callback(safe_callback)
```

### Memory Leak from Callbacks

**Symptom:** Memory usage grows over time

**Cause:** Callbacks not unregistered when activity destroyed

**Solution:**
```python
class MyActivity(Activity):
    def onResume(self, screen):
        super().onResume(screen)
        self.cm = ConnectivityManager.get()
        self.cm.register_callback(self.on_change)
    
    def onPause(self, screen):
        super().onPause(screen)
        # Always unregister!
        self.cm.unregister_callback(self.on_change)
```

### Status Always Shows Online

**Symptom:** `is_online()` returns `True` even when disconnected

**Cause:** Running on desktop (no network module)

**Solution:** This is expected behavior on desktop. Test on real hardware for accurate connectivity status.

### Timer Conflict

**Symptom:** Strange behavior with other timers

**Cause:** ConnectivityManager uses Timer(1)

**Solution:** Use different timer IDs in your app:
```python
from machine import Timer

# ConnectivityManager uses Timer(1)
# task_handler uses Timer(0)
# Use Timer(2) or higher for your app
my_timer = Timer(2)
```

## Integration with WifiService

ConnectivityManager works alongside WifiService:

- **WifiService** - Manages WiFi connections (connect, disconnect, scan)
- **ConnectivityManager** - Monitors connection status and notifies apps

```python
from mpos.net.wifi_service import WifiService
from mpos.net.connectivity_manager import ConnectivityManager

# WifiService for connection management
WifiService.attempt_connecting("MyNetwork", "password")

# ConnectivityManager for status monitoring
cm = ConnectivityManager.get()
cm.register_callback(lambda online: print(f"Status: {online}"))
```

## Implementation Details

**Location:** `MicroPythonOS/internal_filesystem/lib/mpos/net/connectivity_manager.py`

**Pattern:** Singleton with periodic timer

**Key features:**
- `_instance` - Class-level singleton instance
- `callbacks` - List of registered callback functions
- `_check_timer` - Machine Timer for periodic checks
- `_is_online` - Cached online status

**Dependencies:**
- `network` module (MicroPython, optional)
- `machine.Timer` - Periodic connectivity checks
- `requests` / `usocket` - Imported but not currently used for connectivity checks

## See Also

- [WifiService](wifi-service.md) - WiFi connection management
- [DownloadManager](download-manager.md) - HTTP downloads
- [TaskManager](task-manager.md) - Async task management
