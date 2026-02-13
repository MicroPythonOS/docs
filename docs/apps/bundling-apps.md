# Bundling Apps

To bundle your app in an .mpk file, just make an uncompressed "zip" file of it, without including the top-level `com.micropythonos.helloworld/` folder.

It's recommended to make the .mpk file deterministic by:
- setting the file timestamps to a fixed value
- sorting the files, so the order is fixed
- excluding extra file attributes and directories

For example:

```
cd com.micropythonos.helloworld/
find . -type f -exec touch -t 202501010000.00 {} \; # set fixed timestamp to have a deterministic zip file
find . -type f | sort | TZ=CET zip -X -r -0 /tmp/com.micropythonos.helloworld_0.0.2.mpk -@ # sort, -Xclude extra attributes, -recurse into directories and -0 compression
```

## AppStore bundling

The apps at https://apps.MicroPythonOS.com are a curated, manually reviewed, vetted collection, often created and maintainced by the MicroPythonOS core team.

These are manually bundled into a [app_index.json](https://github.com/MicroPythonOS/apps/blob/main/app_index.json) using [`scripts/bundleapps.sh`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/bundleapps.sh) and then pushed to the [apps repo](https://github.com/MicroPythonOS/apps).
