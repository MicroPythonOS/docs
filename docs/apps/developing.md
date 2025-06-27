# Developing Apps

Apps are written in MicroPython and installed in `/apps/`. See [Filesystem Layout](../architecture/filesystem.md) for the app directory structure.

Here we'll go over how to create and install a simple HelloWorld app.

More advanced app examples are available in the [source code repository](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem/apps).

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

```
from mpos.apps import Activity

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

The icon is a simple 64x64 pixel PNG image, which you can create with any tool, such as GIMP.

It's recommended to keep it as small as possible by setting compression level to 9 and not storing any metadata such as background color, resolution, creation time, comments, Exif data, XMP data, thumbnail or color profile.

The size will be somewhere between 3 and 7KB.

## Installing the App

The app can be installed by copying the top-level folder `com.micropythonos.helloworld/` (and its contents) to the `/apps/` folder.

### On desktop

You probably already have a clone of the [internal_filesystem](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem) that you're using to run MicroPythonOS on desktop.

Just copy or move your the top-level folder `com.micropythonos.helloworld/` (and its contents) to `internal_filesystem/apps/` and you're good to go!

### On ESP32

On the ESP32, you can use MicroPython tools such as [mpremote.py](https://github.com/micropython/micropython/tree/master/tools/mpremote) to copy files and folders from-to your device using the MicroPython REPL.

You may need to change this one line:

```
diff --git a/tools/mpremote/mpremote/main.py b/tools/mpremote/mpremote/main.py
index e6e3970..7f5d934 100644
--- a/tools/mpremote/mpremote/main.py
+++ b/tools/mpremote/mpremote/main.py
@@ -477,7 +477,7 @@ class State:
         self.ensure_connected()
         soft_reset = self._auto_soft_reset if soft_reset is None else soft_reset
         if soft_reset or not self.transport.in_raw_repl:
-            self.transport.enter_raw_repl(soft_reset=soft_reset)
+            self.transport.enter_raw_repl(soft_reset=False)
             self._auto_soft_reset = False
 
     def ensure_friendly_repl(self):
```

Then copy your app using:

```
/path/to/mpremote.py fs cp -r com.micropythonos.helloworld/ :/apps/
```

## Starting the App

If the app is installed into the /apps/ folder, it should show up in the launcher after refreshing it.

You can also launch it manually by typing this in the MicroPython REPL:

```
import mpos.apps; mpos.apps.start_app('apps/com.micropythonos.helloworld/')
```
