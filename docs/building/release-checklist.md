# Release Checklist

Follow these steps to create a new release of MicroPythonOS.

1. **Update Version Numbers**:
   - Increment `CURRENT_OS_VERSION` in `internal_filesystem/lib/mpos/info.py`.
   - Update version numbers for modified apps:
     ```bash
     git diff --stat 0.0.4 internal_filesystem/  # Check changes since last release
     git diff 0.0.4 -- internal_filesystem/apps/*/META-INF/*  # Check app manifests
     git diff 0.0.4 -- internal_filesystem/builtin/apps/*/META-INF/*  # Check built-in app manifests
     ```

2. **Update Changelog**:
   - Document changes in `CHANGELOG`.

3. **Commit and Tag**:
   - Commit all changes.
   - Tag the main repository and external repositories (e.g., LightningPiggy):
     ```bash
     git tag -a vX.Y.Z -m "Release vX.Y.Z"
     git push --tags
     ```

4. **Build and Release**:
   - Bundle apps:
     ```bash
     ./scripts/bundle_apps.sh
     ```
   - Build for production:
     ```bash
     ./scripts/build_lvgl_micropython.sh esp32 prod
     ```
   - Release to update and install servers:
     ```bash
     ./scripts/release_to_updates.sh
     ./scripts/release_to_install.sh
     ```

## Notes

- Ensure all repositories are synchronized before tagging.
- Verify builds on target hardware (see [Building for ESP32](esp32.md)).
