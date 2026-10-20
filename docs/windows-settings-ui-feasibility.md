# Windows settings UI — upstream reuse feasibility (D085)

Status: **feasibility investigated in two rounds (2026-10-20); no UI implemented; owner decision
required.** Scope: the settings-UI item of D085. The EXE installer is the following item and is not
covered here. Nothing here changes an accepted decision.

Evidence classes used below: **[measured]** = executed on this machine or read from the pinned
source; **[documented]** = a public API or behaviour taken from its own documentation, not executed
here; **[inferred]** = reasoning from the above, not tested; **[unresolved]** = not established.

Round 1 established what the **unmodified** upstream build requires. Round 2 traced **which
dependency belongs to which feature**, because the unmodified top-level build is not the cost of
every reuse route — that conflation was the error in the first version of this document.

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

**Viable in principle, not yet built.** It retains the official UI sources and changes the
boundary, rather than importing the whole Linux stack or writing a new UI:

1. **Build a subset**: `src/configtool` + `src/lib/{configlib,configwidgetslib}`, with `layout/`,
   `src/kcm`, `src/migrator` and `plasmathemegenerator` excluded. Excluding `layout/` removes X11,
   XKeyboardConfig and xkbfile outright; excluding `kcm` removes Plasma/Kirigami/KCMUtils. This is a
   build-file change, not a source change.
2. **Replace the transport at its single seam**: keep `DBusProvider`'s interface (`available()`,
   `canRestart()`, `availabilityChanged`, `controller()`) and implement it over the product's
   per-user named pipe. The replacement's methods return
   `QDBusPendingCall::fromCompletedCall(QDBusMessage)` — a public Qt API whose documented purpose is
   "simulating a pending call, but that has already finished" **[documented]**. Call sites then
   compile unchanged because they only use `auto` + `QDBusPendingCallWatcher` + `QDBusPendingReply`.
3. **Serve 2 methods for settings and 4 for selection** from the Host: `GetConfig`/`SetConfig`
   already map onto `Configuration::dumpDescription` + `AddonInstance::setConfig`, which the PoC
   proved work on this Host (`POC.md` §B). The four selection methods map onto Fcitx's own
   input-method-group API in the Host.
4. **Remove the layout feature** from `impage.cpp` (2 call sites) and drop `layoutselector`.
5. **KF6**: either ship `KF6WidgetsAddons` + `KF6ItemViews` (both build on Windows), or patch the
   three spots — replace `KColorButton` with `QToolButton`+`QColorDialog` (~20 lines), drop
   `fontbutton.cpp`'s `KFontChooser`, drop `addonselector.cpp` (the addon list is not needed when the
   product exposes one engine's settings).
6. **IsoCodes**: ship the three JSON files, or drop the language-name column.

What stays untouched is the valuable part: the schema-driven option renderer (~2.1k lines), the
key-sequence widget, and the official per-addon config-UI plugin interface.

**Costs and obligations.** Qt6 (Core/Gui/Widgets/DBus) becomes a product dependency — LGPLv3,
dynamically linked, and unavoidable for any upstream-widget route. The UI sources are
**GPL-2.0-or-later** (`src/configtool`, `src/lib`, `src/kcm`), so a fork carrying them puts the
settings executable under GPL-2.0-or-later; neither product repository has a `LICENSE` file, so the
product's own outbound licence has to be settled first. The fork must be recorded with its upstream
revision and licence text, and carried forward on upstream changes.

## 5. Remaining uncertainties [unresolved]

These are why A′ is "viable in principle", not "proven":

1. **Not compiled.** No Qt6 is installed here, so these sources have never been built with MSVC or
   run on Windows. Nothing in the source forbids it, but "builds" is unverified.
2. **The synthesized-reply path is unexecuted.** `fromCompletedCall` is documented, and the call
   sites only need a `QDBusPendingCall`, but whether a `QDBusPendingCallWatcher` delivers `finished`
   for a synthesized call with **no session bus at all** was not run. If it does not, the fallback is
   a small local interface replacing the 11 call sites.
3. **Custom types.** `GetConfig` returns `QDBusVariant` + `FcitxQtConfigTypeList`, and the values are
   `QDBusArgument`-marshalled maps (`configwidget.cpp:84`, `:155`). Building a reply message carrying
   those registered types is plausible but untested.
4. **Selection was not validated at all.** See §6.
5. **GPL and the product licence** are a decision, not a technical unknown.

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
| A′ | **Minimal upstream-UI adaptation** (§4): subset build + transport seam replaced + layout/KF6 reduced | The official UI sources, schema, apply, storage | **Viable in principle; unbuilt; uncertainties in §5.** The smallest route that satisfies "reuse upstream first" |
| A | Full upstream stack: Qt6 + KF6 + X11 + xkbcommon + IsoCodes + rebuilt runtime with keyboard/dbus modules + a session D-Bus | The official UI unmodified | Works, but imports Linux-only features the product does not need and reopens runtime provenance |
| B | Fusion-authored window over `dumpDescription`/`setConfig` via the existing named pipe | Official schema and apply only | **Not authorized by D085**; also the largest new-code surface for the same user-visible result |
| C | No UI: settings edited in `%APPDATA%\Fcitx5\config\fcitx5\conf\pinyin.conf` with the Host stopped (WP3 B3f) | Nothing new | Correct today, but gives the user no settings surface |

Note on C: the tray does **not** help here. Its menu opens the **log** folder
(`%LOCALAPPDATA%\fusion-host`, `windows-daily-use.md` §4), which contains no configuration; the
configuration lives under `%APPDATA%\Fcitx5`.

## 9. Not done in this batch

- No dependency installed, no Qt6, no runtime rebuild, no port, no UI built or launched.
- No product code changed; nothing installed or registered; the owner's Host, tray, browsers and
  profile untouched.
- No selection round-trip; only `PageSize` was applied, in-process, in the earlier PoC.
- `setSubConfig` paths (`dictmanager`, `clearuserdict`, `customphrase`) not exercised.
- No Release ZIP (nothing to package).

## 10. Next action

The recommended next batch is a **bounded build-and-seam spike for A′**, not a full port and not a
custom UI: build the subset from §4 against a Qt6 MSVC toolchain, replace `DBusProvider` with a
pipe-backed provider that serves `GetConfig`/`SetConfig` from the Host, and exercise the two
uncertainties that decide the route — whether a synthesized `QDBusPendingCall` drives the existing
call sites with no session bus (§5.2), and whether the subset builds unmodified apart from the
CMake exclusions (§5.1). If both hold, A′ is the route; if not, the fallback is a local interface at
the 11 call sites, still inside A′.

The settings item stays **open and unimplemented**, and the **EXE installer** remains the next
scheduled item under D085. A Fusion-authored UI remains unauthorized.
