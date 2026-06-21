# Developing Apps

Apps are written in MicroPython and installed in `/apps/`. See [Filesystem Layout](../architecture/filesystem.md) for the app directory structure.

Here we'll go over how to create and install a simple HelloWorld app.

More advanced examples are available in the [source code repository](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem/apps).

## Structure

Create the following file and folder structure:

```
com.micropythonos.helloworld/
├── MANIFEST.JSON
├── icon_64x64.png
└── hello.py
```

This flat layout puts the required `MANIFEST.JSON` and app icon at the top level, alongside your Python files. Subfolders are still allowed if you want to organize larger apps, but **each directory costs about 8 KiB of storage in LittleFS**, so use them sparingly on device.

## App code

In `hello.py`, put:

```python
import lvgl as lv
from mpos import Activity

class Hello(Activity):

    def onCreate(self):
        screen = lv.obj()
        label = lv.label(screen)
        label.set_text('Hello World!')
        label.center()
        self.setContentView(screen)
```

The code above creates a new screen, adds a label, sets the label text, centers the label and activates the screen.

## Manifest

In `MANIFEST.JSON`, put:

```json
{
  "name": "HelloWorld",
  "publisher": "MicroPythonOS",
  "short_description": "Minimal app",
  "long_description": "Demonstrates the simplest app.",
  "fullname": "com.micropythonos.helloworld",
  "version": "0.0.2",
  "category": "development",
  "activities": [
    {
      "entrypoint": "hello.py",
      "classname": "Hello",
      "intent_filters": [
        {
          "action": "main",
          "category": "launcher"
        }
      ]
    }
  ]
}
```

### Services

Apps can also declare **services** — background components that run at boot time with no user interface. Services are defined in the `"services"` array of the manifest:

```json
"services": [
  {
    "entrypoint": "my_boot_service.py",
    "classname": "MyBootService",
    "intent_filters": [
      {
        "action": "boot_completed"
      }
    ]
  }
]
```

Each service entry has:

| Field | Description |
|-------|-------------|
| `entrypoint` | Path to the Python file (relative to the app root) |
| `classname` | Name of the `Service` subclass in that file |
| `intent_filters` | Array of `{ "action": "..." }` objects. Use `"boot_completed"` to run at startup |

Services that subscribe to `"boot_completed"` are started automatically during system boot, after the launcher is displayed. See the [Service documentation](../frameworks/service.md) for details on writing and using services.

## Handling the view action

Apps can register themselves as file viewers by declaring an `intent_filter` with `action: "view"` and a `pathPattern` list in `MANIFEST.JSON`. This lets other apps (and the built-in `FileExplorerActivity`) open files with your app.

### Manifest example

```json
{
  "fullname": "com.example.imageviewer",
  "name": "Image Viewer",
  "version": "1.0.0",
  "activities": [
    {
      "entrypoint": "imageview.py",
      "classname": "ImageView",
      "intent_filters": [
        { "action": "main", "category": "launcher" },
        { "action": "view", "mimeType": "image/*", "pathPattern": [".png", ".jpg", ".jpeg"] }
      ]
    }
  ]
}
```

`pathPattern` entries are matched case-insensitively against the file extension. A leading `*` is optional. `mimeType` is recorded for documentation but is not used for matching.

### Receiving the file

When the system opens your activity via the `view` action, the file path is available in `intent.data` and also in the `filename` extra:

```python
from mpos import Activity

class ImageView(Activity):
    def onResume(self, screen):
        super().onResume(screen)
        path = self.getIntent().extras.get("filename") or self.getIntent().data
        if path:
            self.open_image(path)
```

### Opening a file from your app

To open a file in whatever app is registered for it, send a `view` intent:

```python
from mpos import Intent, Activity

class MyActivity(Activity):
    def on_file_click(self, path):
        self.startActivity(Intent(action="view", data=path))
```

If multiple apps can handle the file type, MicroPythonOS automatically shows an "Open with" chooser.

## Icon

The icon is a simple 64x64 pixel PNG image named `icon_64x64.png` in the app root, which you can create with any tool, such as GIMP.

It's recommended to keep it as small as possible by setting compression level to 9 and not storing any metadata such as background color, resolution, creation time, comments, Exif data, XMP data, thumbnail or color profile.

The size will probably be somewhere between 3 and 10KB.

## Installing the App

The app can be installed by copying the top-level folder `com.micropythonos.helloworld/` (and its contents) to the `/apps/` folder.

### On Desktop

If you are [running MicroPythonOS on desktop](../os-development/running-on-desktop.md) from a source checkout, copy or move the top-level folder `com.micropythonos.helloworld/` (and its contents) to `internal_filesystem/apps/` and you're good to go.

### On ESP32

On the ESP32, you can use MicroPython tools such as [mpremote.py](https://github.com/micropython/micropython/tree/master/tools/mpremote) to copy files and folders from-to your device using the MicroPython REPL.

It's also included in the MicroPythonOS repo at `./lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py`

Then connect your device with a cable and install your app using:

```
/path/to/mpremote.py mkdir :/apps
/path/to/mpremote.py fs cp -r com.micropythonos.helloworld/ :/apps/
```

Take a look at [`scripts/install.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/install.sh) for convenient "install everything" or "install one app" scripting.

There might also exist MicroPython File Managers with a graphical user interface, if you prefer.

## Starting your App

If the app is installed into the /apps/ folder, it should show up in the launcher after refreshing it.

You can also launch it manually by typing this in the MicroPython REPL:

```python
from mpos import AppManager
AppManager.start_app('com.micropythonos.helloworld')
```
## Further reading

Now that your first app works, you could start building apps using standard LVGL for MicroPython calls for the UI and the [MicroPythonOS frameworks](../architecture/frameworks.md).

But before you do, you might want to check out the [app lifecycle](app-lifecycle.md) to understand why we added an onCreate() and what other lifecycle functions are available.

