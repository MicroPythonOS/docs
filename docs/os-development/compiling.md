## Download the code

Clone the repositories:

```
git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
cd MicroPythonOS/
```

That will take a while, because it recursively clones MicroPython, LVGL, ESP-IDF and all their dependencies.

#### Optional: updating the code

If you already have an old clone and you want to update it, the easiest is to just delete it and re-clone.

But if you need to save on bandwidth and time, you can instead do the following, which will *throw away all local modifications*:

```
cd MicroPythonOS/
git submodule foreach --recursive 'git clean -f; git checkout .'
git pull --recurse-submodules
```

## Compile the code

Use the build_mpos.sh script for convenience.

Usage:

```
./scripts/build_mpos.sh <target system>
```

**Target systems**: `esp32`, `esp32s3`, `unix` (= Linux) and `macOS`

**Examples**:

```
./scripts/build_mpos.sh esp32
./scripts/build_mpos.sh esp32s3
./scripts/build_mpos.sh unix
./scripts/build_mpos.sh macOS
```

The resulting build file will be in `lvgl_micropython/build/`, for example:

- `lvgl_micropython/build/lvgl_micropy_unix`
- `lvgl_micropython/build/lvgl_micropy_macOS`
- `lvgl_micropython/build/lvgl_micropy_ESP32_GENERIC_S3-SPIRAM_OCT-16.bin`

