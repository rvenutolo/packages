# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A data repo, not an application. It holds CSV manifests of which software packages get installed on which of the
author's machines. There is no build, no test suite, and no runtime here — the CSVs are consumed remotely over HTTPS by
the `rvenutolo/scripts` repo (`scripts/functions/packages.bash`), which fetches them from
`https://raw.githubusercontent.com/rvenutolo/packages/main/<file>.csv` and pipes them through `awk`. That consumer is
the real spec: an edit that breaks column count or column order silently breaks package installation on every host.

## The CSV files

| File                   | Columns                                                       | Sorted by             |
| ---------------------- | ------------------------------------------------------------- | --------------------- |
| `universal.csv`        | `NAME,TYPE,PACKAGE,PERSONAL DESKTOP,PERSONAL LAPTOP,WORK LAPTOP,SERVER,DISABLED` | `NAME`, case-insensitive |
| `sdkman.csv`           | `NAME,PACKAGE,<4 host columns>,DISABLED`                       | `NAME`                |
| `<id>-<codename>.csv`  | `PACKAGE,<4 host columns>,DISABLED`                            | `PACKAGE`             |

Distro files are named after the OS id and codename (`debian-bookworm.csv`, `ubuntu-noble.csv`). The consumer builds the
URL from `${id}-${codename}.csv` at runtime, so adding support for a new release means adding a file with that exact
name.

`TYPE` in `universal.csv` is one of `nixpkgs`, `nixpkgs-unstable`, `flatpak`, `appimage`. `PACKAGE` is the installer's
own identifier — a nixpkgs attribute path (`bat-extras.batgrep`, `_7zz`), a flatpak app id (`org.kde.haruna`), or an
appimage name.

### Host columns

The four host columns are positional and the consumer hardcodes their indexes (4/5/6/7 in `universal.csv`). `y` means
install on that host; empty means skip. Never reorder or insert columns.

### DISABLED column

Non-empty = the package is skipped, and the string is the reason, printed to the user at install time. So a "note"
cannot be parked there casually — any text disables the row. Existing reasons follow the pattern
`use distro package - <why>` or a dated note like `buggy - ... (2026-05-14)`. To keep a package enabled, leave the field
empty (every row therefore ends in a trailing comma).

## Checks

Both scripts here are not standalone: they `source "${SCRIPTS_DIR}/functions.bash"` from the `rvenutolo/scripts` repo
and only run in an environment where `SCRIPTS_DIR` is exported.

- `./check-csv-columns` — verifies every row of every CSV has the same column count as its header. Run this after any
  bulk edit; a stray comma inside a `DISABLED` reason is the usual way this breaks.
- `./list-universal-types` — prints the distinct `TYPE` values in `universal.csv`.

## Editing conventions

- Adding a package means inserting one row in sort position, not appending. Verify the package really exists in the
  relevant source before adding it (e.g. `nix eval --raw github:NixOS/nixpkgs/nixpkgs-unstable#<attr>.meta.description`
  for `nixpkgs-unstable`).
- Server hosts are headless: GUI apps, flatpaks, and appimages leave `SERVER` empty.
- Commits are Angular-style and typically one logical package change each, e.g. `feat: add lm-sensors for all hosts`,
  `feat: enable qsv and xan on server`.
- `TODO` is a plain scratch list of packages to look into.
