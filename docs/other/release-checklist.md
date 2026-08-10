# Release Checklist

Follow these steps to create a new release of MicroPythonOS.

**Update Version Numbers**:

   - Increment `CURRENT_OS_VERSION` in `internal_filesystem/lib/mpos/build_info.py`
   - Update version numbers for modified apps:

```
git diff --stat 0.6.0 internal_filesystem/  # Check changes since last release, make sure each app change is accompanied by a MANIFEST.json change
```

**Update Changelog**:

- Compare MicroPythonOS/CHANGELOG.md to the "git log" or "git log -p" or "git diff 0.6.0" to see if anything is missing since the last release tag
- Document changes in `CHANGELOG.md`
- Run `./scripts/changelog_to_json.sh` to make sure the CHANGELOG.md is json-friendly

**Commit and push** all changes, also in external repositories (e.g., [LightningPiggy](https://github.com/LightningPiggy/LightningPiggyApp)).

This will trigger the GitHub builds at https://github.com/MicroPythonOS/MicroPythonOS/actions

**Tag the commit**

When finished, check to make sure the tests were green.

Then, tag the freshly built commit with something like:

`git tag 0.17.0 64b8597713`

In the example above, 0.17.0 is the CURRENT_OS_VERSION and 64b8597713 is the commit that was built.

**Push the tag**

Run:

`git push --tags`

This will trigger the github "release.yml" workflow, which will create a [draft release](https://github.com/MicroPythonOS/MicroPythonOS/releases).

**Test the draft release**

Make sure it's all good to go.

**Publish the draft release**

Edit the draft release on GitHub.

- Copy-paste the list from `CHANGELOG.md` into it
- Click "Publish"

**Release to Over-The-Air update**

- Copy `MicroPythonOS_esp32_0.6.0.ota` to the [updates repository](https://github.com/MicroPythonOS/updates) 
- Update the `osupdate*.json` metadata files with the new file and the output from `./scripts/changelog_to_json.sh`
- Commit and push
- Run ../upload_updates.sh

**Release to the web installer**

- Copy `MicroPythonOS_esp32_0.6.0.bin` file to the [web installer](https://github.com/MicroPythonOS/install)
- Update the [manifest.json metadata file](https://github.com/MicroPythonOS/install/blob/master/manifests/esp32/MicroPythonOS_esp32_0.6.x.json)
- Update `index.html` if necessary (for example, if you added a new metadata.json you need to update the 2 references)
- Commit and push
- Run ../upload_install.sh

**Release web version**

- Download the web release .zip from github
- Extract it to ../web/
- Run ../upload_web.sh

**Bundle and publish apps**:

```
./scripts/bundle_apps.sh
cd ../apps/
git add apps/
git commit -a
git push
cd ../badgehub/
./synchronize_badgehub.py
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
- Verify builds on target hardware
