# Focus Borders

MicroPythonOS provides a single, reusable helper for focus highlighting so that apps and framework screens do not have to duplicate the same `FOCUSED`/`DEFOCUSED` event handlers.

## Overview

`add_focus_border` registers two LVGL event callbacks on a widget:

- **`FOCUSED`**: draw a border around the widget and scroll it into view.
- **`DEFOCUSED`**: hide the border again.

The helper is re-exported from the top-level `mpos` module, so apps can import it directly.

## API

```python
from mpos import add_focus_border

add_focus_border(widget)
add_focus_border(widget, width=2)
add_focus_border(widget, width=2, color=lv.color_hex(0xFFFFFF))
add_focus_border(widget, width=2, opacity=lv.OPA._50, radius=5)
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `widget` | required | The LVGL object that should receive focus highlighting. |
| `width`  | `1`      | Border width in pixels when focused. |
| `color`  | theme primary color | Border color. Falls back to `lv.theme_get_color_primary(None)` when omitted. |
| `opacity`| `None`   | Optional `lv.OPA.*` value for the focused border. |
| `radius` | `None`   | Optional corner radius for the focused border. |

### Requirements

For the callbacks to fire, the widget must be part of the default LVGL focus group:

```python
focusgroup = lv.group_get_default()
if focusgroup:
    focusgroup.add_obj(my_widget)
```

`add_focus_border` only draws the visual feedback; it does **not** add the widget to the focus group.

### Touch-aware behavior

On devices that are primarily controlled with a touchscreen, the focus border is intentionally hidden until the user first navigates with a directional input (keyboard, hardware buttons, or a rotary encoder). This keeps touch-only screens clean while still making focused elements obvious once physical navigation begins.

If you want a widget to reveal the focus border immediately, ensure it is added to the default focus group and focus it programmatically:

```python
focusgroup = lv.group_get_default()
if focusgroup:
    focusgroup.add_obj(btn)
    focusgroup.focus_obj(btn)
```

## Examples

### Basic usage

```python
import lvgl as lv
from mpos import Activity, add_focus_border

class MyActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        screen.set_flex_flow(lv.FLEX_FLOW.COLUMN)

        btn = lv.button(screen)
        btn_label = lv.label(btn)
        btn_label.set_text("Focus me")

        add_focus_border(btn)

        focusgroup = lv.group_get_default()
        if focusgroup:
            focusgroup.add_obj(btn)

        self.setContentView(screen)
```

### Thicker, semi-transparent border

```python
add_focus_border(btn, width=3, opacity=lv.OPA._50, radius=5)
```

### Focus highlight on a label

Labels are not focusable by default, but they can be added to the focus group when they act as list rows:

```python
row = lv.label(screen)
row.set_text("Setting row")
row.add_flag(lv.obj.FLAG.CLICKABLE)
add_focus_border(row, width=2)
focusgroup.add_obj(row)
```

## When to use it

Use `add_focus_border` whenever you previously wrote a pair of `FOCUSED`/`DEFOCUSED` callbacks whose only job was to show or hide a border. It is used by the framework itself in places such as the launcher, settings screens, the top-menu drawer, `ImageView`, `MusicPlayer`, `ShowFonts`, `Connect4`, `About`, `HowTo`, and `WiFi Settings`.

If you need additional behavior on focus (for example, starting a timer or loading data), keep your own callbacks and add `add_focus_border` alongside them.

## See Also

- [InputManager](input-manager.md) - Pointer and focus-group utilities
- [App Lifecycle](../apps/app-lifecycle.md) - Activity lifecycle and screen construction
