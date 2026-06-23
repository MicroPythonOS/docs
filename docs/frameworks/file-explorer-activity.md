# FileExplorerActivity

`FileExplorerActivity` is a reusable file browser built into MicroPythonOS. It can browse the local filesystem or let the user pick one or more files, and it integrates with the [view action](../architecture/intents.md#file-type-intents-and-the-view-action) to open files in the appropriate app.

## Overview

The activity is registered for the `pick_file` action, so any app can ask the user to choose files without implementing its own file picker. It is also used directly as the built-in **File Manager** app.

Two modes are supported:

- **`browse`** (default): navigate directories, open files, rename or delete them.
- **`pick`**: select one or more files and return them to the caller. The `pick_file` action automatically uses pick mode, so callers do not need to set the `"mode"` extra.

## Launching

### Pick files with `startActivityForResult`

Using the `pick_file` action automatically opens `FileExplorerActivity` in pick mode, so you do **not** need to set the `"mode"` extra.

```python
from mpos import Intent, Activity

class MyActivity(Activity):
    def _open_file_clicked(self, event):
        intent = Intent(action="pick_file")
        intent.putExtra("start_dir", "/data/audio")
        intent.putExtra("path_pattern", [".wav"])
        self.startActivityForResult(intent, self._on_file_picked)

    def _on_file_picked(self, result):
        if result and result.get("result_code"):
            paths = result.get("data", {}).get("paths", [])
            for path in paths:
                print("Selected:", path)
```

### Browse the filesystem

```python
from mpos import Intent, FileExplorerActivity

class MyActivity(Activity):
    def _browse_files(self):
        intent = Intent(activity_class=FileExplorerActivity)
        intent.putExtra("start_dir", "/sdcard")
        self.startActivity(intent)
```

## Intent extras

| Extra | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | `str` | `"browse"` (or `"pick"` when action is `"pick_file"`) | `"browse"` or `"pick"`. Only needed when launching `FileExplorerActivity` directly; the `"pick_file"` action implies picker mode automatically. |
| `start_dir` | `str` | `"."` | Directory to open. Non-existent paths are walked up to the first existing parent, falling back to `"/"`. |
| `path_pattern` | `str` or `list` | `[]` | File extensions to accept in pick mode, e.g. `[".png", ".jpg"]`. Strings may include a leading `*` (`"*.wav"`). An empty list accepts all files. |

## Pick mode result

When the user confirms the selection, the result callback receives:

```python
{
    "result_code": True,
    "data": {
        "paths": ["/path/to/file1.wav", "/path/to/file2.wav"]
    }
}
```

If no file is selected, the current directory path is returned instead:

```python
{
    "result_code": True,
    "data": {
        "paths": ["/data/audio/"]
    }
}
```

When the user cancels, the result is:

```python
{
    "result_code": False,
    "data": {}
}
```

## Browse mode behavior

In browse mode, tapping a directory navigates into it. Tapping a file sends a `view` intent:

```python
self.startActivity(Intent(action="view", data=path))
```

The system then resolves the file to the appropriate viewer using the manifest-declared `pathPattern` of installed apps, or falls back to the framework's generic `ViewActivity`.

Long-pressing a file or folder opens an action bar with **Delete**, **Rename**, and **Cancel** options. Delete shows a confirmation dialog; Rename launches `RenameActivity`.

## Example: image picker

```python
import lvgl as lv
from mpos import Activity, Intent

class GalleryLauncher(Activity):
    def onCreate(self):
        screen = lv.obj()
        btn = lv.button(screen)
        lv.label(btn).set_text("Choose image")
        btn.add_event_cb(self._choose_image, lv.EVENT.CLICKED, None)
        self.setContentView(screen)

    def _choose_image(self, event):
        intent = Intent(action="pick_file")
        intent.putExtra("start_dir", "/data/images")
        intent.putExtra("path_pattern", [".png", ".jpg", ".jpeg", ".raw"])
        self.startActivityForResult(intent, self._on_image_picked)

    def _on_image_picked(self, result):
        if result and result.get("result_code"):
            paths = result.get("data", {}).get("paths", [])
            if paths and not paths[0].endswith("/"):
                print("Opening image:", paths[0])
```

## See Also

- [Intents](../architecture/intents.md) - How implicit intents and the `view` action work
- [AppManager](app-manager.md) - How file-type handlers are resolved
- [Creating Apps](../apps/creating-apps.md) - Declaring `view` handlers in an app manifest
