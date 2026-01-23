# WidgetAnimator

MicroPythonOS provides a **WidgetAnimator** utility for creating smooth, non-blocking animations on LVGL widgets. It handles common animation patterns like fade-in/fade-out, slide transitions, and value interpolation with automatic cleanup and error handling.

## Overview

WidgetAnimator provides:

- **Fade animations** - Smooth opacity transitions (fade-in/fade-out)
- **Slide animations** - Smooth position transitions (slide-up/slide-down)
- **Value interpolation** - Animate numeric values with custom display callbacks
- **Non-blocking** - Animations run asynchronously without blocking the UI
- **Safe widget access** - Automatically handles deleted widgets without crashes
- **Automatic cleanup** - Stops previous animations to prevent visual glitches
- **Easing functions** - Built-in ease-in-out path for smooth motion

## Quick Start

### Fade In/Out

```python
from mpos import WidgetAnimator
import lvgl as lv

# Create a widget
my_widget = lv.label(screen)
my_widget.set_text("Hello")

# Fade in (500ms)
WidgetAnimator.show_widget(my_widget, anim_type="fade", duration=500)

# Fade out (500ms)
WidgetAnimator.hide_widget(my_widget, anim_type="fade", duration=500)
```

### Convenience Methods

For the most common use case (fade animations), use the convenience methods:

```python
from mpos import WidgetAnimator

# Fade in with default 500ms duration
WidgetAnimator.smooth_show(my_widget)

# Fade out with default 500ms duration
WidgetAnimator.smooth_hide(my_widget)

# Fade out without hiding the widget (just opacity)
WidgetAnimator.smooth_hide(my_widget, hide=False)
```

### Slide Animations

```python
from mpos import WidgetAnimator

# Slide down from above
WidgetAnimator.show_widget(my_widget, anim_type="slide_down", duration=500)

# Slide up from below
WidgetAnimator.show_widget(my_widget, anim_type="slide_up", duration=500)

# Hide by sliding down
WidgetAnimator.hide_widget(my_widget, anim_type="slide_down", duration=500)

# Hide by sliding up
WidgetAnimator.hide_widget(my_widget, anim_type="slide_up", duration=500)
```

### Value Interpolation

Animate numeric values with a custom display callback:

```python
from mpos import WidgetAnimator

# Animate balance from 1000 to 1500 over 5 seconds
def display_balance(value):
    balance_label.set_text(f"{int(value)} sats")

WidgetAnimator.change_widget(
    balance_label,
    anim_type="interpolate",
    duration=5000,
    begin_value=1000,
    end_value=1500,
    display_change=display_balance
)
```

## API Reference

### show_widget()

Show a widget with an animation.

```python
WidgetAnimator.show_widget(widget, anim_type="fade", duration=500, delay=0)
```

**Parameters:**
- `widget` (lv.obj): The widget to show
- `anim_type` (str): Animation type - `"fade"`, `"slide_down"`, or `"slide_up"` (default: `"fade"`)
- `duration` (int): Animation duration in milliseconds (default: 500)
- `delay` (int): Animation delay in milliseconds (default: 0)

**Returns:** The animation object

**Behavior:**
- Removes the `HIDDEN` flag at the start of animation
- Animates opacity (fade) or position (slide)
- Resets final state after animation completes

### hide_widget()

Hide a widget with an animation.

```python
WidgetAnimator.hide_widget(widget, anim_type="fade", duration=500, delay=0, hide=True)
```

**Parameters:**
- `widget` (lv.obj): The widget to hide
- `anim_type` (str): Animation type - `"fade"`, `"slide_down"`, or `"slide_up"` (default: `"fade"`)
- `duration` (int): Animation duration in milliseconds (default: 500)
- `delay` (int): Animation delay in milliseconds (default: 0)
- `hide` (bool): If `True`, adds `HIDDEN` flag after animation. If `False`, only animates opacity/position (default: `True`)

**Returns:** The animation object

**Behavior:**
- Animates opacity (fade) or position (slide)
- Adds `HIDDEN` flag after animation (if `hide=True`)
- Restores original position after slide animations

### change_widget()

Animate a numeric value with a custom display callback.

```python
WidgetAnimator.change_widget(
    widget,
    anim_type="interpolate",
    duration=5000,
    delay=0,
    begin_value=0,
    end_value=100,
    display_change=None
)
```

**Parameters:**
- `widget` (lv.obj): The widget being animated (used for cleanup)
- `anim_type` (str): Animation type - currently only `"interpolate"` is supported
- `duration` (int): Animation duration in milliseconds (default: 5000)
- `delay` (int): Animation delay in milliseconds (default: 0)
- `begin_value` (int/float): Starting value for interpolation (default: 0)
- `end_value` (int/float): Ending value for interpolation (default: 100)
- `display_change` (callable, optional): Callback function called with each interpolated value. If `None`, calls `widget.set_text(str(value))`

**Returns:** The animation object

**Behavior:**
- Interpolates between `begin_value` and `end_value`
- Calls `display_change(value)` for each frame
- Ensures final value is set after animation completes
- Uses ease-in-out path for smooth motion

### Convenience Methods

**`WidgetAnimator.smooth_show(widget, duration=500, delay=0)`**

Fade in a widget (shorthand for `show_widget()` with fade animation).

**`WidgetAnimator.smooth_hide(widget, hide=True, duration=500, delay=0)`**

Fade out a widget (shorthand for `hide_widget()` with fade animation).

## Animation Types

### Fade

Animates widget opacity from 0 to 255 (show) or 255 to 0 (hide).

```python
# Fade in
WidgetAnimator.show_widget(widget, anim_type="fade", duration=500)

# Fade out
WidgetAnimator.hide_widget(widget, anim_type="fade", duration=500)
```

**Use cases:**
- Smooth appearance/disappearance of UI elements
- Transitioning between screens
- Highlighting important information

### Slide Down

Animates widget position from above the screen to its original position (show) or from original position to below (hide).

```python
# Slide down from above
WidgetAnimator.show_widget(widget, anim_type="slide_down", duration=500)

# Slide down to below
WidgetAnimator.hide_widget(widget, anim_type="slide_down", duration=500)
```

**Use cases:**
- Drawer-like panels sliding down
- Notifications appearing from top
- Menu items sliding in

### Slide Up

Animates widget position from below the screen to its original position (show) or from original position to above (hide).

```python
# Slide up from below
WidgetAnimator.show_widget(widget, anim_type="slide_up", duration=500)

# Slide up to above
WidgetAnimator.hide_widget(widget, anim_type="slide_up", duration=500)
```

**Use cases:**
- Keyboard sliding up from bottom
- Bottom sheets appearing
- Menu items sliding in from bottom

### Interpolate

Animates numeric values with smooth easing.

```python
WidgetAnimator.change_widget(
    widget,
    anim_type="interpolate",
    duration=5000,
    begin_value=0,
    end_value=100,
    display_change=lambda v: label.set_text(f"{int(v)}")
)
```

**Use cases:**
- Animating balance updates
- Progress indicators
- Counters
- Numeric displays

## Complete Examples

### Image Viewer with Fade Transitions

```python
from mpos import Activity
from mpos import WidgetAnimator
import lvgl as lv

class ImageViewerActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        
        # Image widget
        self.image = lv.image(self.screen)
        self.image.center()
        self.image.add_flag(lv.obj.FLAG.HIDDEN)  # Start hidden
        
        # Navigation buttons
        prev_btn = lv.button(self.screen)
        prev_btn.align(lv.ALIGN.BOTTOM_LEFT, 0, 0)
        prev_btn.add_event_cb(lambda e: self.show_prev_image(), lv.EVENT.CLICKED, None)
        
        next_btn = lv.button(self.screen)
        next_btn.align(lv.ALIGN.BOTTOM_RIGHT, 0, 0)
        next_btn.add_event_cb(lambda e: self.show_next_image(), lv.EVENT.CLICKED, None)
        
        self.setContentView(self.screen)
    
    def show_prev_image(self):
        # Fade out current image
        WidgetAnimator.smooth_hide(self.image)
        # Load and fade in next image
        self.load_image("prev_image.png")
        WidgetAnimator.smooth_show(self.image)
    
    def show_next_image(self):
        # Fade out current image
        WidgetAnimator.smooth_hide(self.image)
        # Load and fade in next image
        self.load_image("next_image.png")
        WidgetAnimator.smooth_show(self.image)
    
    def load_image(self, path):
        self.image.set_src(f"M:{path}")
```

### Wallet Balance Animation

```python
from mpos import Activity
from mpos import WidgetAnimator
import lvgl as lv

class WalletActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        
        self.balance_label = lv.label(self.screen)
        self.balance_label.set_text("0 sats")
        self.balance_label.align(lv.ALIGN.TOP_MID, 0, 20)
        
        self.setContentView(self.screen)
    
    def on_balance_received(self, old_balance, new_balance, sats_added):
        """Animate balance update when payment received."""
        def display_balance(value):
            self.balance_label.set_text(f"{int(value)} sats")
        
        # Animate from old balance to new balance over 3 seconds
        WidgetAnimator.change_widget(
            self.balance_label,
            anim_type="interpolate",
            duration=3000,
            begin_value=old_balance,
            end_value=new_balance,
            display_change=display_balance
        )
```

### Fullscreen Mode Toggle with Slide Animations

```python
from mpos import Activity
from mpos import WidgetAnimator
import lvgl as lv

class ImageViewerActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        self.fullscreen = False
        
        # Image
        self.image = lv.image(self.screen)
        self.image.center()
        
        # UI controls
        self.controls = lv.obj(self.screen)
        self.controls.set_size(lv.pct(100), 50)
        self.controls.align(lv.ALIGN.BOTTOM_MID, 0, 0)
        
        prev_btn = lv.button(self.controls)
        prev_btn.align(lv.ALIGN.LEFT, 0, 0)
        
        next_btn = lv.button(self.controls)
        next_btn.align(lv.ALIGN.RIGHT, 0, 0)
        
        self.image.add_event_cb(lambda e: self.toggle_fullscreen(), lv.EVENT.CLICKED, None)
        
        self.setContentView(self.screen)
    
    def toggle_fullscreen(self):
        if self.fullscreen:
            # Exit fullscreen - slide controls back up
            self.fullscreen = False
            WidgetAnimator.show_widget(self.controls, anim_type="slide_up", duration=300)
        else:
            # Enter fullscreen - slide controls down
            self.fullscreen = True
            WidgetAnimator.hide_widget(self.controls, anim_type="slide_down", duration=300, hide=False)
```

## Best Practices

### 1. Use Convenience Methods for Simple Cases

```python
# Good - simple and readable
WidgetAnimator.smooth_show(widget)
WidgetAnimator.smooth_hide(widget)

# Less readable - more verbose
WidgetAnimator.show_widget(widget, anim_type="fade", duration=500, delay=0)
```

### 2. Provide Custom Display Callbacks for Value Animations

```python
# Good - custom formatting
def display_balance(value):
    balance_label.set_text(f"{int(value):,} sats")

WidgetAnimator.change_widget(
    balance_label,
    anim_type="interpolate",
    duration=3000,
    begin_value=old_balance,
    end_value=new_balance,
    display_change=display_balance
)

# Less good - default string conversion
WidgetAnimator.change_widget(
    balance_label,
    anim_type="interpolate",
    duration=3000,
    begin_value=old_balance,
    end_value=new_balance
)
```

### 3. Use Appropriate Durations

```python
# Quick feedback (UI elements)
WidgetAnimator.smooth_show(button, duration=200)

# Standard transitions (screens, panels)
WidgetAnimator.smooth_hide(panel, duration=500)

# Slow animations (value updates, important transitions)
WidgetAnimator.change_widget(
    balance_label,
    anim_type="interpolate",
    duration=3000,
    begin_value=old,
    end_value=new,
    display_change=display_fn
)
```

### 4. Handle Widget Deletion Gracefully

WidgetAnimator automatically handles deleted widgets through internal safe widget access. No special handling needed:

```python
# Safe - animation continues even if widget is deleted
WidgetAnimator.show_widget(widget, duration=500)
# ... widget might be deleted here ...
# Animation completes silently without crashing
```

### 5. Combine Animations for Complex Transitions

```python
# Fade out old content
WidgetAnimator.smooth_hide(old_content, duration=300)

# After fade completes, show new content
# (Use delay to sequence animations)
WidgetAnimator.smooth_show(new_content, duration=300, delay=300)
```

## Performance Tips

### 1. Use Shorter Durations for Frequent Animations

```python
# Good - quick feedback for user interactions
WidgetAnimator.smooth_show(button, duration=200)

# Avoid - slow animations for frequent events
WidgetAnimator.smooth_show(button, duration=2000)
```

### 2. Limit Simultaneous Animations

WidgetAnimator automatically stops previous animations on the same widget:

```python
# Safe - previous animation is stopped
WidgetAnimator.show_widget(widget, duration=500)
WidgetAnimator.show_widget(widget, duration=500)  # Previous animation stops
```

### 3. Use Appropriate Easing

All animations use `path_ease_in_out` for smooth, natural motion. This is optimal for most use cases.

## Troubleshooting

### Animation Not Visible

**Symptom**: Animation runs but widget doesn't appear to move/fade

**Possible causes:**
1. Widget is already in the target state
2. Widget is outside the visible area
3. Animation duration is too short to notice

**Solution:**
```python
# Ensure widget is in correct initial state
widget.remove_flag(lv.obj.FLAG.HIDDEN)  # For show animations
widget.set_style_opa(255, 0)  # For fade animations

# Use longer duration for testing
WidgetAnimator.smooth_show(widget, duration=1000)
```

### Widget Disappears After Animation

**Symptom**: Widget fades out but doesn't reappear

**Possible cause**: Using `hide=True` (default) which adds `HIDDEN` flag

**Solution:**
```python
# If you want to keep widget visible but transparent
WidgetAnimator.smooth_hide(widget, hide=False, duration=500)

# If you want to hide it completely
WidgetAnimator.smooth_hide(widget, hide=True, duration=500)
```

### Animation Stutters or Jitters

**Symptom**: Animation is not smooth, has visual glitches

**Possible causes:**
1. Too many simultaneous animations
2. Heavy processing during animation
3. Widget being modified during animation

**Solution:**
```python
# Avoid modifying widget during animation
# Stop any previous animations first
lv.anim_delete(widget, None)

# Then start new animation
WidgetAnimator.smooth_show(widget, duration=500)
```

### Value Animation Shows Wrong Final Value

**Symptom**: Animation completes but final value is incorrect

**Possible cause**: `display_change` callback not handling final value correctly

**Solution:**
```python
# Ensure callback handles all values correctly
def display_balance(value):
    # Handle both intermediate and final values
    self.balance_label.set_text(f"{int(value)} sats")

WidgetAnimator.change_widget(
    balance_label,
    anim_type="interpolate",
    duration=3000,
    begin_value=old_balance,
    end_value=new_balance,
    display_change=display_balance
)
```

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [LVGL Documentation](https://docs.lvgl.io/) - LVGL animation API reference
