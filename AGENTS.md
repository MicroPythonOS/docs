# AGENTS.md

This file provides guidance to agents when working with documentation in this repository.

## Where the docs live

This repository (`../docs/` relative to the MicroPythonOS code repo) is the public documentation site for MicroPythonOS. It is built with MkDocs.

The source Markdown files are under `docs/`. `mkdocs.yml` at the repository root drives navigation and includes.

## Included files

Some Markdown files are intentionally included by other pages rather than listed directly in `mkdocs.yml`. Examples in `docs/os-development/`:

- `compiling.md` is included by `linux.md` and `macos.md`.
- `running-on-desktop.md` is included by `linux.md` and `macos.md`.

This is why `mkdocs build` warns "The following pages exist ... but are not included in the nav" for those files. Do not add them to `nav` unless explicitly requested.
