# LightsManager

MicroPythonOS provides a simple LED control service called **LightsManager** for NeoPixel RGB LEDs on supported hardware.

## Overview

LightsManager provides:

- **One-shot LED control** - Direct control of individual or all LEDs
- **Buffered updates** - Set multiple LEDs then apply all changes at once
- **Hardware abstraction** - Same API works across all boards
- **Predefined colors** - Quick access to common notification colors
- **Frame-based animations** - Integrate with TaskHandler for smooth animations
- **Low overhead** - Lightweight singleton pattern

⚠️ **Note**: LightsManager provides primitives for LED control, not built-in animations. Apps implement custom animations using the `update_frame()` pattern.

## Hardware Support

| Board | LEDs | GPIO | Notes |
|-------|------|------|-------|
| **Fri3d 2024 Badge** | ✅ 5 NeoPixel RGB | GPIO 12 | Full support |
| **Waveshare ESP32-S3** | ❌ None | N/A | No LED hardware |
| **Desktop/Linux** | ❌ None | N/A | Functions return `False` |

## Quick Start

### Check Availability

```python
from mpos import Activity
import mpos.lights as LightsManager

class MyActivity(Activity):
    def onCreate(self):
        if LightsManager.is_available():
            print(f"LEDs available: {LightsManager.get_led_count()}")
        else:
            print("No LED hardware on this device")
```

### Control Individual LEDs

```python
# Set LED 0 to red (buffered)
LightsManager.set_led(0, 255, 0, 0)

# Set LED 1 to green (buffered)
LightsManager.set_led(1, 0, 255, 0)

# Set LED 2 to blue (buffered)
LightsManager.set_led(2, 0, 0, 255)

# Apply all changes to hardware
LightsManager.write()
```

**Important**: LEDs are **buffered**. Changes won't appear until you call `write()`.

### Control All LEDs

```python
# Set all LEDs to blue
LightsManager.set_all(0, 0, 255)
LightsManager.write()

# Turn off all LEDs
LightsManager.clear()
LightsManager.write()
```

### Notification Colors

Quick shortcuts for common colors:

```python
# Convenience method (sets all LEDs + calls write())
LightsManager.set_notification_color("red")     # Success/Error
LightsManager.set_notification_color("green")   # Success
LightsManager.set_notification_color("blue")    # Info
LightsManager.set_notification_color("yellow")  # Warning
LightsManager.set_notification_color("orange")  # Alert
LightsManager.set_notification_color("purple")  # Special
LightsManager.set_notification_color("white")   # General
```

## Custom Animations

LightsManager provides one-shot control - apps create animations by updating LEDs over time.

### Blink Pattern

Simple on/off blinking:

```python
import time
import mpos.lights as LightsManager

def blink_pattern():
    """Blink all LEDs red 5 times."""
    for _ in range(5):
        # Turn on
        LightsManager.set_all(255, 0, 0)
        LightsManager.write()
        time.sleep_ms(200)

        # Turn off
        LightsManager.clear()
        LightsManager.write()
        time.sleep_ms(200)
```

### Rainbow Cycle

Display a rainbow pattern across all LEDs:

```python
def rainbow_cycle():
    """Set each LED to a different rainbow color."""
    colors = [
        (255, 0, 0),    # Red
        (255, 128, 0),  # Orange
        (255, 255, 0),  # Yellow
        (0, 255, 0),    # Green
        (0, 0, 255),    # Blue
    ]

    for i, color in enumerate(colors):
        LightsManager.set_led(i, *color)

    LightsManager.write()
```

### Chase Effect

LEDs light up in sequence:

```python
import time

def chase_effect(color=(0, 255, 0), delay_ms=100, loops=3):
    """Light up LEDs in sequence."""
    led_count = LightsManager.get_led_count()

    for _ in range(loops):
        for i in range(led_count):
            # Clear all
            LightsManager.clear()

            # Light current LED
            LightsManager.set_led(i, *color)
            LightsManager.write()

            time.sleep_ms(delay_ms)

    # Clear after animation
    LightsManager.clear()
    LightsManager.write()
```

### Pulse/Breathing Effect

Fade LEDs in and out smoothly:

```python
import time

def pulse_effect(color=(0, 0, 255), duration_ms=2000):
    """Pulse LEDs with breathing effect."""
    steps = 20  # Number of brightness steps
    delay = duration_ms // (steps * 2)  # Time per step

    # Fade in
    for i in range(steps):
        brightness = i / steps
        r = int(color[0] * brightness)
        g = int(color[1] * brightness)
        b = int(color[2] * brightness)

        LightsManager.set_all(r, g, b)
        LightsManager.write()
        time.sleep_ms(delay)

    # Fade out
    for i in range(steps, 0, -1):
        brightness = i / steps
        r = int(color[0] * brightness)
        g = int(color[1] * brightness)
        b = int(color[2] * brightness)

        LightsManager.set_all(r, g, b)
        LightsManager.write()
        time.sleep_ms(delay)

    # Clear
    LightsManager.clear()
    LightsManager.write()
```

### Random Sparkle

Random LEDs flash briefly:

```python
import time
import random

def sparkle_effect(duration_ms=5000):
    """Random LEDs sparkle."""
    led_count = LightsManager.get_led_count()
    start_time = time.ticks_ms()

    while time.ticks_diff(time.ticks_ms(), start_time) < duration_ms:
        # Pick random LED
        led = random.randint(0, led_count - 1)

        # Random color
        r = random.randint(0, 255)
        g = random.randint(0, 255)
        b = random.randint(0, 255)

        # Flash on
        LightsManager.clear()
        LightsManager.set_led(led, r, g, b)
        LightsManager.write()
        time.sleep_ms(100)

        # Flash off
        LightsManager.clear()
        LightsManager.write()
        time.sleep_ms(50)

    # Clear after animation
    LightsManager.clear()
    LightsManager.write()
```

## Frame-Based LED Animations

For smooth, real-time animations that integrate with the game loop, use the **TaskHandler event system**:

```python
from mpos import Activity
import mpos.ui
import mpos.lights as LightsManager
import time

class LEDAnimationActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        self.last_time = 0
        self.led_index = 0
        self.setContentView(self.screen)

    def onResume(self, screen):
        """Start animation when app resumes."""
        self.last_time = time.ticks_ms()
        mpos.ui.task_handler.add_event_cb(self.update_frame, 1)

    def onPause(self, screen):
        """Stop animation and clear LEDs when app pauses."""
        mpos.ui.task_handler.remove_event_cb(self.update_frame)
        LightsManager.clear()
        LightsManager.write()

    def update_frame(self, a, b):
        """Called every frame - update animation."""
        current_time = time.ticks_ms()
        delta_time = time.ticks_diff(current_time, self.last_time) / 1000.0

        # Update every 0.5 seconds (2 Hz)
        if delta_time > 0.5:
            self.last_time = current_time

            # Clear all LEDs
            LightsManager.clear()

            # Light up current LED
            LightsManager.set_led(self.led_index, 0, 255, 0)
            LightsManager.write()

            # Move to next LED
            self.led_index = (self.led_index + 1) % LightsManager.get_led_count()
```

### Smooth Color Cycle Animation

```python
import math

class ColorCycleActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        self.hue = 0.0  # Color hue (0-360)
        self.last_time = time.ticks_ms()
        self.setContentView(self.screen)

    def onResume(self, screen):
        self.last_time = time.ticks_ms()
        mpos.ui.task_handler.add_event_cb(self.update_frame, 1)

    def onPause(self, screen):
        mpos.ui.task_handler.remove_event_cb(self.update_frame)
        LightsManager.clear()
        LightsManager.write()

    def update_frame(self, a, b):
        current_time = time.ticks_ms()
        delta_time = time.ticks_diff(current_time, self.last_time) / 1000.0
        self.last_time = current_time

        # Rotate hue
        self.hue += 120 * delta_time  # 120 degrees per second
        if self.hue >= 360:
            self.hue -= 360

        # Convert HSV to RGB
        r, g, b = self.hsv_to_rgb(self.hue, 1.0, 1.0)

        # Update all LEDs
        LightsManager.set_all(r, g, b)
        LightsManager.write()

    def hsv_to_rgb(self, h, s, v):
        """Convert HSV to RGB (h: 0-360, s: 0-1, v: 0-1)."""
        h = h / 60.0
        i = int(h)
        f = h - i
        p = v * (1 - s)
        q = v * (1 - s * f)
        t = v * (1 - s * (1 - f))

        if i == 0:
            r, g, b = v, t, p
        elif i == 1:
            r, g, b = q, v, p
        elif i == 2:
            r, g, b = p, v, t
        elif i == 3:
            r, g, b = p, q, v
        elif i == 4:
            r, g, b = t, p, v
        else:
            r, g, b = v, p, q

        return int(r * 255), int(g * 255), int(b * 255)
```

## API Reference

### Functions

**`is_available()`**

Check if LED hardware is available.

- **Returns:** `bool` - `True` if LEDs are available, `False` otherwise

**`get_led_count()`**

Get the number of available LEDs.

- **Returns:** `int` - Number of LEDs (5 on Fri3d badge, 0 elsewhere)

**`set_led(index, r, g, b)`**

Set a single LED color (buffered).

- **Parameters:**
  - `index` (int): LED index (0 to led_count-1)
  - `r` (int): Red component (0-255)
  - `g` (int): Green component (0-255)
  - `b` (int): Blue component (0-255)

**`set_all(r, g, b)`**

Set all LEDs to the same color (buffered).

- **Parameters:**
  - `r` (int): Red component (0-255)
  - `g` (int): Green component (0-255)
  - `b` (int): Blue component (0-255)

**`clear()`**

Turn off all LEDs (buffered). Equivalent to `set_all(0, 0, 0)`.

**`write()`**

Apply buffered LED changes to hardware. **Required** - changes won't appear until you call this.

**`set_notification_color(color_name)`**

Set all LEDs to a predefined color and immediately apply (convenience method).

- **Parameters:**
  - `color_name` (str): Color name ("red", "green", "blue", "yellow", "orange", "purple", "white")

### Predefined Colors

| Color Name | RGB Value | Use Case |
|------------|-----------|----------|
| `"red"` | (255, 0, 0) | Error, alert |
| `"green"` | (0, 255, 0) | Success, ready |
| `"blue"` | (0, 0, 255) | Info, processing |
| `"yellow"` | (255, 255, 0) | Warning, caution |
| `"orange"` | (255, 128, 0) | Alert, attention |
| `"purple"` | (255, 0, 255) | Special, unique |
| `"white"` | (255, 255, 255) | General, neutral |

## Performance Tips

### Buffering for Efficiency

**❌ Bad** - Calling `write()` after each LED:
```python
for i in range(5):
    LightsManager.set_led(i, 255, 0, 0)
    LightsManager.write()  # 5 hardware updates!
```

**✅ Good** - Set all LEDs then write once:
```python
for i in range(5):
    LightsManager.set_led(i, 255, 0, 0)

LightsManager.write()  # 1 hardware update
```

### Update Rate Recommendations

- **Static displays**: Update once, no frame loop needed
- **Notifications**: Update when event occurs
- **Smooth animations**: 20-30 Hz (every 33-50ms)
- **Fast effects**: Up to 60 Hz (but uses more power)

**Example with rate limiting:**
```python
UPDATE_INTERVAL = 0.05  # 20 Hz (50ms)

def update_frame(self, a, b):
    current_time = time.ticks_ms()
    delta_time = time.ticks_diff(current_time, self.last_time) / 1000.0

    if delta_time >= UPDATE_INTERVAL:
        self.last_time = current_time
        # Update LEDs here
        LightsManager.write()
```

### Cleanup in onPause()

Always clear LEDs when your app exits:

```python
def onPause(self, screen):
    # Stop animations
    mpos.ui.task_handler.remove_event_cb(self.update_frame)

    # Clear LEDs
    LightsManager.clear()
    LightsManager.write()
```

This prevents LEDs from staying lit after your app exits.

## Troubleshooting

### LEDs Not Updating

**Symptom**: Called `set_led()` or `set_all()` but LEDs don't change

**Cause**: Forgot to call `write()` to apply changes

**Solution**: Always call `write()` after setting LED colors

```python
# ❌ Wrong - no write()
LightsManager.set_all(255, 0, 0)

# ✅ Correct
LightsManager.set_all(255, 0, 0)
LightsManager.write()
```

### Flickering LEDs

**Symptom**: LEDs flicker or show inconsistent colors during animation

**Possible causes:**

1. **Update rate too high**
   ```python
   # ❌ Bad - updating every frame (60 Hz)
   def update_frame(self, a, b):
       LightsManager.set_all(random_color())
       LightsManager.write()  # Too frequent!

   # ✅ Good - rate limited to 20 Hz
   def update_frame(self, a, b):
       if delta_time >= 0.05:  # 50ms = 20 Hz
           LightsManager.set_all(random_color())
           LightsManager.write()
   ```

2. **Inconsistent timing**
   ```python
   # ✅ Use delta time for smooth animations
   delta_time = time.ticks_diff(current_time, self.last_time) / 1000.0
   ```

3. **Race condition with multiple updates**
   - Only update LEDs from one place (e.g., single update_frame callback)
   - Avoid updating from multiple threads

### LEDs Stay On After App Exits

**Symptom**: LEDs remain lit when you leave the app

**Cause**: Didn't clear LEDs in `onPause()`

**Solution**: Always implement cleanup in `onPause()`

```python
def onPause(self, screen):
    # Stop animation callback
    mpos.ui.task_handler.remove_event_cb(self.update_frame)

    # Clear LEDs
    LightsManager.clear()
    LightsManager.write()
```

### No LEDs on Waveshare Board

**Symptom**: `is_available()` returns `False` on Waveshare

**Cause**: Waveshare ESP32-S3-Touch-LCD-2 has no NeoPixel LEDs

**Solution**:
- Check hardware before using LEDs:
  ```python
  if LightsManager.is_available():
      # Use LEDs
  else:
      # Fallback to screen indicators or sounds
  ```

### Colors Look Wrong

**Symptom**: LEDs show unexpected colors

**Possible causes:**

1. **RGB order confusion** - Make sure you're using (R, G, B) order:
   ```python
   LightsManager.set_led(0, 255, 0, 0)  # Red (R=255, G=0, B=0)
   LightsManager.set_led(1, 0, 255, 0)  # Green (R=0, G=255, B=0)
   LightsManager.set_led(2, 0, 0, 255)  # Blue (R=0, G=0, B=255)
   ```

2. **Value out of range** - RGB values must be 0-255:
   ```python
   # ❌ Wrong - values > 255 wrap around
   LightsManager.set_led(0, 300, 0, 0)

   # ✅ Correct - clamp to 0-255
   LightsManager.set_led(0, min(300, 255), 0, 0)
   ```

## Complete Example: LED Status Indicator

```python
from mpos import Activity
import mpos.lights as LightsManager
import lvgl as lv

class StatusIndicatorActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()

        # Buttons for different statuses
        success_btn = lv.button(self.screen)
        success_btn.set_size(100, 50)
        success_btn.set_pos(10, 10)
        lv.label(success_btn).set_text("Success")
        success_btn.add_event_cb(
            lambda e: self.show_status("success"),
            lv.EVENT.CLICKED,
            None
        )

        error_btn = lv.button(self.screen)
        error_btn.set_size(100, 50)
        error_btn.set_pos(120, 10)
        lv.label(error_btn).set_text("Error")
        error_btn.add_event_cb(
            lambda e: self.show_status("error"),
            lv.EVENT.CLICKED,
            None
        )

        warning_btn = lv.button(self.screen)
        warning_btn.set_size(100, 50)
        warning_btn.set_pos(10, 70)
        lv.label(warning_btn).set_text("Warning")
        warning_btn.add_event_cb(
            lambda e: self.show_status("warning"),
            lv.EVENT.CLICKED,
            None
        )

        clear_btn = lv.button(self.screen)
        clear_btn.set_size(100, 50)
        clear_btn.set_pos(120, 70)
        lv.label(clear_btn).set_text("Clear")
        clear_btn.add_event_cb(
            lambda e: self.show_status("clear"),
            lv.EVENT.CLICKED,
            None
        )

        self.setContentView(self.screen)

    def show_status(self, status_type):
        """Show visual status with LEDs."""
        if not LightsManager.is_available():
            print("No LEDs available")
            return

        if status_type == "success":
            LightsManager.set_notification_color("green")
        elif status_type == "error":
            LightsManager.set_notification_color("red")
        elif status_type == "warning":
            LightsManager.set_notification_color("yellow")
        elif status_type == "clear":
            LightsManager.clear()
            LightsManager.write()

    def onPause(self, screen):
        # Clear LEDs when leaving app
        if LightsManager.is_available():
            LightsManager.clear()
            LightsManager.write()
```

## See Also

- [Creating Apps](../apps/creating-apps.md) - Learn how to create MicroPythonOS apps
- [WidgetAnimator](widget-animator.md) - Smooth UI animations
- [AudioManager](audiomanager.md) - Audio control for sound feedback
- [SensorManager](sensor-manager.md) - Sensor integration for interactive effects
