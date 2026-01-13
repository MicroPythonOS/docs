# SettingsActivity

`SettingsActivity` is used to display and edit a list of multiple settings. It provides a scrollable list interface where users can tap on each setting to edit it individually. Each setting can have different UI types and optional conditional visibility.

## Overview

`SettingsActivity` displays a list of settings that users can browse and edit. When a user taps on a setting, it opens a [`SettingActivity`](setting-activity.md) for editing that individual setting. After editing, the value is automatically saved to SharedPreferences and the list is updated.

## Basic Usage

```python
from mpos import Activity, Intent, SettingsActivity, SharedPreferences

class MyApp(Activity):
    def onCreate(self):
        self.prefs = SharedPreferences("com.example.myapp")
        # ... create UI ...
        
    def open_settings(self):
        intent = Intent(activity_class=SettingsActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("settings", [
            {"title": "Your Name", "key": "user_name", "placeholder": "Enter your name"},
            {"title": "Theme", "key": "theme", "ui": "radiobuttons", "ui_options": [("Light", "light"), ("Dark", "dark")]},
        ])
        self.startActivity(intent)
```

## Settings List Structure

`SettingsActivity` expects a list of setting dictionaries. Each setting dictionary has the same structure as in [`SettingActivity`](setting-activity.md):

### Required Properties
- **`title`** (string): Display name of the setting shown in the list
- **`key`** (string): Unique identifier used as the SharedPreferences key

### Optional Properties
- **`ui`** (string): UI type for editing. Options: `"textarea"` (default), `"radiobuttons"`, `"dropdown"`, `"activity"`
- **`ui_options`** (list): Options for `radiobuttons` and `dropdown` UI types
- **`placeholder`** (string): Placeholder text for textarea input
- **`changed_callback`** (function): Callback function called when the setting value changes
- **`should_show`** (function): Function to determine if this setting should be displayed in the list
- **`dont_persist`** (bool): If `True`, the setting won't be saved to SharedPreferences
- **`activity_class`** (class): Custom Activity class for `"activity"` UI type
- **`value_label`** (widget): Internal reference to the value label (set by SettingsActivity)
- **`cont`** (widget): Internal reference to the container (set by SettingsActivity)

## UI Types

All UI types available in [`SettingActivity`](setting-activity.md) are supported:

### 1. Textarea (Default)

Text input with on-screen keyboard and optional QR code scanner.

```python
{
    "title": "API Key",
    "key": "api_key",
    "placeholder": "Enter your API key"
}
```

### 2. Radio Buttons

Mutually exclusive selection from a list of options.

```python
{
    "title": "Wallet Type",
    "key": "wallet_type",
    "ui": "radiobuttons",
    "ui_options": [
        ("LNBits", "lnbits"),
        ("Nostr Wallet Connect", "nwc")
    ]
}
```

### 3. Dropdown

Dropdown selection from a list of options.

```python
{
    "title": "Language",
    "key": "language",
    "ui": "dropdown",
    "ui_options": [
        ("English", "en"),
        ("German", "de"),
        ("French", "fr")
    ]
}
```

### 4. Custom Activity

Use a custom Activity class for advanced UI implementations.

```python
{
    "title": "Pick a Color",
    "key": "color",
    "ui": "activity",
    "activity_class": ColorPickerActivity
}
```

## Advanced Features

### Conditional Visibility with should_show

The `should_show` function allows you to conditionally display settings based on other settings or app state.

**Function signature:**
```python
def should_show(setting):
    # Return True to show, False to hide
    return condition
```

**Example:**
```python
def should_show_lnbits_settings(setting):
    prefs = SharedPreferences("com.example.wallet")
    wallet_type = prefs.get_string("wallet_type")
    return wallet_type == "lnbits"

settings = [
    {"title": "Wallet Type", "key": "wallet_type", "ui": "radiobuttons", 
     "ui_options": [("LNBits", "lnbits"), ("NWC", "nwc")]},
    {"title": "LNBits URL", "key": "lnbits_url", "placeholder": "https://...",
     "should_show": should_show_lnbits_settings},
    {"title": "LNBits Read Key", "key": "lnbits_readkey",
     "should_show": should_show_lnbits_settings},
]
```

### Callbacks on Value Change

The `changed_callback` is called when a setting value changes. Useful for triggering UI updates or reloading data.

```python
def on_wallet_type_changed(new_value):
    print(f"Wallet type changed to: {new_value}")
    # Reconnect to new wallet backend
    reconnect_wallet(new_value)

settings = [
    {"title": "Wallet Type", "key": "wallet_type", "ui": "radiobuttons",
     "ui_options": [("LNBits", "lnbits"), ("NWC", "nwc")],
     "changed_callback": on_wallet_type_changed},
]
```

### Preventing Persistence

Use `dont_persist` to prevent a setting from being saved to SharedPreferences.

```python
{
    "title": "Temporary Value",
    "key": "temp_value",
    "dont_persist": True
}
```

## Complete Example: Simple Settings List

Here's a minimal app that uses SettingsActivity to edit multiple settings:

```python
import lvgl as lv
from mpos import Activity, Intent, SettingsActivity, SharedPreferences

class SimpleSettingsApp(Activity):
    def onCreate(self):
        self.prefs = SharedPreferences("com.example.simplesettings")
        
        # Create main screen
        self.main_screen = lv.obj()
        self.main_screen.set_flex_flow(lv.FLEX_FLOW.COLUMN)
        
        # Settings button
        settings_btn = lv.button(self.main_screen)
        settings_btn.set_size(lv.pct(80), lv.SIZE_CONTENT)
        settings_label = lv.label(settings_btn)
        settings_label.set_text("Open Settings")
        settings_label.center()
        settings_btn.add_event_cb(self.open_settings, lv.EVENT.CLICKED, None)
        
        self.setContentView(self.main_screen)
    
    def open_settings(self, event):
        intent = Intent(activity_class=SettingsActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("settings", [
            {
                "title": "Your Name",
                "key": "user_name",
                "placeholder": "Enter your name"
            },
            {
                "title": "Theme",
                "key": "theme",
                "ui": "radiobuttons",
                "ui_options": [
                    ("Light", "light"),
                    ("Dark", "dark")
                ]
            },
            {
                "title": "Language",
                "key": "language",
                "ui": "dropdown",
                "ui_options": [
                    ("English", "en"),
                    ("German", "de"),
                    ("French", "fr")
                ]
            }
        ])
        self.startActivity(intent)
```

## Real-World Example: Lightning Piggy Wallet

This example is inspired by the Lightning Piggy DisplayWallet app, showing how to use SettingsActivity with conditional visibility and callbacks:

```python
import lvgl as lv
from mpos import Activity, Intent, SettingsActivity, SharedPreferences

class DisplayWallet(Activity):
    def onCreate(self):
        self.prefs = SharedPreferences("com.lightningpiggy.displaywallet")
        
        # Create main screen
        self.main_screen = lv.obj()
        
        # Settings button
        settings_button = lv.button(self.main_screen)
        settings_button.set_size(lv.pct(20), lv.pct(25))
        settings_button.align(lv.ALIGN.BOTTOM_RIGHT, 0, 0)
        settings_button.add_event_cb(self.settings_button_tap, lv.EVENT.CLICKED, None)
        settings_label = lv.label(settings_button)
        settings_label.set_text(lv.SYMBOL.SETTINGS)
        settings_label.center()
        
        self.setContentView(self.main_screen)
    
    def should_show_lnbits_settings(self, setting):
        """Only show LNBits settings if LNBits is selected"""
        wallet_type = self.prefs.get_string("wallet_type")
        return wallet_type == "lnbits"
    
    def should_show_nwc_settings(self, setting):
        """Only show NWC settings if NWC is selected"""
        wallet_type = self.prefs.get_string("wallet_type")
        return wallet_type == "nwc"
    
    def on_wallet_type_changed(self, new_value):
        """Called when wallet type changes"""
        print(f"Wallet type changed to: {new_value}")
        # Reconnect to new wallet backend
        self.reconnect_wallet(new_value)
    
    def settings_button_tap(self, event):
        intent = Intent(activity_class=SettingsActivity)
        intent.putExtra("prefs", self.prefs)
        intent.putExtra("settings", [
            {
                "title": "Wallet Type",
                "key": "wallet_type",
                "ui": "radiobuttons",
                "ui_options": [
                    ("LNBits", "lnbits"),
                    ("Nostr Wallet Connect", "nwc")
                ],
                "changed_callback": self.on_wallet_type_changed
            },
            {
                "title": "LNBits URL",
                "key": "lnbits_url",
                "placeholder": "https://demo.lnpiggy.com",
                "should_show": self.should_show_lnbits_settings
            },
            {
                "title": "LNBits Read Key",
                "key": "lnbits_readkey",
                "placeholder": "fd92e3f8168ba314dc22e54182784045",
                "should_show": self.should_show_lnbits_settings
            },
            {
                "title": "Optional LN Address",
                "key": "lnbits_static_receive_code",
                "placeholder": "Will be fetched if empty.",
                "should_show": self.should_show_lnbits_settings
            },
            {
                "title": "Nostr Wallet Connect",
                "key": "nwc_url",
                "placeholder": "nostr+walletconnect://69effe7b...",
                "should_show": self.should_show_nwc_settings
            },
            {
                "title": "Optional LN Address",
                "key": "nwc_static_receive_code",
                "placeholder": "Optional if present in NWC URL.",
                "should_show": self.should_show_nwc_settings
            }
        ])
        self.startActivity(intent)
    
    def reconnect_wallet(self, wallet_type):
        """Reconnect to the selected wallet backend"""
        if wallet_type == "lnbits":
            url = self.prefs.get_string("lnbits_url")
            readkey = self.prefs.get_string("lnbits_readkey")
            # Initialize LNBits wallet
            print(f"Connecting to LNBits at {url}")
        elif wallet_type == "nwc":
            nwc_url = self.prefs.get_string("nwc_url")
            # Initialize NWC wallet
            print(f"Connecting to NWC at {nwc_url}")
```

## UI Layout

SettingsActivity displays settings in a scrollable list with the following layout for each setting:

```
┌─────────────────────────────────┐
│ Setting Title                   │
│ Current Value (smaller text)    │
└─────────────────────────────────┘
```

- **Title**: Bold, larger font (montserrat_16)
- **Value**: Smaller font (montserrat_12), gray color
- **Container**: Full width, clickable, with border on focus
- **Scrollable**: Settings list scrolls if it exceeds screen height

## Tips and Best Practices

1. **Load prefs once in onCreate()**: Load SharedPreferences in your main Activity's `onCreate()` and pass it to SettingsActivity. This is faster than loading it multiple times.

2. **Use should_show for conditional settings**: Hide settings that don't apply based on other settings using the `should_show` function. This keeps the UI clean and prevents confusion.

3. **Use changed_callback for side effects**: If changing a setting requires reloading data or updating the UI, use `changed_callback` instead of checking the value on resume.

4. **Group related settings**: Organize settings logically in the list so related options are near each other.

5. **Provide meaningful placeholders**: Help users understand what format is expected with clear placeholder text.

6. **Use radio buttons for small sets**: Use radio buttons for 2-5 options, dropdown for larger lists.

7. **Consider performance**: If you have many settings with complex `should_show` functions, the list rebuild might be slow. Optimize your condition checks.

## Comparison with SettingActivity

| Feature | SettingActivity | SettingsActivity |
|---------|-----------------|------------------|
| **Purpose** | Edit a single setting | Edit multiple settings |
| **UI** | Full-screen editor | Scrollable list |
| **Use case** | Detailed editing of one setting | Browse and edit multiple settings |
| **Callbacks** | `changed_callback` | `changed_callback` + `should_show` |
| **Best for** | Settings that need focus | Quick access to many settings |

## See Also

- [`SettingActivity`](setting-activity.md) - For editing a single setting
- [`SharedPreferences`](preferences.md) - For persisting settings
