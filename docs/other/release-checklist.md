# Release Checklist

Follow these steps to create a new release of MicroPythonOS. Replace `<version>`
with the new version (for example `0.21.0`).

**Update Version Numbers**:

- Set `release = "<version>"` in `BuildInfo.version` in
  `internal_filesystem/lib/mpos/build_info.py` (the field used to be called
  `CURRENT_OS_VERSION`).
- Bump the version in the `MANIFEST.JSON` of every modified app:

```
git diff --stat <previous-tag> -- internal_filesystem/  # each app change needs a MANIFEST bump
make check-app-versions                                 # lists apps still missing a bump
```

**Update Changelog**:

- Compare `CHANGELOG.md` against `git log <previous-tag>..main` and
  `git diff <previous-tag>`; cover everything since the last release.
- Keep the entries short and user-facing, like the previous releases.
- Move the "Future release (next version)" entries under a new `<version>`
  heading, leaving an empty "Future release" section at the top.
- Check that the new section is JSON-friendly. `scripts/changelog_to_json.sh`
  rewrites double quotes to single quotes and escapes newlines, so run it on a
  copy that contains only the new section:

```
mkdir -p tmp/cl_json && cp CHANGELOG.md tmp/cl_json/
cd tmp/cl_json && bash ../../scripts/changelog_to_json.sh
```

**Commit and push** the code repo and any external repositories (for example
[LightningPiggy](https://github.com/LightningPiggy/LightningPiggyApp)). Wait for
the GitHub builds at https://github.com/MicroPythonOS/MicroPythonOS/actions to
be green.

**Tag the commit**

Tag the commit the builds ran on (the repository uses lightweight tags):

```
git tag <version> <commit>
git push origin tag <version>
```

This runs `.github/workflows/release.yml`, which downloads the build artifacts
and creates a [draft release](https://github.com/MicroPythonOS/MicroPythonOS/releases).
Check that all expected assets are present: `esp32`, `esp32s3`, `esp32-small`,
`esp32s3-unphone`, `esp32p4` (`.bin` and `.ota`), the desktop builds and
`MicroPythonOS_web_<version>.zip`.

**Test the draft release**

- Flash a release `.bin` (web installer, or `esptool write_flash 0 <bin>`) on
  at least one ESP32-S3 board.
- Test OTA, desktop and web as applicable, and run the on-device test suite
  (see `tests/README.md`).
- **Required: validate the previous release can install this one.** Flash the
  previous release's factory `.bin`, then run the OTA release validation (with
  webcam verification when the rig has a camera). It must pass both the
  `previous → new` and `new → new` passes, with the OTA partition alternating
  `ota_0 → ota_1 → ota_0`:

  ```
  python3 scripts/validate_ota_release.py \
      --port /dev/ttyACM0 --camera /dev/video0 \
      --relay-port /dev/ttyUSB0 \
      --base-bin MicroPythonOS_esp32s3_<previous>.bin \
      --base-version <previous> --target-version <version>
  ```

  Keep `tmp/ota_validation/report.json` and the camera comparison images with
  the release notes. See [OTA Upgrade Validation](ota-release-validation.md)
  for the full procedure and manual fallback.

**Publish the draft release**

- Copy the `<version>` section from `CHANGELOG.md` into the release notes.
- Make sure **Set as the latest release** is selected. Older workflow versions
  created the draft with `make_latest: false`, which left the previous release
  as `latest`.
- Publish.

**Optional: build and test dependent projects**

Projects like https://github.com/fri3dcamp/badge_firmware_Micropythonos rely on
MicroPythonOS releases; rebuild them now or after the next steps.

**Release to Over-The-Air update**

The [updates repo](https://github.com/MicroPythonOS/updates) keeps the metadata
in `osupdate*.json`. It is a GitHub archive: the live
updates.micropythonos.com server is updated by rsyncing it (see its README).
Most board files are **symlinks** to the three real files
`osupdate_esp32.json`, `osupdate_esp32s3.json` and `osupdate_esp32p4.json`;
edit the real files (and any regular board file that is not a symlink).

The whole step is automated from the published release:

```
python3 scripts/release_metadata.py <version> --updates-only --commit --push
```

It downloads the `.ota` assets from the GitHub release, copies them into
`updates/releases/<target>/` (`esp32`, `esp32s3`, `esp32p4`) and sets
`version`, `download_url` and `changelog` in the three real JSON files (the
changelog is the `<version>` section of `CHANGELOG.md`, escaped like
`scripts/changelog_to_json.sh` does). Without `--commit --push` it only
prepares the working copies so the diff can be reviewed first.

- Run `../upload_updates.sh` from the parent folder (one level up). It rsyncs with
  `--delete`, so run it from the working copy that has all previous `.ota`
  files.
- Verify: `curl -s https://updates.micropythonos.com/osupdate_esp32s3.json`.

**Release to the web installer**

The [install repo](https://github.com/MicroPythonOS/install) backs
install.micropythonos.com and is also a GitHub archive: the live server is
updated by rsyncing it (see its README).

```
python3 scripts/release_metadata.py <version> --install-only --commit --push
```

It downloads the `.bin` assets from the GitHub release, copies them into
`install/firmware_images/<target>/`, writes the current
`manifests/<target>/MicroPythonOS_<target>_<major.minor>.x.json` (older
`<major.minor>.x` manifests stay frozen for "Show old releases") and updates
`index.html`: the three current entries, a block for the previous release at
the top of "Show old releases" when the minor version changes, and the fallback
`esp-web-install-button manifest=`.

- Run `../upload_install.sh` from the parent folder (one level up).
- Verify: `curl -s https://install.micropythonos.com/manifests/esp32s3/MicroPythonOS_esp32s3_<major.minor>.x.json`.

Drop the `--install-only` / `--updates-only` / `--web-only` flag to prepare all
three repos in one go (the flag only decides which one to touch; the release
assets are shared).

**Release the web version**

`../web/` is the working copy served at web.micropythonos.com, and it is
deployed the same way as the other two: rsync the working copy with
`../upload_web.sh`. It has no git remote, so commit it but never pass `--push`
for it.

```
python3 scripts/release_metadata.py <version> --web-only --commit
```

It downloads `MicroPythonOS_web_<version>.zip` and extracts it over the working
copy (`index.html`, `micropython.{html,js,data,wasm}`, `mpos.html`,
`shell.html`), which is exactly what `../upload_web.sh` serves. This used to be
a manual download-and-unpack step, and a release can look complete while the
web build is still the previous one, so `release_metadata.py` now covers it and
`upload_all.sh` runs `upload_web.sh`.

- Run `../upload_web.sh` from the parent folder (one level up). It rsyncs with
  `--delete`, so run it from the working copy that holds the new build.
- Verify the live build is the new one: the `Content-Length` of
  `curl -sI https://web.micropythonos.com/micropython.wasm` matches
  `stat -c %s ../web/micropython.wasm`, and https://web.micropythonos.com
  loads.

**Update the website**

`../website/` is the working copy served at micropythonos.com, and `../web/` is
the browser build served at web.micropythonos.com. Both are plain rsync
deploys through `../upload_site.sh`, which gzips large assets (the server has
`gzip_static on`), refuses a `--comtoorg` rewrite when the `.org` names do not
resolve, and verifies the upload over HTTP.

- Write the news entry for `<version>`: copy `../website/news/0.22.0.html`,
  rewrite it from the new `CHANGELOG.md` section, then add it to the top of
  `../website/news/index.html`, `../website/news/feed.xml` and
  `../website/sitemap.xml`.
- Update the release strip on the home page (the `release-badge` link in
  `../website/index.html`) and its three "What's new" cards.
- Refresh the screenshot grid every few releases: the clips are captured from
  a running desktop build with `MPOSController` (see the lab tooling table in
  `AGENTS.md`), so the posters on the site match what users actually see.
- Run `../upload_site.sh website`.
- Run `../upload_site.sh web` after `release_metadata.py --web-only`.

**Bundle and publish apps**

See [Publishing Apps](../apps/publishing-apps.md) for the full flow. For a full
release (every app), check the diff before committing: the full bundle
regenerates the whole index and can revert index-only fixes (such as a publisher
change) and rebuild apps whose external sources are dirty.

```
make check-app-versions
./scripts/bundle_apps.sh
cd ../apps/
git add app_index.json apps/
git commit -a
git push
cd ../badgehub/
./synchronize_apps.py     # not synchronize_badgehub.py
cd ..
./upload_apps.sh
```

** Announce the release **

- On the community https://chat.MicroPythonOS.com
- In the LightningPiggy chat
- In the Fri3d Camp chat
- On Nostr
- On Twitter
- In the Press

## Notes

- Ensure all repositories are pushed before tagging.
- Verify builds on target hardware.
- Release artifacts are built from `main`; `release.yml` only picks up build
  workflows whose `head_sha` matches the tag commit, so tag a commit that was
  pushed to `main` and whose builds have finished.
