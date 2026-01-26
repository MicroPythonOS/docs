# Release Checklist

Follow these steps to create a new release of MicroPythonOS.

**Update Version Numbers**:

   - Increment `CURRENT_OS_VERSION` in `internal_filesystem/lib/mpos/info.py`.
   - Update version numbers for modified apps:

```
git diff --stat 0.6.0 internal_filesystem/  # Check changes since last release, make sure each app change is accompanied by a META-INF/MANIFEST.json change
```

**Update Changelog**:

- Compare MicroPythonOS/CHANGELOG.md to the "git log" or "git log -p" or "git diff 0.6.0" to see if anything is missing since the last release tag
- Document changes in `CHANGELOG.md`
- Run `./scripts/changelog_to_json.sh` to make sure the CHANGELOG.md is json-friendly

**Commit and push** all changes, also in external repositories (e.g., [LightningPiggy](https://github.com/LightningPiggy/LightningPiggyApp)).

This will trigger the GitHub builds at https://github.com/MicroPythonOS/MicroPythonOS/actions

**Download the builds**

When finished, download and extract these builds as artifacts from the [GitHub actions](https://github.com/MicroPythonOS/MicroPythonOS/actions):

- `MicroPythonOS_esp32_0.6.0.ota.zip`
- `MicroPythonOS_esp32_0.6.0.bin.zip`
- `MicroPythonOS_amd64_linux_0.6.0.elf.zip`
- `MicroPythonOS_amd64_macOS_0.6.0.bin.zip`

**Release to Over-The-Air update**

- Copy `MicroPythonOS_esp32_0.6.0.ota` to the [updates repository](https://github.com/MicroPythonOS/updates) 
- Update the `osupdate*.json` metadata files with the new file and the output from `./scripts/changelog_to_json.sh`

**Release to the web installer**

- Copy `MicroPythonOS_esp32_0.6.0.bin` file to the [web installer](https://github.com/MicroPythonOS/install)
- Update the [manifest.json metadata file](https://github.com/MicroPythonOS/install/blob/master/manifests/esp32/MicroPythonOS_esp32_0.6.x.json)
- Update `index.html` if necessary (for example, if you added a new metadata.json you need to update the 2 references)

**Release to GitHub**

- Upload the builds a [new GitHub release](https://github.com/MicroPythonOS/MicroPythonOS/releases/new)
- Fill in the new tag (e.g. 0.6.0)
- Copy-paste the list from `CHANGELOG.md` in it
- Add the "What to do" section at the bottom

**Bundle and publish apps**:

```
./scripts/bundle_apps.sh
pushd ../apps/
git add apps/
git commit -a
git push
```

** Announce the release **

- On the community https://chat.MicroPythonOS.com
- In the LightningPiggy chat
- On Nostr
- On Twitter
- In the Press

## Notes

- Ensure all repositories are pushed before tagging.
- Verify builds on target hardware
