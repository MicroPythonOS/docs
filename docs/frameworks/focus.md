# Focus Highlight

MicroPythonOS provides a single, reusable helper for focus highlighting so that apps and framework screens do not have to duplicate the same `FOCUSED`/`DEFOCUSED` event handlers.

## Overview

`add_focus_highlight` registers two LVGL event callbacks on a widget:

- **`FOCUSED`**: draw a border around the widget (or tint its background) and scroll it into view.
- **`DEFOCUSED`**: remove the highlight again.

The helper is re-exported from the top-level `mpos` module, so apps can import it directly.

Two highlight modes are available:

| Mode | Effect |
|------|--------|
| `"border"` (default) | Draws a border around the widget on focus. |
| `"bg"` | Tints the widget's background on focus, leaving border styles intact. |

## API

```python
from mpos import add_focus_highlight

# Border mode (default)
add_focus_highlight(widget)
add_focus_highlight(widget, width=2)
add_focus_highlight(widget, width=2, color=lv.color_hex(0xFFFFFF))
add_focus_highlight(widget, width=2, opacity=lv.OPA._50, radius=5)

# Background mode
add_focus_highlight(widget, mode="bg")
add_focus_highlight(widget, mode="bg", color=lv.color_hex(0x444444))
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `widget` | required | The LVGL object that should receive focus highlighting. |
| `mode` | `"border"` | Highlight mode: `"border"` draws a border, `"bg"` tints the background. |
| `width` | `1` | Border width in pixels when focused (ignored in `"bg"` mode). |
| `color` | theme primary color | Highlight color. Falls back to `lv.theme_get_color_primary(None)` when omitted. |
| `opacity` | `None` | Optional `lv.OPA.*` value for the focused border (ignored in `"bg"` mode). |
| `radius` | `None` | Optional corner radius for the focused border (ignored in `"bg"` mode). |

### Widget and focus group

`add_focus_highlight` adds the widget to the default LVGL focus group automatically. You do not need to call `focusgroup.add_obj()` manually.

### Touch-aware behavior

On devices that are primarily controlled with a touchscreen, the focus highlight is intentionally hidden until the user first navigates with a directional input (keyboard, hardware buttons, or a rotary encoder). This keeps touch-only screens clean while still making focused elements obvious once physical navigation begins.

If you want a widget to reveal the focus highlight immediately, focus it programmatically:

```python
focusgroup = lv.group_get_default()
if focusgroup:
    focusgroup.focus_obj(btn)
```

## Examples

### Basic usage (border mode)

```python
import lvgl as lv
from mpos import Activity, add_focus_highlight

class MyActivity(Activity):
    def onCreate(self):
        screen = lv.obj()
        screen.set_flex_flow(lv.FLEX_FLOW.COLUMN)

        btn = lv.button(screen)
        btn_label = lv.label(btn)
        btn_label.set_text("Focus me")

        add_focus_highlight(btn)

        self.setContentView(screen)
```

### Background mode

Use `mode="bg"` when the widget already has a persistent border that shouldn't be overwritten:

```python
btn = lv.button(screen)
btn.set_style_border_width(2, lv.PART.MAIN)
btn.set_style_border_color(lv.color_hex(0x333333), lv.PART.MAIN)
add_focus_highlight(btn, mode="bg")
```

### Thicker, semi-transparent border

```python
add_focus_highlight(btn, width=3, opacity=lv.OPA._50, radius=5)
```

### Focus highlight on a label

Labels are not focusable by default, but they can be made focusable when they act as list rows:

```python
row = lv.label(screen)
row.set_text("Setting row")
row.add_flag(lv.obj.FLAG.CLICKABLE)
add_focus_highlight(row, width=2)
```

## Compatibility

`add_focus_border` is kept as a compatibility wrapper for the pre‑0.15.0 API. It delegates to `add_focus_highlight` with `mode="border"`. New code should use `add_focus_highlight` directly.

## When to use it

Use `add_focus_highlight` whenever you previously wrote a pair of `FOCUSED`/`DEFOCUSED` callbacks whose only job was to show or hide a border or background tint. It is used by the framework itself in places such as the launcher, settings screens, the top-menu drawer, `ImageView`, `MusicPlayer`, `ShowFonts`, `Connect4`, `About`, `HowTo`, and `WiFi Settings`.

If you need additional behavior on focus (for example, starting a timer or loading data), keep your own callbacks and add `add_focus_highlight` alongside them.

## See Also

- [InputManager](input-manager.md) - Pointer and focus-group utilities
- [App Lifecycle](../apps/app-lifecycle.md) - Activity lifecycle and screen construction
