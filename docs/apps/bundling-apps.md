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

From the parent directory that contains your app folder, create a ZIP that includes the top-level folder.
You can usually do that by just right-clicking the folder (like `com.example.yourapp`) in your file explorer and choosing "Add to .zip" or "Compress".

Although you don't have to, it's recommended to make the `.mpk` deterministic by setting file timestamps to a fixed value, sorting entries, and excluding extra file attributes. That way, your .mpk will only change if the actual contents changed. Here's a script to do that:

```bash
cd internal_filesystem/apps/
find com.micropythonos.helloworld -exec touch -t 202501010000.00 {} \;
(find com.micropythonos.helloworld -type d; find com.micropythonos.helloworld -type f) | sort | TZ=CET zip -X -r -0 /tmp/com.micropythonos.helloworld_0.0.2.mpk -@
```

This:

- sorts directories before files
- uses `-0` for stored (uncompressed) entries
- uses `-X` to exclude extra file attributes
- places `com.micropythonos.helloworld/` as the first entry

## Python packages

An app can also be shipped as a proper Python package. This lets you split an app into many modules, use relative imports, and import shared code cleanly.

To opt in, the app folder must contain an `__init__.py` file, and **every directory on the path to each entrypoint** must also contain an `__init__.py` (or compiled `__init__.mpy`). The AppManager then imports the entrypoint as part of the package. The module name becomes the package fullname dotted with the entrypoint path, for example `com_micropythonos_nostr.chat_list_activity`.

Example `com_micropythonos_nostr` layout:

```
com_micropythonos_nostr/
com_micropythonos_nostr/__init__.py
com_micropythonos_nostr/MANIFEST.JSON
com_micropythonos_nostr/icon_64x64.png
com_micropythonos_nostr/chat_list_activity.py
com_micropythonos_nostr/nostr_service.py
com_micropythonos_nostr/...
```

The `MANIFEST.JSON` still declares `chat_list_activity.py` as the entrypoint; the framework detects the `__init__.py` and loads it as a package automatically.

Relative imports work as usual inside the package:

```python
from . import nostr_service
from .nostr_service import NostrService
```

## Native modules

Apps can include native C/C++ extension modules compiled as MicroPython `.native.mpy` files. Place the compiled `.mpy` in the app folder and import it like a normal module. For build instructions and an example, see [Native C/C++ Apps](native-apps.md).

## AppStore bundling

The apps at https://apps.MicroPythonOS.com are a curated, manually reviewed, vetted collection, often created and maintained by the MicroPythonOS core team.

These are bundled into an [`app_index.json`](https://github.com/MicroPythonOS/apps/blob/main/app_index.json) using [`scripts/bundle_apps.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/bundle_apps.sh) and then pushed to the [apps repo](https://github.com/MicroPythonOS/apps).

## BadgeHub.eu publishing

For community and event-badge publishing, MicroPythonOS also supports the [BadgeHub.eu](https://badgehub.eu) appstore backend. See [BadgeHub Apps](badgehub.md) for how to publish an `.mpk` there.
