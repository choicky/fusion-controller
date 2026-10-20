# Stage A capability mapping — global settings, input-method management, Shift/raw-input

Source-first mapping produced for the Stage A implementation instruction. Every claim is
traced to the exact pinned source or the current `fusion-windows` product source at base
`fe23f07` (product PR #28 merged) / controller `origin/main` `96fb28c` (PR #51 merged).

Authority sources read for this map (all at their pinned revisions):
- pinned fcitx5 core `a7f2528773ebfec98a20c49b6eea2333110b1c88` —
  reference checkout `/home/choicky/coding-projects/fcitx5-upstream-empty-range-preflight`
  (the `fcitx5-msvc` empty-range preflight tree; same generation as `runtime-wp3-20261018`).
- pinned fcitx5-chinese-addons `61474bd3aa9fca26d1c31df93343035697e9f265` —
  `/home/choicky/coding-projects/fcitx5-chinese-addons` (`git show 61474bd:im/pinyin/pinyin.h`).
- owner fork `choicky/fcitx5-configtool` `windows/fusion-settings` `0234c64` (base `9bd6022`).
- product `fusion-windows` `stage-a-settings` worktree at `fe23f07`.
- controller D072 (`windows-architecture.md`), D082, D085, D086, WP3 (`windows-stage6-wp3.md`),
  `windows-settings-ui-feasibility.md`, `provider-settings-acceptance.md`, `evidence/settings-ui-spike-2026-10-20/`.

## 1. The "Shift / raw-input commit" request is a per-input-method Pinyin option, not a global setting

The behaviour the request names ("press Shift / Enter to commit the raw input") is the official
Pinyin `KeyListOption` **`CommitRawInput`** (`im/pinyin/pinyin.h:275-282`), config key
`CommitRawInput`, on config URI **`fcitx://config/inputmethod/pinyin`**. Its default keys are
`Return, KP_Enter, Control+Return, Control+KP_Enter, Shift+Return, Shift+KP_Enter,
Control+Shift+Return, Control+Shift+KP_Enter`, constrain `AllowModifierLess`. The sibling
per-IM option `SwitchInputMethodBehavior` (`pinyin.h:234`, `fcitx://config/inputmethod/pinyin`)
controls what a *switch* does (Clear / CommitPreedit / CommitDefault); it is also not global.

There is **no bare-Shift "commit raw input" option** in the pinned sources. A lone `Shift_L` is the
**global** `AltTriggerKeys` *activate* hotkey (`fcitx5/src/lib/fcitx/globalconfig.cpp`, key
`AltTriggers`, default `{Shift_L}`), whose meaning is "trigger/activate the input method", not
"commit". So "Shift belongs to global settings" is only true of the *trigger* hotkey, and that
trigger is inert on this product path (§4). The commit-with-Shift the request means is
`CommitRawInput`'s `Shift+Return` family.

Because `CommitRawInput` is a normal Pinyin `KeyListOption`, the current `FusionSettings.exe`,
which already loads the whole `fcitx://config/inputmethod/pinyin` schema through the official
`ConfigWidget`, **already renders and can edit it**. No Fusion option or state machine is needed on
the settings side; the option is reused verbatim.

### Windows event requirements for `CommitRawInput` to fire

The real product key path is `TextService::OnKeyDown` → `physical_key` (`service.cpp:87-111`) →
`SemanticController::begin_key`/`begin_real_key` (`semantic.cpp:246`) → `BackendStimulus{keysym,
modifiers}` → `FcitxBackend::send` builds `fcitx::Key(KeySym(keysym), KeyStates(modifiers))` as a
**key-press** (`backend.cpp:500-506`). Translation is `translate_key` (`key_map.cpp:32`), used only
when `FUSION_WITH_FCITX5` is on (product build).

Modifier encoding today (`key_map.cpp:37`): `modifiers = (shift?1:0) | (caps?2:0)` — Shift bit and
CapsLock bit are carried and match Fcitx `KeyModifier` (`Shift=1<<0`, `CapsLock=1<<1`). Ctrl / Alt /
Win are **not** encoded: `classify_key` returns `{Pass, modified=true}` for them (`key_map.cpp:6`)
and `translate_key` returns early with `keysym=0, modifiers=0` (`key_map.cpp:35-36`);
`routes_key` then refuses any `modified` action (`semantic.cpp:240`), so Ctrl/Alt/Win combos never
build a stimulus. KP_Enter is **not** distinguished: `translate_key` maps only `VK_RETURN(0x0D)` →
`0xff0d` (`key_map.cpp:51-53`), never the numpad `KP_Enter (0xff8d)`.

Consequence, per default-key entry:
- `Shift+Return` → keysym 0xff0d, mod Shift: **delivered to the engine** while a composition/page is
  present (`return_has_engine_work`, `service.cpp:865-872`); an idle Return is left to the app (R36).
- `Return` → 0xff0d, mod 0: same, but R36 leaves idle Return native.
- `KP_Enter` → currently mis-sent as 0xff0d or not matched: **fixable by keysym translation only**
  (`0xff8d`), no ownership change. In scope.
- `Control+Return`, `Control+Shift+Return`, `Alt+…` → **not delivered** (no modifier bit + `modified`
  filtered by `routes_key`). Routing a modifier-held key to the engine is exactly the **D072 §5
  UNFROZEN** "production effect-capable key classification" and the WP3-B6 "modifier combinations
  never reach Fcitx (R19)" boundary. Changing it is an architecture decision, not a bounded
  adaptation. Reported as the boundary in the Change Contract; NOT implemented.

Key-up / bare-modifier: `OnTestKeyUp`/`OnKeyUp` set `pfEaten=FALSE` and forward nothing
(`service.cpp:879-900`); a lone Shift press/release never reaches the engine. Fcitx
`CommitRawInput` and hotkeys are **press-driven** (`instance.cpp:790-904` match on key event), so
key-up is not required for them. The instruction to "preserve ordinary Shift+letter and key-up /
modifier handling and not create an independent Shift state machine" is therefore satisfied simply
by leaving the existing key-up/modifier behaviour untouched and adding only the KP_Enter keysym.

## 2. Global configuration → official schema / API / widget

`fcitx://config/global` is served by Fcitx core as the **`GlobalConfig`** object
(`globalconfig.h:25`), two nested groups: `Hotkey` (`HotkeyConfig`) and `Behavior`
(`BehaviorConfig`). Official access, exactly as the Fcitx D-Bus controller does it
(`modules/dbus/dbusmodule.cpp:343-345, 425-432`):
- schema: `instance_->globalConfig().config().dumpDescription(raw)` — `config()` returns
  `const Configuration&` (`globalconfig.h` public), the same object `dump_settings_description`
  already consumes for per-IM pages (`backend.cpp:258`).
- values: `instance_->globalConfig().save(raw)` (`globalconfig.h:160`).
- apply: `instance_->globalConfig().load(raw, /*partial=*/true)`; then
  `if (globalConfig().safeSave()) instance_->reloadConfig();` (`dbusmodule.cpp:429-432`;
  `reloadConfig()` is public, `instance.h:401`).

The current product Host does **not** handle the global URI at all
(`parse_supported_input_method_uri` only accepts `pinyin`/`shuangpin`, `backend.cpp:63-70`). Stage A
adds the bounded global branch reusing the same `invoke_settings` event-loop one-at-a-time plumbing,
`merge_schema_values` delta/no-change logic, and composition guard already present.

### Windows-supported vs unsupported global options

`GlobalConfig` self-hides the Linux-only entries through Fcitx's own mechanism:
`OverrideXkbOption`/`CustomXkbOption` are `ConditionalHidden<!hasKeyboard>` and `hasKeyboard` is
`ENABLE_KEYBOARD`, which the packaged runtime builds `OFF`, so the official schema **already omits
them**; `EnabledAddons`/`DisabledAddons`/`PreloadInputMethod` are `hidden`. No Fusion filter is
needed for those — this is the "hide unsupported via the official schema" answer.

Functional support of what *does* render on the Windows product path:

| group | option | Windows product status | reason (source) |
| --- | --- | --- | --- |
| Behavior | `DefaultPageSize` | **supported** | read by engines lacking their own page size; popup page width is engine truth (`candidate_projection` mirrors `CandidateLayoutHint`) |
| Behavior | `ActiveByDefault` | **partial** | OS/TSF activation + U6 open-on-new-window is authoritative (`service.cpp` `declare_keyboard_open`/`hard_policy_allows`); this flag is not the Windows enable switch |
| Behavior | `ShareInputState` | **supported (engine semantics)** | consumed by the per-context pipeline; note per-context isolation is fenced by D072 §9/Stage 4 |
| Behavior | `resetStateWhenFocusIn` | **partial** | focus is not a semantic discontinuity (D072 §9); engine-internal |
| Behavior | `PreeditEnabledByDefault` | **supported** | drives whether the in-document client preedit is shown |
| Behavior | `ShowInputMethodInformation`, `CompactInputMethodInformation`, `ShowFirstInputMethodInformation`, `showInputMethodInformationWhenFocusIn` | **inert** | rendered by Fcitx's classic-ui/panel the product does not ship; the product's own candidate popup is a separate projection |
| Behavior | `AllowInputMethodForPassword`, `ShowPreeditForPassword` | **inert / must-not-enable** | the product's own InputScope/hard-policy blocks password fields before a stimulus exists (WP3 B5); enabling here would not override the Windows policy |
| Behavior | `AutoSavePeriod` | **supported** | drives Fcitx `periodicalSave_` (`instance.cpp reloadConfig`) |
| Hotkey | `TriggerKeys`, `AltTriggerKeys`, `ActivateKeys`, `DeactivateKeys`, `Enumerate*`, `PrevPage/NextPage/PrevCandidate/NextCandidate/TogglePreedit`, `ModifierOnlyKeyTimeout`, `ConsumeRedundantActivateKeys` | **inert on Windows** | these are matched by Fcitx **core as ordinary per-focus-context key events** (`instance.cpp:790-904`), not an X11 grab. The product never delivers bare Shift / modifier-held keys to the engine (§1; WP3-B6), and Ctrl+Space enable is the OS/TSF compartment. Page/candidate moves are synthesized as explicit `BackendStimulus` actions (Stage 5), not by forwarding these hotkeys. |

So "schema presence ≠ functionality": the global page is served from the official schema, and the
inert options are **explained** (GUI limitations label + this record), not hidden by a new filter.

## 3. Available/enabled input methods → official management APIs

Official surface (`fcitx5/src/lib/fcitx`):
- enumerate ALL available: `InputMethodManager::foreachEntries(fn(const InputMethodEntry&))`
  (`inputmethodmanager.h:142`). There is **no** `availableInputMethods()` on the manager; the D-Bus
  `AvailableInputMethods` just wraps `foreachEntries` (`dbusmodule.cpp:251-264`).
- enabled/ordered list: `currentGroup()`→`InputMethodGroup::inputMethodList()`
  (`inputmethodgroup.h:47`), `defaultInputMethod()/setDefaultInputMethod()` (49-50).
- apply + persist: `setGroup(InputMethodGroup)` (`inputmethodmanager.h:115`) then `save()` (72).
- active-vs-fallback semantics (WP3 finding, confirmed in `inputmethodgroup`/`instance`):
  **`Items/0` is the method in use**; `DefaultIM` is only the fallback; a bound context follows
  `Items/0`. `setDefaultInputMethod` rewrites a default equal to `Items/0` to `Items/1`.
- persistence = `profile` INI: `Groups/<n>/Items/<i>/Name|Layout`, `Default Layout`, `DefaultIM`,
  top-level `GroupOrder` (`inputmethodconfig_p.h:16-33`, `inputmethodmanager.cpp:453`). Removing a
  list item edits `profile` only — it never touches the method's `inputmethod/*.conf` or the user
  dictionaries/learning data (those live under `%APPDATA%\Fcitx5\pinyin` etc.). "Remove ≠ delete
  data" is satisfied by construction.
- validation: core `setGroup` (`inputmethodmanager.cpp:351-353`) **silently drops** unknown IMs and
  keeps fallback/default (354-358); an **empty** list is NOT rejected by core (persisted, then erased
  on next load). So "prevent invalid empty lists and unavailable selections" is a **product guard**
  the Stage A backend must add explicitly.

Packaged runtime truth (WP3): the runtime defines **only `pinyin` and `shuangpin`**
(`data/fcitx5/inputmethod/{pinyin,shuangpin}.conf`). "Methods actually available in the packaged
runtime" = that set. Multi-group editing, a keyboard-layout picker (X11) and a general
addon/dictionary manager are explicitly out of scope.

The current product already drives the official `Items/0`-front reorder for these two methods
(`settings_select_input_method`, `backend.cpp:573-627`) and reads a flat `available` (780). Stage A
generalizes that single selection primitive into a coherent **list-management** primitive (ordered
enabled list + active + fallback + available, with add/remove/reorder and the empty/unavailable
guards), reusing the same `invoke_settings` plumbing and official `setGroup/save`. It is one surface,
not a second manager.

## 4. Distinguishing the boundaries the instruction calls out

- Global hotkeys ≠ engine-specific commit behaviour: `AltTriggers`(Shift_L)/`Triggers`(Ctrl+Space) are
  global *trigger* hotkeys; `CommitRawInput` is per-Pinyin *commit*. They are different objects
  (`globalconfig.cpp` vs `im/pinyin/pinyin.h`) and different config URIs.
- Enabling an installed method ≠ installing a new engine: add/remove edits the group's `Items`
  membership over the methods the runtime already defines; no engine install, no `.conf` creation.
- Removing a list entry ≠ deleting user data: `Items` removal only rewrites `profile`; learning/dicts
  are untouched.
- Fcitx groups ≠ Windows TIP/language registration: the `profile` group list is Fcitx's own concept;
  the Windows TIP/`FusionTsf.dll` activation guide registration is separate and unchanged here.
