# SettingActivity

`SettingActivity` is used to edit a single setting with various UI options. It provides a flexible framework for collecting user input with support for text input, radio buttons, dropdowns, and custom Activity implementations.

## Overview

`SettingActivity` displays a single setting that the user can modify. The setting is passed via Intent extras and can be configured with different UI types. When the user saves the setting, it's automatically persisted to SharedPreferences (unless `dont_persist` is set to true).

## Basic Usage

```python
from mpos import Activity, Intent, SettingActivity, SharedPreferences

class MyApp(Activity):
    def onCreate(self):
        self.prefs = SharedPreferences("com.example.myapp")
        # ... create UI ...
        
    def open_setting(self):
        intent = Intent(activity_class=SettingActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("setting", {
            "title": "Your Name",
            "key": "user_name",
            "placeholder": "Enter your name"
        })
        self.startActivity(intent)
```

## Setting Dictionary Structure

Each setting is defined as a dictionary with the following properties:

### Required Properties
- **`title`** (string): Display name of the setting shown at the top
- **`key`** (string): Unique identifier used as the SharedPreferences key

### Optional Properties
- **`ui`** (string): UI type to use for editing. Options: `"textarea"` (default), `"radiobuttons"`, `"dropdown"`, `"activity"`
- **`ui_options`** (list): Options for `radiobuttons` and `dropdown` UI types
- **`placeholder`** (string): Placeholder text for textarea input
- **`default_value`** (string): Default value to select or fill in
- **`changed_callback`** (function): Callback function called when the setting value changes
- **`should_show`** (function): Function to determine if this setting should be displayed
- **`dont_persist`** (bool): If `True`, the setting won't be saved to SharedPreferences
- **`activity_class`** (class): Custom Activity class for `"activity"` UI type
- **`value_label`** (widget): Internal reference to the value label (set by SettingsActivity)
- **`cont`** (widget): Internal reference to the container (set by SettingsActivity)

## UI Types

### 1. Textarea (Default)

Text input with an on-screen keyboard and optional QR code scanner.

**When to use:** For text input like URLs, API keys, names, etc.

**Example:**
```python
{
    "title": "API Key",
    "key": "api_key",
    "placeholder": "Enter your API key",
    "ui": "textarea"  # or omit this, textarea is default
}
```

**Features:**
- On-screen keyboard for text input
- QR code scanner button (for scanning data from QR codes)
- Placeholder text support
- Single-line input

### 2. Radio Buttons

Mutually exclusive selection from a list of options.

**When to use:** For choosing one option from a small set of choices.

**Format for `ui_options`:**
```python
ui_options = [
    ("Display Label 1", "value1"),
    ("Display Label 2", "value2"),
    ("Display Label 3", "value3")
]
```

**Example:**
```python
{
    "title": "Theme",
    "key": "theme",
    "ui": "radiobuttons",
    "ui_options": [
        ("Light", "light"),
        ("Dark", "dark"),
        ("Auto", "auto")
    ]
}
```

**Features:**
- Circular radio button indicators
- Only one option can be selected at a time
- Displays both label and value (if different)

### 3. Dropdown

Dropdown selection from a list of options.

**When to use:** For choosing one option from a larger set of choices.

**Format for `ui_options`:**
```python
ui_options = [
    ("Display Label 1", "value1"),
    ("Display Label 2", "value2"),
    ("Display Label 3", "value3")
]
```

**Example:**
```python
{
    "title": "Language",
    "key": "language",
    "ui": "dropdown",
    "ui_options": [
        ("English", "en"),
        ("German", "de"),
        ("French", "fr"),
        ("Spanish", "es")
    ]
}
```

**Features:**
- Compact dropdown menu
- Shows label and value if different
- Automatically selects the current value

### 4. Custom Activity

Use a custom Activity class for advanced UI implementations.

**When to use:** For complex settings that need custom UI beyond the built-in options.

**Example:**
```python
class ColorPickerActivity(Activity):
    def onCreate(self):
        # Custom color picker UI
        pass

setting = {
    "title": "Pick a Color",
    "key": "color",
    "ui": "activity",
    "activity_class": ColorPickerActivity
}
```

**Requirements:**
- Must provide `activity_class` parameter
- The custom Activity receives the setting and prefs via Intent extras
- Must call `self.finish()` to return to the previous screen

## Callbacks and Advanced Features

### changed_callback

Called when the setting value changes (after saving). Useful for triggering UI updates or reloading data.

**Example:**
```python
def on_theme_changed(new_value):
    print(f"Theme changed to: {new_value}")
    # Reload UI with new theme
    self.apply_theme(new_value)

setting = {
    "title": "Theme",
    "key": "theme",
    "ui": "radiobuttons",
    "ui_options": [("Light", "light"), ("Dark", "dark")],
    "changed_callback": on_theme_changed
}
```

**Important:** The callback is only called if the value actually changed (old value != new value).

### should_show

Function to conditionally show/hide a setting. Used primarily in SettingsActivity.

**Example:**
```python
def should_show_advanced_options(setting):
    prefs = SharedPreferences("com.example.app")
    return prefs.get_string("mode") == "advanced"

setting = {
    "title": "Advanced Option",
    "key": "advanced_setting",
    "should_show": should_show_advanced_options
}
```

### dont_persist

Prevent the setting from being saved to SharedPreferences.

**Example:**
```python
setting = {
    "title": "Temporary Value",
    "key": "temp_value",
    "dont_persist": True
}
```

## Complete Example: Simple Settings App

Here's a minimal app that uses SettingActivity to edit a single setting:

```python
import lvgl as lv
from mpos import Activity, Intent, SettingActivity, SharedPreferences

class SimpleSettingsApp(Activity):
    def onCreate(self):
        self.prefs = SharedPreferences("com.example.simplesettings")
        
        # Create main screen
        self.main_screen = lv.obj()
        self.main_screen.set_flex_flow(lv.FLEX_FLOW.COLUMN)
        
        # Display current value
        self.value_label = lv.label(self.main_screen)
        self.update_value_label()
        
        # Settings button
        settings_btn = lv.button(self.main_screen)
        settings_btn.set_size(lv.pct(80), lv.SIZE_CONTENT)
        settings_label = lv.label(settings_btn)
        settings_label.set_text("Edit Setting")
        settings_label.center()
        settings_btn.add_event_cb(self.open_settings, lv.EVENT.CLICKED, None)
        
        self.setContentView(self.main_screen)
    
    def onResume(self, screen):
        super().onResume(screen)
        self.update_value_label()
    
    def update_value_label(self):
        value = self.prefs.get_string("my_setting", "(not set)")
        self.value_label.set_text(f"Current value: {value}")
    
    def open_settings(self, event):
        intent = Intent(activity_class=SettingActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("setting", {
            "title": "My Setting",
            "key": "my_setting",
            "placeholder": "Enter a value",
            "changed_callback": self.on_setting_changed
        })
        self.startActivity(intent)
    
    def on_setting_changed(self, new_value):
        print(f"Setting changed to: {new_value}")
        self.update_value_label()
```

## Real-World Example: AppStore Backend Selection

This example is inspired by the AppStore app, showing how to use SettingActivity with radio buttons and a callback:

```python
import lvgl as lv
from mpos import Activity, Intent, SettingActivity, SharedPreferences

class AppStore(Activity):
    BACKENDS = [
        ("MPOS GitHub", "github,https://apps.micropythonos.com/app_index.json"),
        ("BadgeHub Test", "badgehub,https://badgehub.p1m.nl/api/v3"),
        ("BadgeHub Prod", "badgehub,https://badge.why2025.org/api/v3")
    ]
    
    def onCreate(self):
        self.prefs = SharedPreferences("com.micropythonos.appstore")
        
        self.main_screen = lv.obj()
        self.main_screen.set_flex_flow(lv.FLEX_FLOW.COLUMN)
        
        # Settings button
        settings_btn = lv.button(self.main_screen)
        settings_btn.set_size(lv.pct(20), lv.pct(25))
        settings_btn.align(lv.ALIGN.TOP_RIGHT, 0, 0)
        settings_label = lv.label(settings_btn)
        settings_label.set_text(lv.SYMBOL.SETTINGS)
        settings_label.center()
        settings_btn.add_event_cb(self.open_backend_settings, lv.EVENT.CLICKED, None)
        
        self.setContentView(self.main_screen)
    
    def open_backend_settings(self, event):
        intent = Intent(activity_class=SettingActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("setting", {
            "title": "AppStore Backend",
            "key": "backend",
            "ui": "radiobuttons",
            "ui_options": self.BACKENDS,
            "changed_callback": self.backend_changed
        })
        self.startActivity(intent)
    
    def backend_changed(self, new_value):
        print(f"Backend changed to {new_value}, refreshing app list...")
        # Trigger app list refresh with new backend
        self.refresh_app_list()
    
    def refresh_app_list(self):
        # Implementation to refresh the app list
        pass
```

## Tips and Best Practices

1. **Load prefs once in onCreate()**: Load SharedPreferences in your main Activity's `onCreate()` and pass it to SettingActivity. This is faster than loading it multiple times.

2. **Use changed_callback for side effects**: If changing a setting requires reloading data or updating the UI, use `changed_callback` instead of checking the value on resume.

3. **Validate input**: For textarea inputs, consider validating the input in your `changed_callback` before using it.

4. **Use radio buttons for small sets**: Use radio buttons for 2-5 options, dropdown for larger lists.

5. **Provide meaningful placeholders**: Help users understand what format is expected with clear placeholder text.

6. **Consider conditional visibility**: Use `should_show` in SettingsActivity to hide settings that don't apply based on other settings.
