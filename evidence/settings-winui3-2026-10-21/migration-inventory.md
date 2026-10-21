# Migration inventory — Windows settings (C# + WinUI 3)

Status: **design inventory**, produced before any client code. Nothing in this table is implemented, compiled or
validated; the column "new location" is the accepted navigation target, not an existing screen. The blocking reason is
in `blocker.md`.

## 0. Method and sources

Every row was read from the pinned sources on this machine, not from the current UI's labels:

- Pinyin/Shuangpin engine schema: `fcitx5-chinese-addons` `61474bd3aa9fca26d1c31df93343035697e9f265`,
  `im/pinyin/pinyin.h` (`PinyinEngineConfig`, `FuzzyConfig`) — the configuration object behind
  `fcitx://config/inputmethod/pinyin`.
- Global schema: `fcitx5` `a7f2528773ebfec98a20c49b6eea2333110b1c88`, `src/lib/fcitx/globalconfig.cpp` — behind
  `fcitx://config/global`.
- Input-method group: `fcitx://config/inputmethod` (group `Items`, `DefaultIM`), as implemented in Stage A
  (`settings_api_version` 2, `src/fcitx/im_list.hpp`).
- Transport: `include/fusion/settings.hpp` + `settings_protocol.hpp` — a lossless `dumpDescription()` view
  (name, type, description, default, properties) plus `save()` values. **The client must be schema-driven**; the
  tables below exist to prove coverage and to decide placement, not to hard-code editors.

Storage for all three URIs is Fcitx's own per-user configuration (`%APPDATA%\Fcitx5\config`), written by the Host
through the existing `SetConfig` path. No new store is introduced.

Legend — **Status**: `S` supported and reachable in the design · `NA` in the schema but has no effect on this Windows
path (must be shown with an explanation, not hidden silently) · `DEF` deferred, reason given · `VER` needs a Windows
check before it is claimed.

## 1. Input methods — `fcitx://config/inputmethod`

| Item | Type / default | Old UI | New location | Status |
|---|---|---|---|---|
| Group membership (`Items`) | ordered list of IM names; product default = the runtime's methods | list page, "current group" table | Input methods · group list | S |
| Active method (`Items/0`) | first entry of the group; follows bound contexts | list page, first row | Input methods · active badge | S |
| Fallback method (`DefaultIM`) | string; empty = none | list page, combo | Input methods · "Fallback method" | S |
| Order | positional | drag/up-down buttons | Input methods · drag handle + keyboard move | S |
| Add/remove | membership only; removing never deletes the method definition or learning data | add/remove buttons | Input methods · Add…/Remove | S |
| Activation vs editing | — | editing a method's settings does not activate it | method card → "Configure" opens the method page without changing `Items/0` | S (design rule) |

## 2. Pinyin and Shuangpin — `fcitx://config/inputmethod/pinyin` (one shared schema)

Pinyin and Shuangpin are **the same engine configuration**. The client must hold **one** draft for these options and
expose it from both the Pinyin page and the Shuangpin page; there is no second copy and no per-profile storage.

### 2.1 Engine and candidates

| Option | Type / default | New location | Status |
|---|---|---|---|
| `ShuangpinProfile` | enum, default Ziranma | Candidates and input · Shuangpin profile | S |
| `ShowShuangpinMode` | bool, true | Candidates and input · More settings | S |
| `PageSize` | int 3–10, default 7 | Candidates and input · Candidates per page | S |
| `SpellEnabled` | bool, true | Candidates and input · English candidates | S |
| `SymbolsEnabled` | bool, true | Candidates and input · Symbol candidates | S |
| `ChaiziEnabled` | bool, true | Candidates and input · More settings | S |
| `ExtBEnabled` | bool, true (desktop) | Candidates and input · More settings | S |
| `StrokeCandidateEnabled` | bool, true | Candidates and input · Stroke candidates | S |
| `CloudPinyinEnabled` | bool, false | Candidates and input · More settings (needs a network check) | VER |
| `CloudPinyinIndex` | int 1–10, default 2 | Candidates and input · More settings | S |
| `CloudPinyinAnimation` | bool, true | Candidates and input · More settings | S |
| `KeepCloudPinyinPlaceHolder` | bool, false | Candidates and input · More settings | S |
| `CloudPinyin` (sub-config) | URI `fcitx://config/addon/cloudpinyin` | Candidates and input · "Configure Cloud Pinyin" | DEF — separate addon page; not part of this surface |
| `PreeditMode` | enum, default ComposingPinyin (desktop) | Candidates and input · Preedit mode | S |
| `PreeditCursorPositionAtBeginning` | bool, true (desktop) | Candidates and input · Preedit cursor | S |
| `PinyinInPreedit` | bool, false | Candidates and input · Show full pinyin in preedit | S |
| `Prediction` | bool, false on desktop | Candidates and input · Prediction | S |
| `KeepCurrentContext` | bool, true | Candidates and input · Prediction · More | S |
| `PredictionSize` | int 3–100, default 49 | Candidates and input · Prediction size | S |
| `BackspaceBehaviorOnPrediction` | enum, default BackspaceWhenNotUsingOnScreenKeyboard | Candidates and input · Prediction · More | S |
| `SwitchInputMethodBehavior` | enum, default CommitPreedit | Keys · "Action when switching input method" | S |
| `Learning` | bool, true | Candidates and input · Learning | S |

### 2.2 Fuzzy pinyin (`Fuzzy` sub-config)

All 19 switches live under `Fuzzy` in the same URI. Design: one "Fuzzy pinyin" expander on the Candidates and input
page, grouped as *initial/final pairs* and *segmentation*, with the defaults shown.

| Option | Default | Option | Default |
|---|---|---|---|
| `VE_UE` (ue → ve) | true | `AN_ANG` | false |
| `NG_GN` (common typo) | true | `EN_ENG` | false |
| `LowerCaseMatchLetter` | true | `IAN_IANG` | false |
| `Inner` (xian → xi'an) | true | `IN_ING` | false |
| `InnerShort` (qie → qi'e) | true | `U_OU` | false |
| `PartialFinal` (e → en/eng/ei) | false | `UAN_UANG` | false |
| `PartialSp` (shuangpin > 4) | false | `C_CH` | false |
| `V_U` | false | `F_H` | false |
| | | `L_N` | false |
| | | `L_R` | false |
| | | `S_SH` | false |
| | | `Z_ZH` | false |
| `Correction` (CorrectionLayout) | None on desktop | — | — |

Status: `S` for all; `PartialSp` and `Correction` are shown in "More settings" (they are `OptionalHideInDescription`
upstream, i.e. advanced).

### 2.3 Pinyin key bindings (`KeyListOption`, ordered, multiple values)

| Option | Default keys | Positional? | New location | Status |
|---|---|---|---|---|
| `ForgetWord` | Control+7 | no | Keys · Learning | S |
| `PrevPage` | *(empty)* | no | Keys · Paging | S |
| `NextPage` | *(empty)* | no | Keys · Paging | S |
| `PrevCandidate` | Shift+Tab | no | Keys · Candidate navigation | S |
| `NextCandidate` | Tab | no | Keys · Candidate navigation | S |
| `CurrentCandidate` | *(empty)* | no | Keys · Candidate navigation | S |
| `CommitRawInput` | Return, KP_Enter, Control+Return, Control+KP_Enter, Shift+Return, Shift+KP_Enter, Control+Shift+Return, Control+Shift+KP_Enter | no | Keys · Commit | S (with the Windows caveat below) |
| `SecondCandidate` … `ThirdCandidate` | positional digit keys | **yes** — N-th key selects N-th candidate | Keys · Candidate selection | S, order preserved |
| `UseKeypadAsSelection` | bool, false | — | Keys · Candidate selection | S |
| `ChooseCharFromPhrase` | *(empty)* | no | Keys · Phrase selection | S |
| `ChooseLastCharFromPhrase` | `Key`, *(schema default)* | — | Keys · Phrase selection | S |
| `BackSpaceToUnselect` | bool, true | — | Keys · Phrase selection | S |
| `FilterByStroke` | *(empty)* | no | Keys · More | S |
| `QuickPhraseKey` | `Key`, default `;` | — | Keys · Quick phrase | S |
| `VAsQuickphrase` | bool, false | — | Keys · Quick phrase | S |
| `QuickPhrase` (external) | URI `fcitx://config/addon/quickphrase` | — | Keys · Quick phrase · Manage | DEF — separate tool, not implemented on Windows |

**Windows caveat (measured earlier, unchanged by this UI):** `KP_Enter` entries never match because the TSF path
delivers the main Return keysym, and Ctrl/Alt combos are not delivered to the engine (`routes_key` refuses modified
keys). The editor must show these bindings as the engine stores them, and must not claim they take effect. This is
recorded in `windows-architecture.md` §22 and is not changed by the settings task.

### 2.4 Advanced engine values

| Option | Type / default | New location | Status |
|---|---|---|---|
| `Number of sentence` (`Nbest`) | int, *(schema default)* | Candidates and input · More settings | S |
| `WordCandidateLimit` | int, *(schema default)* | Candidates and input · More settings | S |
| `LongWordLengthLimit` | int, *(schema default)* | Candidates and input · More settings | S |
| `CustomPhrase` (external) | URI `fcitx://config/addon/customphrase` | Candidates and input · More settings · Manage custom phrases | DEF — separate tool |
| `DictManager` (external) | URI `fcitx://config/addon/dictmanager` | Candidates and input · More settings · Manage dictionaries | DEF — separate tool |
| `FirstRun` | hidden bool, true | — | not surfaced (upstream hides it) |

## 3. Global configuration — `fcitx://config/global`

| Option | Type / default | New location | Status |
|---|---|---|---|
| `TriggerKeys` | key list, default Control+space, Zenkaku_Hankaku, Hangul | Keys · Toggle input method | S |
| `EnumerateWithTriggerKeys` | bool, *(schema default)* | Keys · More | S |
| `ActivateKeys` | key list, default Hangul_Hanja | Keys · Activate / deactivate | NA — no Windows behaviour |
| `DeactivateKeys` | key list, default Hangul_Romaja | Keys · Activate / deactivate | NA |
| `ConsumeRedundantActivateKeys` | bool, *(schema default)* | Keys · More | NA |
| `AltTriggerKeys` | key list, default Shift_L | Keys · Temporary toggle | NA — bare Shift is not routed (unresolved product boundary) |
| `EnumerateForwardKeys` / `EnumerateBackwardKeys` | key lists, *(schema default)* | Keys · Group switching | NA |
| `EnumerateSkipFirst` | bool | Keys · Group switching | NA |
| `EnumerateGroupForwardKeys` / `EnumerateGroupBackwardKeys` | key lists | Keys · Group switching | NA |
| `PrevPage` / `NextPage` (global fallback) | key lists | Keys · Paging fallback | S — separate variables, **not** an inheritance layer for the Pinyin paging keys |
| `PrevCandidate` / `NextCandidate` (global fallback) | key lists, default Shift+Tab / Tab | Keys · Candidate fallback | S |
| `TogglePreedit` | key list, default Control+Alt+P | Keys · More | NA |
| `ModifierOnlyKeyTimeout` | int −1…5000 | Keys · More | NA |
| `ActiveByDefault` | bool | Input methods · Behaviour | NA — Windows TSF decides |
| `resetStateWhenFocusIn` | enum | Input methods · Behaviour | NA |
| `ShareInputState` | enum | Input methods · Behaviour | NA |
| `PreeditEnabledByDefault` | bool, true | Candidates and input · Preedit | NA |
| `ShowInputMethodInformation` | bool, true | Candidates and input · Info panel | NA |
| `showInputMethodInformationWhenFocusIn` | bool, false | Candidates and input · Info panel | NA |
| `CompactInputMethodInformation` | bool | Candidates and input · Info panel | NA |
| `ShowFirstInputMethodInformation` | bool | Candidates and input · Info panel | NA |
| `DefaultPageSize` | int 3–10 | Candidates and input · Fallback page size | S |
| `AllowInputMethodForPassword` | bool | Candidates and input · Password fields | NA — hard policy in the TIP |
| `ShowPreeditForPassword` | bool | Candidates and input · Password fields | NA |
| `CustomXkbOption` | string | — | NA — no keyboard layouts in this runtime |
| `EnabledAddons` / `DisabledAddons` | hidden string lists | — | not surfaced (upstream hides them; changing them is not a user setting) |
| `PreloadInputMethod` | hidden bool | — | not surfaced |
| `AutoSavePeriod` | int | Candidates and input · More | NA |
| `Hotkey` / `Behavior` (nested) | sub-configs | — | DEF — no Windows hotkey or behaviour surface exists; shown read-only with an explanation |

## 4. Cross-cutting semantics the client must honour

1. **One draft per underlying option.** Pinyin and Shuangpin link to the same `pinyin` schema; a second view must not
   hold a stale copy (a save from the stale view would revert the other). One canonical draft/baseline keyed by
   `(URI, option path)`.
2. **Positional lists stay positional.** Candidate-selection key lists are indexed; no sorting, no de-duplication.
3. **Empty ≠ default.** `PrevPage`/`NextPage` are genuinely empty by default. Reset writes the schema default; clearing
   writes an empty list. The UI must distinguish "cleared" from "reset".
4. **Schema-known unsubmitted options survive.** Only options the client actually changed are submitted; the Host
   merges into the full `save()` tree, and keys the schema does not know are dropped by Fcitx itself (measured in
   Stage A). The client must not claim arbitrary unknown keys survive.
5. **Comma/period presets edit the same official list.** A preset is a shortcut that replaces the list's content after
   an explicit confirmation; it is not a separate boolean and it does not merge silently.
6. **Alternatives are not chords.** One action = N independent bindings; one binding = one complete chord. Recording
   Escape cancels; an unrecognized recording leaves the previous value.
7. **Duplicate/overlapping bindings** are reported as warnings computed from the values shown; the UI must not invent
   priority rules or silently rewrite a list.
8. **No atomic claim across URIs.** Saving `global` and `pinyin` can partially succeed; report per-URI results and keep
   the remaining edits.
9. **No-change save writes nothing** (no setter call), so no configuration or learning file is touched.

## 5. Coverage summary

| Group | Options in schema | Reachable in the design | Deferred | Not applicable on Windows |
|---|---|---|---|---|
| Input-method list | 4 concepts | 4 | 0 | 0 |
| Pinyin/Shuangpin engine + fuzzy | 22 + 20 | 41 | 1 (CloudPinyin sub-page) | 0 |
| Pinyin key bindings | 17 | 15 | 2 (QuickPhrase, DictManager/CustomPhrase external tools) | 0 |
| Global configuration | 31 | 12 | 2 (Hotkey/Behavior sub-configs) | 17 |

The 17 "not applicable" global options are still **shown** with a one-line explanation: hiding them would make the
surface disagree with the official schema and would look like a missing feature.

## 6. What this inventory does not establish

- Any Windows effect of a value: the mapping is schema-level only. "A displayed value is not proof the engine acted on
  it" (the same rule the Stage A record used).
- Any key translation: the `KP_Enter` / Ctrl+Alt caveats above come from the earlier measured key path, not from this
  inventory.
- Editor behaviour of the new client: it is not compiled (see `blocker.md`).
