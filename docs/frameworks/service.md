# Service

MicroPythonOS provides a **Service** framework for background operations that run independently of the UI, inspired by Android's Service model. Services are ideal for WiFi auto-connect, web server startup, async REPL tasks, periodic update checks, and similar boot-time or long-running background work.

## Overview

Services differ from Activities in key ways:

- **No user interface** — A Service has no screen, no LVGL objects, and no lifecycle tied to visibility.
- **Independent lifecycle** — Services are created, started, and destroyed by the system without user interaction.
- **Boot-time execution** — Services can subscribe to the `boot_completed` intent to run automatically at system startup.
- **Long-running** — A Service can run indefinitely (e.g., polling for updates) or terminate after completing a task.

```
┌───────────────────────────────────────────┐
│              Service Lifecycle              │
├───────────────────────────────────────────┤
│                                            │
│    ┌────────────┐                          │
│    │  onCreate  │  ← Initialize resources  │
│    └─────┬──────┘                          │
│          │                                  │
│          ▼                                  │
│    ┌────────────┐                          │
│    │  onStart   │  ← Receive intent, work  │
│    └─────┬──────┘                          │
│          │                                  │
│          ▼                                  │
│    ┌────────────┐                          │
│    │ onDestroy  │  ← Cleanup resources     │
│    └────────────┘                          │
│                                            │
└───────────────────────────────────────────┘
```

## Service Base Class

The `Service` class is defined in `mpos.app.service` and re-exported from `mpos`:

```python
from mpos import Service

class MyService(Service):

    def __init__(self):
        super().__init__()

    def onCreate(self):
        # One-time initialization
        pass

    def onStart(self, intent=None):
        # Service is starting — do the work
        pass

    def onDestroy(self):
        # Clean up resources
        pass
```

### Lifecycle Methods

| Method | When Called | Purpose |
|--------|-------------|---------|
| `onCreate()` | After instantiation, before `onStart()` | One-time setup (allocating resources, opening files) |
| `onStart(intent)` | After `onCreate()` when the service is started | Perform the service's work; `intent` contains the action that triggered it |
| `onDestroy()` | When the service is being shut down | Release all resources, stop threads, cancel tasks |

### Properties

- `self.appFullName` — Set automatically by the system before `onCreate()`. Contains the fullname of the app that owns this service (e.g., `"com.micropythonos.osupdate"`). Use it instead of hard-coding the package name.

## Declaring Services

Services can be registered in two ways:

### 1. Programmatic Registration (System Services)

System services register themselves directly with `AppManager.register_service()` at module level:

```python
from mpos import Service
from mpos.content.app_manager import AppManager

class WifiBootService(Service):

    def onStart(self, intent):
        import _thread
        from mpos import WifiService, TaskManager
        _thread.stack_size(TaskManager.good_stack_size())
        _thread.start_new_thread(WifiService.auto_connect, ())

# Register at module level
AppManager.register_service("boot_completed", WifiBootService, fullname="com.micropythonos.system")
```

Parameters for `register_service()`:

- `action` (str) — Intent action string. Use `"boot_completed"` for boot-time services.
- `service_cls` (type) — The Service subclass.
- `fullname` (str, optional) — App fullname to associate with the service. Defaults to `None`.

### 2. Manifest-Declared Services (App Services)

Apps declare services in their `META-INF/MANIFEST.JSON` under a `"services"` array:

```json
{
  "name": "OSUpdate",
  "fullname": "com.micropythonos.osupdate",
  "version": "0.1.5",
  "activities": [
    {
      "entrypoint": "assets/osupdate.py",
      "classname": "OSUpdate",
      "intent_filters": [
        { "action": "main", "category": "launcher" }
      ]
    }
  ],
  "services": [
    {
      "entrypoint": "assets/osupdate_boot_service.py",
      "classname": "OSUpdateService",
      "intent_filters": [
        { "action": "boot_completed" }
      ]
    }
  ]
}
```

Each service entry requires:

| Field | Description |
|-------|-------------|
| `entrypoint` | Path to the Python file (relative to the app root) |
| `classname` | Name of the Service subclass in that file |
| `intent_filters` | Array of `{ "action": "..." }` objects. `"boot_completed"` triggers the service at startup |

The corresponding service code:

```python
# assets/osupdate_boot_service.py
from mpos import Service, ConnectivityManager, TaskManager

class OSUpdateService(Service):

    def __init__(self):
        super().__init__()
        self._running = False

    def onStart(self, intent):
        self._running = True
        TaskManager.create_task(self._boot_loop())

    def onDestroy(self):
        self._running = False

    async def _boot_loop(self):
        cm = ConnectivityManager.get()
        while self._running:
            await TaskManager.sleep(30)
            if cm.is_online():
                print("OSUpdateService: network connected")
            else:
                print("OSUpdateService: network not connected")
```

## Boot Services

Services that subscribe to the `boot_completed` intent are started automatically during system boot. The current boot-time service set includes:

| Service | App | Purpose |
|---------|-----|---------|
| `WifiBootService` | `com.micropythonos.system` | Auto-connects WiFi in a background thread |
| `WebServerBootService` | `com.micropythonos.system` | Starts the HTTP web server if enabled |
| `AIOReplService` | `com.micropythonos.system` | Starts the asyncio REPL task for interactive debugging |
| `OSUpdateService` | `com.micropythonos.osupdate` | Periodically checks network connectivity for OTA updates |

### Boot Order

Services are started after the launcher is displayed, in this sequence:

1. Launcher Activity is created and displayed
2. `auto_start_app_early()` runs (early-start apps)
3. `auto_start_app()` runs (normal auto-start apps)
4. **Boot services are started** — all `boot_completed` services are instantiated and `onStart()` is called
5. `TaskManager.start()` runs the asyncio event loop

The order among boot services is non-deterministic.

## Complete Example

### System Service

```python
# my_service.py
from mpos import Service
from mpos.content.app_manager import AppManager

class BootLogger(Service):

    def __init__(self):
        super().__init__()

    def onCreate(self):
        print(f"BootLogger: initialized from {self.appFullName}")

    def onStart(self, intent):
        print(f"BootLogger: received intent action={intent.action}")
        print("BootLogger: system boot is complete")

    def onDestroy(self):
        print("BootLogger: shutting down")

AppManager.register_service("boot_completed", BootLogger, fullname="com.example.myservice")
```

### App Service (manifest-declared)

```
com.example.myapp/
├── META-INF/
│   └── MANIFEST.JSON      # Contains "services" array
├── assets/
│   ├── main.py            # Activity entry point
│   └── my_service.py      # Service entry point
└── res/
    └── mipmap-mdpi/
        └── icon_64x64.png
```

## Service vs. Activity

| Aspect | Activity | Service |
|--------|----------|---------|
| UI | Has a screen with LVGL widgets | No UI |
| Lifecycle triggers | User navigation, back button | Intent actions (e.g., `boot_completed`) |
| Persistence | Exists only while in the activity stack | Can run indefinitely after boot |
| Foreground state | Tracked via `has_foreground()` | Not applicable |
| Thread usage | UI runs on main LVGL thread | May spawn threads or use asyncio tasks |

## Best Practices

### Do's

✅ Use `self.appFullName` instead of hard-coding the package name  
✅ Call `super().__init__()` in your `__init__` method  
✅ Clean up threads and tasks in `onDestroy()`  
✅ Use `TaskManager.create_task()` for async operations in services  
✅ Keep `onCreate()` lightweight — do heavy work in `onStart()`  

### Don'ts

❌ Don't create LVGL UI objects in a Service (no screen context)  
❌ Don't block `onStart()` with synchronous waiting — use asyncio or threads  
❌ Don't forget to stop background tasks when the service is destroyed  
❌ Don't assume boot services run in a specific order  

## See Also

- [AppManager](../frameworks/app-manager.md) — Service registration and boot management
- [App Lifecycle](../apps/app-lifecycle.md) — Activity lifecycle for UI apps
- [Boot Sequence](../architecture/boot-sequence.md) — Detailed boot flow
- [TaskManager](../frameworks/task-manager.md) — Async task management for services
- [Creating Apps](../apps/creating-apps.md) — How to add services to your app's manifest
