# DownloadManager

MicroPythonOS provides a centralized HTTP download service called **DownloadManager** that handles async file downloads with support for progress tracking, streaming, and automatic session management.

## Overview

DownloadManager provides:

- **Three output modes** - Download to memory, file, or stream with callbacks
- **Automatic session management** - Shared aiohttp session with connection reuse
- **Thread-safe** - Safe for concurrent downloads across apps
- **Progress tracking** - Real-time download progress callbacks
- **Retry logic** - Automatic retry on chunk failures (3 attempts)
- **Resume support** - HTTP Range headers for partial downloads
- **Memory efficient** - Chunked downloads (1KB chunks)

## Quick Start

### Download to Memory

```python
from mpos import TaskManager, DownloadManager

class MyActivity(Activity):
    def onCreate(self):
        TaskManager.create_task(self.fetch_data())

    async def fetch_data(self):
        # Download JSON data
        data = await DownloadManager.download_url(
            "https://api.example.com/data.json"
        )

        if data:
            import json
            parsed = json.loads(data)
            print(f"Got {len(parsed)} items")
        else:
            print("Download failed")
```

**Returns:**
- `bytes` - Downloaded content on success
- `None` - On failure (network error, HTTP error, timeout)

### Download to File

```python
async def download_app(self, url):
    # Download .mpk file
    success = await DownloadManager.download_url(
        "https://apps.micropythonos.com/app.mpk",
        outfile="/sdcard/app.mpk"
    )

    if success:
        print("Download complete!")
    else:
        print("Download failed")
```

**Returns:**
- `True` - File downloaded successfully
- `False` - Download failed

### Download with Progress

```python
async def download_with_progress(self):
    progress_bar = lv.bar(self.screen)
    progress_bar.set_range(0, 100)

    async def update_progress(percent):
        progress_bar.set_value(percent, lv.ANIM.ON)
        print(f"Downloaded: {percent}%")

    success = await DownloadManager.download_url(
        "https://example.com/large_file.bin",
        outfile="/sdcard/large_file.bin",
        progress_callback=update_progress
    )
```

**Progress callback:**
- Called with percentage (0-100) as integer
- Must be an async function
- Called after each chunk is downloaded

### Streaming with Callbacks

```python
async def stream_download(self):
    processed_bytes = 0

    async def process_chunk(chunk):
        nonlocal processed_bytes
        # Process each chunk as it arrives
        processed_bytes += len(chunk)
        print(f"Processed {processed_bytes} bytes")
        # Could write to custom location, parse, etc.

    success = await DownloadManager.download_url(
        "https://example.com/stream",
        chunk_callback=process_chunk
    )
```

**Chunk callback:**
- Called for each 1KB chunk received
- Must be an async function
- Cannot be used with `outfile` parameter

## API Reference

### `DownloadManager.download_url()`

Download a URL with flexible output modes.

```python
async def download_url(url, outfile=None, total_size=None,
                      progress_callback=None, chunk_callback=None,
                      headers=None)
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | str | URL to download (required) |
| `outfile` | str | Path to write file (optional) |
| `total_size` | int | Expected size in bytes for progress tracking (optional) |
| `progress_callback` | async function | Callback for progress updates (optional) |
| `chunk_callback` | async function | Callback for streaming chunks (optional) |
| `headers` | dict | Custom HTTP headers (optional) |

**Returns:**

- **Memory mode** (no `outfile` or `chunk_callback`): `bytes` on success, `None` on failure
- **File mode** (`outfile` provided): `True` on success, `False` on failure
- **Stream mode** (`chunk_callback` provided): `True` on success, `False` on failure

**Raises:**

- `ValueError` - If both `outfile` and `chunk_callback` are provided

**Example:**
```python
# Memory mode
data = await DownloadManager.download_url("https://example.com/data.json")

# File mode
success = await DownloadManager.download_url(
    "https://example.com/file.bin",
    outfile="/sdcard/file.bin"
)

# Stream mode
async def process(chunk):
    print(f"Got {len(chunk)} bytes")

success = await DownloadManager.download_url(
    "https://example.com/stream",
    chunk_callback=process
)
```

### Utility Functions

#### `DownloadManager.is_network_error(exception)`

Check if an exception is a recoverable network error.

Recognizes common network error codes and messages that indicate temporary connectivity issues that can be retried.

**Parameters:**
- `exception` - Exception to check

**Returns:**
- `bool` - True if this is a network error that can be retried

**Example:**
```python
try:
    await DownloadManager.download_url(url)
except Exception as e:
    if DownloadManager.is_network_error(e):
        # Network error - wait and retry
        await TaskManager.sleep(2)
        # Retry with resume...
    else:
        # Fatal error - show error to user
        raise
```

**Detected error codes:**
- `-110` - ETIMEDOUT (connection timed out)
- `-113` - ECONNABORTED (connection aborted)
- `-104` - ECONNRESET (connection reset by peer)
- `-118` - EHOSTUNREACH (no route to host)
- `-202` - DNS/connection error (network not ready)

**Detected error messages:**
- "connection reset", "connection aborted"
- "broken pipe", "network unreachable", "host unreachable"
- "failed to download chunk"

#### `DownloadManager.get_resume_position(outfile)`

Get the current size of a partially downloaded file.

Useful for implementing resume functionality with Range headers.

**Parameters:**
- `outfile` - Path to file

**Returns:**
- `int` - File size in bytes, or 0 if file doesn't exist

**Example:**
```python
resume_from = DownloadManager.get_resume_position("/sdcard/file.bin")
if resume_from > 0:
    headers = {'Range': f'bytes={resume_from}-'}
    await DownloadManager.download_url(url, outfile=outfile, headers=headers)
else:
    # Start new download
    await DownloadManager.download_url(url, outfile=outfile)
```

### Session Management Functions

#### `DownloadManager.is_session_active()`

Check if an HTTP session is currently active.

**Returns:**
- `bool` - True if session exists

**Example:**
```python
if DownloadManager.is_session_active():
    print("Session active")
```

#### `DownloadManager.close_session()`

Explicitly close the HTTP session (rarely needed).

**Returns:**
- None (awaitable)

**Example:**
```python
await DownloadManager.close_session()
```

**Note:** Sessions are automatically managed. This is mainly for testing.

## Common Patterns

### Download with Timeout

```python
from mpos import TaskManager, DownloadManager

async def download_with_timeout(self, url, timeout=10):
    try:
        data = await TaskManager.wait_for(
            DownloadManager.download_url(url),
            timeout=timeout
        )
        return data
    except asyncio.TimeoutError:
        print(f"Download timed out after {timeout}s")
        return None
```

### Download Multiple Files Concurrently

```python
async def download_icons(self, apps):
    """Download app icons concurrently with individual timeouts"""
    for app in apps:
        if not app.icon_data:
            try:
                app.icon_data = await TaskManager.wait_for(
                    DownloadManager.download_url(app.icon_url),
                    timeout=5  # 5 seconds per icon
                )
            except Exception as e:
                print(f"Icon download failed: {e}")
                continue

            # Update UI with icon
            if app.icon_data:
                self.update_icon_display(app)
```

### Download with Explicit Size

```python
async def download_mpk(self, app):
    """Download app package with known size"""
    await DownloadManager.download_url(
        app.download_url,
        outfile=f"/sdcard/{app.fullname}.mpk",
        total_size=app.download_url_size,  # Use known size
        progress_callback=self.update_progress
    )
```

**Benefits of providing `total_size`:**
- More accurate progress percentages
- Avoids default 100KB assumption
- Better user experience

### Resume Partial Download

```python
async def resume_download(self, url, outfile):
    """Resume a partial download using Range headers"""
    # Use DownloadManager utility to get resume position
    bytes_written = DownloadManager.get_resume_position(outfile)
    
    if bytes_written > 0:
        print(f"Resuming from {bytes_written} bytes")
        headers = {'Range': f'bytes={bytes_written}-'}
    else:
        print("Starting new download")
        headers = None

    # Download remaining bytes
    success = await DownloadManager.download_url(
        url,
        outfile=outfile,
        headers=headers
    )

    return success
```

**Note:** Server must support HTTP Range requests.

### Error Handling with Network Detection

```python
async def robust_download(self, url, outfile):
    """Download with comprehensive error handling and retry"""
    max_retries = 3
    retry_delay = 2
    
    for attempt in range(max_retries):
        try:
            success = await DownloadManager.download_url(url, outfile=outfile)
            
            if success:
                return True
            else:
                print("Download failed")
                return False

        except Exception as e:
            if DownloadManager.is_network_error(e):
                # Network error - retry with resume
                print(f"Network error (attempt {attempt + 1}/{max_retries}): {e}")
                
                if attempt < max_retries - 1:
                    await TaskManager.sleep(retry_delay)
                    # Resume from last position
                    continue
                else:
                    print("Max retries reached")
                    return False
            else:
                # Non-network error - don't retry
                print(f"Fatal error: {e}")
                return False
    
    return False
```

**Benefits:**
- Automatic retry on network errors
- Resume from last position
- No retry on fatal errors (404, invalid URL, etc.)
- User-friendly error messages

## Session Management

### Automatic Lifecycle

DownloadManager automatically manages the aiohttp session:

1. **Lazy initialization**: Session created on first download
2. **Connection reuse**: HTTP keep-alive for performance
3. **Automatic cleanup**: Session cleared when idle
4. **Thread-safe**: Safe for concurrent downloads

### Session Behavior

```python
# First download creates session
data1 = await DownloadManager.download_url(url1)
# Session is now active

# Second download reuses session (faster)
data2 = await DownloadManager.download_url(url2)
# HTTP keep-alive connection reused

# After download completes, session auto-closes if idle
# (no refcount - session cleared)
```

**Performance benefits:**
- **Connection reuse**: Avoid TCP handshake overhead
- **Shared session**: One session across all apps
- **Memory efficient**: Session cleared when not in use

## Progress Tracking

### Progress Calculation

Progress is calculated based on:
1. **Content-Length header** (if provided by server)
2. **Explicit `total_size` parameter** (overrides header)
3. **Default assumption** (100KB if neither available)

```python
async def download_with_unknown_size(self):
    """Handle download without Content-Length"""
    downloaded_bytes = [0]

    async def track_progress(percent):
        # Percent may be inaccurate if size unknown
        print(f"Progress: {percent}%")

    data = await DownloadManager.download_url(
        url_without_content_length,
        progress_callback=track_progress
    )
```

### Progress Callback Signature

```python
async def progress_callback(percent: int):
    """
    Args:
        percent: Progress percentage (0-100)
    """
    # Update UI, log, etc.
    self.progress_bar.set_value(percent, lv.ANIM.ON)
```

## Retry Logic

DownloadManager automatically retries failed chunk reads:

- **Retry count**: 3 attempts per chunk
- **Timeout per attempt**: 10 seconds
- **Exponential backoff**: No (immediate retry)

```python
# Automatic retry example
while tries_left > 0:
    try:
        chunk = await response.content.read(1024)
        break  # Success
    except Exception as e:
        print(f"Chunk read error: {e}")
        tries_left -= 1

if tries_left == 0:
    # All retries failed - abort download
    return False
```

**Retry behavior:**
- Network hiccup: Automatic retry
- Permanent failure: Returns False/None after 3 attempts
- Partial download: File closed, may need cleanup

## HTTP Headers

### Custom Headers

```python
# Example: Custom user agent
await DownloadManager.download_url(
    url,
    headers={
        'User-Agent': 'MicroPythonOS/0.3.3',
        'Accept': 'application/json'
    }
)

# Example: API authentication
await DownloadManager.download_url(
    api_url,
    headers={
        'Authorization': 'Bearer YOUR_TOKEN'
    }
)
```

### Range Requests

```python
# Download specific byte range
await DownloadManager.download_url(
    url,
    headers={
        'Range': 'bytes=1000-2000'  # Download bytes 1000-2000
    }
)

# Resume from byte 5000
await DownloadManager.download_url(
    url,
    outfile=partial_file,
    headers={
        'Range': 'bytes=5000-'  # Download from byte 5000 to end
    }
)
```

## Performance Considerations

### Memory Usage

**Per download:**
- Chunk buffer: 1KB
- Progress callback overhead: ~50 bytes
- Total: ~1-2KB per concurrent download

**Shared:**
- aiohttp session: ~2KB
- Total baseline: ~3KB

### Concurrent Downloads

```python
# Good: Limited concurrency
async def download_batch(self, urls):
    max_concurrent = 5
    for i in range(0, len(urls), max_concurrent):
        batch = urls[i:i+max_concurrent]
        results = []
        for url in batch:
            try:
                data = await TaskManager.wait_for(
                    DownloadManager.download_url(url),
                    timeout=10
                )
                results.append(data)
            except Exception as e:
                results.append(None)
        # Process batch results
```

**Guidelines:**
- **Limit concurrent downloads**: 5-10 max recommended
- **Use timeouts**: Prevent stuck downloads
- **Handle failures**: Don't crash on individual failures
- **Monitor memory**: Check free RAM on device

### Chunk Size

Fixed at 1KB for balance between:
- **Memory**: Small chunks use less RAM
- **Performance**: Larger chunks reduce overhead
- **Responsiveness**: Small chunks = frequent progress updates

## Troubleshooting

### Download Fails (Returns None/False)

**Possible causes:**
1. Network not connected
2. Invalid URL
3. HTTP error (404, 500, etc.)
4. Server timeout
5. SSL/TLS error

**Solution:**
```python
# Check network first
try:
    import network
    if not network.WLAN(network.STA_IF).isconnected():
        print("WiFi not connected!")
        return
except ImportError:
    pass  # Desktop mode

# Download with error handling
data = await DownloadManager.download_url(url)
if data is None:
    print("Download failed - check network and URL")
```

### Progress Callback Not Called

**Possible causes:**
1. Server doesn't send Content-Length
2. total_size not provided
3. Download too fast (single chunk)

**Solution:**
```python
# Provide explicit size
await DownloadManager.download_url(
    url,
    total_size=expected_size,  # Provide if known
    progress_callback=callback
)
```

### Memory Leak

**Problem:** Memory usage grows over time

**Cause:** Large files downloaded to memory

**Solution:**
```python
# Bad: Download large file to memory
data = await DownloadManager.download_url(large_url)  # OOM!

# Good: Download to file
success = await DownloadManager.download_url(
    large_url,
    outfile="/sdcard/large.bin"  # Streams to disk
)
```

### File Not Created

**Possible causes:**
1. Directory doesn't exist
2. Insufficient storage space
3. Permission error

**Solution:**
```python
import os

# Ensure directory exists
try:
    os.mkdir("/sdcard/downloads")
except OSError:
    pass  # Already exists

# Check available space
# (No built-in function - monitor manually)

# Download with error handling
success = await DownloadManager.download_url(
    url,
    outfile="/sdcard/downloads/file.bin"
)

if not success:
    print("Download failed - check storage and permissions")
```

### ValueError Exception

**Cause:** Both `outfile` and `chunk_callback` provided

**Solution:**
```python
# Bad: Conflicting parameters
await DownloadManager.download_url(
    url,
    outfile="file.bin",
    chunk_callback=process  # ERROR!
)

# Good: Choose one output mode
await DownloadManager.download_url(
    url,
    outfile="file.bin"  # File mode
)

# Or:
await DownloadManager.download_url(
    url,
    chunk_callback=process  # Stream mode
)
```

## Implementation Details

**Location**: `/home/user/MicroPythonOS/internal_filesystem/lib/mpos/net/download_manager.py`

**Pattern**: Module-level singleton (similar to AudioFlinger, SensorManager)

**Key features:**
- **Session pooling**: Single shared aiohttp.ClientSession
- **Refcount tracking**: Session lifetime based on active downloads
- **Thread safety**: Uses `_thread.allocate_lock()` for session access
- **Graceful degradation**: Returns None/False on desktop if aiohttp unavailable

**Dependencies:**
- `aiohttp` - HTTP client library (MicroPython port)
- `mpos.TaskManager` - For timeout handling (`wait_for`)

## Migration from Direct aiohttp

If you're currently using aiohttp directly:

```python
# Old: Direct aiohttp usage
import aiohttp

class MyApp(Activity):
    def onCreate(self):
        self.session = aiohttp.ClientSession()

    async def download(self, url):
        async with self.session.get(url) as response:
            return await response.read()

    def onDestroy(self, screen):
        # Bug: Can't await in non-async method!
        await self.session.close()
```

```python
# New: DownloadManager
from mpos import DownloadManager

class MyApp(Activity):
    # No session management needed!

    async def download(self, url):
        return await DownloadManager.download_url(url)

    # No onDestroy cleanup needed!
```

**Benefits:**
- No session lifecycle management
- Automatic connection reuse
- Thread-safe
- Consistent error handling

## See Also

- [TaskManager](task-manager.md) - Async task management
- [Preferences](preferences.md) - Persistent configuration storage
- [SensorManager](sensor-manager.md) - Sensor data access
