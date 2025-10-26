## Running on Linux or MacOS

1. Download a release binary (e.g., `MicroPythonOS_amd64_Linux`, `MicroPythonOS_amd64_MacOS`) or build your own [on MacOS](../os-development/macos.md) or [Linux](../os-development/linux.md).

2. Get the `local_filesystem/` folder

    You probably already have a local clone that contains the [internal_filesystem](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/internal_filesystem).

    If not, then clone it now:

    <pre>
    ```
    git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
    cd MicroPythonOS/
    ```
    </pre>

3. Start it from the local_filesystem/ folder:

    <pre>
    ```
    cd internal_filesystem/ # make sure you're in the right place to find the filesystem
    /path/to/release_binary -X heapsize=32M -v -i -c "$(cat boot_unix.py main.py)"
    ```
    </pre>

    There's also a convenient `./scripts/run_desktop.sh` script that will attempt to start the latest build that you compiled yourself.

### Modifying files

You'll notice that, whenever you change a file on your local system, the changes are immediately visible whenever you reload the file.

This results in a very quick coding cycle.

Give this a try by editing `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py` and then restarting the "About" app. Powerful stuff!
