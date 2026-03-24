# WebServer

MicroPythonOS provides a **WebServer** helper that wraps WebREPL-based HTTP serving with saved settings. It exposes a simple class API for starting/stopping the service and inspecting configuration.

## Overview

WebServer provides:

- **Saved settings** - Port, password, and autostart are stored in SharedPreferences
- **Simple control API** - Start/stop/status helpers
- **WebREPL HTTP bridge** - Uses the WebREPL HTTP accept handler

## Quick Start

```python
from mpos import WebServer

# Start server using saved settings
WebServer.start()

# Check status
print(WebServer.status())

# Stop server
WebServer.stop()
```

## Settings

Settings live in SharedPreferences under `com.micropythonos.settings.webserver`:

- `autostart` (string: "True"/"False")
- `port` (string: "7890")
- `password` (string, max 9 chars)

## API Reference

### `WebServer.status()`

Return a dict with:

- `state` ("started" or "stopped")
- `started` (bool)
- `port` (int)
- `password` (string)
- `autostart` (bool)
- `last_error` (exception or None)

### `WebServer.start()`

Load settings and start the WebREPL HTTP server.

### `WebServer.stop()`

Stop the server if running.

### `WebServer.apply_settings(restart_if_running=True)`

Reload settings and restart the server if it was already running.

### `WebServer.auto_start()`

Start the server only if `autostart` is enabled in preferences.

## Implementation Details

- **Location:** `MicroPythonOS/internal_filesystem/lib/mpos/webserver/webserver.py`
- **Exports:** `from mpos import WebServer`

## See Also

- [WifiService](wifi-service.md) - Network connectivity
- [ConnectivityManager](connectivity-manager.md) - Connectivity state
