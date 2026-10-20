# Upstream patch inventory — provenance and hashes

The experiment modified two pinned upstream checkouts. This file records **what** changed and
proves the identity of each patch file. It deliberately does **not** contain the patches: their
content is retained upstream source under GPL-2.0-or-later / LGPL-2.1-or-later, and committing or
publishing it needs the owner's licence decision, which has not been made.

**Consequence, stated plainly: this directory alone cannot reproduce the patched build.** The patch
files and the patched checkouts exist only in the recovery workspace
(`C:\Users\choicky\worktmp\fusion-windows\official-settings-ui\patches\` and `spike/upstream/`).
What follows is an inventory with hashes, not a reviewable patch.

## 1. fcitx5-qt — `0285a5d18367d8f3af80dab7e4819d5555982340` (5.1.16)

Patch: `fcitx5-qt-0285a5d-windows-spike.patch`, 1601 bytes,
SHA-256 `13f9caf456f66278368b549095db576c626836db047ea16105cb87a39059ea03`.

| File | Change | Diffstat |
| --- | --- | --- |
| `qt6/widgetsaddons/fcitxqtconfiguifactory.cpp` | MSVC portability: `std::filesystem::path` → `QString` conversion helper at the two call sites (`QLibrary::isLibrary`, `new QPluginLoader`) | +14 / −2 |

Rationale: `std::filesystem::path` converts implicitly to `std::string` only where
`path::string_type` is `std::string` (POSIX); on Windows it is `std::wstring` and the code does not
compile. This is the only fcitx5-qt change, and it is plausibly upstreamable as a portable fix.

## 2. fcitx5-configtool — `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b` (5.1.16)

Patch: `fcitx5-configtool-9bd6022-windows-spike.patch`, 29534 bytes,
SHA-256 `19c606063857a1b9ada41510c718dcd701fde5f97b14a4a03ab6cc3e7a01c93b`.

| File | Change | Diffstat |
| --- | --- | --- |
| `src/lib/configlib/dbusprovider.h` | transport seam: `PendingCall`, `ControllerProxy` (`GetConfig`/`SetConfig`/`CanRestart`), pipe-backed `DBusProvider` | +90 / − |
| `src/lib/configlib/dbusprovider.cpp` | seam implementation: named-pipe framing, request dispatch, reply assembly | +227 / − |
| `src/lib/configlib/launchexternal.h` | **new** — declaration of `launchExternalConfig` (upstream declares it in the excluded `addonmodel.h`) | +23 |
| `src/lib/configlib/launchexternal.cpp` | **new** — the function body extracted verbatim from `addonmodel.cpp`, plus the same MSVC `fs::path` fix | +90 |
| `src/lib/configlib/CMakeLists.txt` | subset source list, `config.h` generation, include path | +32 / − |
| `src/lib/configwidgetslib/configwidget.cpp` | call sites adapted to `PendingCall`; reply unpacking | +39 / − |
| `src/lib/configwidgetslib/configwidget.h` | `PendingCall` forward declaration in namespace; new slot | +4 / − |
| `src/lib/configwidgetslib/fontbutton.cpp` | `KFontChooser` → `QFontDialog` | +25 / − |
| `src/lib/configwidgetslib/optionwidget.cpp` | `KColorButton` → `QToolButton` + `QColorDialog` swatch | +44 / − |
| `src/lib/configwidgetslib/CMakeLists.txt` | subset source list (no KF6, no layoutlib) | +13 / − |

Totals: 10 files, +483 / −104 (including the two new files, which `git diff` counts once they are
intent-to-add).

## Retained vs excluded sources (configtool)

- **Retained** — `configlib`: `dbusprovider`, `launchexternal`, `logging`, `font`.
  `configwidgetslib`: `configwidget`, `categoryhelper`, `erroroverlay`, `fontbutton`,
  `keylistwidget`, `listoptionwidget`, `optionwidget`, `optionaloptionwidget`, `varianthelper`,
  `verticalscrollarea`.
- **Excluded with their features** — `model`/`addonmodel`/`addonselector` (addon-list page, KF6
  delegate), `impage` (input-method selection), `layoutselector` + `layout/` (X11/XKB),
  `imconfig`/`layoutmodel`/`layoutprovider`, `iso639` (three JSON data files), `po`,
  `src/configtool` shell (`mainwindow`, `main`, KCM, migrator).

The experimental window is the **`ConfigWidget` only** — the schema-driven option renderer — not the
complete `configtool` application: no addon list, no input-method page, no main window chrome.

## Provenance

Both checkouts were copied from the pinned revisions recorded in the feasibility document and
patched locally; neither is a fork on any hosting service, and no upstream contribution was opened.
Patch files were generated with `git diff` (plus intent-to-add for the two new files) in each
checkout at the recorded revision, so applying either patch to a clean checkout of the stated
revision reproduces the experiment's upstream state.
