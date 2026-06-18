# Bundling Apps

Apps are distributed as `.mpk` files. An `.mpk` is a ZIP archive (usually stored without compression to speed up installation on device) with a strict layout:

- The **first entry** in the ZIP stream **must** be a top-level directory whose name matches the app's fullname exactly, followed by `/` (for example, `com.micropythonos.helloworld/`).
- That top-level directory **must** be the only top-level entry in the archive.
- All other files and subdirectories live under that single top-level directory.

So a valid archive looks like this in stream order:

```
com.micropythonos.helloworld/
com.micropythonos.helloworld/MANIFEST.JSON
com.micropythonos.helloworld/icon_64x64.png
com.micropythonos.helloworld/hello.py
```

MicroPythonOS validates this layout while extracting, and rejects packages that do not follow it. This ensures packages are unambiguous and can be streamed safely onto devices with limited storage.

## Creating an .mpk

From the parent directory that contains your app folder, create a stored (uncompressed) ZIP that includes the top-level folder. It's recommended to make the `.mpk` deterministic by setting file timestamps to a fixed value, sorting entries, and excluding extra file attributes:

```
cd internal_filesystem/apps/
find com.micropythonos.helloworld -exec touch -t 202501010000.00 {} \;
(find com.micropythonos.helloworld -type d; find com.micropythonos.helloworld -type f) | sort | TZ=CET zip -X -r -0 /tmp/com.micropythonos.helloworld_0.0.2.mpk -@
```

This:

- sorts directories before files
- uses `-0` for stored (uncompressed) entries
- uses `-X` to exclude extra file attributes
- places `com.micropythonos.helloworld/` as the first entry

## AppStore bundling

The apps at https://apps.MicroPythonOS.com are a curated, manually reviewed, vetted collection, often created and maintained by the MicroPythonOS core team.

These are bundled into an [`app_index.json`](https://github.com/MicroPythonOS/apps/blob/main/app_index.json) using [`scripts/bundle_apps.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/bundle_apps.sh) and then pushed to the [apps repo](https://github.com/MicroPythonOS/apps).
