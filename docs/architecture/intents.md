# Intents in MicroPythonOS

## Overview

Intents are a core messaging mechanism in MicroPythonOS that enable inter-activity communication and app navigation. Inspired by Android's Intent system, they provide a flexible, decoupled way for activities to communicate and launch other activities without direct dependencies.

An Intent is essentially a request to perform an action. It can be:

- **Explicit**: Directly targeting a specific activity class
- **Implicit**: Specifying an action that the system resolves to one or more activities

Intents are the primary mechanism for:

- Navigating between activities
- Passing data between activities
- Receiving results from launched activities
- Enabling app extensibility through action-based routing

## Intent Types

### Explicit Intents

An explicit intent directly specifies the target activity class. Use explicit intents when you know exactly which activity should handle the request.

**Characteristics:**

- Direct activity targeting
- Predictable behavior
- Used for internal app navigation
- No ambiguity in routing

**Example:**

```python
from mpos import Intent, Activity

class HomeActivity(Activity):
    def on_settings_click(self):
        # Create explicit intent targeting SettingsActivity
        intent = Intent(activity_class=SettingsActivity)
        self.startActivity(intent)

class SettingsActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text("Settings")
        self.setContentView(screen)
```

### Implicit Intents

An implicit intent specifies an action without naming a specific activity. The system resolves which activity (or activities) can handle the action.

**Characteristics:**

- Action-based routing
- Extensible across apps
- Multiple handlers possible
- Automatic chooser UI for multiple handlers

**Example:**

```python
from mpos import Intent, AppManager

# App 1: Register handler for SEND action
AppManager.register_activity("android.intent.action.SEND", ShareActivity)

# App 2: Register another handler for SEND action
AppManager.register_activity("android.intent.action.SEND", EmailActivity)

# Sending activity: Create implicit intent
class HomeActivity(Activity):
    def on_share_click(self):
        intent = Intent(action="android.intent.action.SEND")
        intent.putExtra("content", "Check this out!")
        self.startActivity(intent)
        # System shows ChooserActivity if multiple handlers exist
        # Or launches directly if only one handler
```

## Intent Class Reference

### Constructor

```python
Intent(activity_class=None, action=None, data=None, extras=None)
```

**Parameters:**

- `activity_class` (class, optional): Target activity class for explicit intents
- `action` (str, optional): Action identifier for implicit intents (e.g., `"android.intent.action.SEND"`)
- `data` (any, optional): Primary data payload (single item)
- `extras` (dict, optional): Additional key-value data

**Example:**

```python
# Explicit intent
intent1 = Intent(activity_class=DetailActivity)

# Implicit intent
intent2 = Intent(action="android.intent.action.SEND")

# With data
intent3 = Intent(activity_class=DetailActivity, data="file.txt")

# With extras
intent4 = Intent(activity_class=DetailActivity, extras={"id": 123})
```

### Attributes

| Attribute | Type | Purpose | Example |
|-----------|------|---------|---------|
| `activity_class` | class or None | Target activity for explicit intents | `SettingsActivity` |
| `action` | str or None | Action identifier for implicit intents | `"android.intent.action.SEND"` |
| `data` | any or None | Primary data payload | URL, file path, or any object |
| `extras` | dict | Additional key-value data | `{"item_id": 42, "title": "Details"}` |
| `flags` | dict | Navigation behavior modifiers | `{"clear_top": True, "no_animation": False}` |

### Methods

#### `putExtra(key, value)`

Add a key-value pair to the extras dictionary. Supports method chaining.

**Parameters:**

- `key` (str): Key name
- `value` (any): Value (any Python object)

**Returns:** `self` (for method chaining)

**Example:**

```python
intent = Intent(activity_class=DetailActivity)
intent.putExtra("item_id", 123)
intent.putExtra("title", "Item Details")
intent.putExtra("data", {"nested": "object"})

# Or use method chaining
intent = Intent(activity_class=DetailActivity) \
    .putExtra("item_id", 123) \
    .putExtra("title", "Item Details")
```

#### `addFlag(flag, value=True)`

Add or modify navigation flags. Supports method chaining.

**Parameters:**

- `flag` (str): Flag name
- `value` (bool, optional): Flag value (default: True)

**Returns:** `self` (for method chaining)

**Supported Flags:**

- `"clear_top"`: Clear all activities above the target
- `"no_history"`: Don't add to activity stack
- `"no_animation"`: Disable transition animation

**Example:**

```python
intent = Intent(activity_class=DetailActivity)
intent.addFlag("clear_top", True)
intent.addFlag("no_animation", False)

# Or use method chaining
intent = Intent(activity_class=DetailActivity) \
    .addFlag("clear_top") \
    .addFlag("no_animation", False)
```

## Intent Lifecycle

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Intent Execution Flow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Create Intent                                               │
│     ├─ Explicit: Intent(activity_class=TargetActivity)         │
│     └─ Implicit: Intent(action="com.example.ACTION")           │
│                                                                  │
│  2. Add Data (Optional)                                         │
│     ├─ putExtra("key", value)                                  │
│     ├─ addFlag("flag_name", value)                             │
│     └─ Set data attribute                                       │
│                                                                  │
│  3. Start Activity                                              │
│     ├─ startActivity(intent)        → Fire and forget          │
│     └─ startActivityForResult(intent, callback) → Expect result│
│                                                                  │
│  4. ActivityNavigator Processing                                │
│     ├─ Validate Intent type                                    │
│     ├─ If implicit: Resolve activity via AppManager            │
│     │  ├─ Single handler: Launch directly                      │
│     │  ├─ Multiple handlers: Show ChooserActivity              │
│     │  └─ No handlers: Print warning, return                   │
│     └─ If explicit: Launch directly                            │
│                                                                  │
│  5. Activity Launch                                             │
│     ├─ Instantiate activity class                              │
│     ├─ Set activity.intent = intent                            │
│     ├─ Set activity._result_callback (if startActivityForResult)
│     ├─ Call activity.onCreate()                                │
│     └─ Return activity instance                                │
│                                                                  │
│  6. Receiving Activity                                          │
│     ├─ getIntent() → Retrieve Intent object                    │
│     ├─ intent.extras.get("key") → Access data                 │
│     └─ setResult() + finish() → Return result (if needed)      │
│                                                                  │
│  7. Result Callback (if startActivityForResult)                │
│     ├─ Activity calls finish()                                 │
│     ├─ Result callback invoked with result dict                │
│     └─ Callback receives: {"result_code": code, "data": dict}  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Activity Stack Evolution

```
Initial State:
┌─────────────┐
│  Launcher   │  ← Foreground
└─────────────┘

After startActivity(SettingsActivity):
┌─────────────┐
│  Settings   │  ← Foreground (onCreate, onStart, onResume called)
├─────────────┤
│  Launcher   │  ← Background (onPause, onStop called)
└─────────────┘

After startActivity(DetailActivity):
┌─────────────┐
│  Detail     │  ← Foreground (onCreate, onStart, onResume called)
├─────────────┤
│  Settings   │  ← Background (onPause, onStop called)
├─────────────┤
│  Launcher   │  ← Background
└─────────────┘

After finish() in DetailActivity:
┌─────────────┐
│  Settings   │  ← Foreground (onResume called)
├─────────────┤
│  Launcher   │  ← Background
└─────────────┘
(DetailActivity destroyed, onDestroy called)
```

## ActivityNavigator - Intent Routing Engine

The `ActivityNavigator` is the core routing engine that processes intents and launches activities. It handles both explicit and implicit intent resolution.

**Location:** [`MicroPythonOS/internal_filesystem/lib/mpos/activity_navigator.py`](MicroPythonOS/internal_filesystem/lib/mpos/activity_navigator.py)

### `startActivity(intent)`

Launch an activity without expecting a result.

**Parameters:**

- `intent` (Intent): The intent to process

**Behavior:**

- Validates intent type
- For implicit intents: resolves handlers via AppManager
- For explicit intents: launches directly
- Shows ChooserActivity if multiple handlers exist

**Example:**

```python
from mpos import Intent, ActivityNavigator

intent = Intent(activity_class=SettingsActivity)
ActivityNavigator.startActivity(intent)

# Or from within an activity
class HomeActivity(Activity):
    def on_settings_click(self):
        intent = Intent(activity_class=SettingsActivity)
        self.startActivity(intent)  # Delegates to ActivityNavigator
```

### `startActivityForResult(intent, result_callback)`

Launch an activity and receive a result via callback.

**Parameters:**

- `intent` (Intent): The intent to process
- `result_callback` (callable): Function to invoke when activity finishes

**Callback Signature:**

```python
def result_callback(result):
    # result is a dict with:
    # - "result_code": str (e.g., "OK", "CANCEL")
    # - "data": dict (result data)
    pass
```

**Example:**

```python
from mpos import Intent, ActivityNavigator

def on_photo_selected(result):
    if result["result_code"] == "OK":
        photo_path = result["data"].get("photo_path")
        print(f"Selected photo: {photo_path}")

intent = Intent(activity_class=GalleryActivity)
ActivityNavigator.startActivityForResult(intent, on_photo_selected)

# Or from within an activity
class HomeActivity(Activity):
    def on_gallery_click(self):
        intent = Intent(activity_class=GalleryActivity)
        self.startActivityForResult(intent, self.on_photo_selected)
    
    def on_photo_selected(self, result):
        if result["result_code"] == "OK":
            photo_path = result["data"].get("photo_path")
```

## AppManager Intent Resolution

The `AppManager` maintains a registry of activities and their associated actions, enabling implicit intent resolution.

**Location:** [`MicroPythonOS/internal_filesystem/lib/mpos/content/app_manager.py`](MicroPythonOS/internal_filesystem/lib/mpos/content/app_manager.py)

### `register_activity(action, activity_cls)`

Register an activity to handle a specific action.

**Parameters:**

- `action` (str): Action identifier (e.g., `"android.intent.action.SEND"`)
- `activity_cls` (type): Activity class

**Example:**

```python
from mpos import AppManager

# Register handlers for SEND action
AppManager.register_activity("android.intent.action.SEND", ShareActivity)
AppManager.register_activity("android.intent.action.SEND", EmailActivity)

# Register handler for VIEW action
AppManager.register_activity("android.intent.action.VIEW", BrowserActivity)
```

### `resolve_activity(intent)`

Find all activities that handle an intent's action.

**Parameters:**

- `intent` (Intent): Intent with action attribute

**Returns:** List of Activity classes (empty if no handlers)

**Example:**

```python
from mpos import Intent, AppManager

intent = Intent(action="android.intent.action.SEND")
handlers = AppManager.resolve_activity(intent)

if len(handlers) == 0:
    print("No handler for SEND action")
elif len(handlers) == 1:
    print(f"Single handler: {handlers[0]}")
else:
    print(f"Multiple handlers: {handlers}")
```

### `query_intent_activities(intent)`

Android-compatible alias for `resolve_activity()`. Identical behavior.

**Parameters:**

- `intent` (Intent): Intent with action attribute

**Returns:** List of Activity classes

**Example:**

```python
from mpos import Intent, AppManager

intent = Intent(action="android.intent.action.SEND")
handlers = AppManager.query_intent_activities(intent)
```

## Common Patterns

### Pattern 1: Simple Navigation

Navigate to a known activity within the app using an explicit intent.

```python
from mpos import Intent, Activity

class HomeActivity(Activity):
    def on_settings_click(self):
        intent = Intent(activity_class=SettingsActivity)
        self.startActivity(intent)

class SettingsActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text("Settings")
        self.setContentView(screen)
```

**Key Points:**

- Direct activity targeting
- No data passing needed
- Fire-and-forget navigation

---

### Pattern 2: Data Passing with Extras

Pass data to another activity using intent extras.

```python
from mpos import Intent, Activity

class ListActivity(Activity):
    def on_item_click(self, item_id, title):
        intent = Intent(activity_class=DetailActivity)
        intent.putExtra("item_id", item_id)
        intent.putExtra("title", title)
        self.startActivity(intent)

class DetailActivity(Activity):
    def onCreate(self):
        intent = self.getIntent()
        item_id = intent.extras.get("item_id")
        title = intent.extras.get("title")
        
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text(f"{title} (ID: {item_id})")
        self.setContentView(screen)
```

**Key Points:**
- Use `putExtra()` for method chaining
- Access via `intent.extras.get(key)`
- Supports any Python object

---

### Pattern 3: Result Handling with Callbacks

Get data back from another activity using `startActivityForResult()`.

```python
from mpos import Intent, Activity

class PhotoEditorActivity(Activity):
    def on_gallery_click(self):
        intent = Intent(activity_class=GalleryActivity)
        self.startActivityForResult(intent, self.on_photo_selected)
    
    def on_photo_selected(self, result):
        if result["result_code"] == "OK":
            photo_path = result["data"].get("photo_path")
            self.display_photo(photo_path)
        elif result["result_code"] == "CANCEL":
            print("User cancelled photo selection")
    
    def display_photo(self, photo_path):
        # Display the selected photo
        pass

class GalleryActivity(Activity):
    def on_photo_tap(self, photo_path):
        self.setResult("OK", {"photo_path": photo_path})
        self.finish()
    
    def on_cancel_click(self):
        self.setResult("CANCEL", {})
        self.finish()
```

**Key Points:**
- Callback receives `{"result_code": code, "data": dict}`
- Must call `setResult()` before `finish()`
- Callback only invoked if result is set

---

### Pattern 4: Implicit Intent with Action

Allow multiple apps to handle an action using implicit intents.

```python
from mpos import Intent, AppManager, Activity

# In App 1: Register handler
AppManager.register_activity("android.intent.action.SEND", ShareActivity)

# In App 2: Register handler
AppManager.register_activity("android.intent.action.SEND", EmailActivity)

# Sending activity
class HomeActivity(Activity):
    def on_share_click(self):
        intent = Intent(action="android.intent.action.SEND")
        intent.putExtra("content", "Check this out!")
        self.startActivity(intent)
        # If multiple handlers: ChooserActivity shown
        # If single handler: Launched directly
        # If no handlers: Warning printed
```

**Key Points:**
- Action-based routing
- Automatic handler resolution
- ChooserActivity for multiple handlers
- Extensible across apps

---

### Pattern 5: Method Chaining

Use fluent API for readable intent construction.

```python
from mpos import Intent, Activity

class HomeActivity(Activity):
    def on_detail_click(self, item_id):
        intent = Intent(activity_class=DetailActivity) \
            .putExtra("id", item_id) \
            .putExtra("title", "Item Details") \
            .addFlag("no_animation", True)
        
        self.startActivity(intent)
```

**Key Points:**

- Both `putExtra()` and `addFlag()` return `self`
- Enables readable, chainable API
- Reduces boilerplate

---

### Pattern 6: Complex Data Passing

Pass complex objects and references between activities.

```python
from mpos import Intent, Activity

class AppStoreActivity(Activity):
    def show_app_detail(self, app):
        intent = Intent(activity_class=AppDetailActivity)
        intent.putExtra("app", app)
        intent.putExtra("appstore", self)
        self.startActivity(intent)

class AppDetailActivity(Activity):
    def onCreate(self):
        intent = self.getIntent()
        self.app = intent.extras.get("app")
        self.appstore = intent.extras.get("appstore")
        
        # Use app and appstore objects
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text(f"App: {self.app.name}")
        self.setContentView(screen)
```

**Key Points:**

- Complex data structures supported
- Can pass activity references
- Useful for passing configuration objects

## Real-World Examples

### Example 1: Settings App Navigation

From [`MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.settings/assets/settings.py`](MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.settings/assets/settings.py):

```python
from mpos import Intent, Activity

class SettingsActivity(Activity):
    def navigate_to_calibration(self):
        from calibrate_imu import CalibrateIMUActivity
        
        intent = Intent(activity_class=CalibrateIMUActivity)
        self.startActivity(intent)
    
    def reset_to_bootloader(self):
        from reset import ResetIntoBootloader
        
        intent = Intent(activity_class=ResetIntoBootloader)
        self.startActivity(intent)
```

**Pattern:** Explicit intents for internal navigation within settings app.

---

### Example 2: WiFi App with Result Handling

From [`MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.wifi/assets/wifi.py`](MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.wifi/assets/wifi.py):

```python
from mpos import Intent, Activity

class WiFiActivity(Activity):
    def add_network_callback(self):
        intent = Intent(activity_class=EditNetwork)
        intent.putExtra("selected_ssid", None)
        self.startActivityForResult(intent, self.edit_network_result_callback)
    
    def select_ssid_cb(self, ssid):
        intent = Intent(activity_class=EditNetwork)
        intent.putExtra("selected_ssid", ssid)
        intent.putExtra("known_password", WifiService.get_network_password(ssid))
        intent.putExtra("hidden", WifiService.get_network_hidden(ssid))
        self.startActivityForResult(intent, self.edit_network_result_callback)
    
    def edit_network_result_callback(self, result):
        if result["result_code"] == "OK":
            network_config = result["data"].get("network_config")
            self.save_network(network_config)

class EditNetwork(Activity):
    def onCreate(self):
        intent = self.getIntent()
        self.selected_ssid = intent.extras.get("selected_ssid")
        self.known_password = intent.extras.get("known_password")
        self.known_hidden = intent.extras.get("hidden", False)
        
        # Build UI with pre-filled values
```

**Pattern:** Data passing with `startActivityForResult()` for configuration workflows.

---

### Example 3: Camera QR Scanning

From [`MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.wifi/assets/wifi.py`](MicroPythonOS/internal_filesystem/builtin/apps/com.micropythonos.wifi/assets/wifi.py):

```python
from mpos import Intent, Activity

class WiFiActivity(Activity):
    def open_camera_for_qr(self):
        self.startActivityForResult(
            Intent(activity_class=CameraActivity)
                .putExtra("scanqr_intent", True),
            self.gotqr_result_callback
        )
    
    def gotqr_result_callback(self, result):
        if result["result_code"] == "OK":
            qr_data = result["data"].get("qr_data")
            self.process_qr_code(qr_data)
```

**Pattern:** Method chaining with `startActivityForResult()` for specialized camera modes.

## Best Practices

### 1. Choose the Right Intent Type

**Use Explicit Intents when:**

- Navigating within your app
- You know the exact target activity
- You want predictable, direct navigation

**Use Implicit Intents when:**

- You want to enable app extensibility
- Multiple apps might handle the action
- You want to decouple from specific implementations

```python
# Good: Explicit for internal navigation
intent = Intent(activity_class=SettingsActivity)
self.startActivity(intent)

# Good: Implicit for extensible actions
intent = Intent(action="android.intent.action.SEND")
self.startActivity(intent)
```

---

### 2. Always Check Result Codes

When using `startActivityForResult()`, always check the result code before accessing data.

```python
# Good: Check result code first
def on_result(self, result):
    if result["result_code"] == "OK":
        data = result["data"].get("key")
    elif result["result_code"] == "CANCEL":
        print("User cancelled")

# Avoid: Assuming success
def on_result(self, result):
    data = result["data"].get("key")  # May fail if cancelled
```

---

### 3. Use Meaningful Action Names

For implicit intents, use descriptive action names that clearly indicate the operation.

```python
# Good: Clear, descriptive action names
Intent(action="android.intent.action.SEND")
Intent(action="android.intent.action.VIEW")
Intent(action="com.example.app.EDIT_PROFILE")

# Avoid: Vague action names
Intent(action="do_something")
Intent(action="action1")
```

---

### 4. Pass Only Serializable Data

While intents can pass any Python object, prefer simple, serializable data types.

```python
# Good: Simple, serializable data
intent.putExtra("item_id", 123)
intent.putExtra("title", "Item Title")
intent.putExtra("config", {"key": "value"})

# Acceptable: Complex objects when necessary
intent.putExtra("app", app_object)

# Avoid: Large objects or circular references
intent.putExtra("large_data", huge_list)
```

---

### 5. Handle Missing Extras Gracefully

Always use `.get()` with default values when accessing extras.

```python
# Good: Use get() with defaults
item_id = intent.extras.get("item_id", 0)
title = intent.extras.get("title", "Untitled")

# Avoid: Direct access without defaults
item_id = intent.extras["item_id"]  # KeyError if missing
```

---

### 6. Register Activities Early

Register implicit intent handlers during app initialization.

```python
# In app initialization
from mpos import AppManager

AppManager.register_activity("android.intent.action.SEND", ShareActivity)
AppManager.register_activity("android.intent.action.VIEW", BrowserActivity)
```

---

### 7. Use Method Chaining for Readability

Leverage method chaining to make intent construction more readable.

```python
# Good: Method chaining
intent = Intent(activity_class=DetailActivity) \
    .putExtra("id", 123) \
    .putExtra("title", "Details") \
    .addFlag("no_animation", True)

# Less readable: Multiple statements
intent = Intent(activity_class=DetailActivity)
intent.putExtra("id", 123)
intent.putExtra("title", "Details")
intent.addFlag("no_animation", True)
```

---

### 8. Clean Up Result Callbacks

Result callbacks are automatically cleaned up after invocation, but ensure your callback doesn't hold unnecessary references.

```python
# Good: Callback doesn't hold unnecessary state
def on_result(self, result):
    if result["result_code"] == "OK":
        self.process_data(result["data"])

# Avoid: Callback holding large state
def on_result(self, result):
    self.large_cache = result["data"]  # Unnecessary retention
```

## Comparison with Android

MicroPythonOS Intents are inspired by Android's Intent system but simplified for embedded environments:

| Feature | MicroPythonOS | Android | Notes |
|---------|---------------|---------|-------|
| **Explicit Intents** | ✅ Supported | ✅ Supported | Direct activity targeting |
| **Implicit Intents** | ✅ Supported | ✅ Supported | Action-based routing |
| **Intent Extras** | ✅ Dict-based | ✅ Bundle-based | MicroPythonOS simpler |
| **Intent Flags** | ⚠️ Partial | ✅ Full | Limited flag support |
| **Intent Filters** | ❌ Programmatic only | ✅ In manifest | No manifest-based registration |
| **Categories** | ❌ Not supported | ✅ Supported | Simplified routing |
| **Data Types** | ❌ Not matched | ✅ MIME types | No type filtering |
| **URI Schemes** | ❌ Not matched | ✅ Supported | No scheme filtering |
| **Chooser UI** | ✅ ChooserActivity | ✅ Intent chooser | Custom implementation |
| **Result Callbacks** | ✅ Callback-based | ✅ onActivityResult() | Different mechanism |
| **Method Chaining** | ✅ putExtra() returns self | ❌ No chaining | MicroPythonOS advantage |

### Key Differences

**Simplified Intent Filters:**

- MicroPythonOS uses programmatic registration via `AppManager.register_activity()`
- Android uses manifest-based intent filters
- MicroPythonOS approach is more flexible for dynamic app loading

**Callback-Based Results:**

- MicroPythonOS uses callbacks: `startActivityForResult(intent, callback)`
- Android uses `onActivityResult()` lifecycle method
- MicroPythonOS approach fits better with async/event-driven model

**Method Chaining:**

- MicroPythonOS `putExtra()` and `addFlag()` return `self`
- Android methods don't support chaining
- MicroPythonOS provides more fluent API

## Source Files

The Intent system is implemented across these core files:

- **Intent Class:** [`MicroPythonOS/internal_filesystem/lib/mpos/content/intent.py`](MicroPythonOS/internal_filesystem/lib/mpos/content/intent.py)
- **ActivityNavigator:** [`MicroPythonOS/internal_filesystem/lib/mpos/activity_navigator.py`](MicroPythonOS/internal_filesystem/lib/mpos/activity_navigator.py)
- **AppManager:** [`MicroPythonOS/internal_filesystem/lib/mpos/content/app_manager.py`](MicroPythonOS/internal_filesystem/lib/mpos/content/app_manager.py)
- **Activity Integration:** [`MicroPythonOS/internal_filesystem/lib/mpos/app/activity.py`](MicroPythonOS/internal_filesystem/lib/mpos/app/activity.py)

## Related Documentation

- [App Lifecycle](../apps/app-lifecycle.md) - Activity lifecycle and Intent basics
- [AppManager](../frameworks/app-manager.md) - Intent resolution and activity registration
- [Activity](../frameworks/activity.md) - Activity class and lifecycle methods
- [SettingActivity](../frameworks/setting-activity.md) - Intent extras for settings configuration
