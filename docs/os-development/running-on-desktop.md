## Running on desktop

1. **Make sure you have the `internal_filesystem/` folder**

    If you built from source, you will already have a local clone that contains it.

    If you used a pre-built binary, then you need to get a local clone and change into that directory with:

    <pre>
    ```
    git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
    cd MicroPythonOS/
    ```
    </pre>

2. **Make sure you have the software**

    If you built from source, you will already have it in `lvgl_micropython/build/lvgl_micropy_unix`

    If you downloaded a pre-built binary (like `MicroPythonOS_amd64_macOS_0.7.1.bin` or `MicroPythonOS_amd64_linux_0.7.1.elf`), then copy it to the right location:

    <pre>
    ```
    mkdir -p lvgl_micropython/build
    cp /Users/yourname/MicroPythonOS_amd64_macOS_0.7.1.bin lvgl_micropython/build/lvgl_micropy_macOS # for macOS
    cp /home/yourname/MicroPythonOS_amd64_linux_0.7.1.elf lvgl_micropython/build/lvgl_micropy_unix # for Linux or WSL2 on Windows 11
    ``` 
    </pre>

3. **Start the software:**
    
    You're now ready to run it with:

    <pre>
    ```
    ./scripts/run_desktop.sh
    ```
    </pre>

### Notes on MacOS

If you get an error about a missing `/opt/homebrew/opt/libffi/lib/libffi.8.dylib` then fix that with: `brew install libffi`

If you get an error about the code being unsigned, then allow it like this:

![Allow Anyway on MacOS](/os-development/macos-allow-anyway.png)

    

## Making Changes on Desktop

You'll notice that whenever you change a file in `internal_filesystem/`, the changes are immediately visible on desktop when you reload the file or restart the app.

When you run `./scripts/run_desktop.sh`, the OS runs the MicroPythonOS scripts **directly from `internal_filesystem/`**. This means:

- **All changes to Python files are immediately active** - no build or install needed
- **Instant testing** - edit a file, restart the app, see the changes
- **Fast iteration cycle** - the recommended way to develop and test

**Try it yourself:**

1. Edit `internal_filesystem/builtin/apps/com.micropythonos.about/assets/about.py`
2. Run `./scripts/run_desktop.sh`
3. Open the About app
4. See your changes immediately!


## Making Changes on ESP32

Once you've tested your changes on desktop and they work correctly, or you're doing things you can't test on desktop, then you can deploy to physical hardware.

{!os-development/installing-on-esp32.md!}


