# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A monorepo of ~30 independent plugins for [calibre](https://calibre-ebook.com/) (kiwidude's plugins). Each top-level folder (e.g. `extract_isbn/`, `goodreads/`, `manage_series/`) is a **separate, independently-versioned and independently-released plugin**. There is no shared application — plugins do not import each other. The `common/` folder is the only shared code (see below).

Plugins are one of two calibre types (see `README.md` table):
- **UI / Interface Action** plugins — add toolbar/menu actions to the calibre GUI. `__init__.py` subclasses `InterfaceActionBase` and points `actual_plugin` at an `action.py:SomeAction` (subclass of `InterfaceAction`).
- **Metadata Source** plugins — download metadata/covers. `__init__.py` subclasses `calibre.ebooks.metadata.sources.base.Source` directly (e.g. `goodreads/__init__.py`).

## Per-plugin file conventions

- `__init__.py` — plugin class; holds `name`, `version = (major, minor, build)`, `minimum_calibre_version`. The build derives the zip filename and release version by regex-parsing `name` and `version` from this file, so keep them as simple literals.
- `action.py` — UI plugin entry point (`genesis()`, action wiring). `config.py` — config widget + `plugin_prefs = JSONConfig('plugins/<Plugin Name>')` with `STORE_NAME` / `KEY_*` constants and a `DEFAULT_STORE_VALUES` dict. Other modules (`jobs.py`, `dialogs.py`, `worker.py`, etc.) vary per plugin.
- `plugin-import-name-<folder>.txt` — empty marker file required by calibre so the plugin is importable as `calibre_plugins.<folder>`. Imports inside a plugin use the fully-qualified form `from calibre_plugins.<folder>.module import ...`.
- `CHANGELOG.md` — must follow [Common Changelog](https://common-changelog.org) format; release automation extracts the section matching the current version. Bump `version` in `__init__.py` AND add a changelog entry together.
- `translations/*.po` + `.pot`, `.tx/config` (Transifex), `images/`, `.docs/`, `.build/` (batch files).

## Common code (`common/`)

`common_*.py` files (`common_compatibility`, `common_dialogs`, `common_icons`, `common_menus`, `common_widgets`) are the single source of shared utilities. **They are not committed into each plugin folder** — `build.cmd` copies them in at zip time and deletes them after. Therefore:
- In plugin source, import them as `from common_icons import get_icon` (bare). `common/build.py` rewrites `from common_` → `from calibre_plugins.<plugin>.common_` when packaging.
- Edit shared behavior once in `common/`; it propagates to every plugin on next build.
- `common_compatibility.py` shows the house style for Qt version shims.

## Build / develop / release

Tooling is **Windows batch files** (`.cmd`) in each plugin's `.build/` folder, run from within that folder. There are no Linux equivalents in-repo (the maintainer notes this in `CONTRIBUTING.md`). Core logic lives in cross-platform Python (`common/build.py`, `common/release.py`) invoked by the batch files.

| Command (from `<plugin>/.build/`) | Does |
| --- | --- |
| `build.cmd` | Compile `.po`→`.mo`, copy `common_*.py` in, run `common/build.py` to make the **uncompressed** zip (`ZIP_STORED`, for fast calibre load), then `calibre-customize -a` to install it. |
| `debug.cmd` | `build.cmd` + launch `calibre-debug -g`. **Primary dev loop** — shows plugin load errors and lets you test interactively. |
| `generate-pot.cmd` | Regenerate the `.pot` via `pygettext`. Run after adding/changing any `_()` string. The list of scanned `.py` files is hardcoded in this file — update it when adding a new translatable source file. |
| `transifex-pull.cmd` | `tx pull` latest translations. |
| `release.cmd` | `build.cmd` + `common/release.py` to tag + create a GitHub release with the zip (maintainers only; needs `CALIBRE_GITHUB_TOKEN`). Release tags are prefixed per-plugin, e.g. `extract_isbn-v1.2.3`. |

VS Code: `.vscode/tasks.json` exposes these batch files as tasks (default shortcut **Alt+Shift+F10**).

There is **no test suite, linter, or CI** in this repo. Verification is manual via `debug.cmd` and interactive testing inside calibre. `__init__.py` files include a `calibre-debug -e __init__.py` self-test block for config widgets.

## Compatibility constraints (important)

- Target **calibre v2.0+ / PyQt5+**. PyQt4 support is dropped.
- Always import Qt via the shim, never PyQt5 directly:
  ```python
  try:
      from qt.core import QWidget
  except ImportError:
      from PyQt5.Qt import QWidget
  ```
- Code must run on both Python 2 and 3 — files start with `from __future__ import unicode_literals, division, absolute_import, print_function`, use `six` for compat (`from six import text_type as unicode`), and guard stdlib moves (`urllib`/`urllib.parse`, `queue`/`Queue`).
- Wrap `load_translations()` at module top in `try/except NameError` (it only exists in calibre 1.9+).

## Working on a plugin

Each plugin is self-contained — scope changes to its folder (plus `common/` if shared). When committing per `CONTRIBUTING.md`: bump `version` in `__init__.py`, update `CHANGELOG.md`, strip debug logging, regenerate `.pot` if strings changed, and rebuild. Single `main` branch.
