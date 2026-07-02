# NotificationManager

`NotificationManager` is a system-wide framework for posting, displaying, and dispatching notifications. It is inspired by Android's notification system, but tailored for resource-constrained devices.

Notifications appear in the top notification bar and in the pull-down drawer. Tapping a notification dispatches its attached `Intent`, which can open an activity or start an app.

## Overview

- Apps post `Notification` objects through `NotificationManager.notify()`.
- The system UI listens for changes and updates the notification bar and drawer automatically.
- Notifications are persisted to storage (with a debounced write) so they survive reboots.
- Notifications can be cancelled individually, in bulk, or automatically when tapped.

## Basic Usage

```python
from mpos import NotificationManager, Notification, Intent

notification = Notification(
    notification_id="myapp.event",
    icon=lv.SYMBOL.BELL,
    title="Event",
    text="Something happened!",
    intent=Intent(action="main", app_fullname="com.example.myapp"),
    auto_cancel=True,
)

NotificationManager.notify(notification)
```

To remove a notification:

```python
NotificationManager.cancel("myapp.event")
```

## Notification Object

`Notification` is a plain data object that describes a single notification.

### Constructor Arguments

| Argument | Type | Description |
|----------|------|-------------|
| `notification_id` | `str` | Unique identifier. Also accepts `uniqueidString` for compatibility. Required. |
| `icon` | `str` or `lv.image_dsc_t` | Icon shown in the bar/drawer. Can be an `lv.SYMBOL.*` string, another string, or an LVGL image descriptor. |
| `title` | `str` | Short title. |
| `text` | `str` | Longer body text. |
| `priority` | `int` | One of `Notification.PRIORITY_MIN`, `PRIORITY_LOW`, `PRIORITY_DEFAULT`, `PRIORITY_HIGH`, `PRIORITY_MAX`. Higher priority notifications are shown first. |
| `intent` | `Intent` | `Intent` to dispatch when the user taps the notification. |
| `auto_cancel` | `bool` | If `True` (default), the notification is cancelled automatically after it is tapped. |
| `app_fullname` | `str` | App that owns the notification. Used as a fallback target if the intent has no explicit target. |

### Priority Levels

```python
Notification.PRIORITY_MIN     = -1
Notification.PRIORITY_LOW     =  0
Notification.PRIORITY_DEFAULT =  1
Notification.PRIORITY_HIGH    =  2
Notification.PRIORITY_MAX     =  3
```

The drawer sorts notifications by priority, then by most recent update time.

## Notification sounds

When a new notification arrives, the system can play a short RTTTL ringtone on the device's buzzer output. The sound is stored as an RTTTL string in the Settings app's `SharedPreferences` under the key `notification_sound`.

Available sounds:

| Label | RTTTL string |
|-------|--------------|
| `None` | (empty string, no sound) |
| `Coin` | `coin:d=8,o=6,b=200:16b5,e6` |
| `Scale up` | `scale_up:d=32,o=5,b=100:c,c#,d#,e,f#,g#,a#,b` |
| `Superhappy` | `superhappy:d=8,o=5,b=635:c,e,g,c,e,g,c,e,g,c6,e6,g6,c6,e6,g6,c7,e7,g7,c7,e7,g7,c7,e7,g7` |

The default sound is `Coin`. If no buzzer output is available, the notification posts silently. Apps can change the sound programmatically:

```python
from mpos import SharedPreferences

prefs = SharedPreferences("com.micropythonos.settings")
prefs.put_string("notification_sound", "coin:d=8,o=6,b=200:16b5,e6")
prefs.commit()
```

## NotificationManager API

All methods are class methods. `NotificationManager` initializes itself lazily on first use.

### `notify(notification)`

Post or update a notification.

- If a notification with the same ID already exists, its content is updated in place without a new persistence flash.
- If it is new, it is added, the list is trimmed to `MAX_NOTIFICATIONS` (20), and persistence is scheduled.

```python
NotificationManager.notify(Notification(
    notification_id="osupdate.available",
    icon=lv.SYMBOL.DOWNLOAD,
    title="Update available",
    text="MicroPythonOS 1.2.3 is ready to install",
    priority=Notification.PRIORITY_HIGH,
))
```

### `cancel(notification_id)`

Remove a single notification. Writes immediately so the notification does not reappear after reboot.

```python
NotificationManager.cancel("osupdate.available")
```

### `cancel_all()`

Remove all notifications.

```python
NotificationManager.cancel_all()
```

### `get_notifications()`

Return a sorted list of active notifications, highest priority first.

```python
for n in NotificationManager.get_notifications():
    print(n.title, n.text)
```

### `get_notification(notification_id)`

Return a single notification by ID, or `None` if it doesn't exist.

```python
n = NotificationManager.get_notification("osupdate.available")
```

### `trigger(notification_id_or_object)`

Dispatch the notification's intent. This is what the system calls when the user taps a notification in the drawer.

- If the intent has an explicit `activity_class` or `action`, `ActivityNavigator.startActivity()` is used.
- Otherwise, the owning `app_fullname` is started via `AppManager.start_app()`.
- If `auto_cancel` is enabled and the dispatch succeeds, the notification is cancelled.

```python
NotificationManager.trigger("osupdate.available")
```

### `register_listener(callback, notify_immediately=True)` / `unregister_listener(callback)`

Register a function to be called whenever notifications change. The top menu/drawer uses this to refresh the UI.

```python
def on_notifications_changed():
    print("Notifications updated")

NotificationManager.register_listener(on_notifications_changed)
```

## Complete Example

```python
import lvgl as lv
from mpos import Activity, NotificationManager, Notification, Intent

class MyApp(Activity):

    def show_notification(self):
        intent = Intent(action="main", app_fullname="com.example.myapp")
        notification = Notification(
            notification_id="com.example.myapp.done",
            icon=lv.SYMBOL.OK,
            title="Done",
            text="The operation finished successfully.",
            intent=intent,
            auto_cancel=True,
            app_fullname="com.example.myapp",
        )
        NotificationManager.notify(notification)
```

## Best Practices

### Do's

✅ Use a stable, unique `notification_id` so the same event doesn't create duplicate notifications  
✅ Set a meaningful `app_fullname` so tapping the notification can fall back to launching your app  
✅ Cancel notifications when they are no longer relevant  
✅ Use `auto_cancel=True` for one-shot notifications that should disappear after being tapped  

### Don'ts

❌ Don't rely on `lv.image_dsc_t` icons surviving a reboot — only string icons are persisted  
❌ Don't post large amounts of text; storage and display space are limited  
❌ Don't use more than `MAX_NOTIFICATIONS` (20) active notifications  

## See Also

- [Service](service.md) — Background services that often post notifications at boot
- [SettingActivity](setting-activity.md) — Per-app settings storage, useful with notification preferences
- [App Lifecycle](../apps/app-lifecycle.md) — Foreground handling when notifications launch your app
