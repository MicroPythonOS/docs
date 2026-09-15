# USBManager

USBManager is a singleton framework that provides a unified API for USB host support on ESP32: DisplayLink display adapters plus boot-protocol HID mice and keyboards. It owns the host lifecycle, display/panel switching, HID arming, and the 1-second poll timer that drives the whole subsystem.

## Overview

USB host support is opt-in and ESP32-S3-only (needs USB OTG):

```
./scripts/build_mpos.sh esp32s3 --usb
```

This compiles in the `usb` C module (`c_mpos/usb/`) alongside TinyUSB device mode. **CDC is the default**: the device boots with a USB-serial console and the OTG peripheral stays in device mode. Host mode starts only on explicit request — Settings → "USB Host Mode", or `USBManager.activate()` — and while it is active USB-CDC is gone (console remains over UART REPL where exposed, or WebREPL over WiFi). Deactivating brings CDC back; the choice persists across reboots. Holding BOOT at boot forces CDC regardless of the persisted flag.

USBManager centralizes all USB-host operations in a single class with class methods:

- **Unified API** - Single class for display + HID management
- **Clean Namespace** - No scattered functions cluttering imports
- **Testable** - USBManager can be tested independently with a fake `usb` module
- **Hotplug** - The poll timer auto-switches to a ready display and auto-reverts on unplug
- **HID** - Mice and keyboards arm together, enabled per-kind from live device state

```python
from mpos import USBManager

if USBManager.is_available():
    USBManager.arm_display()
    USBManager.arm_hid()
```

## Architecture

USBManager is implemented as a singleton using class variables and class methods. No instance creation is needed:

```python
class USBManager:
    _usb_dev = None       # usb.Display handle (the adapter)
    _usb_mouse = None     # USBMouse indev, enabled only while a mouse streams
    _usb_keyboard = None  # USBHIDKeyboard indev, enabled only while a keyboard streams
    _hid_hub = None       # HIDHub demux shared by both indevs

    @classmethod
    def arm_display(cls, width=640, height=480):
        ...
```

The layers, bottom to top:

- **C module `usb`** (`c_mpos/usb/`): `Display` class (start/poll/ready, frame upload) plus module-level host functions (`bus_devices`, `lsusb`, `hub_ports`, `reset_port`, watchdog toggles) and HID transport (`hid_start/poll/drain/state/...`). Upstream Pico_USB_Disp is vendored under `c_mpos/usb/upstream/` with a few clearly-commented MPOS adaptations (hub-port watchdog, `lsusb`, held-handle hook).
- **Drivers**: `drivers/display/usb_display.py` (`USBDisplayDriver`, a `DisplayDriver` behind a shim bus) and `drivers/indev/usb_hid.py` (parser registry + `HIDHub` demux + `USBMouse` + `USBHIDKeyboard`). Report parsing is Python-side, so new device kinds never need C changes.
- **USBManager** (`mpos/usb/`): boot arming (`arm_display`, `arm_hid`), the 1 s LVGL poll timer (HID pump + display state machine + auto-switch), panel/USB swapping, touch remapping, and HID/watchdog coexistence (port-exact idle-reset skip for claimed HIDs, parked-only global suppression).

Only DisplayLink DL-1xx adapters work on ESP32-S3 (Full-Speed OTG); DL-165/DL-195 recommended. T6/MS91xx need High-Speed, i.e. ESP32-P4 only (protocol code is vendored but untested on hardware).

### HCD channel budget

The S3 DWC_OTG core has 8 host channels (~7 usable): one channel per USB *pipe*, held for the pipe's lifetime. Rule of thumb: **max 1 hub + 2 downstream devices** on S3. ESP32-P4/S31 have 16 channels, so **1 hub + 4 devices** fits there. Claim priority is display > mouse > keyboard; a claim that fails with channel exhaustion parks silently with backoff until a topology change or `usb.hid_retry()`.

### HID/watchdog coexistence

A healthy, enumerated HID reads exactly like a wedged adapter (connected + enabled, no bus growth), which the hub watchdog's idle auto-reset would otherwise `PORT_RESET` ~15s after plug. Two-part answer: open HID handles resolve to their (hub, port), so the sweep skips the auto-reset exactly on HID-owned ports and logs `HID device, auto-reset skipped` inline, while other ports keep healing; parked devices have no open handle and stay covered by a global suppression (restored when the park clears). The disabled-port recovery path is never suppressed.

### Display switching

When host mode is active, `mpos.main` arms the display at boot without waiting; the poll timer switches the UI over automatically when a monitor becomes ready and reverts on disconnect. The swap suspends the LVGL pump, tears down activities, repoints indevs to the new display (panel touch is remapped through the panel's own proven mapping — no per-board tables), moves the topmenu, and restarts the launcher. Resolution floor is 640x480: smaller modes need a sub-25 MHz pixel clock that real monitors cannot sync to.

## Usage

### Host mode activation

```python
from mpos import USBManager

USBManager.activate()    # leave CDC device mode, start host, arm display+HID
USBManager.deactivate()  # stop host, bring CDC back (REPL rejoins automatically)
USBManager.host_mode_active()  # live host state
```

Both persist the choice (`host_mode` in the `com.micropythonos.usb`
preferences) so reboots keep it; pass `persist=False` for a one-shot switch.
`mpos.main` honors the persisted flag at boot. Both are idempotent and safe
to retry. On stock builds (no `usb` module) they return `False`.

### Checking availability and arming

```python
from mpos import USBManager

if USBManager.is_available():
    dev = USBManager.arm_display()  # construct + start host + poll timer
    USBManager.arm_hid()            # HID client + mouse/keyboard indevs
```

Both are idempotent and safe to retry. On stock builds (no `usb` module) they return `None`. In host-mode builds these run automatically: `activate()` calls them, and boot does too when host mode persisted.

### Manual display switching

```python
from mpos import USBManager

USBManager.switch_to_usb(timeout_s=5)  # init adapter, swap UI over
USBManager.switch_to_panel()           # swap back, free USB buffers
```

### REPL inspection (on `--usb` builds)

```python
import usb

print(usb.lsusb())       # Linux-style bus listing with VID:PID + product strings
usb.bus_devices()        # [addr, ...] currently sensed
usb.hub_ports()          # [(hub_addr, port, connected, enabled, high_speed), ...]
usb.reset_port(2, 3)     # re-enumerate one hub port (never a high-speed uplink)
usb.hid_state()          # [(addr, kind, vid, pid, speed), ...] streaming HIDs
usb.hid_parked()         # [(vid, pid, kind, fails), ...] parked (fails=255) or cooling down
usb.hid_retry()          # clear parked/cooldown and rescan now
usb.hid_poll_stats()     # [(addr, kind, polls, ch_fails), ...] sample twice, diff polls
usb.hid_loop_lag()       # ms since the HID task pumped events (~100 healthy)
```

## API Reference

### Class Methods

- `is_available()` - `True` when the `usb` C module is importable (`--usb` build).
- `activate(persist=True)` - Leave CDC device mode, start the host stack, arm display + HID. Returns `False` on stock builds or failure.
- `deactivate(persist=True)` - Stop host (UI back to panel, indevs removed), bring CDC back. Returns `False` on stock builds or failure.
- `host_mode_active()` - Live host state (`True` between successful activate/deactivate).
- `host_boot_requested()` - Persisted flag honored at boot (`False` when BOOT held).
- `arm_display(width=640, height=480)` - Construct (once) and start the adapter handle, ensure the poll timer. Returns the `usb.Display` or `None`.
- `arm_hid()` - Start the HID client, create the shared `HIDHub` plus `USBMouse`/`USBHIDKeyboard` indevs (disabled until their kind streams). Returns the mouse or `None`.
- `switch_to_usb(width=640, height=480, timeout_s=0)` - Blocking-init the adapter and swap the UI to it.
- `switch_to_panel()` - Swap back to the panel display.
- `try_init_usb_display(width=640, height=480, timeout_s=10, buf_lines=16)` - Wait for READY and build the `USBDisplayDriver`. Raises `RuntimeError` on timeout.

### The `usb` C module

- `usb.Display(port=0, width=0, height=0, ignore_edid=False)` - Adapter handle. `start()`, `poll()` (True on READY/disconnect/mode change), `ready()`, `width()`, `height()`, `chip_name()`, `update_565(x, y, w, h, buf)`, `fill(x, y, w, h, color)`, `flush(timeout_ms=100)`, `set_mode(w, h)`, `force_reenum()` (root-port power cycle).
- Host inspection: `bus_devices()`, `lsusb()`, `hub_ports()`, `reset_port(hub_addr, port[, power_cycle[, force]])`, `set_watchdog(on)`, `auto_reset_idle([on])` (bare call reads back), `set_log(on)`.
- HID transport: `hid_start()`, `hid_poll()`, `hid_drain()`, `hid_state()`, `hid_claimed_addrs()`, `hid_parked()`, `hid_retry()`, `hid_poll_stats()`, `hid_loop_lag()`, `hid_verbose([on])`, `hid_set_kbd_transient([on])` (keyboards stay persistent by default; the toggle is a live A/B switch).
- Mode switching: `activate_host()`, `deactivate_host()`, `host_active()`.

## Limitations

- ESP32-S3 only; DisplayLink DL-1xx only (tested on DL-165/DL-195).
- Activating host mode kills USB-CDC until deactivated (announce it before
  toggling on no-UART boards); UART REPL where exposed, WebREPL over WiFi,
  and holding BOOT at boot (forces CDC) are the ways back.
- 640x480 minimum mode; rotation fixed to `_0`; RGB565 only.
- Full display + mouse + keyboard combo cannot fit the S3 channel budget persistently (the keyboard parks until something unplugs); transient keyboard polling (`usb.hid_set_kbd_transient(True)`) holds no persistent pipe and is the experimental fit-the-combo option.
- A wedged adapter behind an externally powered hub is not recoverable by port resets — power-cut the adapter itself.
- Field upgrades across this rename need `--erase-all` or a lib re-sync: the frozen C module changed name (`usb_disp` to `usb`), so stale flash shadows will not fall back.
