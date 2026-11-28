# InputActivity

`InputActivity` is a generic, reusable single-value input screen. It handles the UI and user input for one setting, then returns the value to the caller via `startActivityForResult`. It does **not** persist anything itself.

## Overview

`InputActivity` is useful any time you need to ask the user for a single value: a text string, a choice from radio buttons or a dropdown, or an integer from a slider. It supports the same input widgets as `SettingActivity`, but is independent of `SharedPreferences`.

`SettingActivity` is a thin wrapper around `InputActivity` that adds `SharedPreferences` persistence, row-label updates, and `changed_callback` handling. Apps that only need the input UI can use `InputActivity` directly.

## Basic Usage

```python
from mpos import Activity, Intent, InputActivity

class MyApp(Activity):
    def ask_for_password(self):
        intent = Intent(activity_class=InputActivity)
        intent.putExtra("setting", {
            "title": "WiFi Password",
            "placeholder": "Enter password",
            "ui": "textarea"
        })
        intent.putExtra("value", "")
        self.startActivityForResult(intent, self.on_password_result)

    def on_password_result(self, result):
        if result and result.get("result_code"):
            password = result["data"]["value"]
            print("Password entered:", password)
```

## Intent Extras

### Required extras

- **`setting`** (dict): Metadata describing the input.

### Optional extras

- **`value`** (string): Initial value to display. If omitted, `setting["default_value"]` is used, falling back to `""`.

## Setting Dictionary Structure

The `setting` dict has the same shape as the one used by `SettingActivity` and `SettingsActivity`:

### Required properties

- **`title`** (string): Display name shown at the top of the input screen

### Optional properties

- **`key`** (string): Identifier returned in the result. Not used by `InputActivity` itself, but helpful for callers.
- **`ui`** (string): UI type. Options: `"textarea"` (default), `"radiobuttons"`, `"dropdown"`, `"slider"`, `"activity"`
- **`ui_options`** (list): List of `(label, value)` tuples for `radiobuttons` and `dropdown`
- **`placeholder`** (string): Placeholder text for `textarea`
- **`default_value`** (string): Default value if no `value` extra is provided
- **`note`** (string): Short informational text shown below the input widget
- **`min`** (int): Minimum value for `"slider"` (default: `0`)
- **`max`** (int): Maximum value for `"slider"` (default: `100`)
- **`allow_deselect`** (bool): For `radiobuttons`, allow the user to un-select the active option (default: `False`)
- **`selected_callback`** (callable): For `radiobuttons` and `dropdown`, called as `selected_callback(value)` every time the user picks an option (including re-tapping the already-selected radio), **before** Save. Use it to preview a choice live, e.g. play the sound the user just tapped. Independent of `changed_callback`, which still fires only after Save when the stored value changed. Exceptions inside it are logged and swallowed. (default: `None`)
- **`scan_check`** (callable): For `textarea`, called as `scan_check(text)` after data is scanned from a QR code into the field. Return `True` (or `None`) when the data has the expected format, `False` to show the standard warning (`This doesn't look like a valid <title>. Check that you scanned the right QR code.`, with the setting's title filled in), or a string to show that text as the warning instead. The warning appears below the field (below the on-screen keyboard while that is open), the scanned text is still filled in, and Save is not blocked. It disappears when a later scan passes the check or the user edits the text. Exceptions inside it are logged and swallowed. (default: `None`)

## Result Contract

`InputActivity` returns a result dict with the standard `Activity` result shape:

```python
{
    "result_code": True,       # True on Save, False on Cancel/back
    "data": {
        "value": "<new_value>",
        "key": "<setting key>",
        "ui": "<ui type>"
    }
}
```

The caller is responsible for persisting or acting on the returned `value`.

## UI Types

### Textarea (default)

Text input with an on-screen keyboard. If the device has a camera, a "Scan data from QR code" button is also shown.

```python
{
    "title": "API Key",
    "key": "api_key",
    "placeholder": "Enter your API key"
}
```

To catch a scan of the wrong QR code, give the setting a `scan_check`:

```python
def looks_like_api_key(text):
    if len(text) == 32 and all(c in "0123456789abcdef" for c in text):
        return True
    if text.startswith(("http://", "https://")):
        return "This QR code looks like a web address, not an API key. Check that you scanned the right QR code."
    return False  # standard warning

{
    "title": "API Key",
    "key": "api_key",
    "scan_check": looks_like_api_key
}
```

### Radio Buttons

Mutually exclusive selection from a small list of options.

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

### Dropdown

Compact selection from a larger list of options.

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

### Slider

Integer selection within a range.

```python
{
    "title": "Volume",
    "key": "volume",
    "ui": "slider",
    "default_value": "50",
    "min": 0,
    "max": 100
}
```

## Cancel and Back Handling

Pressing the **Cancel** button or swiping back returns:

```python
{"result_code": False, "data": {"value": "...", "key": "...", "ui": "..."}}
```

Callers should check `result.get("result_code")` before using the value.

## See Also

- [`SettingActivity`](setting-activity.md) - Wrapper that persists the input value to `SharedPreferences`
- [`SettingsActivity`](settings-activity.md) - List of multiple settings, each edited through `SettingActivity`/`InputActivity`
- [`SharedPreferences`](preferences.md) - For persisting values yourself
