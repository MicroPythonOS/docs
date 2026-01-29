# Developing Apps

Apps are written in MicroPython and installed in `/apps/`. See [Filesystem Layout](../architecture/filesystem.md) for the app directory structure.

Here we'll go over how to create and install a simple HelloWorld app.

More advanced examples are available in the [source code repository](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem/apps).

## Structure

Create the following file and folder structure:

```
com.micropythonos.helloworld/
├── assets/
│   └── hello.py
├── META-INF/
│   └── MANIFEST.JSON
└── res/
    └── mipmap-mdpi/
        └── icon_64x64.png
```

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

```
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
      "entrypoint": "assets/hello.py",
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

## Icon

The icon is a [simple 64x64 pixel PNG image](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/builtin/apps/com.micropythonos.launcher/res/mipmap-mdpi/icon_64x64.png), which you can create with any tool, such as GIMP.

It's recommended to keep it as small as possible by setting compression level to 9 and not storing any metadata such as background color, resolution, creation time, comments, Exif data, XMP data, thumbnail or color profile.

The size will probably be somewhere between 3 and 10KB.

## Installing the App

The app can be installed by copying the top-level folder `com.micropythonos.helloworld/` (and its contents) to the `/apps/` folder.

### On Desktop

You probably already have a clone of the [internal_filesystem](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem) that you're using to [run MicroPythonOS on desktop](../os-development/running-on-desktop.md).

Just copy or move your the top-level folder `com.micropythonos.helloworld/` (and its contents) to `internal_filesystem/apps/` and you're good to go!

### On ESP32

On the ESP32, you can use MicroPython tools such as [mpremote.py](https://github.com/micropython/micropython/tree/master/tools/mpremote) to copy files and folders from-to your device using the MicroPython REPL.

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

