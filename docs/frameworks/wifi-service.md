# WifiService

MicroPythonOS provides a centralized WiFi management service called **WifiService** that handles WiFi connections, network scanning, and credential storage. It's designed to work alongside ConnectivityManager for comprehensive network management.

## Overview

WifiService provides:

- **Auto-connect on boot** - Automatically connects to saved networks when device starts
- **Network scanning** - Scan for available WiFi networks
- **Credential management** - Save, retrieve, and forget network passwords
- **Hidden network support** - Connect to networks that don't broadcast SSID
- **Concurrent access locking** - Thread-safe WiFi operations
- **Desktop simulation** - Mock WiFi for desktop testing
- **ADC2 compatibility** - Temporarily disable WiFi for ESP32-S3 ADC2 operations

## Quick Start

### Checking Connection Status

```python
from mpos import WifiService

# Check if WiFi is connected
if WifiService.is_connected():
    ssid = WifiService.get_current_ssid()
    print(f"Connected to: {ssid}")
else:
    print("Not connected to WiFi")
```

### Scanning for Networks

```python
from mpos import WifiService

# Scan for available networks
networks = WifiService.scan_networks()

for ssid in networks:
    print(f"Found network: {ssid}")
```

**Note:** `scan_networks()` manages the busy flag automatically. If WiFi is already busy (e.g., connecting), it returns an empty list.

### Connecting to a Network

```python
from mpos import WifiService

# Save network credentials first
WifiService.save_network("MyNetwork", "password123")

# Connect to the network
success = WifiService.attempt_connecting("MyNetwork", "password123")

if success:
    print("Connected!")
else:
    print("Connection failed")
```

### Managing Saved Networks

```python
from mpos import WifiService

# Save a network
WifiService.save_network("HomeWiFi", "mypassword")

# Save a hidden network
WifiService.save_network("HiddenNetwork", "secret", hidden=True)

# Get list of saved networks
saved = WifiService.get_saved_networks()
print(f"Saved networks: {saved}")

# Get password for a network
password = WifiService.get_network_password("HomeWiFi")

# Forget a network
WifiService.forget_network("OldNetwork")
```

## Common Patterns

### WiFi Settings Screen

```python
from mpos import Activity, WifiService
import lvgl as lv

class WifiSettingsActivity(Activity):
    def onCreate(self):
        self.screen = lv.obj()
        
        # Status label
        self.status = lv.label(self.screen)
        self.update_status()
        self.status.align(lv.ALIGN.TOP_MID, 0, 10)
        
        # Scan button
        scan_btn = lv.button(self.screen)
        scan_btn.set_size(120, 40)
        scan_btn.align(lv.ALIGN.TOP_MID, 0, 50)
        scan_label = lv.label(scan_btn)
        scan_label.set_text("Scan")
        scan_label.center()
        scan_btn.add_event_cb(self.on_scan, lv.EVENT.CLICKED, None)
        
        # Network list
        self.network_list = lv.list(self.screen)
        self.network_list.set_size(280, 150)
        self.network_list.align(lv.ALIGN.CENTER, 0, 30)
        
        self.setContentView(self.screen)
    
    def update_status(self):
        if WifiService.is_connected():
            ssid = WifiService.get_current_ssid()
            self.status.set_text(f"Connected: {ssid}")
        else:
            self.status.set_text("Not connected")
    
    def on_scan(self, event):
        if WifiService.is_busy():
            print("WiFi is busy")
            return
        
        # Clear list
        self.network_list.clean()
        
        # Scan for networks
        networks = WifiService.scan_networks()
        saved = WifiService.get_saved_networks()
        
        for ssid in networks:
            btn = self.network_list.add_button(None, ssid)
            if ssid in saved:
                # Mark saved networks
                btn.set_style_bg_color(lv.color_hex(0x4CAF50), 0)
            btn.add_event_cb(
                lambda e, s=ssid: self.on_network_selected(s),
                lv.EVENT.CLICKED,
                None
            )
    
    def on_network_selected(self, ssid):
        # Show password dialog or connect if already saved
        password = WifiService.get_network_password(ssid)
        if password:
            success = WifiService.attempt_connecting(ssid, password)
            if success:
                self.update_status()
```

### Background Auto-Connect

WifiService automatically handles auto-connect on boot. This is typically started from `main.py`:

```python
import _thread
from mpos import WifiService
from mpos import TaskManager

# Start auto-connect in background thread
_thread.stack_size(TaskManager.good_stack_size())
_thread.start_new_thread(WifiService.auto_connect, ())
```

**Auto-connect behavior:**
1. Loads saved networks from SharedPreferences
2. Scans for available networks
3. Tries to connect to saved networks (strongest signal first)
4. Also tries hidden networks that don't appear in scan
5. Syncs time via NTP on successful connection
6. Disables WiFi if no networks found (power saving)

### Temporarily Disabling WiFi for ADC2

On ESP32-S3, ADC2 pins (GPIO11-20) don't work when WiFi is active. WifiService provides methods to temporarily disable WiFi:

```python
from mpos import WifiService

def read_adc2_sensor():
    """Read from ADC2 pin which requires WiFi to be disabled."""
    try:
        # Disable WiFi (raises RuntimeError if WiFi is busy)
        was_connected = WifiService.temporarily_disable()
        
        # Now safe to read ADC2
        from machine import ADC, Pin
        adc = ADC(Pin(15))  # GPIO15 is on ADC2
        value = adc.read()
        
        return value
        
    finally:
        # Re-enable WiFi (reconnects if was connected)
        WifiService.temporarily_enable(was_connected)
```

**Important:** Always call `temporarily_enable()` in a `finally` block to ensure WiFi is re-enabled.

## API Reference

### Connection Functions

#### `WifiService.connect(network_module=None)`

Scan for available networks and connect to the first saved network found.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `bool` - `True` if successfully connected, `False` otherwise

**Example:**
```python
if WifiService.connect():
    print("Connected to a saved network")
```

---

#### `WifiService.attempt_connecting(ssid, password, network_module=None, time_module=None)`

Attempt to connect to a specific WiFi network.

**Parameters:**
- `ssid` (str) - Network SSID to connect to
- `password` (str) - Network password
- `network_module` - Network module for dependency injection (testing)
- `time_module` - Time module for dependency injection (testing)

**Returns:**
- `bool` - `True` if successfully connected, `False` otherwise

**Behavior:**
- Waits up to 10 seconds for connection
- Syncs time via NTP on success
- Returns `False` if WiFi is disabled during connection

**Example:**
```python
success = WifiService.attempt_connecting("MyNetwork", "password123")
```

---

#### `WifiService.auto_connect(network_module=None, time_module=None)`

Auto-connect to a saved WiFi network on boot.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)
- `time_module` - Time module for dependency injection (testing)

**Behavior:**
- Loads saved networks from SharedPreferences
- Checks if WiFi is busy before proceeding
- Tries saved networks in order of signal strength
- Also tries hidden networks
- Disables WiFi if no connection made (power saving)

**Example:**
```python
import _thread
_thread.start_new_thread(WifiService.auto_connect, ())
```

---

#### `WifiService.disconnect(network_module=None)`

Disconnect from current WiFi network and disable WiFi.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Example:**
```python
WifiService.disconnect()
```

---

#### `WifiService.is_connected(network_module=None)`

Check if WiFi is currently connected.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `bool` - `True` if connected, `False` otherwise

**Note:** Returns `False` if WiFi operations are in progress (`wifi_busy` is `True`).

**Example:**
```python
if WifiService.is_connected():
    print("WiFi is connected")
```

---

#### `WifiService.get_current_ssid(network_module=None)`

Get the SSID of the currently connected network.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `str` or `None` - Current SSID if connected, `None` otherwise

**Example:**
```python
ssid = WifiService.get_current_ssid()
if ssid:
    print(f"Connected to: {ssid}")
```

### Scanning Functions

#### `WifiService.scan_networks(network_module=None)`

Scan for available WiFi networks.

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `list` - List of SSID strings found

**Behavior:**
- Manages `wifi_busy` flag automatically
- Returns empty list if WiFi is already busy
- Filters out empty SSIDs and invalid lengths
- Returns mock data on desktop

**Example:**
```python
networks = WifiService.scan_networks()
for ssid in networks:
    print(f"Found: {ssid}")
```

---

#### `WifiService.is_busy()`

Check if WiFi operations are currently in progress.

**Returns:**
- `bool` - `True` if WiFi is busy, `False` if available

**Example:**
```python
if not WifiService.is_busy():
    networks = WifiService.scan_networks()
```

### Credential Management

#### `WifiService.save_network(ssid, password, hidden=False)`

Save a new WiFi network credential.

**Parameters:**
- `ssid` (str) - Network SSID
- `password` (str) - Network password
- `hidden` (bool) - Whether this is a hidden network (default: `False`)

**Behavior:**
- Saves to SharedPreferences (`com.micropythonos.system.wifiservice`)
- Updates class-level cache
- Hidden networks are always tried during auto-connect

**Example:**
```python
# Save regular network
WifiService.save_network("HomeWiFi", "password123")

# Save hidden network
WifiService.save_network("SecretNetwork", "hidden_pass", hidden=True)
```

---

#### `WifiService.forget_network(ssid)`

Remove a saved WiFi network.

**Parameters:**
- `ssid` (str) - Network SSID to forget

**Returns:**
- `bool` - `True` if network was found and removed, `False` otherwise

**Example:**
```python
if WifiService.forget_network("OldNetwork"):
    print("Network forgotten")
else:
    print("Network not found")
```

---

#### `WifiService.get_saved_networks()`

Get list of saved network SSIDs.

**Returns:**
- `list` - List of saved SSID strings

**Example:**
```python
saved = WifiService.get_saved_networks()
print(f"Saved networks: {saved}")
```

---

#### `WifiService.get_network_password(ssid)`

Get the saved password for a network.

**Parameters:**
- `ssid` (str) - Network SSID

**Returns:**
- `str` or `None` - Password if found, `None` otherwise

**Example:**
```python
password = WifiService.get_network_password("HomeWiFi")
if password:
    print("Password found")
```

### ADC2 Compatibility

#### `WifiService.temporarily_disable(network_module=None)`

Temporarily disable WiFi for operations that require it (e.g., ESP32-S3 ADC2).

**Parameters:**
- `network_module` - Network module for dependency injection (testing)

**Returns:**
- `bool` - `True` if WiFi was connected before disabling, `False` otherwise

**Raises:**
- `RuntimeError` - If WiFi operations are already in progress

**Example:**
```python
try:
    was_connected = WifiService.temporarily_disable()
    # Do ADC2 operations here
finally:
    WifiService.temporarily_enable(was_connected)
```

---

#### `WifiService.temporarily_enable(was_connected, network_module=None)`

Re-enable WiFi after temporary disable operation.

**Parameters:**
- `was_connected` (bool) - Return value from `temporarily_disable()`
- `network_module` - Network module for dependency injection (testing)

**Behavior:**
- Clears `wifi_busy` flag
- Starts auto-connect thread if WiFi was previously connected

**Example:**
```python
WifiService.temporarily_enable(was_connected)
```

## Desktop Mode

On desktop (Linux/macOS), WifiService provides simulated behavior for testing:

| Method | Desktop Behavior |
|--------|------------------|
| `is_connected()` | Always returns `True` |
| `scan_networks()` | Returns mock SSIDs: "Home WiFi", "Pretty Fly for a Wi Fi", etc. |
| `attempt_connecting()` | Simulates 2-second connection delay, always succeeds |
| `get_current_ssid()` | Returns simulated connected SSID |
| `disconnect()` | Prints message, no-op |

This allows WiFi-related apps to be tested on desktop without actual WiFi hardware.

## Credential Storage

Network credentials are stored using SharedPreferences:

- **Preference name:** `com.micropythonos.system.wifiservice`
- **Key:** `access_points`
- **Format:** Dictionary `{ssid: {password: "...", hidden: bool}}`

**Storage location:** `data/com.micropythonos.system.wifiservice/prefs.json`

```json
{
  "access_points": {
    "HomeWiFi": {"password": "mypassword"},
    "HiddenNetwork": {"password": "secret", "hidden": true}
  }
}
```

## Troubleshooting

### WiFi Won't Connect

**Symptom:** `attempt_connecting()` returns `False`

**Possible causes:**
1. Wrong password
2. Network out of range
3. WiFi hardware issue

**Solution:**
```python
# Check if WiFi is busy
if WifiService.is_busy():
    print("WiFi is busy with another operation")
    return

# Try connecting with verbose output
success = WifiService.attempt_connecting("SSID", "password")
# Check console for "WifiService:" messages
```

### Scan Returns Empty List

**Symptom:** `scan_networks()` returns `[]`

**Possible causes:**
1. WiFi is busy (connecting, scanning)
2. No networks in range
3. WiFi hardware not initialized

**Solution:**
```python
# Check if busy first
if WifiService.is_busy():
    print("WiFi is busy, try again later")
else:
    networks = WifiService.scan_networks()
    if not networks:
        print("No networks found in range")
```

### Auto-Connect Not Working

**Symptom:** Device doesn't connect to saved networks on boot

**Possible causes:**
1. No networks saved
2. Saved networks not in range
3. WiFi busy flag stuck

**Solution:**
```python
# Check saved networks
saved = WifiService.get_saved_networks()
print(f"Saved networks: {saved}")

# Check busy state
print(f"WiFi busy: {WifiService.is_busy()}")

# Manually trigger auto-connect
import _thread
_thread.start_new_thread(WifiService.auto_connect, ())
```

### ADC2 RuntimeError

**Symptom:** `RuntimeError: Cannot disable WiFi: WifiService is already busy`

**Cause:** Trying to disable WiFi while scanning or connecting

**Solution:**
```python
# Wait for WiFi to be available
import time
while WifiService.is_busy():
    time.sleep(0.1)

# Now safe to disable
was_connected = WifiService.temporarily_disable()
```

### Hidden Network Not Connecting

**Symptom:** Hidden network saved but never connects

**Cause:** Network not marked as hidden when saved

**Solution:**
```python
# Save with hidden=True
WifiService.save_network("HiddenSSID", "password", hidden=True)
```

## Implementation Details

**Location:** `MicroPythonOS/internal_filesystem/lib/mpos/net/wifi_service.py`

**Pattern:** Static class with class-level state

**Key features:**
- `wifi_busy` - Class-level lock for concurrent access
- `access_points` - Cached dictionary of saved networks
- `_desktop_connected_ssid` - Simulated SSID for desktop mode

**Dependencies:**
- `network` module (MicroPython, not available on desktop)
- `mpos.config.SharedPreferences` - Credential storage
- `mpos.time` - NTP time sync after connection

## See Also

- [ConnectivityManager](connectivity-manager.md) - Network connectivity monitoring
- [DownloadManager](download-manager.md) - HTTP downloads
- [SharedPreferences](preferences.md) - Persistent storage
