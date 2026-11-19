# MultiTouchGesture

`MultiTouchGesture` turns two fingers into pinch scale, rotation and pan. It
works with the touch controllers that report more than one point; the second
finger is gesture-only, so it never clicks the widget underneath it.

## Supported hardware

| Board | Controller | Touch points |
|---|---|---|
| Makerfabs MaTouch ESP32-S3 2.8 | GT911 | 5 |
| Waveshare ESP32-P4-WIFI6-Touch-LCD-4.3 | GT911 | 5 |
| D-Shell | GT911 | 5 |
| Waveshare ESP32-S3-Touch-LCD-3.5 | FT6336 | 2 |
| fri3d 2026 | CST816S | 1 — no multi-touch |
| Waveshare ESP32-S3-Touch-LCD-2 / 1.54 | CST816S | 1 — no multi-touch |
| unphone | XPT2046 | 1 — no multi-touch |

Single-touch boards keep working exactly as before: `get_touches()` is simply
absent, `MultiTouchGesture.indev` is `None`, and gestures stay disabled.

## Overview

The touch driver exposes every point of the last report:

```python
driver.get_touches()      # [(id, x, y), ...] or [] when nothing is down
driver.get_touch_count()  # len(get_touches())
```

`id` is `None` for controllers that do not report one (the GT911 on the MaTouch
does not). The coordinates are already passed through the driver's calibration
and startup rotation, i.e. they are in LVGL's pre-display-rotation space.

`MultiTouchGesture` converts them to logical coordinates, applies LVGL's display
rotation, and tracks the pair:

```python
from mpos import MultiTouchGesture

gesture = MultiTouchGesture()      # finds the pointer indev with get_touches()
gesture.poll()
```

Poll it from an LVGL timer while the screen is visible:

```python
import lvgl as lv

class Zoomable(SomeActivity):
    def onCreate(self):
        self.gesture = MultiTouchGesture()
        self.gesture_timer = None

    def onResume(self, screen):
        if self.gesture.indev is not None and self.gesture_timer is None:
            self.gesture.reset()
            self.gesture_timer = lv.timer_create(self._poll_gesture, 30, None)
            self.gesture_timer.set_repeat_count(-1)

    def onPause(self, screen):
        if self.gesture_timer is not None:
            self.gesture_timer.delete()
            self.gesture_timer = None

    def _poll_gesture(self, timer):
        gesture = self.gesture
        gesture.poll()
        if gesture.fingers >= 2 and gesture.active:
            # gesture.scale, gesture.rotation, gesture.center
            ...
```

## Gesture state

After `poll()`:

| Attribute | Meaning |
|---|---|
| `fingers` | Number of points in the last report |
| `active` | A pair is being tracked (the fingers are far enough apart) |
| `just_started` / `just_ended` | The pair became active / the gesture ended on this poll |
| `scale` | Cumulative pinch factor since the pair became active (`1.0` = no change) |
| `rotation` | Cumulative rotation since the pair became active, in radians, normalised to `-pi..pi` |
| `center` | Current midpoint in logical coordinates |
| `delta_x` / `delta_y` | Midpoint movement since the previous poll |
| `points` | Logical `(x, y)` points of the last report |

A pair closer together than `_MIN_PAIR_DISTANCE` (12 px) is ignored until the
fingers spread, which keeps jitter from starting a gesture. Point order is kept
stable by matching ids when a driver provides them and by nearest previous point
otherwise, so the scale does not jump when a controller reorders slots.

## Applying it to an image

`lv.image` can scale, rotate and pan inside a fixed viewport:

```python
image.set_inner_align(lv.image.ALIGN.CENTER)
image.set_size(viewport_width, viewport_height)
image.set_scale(round(fit_scale * zoom))       # 256 = 1:1
image.set_rotation(angle_in_tenths_of_degree)
image.set_offset_x(offset_x)
image.set_offset_y(offset_y)
```

With `ALIGN.CENTER` the decoded image is centred in the object and the object
clips it. `set_scale`/`set_rotation` transform around the decoded image's centre
pivot, and `set_offset_x/y` pan. To keep the content point under the gesture
midpoint fixed while zooming:

```
offset = pointer - image_base - pivot - R(angle) * (content_point - pivot) * scale
```

The Image View app (`com.micropythonos.imageview`) is the reference
implementation: 1× fit to 4× zoom, two-finger rotation, one-finger pan, and a
plain tap still toggles fullscreen because a pan does not produce a click.

## Pointer event suppression

When a second finger lands, the driver calls `indev.wait_release()` and reports
`RELEASED` to LVGL until every finger lifts. LVGL sends `PRESS_LOST` to the
pressed widget and stops processing that contact, so a pinch cannot trigger a
button, a click, or the back-swipe gesture. The system gesture strips handle
`PRESS_LOST` by hiding their hint button.

One-finger input is unchanged: the driver's first point still drives the normal
LVGL pointer path, and `InputManager.pointer_xy()` and all widget events keep
working as before.

## Adding multi-touch to another driver

A driver opts in by implementing:

```python
def get_touches(self):
    return [(id, x, y), ...]

def get_touch_count(self):
    return len(self._touch_points)
```

where the coordinates have already been passed through `_calc_coords()`. Keep
the single-point path in `_get_coords()` returning raw coordinates; `_read()`
applies `_calc_coords()` to them as it always did.

Two-finger suppression belongs in the driver, because a framework timer polls
more slowly than LVGL's input tick. Set `_multi_touch_active`, call
`wait_release()` once per contact, and keep reporting `RELEASED` until the
controller reports zero points.
