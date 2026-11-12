# MicroPythonOS docs

Live site: https://docs.micropythonos.com/

## Building and publishing

The site is built with MkDocs Material and uploaded to the web server; it is
**not** served from GitHub Pages.

```bash
# one-time, or whenever requirements.txt changes
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt

.venv/bin/mkdocs build          # writes site/
../upload_docs.sh               # rsync site/ to docs.micropythonos.com
```

`upload_docs.sh` lives in the parent directory (`MicroPythonOS/claude/`) and
mirrors `site/` onto the server with `--delete`.

Dependencies are pinned in `requirements.txt` on purpose: the theme's asset
filenames are content-hashed, so building with a different MkDocs Material
version replaces every hashed bundle. Browsers that still have a page cached
then ask for a bundle that no longer exists and the page loses its JavaScript
(search stays at "initializing search"). Rebuild with the pinned versions.

## Local preview

```bash
.venv/bin/mkdocs serve
```
