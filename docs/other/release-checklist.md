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
in `osupdate*.json`. Most board files are **symlinks** to the three real files
`osupdate_esp32.json`, `osupdate_esp32s3.json` and `osupdate_esp32p4.json`;
edit the real files (and any regular board file that is not a symlink).

- Copy the new `.ota` files from the release into `updates/releases/<target>/`
  (`esp32`, `esp32s3`, `esp32p4`).
- In the real JSON files, set `version`, `download_url` and `changelog`. Use
  the escaped output of `scripts/changelog_to_json.sh` for the new section;
  it uses single quotes and `\n` escapes.
- Commit and push the updates repo.
- Run `../upload_updates.sh` from the `../claude/` folder. It rsyncs with
  `--delete`, so run it from the working copy that has all previous `.ota`
  files.
- Verify: `curl -s https://updates.micropythonos.com/osupdate_esp32s3.json`.

**Release to the web installer**

- Copy the new `.bin` files into `install/firmware_images/<target>/`.
- Add `install/manifests/<target>/MicroPythonOS_<target>_<version>.x.json`
  (copy the previous manifest and update `version` and the `path`).
- In `index.html`, update:
  - the three current-version radio entries,
  - the new entries at the top of "Show old releases",
  - the fallback `esp-web-install-button manifest=`.
- Commit and push, then run `../upload_install.sh` from `../claude/`.
- Verify: `curl -s https://install.micropythonos.com/manifests/esp32s3/MicroPythonOS_esp32s3_<version>.x.json`.

**Release web version**

- Download `MicroPythonOS_web_<version>.zip` from the release and extract it
  into `../web/`.
- Commit the web repo and run `../upload_web.sh` from `../claude/`.
- Verify: https://web.micropythonos.com

**Bundle and publish apps**

`bundle_apps.sh` regenerates the whole index from the local manifests, so check
the diff before committing: it can revert index-only fixes (such as a publisher
change) and rebuild apps whose external sources are dirty.

```
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
