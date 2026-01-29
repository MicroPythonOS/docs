# App Lifecycle

MicroPythonOS uses an Android-inspired Activity lifecycle model. Understanding this lifecycle is essential for building apps that behave correctly when users navigate between screens, switch apps, or return to the launcher.

## Overview

Every app in MicroPythonOS is built around **Activities**. An Activity represents a single screen with a user interface. The system manages a stack of activities and calls specific lifecycle methods as activities are created, started, paused, resumed, stopped, and destroyed.

```
┌─────────────────────────────────────────────────────────────┐
│                    Activity Lifecycle                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    ┌──────────┐                                              │
│    │ onCreate │  ← Activity instantiated, build UI here      │
│    └────┬─────┘                                              │
│         │                                                    │
│         ▼                                                    │
│    ┌──────────┐                                              │
│    │ onStart  │  ← Screen about to become visible            │
│    └────┬─────┘                                              │
│         │                                                    │
│         ▼                                                    │
│    ┌──────────┐                                              │
│    │ onResume │  ← Activity in foreground, user can interact │
│    └────┬─────┘                                              │
│         │                                                    │
│         │ ◄──────────────────────────────────────────┐       │
│         │                                            │       │
│         ▼                                            │       │
│    ┌──────────┐                                      │       │
│    │ onPause  │  ← Another activity coming to front  │       │
│    └────┬─────┘                                      │       │
│         │                                            │       │
│         ▼                                            │       │
│    ┌──────────┐                                      │       │
│    │ onStop   │  ← Activity no longer visible        │       │
│    └────┬─────┘                                      │       │
│         │                                            │       │
│         ├─────────────────────────────────────────────┘       │
│         │  (user navigates back)                             │
│         │                                                    │
│         ▼                                                    │
│    ┌───────────┐                                             │
│    │ onDestroy │  ← Activity being removed from stack        │
│    └───────────┘                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Lifecycle Methods

### onCreate()

Called when the activity is first created. This is where you should:

- Create your UI (LVGL widgets)
- Initialize instance variables
- Set up the content view with `setContentView()`

```python
from mpos import Activity
import lvgl as lv

class MyActivity(Activity):
    def onCreate(self):
        # Create the screen
        screen = lv.obj()
        
        # Add UI elements
        label = lv.label(screen)
        label.set_text("Hello World!")
        label.center()
        
        # Activate the screen
        self.setContentView(screen)
```

**Important:** Always call `setContentView()` at the end of `onCreate()` to display your screen.

### onStart(screen)

Called when the activity is about to become visible. The `screen` parameter is the LVGL screen object you created.

Use this for:

- Preparing resources needed for display
- Starting animations that should play when the screen appears

```python
def onStart(self, screen):
    print("Activity is starting...")
```

### onResume(screen)

Called when the activity enters the foreground and becomes interactive. This is called:

- After `onStart()` when the activity first appears
- When returning to this activity from another activity (e.g., user presses back)

Use this for:

- Registering callbacks (network, sensors, etc.)
- Starting background tasks
- Refreshing data that may have changed

```python
def onResume(self, screen):
    super().onResume(screen)  # Sets _has_foreground = True
    
    # Register for network changes
    self.connectivity_manager = ConnectivityManager.get()
    self.connectivity_manager.register_callback(self.on_network_change)
    
    # Refresh data
    self.refresh_app_list()
```

**Important:** Call `super().onResume(screen)` to properly track foreground state.

### onPause(screen)

Called when the activity is about to lose focus (another activity is coming to the foreground).

Use this for:

- Unregistering callbacks
- Pausing ongoing operations
- Saving temporary state

```python
def onPause(self, screen):
    # Unregister callbacks
    if self.connectivity_manager:
        self.connectivity_manager.unregister_callback(self.on_network_change)
    
    super().onPause(screen)  # Sets _has_foreground = False
```

**Important:** Call `super().onPause(screen)` to properly track foreground state.

### onStop(screen)

Called when the activity is no longer visible (fully covered by another activity).

Use this for:

- Releasing resources that aren't needed while hidden
- Stopping expensive operations

```python
def onStop(self, screen):
    # Stop any expensive background work
    self.cancel_pending_downloads()
```

### onDestroy(screen)

Called when the activity is being removed from the activity stack (user navigated back past this activity).

Use this for:

- Final cleanup
- Releasing all resources

```python
def onDestroy(self, screen):
    # Clean up resources
    self.cleanup_temp_files()
```

## Activity Stack

MicroPythonOS maintains a stack of activities. When you start a new activity, it's pushed onto the stack. When the user navigates back, the top activity is popped and destroyed.

```
Activity Stack:
┌─────────────────┐
│  SettingsDetail │  ← Top (visible, has focus)
├─────────────────┤
│    Settings     │  ← Paused, stopped
├─────────────────┤
│    Launcher     │  ← Paused, stopped
└─────────────────┘
```

### Navigation Flow

1. **Starting a new activity:** Current activity receives `onPause()` → `onStop()`, new activity receives `onCreate()` → `onStart()` → `onResume()`

2. **Going back:** Current activity receives `onPause()` → `onStop()` → `onDestroy()`, previous activity receives `onResume()`

## Starting Activities

### Basic Navigation

Use `startActivity()` to navigate to another activity:

```python
from mpos import Intent

class MyActivity(Activity):
    def on_settings_click(self):
        intent = Intent(activity_class=SettingsActivity)
        self.startActivity(intent)
```

### Passing Data with Intents

Use Intent extras to pass data between activities:

```python
# Sending activity
def on_item_click(self, item_id):
    intent = Intent(activity_class=DetailActivity)
    intent.putExtra("item_id", item_id)
    intent.putExtra("title", "Item Details")
    self.startActivity(intent)

# Receiving activity
class DetailActivity(Activity):
    def onCreate(self):
        intent = self.getIntent()
        item_id = intent.extras.get("item_id")
        title = intent.extras.get("title")
        # Use the data...
```

### Getting Results from Activities

Use `startActivityForResult()` when you need a result back:

```python
class PhotoPickerActivity(Activity):
    def on_gallery_click(self):
        intent = Intent(activity_class=GalleryActivity)
        self.startActivityForResult(intent, self.on_photo_selected)
    
    def on_photo_selected(self, result):
        if result["result_code"] == "OK":
            photo_path = result["data"].get("photo_path")
            self.display_photo(photo_path)

# In GalleryActivity
class GalleryActivity(Activity):
    def on_photo_tap(self, photo_path):
        self.setResult("OK", {"photo_path": photo_path})
        self.finish()
```

### Finishing an Activity

Call `finish()` to close the current activity and return to the previous one:

```python
def on_done_click(self):
    # Optionally set a result first
    self.setResult("OK", {"selected_item": self.selected})
    self.finish()
```

## Foreground State

Activities track whether they're in the foreground. This is useful for:

- Canceling operations when the user navigates away
- Avoiding UI updates when the activity isn't visible

### Checking Foreground State

```python
def on_data_loaded(self, data):
    # Only update UI if we're still in the foreground
    if self.has_foreground():
        self.update_list(data)
```

### Conditional Execution

Use `if_foreground()` to execute code only when in foreground:

```python
def on_download_complete(self, data):
    self.if_foreground(self.show_download_result, data)
```

### Thread-Safe UI Updates

When updating UI from a background thread, use `update_ui_threadsafe_if_foreground()`:

```python
def background_task(self):
    # Running in a separate thread
    result = self.fetch_data()
    
    # Safely update UI on main thread, only if in foreground
    self.update_ui_threadsafe_if_foreground(
        self.display_result, 
        result
    )
```

## Complete Example

Here's a complete example showing proper lifecycle management:

```python
import lvgl as lv
from mpos import Activity, ConnectivityManager, TaskManager

class NetworkAwareActivity(Activity):
    
    def __init__(self):
        super().__init__()
        self.connectivity_manager = None
        self.data = None
    
    def onCreate(self):
        # Create UI
        self.screen = lv.obj()
        
        self.status_label = lv.label(self.screen)
        self.status_label.set_text("Loading...")
        self.status_label.center()
        
        self.refresh_button = lv.button(self.screen)
        self.refresh_button.align(lv.ALIGN.BOTTOM_MID, 0, -20)
        self.refresh_button.add_event_cb(
            lambda e: self.refresh_data(),
            lv.EVENT.CLICKED, None
        )
        btn_label = lv.label(self.refresh_button)
        btn_label.set_text("Refresh")
        
        self.setContentView(self.screen)
    
    def onResume(self, screen):
        super().onResume(screen)
        
        # Register for network changes
        self.connectivity_manager = ConnectivityManager.get()
        self.connectivity_manager.register_callback(self.on_network_change)
        
        # Initial data load
        if self.connectivity_manager.is_online():
            self.refresh_data()
        else:
            self.status_label.set_text("Waiting for network...")
    
    def onPause(self, screen):
        # Unregister callbacks
        if self.connectivity_manager:
            self.connectivity_manager.unregister_callback(self.on_network_change)
        
        super().onPause(screen)
    
    def on_network_change(self, online):
        if online and self.has_foreground():
            self.refresh_data()
        elif not online:
            self.status_label.set_text("Network disconnected")
    
    def refresh_data(self):
        self.status_label.set_text("Loading...")
        TaskManager.create_task(self.fetch_data_async())
    
    async def fetch_data_async(self):
        try:
            # Simulate network request
            await TaskManager.sleep(1)
            self.data = {"items": ["Item 1", "Item 2", "Item 3"]}
            
            # Update UI only if still in foreground
            if self.has_foreground():
                self.status_label.set_text(f"Loaded {len(self.data['items'])} items")
        except Exception as e:
            if self.has_foreground():
                self.status_label.set_text(f"Error: {e}")
```

## Best Practices

### Do's

✅ Always call `super().__init__()` in your `__init__` method  
✅ Call `super().onResume(screen)` and `super().onPause(screen)` to track foreground state  
✅ Register callbacks in `onResume()` and unregister in `onPause()`  
✅ Check `has_foreground()` before updating UI from async operations  
✅ Use `setContentView()` at the end of `onCreate()`  
✅ Clean up resources in `onDestroy()`  

### Don'ts

❌ Don't do heavy work in `onCreate()` - it blocks the UI  
❌ Don't forget to unregister callbacks - causes memory leaks  
❌ Don't update UI from background threads without `update_ui_threadsafe_if_foreground()`  
❌ Don't assume the activity is still visible after async operations  
❌ Don't store references to LVGL objects after `onDestroy()`  

## Lifecycle Comparison with Android

| MicroPythonOS | Android | Notes |
|---------------|---------|-------|
| `onCreate()` | `onCreate()` | Same purpose |
| `onStart(screen)` | `onStart()` | MicroPythonOS passes screen |
| `onResume(screen)` | `onResume()` | MicroPythonOS passes screen |
| `onPause(screen)` | `onPause()` | MicroPythonOS passes screen |
| `onStop(screen)` | `onStop()` | MicroPythonOS passes screen |
| `onDestroy(screen)` | `onDestroy()` | MicroPythonOS passes screen |
| `finish()` | `finish()` | Same purpose |
| `startActivity(intent)` | `startActivity(intent)` | Same purpose |
| `startActivityForResult()` | `startActivityForResult()` | Callback-based in MicroPythonOS |
| `setResult()` | `setResult()` | Same purpose |
| `getIntent()` | `getIntent()` | Same purpose |

## See Also

- [Creating Apps](creating-apps.md) - Getting started with app development
- [AppManager](../frameworks/app-manager.md) - App management and launching
- [Intent System](../frameworks/app-manager.md#intent-resolution) - Intent-based navigation
- [TaskManager](../frameworks/task-manager.md) - Async task management
