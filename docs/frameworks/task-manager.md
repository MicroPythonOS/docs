# TaskManager

MicroPythonOS provides a centralized task management service called **TaskManager** that wraps MicroPython's `uasyncio` for managing asynchronous operations. It enables apps to run background tasks, schedule delayed operations, and coordinate concurrent activities.

## Overview

TaskManager provides:

✅ **Simplified async interface** - Easy wrappers around uasyncio primitives
✅ **Task creation** - Launch background coroutines without boilerplate
✅ **Sleep operations** - Async delays in seconds or milliseconds
✅ **Timeouts** - Wait for operations with automatic timeout handling
✅ **Event notifications** - Simple async event signaling
✅ **Centralized management** - Single point of control for all async operations

## Quick Start

### Creating Background Tasks

```python
from mpos.app.activity import Activity
from mpos import TaskManager

class MyActivity(Activity):
    def onCreate(self):
        # Launch a background task
        TaskManager.create_task(self.download_data())

    async def download_data(self):
        print("Starting download...")
        # Simulate network operation
        await TaskManager.sleep(2)
        print("Download complete!")
```

**Key points:**
- Use `TaskManager.create_task()` to launch coroutines
- Tasks run concurrently with UI operations
- Tasks continue until completion or app termination

### Delayed Operations

```python
async def delayed_operation(self):
    # Wait 3 seconds
    await TaskManager.sleep(3)
    print("3 seconds later...")

    # Wait 500 milliseconds
    await TaskManager.sleep_ms(500)
    print("Half a second later...")
```

**Available sleep methods:**
- `sleep(seconds)` - Sleep for specified seconds (float)
- `sleep_ms(milliseconds)` - Sleep for specified milliseconds (int)

### Timeout Operations

```python
from mpos import TaskManager, DownloadManager

async def download_with_timeout(self):
    try:
        # Wait max 10 seconds for download
        data = await TaskManager.wait_for(
            DownloadManager.download_url("https://example.com/data.json"),
            timeout=10
        )
        print(f"Downloaded {len(data)} bytes")
    except Exception as e:
        print(f"Download timed out or failed: {e}")
```

**Timeout behavior:**
- If operation completes within timeout: returns result
- If operation exceeds timeout: raises `asyncio.TimeoutError`
- Timeout is in seconds (float)

### Event Notifications

```python
async def wait_for_event(self):
    # Create an event
    event = await TaskManager.notify_event()

    # Wait for the event to be signaled
    await event.wait()
    print("Event occurred!")
```

## Common Patterns

### Downloading Data

```python
from mpos import TaskManager, DownloadManager

class AppStoreActivity(Activity):
    def onResume(self, screen):
        super().onResume(screen)
        # Download app index in background
        TaskManager.create_task(self.download_app_index())

    async def download_app_index(self):
        try:
            data = await DownloadManager.download_url(
                "https://apps.micropythonos.com/app_index.json"
            )
            if data:
                parsed = json.loads(data)
                self.update_ui(parsed)
        except Exception as e:
            print(f"Download failed: {e}")
```

### Periodic Tasks

```python
async def monitor_sensor(self):
    """Check sensor every second"""
    while self.monitoring:
        value = sensor_manager.read_accelerometer()
        self.update_display(value)
        await TaskManager.sleep(1)

def onCreate(self):
    self.monitoring = True
    TaskManager.create_task(self.monitor_sensor())

def onDestroy(self, screen):
    self.monitoring = False  # Stop monitoring loop
```

### Download with Progress

```python
async def download_large_file(self):
    progress_label = lv.label(self.screen)

    async def update_progress(percent):
        progress_label.set_text(f"Downloading: {percent}%")

    success = await DownloadManager.download_url(
        "https://example.com/large.bin",
        outfile="/sdcard/large.bin",
        progress_callback=update_progress
    )

    if success:
        progress_label.set_text("Download complete!")
```

### Concurrent Downloads

```python
async def download_multiple_icons(self, apps):
    """Download multiple icons concurrently"""
    tasks = []
    for app in apps:
        task = DownloadManager.download_url(app.icon_url)
        tasks.append(task)

    # Wait for all downloads (with timeout per icon)
    results = []
    for i, task in enumerate(tasks):
        try:
            data = await TaskManager.wait_for(task, timeout=5)
            results.append(data)
        except Exception as e:
            print(f"Icon {i} failed: {e}")
            results.append(None)

    return results
```

## API Reference

### Task Creation

#### `TaskManager.create_task(coroutine)`

Create and schedule a background task.

**Parameters:**
- `coroutine` - Coroutine object to execute (must be async def)

**Returns:**
- Task object (usually not needed)

**Example:**
```python
TaskManager.create_task(self.background_work())
```

### Sleep Operations

#### `TaskManager.sleep(seconds)`

Async sleep for specified seconds.

**Parameters:**
- `seconds` (float) - Time to sleep in seconds

**Returns:**
- None (awaitable)

**Example:**
```python
await TaskManager.sleep(2.5)  # Sleep 2.5 seconds
```

#### `TaskManager.sleep_ms(milliseconds)`

Async sleep for specified milliseconds.

**Parameters:**
- `milliseconds` (int) - Time to sleep in milliseconds

**Returns:**
- None (awaitable)

**Example:**
```python
await TaskManager.sleep_ms(500)  # Sleep 500ms
```

### Timeout Operations

#### `TaskManager.wait_for(awaitable, timeout)`

Wait for an operation with timeout.

**Parameters:**
- `awaitable` - Coroutine or awaitable object
- `timeout` (float) - Maximum time to wait in seconds

**Returns:**
- Result of the awaitable if completed in time

**Raises:**
- `asyncio.TimeoutError` - If operation exceeds timeout

**Example:**
```python
try:
    result = await TaskManager.wait_for(
        download_operation(),
        timeout=10
    )
except asyncio.TimeoutError:
    print("Operation timed out")
```

### Event Notifications

#### `TaskManager.notify_event()`

Create an async event for coordination.

**Returns:**
- asyncio.Event object

**Example:**
```python
event = await TaskManager.notify_event()
await event.wait()  # Wait for event
event.set()  # Signal event
```

## Integration with UI

### Safe UI Updates from Async Tasks

LVGL operations must run on the main thread. Use UI operations sparingly from async tasks:

```python
async def background_task(self):
    # Do async work
    data = await fetch_data()

    # Update UI (safe - LVGL thread-safe)
    self.label.set_text(f"Got {len(data)} items")
```

For complex UI updates, consider using callbacks or state variables:

```python
async def download_task(self):
    self.download_complete = False
    data = await DownloadManager.download_url(url)
    self.data = data
    self.download_complete = True

def onCreate(self):
    TaskManager.create_task(self.download_task())
    # Check download_complete flag periodically
```

### Activity Lifecycle

Tasks continue running even when activity is paused. Clean up tasks in `onDestroy()`:

```python
def onCreate(self):
    self.running = True
    TaskManager.create_task(self.monitor_loop())

async def monitor_loop(self):
    while self.running:
        await self.check_status()
        await TaskManager.sleep(1)

def onDestroy(self, screen):
    self.running = False  # Stop task loop
```

## Common Use Cases

### Network Operations

```python
# Download JSON data
data = await DownloadManager.download_url("https://api.example.com/data")
parsed = json.loads(data)

# Download to file
success = await DownloadManager.download_url(
    "https://example.com/file.bin",
    outfile="/sdcard/file.bin"
)

# Stream processing
async def process_chunk(chunk):
    # Process each chunk as it arrives
    pass

await DownloadManager.download_url(
    "https://example.com/stream",
    chunk_callback=process_chunk
)
```

### WebSocket Communication

```python
from mpos import TaskManager
import websocket

async def websocket_listener(self):
    ws = websocket.WebSocketApp(url, callbacks)
    await ws.run_forever()

def onCreate(self):
    TaskManager.create_task(self.websocket_listener())
```

### Sensor Polling

```python
import mpos.sensor_manager as SensorManager
from mpos import TaskManager

async def poll_sensors(self):
    while self.active:
        accel = SensorManager.read_accelerometer()
        gyro = SensorManager.read_gyroscope()

        self.update_display(accel, gyro)
        await TaskManager.sleep(0.1)  # 100ms = 10 Hz
```

## Performance Considerations

### Task Overhead

- **Minimal overhead**: TaskManager is a thin wrapper around uasyncio
- **Concurrent tasks**: Dozens of tasks can run simultaneously
- **Memory**: Each task uses ~1-2KB of RAM
- **Context switches**: Tasks yield automatically during await

### Best Practices

1. **Use async for I/O**: Network, file operations, sleep
2. **Avoid blocking**: Don't use time.sleep() in async functions
3. **Clean up tasks**: Set flags to stop loops in onDestroy()
4. **Handle exceptions**: Always wrap operations in try/except
5. **Limit concurrency**: Don't create unbounded task lists

### Example: Bounded Concurrency

```python
async def download_with_limit(self, urls, max_concurrent=5):
    """Download URLs with max concurrent downloads"""
    results = []
    for i in range(0, len(urls), max_concurrent):
        batch = urls[i:i+max_concurrent]
        batch_results = []

        for url in batch:
            task = DownloadManager.download_url(url)
            try:
                data = await TaskManager.wait_for(task, timeout=10)
                batch_results.append(data)
            except Exception as e:
                batch_results.append(None)

        results.extend(batch_results)

    return results
```

## Troubleshooting

### Task Never Completes

**Problem:** Task hangs indefinitely

**Solution:** Add timeout:
```python
try:
    result = await TaskManager.wait_for(task, timeout=30)
except asyncio.TimeoutError:
    print("Task timed out")
```

### Memory Leak

**Problem:** Memory usage grows over time

**Solution:** Ensure task loops exit:
```python
async def loop_task(self):
    while self.running:  # Check flag
        await work()
        await TaskManager.sleep(1)

def onDestroy(self, screen):
    self.running = False  # Stop loop
```

### UI Not Updating

**Problem:** UI doesn't reflect async task results

**Solution:** Ensure UI updates happen on main thread (LVGL is thread-safe in MicroPythonOS):
```python
async def task(self):
    data = await fetch()
    # Direct UI update is safe
    self.label.set_text(str(data))
```

### Exception Not Caught

**Problem:** Exception crashes app

**Solution:** Wrap task in try/except:
```python
TaskManager.create_task(self.safe_task())

async def safe_task(self):
    try:
        await risky_operation()
    except Exception as e:
        print(f"Task failed: {e}")
```

## Implementation Details

**Location**: `/home/user/MicroPythonOS/internal_filesystem/lib/mpos/task_manager.py`

**Pattern**: Wrapper around `uasyncio` module

**Key features:**
- `create_task()` - Wraps `asyncio.create_task()`
- `sleep()` / `sleep_ms()` - Wrap `asyncio.sleep()`
- `wait_for()` - Wraps `asyncio.wait_for()` with timeout handling
- `notify_event()` - Creates `asyncio.Event()` objects

**Thread model:**
- All async tasks run on main asyncio event loop
- No separate threads created (unless using `_thread` module separately)
- Tasks cooperatively multitask via await points

## See Also

- [DownloadManager](download-manager.md) - HTTP download utilities
- [AudioFlinger](audioflinger.md) - Audio playback (uses TaskManager internally)
- [SensorManager](sensor-manager.md) - Sensor data access
