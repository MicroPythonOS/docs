Before merging a pull request, we should consider the following:

Making sure to update related things:

- has [CHANGELOG.md](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/CHANGELOG.md) been updated?
- has [MAINTAINERS.md](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/MAINTAINERS.md) been updated?
- has the app's version number (META-INF/MANIFEST.JSON) been incremented?
- has the [documentation](https://GitHub.com/MicroPythonOS/docs) been updated?

Making sure the logic is sound:

- if it changes a setting definition, will old settings be properly migrated to the new setting?
- has a unit test been added or updated to make catch future regressions?

Making sure it's easy to review:

- are whitespace changes done in a separate commit (or ideally, separate pull request) from functional changes?

