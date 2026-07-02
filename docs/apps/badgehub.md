# BadgeHub.eu Apps

[BadgeHub.eu](https://badgehub.eu) is a community appstore for event badges and MicroPythonOS devices. MicroPythonOS ships with a BadgeHub backend in the AppStore app, so users can browse, install, and update BadgeHub-published apps directly on their device.

## What is BadgeHub?

BadgeHub hosts `.mpk` packages and exposes a JSON API that MicroPythonOS queries for app listings, project details, and downloadable releases. It is particularly useful for:

- conference or camp badge apps
- community-built utilities
- distributing apps without going through the curated MicroPythonOS app index

## Backend endpoints

The BadgeHub backend uses API version 3:

- `https://badgehub.eu/api/v3/project-summaries` — list of all published projects with summary metadata.
- `https://badgehub.eu/api/v3/projects/<slug>` — full project details, including releases, icons, and descriptions.

Each project maps to one MicroPythonOS app package. The project's slug becomes the app identifier in the store.

## Project metadata

A project detail response contains fields such as:

- `name` — display name in the AppStore.
- `description` / `long_description` — short and long descriptions.
- `icon_url` — URL to a 64×64 or larger app icon.
- `publisher` — author or organization.
- `releases` — list of published releases, each with a version and a download URL.
- `main_executable` — optional flag indicating the primary app executable for this project.

The AppManager uses the latest release's download URL to fetch the `.mpk`.

## Releasing an app on BadgeHub

1. Build your `.mpk` following the [Bundling Apps](bundling-apps.md) guide. The top-level folder in the ZIP must match the app's `fullname` exactly.
2. Ensure the archive contains a valid `MANIFEST.JSON`.
3. Upload the `.mpk` to your BadgeHub project.
4. MicroPythonOS will detect the new release the next time the AppStore refreshes the BadgeHub backend.

## Version handling

Releases on BadgeHub should use [semantic versioning](https://semver.org/) strings (for example, `1.2.3`). The AppStore compares the installed version against the latest BadgeHub release and offers an update when the BadgeHub version is newer.

## Switching backends in the AppStore app

The AppStore app can switch between the curated MicroPythonOS backend and the BadgeHub backend. On supported firmware builds, BadgeHub is the default backend. Tap the backend selector in the AppStore UI to switch sources.

## See also

- [Bundling Apps](bundling-apps.md) — creating `.mpk` packages
- [Creating Apps](creating-apps.md) — writing an app from scratch
- [App Lifecycle](app-lifecycle.md) — `Activity` and `Service` lifecycle
