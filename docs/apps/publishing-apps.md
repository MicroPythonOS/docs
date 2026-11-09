# Publishing Apps

MicroPythonOS apps are distributed through two independent channels:

- the **curated index** at [apps.micropythonos.com](https://apps.micropythonos.com),
  served from the [MicroPythonOS/apps](https://github.com/MicroPythonOS/apps)
  repository, which is the AppStore's default backend;
- **BadgeHub.eu**, a community app store that hosts the same `.mpk` packages.

Both channels identify a release by the `version` field in `MANIFEST.JSON`, so a
new package is only offered to users when that field changes. Everything else on
this page follows from that rule.

## 1. Bump the app version

Edit `internal_filesystem/apps/<fullname>/MANIFEST.JSON` and raise `version`
(for example `0.5.0` → `0.5.1` for a fix, `0.6.0` for a feature). Use
[semantic versioning](https://semver.org/): users compare version numbers, not
dates or file hashes.

Check before you bundle:

```sh
make check-app-versions                       # every bundled app
make check-app-versions ARGS='--changed'      # only apps changed since origin/main
```

The check lists commits that touched an app after its last version change, plus
uncommitted edits. An app in that list would be rebuilt but never offered as an
update. It is also run by CI on pull requests that touch
`internal_filesystem/apps/`.

Apps that live in another repository (see
[External apps](#external-apps)) are reported as skipped — check them in their
own repository.

## 2. Build the package

```sh
./scripts/bundle_apps.sh --app com.example.myapp
```

This creates `../apps/apps/<fullname>/mpks/<fullname>_<version>.mpk` and the
matching icon, and updates only that app's entry in
`../apps/app_index.json`. Other entries are left byte-for-byte untouched, so a
single-app release cannot regenerate (and accidentally revert) unrelated index
fixes.

Without `--app` the script rebuilds **every** app and regenerates the whole
index. Use that only when you intend to release everything, and read
`git diff` in `../apps` before committing — it can revert index-only fixes and
rebuild apps whose external source checkout is dirty.

The package layout is documented in [Bundling Apps](bundling-apps.md). To check
what ended up inside a package:

```sh
unzip -l ../apps/apps/com.example.myapp/mpks/com.example.myapp_0.5.1.mpk
unzip -p ../apps/apps/com.example.myapp/mpks/com.example.myapp_0.5.1.mpk \
  com.example.myapp/MANIFEST.JSON
```

The script also warns when an app has uncommitted changes; such a package does
not correspond to any commit, so commit first.

## 3. Publish the curated index

```sh
cd ../apps
git add app_index.json apps/
git commit -m "Release 0.5.1: Music Player mp3 gating"
git push
cd ..
./upload_apps.sh
```

`upload_apps.sh` rsyncs `../apps` to the web server. Pushing to GitHub alone does
not update `apps.micropythonos.com`.

Verify:

```sh
curl -s https://apps.micropythonos.com/app_index.json \
  | python3 -c "import json,sys; print([a['version'] for a in json.load(sys.stdin) if a['fullname']=='com.example.myapp'])"
```

## 4. Publish on BadgeHub

BadgeHub uploads use a per-app API token from `../badgehub/apps.csv`
(`packagename,api_token`). Preview, then publish:

```sh
cd ../badgehub
./synchronize_apps.py --app com.example.myapp --dry-run
./synchronize_apps.py --app com.example.myapp
```

The script reads the latest package from `../apps/apps/<fullname>/mpks/`,
updates the project metadata and icon, uploads the `.mpk`, and publishes a new
revision. Apps whose metadata and package are already in sync are skipped, and
`--app` restricts the run to the apps you name.

Uploading the `.mpk` through the BadgeHub web UI (**Upload release** on
`https://badgehub.eu/page/project/<fullname>`) is equivalent for a single app,
but the script is reproducible and updates the icon and metadata in the same
step. See [BadgeHub Apps](badgehub.md) for the project setup.

Verify:

```sh
curl -s https://badgehub.eu/api/v3/projects/com.example.myapp \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['version']['revision'], d['name'])"
```

## Releasing several apps

Repeat `--app` for each app in both tools:

```sh
./scripts/bundle_apps.sh --app com.example.one --app com.example.two
# commit/push ../apps, upload_apps.sh
cd ../badgehub && ./synchronize_apps.py --app com.example.one --app com.example.two
```

## External apps

Some app directories under `internal_filesystem/apps/` are symlinks into
separate repositories (`../external_apps/...`). Their `MANIFEST.JSON` version
lives in that repository, so:

- commit and push the version bump **there** first — the bundle only reads the
  working tree;
- a dirty external checkout produces a package that matches no commit and no
  version, and `bundle_apps.sh` will warn about it;
- `make check-app-versions` skips them; run the same `git log` check in the app
  repository.

## Checklist

- [ ] `MANIFEST.JSON` version bumped, `make check-app-versions` clean
- [ ] app changes committed (no dirty working tree for that app)
- [ ] `./scripts/bundle_apps.sh --app <fullname>` run
- [ ] package contents and version verified with `unzip -p`
- [ ] `../apps` committed, pushed and uploaded with `upload_apps.sh`
- [ ] `synchronize_apps.py --app <fullname>` run
- [ ] both APIs report the new version
- [ ] existing install offers the update in the AppStore

## See also

- [Bundling Apps](bundling-apps.md) — the `.mpk` archive layout
- [BadgeHub Apps](badgehub.md) — project setup on BadgeHub.eu
- [Creating Apps](creating-apps.md) — writing an app from scratch
