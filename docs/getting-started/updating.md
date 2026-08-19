# Updating

## The supported way: over-the-air updates

The only supported way to update MicroPythonOS is the built-in **OSUpdate**
app (over-the-air updates). OTA updates write the new OS to the inactive
system partition and leave your storage partition untouched: installed
apps, preferences (WiFi credentials, wallet-display configs, Nostr keys),
and user data are all preserved.

When an update is available, the notification bar offers it, or you can
open the OSUpdate app directly.

## Reinstalling with the web installer is not an update

The [WebSerial installer](https://install.micropythonos.com) performs a
**clean install**, and reflashing a device that already has MicroPythonOS
on it can cost you your data:

- If you select **"Erase device"** during installation, your storage
  partition — all apps, settings, and keys — is **certainly** erased.
- Even without erasing, a newer version **may use a different partition
  layout** than the one on your device (this can happen from time to time
  as MicroPythonOS evolves). In that case the storage partition is
  reformatted on first boot, **without any prompt**.

So, just like for any electronics device: **make a backup of important
files before reinstalling the software.** For example, over USB with
[mpremote](https://docs.micropython.org/en/latest/reference/mpremote.html):

```bash
mpremote fs cp -r :/apps :/prefs :/data ./mpos-backup/
```

## Advanced: manual offline update

If a device has no network access, you can apply an update manually by
writing the over-the-air update file (for example
`MicroPythonOS_esp32s3_0.17.3.ota` from the
[releases](https://github.com/MicroPythonOS/MicroPythonOS/releases)) to
the OTA application offsets in flash with esptool. For the esp32s3 build,
for example, the two OTA slots are at `0x20000` and `0x3A0000`:

```bash
python3 -m esptool --chip esp32s3 write_flash 0x20000 MicroPythonOS_esp32s3_0.17.3.ota
```

This writes the same bytes an OTA update would, so the storage partition
is left alone — but it is an advanced action: the offsets differ per
board/build, and writing to the wrong offset can corrupt the device's
filesystem. Making a backup first is strongly advised.
