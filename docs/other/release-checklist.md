# Release Checklist

Follow these steps to create a new release of MicroPythonOS.

**Update Changelog**:

Document changes in `CHANGELOG.md`.

**Update Version Numbers**:

   - Increment `CURRENT_OS_VERSION` in `internal_filesystem/lib/mpos/info.py`.
   - Update version numbers for modified apps:

```
git diff --stat 0.0.4 internal_filesystem/  # Check changes since last release
git diff 0.0.4 -- internal_filesystem/apps/*/META-INF/*  # Check app manifests
git diff 0.0.4 -- internal_filesystem/builtin/apps/*/META-INF/*  # Check built-in app manifests
```

**Commit and push** all changes, also in external repositories (e.g., [LightningPiggy](https://github.com/LightningPiggy/LightningPiggyApp)):

**Bundle and publish apps**:

```
./scripts/bundle_apps.sh
pushd ../apps/
git add apps/
git commit -a
git push
```

**Build for all supported devices**

```
./scripts/build_all.sh
```

**Release to GitHub**

- Upload ``` ../build_outputs/ ``` to a [new GitHub release](https://github.com/MicroPythonOS/MicroPythonOS/releases/new)
- Add the CHANGELOG.md
- Tag the code with the new release

**Copy the builds to the [install](https://github.com/MicroPythonOS/install) and [updates](https://github.com/MicroPythonOS/updates) repositories**:

This is a manual action, but check out these old scripts for inspiration:
```
scripts/release_to_updates.sh
scripts/release_to_install.sh
```

## Notes

- Ensure all repositories are pushed before tagging.
- Verify builds on target hardware (see [Building for ESP32](esp32.md)).
