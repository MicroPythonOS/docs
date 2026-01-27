# DownloadManager

Centralized HTTP download service with flexible output modes.

## Overview

- **Three output modes** - Download to memory, file, or stream with callbacks
- **Retry logic** - Automatic retry on chunk failures (3 attempts)
- **Progress tracking** - Real-time download progress callbacks
- **Resume support** - HTTP Range headers for partial downloads
- **Memory efficient** - Chunked downloads (1KB chunks)

## Quick Start

### Async Download to Memory

```python
from mpos import DownloadManager

data = await DownloadManager.download_url("https://api.example.com/data.json")
if data:
    import json
    parsed = json.loads(data)
```

### Sync Download to Memory

```python
from mpos import DownloadManager

# Synchronous usage - no await needed
data = DownloadManager.download_url("https://api.example.com/data.json")
if data:
    import json
    parsed = json.loads(data)
```

### Download to File

```python
success = await DownloadManager.download_url(
    "https://example.com/file.bin",
    outfile="/sdcard/file.bin"
)
```

### Download with Progress

```python
async def update_progress(percent):
    print(f"Downloaded: {percent}%")

success = await DownloadManager.download_url(
    "https://example.com/large_file.bin",
    outfile="/sdcard/large_file.bin",
    progress_callback=update_progress
)
```

### Stream Processing

```python
async def process_chunk(chunk):
    print(f"Got {len(chunk)} bytes")

success = await DownloadManager.download_url(
    "https://example.com/stream",
    chunk_callback=process_chunk
)
```

## Sync/Async Compatibility

The `DownloadManager.download_url()` method automatically detects whether it's called from an async or sync context:

- **From async context**: Returns a coroutine that can be awaited
- **From sync context**: Runs synchronously and returns the result directly

This means you can use the same API in both async and sync code without any wrapper functions.

## API Reference

### `DownloadManager.download_url()`

```python
def download_url(url, outfile=None, total_size=None,
                 progress_callback=None, chunk_callback=None,
                 headers=None, speed_callback=None)
```

**Note:** This method works in both async and sync contexts. When called from an async function, it returns a coroutine. When called from a sync function, it runs synchronously.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | str | URL to download (required) |
| `outfile` | str | Path to write file (optional) |
| `total_size` | int | Expected size in bytes for progress tracking (optional) |
| `progress_callback` | async function | Callback for progress updates (optional) |
| `chunk_callback` | async function | Callback for streaming chunks (optional) |
| `headers` | dict | Custom HTTP headers (optional) |
| `speed_callback` | async function | Callback for download speed (optional) |

**Returns:**
- **Memory mode**: `bytes` on success
- **File mode**: `True` on success
- **Stream mode**: `True` on success

**Raises:**
- `ValueError` - If both `outfile` and `chunk_callback` are provided
- `OSError` - On network or file I/O errors

### `DownloadManager.is_network_error(exception)`

Check if an exception is a recoverable network error.

```python
try:
    await DownloadManager.download_url(url)
except Exception as e:
    if DownloadManager.is_network_error(e):
        # Network error - retry
        await asyncio.sleep(2)
    else:
        # Fatal error
        raise
```

**Detected errors:**
- Error codes: `-110` (ETIMEDOUT), `-113` (ECONNABORTED), `-104` (ECONNRESET), `-118` (EHOSTUNREACH), `-202` (DNS error)
- Error messages: "connection reset", "connection aborted", "broken pipe", "network unreachable", "host unreachable"

### `DownloadManager.get_resume_position(outfile)`

Get the current size of a partially downloaded file.

```python
resume_from = DownloadManager.get_resume_position("/sdcard/file.bin")
if resume_from > 0:
    headers = {'Range': f'bytes={resume_from}-'}
    await DownloadManager.download_url(url, outfile=outfile, headers=headers)
else:
    await DownloadManager.download_url(url, outfile=outfile)
```

## Common Patterns

### Download with Timeout

```python
from mpos import TaskManager, DownloadManager

async def download_with_timeout(url, timeout=10):
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
async def download_icons(apps):
    for app in apps:
        if not app.icon_data:
            try:
                app.icon_data = await TaskManager.wait_for(
                    DownloadManager.download_url(app.icon_url),
                    timeout=5
                )
            except Exception as e:
                print(f"Icon download failed: {e}")
```

### Resume Partial Download

```python
async def resume_download(url, outfile):
    bytes_written = DownloadManager.get_resume_position(outfile)
    
    if bytes_written > 0:
        headers = {'Range': f'bytes={bytes_written}-'}
    else:
        headers = None
    
    success = await DownloadManager.download_url(
        url,
        outfile=outfile,
        headers=headers
    )
    return success
```

### Error Handling with Retry

```python
async def robust_download(url, outfile, max_retries=3):
    for attempt in range(max_retries):
        try:
            success = await DownloadManager.download_url(url, outfile=outfile)
            if success:
                return True
        except Exception as e:
            if DownloadManager.is_network_error(e):
                if attempt < max_retries - 1:
                    await asyncio.sleep(2)
                    continue
            else:
                raise
    return False
```

## Performance Considerations

### Memory Usage

- Chunk buffer: 1KB
- Progress callback overhead: ~50 bytes
- Total per download: ~1-2KB

### Concurrent Downloads

Limit to 5-10 concurrent downloads. Use timeouts to prevent stuck downloads.

```python
async def download_batch(urls, max_concurrent=5):
    for i in range(0, len(urls), max_concurrent):
        batch = urls[i:i+max_concurrent]
        for url in batch:
            try:
                data = await TaskManager.wait_for(
                    DownloadManager.download_url(url),
                    timeout=10
                )
            except Exception as e:
                print(f"Download failed: {e}")
```

### Large Files

Always download large files to disk, not memory:

```python
# Bad: OOM risk
data = await DownloadManager.download_url(large_url)

# Good: Streams to disk
success = await DownloadManager.download_url(
    large_url,
    outfile="/sdcard/large.bin"
)
```

## Troubleshooting

### Download Fails (Raises Exception)

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
try:
    data = await DownloadManager.download_url(url)
except OSError as e:
    print(f"Download failed: {e}")
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
    total_size=expected_size,
    progress_callback=callback
)
```

### ValueError Exception

**Cause:** Both `outfile` and `chunk_callback` provided

**Solution:**
```python
# Choose one output mode
await DownloadManager.download_url(url, outfile="file.bin")
# Or:
await DownloadManager.download_url(url, chunk_callback=process)
```

## Implementation

**Location:** `MicroPythonOS/internal_filesystem/lib/mpos/net/download_manager.py`

**Key features:**
- Per-request aiohttp sessions
- Automatic retry on chunk failures
- Thread-safe for concurrent downloads
- Graceful degradation on desktop if aiohttp unavailable

**Dependencies:**
- `aiohttp` - HTTP client library
- `mpos.TaskManager` - For timeout handling

## See Also

- [TaskManager](task-manager.md) - Async task management
- [ConnectivityManager](connectivity-manager.md) - Network connectivity monitoring
- [SharedPreferences](preferences.md) - Persistent storage
