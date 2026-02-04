# AppManager

MicroPythonOS provides a centralized app management service called **AppManager**. It handles app discovery, installation, uninstallation, launching, and version management across the system.

## Overview

AppManager provides:

- **App discovery** - Automatically finds and catalogs all installed apps
- **App registry** - Maintains a sorted list of apps with metadata (name, version, icon, etc.)
- **App installation** - Installs apps from `.mpk` (MicroPython Package) ZIP files
- **App uninstallation** - Removes user-installed apps (preserves built-in apps)
- **App launching** - Starts apps with proper environment setup and activity management
- **Version management** - Compares versions and detects available updates
- **Intent resolution** - Resolves activities that handle specific intents (Android-inspired)
- **Launcher management** - Finds and restarts the system launcher

## Quick Start

### Initialization

AppManager is initialized at system startup and automatically discovers all apps:

```python
from mpos import AppManager

# Get list of all installed apps (auto-discovers on first call)
apps = AppManager.get_app_list()

for app in apps:
    print(f"{app.name} v{app.version}")
```

### Listing Apps

```python
from mpos import AppManager

# Get all apps (sorted by name)
all_apps = AppManager.get_app_list()

# Get specific app by fullname
app = AppManager.get("com.example.myapp")

# Access app by fullname (raises KeyError if not found)
try:
    app = AppManager["com.example.myapp"]
except KeyError:
    print("App not found")
```

### Launching Apps

```python
from mpos import AppManager

# Start an app by fullname
success = AppManager.start_app("com.example.myapp")

if success:
    print("App started successfully")
else:
    print("Failed to start app")
```

### Installing Apps

```python
from mpos import AppManager

# Install an app from a .mpk file
# .mpk files are ZIP archives containing the app structure
AppManager.install_mpk(
    temp_zip_path="/tmp/myapp.mpk",
    dest_folder="apps/com.example.myapp"
)

# AppManager automatically refreshes the app list after installation
```

### Uninstalling Apps

```python
from mpos import AppManager

# Uninstall a user-installed app
AppManager.uninstall_app("com.example.myapp")

# Built-in apps cannot be uninstalled
# AppManager automatically refreshes the app list after uninstallation
```

## App Discovery

AppManager discovers apps by scanning two directories:

1. **`builtin/apps/`** - System-provided apps (cannot be uninstalled)
2. **`apps/`** - User-installed apps (can be uninstalled)

User-installed apps override built-in apps with the same fullname.

### Discovery Process

```python
from mpos import AppManager

# Manually trigger app discovery (called automatically on first use)
AppManager.refresh_apps()

# Clear cached app list
AppManager.clear()

# Refresh will repopulate the cache
AppManager.get_app_list()
```

### App Structure

Each app must have this directory structure:

```
apps/com.example.myapp/
├── META-INF/
│   └── MANIFEST.JSON          # App metadata
├── assets/
│   ├── main.py                # Main activity entry point
│   ├── icon.png               # App icon
│   └── ...                    # Other assets
└── ...
```

The `MANIFEST.JSON` file contains app metadata:

```json
{
  "fullname": "com.example.myapp",
  "name": "My App",
  "version": "1.0.0",
  "description": "A sample app",
  "main_launcher_activity": {
    "entrypoint": "assets/main.py",
    "classname": "Main"
  }
}
```

## App Information

Each app object provides metadata:

```python
from mpos.content.app_manager import AppManager

app = AppManager.get("com.example.myapp")

# Access app properties
print(f"Name: {app.name}")
print(f"Fullname: {app.fullname}")
print(f"Version: {app.version}")
print(f"Description: {app.description}")
print(f"Installed path: {app.installed_path}")
print(f"Icon: {app.icon}")
print(f"Main activity: {app.main_launcher_activity}")
```

## Version Management

AppManager provides version comparison utilities:

### Comparing Versions

```python
from mpos import AppManager

# Compare two version strings
# Returns True if ver1 > ver2, False otherwise
is_newer = AppManager.compare_versions("2.0.0", "1.5.0")
# Returns: True

is_newer = AppManager.compare_versions("1.5.0", "2.0.0")
# Returns: False

is_newer = AppManager.compare_versions("1.0.0", "1.0.0")
# Returns: False (equal versions)
```

### Checking for Updates

```python
from mpos import AppManager

# Check if a newer version is available
has_update = AppManager.is_update_available("com.example.myapp", "2.0.0")

if has_update:
    print("Update available!")
```

## Installation Management

### Installing Apps

```python
from mpos import AppManager

# Install from a .mpk file (ZIP archive)
AppManager.install_mpk(
    temp_zip_path="/tmp/downloaded_app.mpk",
    dest_folder="apps/com.example.newapp"
)

# After installation, the app is immediately available
app = AppManager.get("com.example.newapp")
```

### Checking Installation Status

```python
from mpos import AppManager

# Check if app is installed (by name)
is_installed = AppManager.is_installed_by_name("com.example.myapp")

# Check if app is installed at specific path
is_installed = AppManager.is_installed_by_path("apps/com.example.myapp")

# Check if app is built-in
is_builtin = AppManager.is_builtin_app("com.example.myapp")

# Check if app is an override of a built-in app
is_override = AppManager.is_overridden_builtin_app("com.example.myapp")
```

### Uninstalling Apps

```python
from mpos import AppManager

# Uninstall a user-installed app
AppManager.uninstall_app("com.example.myapp")

# Built-in apps cannot be uninstalled
# Attempting to uninstall a built-in app will fail silently
```

## Intent Resolution

AppManager implements Android-inspired intent resolution for activity discovery:

### Registering Activities

Activities register themselves with AppManager to handle specific intents:

```python
from mpos import AppManager
from mpos import Activity

class ShareActivity(Activity):
    pass

# Register activity to handle "android.intent.action.SEND" intent
AppManager.register_activity("android.intent.action.SEND", ShareActivity)
```

### Resolving Intents

```python
from mpos import AppManager, Intent

# Create an intent
intent = Intent(action="android.intent.action.SEND")

# Find all activities that handle this intent
activities = AppManager.resolve_activity(intent)

# Or use the Android-like method name
activities = AppManager.query_intent_activities(intent)

for activity_class in activities:
    print(f"Activity: {activity_class.__name__}")
```

## Launcher Management

AppManager provides utilities for launcher discovery and restart:

### Finding the Launcher

```python
from mpos import AppManager

# Get the system launcher app
launcher = AppManager.get_launcher()

if launcher:
    print(f"Launcher: {launcher.fullname}")
    print(f"Name: {launcher.name}")
```

### Restarting the Launcher

```python
from mpos import AppManager

# Stop all activities and restart the launcher
AppManager.restart_launcher()

# This is useful when:
# - Exiting an app to return to home screen
# - Recovering from a crash
# - Refreshing the app list after installation/uninstallation
```

## App Execution

AppManager handles the low-level execution of app scripts:

### Starting Apps

```python
from mpos import AppManager

# Start an app (high-level interface)
success = AppManager.start_app("com.example.myapp")

if success:
    print("App started")
else:
    print("Failed to start app")
```

### Script Execution

AppManager can execute arbitrary Python scripts with proper environment setup:

```python
from mpos import AppManager

# Execute a script file
success = AppManager.execute_script(
    script_source="assets/main.py",
    is_file=True,
    classname="Main",
    cwd="apps/com.example.myapp/assets/"
)

# Execute inline script
success = AppManager.execute_script(
    script_source="print('Hello')",
    is_file=False,
    classname="Main"
)
```

### Execution Details

When executing a script, AppManager:

1. **Reads the file** (if `is_file=True`)
2. **Compiles the script** to bytecode
3. **Sets up globals** with LVGL and other imports
4. **Executes the script** in the prepared environment
5. **Finds the main activity class** by name
6. **Starts the activity** using the Activity framework
7. **Restores sys.path** to clean up

The execution environment includes:

```python
script_globals = {
    'lv': lv,  # LVGL module
    '__name__': "__main__"
}
```

If a `cwd` is provided, it's added to `sys.path` for relative imports.

## Complete Example: App Store Integration

Here's how the App Store uses AppManager:

```python
from mpos import Activity, AppManager
import lvgl as lv

class AppStoreActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        
        # List all installed apps
        apps = AppManager.get_app_list()
        
        # Create a list view
        list_view = lv.list(self.screen)
        list_view.set_size(lv.pct(100), lv.pct(100))
        
        for app in apps:
            # Create button for each app
            btn = list_view.add_button(lv.SYMBOL.DOWNLOAD, app.name)
            btn.add_event_cb(
                lambda e, a=app: self.on_app_selected(a),
                lv.EVENT.CLICKED,
                None
            )
        
        self.setContentView(self.screen)
    
    def on_app_selected(self, app):
        # Check if update is available
        if AppManager.is_update_available(app.fullname, "2.0.0"):
            print(f"Update available for {app.name}")
            # Download and install update
            # AppManager.install_mpk(...)
        else:
            # Launch the app
            AppManager.start_app(app.fullname)
```

## API Reference

### App Discovery

**`get_app_list()`**

Get list of all installed apps (sorted by name).

- **Returns:** `list[App]` - List of App objects

**`get(fullname)`**

Get app by fullname.

- **Parameters:**
  - `fullname` (str): App fullname (e.g., `"com.example.myapp"`)

- **Returns:** `App` or `None` - App object or None if not found

**`__getitem__(fullname)`**

Get app by fullname (raises KeyError if not found).

- **Parameters:**
  - `fullname` (str): App fullname

- **Returns:** `App` - App object

- **Raises:** `KeyError` - If app not found

**`refresh_apps()`**

Scan directories and rebuild app list.

- **Behavior:** Clears cache and rediscovers all apps

**`clear()`**

Clear cached app list.

- **Behavior:** Empties internal caches; call `get_app_list()` to repopulate

### App Installation

**`install_mpk(temp_zip_path, dest_folder)`**

Install app from .mpk (ZIP) file.

- **Parameters:**
  - `temp_zip_path` (str): Path to temporary .mpk file
  - `dest_folder` (str): Destination folder (e.g., `"apps/com.example.myapp"`)

- **Behavior:** Extracts ZIP, removes temp file, refreshes app list

**`uninstall_app(app_fullname)`**

Uninstall a user-installed app.

- **Parameters:**
  - `app_fullname` (str): App fullname

- **Behavior:** Removes app folder from `apps/`, refreshes app list

- **Note:** Built-in apps cannot be uninstalled

### Version Management

**`compare_versions(ver1, ver2)`**

Compare two version strings.

- **Parameters:**
  - `ver1` (str): First version (e.g., `"1.2.3"`)
  - `ver2` (str): Second version (e.g., `"1.0.0"`)

- **Returns:** `bool` - `True` if ver1 > ver2, `False` otherwise

- **Behavior:** Parses versions as dot-separated integers, compares component-wise

**`is_update_available(app_fullname, new_version)`**

Check if newer version is available.

- **Parameters:**
  - `app_fullname` (str): App fullname
  - `new_version` (str): New version to check

- **Returns:** `bool` - `True` if new_version > installed version

### Installation Status

**`is_installed_by_name(app_fullname)`**

Check if app is installed (user or built-in).

- **Parameters:**
  - `app_fullname` (str): App fullname

- **Returns:** `bool` - `True` if installed

**`is_installed_by_path(dir_path)`**

Check if app is installed at specific path.

- **Parameters:**
  - `dir_path` (str): Directory path (e.g., `"apps/com.example.myapp"`)

- **Returns:** `bool` - `True` if directory exists and has valid manifest

**`is_builtin_app(app_fullname)`**

Check if app is built-in (in `builtin/apps/`).

- **Parameters:**
  - `app_fullname` (str): App fullname

- **Returns:** `bool` - `True` if built-in

**`is_overridden_builtin_app(app_fullname)`**

Check if user app overrides a built-in app.

- **Parameters:**
  - `app_fullname` (str): App fullname

- **Returns:** `bool` - `True` if both user and built-in versions exist

### App Execution

**`start_app(fullname)`**

Start an app by fullname.

- **Parameters:**
  - `fullname` (str): App fullname

- **Returns:** `bool` - `True` if successful

- **Behavior:**
  1. Sets app as foreground
  2. Loads app metadata
  3. Executes main activity script
  4. Shows/hides top menu bar based on app type

**`execute_script(script_source, is_file, classname, cwd=None)`**

Execute a Python script with proper environment.

- **Parameters:**
  - `script_source` (str): Script path (if `is_file=True`) or script code
  - `is_file` (bool): `True` if script_source is a file path
  - `classname` (str): Name of main activity class to instantiate
  - `cwd` (str, optional): Working directory to add to sys.path

- **Returns:** `bool` - `True` if successful

- **Behavior:**
  1. Reads file (if `is_file=True`)
  2. Compiles script to bytecode
  3. Executes in prepared environment
  4. Finds and instantiates main activity class
  5. Starts activity using Activity framework
  6. Restores sys.path

### Intent Resolution

**`register_activity(action, activity_cls)`**

Register activity to handle intent action.

- **Parameters:**
  - `action` (str): Intent action (e.g., `"android.intent.action.SEND"`)
  - `activity_cls` (type): Activity class

**`resolve_activity(intent)`**

Find activities that handle intent.

- **Parameters:**
  - `intent` (Intent): Intent object with `action` attribute

- **Returns:** `list[type]` - List of Activity classes

**`query_intent_activities(intent)`**

Same as `resolve_activity()` (Android-like naming).

- **Parameters:**
  - `intent` (Intent): Intent object

- **Returns:** `list[type]` - List of Activity classes

### Launcher Management

**`get_launcher()`**

Find the system launcher app.

- **Returns:** `App` or `None` - Launcher app object

**`restart_launcher()`**

Stop all activities and restart launcher.

- **Behavior:**
  1. Removes and stops all activities
  2. Starts launcher app
  3. Useful for returning to home screen

## Best Practices

### Do's

✅ Call `AppManager.refresh_apps()` after installing/uninstalling apps  
✅ Check `AppManager.get(fullname)` before starting an app  
✅ Use `AppManager.start_app()` for user-initiated app launches  
✅ Handle version comparison for update checks  
✅ Use intent resolution for extensible app interactions  

### Don'ts

❌ Directly manipulate app directories (use `install_mpk()` and `uninstall_app()`)  
❌ Assume app is installed without checking  
❌ Start apps without verifying they exist  
❌ Modify app metadata directly (read-only)  
❌ Uninstall built-in apps (will fail)  

## Troubleshooting

### App Not Found

**Symptom**: `AppManager.get(fullname)` returns `None`

**Causes:**
- App not installed
- Fullname is incorrect
- App list not refreshed after installation

**Solution:**
```python
from mpos import AppManager

# Refresh app list
AppManager.refresh_apps()

# Check if app exists
app = AppManager.get("com.example.myapp")
if app:
    print(f"Found: {app.name}")
else:
    print("App not found")
    # List all apps to verify
    for app in AppManager.get_app_list():
        print(f"  - {app.fullname}")
```

### App Fails to Start

**Symptom**: `start_app()` returns `False`

**Possible causes:**
- App not installed
- Missing or invalid manifest
- Main activity class not found
- Script execution error

**Solution:**
```python
from mpos import AppManager

# Verify app exists
app = AppManager.get("com.example.myapp")
if not app:
    print("App not installed")
    return

# Check app has installed path
if not app.installed_path:
    print("App has no installed path")
    return

# Check main activity is defined
if not app.main_launcher_activity:
    print("App has no main launcher activity")
    return

# Try to start
success = AppManager.start_app("com.example.myapp")
if not success:
    print("Failed to start - check console for errors")
```

### Installation Fails

**Symptom**: `install_mpk()` doesn't create app

**Possible causes:**
- Invalid ZIP file
- Destination folder already exists
- Insufficient permissions
- Missing manifest in ZIP

**Solution:**
```python
from mpos import AppManager

# Verify ZIP file is valid
import zipfile
try:
    with zipfile.ZipFile("/tmp/app.mpk", "r") as z:
        print("ZIP contents:", z.namelist())
except Exception as e:
    print(f"Invalid ZIP: {e}")
    return

# Install with error handling
try:
    AppManager.install_mpk("/tmp/app.mpk", "apps/com.example.newapp")
    print("Installation successful")
except Exception as e:
    print(f"Installation failed: {e}")

# Verify installation
app = AppManager.get("com.example.newapp")
if app:
    print(f"App installed: {app.name}")
```

## See Also

- [Creating Apps](../apps/creating-apps.md) - How to create MicroPythonOS apps
- [Activity Framework](../apps/app-lifecycle.md) - Activity lifecycle and management
- [Intent System](../architecture/intents.md) - Intent-based app communication
- [Frameworks Overview](../architecture/frameworks.md) - All available frameworks
