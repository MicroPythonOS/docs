# SDCardManager

SDCardManager provides a unified API for SD card initialization, mounting, and management in both SPI and SDIO modes. It follows the singleton + class-method-delegation pattern used by all MPOS frameworks.

## Overview

SDCardManager centralizes all SD card operations in a single class:

- **Dual-Mode Support** - Auto-detects SPI or SDIO based on provided pins
- **Unified API** - Single `from mpos import SDCardManager` for all SD card needs
- **Safe Mounting** - Idempotent mount (treats already-mounted as success)
- **Format Support** - Format uninitialized cards as FAT32
- **Singleton Pattern** - Class methods delegate to a single instance initialized at boot

## Architecture

SDCardManager is initialized once at boot by the board definition, then accessed via class methods:

```python
class SDCardManager:
    _instance = None

    @classmethod
    def init(cls, mode=None, spi_bus=None, cs_pin=None, cmd_pin=None, clk_pin=None,
             d0_pin=None, d1_pin=None, d2_pin=None, d3_pin=None, slot=1, width=None, freq=20000000):
        ...

    @classmethod
    def mount(cls, format=False):
        if not cls._instance:
            return False
        ...
```

The board definition calls `SDCardManager.init(...)` at boot with the hardware-specific pin configuration. Apps then call `mount()`, `is_mounted()`, `get_mount_point()`, etc. without needing to know about the underlying hardware.

## Usage

### Basic Mount

```python
from mpos import SDCardManager

SDCardManager.mount()
mountpoint = SDCardManager.get_mount_point()
if mountpoint:
    import os
    os.listdir(mountpoint)
```

### Checking Mount Status

```python
from mpos import SDCardManager

if SDCardManager.is_mounted():
    mp = SDCardManager.get_mount_point()
    mode = SDCardManager.get_mode()
    print("SD card mounted at %s (%s mode)" % (mp, mode))
```

### Format and Mount

```python
from mpos import SDCardManager

SDCardManager.mount(format=True)
```

### Listing Contents

```python
from mpos import SDCardManager

if SDCardManager.is_mounted():
    contents = SDCardManager.get_raw().list(SDCardManager.get_mount_point())
    for item in contents:
        print(item)
```

## API Reference

### Class Methods

#### `init(mode=None, spi_bus=None, cs_pin=None, cmd_pin=None, clk_pin=None, d0_pin=None, d1_pin=None, d2_pin=None, d3_pin=None, slot=1, width=None, freq=20000000)`

Initialize the SD card hardware. Called once at boot by the board definition. Mode is auto-detected from which pins are provided (SDIO pins → SDIO mode, SPI pins → SPI mode). Can be overridden with the `mode` parameter.

**Parameters:**
- `mode` (str, optional): `'spi'` or `'sdio'`. Auto-detected if not provided.
- `spi_bus` (machine.SPI, optional): SPI bus instance for SPI mode.
- `cs_pin` (int, optional): Chip-select pin number for SPI mode.
- `cmd_pin` (int, optional): CMD pin for SDIO mode.
- `clk_pin` (int, optional): CLK pin for SDIO mode.
- `d0_pin` (int, optional): Data 0 pin for SDIO mode (required).
- `d1_pin` (int, optional): Data 1 pin for 4-bit SDIO mode.
- `d2_pin` (int, optional): Data 2 pin for 4-bit SDIO mode.
- `d3_pin` (int, optional): Data 3 pin for 4-bit SDIO mode.
- `slot` (int): SDIO slot number (0 or 1). Default: `1`.
- `width` (int, optional): SDIO bus width (`1` or `4`). Auto-detected from data pins if not provided.
- `freq` (int): SDIO clock frequency in Hz. Default: `20000000`.

**Example:**
```python
import machine
from mpos import SDCardManager

# SPI mode
spi = machine.SPI(2, baudrate=20000000, sck=machine.Pin(40), mosi=machine.Pin(41), miso=machine.Pin(38))
SDCardManager.init(spi_bus=spi, cs_pin=21)

# SDIO 1-bit mode
SDCardManager.init(cmd_pin=39, clk_pin=40, d0_pin=38, width=1)
```

---

#### `mount(format=False)`

Mount the SD card at `/sdcard`. Returns `True` on success, `False` on failure. Treats already-mounted as success. If `format=True` and the mount fails, the card is formatted as FAT32 and re-mounted.

**Parameters:**
- `format` (bool): Whether to format the card if mount fails. Default: `False`.

**Returns:** bool - `True` if mounted, `False` otherwise.

**Example:**
```python
SDCardManager.mount()
SDCardManager.mount(format=True)
```

---

#### `is_mounted()`

Check whether the SD card is currently mounted at `/sdcard`. Verifies by listing the root directory and testing with a temporary directory.

**Returns:** bool - `True` if mounted, `False` otherwise.

**Example:**
```python
if SDCardManager.is_mounted():
    print("SD card is ready")
```

---

#### `get_mount_point()`

Get the mount point path if the card is mounted, or `None` if not.

**Returns:** str or None - `"/sdcard"` if mounted, `None` otherwise.

**Example:**
```python
mp = SDCardManager.get_mount_point()
if mp:
    print("Mounted at %s" % mp)
```

---

#### `get_mode()`

Get the current mode (`'spi'` or `'sdio'`), or `None` if not initialized.

**Returns:** str or None

**Example:**
```python
mode = SDCardManager.get_mode()
print("SD card mode: %s" % mode)
```

---

#### `format()`

Format the SD card as FAT32. Unmounts first if already mounted.

**Returns:** bool - `True` on success, `False` on failure.

**Example:**
```python
SDCardManager.format()
```

---

#### `get_raw()`

Get the underlying `SDCardManager` instance. Useful for calling instance methods like `list()`.

**Returns:** SDCardManager instance or None

**Example:**
```python
instance = SDCardManager.get_raw()
if instance:
    files = instance.list(SDCardManager.get_mount_point())
```

## Practical Examples

### App Using SD Card for File Storage

Real-world example from the Retro-Go launcher app:

```python
from mpos import SDCardManager

class RetroLauncher(Activity):
    def onResume(self, screen):
        self.bootfile_prefix = ""
        SDCardManager.mount()
        prefix = SDCardManager.get_mount_point()
        if prefix:
            self.bootfile_prefix = prefix + "/"
```

([View on GitHub](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/apps/com.micropythonos.doom_launcher/retrogo_launcher.py#L122))

### SPI Board Configuration

```python
import machine
from mpos import SDCardManager

spi = machine.SPI(2, baudrate=20000000, sck=machine.Pin(40), mosi=machine.Pin(41), miso=machine.Pin(38))
SDCardManager.init(spi_bus=spi, cs_pin=21)
```

### SDIO 4-Bit Board Configuration

```python
from mpos import SDCardManager

SDCardManager.init(
    cmd_pin=39, clk_pin=40,
    d0_pin=38, d1_pin=37, d2_pin=36, d3_pin=35,
    width=4, freq=20000000
)
```

## Implementation Details

### File Structure

```
mpos/
├── sdcard.py          # SDCardManager implementation
└── __init__.py        # Exports SDCardManager
```

### Initialization Flow

1. **Boot** - Board definition calls `SDCardManager.init(...)` with hardware-specific pins
2. **On Demand** - Apps call `SDCardManager.mount()` when they need SD access
3. **Safe** - Mount is idempotent; already-mounted returns `True`
4. **Fallback** - If `format=True`, failed mount triggers format-and-retry

## Design Patterns

### Singleton with Class Method Delegation

```python
class SDCardManager:
    _instance = None  # Set once at boot

    @classmethod
    def mount(cls, format=False):
        if not cls._instance:
            return False
        return cls._instance._try_mount(_MOUNT_POINT)
```

All public methods are class methods that forward to the single instance. Apps never create instances directly.

## Related Frameworks

- **[FileExplorerActivity](file-explorer-activity.md)** - File browser that uses SDCardManager for SD card access
- **[BuildInfo](build-info.md)** - Build and board configuration metadata

## See Also

- [Architecture Overview](../architecture/overview.md)
- [Frameworks](../architecture/frameworks.md)
- [Creating Apps](../apps/creating-apps.md)
