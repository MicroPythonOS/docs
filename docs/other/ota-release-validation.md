# OTA Upgrade Validation

An OTA release is only useful if the release *before* it can actually install
it. Before publishing a release, prove that upgrade path on real hardware:

```
previous release  ──OTA──▶  new release  ──OTA──▶  new release (again)
     0.20.0                      0.21.0                  0.21.0
```

The second pass matters too: it proves a device that is already on the new
release can install the same image again. That exercises the OTA write path and
the ESP32 OTA0/OTA1 partition switch a second time, which is exactly what a
user does when they reinstall or when a release is re-published.

The automated script is
[`scripts/validate_ota_release.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/validate_ota_release.py).
It drives the UI over serial and, when a webcam is present, verifies the
physical display against the device framebuffer with
[`scripts/mpos_screen_camera.py`](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/mpos_screen_camera.py).

This validation is a **release gate**, not an optional smoke test. It cannot run
in CI because it needs a badge, a real WiFi network and the published updates
server.

## What it verifies

- The previous release's factory image boots and reports its expected version.
- WiFi provisioning survives a reboot and the device comes online.
- The **OSUpdate boot service** posts the "OS update available" notification
  (the notification path, not a hidden API shortcut).
- Tapping the notification opens OSUpdate, which offers the expected version.
- "Install new version" downloads the `.ota` from the live updates server,
  writes the inactive OTA partition and reboots automatically.
- The new release boots, reports the target version and boots from the **other**
  OTA partition.
- The About app renders the same version and partition information that the
  scripts read over the REPL.
- OSUpdate offers "Install same version" on the new release; installing it
  writes the original OTA partition and reboots back into it.
- The final release still reports the target version, on the original partition.

With a webcam, each of those full-screen UI states is also compared with the
device framebuffer (SIFT/RANSAC alignment, structural similarity and focus
metrics). This catches a blank, frozen, rotated or otherwise physically broken
display that a serial-only check would miss.

## Prerequisites

- A badge flashed with a USB serial console (`/dev/ttyACM0` by default).
- The factory `.bin` of the **previous** release on the host. For 0.21.0 that is
  `MicroPythonOS_esp32s3_0.20.0.bin`.
- The updates server (or `tmp/ota_*/json` rollout) already serving the target
  release. The script reads `osupdate_<hardware_id>.json` and checks that its
  `version` matches `--target-version` before touching the device.
- Host packages: `esptool`, `pyserial`.
- For webcam checks: `ffmpeg`, `v4l-utils`, `numpy` and
  `opencv-python-headless`.
- A webcam pointed at the badge. For a repeatable rig, run a focus sweep once
  and reuse the camera config:

```bash
python3 scripts/mpos_screen_camera.py list
python3 scripts/mpos_screen_camera.py focus-sweep --port /dev/ttyACM0 \
    --start 30 --stop 55 --step 5
python3 scripts/mpos_screen_camera.py verify --port /dev/ttyACM0 \
    --out-dir tmp/camtests
```

### Camera thresholds for a small screen in frame

The default thresholds assume the badge fills a good part of the camera frame.
On a rig where it is small (or behind a glossy window) alignment is still
reliable, but NCC and effective-pixel scores are lower. Save a per-rig config
and pass it with `--camera-config`:

```json
{
  "device": "/dev/video0",
  "thresholds": {
    "min_inliers": 30,
    "min_ncc_gray": 0.55,
    "min_ncc_gradient": 0.55,
    "min_eff_px_per_screen_px": 0.75
  }
}
```

Camera controls are not passed to `verify`; set them with `v4l2-ctl` (or
`mpos_screen_camera.py config --apply`) before the run. They persist between
captures. On a bright LCD, manual exposure is often required:

```bash
v4l2-ctl -d /dev/video0 \
    -c auto_exposure=1 -c exposure_time_absolute=300 -c gain=0 \
    -c focus_automatic_continuous=0 -c focus_absolute=35
```

## Run the validation

```bash
export MPOS_WIFI_SSID='lab-network'
export MPOS_WIFI_PASSWORD='secret'

python3 scripts/validate_ota_release.py \
    --port /dev/ttyACM0 \
    --camera /dev/video0 \
    --relay-port /dev/ttyUSB0 \
    --base-bin MicroPythonOS_esp32s3_0.20.0.bin \
    --base-version 0.20.0 \
    --target-version 0.21.0 \
    --out-dir tmp/ota_validation
```

Use `--skip-flash` to validate a board that is already on the base release, and
`--no-camera` on a rig without a webcam. `--dry-run` validates the host inputs
and prints the plan without touching the device. `--relay-port` power-cycles the
badge between esptool retries, which makes flashing survive the flaky USB links
seen on some rigs.

If the target release is no longer the latest on the live server (for example
you are re-validating an older point release), serve its metadata locally and
point both the host check and the device at it:

```bash
python3 scripts/ota_validation_server.py \
    --version 0.21.0 \
    --ota-url https://updates.micropythonos.com/releases/esp32s3/MicroPythonOS_esp32s3_0.21.0.ota \
    --hardware-id <hardware_id> --port 8765

python3 scripts/validate_ota_release.py ... \
    --updates-url http://<host-lan-ip>:8765 \
    --mirror-url http://<host-lan-ip>:8765
```

The same commands are available as Make targets:

```bash
make ota-validation ARGS='--port /dev/ttyACM0 --camera /dev/video0 ...'
make ota-validation-server ARGS='--version 0.21.0 --ota-url ... --hardware-id ...'
```

`--mirror-url` only changes the device for the current boot; it is applied after
the WiFi reboot and before the OSUpdate boot service's 90-second check.

By default the script writes the full factory image and relies on the blank
`otadata` region inside that image, so the badge boots `ota_0` and the
`ota_0 → ota_1 → ota_0` sequence is meaningful. A full `esptool --erase-all` is
available with `--erase-all`, but it is slower and can wedge flaky USB links.
Either way, verify that the first check reports `partition.base ota_0`.

## Expected result

For a clean flash of 0.20.0 the report ends with:

```
[PASS] build.base_version BuildInfo.version = '0.20.0', expected '0.20.0'
[PASS] partition.base ota_0
[PASS] ota.notification osupdate.update_available
[PASS] osupdate.opened_from_notification OS update available
[PASS] osupdate.install_clicked Install new version
[PASS] ota.download_finished device rebooted after OTA
[PASS] build.version.after_first_ota BuildInfo.version = '0.21.0', expected '0.21.0'
[PASS] partition.after_first_ota running partition = 'ota_1', expected 'ota_1'
[PASS] osupdate.install_clicked Install same version
[PASS] build.version.after_second_ota BuildInfo.version = '0.21.0', expected '0.21.0'
[PASS] partition.after_second_ota running partition = 'ota_0', expected 'ota_0'
[PASS] partition.toggle ota_0 -> ota_1 -> ota_0
```

The same values must be visible in the About app:

```
Release version: 0.21.0
Partition.RUNNING: <Partition ... label=ota_1 ...>
Next update partition: <Partition ... label=ota_0 ...>
```

Note that the second pass has **no notification**: OSUpdate only notifies when
the server version is newer. The script opens OSUpdate directly and uses the
enabled "Install same version" button, which is the intended UI for that case.

## Artifacts

Everything goes to `--out-dir` (default `tmp/ota_validation`):

- `report.json` — every check with pass/fail and details, written incrementally
  so a failed run still leaves evidence.
- `<label>_frame.jpg`, `<label>_aligned.png`, `<label>_compare.jpg`,
  `<label>_edges.jpg` — webcam captures, the aligned screen, the
  device-vs-camera comparison and edge maps for each camera checkpoint
  (`base`, `update_available`, `after_first_ota`, `same_version_available`,
  `after_second_ota`).
- `metrics.jsonl` — the numeric camera metrics for each checkpoint.

The comparison images are the fastest way to review a failure: the left panel is
the device framebuffer, the right panel is what the camera saw.

## Manual fallback

If the automated script fails on the rig itself, perform the same steps by hand
and record the About screens before and after each update:

1. Flash the previous release's factory `.bin` with `esptool write_flash 0
   <bin>` (add `--erase-all` if your USB link is reliable), boot it and connect
   WiFi.
2. Wait for "OS update available", open the notification drawer and tap it.
3. In OSUpdate, confirm the offered version, tap "Install new version" and let
   the badge reboot.
4. Open About and record `Release version` and `Partition.RUNNING`. It must be
   the target version on the other OTA slot.
5. Open OSUpdate again, tap "Install same version", let it reboot.
6. Open About again. It must still be the target version, now back on the
   original OTA slot.
7. Keep the About screenshots and the `report.json` with the release notes.

## Failure triage

- **No notification**: confirm the device has an IP, then check that
  `https://updates.micropythonos.com/osupdate_<hardware_id>.json` serves the
  target version. The boot service waits 90 s after boot before checking. Use
  `--force-check` only to separate a boot-service problem from a networking
  problem; a release gate should pass without it.
- **Wrong version offered**: the `updates` repo was not rolled over, or the
  board's `hardware_id` maps to a different JSON file. Verify with
  `curl -s https://updates.micropythonos.com/osupdate_<hardware_id>.json`.
- **Partition did not toggle**: the base image did not reset `otadata` (use the
  factory `.bin`, or flash with `--erase-all`), or `set_boot()` failed. Re-run
  with a clean flash before treating it as a release blocker.
- **Serial write timeout or Input/output error**: the USB-CDC link wedged.
  Power-cycle the badge (relay or unplug) and re-run. The gate deliberately does
  not auto-reset during a download: interrupting an OTA write is worse than
  failing the run. `--relay-port` only retries the flash step.
- **Camera check fails**: check focus/exposure first
  (`focus-sweep`, then `verify`). If the comparison image shows the wrong
  screen, the framebuffer reference or the camera alignment is stale; delete
  `~/.cache/mpos_screen_camera` and retry.

The helper logic has hardware-free tests:

```bash
python3 tests/cpython_validate_ota_release.py
```
