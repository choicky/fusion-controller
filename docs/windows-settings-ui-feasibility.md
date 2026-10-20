# Windows settings UI — upstream reuse feasibility (D085)

Status: **feasibility measured on 2026-10-20; no UI implemented; owner decision required.**
Scope: the settings-UI item of D085 (2026-10-20). The EXE installer is the following item and is not
covered here. Nothing in this document changes an accepted decision.

## 1. What was asked

D085 schedules an official settings UI that **prefers reusing Fcitx5's own settings UI and
configuration semantics** over a Fusion-specific configuration surface, and states that
feasibility on Windows is **not established**: the upstream UI's Windows dependencies and the way it
would communicate with this Host had to be inspected before an approach was chosen. A Fusion-written
UI is a fallback, **not authorized** by that decision.

The instruction for this batch added the rule that if reuse needs an architectural boundary change or
proves infeasible, the dependent implementation stops and reports source versions, files/call chain,
the measured obstacle and the smallest viable alternatives — and that the official UI must not be
silently replaced by a new Fusion UI.

## 2. Measured boundary: the official front end cannot be built or connected here

Inspected upstream, at the versions that match the pinned runtime generation:

| Component | Revision | Version |
| --- | --- | --- |
| `fcitx5-configtool` | `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b` | 5.1.16 |
| `fcitx5-qt` | `0285a5d18367d8f3af80dab7e4819d5555982340` | 5.1.16 |
| fcitx5 (pinned runtime) | `a7f2528773ebfec98a20c49b6eea2333110b1c88` | 5.1.23 |
| fcitx5-chinese-addons | `61474bd3aa9fca26d1c31df93343035697e9f265` | 5.1.15 |
| libime | `171edcf137001e8eb8274f53ec010058cda70b09` | 1.1.17 |

A real configure of `fcitx5-configtool` in its most favourable configuration
(`-DENABLE_KCM=OFF -DENABLE_TEST=OFF`, i.e. without the Plasma/Kirigami branch) stops at
`CMakeLists.txt:21`:

```
Could not find a package configuration file provided by "Qt6" (requested version 6.4.0)
```

A probe project that repeats every declared `find_package` measures 4 of the 17 requirements present
(ECM, Fcitx5Core, Fcitx5Config, Fcitx5Utils); **Qt6, all four KF6 modules, Fcitx5Qt6DBusAddons,
Fcitx5Qt6WidgetsAddons, IsoCodes, X11, XKeyboardConfig, PkgConfig and Gettext are absent**, and
`xkbfile` cannot be probed because pkg-config itself is absent — 13 of 17 missing.
`Fcitx5ModuleKeyboard`, which the daemon-side `src/modules/dbus` links, is absent as well.

Three of these absences are structural, not environmental:

1. **X11 is unconditional.** `CMakeLists.txt:82-84` requires `X11`, `XKeyboardConfig` and
   `pkg-config xkbfile` outside every `option()`, so no configuration removes them.
   `fcitx5-qt/CMakeLists.txt:45` requires `XKBCommon` unconditionally, and
   `fcitx5-qt/qt6/CMakeLists.txt:2` requires Qt6 `DBus`.
2. **The transport does not exist.** Both front ends reach a running Fcitx over D-Bus:
   `fcitx5-configtool/src/lib/configlib/dbusprovider.h` holds a `FcitxQtControllerProxy`, and the
   daemon side is `/controller` on `org.fcitx.Fcitx5`
   (`fcitx5/src/modules/dbus/dbusmodule.cpp:57`, `:784-786`, methods `GetConfig`/`SetConfig`).
   The pinned runtime ships **no dbus addon** (`runtime-release/data/fcitx5/addon/` has only
   clipboard, imselector, pinyin, pinyinhelper, punctuation, quickphrase, spell, unicode), and
   `src/modules/dbus/CMakeLists.txt` links `Fcitx5::Module::Keyboard`, which this runtime does not
   export because it was built with `-DENABLE_KEYBOARD=OFF` and `-DENABLE_DBUS=OFF`
   (`tools/fcitx/build-native.ps1:92`).
3. **The runtime's `fcitx5-configtool` is not a program.** `runtime-release/bin/fcitx5-configtool` is
   a `/bin/sh` wrapper — a Linux artifact. `bin/Fcitx5Config.dll` is the configuration library, not
   a UI.

Reuse therefore requires a new Qt6/KF6/X11/xkbcommon/IsoCodes dependency stack, a runtime rebuilt
with the keyboard and dbus modules, and a session D-Bus on Windows. That is a new transport plus a
new third-party stack: an architectural boundary change, not a Windows adaptation of an existing
seam.

**Licence.** Every inspected source directory of `fcitx5-configtool` (`src/configtool`, `src/lib`,
`src/kcm`) is `GPL-2.0-or-later`, unlike the `LGPL-2.1-or-later` runtime. Importing or forking it
puts the derived settings front end under GPL-2.0-or-later. Neither product repository currently
carries a `LICENSE` file, so the product's own outbound licence is itself unstated — that has to be
settled before GPL code is imported.

## 3. What is reusable today, officially, on this Host

Two upstream mechanisms work unchanged and are enough for a settings front end to be a *front end*:

- `fcitx::Configuration::dumpDescription(RawConfig&)` serialises the whole option schema — type,
  description, default, integer bounds, enum names with i18n — from the addon's own compile-time
  `FCITX_CONFIGURATION` macros. There is no second option list to maintain.
- `fcitx::AddonInstance::setConfig(const RawConfig&)` is the official validated apply. The pinyin
  override (`im/pinyin/pinyin.h:426-430`) does `config_.load(config, true)`,
  `safeSaveAsIni(config_, "conf/pinyin.conf")` and `populateConfig()` — apply and persistence through
  Fcitx's own writer, with no Host restart.

Measured on an isolated per-user profile (`USERPROFILE`/`APPDATA`/`LOCALAPPDATA` redirected; the
owner's profile untouched), with the same static addon set and `--disable=all --enable=...` argv the
Host uses:

| Check | Result |
| --- | --- |
| Schema from `dumpDescription` | `PageSize` Integer, "Candidates Per Page", default 7, min 3, max 10 |
| Shuangpin scheme enum | `Ziranma` (default), `MS`, `Ziguang`, `ABC`, `Zhongwenzhixing`, `PinyinJiajia`, `Xiaohe`, `GB Standard` |
| Engine page size before apply | 7 (read from the engine's own `CandidateList::pageSize()`) |
| `setConfig` with `PageSize=9` | re-read 9 |
| Engine page size after apply | 9 |
| On-disk `conf/pinyin.conf` | `PageSize=9` |
| Orderly restart | 9, engine 9 |
| No-change run | `pinyin.conf` byte-identical |
| `user.dict` / `user.history` | unchanged across an apply |

Both `pinyin` and `shuangpin` input methods exist in the runtime
(`runtime-release/data/fcitx5/inputmethod/{pinyin,shuangpin}.conf`), so Pinyin/Shuangpin selection
and the Shuangpin scheme are reachable through the official schema alone.

## 4. Constraints a future front end must honour

These are measured, not assumed:

1. **Describe per input method.** `AddonInstance::getConfig()` returns the pinyin addon's `&config_`,
   which exposes `ShuangpinProfile` and `ShowShuangpinMode` even for the `pinyin` input method. The
   per-IM call `InputMethodEngine::getConfigForInputMethod(entry)` returns `&pyConfig_`, where they
   are hidden (`im/pinyin/pinyin.cpp:2484-2493`). A UI that reads `getConfig()` would offer Shuangpin
   controls under Pinyin.
2. **One hide does not reach the description.** `Fuzzy$FuzzyConfig$PartialSp` stays visible for the
   `pinyin` input method even though `pinyin.cpp:1003` hides it, because
   `dumpDescriptionImpl` describes a sub-config from `option->subConfigSkeleton()`, which returns a
   fresh skeleton (`fcitx-config/option.h:384-388`) rather than the live instance. Hidden flags are
   therefore not authoritative for sub-config options.
3. **No hide loss across apply.** An earlier reading of `pyConfig_ = config_`
   (`im/pinyin/pinyin.cpp:1310`) suggested `setConfig` would drop the construction-time hides.
   Measured: it does not — `Configuration::copyHelper`
   (`fcitx-config/configuration.cpp:98-107`) copies values into the destination's own option objects,
   so annotations survive. No fix is needed and none was written.
4. **The Host must stay the only applier.** WP3 B3f measured that a configuration file edited while
   the Host runs is overwritten by the Host's next save. Any front end has to apply through the
   running Host (an IPC request answered on the Fcitx event loop), not by writing files behind it.
5. **Any IPC extension is a protocol change.** `include/fusion/ipc_protocol.hpp` has
   `protocol_version = 4` and a `MessageType` enum of `0x01..0x0A`; `parse_header` rejects type bytes
   outside that range. Adding configuration messages raises the version and needs its own bounded
   batch.

## 5. Smallest viable alternatives

Listed in increasing cost. None is started; D085 leaves the UI's scope undecided beyond "official
settings for what the product actually exposes", and a Fusion-written UI needs an explicit
authorization that this record does not give.

| # | Approach | What it reuses | Cost / risk |
| --- | --- | --- | --- |
| A | Rebuild the runtime with keyboard + dbus modules, add Qt6/KF6/X11/xkbcommon/IsoCodes, build `fcitx5-qt` + `fcitx5-configtool`, and supply a session D-Bus on Windows | The official UI itself | Largest: new dependency stack, runtime rebuild, a D-Bus daemon on Windows, GPL-2.0-or-later obligations, and the `GetConfig`/`SetConfig` surface has to be served by the Host. Reopens runtime provenance for every later batch |
| B | A thin native Windows settings window that renders `dumpDescription` and applies through `setConfig` over the existing per-user named pipe | Official schema, official apply, official storage; no option list, no second configuration model | Moderate: a new executable, three new IPC message types and a `protocol_version` bump, tray + Start Menu entries, package layout. The window itself is Fusion-authored — **requires an explicit D085 authorization** |
| C | No UI: document that settings are edited in `%APPDATA%\Fcitx5\config\fcitx5\conf\pinyin.conf` with the Host stopped (WP3 B3f), and keep the tray's "open folder" affordance | Nothing new | Smallest, and honest about today's behaviour; leaves the settings requirement without a user-facing surface |

The measured facts above apply to all three: the schema and apply path are ready, so B does not need
to invent a configuration model, and C remains correct until either A or B is chosen.

## 6. Not done in this batch

- No UI was built or launched, in either repository, and no product code changed.
- Nothing was installed or registered; the owner's Host, tray and browsers were not touched.
- The owner's profile was not read or written (no file under `%APPDATA%\Fcitx5` has an mtime inside
  the PoC window).
- Only `PageSize` was applied end to end; no Shuangpin-profile change and no real-application
  candidate page were observed.
- `setSubConfig` paths (`dictmanager`, `clearuserdict`, `customphrase`) were not exercised.
- No Release ZIP was built, because no integration was delivered to package.

## 7. Next action

The owner chooses between A, B and C. Until then the next scheduled work is unchanged: the settings
UI item stays open, and the **EXE installer** follows it under D085. B is the only option that can
produce a usable settings surface without reopening the runtime, and it needs an explicit decision
that a Fusion-authored window over the official schema is authorized — D085 currently says it is not.
