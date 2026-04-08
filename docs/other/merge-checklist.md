Before merging a pull request, we should consider the following:

Making sure to update related things:

- does the "Future release (next version)" section at the top of [CHANGELOG.md](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/CHANGELOG.md) need to be expanded?
- does an App's version number (META-INF/MANIFEST.JSON) need incrementing? Normally only when you modify an App.
- does the [documentation](https://GitHub.com/MicroPythonOS/docs) need updating? Usually when you modify or add a Framework, but can also be for other things. Always good!
- does [MAINTAINERS.md](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/MAINTAINERS.md) need to be updated? Usually only when adding a new board that you'll be maintaining.

Making sure the logic is sound:

- if it changes a Setting's definition, will old settings be properly migrated to the new setting?
- did you add or expand one or more unit tests to make sure your code works, in all edge cases, and future regressions are caught?

Making sure it's easy to review:

- are non-functional changes that result in a big diff done in a separate commit (or ideally, separate pull request) from functional changes?

