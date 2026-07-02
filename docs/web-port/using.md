# Using the Web Port

MicroPythonOS can run entirely in a web browser thanks to a WebAssembly build. This is the easiest way to try the latest bleeding-edge code without flashing hardware.

## Live demo

The latest `main` branch is automatically built and published to GitHub Pages:

**https://micropythonos.github.io/MicroPythonOS/**

Open that link in a modern desktop browser and the OS will boot directly into the launcher.

## Refreshing the filesystem

The web port stores writable data (`/data` and `/apps`) in the browser's IndexedDB. The read-only system files are baked into the WebAssembly preload. If you want a completely clean start — for example, after a major upstream change or to clear stale app data — you need to wipe the stored filesystem:

1. Open the developer tools (`F12`).
2. Go to **Application** → **Storage** → **Local Storage / IndexedDB** (or the equivalent in your browser).
3. Clear site data / delete everything for `micropythonos.github.io`.
4. Close the tab or press `F5` to reload the page.

After the reload the OS will boot with a fresh filesystem. The bundled demo apps will be re-seeded on first run.

## Limitations in the browser

The web port is a faithful approximation of the real OS, but a browser is not a microcontroller. Expect the following differences:

- **No raw network sockets** — HTTP is handled through the browser's `fetch()` API, and CORS applies to cross-origin requests.
- **No Bluetooth, ADC, IMU, or camera** — hardware-specific features either raise errors or return stubs.
- **Persistence is per-origin** — clearing site data wipes `/data` and `/apps`.

For everything else, the web port behaves like a desktop build: the same apps, the same launcher, and the same Python runtime.

## Reporting issues

If the live demo fails to boot, open the browser console and look for fatal errors such as `memory access out of bounds`, `function signature mismatch`, or `MicroPythonOS exiting`. Transient messages like `no ADC`, `fetch failed`, or `aiorepl ... EIO` are expected and harmless.
