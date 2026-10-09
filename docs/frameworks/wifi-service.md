# WifiService

MicroPythonOS provides a centralized WiFi management service called **WifiService** that handles WiFi connections, hotspot mode, network scanning, and credential storage. It's designed to work alongside ConnectivityManager for comprehensive network management.

## Overview

WifiService provides:

- **Auto-connect on boot** - Automatically connects to saved networks when device starts
- **Hotspot mode** - Enable/disable AP mode with saved settings
- **Network scanning** - Scan for available WiFi networks
- **Credential management** - Save, retrieve, and forget network passwords
- **Hidden network support** - Connect to networks that don't broadcast SSID
- **Concurrent access locking** - Thread-safe WiFi operations
- **Desktop simulation** - Mock WiFi for desktop testing
- **ADC2 compatibility** - Temporarily disable WiFi for ESP32-S3 ADC2 operations
- **IP helpers** - Get IPv4 address, netmask, and gateway

## Quick Start

### Checking Connection Status

```python
from mpos import WifiService

if WifiService.is_connected():
    ssid = WifiService.get_current_ssid()
    print(f"Connected to: {ssid}")
else:
    print("Not connected to WiFi")
```

### Scanning for Networks

```python
from mpos import WifiService

networks = WifiService.scan_networks()
if networks is None:
    print("WiFi is busy, try again later")
else:
    for ssid in networks:
        print(f"Found network: {ssid}")
```

`scan_networks()` returns `None`, not an empty list, while another WiFi operation is running (see [Busy Flag](#busy-flag)).

### Connecting to a Network

```python
from mpos import WifiService

WifiService.save_network("MyNetwork", "password123")

success = WifiService.attempt_connecting("MyNetwork", "password123")
if success is None:
    print("WiFi is busy, try again later")
elif success:
    print("Connected!")
else:
    print("Connection failed")
```

`attempt_connecting()` returns `None` without trying when another WiFi operation is running. `None` is falsy, so code that only checks `if success:` treats it as a failed attempt.

### Managing Saved Networks

```python
from mpos import WifiService

WifiService.save_network("HomeWiFi", "mypassword")
WifiService.save_network("HiddenNetwork", "secret", hidden=True)

saved = WifiService.get_saved_networks()
print(f"Saved networks: {saved}")

password = WifiService.get_network_password("HomeWiFi")

WifiService.forget_network("OldNetwork")
```

## Hotspot Mode

WifiService supports Access Point mode using settings stored in SharedPreferences:

- Namespace: `com.micropythonos.settings.hotspot`
- Keys: `enabled`, `ssid`, `password`, `authmode`

```python
from mpos import WifiService

# Enable hotspot
WifiService.enable_hotspot()

# Disable hotspot
WifiService.disable_hotspot()

# Check hotspot state
if WifiService.is_hotspot_enabled():
    print("Hotspot is active")
```

### Auto-connect vs Hotspot

When hotspot is enabled in preferences, `auto_connect()` skips STA mode and starts the hotspot instead.

## Common Patterns

### WiFi Settings Screen

```python
from mpos import Activity, WifiService
import lvgl as lv

class WifiSettingsActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()

        self.status = lv.label(self.screen)
        self.update_status()
        self.status.align(lv.ALIGN.TOP_MID, 0, 10)

        scan_btn = lv.button(self.screen)
        scan_btn.set_size(120, 40)
        scan_btn.align(lv.ALIGN.TOP_MID, 0, 50)
        scan_label = lv.label(scan_btn)
        scan_label.set_text("Scan")
        scan_label.center()
        scan_btn.add_event_cb(self.on_scan, lv.EVENT.CLICKED, None)

        self.network_list = lv.list(self.screen)
        self.network_list.set_size(280, 150)
        self.network_list.align(lv.ALIGN.CENTER, 0, 30)

        self.setContentView(self.screen)

    def update_status(self):
        if WifiService.is_connected():
            ssid = WifiService.get_current_ssid()
            self.status.set_text(f"Connected: {ssid}")
        elif WifiService.is_hotspot_enabled():
            self.status.set_text("Hotspot active")
        else:
            self.status.set_text("Not connected")

    def on_scan(self, event):
        networks = WifiService.scan_networks()
        if networks is None:
            print("WiFi is busy")  # keep the current list
            return

        self.network_list.clean()
        saved = WifiService.get_saved_networks()

        for ssid in networks:
            btn = self.network_list.add_button(None, ssid)
            if ssid in saved:
                btn.set_style_bg_color(lv.color_hex(0x4CAF50), 0)
            btn.add_event_cb(
                lambda e, s=ssid: self.on_network_selected(s),
                lv.EVENT.CLICKED,
                None
            )

    def on_network_selected(self, ssid):
        password = WifiService.get_network_password(ssid)
        if password:
            success = WifiService.attempt_connecting(ssid, password)
            if success:
                self.update_status()
```

### Background Auto-Connect

```python
import _thread
from mpos import WifiService, TaskManager

_thread.stack_size(TaskManager.good_stack_size())
_thread.start_new_thread(WifiService.auto_connect, ())
```

**Auto-connect behavior:**
1. Loads saved networks from SharedPreferences
2. Checks if WiFi is busy before proceeding
3. Tries saved networks in order of signal strength
4. Also tries hidden networks
5. Syncs time via NTP on success
6. Disables WiFi if no networks found (power saving)
7. Restores hotspot if it was active before attempting STA mode

### Temporarily Disabling WiFi for ADC2

On ESP32-S3, ADC2 pins (GPIO11-20) don't work when WiFi is active:

```python
from mpos import WifiService

try:
    was_connected = WifiService.temporarily_disable()
    # ADC2 operations here
finally:
    WifiService.temporarily_enable(was_connected)
```

## IP Address Helpers

```python
from mpos import WifiService

print(WifiService.get_ipv4_address())
print(WifiService.get_ipv4_netmask())
print(WifiService.get_ipv4_gateway())
```

## Busy Flag

WifiService runs one radio operation at a time. `auto_connect()`, `attempt_connecting()`, `scan_networks()` and `temporarily_disable()` each hold the class-level `wifi_busy` flag while they run. While another operation holds it:

| Method | While busy |
|--------|------------|
| `attempt_connecting()` | Returns `None` without trying |
| `scan_networks()` | Returns `None` (a scan that finds nothing returns `[]`) |
| `auto_connect()` | Returns without connecting |
| `temporarily_disable()` | Raises `RuntimeError` |
| `enable_hotspot()` | Returns `False` |
| `is_connected()` | Returns `False` |
| `get_ipv4_address()`, `get_ipv4_netmask()`, `get_ipv4_gateway()` | Return `None` |

ConnectivityManager's periodic reconnect is skipped while the flag is held, so it can't interrupt a scan or a connect started by an app.

`is_busy()` tells you whether an operation is running, but the flag can change between that check and your call. Check the `None` return values instead of relying on `is_busy()` alone.

## API Reference

### Connection Functions

#### `WifiService.connect(network_module=None, time_module=None)`

Scan for available networks and connect to the first saved network found.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)
- `time_module` - Time module for dependency injection (testing)

**Returns:**
- `bool` - `True` if successfully connected, `False` otherwise

---

#### `WifiService.attempt_connecting(ssid, password, network_module=None, time_module=None)`

Attempt to connect to a specific WiFi network. Holds the [busy flag](#busy-flag) while it runs.

**Parameters:**
- `ssid` (str) - Network SSID to connect to
- `password` (str) - Network password
- `network_module` - Network module for dependency injection (testing)
- `time_module` - Time module for dependency injection (testing)

**Returns:**
- `bool` - `True` if successfully connected, `False` otherwise
- `None` - another WiFi operation is running; nothing was attempted

---

#### `WifiService.auto_connect(network_module=None, time_module=None)`

Auto-connect to a saved WiFi network on boot.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)
- `time_module` - Time module for dependency injection (testing)

---

#### `WifiService.disconnect(network_module=None)`

Disconnect from current WiFi network and disable WiFi (also disables hotspot).

---

#### `WifiService.is_connected(network_module=None)`

Check if WiFi is currently connected. Returns `True` for hotspot mode when AP is active.

### Scanning Functions

#### `WifiService.scan_networks(network_module=None)`

Scan for available WiFi networks. Holds the [busy flag](#busy-flag) while it runs.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `list` - SSIDs found (empty if none were found)
- `None` - another WiFi operation is running; nothing was scanned

---

#### `WifiService.is_busy()`

Return `True` while a WiFi operation holds the [busy flag](#busy-flag).

### Hotspot Functions

#### `WifiService.enable_hotspot(network_module=None)`

Enable hotspot mode using settings stored in `com.micropythonos.settings.hotspot`.

#### `WifiService.disable_hotspot(network_module=None)`

Disable hotspot mode.

#### `WifiService.is_hotspot_enabled(network_module=None)`

Check if hotspot mode is currently enabled.

### IP Helpers

#### `WifiService.get_ipv4_address(network_module=None)`

Return IPv4 address for STA or AP mode.

#### `WifiService.get_ipv4_netmask(network_module=None)`

Return IPv4 netmask for STA or AP mode.

#### `WifiService.get_ipv4_gateway(network_module=None)`

Return IPv4 gateway for STA or AP mode.

### ADC2 Compatibility

#### `WifiService.temporarily_disable(network_module=None)`

Temporarily disable WiFi for operations that require it (e.g., ESP32-S3 ADC2).

#### `WifiService.temporarily_enable(was_connected, network_module=None)`

Re-enable WiFi after temporary disable operation. Restores hotspot if it was active before.

## Desktop Mode

On desktop (Linux/macOS), WifiService provides simulated behavior for testing:

| Method | Desktop Behavior |
|--------|------------------|
| `is_connected()` | Always returns `True` |
| `scan_networks()` | Returns mock SSIDs (ignores the busy flag) |
| `attempt_connecting()` | Simulates 2-second connection delay, succeeds unless WiFi is busy |
| `get_current_ssid()` | Returns simulated connected SSID |
| `disconnect()` | Prints message, no-op |
| `enable_hotspot()` | Simulated hotspot state |

## Credential Storage

Network credentials are stored using SharedPreferences:

- **Preference name:** `com.micropythonos.system.wifiservice`
- **Key:** `access_points`
- **Format:** Dictionary `{ssid: {password: "...", hidden: bool}}`

**Storage location:** `data/com.micropythonos.system.wifiservice/prefs.json`

## Implementation Details

**Location:** `MicroPythonOS/internal_filesystem/lib/mpos/net/wifi_service.py`

**Pattern:** Static class with class-level state

**Key features:**
- `wifi_busy` - Class-level lock for concurrent access
- `access_points` - Cached dictionary of saved networks
- `_desktop_connected_ssid` - Simulated SSID for desktop mode
- `hotspot_enabled` - Hotspot state flag

## See Also

- [ConnectivityManager](connectivity-manager.md) - Network connectivity monitoring
- [DownloadManager](download-manager.md) - HTTP downloads
- [SharedPreferences](preferences.md) - Persistent storage
