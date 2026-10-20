# Windows settings UI — upstream reuse feasibility (D085)

Status: **feasibility established over three rounds (2026-10-20). Round 3 — a bounded build-and-transport
spike on this machine — built the retained upstream subset with Qt 6.8.3/MSVC, replaced the transport
at the provider seam, and drove one real round trip through the official widget layer (Pinyin `PageSize`
7 → 9, applied, used by the engine, persistent across an orderly restart). Route A′ is proven for the
configuration surface; product integration has not started. The owner's licence decision is a
prerequisite for introducing retained upstream source into the product; the EXE installer is the next
separate item.** Evidence: `evidence/settings-ui-spike-2026-10-20/` (index: `final-report.md`).
Scope: the settings-UI item of D085. Nothing here changes an accepted decision.

Evidence classes used below: **[measured]** = executed on this machine or read from the pinned
source; **[documented]** = a public API or behaviour taken from its own documentation, not executed
here; **[inferred]** = reasoning from the above, not tested; **[unresolved]** = not established.

Round 1 established what the **unmodified** upstream build requires. Round 2 traced **which
dependency belongs to which feature**, because the unmodified top-level build is not the cost of
every reuse route — that conflation was the error in the first version of this document. Round 3
built the subset, measured the transport seam, and ran the UI round trip on Windows.

## 1. What was asked

D085 schedules an official settings UI that **prefers reusing Fcitx5's own settings UI and
configuration semantics** over a Fusion-specific configuration surface, and states that feasibility
on Windows is **not established**. A Fusion-written UI is a fallback, **not authorized** by that
decision. The batch instruction adds: do not dismiss upstream reuse merely because Qt is absent
locally or the unmodified top-level CMake requires X11.

## 2. Round 1 — the unmodified build cannot be configured on this machine [measured]

Inspected at the revisions that match the pinned runtime generation:

| Component | Revision | Version |
| --- | --- | --- |
| `fcitx5-configtool` | `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b` | 5.1.16 |
| `fcitx5-qt` | `0285a5d18367d8f3af80dab7e4819d5555982340` | 5.1.16 |
| fcitx5 (pinned runtime) | `a7f2528773ebfec98a20c49b6eea2333110b1c88` | 5.1.23 |
| fcitx5-chinese-addons | `61474bd3aa9fca26d1c31df93343035697e9f265` | 5.1.15 |
| libime | `171edcf137001e8eb8274f53ec010058cda70b09` | 1.1.17 |

A real configure with `-DENABLE_KCM=OFF -DENABLE_TEST=OFF` (the most favourable configuration, which
drops the Plasma/Kirigami branch) stops at `CMakeLists.txt:21`:

```
Could not find a package configuration file provided by "Qt6" (requested version 6.4.0)
```

A probe of every declared requirement: **4 of 17 present** (ECM, Fcitx5Core, Fcitx5Config,
Fcitx5Utils); absent are Qt6, the four KF6 modules, `Fcitx5Qt6DBusAddons`,
`Fcitx5Qt6WidgetsAddons`, IsoCodes, X11, XKeyboardConfig, PkgConfig and Gettext, with `xkbfile`
unprobeable because pkg-config itself is absent.

**This is a fact about the unmodified top-level build on this machine.** It is not, by itself, the
cost of adapting the UI, which is what §3–§5 examine.

## 3. Round 2 — where each dependency actually comes from [measured]

### 3.1 The reusable widget layer is Qt-only

`fcitx5-qt/qt6/widgetsaddons/CMakeLists.txt` builds four sources
(`fcitxqtconfiguifactory`, `fcitxqtconfiguiplugin`, `fcitxqtconfiguiwidget`,
`fcitxqtkeysequencewidget`) and links **only** `Fcitx5::Utils`, `Qt6::Core`, `Qt6::Widgets`.
`fcitxqtkeysequencewidget.cpp` includes `QKeyEvent`, `QToolButton`, `QTimer` and `fcitx-utils/key.h`;
`fcitxqtconfiguifactory.cpp` uses `QPluginLoader`. **No D-Bus, no KF6, no X11.** This is the layer
that provides the key-capture widget and the per-addon config-UI plugin interface.

### 3.2 X11 belongs to exactly one feature

`layout/CMakeLists.txt` builds `layoutlib` from **one file**, `keyboardlayoutwidget.cpp`, linking
`X11`, `X11Xkblib` and `PkgConfig::XkbFile`. Its only consumer is
`configwidgetslib/layoutselector.cpp` (`#include "keyboardlayoutwidget.h"`), which is reached from
`impage.cpp:298` and `:360`. That is the keyboard-layout/variant picker for an input method — a
Linux XKB feature with no meaning on this product, whose runtime has no keyboard input method at all
(`-DENABLE_KEYBOARD=OFF`). The unconditional `find_package(X11/XKeyboardConfig/xkbfile)` at
`CMakeLists.txt:82-84` exists to serve that one widget.

### 3.3 KF6 belongs to three option types and one delegate

The retained UI's KF6 includes are five lines:

| File | Include | Purpose |
| --- | --- | --- |
| `configwidgetslib/optionwidget.cpp:17` | `KColorButton` | the `Color` option type (`ColorOptionWidget`, lines 410–470) |
| `configwidgetslib/fontbutton.cpp:9` | `KFontChooser` | the `Font` option type |
| `configwidgetslib/addonselector.cpp:14` | `KWidgetItemDelegate` | the add-all-addons list |
| `configtool/main.cpp:14-15` | `KDBusService`, `KWindowSystem` | optional single-instance, inside `#ifdef CONFIG_QT_UNIQUE_APP_SUPPORT` |

`main.cpp`'s KF6 use is already conditional and only compiled when the KF6 targets exist
(`CMakeLists.txt:42-44`). The pinyin schema contains **no `Color` and no `Font` option**
(`POC.md` §B, full inventory), so two of the three are unreachable for this product.

### 3.4 The D-Bus dependency is one class with eleven call sites

`DBusProvider` (`configlib/dbusprovider.h/.cpp`, 60 lines) is constructed in exactly one place —
`configtool/mainwindow.cpp:30` — and passed to the three pages. It owns an `FcitxQtWatcher` and
creates an `FcitxQtControllerProxy` on `/controller`. Everything else reaches the daemon through
`dbus_->controller()->…`.

Actual proxy calls in the retained UI (not guards): **11**, over 11 distinct methods.

| Purpose | Methods | Calls |
| --- | --- | --- |
| **Settings surface** | `GetConfig`, `SetConfig` | 2 (`configwidget.cpp:71`, `:145`) |
| **Input-method selection** | `InputMethodGroups`, `InputMethodGroupInfo`, `SetInputMethodGroupInfo`, `AvailableInputMethods` | 4 |
| Group management, update, restart (droppable) | `AddInputMethodGroup`, `RemoveInputMethodGroup`, `Refresh`, `Restart`, `CheckUpdate` | 5 |

The call idiom is uniform and shallow: `auto call = dbus_->controller()->X(...)`, then
`new QDBusPendingCallWatcher(call, this)`, then `QDBusPendingReply<T> reply = *watcher`. The proxy
methods are **not virtual** (it is a generated `QDBusAbstractInterface`), so a replacement cannot be
a subclass; it must be a different class exposing the same method names returning a
`QDBusPendingCall`.

### 3.5 The other requirements are small or data-only

- **IsoCodes** is not a library here: `configlib/iso639.cpp` reads three JSON files named by
  `ISOCODES_ISO639_{2,3,5}_JSON` and builds a code→name map. Shipping the JSON (or dropping the
  language-name column) removes it.
- **PkgConfig / Gettext** are build-time tooling; Gettext is only for the translation catalogs.
- **Qt6** is the one requirement that cannot be removed by any adaptation: the widgets *are* Qt
  widgets.

### 3.6 Feature-to-dependency map

| Feature | Sources | Needs |
| --- | --- | --- |
| Option rendering from the schema (the hard part) | `configwidgetslib/{configwidget,optionwidget,listoptionwidget,optionaloptionwidget,varianthelper,keylistwidget,verticalscrollarea,erroroverlay,categoryhelper}.cpp` (~2.1k lines) | Qt6 Widgets, `Fcitx5Qt6WidgetsAddons`, KF6 for `Color`/`Font` only |
| Input-method page (selection) | `configwidgetslib/impage.cpp`, `configlib/imconfig.cpp` | Qt6, the 4 selection methods, `layoutselector` for layouts |
| Addon list page | `configwidgetslib/addonselector.cpp` | Qt6, KF6 `KWidgetItemDelegate`, 2 addon methods |
| Keyboard-layout picker | `layout/keyboardlayoutwidget.cpp`, `layoutselector.cpp` | **X11, Xkblib, xkbfile** |
| KCM (Plasma settings module) | `src/kcm`, `plasmathemegenerator` | Plasma, Kirigami, KCMUtils |
| Migrators | `src/migrator` | Qt only, not part of the product's scope |
| Transport | `configlib/dbusprovider.cpp`, `Fcitx5Qt6DBusAddons` | Qt6 DBus + a daemon-side `/controller` |

## 4. The minimal upstream-UI adaptation (route A′)

**Built and exercised in the Round-3 spike (§5); not integrated into the product.** It retains the
official UI sources and changes the boundary, rather than importing the whole Linux stack or
writing a new UI. The spike's experimental target was the official **`ConfigWidget`** — the
schema-driven option renderer — hosted in a minimal test window. It was **not** the complete
`configtool` application: no addon list, no input-method page, no application shell or main-window
chrome.

1. **Build a subset**: `src/lib/{configlib,configwidgetslib}`, with `layout/`, `src/kcm`,
   `src/migrator`, `plasmathemegenerator` and the `src/configtool` shell excluded. Excluding
   `layout/` removes X11, XKeyboardConfig and xkbfile outright; excluding `kcm` removes
   Plasma/Kirigami/KCMUtils. **[measured, Round 3]** The CMake exclusions alone were **not**
   sufficient: source adjustments were required as well, enumerated in the items below and in
   `evidence/settings-ui-spike-2026-10-20/patch-inventory.md`. Retained: `configlib`
   (`dbusprovider`, `launchexternal`, `logging`, `font`) and `configwidgetslib` (`configwidget`,
   the option widgets, `keylistwidget`, `varianthelper`, `erroroverlay`, `verticalscrollarea`,
   `fontbutton`, `categoryhelper`); excluded with their features: addon list, input-method page,
   keyboard-layout picker, IsoCodes language names, translations, application shell.
2. **Replace the transport at its single seam**: keep `DBusProvider`'s interface (`available()`,
   `canRestart()`, `availabilityChanged`, `controller()`) and implement it over the product's
   per-user named pipe. Two approaches were tried; the order matters for anyone reading the record:
   - *Historical — disproven.* The first attempt returned
     `QDBusPendingCall::fromCompletedCall(QDBusMessage)` from the provider's methods, so that the
     upstream call sites would compile unchanged. The synthesized call and its watcher behaved
     correctly with no session bus at all, and custom `QDBusArgument` types read back through
     `reply().arguments()` — but `QDBusPendingReply` rejects the reply
     (`Unexpected reply signature: got "", expected "v"`), because QtDBus computes the signature
     only during wire marshalling and exposes no setter. This route is kept here as experimental
     evidence; it is not the plan. Details:
     `evidence/settings-ui-spike-2026-10-20/transport-probe.md`.
   - *Current.* The seam is the local asynchronous interface this document had already named as the
     fallback: a `ControllerProxy` whose `GetConfig`/`SetConfig`/`CanRestart` return a local
     `PendingCall` (a QObject emitting `finished()` and carrying the reply message), with the two
     `configwidget.cpp` call sites adapted to it. The GUI thread never blocks on pipe I/O, and the
     engine side schedules every touch onto Fcitx's event loop. This is what drove the round trip
     in §5.
3. **Serve the settings methods; selection stays out of scope.** `GetConfig` and `SetConfig` were
   served from the isolated engine over the pipe in the spike, on the official semantics
   (`Configuration::dumpDescription` for the schema, `save()` for the values,
   `InputMethodEngine::setConfigForInputMethod` for the apply). The four selection methods
   (`InputMethodGroups`, `InputMethodGroupInfo`, `SetInputMethodGroupInfo`,
   `AvailableInputMethods`) were **not** implemented — selection is a separate surface (§6) and
   remains product-integration work.
4. **Drop the layout feature.** `layout/` and `layoutselector.cpp` are excluded with the subset, so
   no `impage.cpp` edit is needed (that file is excluded too).
5. **KF6 reduces to two widgets in the retained subset**: `KColorButton` → `QToolButton` +
   `QColorDialog` (`optionwidget.cpp`, the `Color` option type) and `KFontChooser` → `QFontDialog`
   (`fontbutton.cpp`, retained — not dropped). `addonselector.cpp` is excluded with the addon list,
   and `configtool/main.cpp`'s KF6 use is already inside `#ifdef CONFIG_QT_UNIQUE_APP_SUPPORT`.
6. **IsoCodes is not needed**: `iso639.cpp` is excluded with the keyboard-layout list.

Two further source adjustments were required and are part of the recorded delta: one genuine MSVC
portability fix in `fcitx5-qt` (`std::filesystem::path` → `QString` in `fcitxqtconfiguifactory.cpp`,
where the implicit POSIX conversion does not exist on Windows), and `launchExternalConfig` extracted
verbatim from the excluded `addonmodel.cpp` so the `External` option type keeps its upstream
behaviour. Note that `External` rows render but launch nothing on Windows — no wrapper binaries
exist there — which is upstream's own fallback, not a validated feature.

What stays untouched is the valuable part: the schema-driven option renderer and its supporting
widgets (~3.4k lines across the retained files, transport included), the key-sequence widget, and
the official per-addon config-UI plugin interface.

**Costs and obligations.** Qt6 (Core/Gui/Widgets/DBus) becomes a product dependency — LGPLv3,
dynamically linked, and unavoidable for any upstream-widget route. The UI sources are
**GPL-2.0-or-later** (`src/configtool`, `src/lib`, `src/kcm`), so a fork carrying them puts the
settings executable under GPL-2.0-or-later; neither product repository has a `LICENSE` file, so the
product's own outbound licence has to be settled first. **That owner decision is a prerequisite,
not a later step: no retained upstream source may be introduced into the product before it.** The
fork must be recorded with its upstream revision and licence text, and carried forward on upstream
changes.

## 5. Round 3 — the bounded build-and-transport spike [measured]

Executed 2026-10-20 on machine 7700, entirely inside a task directory
(`…\official-settings-ui\spike\`): task-local Qt 6.8.3 `win64_msvc2022_64` (Qt6DBus included),
ECM 6.10.0, MSVC 14.51.36231 + Windows SDK 10.0.26100.0; the pinned runtime reused, not rebuilt.
No system dependency, PATH, installed input method, production Host or owner profile touched;
every run redirected `APPDATA`/`LOCALAPPDATA`/`USERPROFILE` to a disposable profile.

**Durable evidence:** `evidence/settings-ui-spike-2026-10-20/` — commands and artifact hashes
(`build-and-run.md`), the transport probe (`transport-probe.md`), the full round-trip transcript
(`ui-roundtrip.log`), the no-change comparison (`values-comparison.md`), the upstream delta with
hashes and provenance (`patch-inventory.md`), the window screenshot (`settings-window.png`) and the
index (`final-report.md`). The working tree — sources, binaries, patched checkouts, profiles —
stays in the recovery workspace and is not committed; because the retained upstream patches are
local-only pending the owner's licence decision (§9), that evidence directory alone cannot
reproduce the patched build.

The three uncertainties the two-round record left open (not compiled; the synthesized-reply path;
the custom types) were measured, and the second answered **against** the expectation:

1. **The subset builds and links on Windows.** `Fcitx5Qt6DBusAddons` + `Fcitx5Qt6WidgetsAddons`
   from the pinned fcitx5-qt (a 40-line replacement CMakeLists adding only those two modules; the
   top-level build's X11/XKBCommon/Gettext requirements belong to excluded features), `configlib`
   and `configwidgetslib` from the pinned configtool, and an experimental window — all MSVC, C++20
   (fcitx-utils `key.h` uses `std::ranges`), with an `<unistd.h>` shim. Source adjustments were in
   fact required beyond CMake exclusions, as anticipated: the two KF6 widgets became Qt equivalents
   (`KColorButton` → `QToolButton`+`QColorDialog`, `KFontChooser` → `QFontDialog`), one genuine MSVC
   portability fix in fcitx5-qt (`std::filesystem::path` → `QString`, two call sites), and
   `launchExternalConfig` extracted verbatim out of the excluded `addonmodel.cpp` so the `External`
   option type keeps its upstream behaviour.
2. **The synthesized-reply route is disproven; the local-interface fallback is proven.** With no
   session bus at all: `QDBusPendingCall::fromCompletedCall` succeeds, the
   `QDBusPendingCallWatcher` delivers `finished()`, and custom `QDBusArgument`-marshalled
   structures plus `QDBusVariant` maps read back through `watcher->reply().arguments()` — but
   `QDBusPendingReply` rejects the reply (`Unexpected reply signature: got "", expected "v"`).
   QtDBus computes the signature only during wire marshalling and offers no setter. The seam that
   actually shipped is the local asynchronous interface at the affected call sites (`ControllerProxy`
   + `PendingCall`), and it drives the unmodified renderer.
3. **One real UI round trip through the official widget layer.** The experimental window is the
   genuine upstream `ConfigWidget` (schema-driven renderer, no local re-implementation), over a
   task-local named pipe to an isolated engine process (pinned runtime, six static addons, Fcitx's
   event loop for every engine touch). Load showed the engine's `PageSize` 7; 7 → 9 through the
   widget layer applied via the `setConfigForInputMethod` path; re-read returned 9; the engine's own
   candidate list used 9; 9 persisted across an orderly engine restart; a no-change load/apply left
   the engine value tree and every profile file (config + learning) byte-identical. The official
   schema served over the seam was 43 value keys, 2 config types and 62 renderable options — the
   page **rendered** from that schema; only `PageSize` was driven through a round trip, so no claim
   is made for the other options, for `External` actions, for selection, or for the production Host.
4. **Pinyin/Shuangpin selection remains unimplemented** (§6) — only the configuration surface is
   proven.

### 5.1 The fcitx5-macos reference check: shared semantics, platform-specific GUI [measured]

Fcitx5 on macOS ships its **own** SwiftUI settings UI (`fcitx5-macos` `src/config/*.swift`), not the
Qt configtool — confirming that platform-specific GUIs over shared engine semantics are the
upstream-sanctioned pattern, not a departure from it. Its C++ bridge (`src/config/config.cpp`)
implements **exactly the same configuration semantics** the Linux DBus module exposes, which this
spike's adapter mirrors one-to-one:

- the same three config URIs — `fcitx://config/global`, `fcitx://config/addon/<addon>[/<sub>]`,
  `fcitx://config/inputmethod/<im>` (`config.h:8-10`);
- schema from `Configuration::dumpDescription()`, values from `Configuration::save()` → `RawConfig`;
- the same apply path — `globalConfig().load(cfg, true)` + `safeSave()` + `reloadConfig()`;
  `addon->setConfig()/setSubConfig()`; `engine->setConfigForInputMethod(entry, cfg)`;
- engine access scheduled onto the Fcitx thread's event dispatcher and completed through a
  promise/future (`with_fcitx`, `src/fcitx.h:52-71`) — the same discipline the spike's engine
  service uses.

The differences are serialization and threading only, and none of them changes our design: macOS
merges spec+values into one JSON document over a synchronous C ABI and **blocks the calling thread**
on the future, while our seam keeps schema and values separate (as the upstream Qt renderer expects)
and keeps the GUI thread asynchronous. Nothing from macOS needed porting to Windows; the check
corroborates that the spike reused the right, shared official semantics.

## 6. Selection is not configuration

These are two different surfaces and only one of them has been demonstrated:

- **Configuring the Shuangpin scheme and Pinyin options** is a *configuration* operation. The PoC
  proved the official path works: `dumpDescription` yields the schema (including
  `ShuangpinProfile` with its enum), `setConfig` applies it, the engine uses it, it persists.
- **Selecting Pinyin or Shuangpin** is an *input-method-group* operation
  (`InputMethodGroupInfo`/`SetInputMethodGroupInfo` over the controller; `Items/0` in Fcitx's
  `profile`, per WP3). The runtime does register both input methods
  (`data/fcitx5/inputmethod/{pinyin,shuangpin}.conf`), but **no selection round-trip was performed**,
  in the UI or in the PoC. WP3 validated selection on the product path through the profile file, not
  through this UI seam.

## 7. What the engine PoC does and does not prove

`POC.md` §B is an **isolated, in-process engine PoC**: a harness linked against the pinned runtime,
constructing its own `fcitx::Instance` with the Host's argv on a redirected user profile. It proves
the *engine* can dump the schema, apply a value, use it and persist it.

It does **not** prove a Host IPC path (no `MessageType` was added, `protocol_version` is still 4), and
it is **not** real-application UI validation: no window existed, no application was involved, and the
page size was read from the engine's candidate list inside the harness.

## 8. Routes

| # | Route | Reuses | Status |
| --- | --- | --- | --- |
| A′ | **Minimal upstream-UI adaptation** (§4): subset build + transport seam replaced + layout/KF6 reduced | The official UI sources, schema, apply, storage | **Built and validated by the Round-3 spike for the configuration surface (Pinyin `PageSize` round trip, persistence, no-change safety); ready for the implementation batch once the licence prerequisite (§9) is decided.** The smallest route that satisfies "reuse upstream first" |
| A | Full upstream stack: Qt6 + KF6 + X11 + xkbcommon + IsoCodes + rebuilt runtime with keyboard/dbus modules + a session D-Bus | The official UI unmodified | Works, but imports Linux-only features the product does not need and reopens runtime provenance |
| B | Fusion-authored window over `dumpDescription`/`setConfig` via the existing named pipe | Official schema and apply only | **Not authorized by D085**; also the largest new-code surface for the same user-visible result |
| C | No UI: settings edited in `%APPDATA%\Fcitx5\config\fcitx5\conf\pinyin.conf` with the Host stopped (WP3 B3f) | Nothing new | Correct today, but gives the user no settings surface |

Note on C: the tray does **not** help here. Its menu opens the **log** folder
(`%LOCALAPPDATA%\fusion-host`, `windows-daily-use.md` §4), which contains no configuration; the
configuration lives under `%APPDATA%\Fcitx5`.

## 9. Fork, patch scope, and licences [measured]

The Round-3 upstream deltas are exactly two patch files. Their inventory, hashes and provenance are
committed in `evidence/settings-ui-spike-2026-10-20/patch-inventory.md`; **the patch content itself
is not committed** (see the licence boundary below).

- `fcitx5-qt` `0285a5d`: **1 file** — `fcitxqtconfiguifactory.cpp` MSVC portability
  (`std::filesystem::path` → `QString`). This one is plausibly upstreamable as a portable fix; if
  accepted it removes the need for an fcitx5-qt fork.
- `fcitx5-configtool` `9bd6022`: **10 files** — the transport seam (`dbusprovider.h/.cpp`), the two
  `configwidget` call sites, the two KF6→Qt widget replacements, both subset `CMakeLists.txt`, and
  the new `launchexternal` TU (verbatim extraction from the excluded `addonmodel.cpp`). These
  changes are inherent to the Windows port (upstream assumes D-Bus + KF6) and **constitute a
  maintained fork or patch series**; no GitHub fork was created in the spike.

Licences of the retained sources/dependencies: configtool `src/lib` **GPL-2.0-or-later**;
fcitx5-qt widgetsaddons LGPL-2.1-or-later, dbusaddons BSD-3-Clause; Qt 6.8.3 LGPL-3/GPL-2-or-later
(dynamic link); ECM BSD-3-Clause (build-time only). Neither product repository's outbound licence
was chosen or changed.

**Licence boundary.** The owner's licence decision is an explicit **prerequisite**: retained
upstream source may not be committed to, or introduced into, either product repository until it is
made. Until then the patches stay local to the recovery workspace, and the committed evidence is an
inventory with hashes — not a distributable patch. This document does not choose a licence.

## 10. What remains, and the next action

Remaining work for the settings item, in prerequisite order:

1. **The owner's licence decision** (§9) — required *before* step 2, because step 2 introduces
   retained upstream source into the product. No product licence is chosen here.
2. **The implementation batch** (smallest viable), once step 1 is settled: port the proven pipe
   transport into the product (engine-side Get/Set handlers in the product backend on the Fcitx
   event loop + the `ControllerProxy`/`PendingCall` adapter + the retained widget subset), Pinyin
   page first. The spike's engine service is the reference implementation of the semantics, and
   `evidence/settings-ui-spike-2026-10-20/` is the reviewable record of what it proved.
3. **Selection stays separate** (§6): selecting Pinyin/Shuangpin is input-method-group management,
   not a config option; it needs its own mapping and is not claimed by anything here. `External`
   options likewise render but have no Windows launch path — product-integration work, not a
   validated feature.
4. **Packaging**: the EXE installer remains the next separate batch under D085 (framework,
   signing, channel still undecided under U4).

The settings item stays **open until actual product integration is complete** and appropriately
validated. U4, U6, WP7 and the other gates are untouched by this document. A Fusion-authored UI
remains unauthorized, and the Round-3 spike did not need one: the official widget layer ran on
Windows as-is (modulo the recorded patches).
