# Migration inventory — Windows settings (C# + WinUI 3)

Status: **design inventory, corrected 2026-10-21.** It records what the schema actually contains and where each item is
meant to live in the new navigation. **Nothing here is implemented**: the client is not compiled (see `blocker.md`), so
"disposition" is a design decision, not an observed behaviour.

## 0. Method

Read from the pinned sources on this machine:

- `fcitx5-chinese-addons` `61474bd3aa9fca26d1c31df93343035697e9f265` — `im/pinyin/pinyin.h` (the `PinyinEngineConfig`
  object behind `fcitx://config/inputmethod/pinyin`).
- `fcitx5` `a7f2528773ebfec98a20c49b6eea2333110b1c88` — `src/lib/fcitx/globalconfig.cpp` (`Hotkey` and `Behavior`
  containers behind `fcitx://config/global`) and `src/lib/fcitx/instance.cpp` (the consumers).
- Packaged runtime `runtime-wp3-20261018` — `data/fcitx5/addon/*.conf` decides which addons actually exist.

Three facts are kept separate for every row, because they fail independently:

1. **Schema** — the option exists, with its official type/default/constraints.
2. **Consumer** — the Fcitx code that reads it (file and role).
3. **Delivery** — whether the event that triggers that consumer can reach it through this product's TSF path.

Disposition vocabulary: **R** reachable in the design (schema + consumer + a delivery path that exists) · **C** conditional
limitation, stated in the row (consumer exists, but the triggering event is not delivered on this path) · **X**
external/sub-configuration link to another configuration surface · **H** hidden by upstream, not exposed · **NV** not
validated (no runtime test executed; the default for anything not measured).

A row marked **R** means "the design exposes it"; it does **not** mean the value was observed to take effect. Nothing in
this document is a product-validation claim.

## 1. Input-method management — `fcitx://config/inputmethod`

Not a `Config` schema but a group of operations (`settings_api_version` 2). Listed so that operations are not counted as
configuration fields.

| Item | Semantics | Old UI | New location | Disposition |
|---|---|---|---|---|
| Group membership (`Items`) | ordered list of method names in the group | list page | Input methods · group list | R (already implemented in the current client) |
| Active method (`Items/0`) | the method a bound context follows | first row | Input methods · active badge | R |
| Fallback method (`DefaultIM`) | fallback only, **not** the active method | combo | Input methods · fallback | R |
| Order | positional | up/down | Input methods · drag / keyboard move | R |
| Add / remove | membership only; never deletes a method definition or learning data | buttons | Input methods · add/remove | R |

Editing a method's settings must not change `Items/0`. These are **operations over the official group**, not `SetConfig`
writes of a schema field, and must not be presented as one atomic save with option edits.

## 2. Pinyin and Shuangpin — `fcitx://config/inputmethod/pinyin`

One shared engine configuration. Shuangpin is a *profile value* inside it, not a separate schema, so a single draft per
option serves both views; the views may show different subsets, but must not keep separate copies.

**Shared-field identity (explicit mapping).** The two input methods declare the same addon: `data/fcitx5/inputmethod/pinyin.conf`
and `shuangpin.conf` both carry `Addon=pinyin`, while the settings layer addresses each method by its **own public URI**
(`fcitx://config/inputmethod/pinyin` and `fcitx://config/inputmethod/shuangpin`). Both URIs resolve to the **same**
`PinyinEngineConfig` instance owned by the `pinyin` addon. The draft identity is therefore normalised to
**`(addon:pinyin, option path)`**, not to the public URI: a literal `(URI, path)` key would give the two views independent
drafts of one value, and saving the stale one would revert the other. Method-specific fields (`ShuangpinProfile`,
`ShowShuangpinMode`) are ordinary fields of that same object, not a second store. Visibility stays per schema: the views may
place or hide options differently (e.g. `OptionalHideInDescription` entries live under More), but a field hidden in one view
is never fabricated there, and the value edited in either view is the same one.

### 2.1 Engine, candidates and preedit

| Path | Type · default · constraints | Consumer | New location | Disposition |
|---|---|---|---|---|
| `ShuangpinProfile` | enum · `Ziranma` | pinyin engine | Candidates and input · Shuangpin profile | R |
| `ShowShuangpinMode` | bool · `true` | pinyin engine | Candidates and input · More | R |
| `PageSize` | int · `7` · 3–10 | pinyin engine | Candidates and input · candidates per page | R |
| `SpellEnabled` | bool · `true` | pinyin engine (spell addon) | Candidates and input · English candidates | R |
| `SymbolsEnabled` | bool · `true` | pinyin engine | Candidates and input · symbol candidates | R |
| `ChaiziEnabled` | bool · `true` | pinyin engine | Candidates and input · More | R |
| `ExtBEnabled` | bool · `true` on desktop | pinyin engine | Candidates and input · More | R |
| `StrokeCandidateEnabled` | bool · `true` | pinyin engine | Candidates and input · stroke candidates | R |
| `CloudPinyinEnabled` | bool · `false` | cloudpinyin addon | Candidates and input · More | C — the `cloudpinyin` addon is **not** in the packaged set, so the value is stored but the feature cannot act here |
| `CloudPinyinIndex` | int · `2` · 1–10 | cloudpinyin addon | Candidates and input · More | C — same missing addon |
| `CloudPinyinAnimation` | bool · `true` | cloudpinyin addon | Candidates and input · More | C — same missing addon |
| `KeepCloudPinyinPlaceHolder` | bool · `false` | cloudpinyin addon | Candidates and input · More | C — same missing addon |
| `PreeditMode` | enum · `ComposingPinyin` on desktop | pinyin engine | Candidates and input · preedit mode | R |
| `PreeditCursorPositionAtBeginning` | bool · `true` | pinyin engine | Candidates and input · preedit cursor | R |
| `PinyinInPreedit` | bool · `false` | pinyin engine | Candidates and input · show full pinyin | R |
| `Prediction` | bool · `false` on desktop | pinyin engine | Candidates and input · prediction | R |
| `KeepCurrentContext` | bool · `true` | pinyin engine | Candidates and input · prediction · more | R |
| `PredictionSize` | int · `49` · 3–100 | pinyin engine | Candidates and input · prediction size | R |
| `BackspaceBehaviorOnPrediction` | enum · `BackspaceWhenNotUsingOnScreenKeyboard` | pinyin engine | Candidates and input · prediction · more | R |
| `SwitchInputMethodBehavior` | enum · `CommitPreedit` | pinyin engine | Keys · action when switching input method | R |
| `Learning` | bool · `true` | pinyin engine | Candidates and input · learning | R |

### 2.2 Fuzzy pinyin — `Fuzzy` (nested container)

`Fuzzy` holds 20 switches plus `Correction`. The container is **not** an additional setting; only its children are edited.

| Child | Default | Child | Default |
|---|---|---|---|
| `VE_UE` | `true` | `AN_ANG` | `false` |
| `NG_GN` | `true` | `EN_ENG` | `false` |
| `LowerCaseMatchLetter` | `true` | `IAN_IANG` | `false` |
| `Inner` | `true` | `IN_ING` | `false` |
| `InnerShort` | `true` | `U_OU` | `false` |
| `PartialFinal` | `false` | `UAN_UANG` | `false` |
| `PartialSp` | `false` | `C_CH` | `false` |
| `V_U` | `false` | `F_H` | `false` |
| | | `L_N` | `false` |
| | | `L_R` | `false` |
| | | `S_SH` | `false` |
| | | `Z_ZH` | `false` |
| `Correction` | `None` on desktop | — | — |

All children: **R**, on Candidates and input · fuzzy pinyin (advanced items under More).

### 2.3 Key bindings — every entry is an **ordered list of alternative bindings** for its named action

| Path | Default | Positional? | Consumer | New location | Disposition |
|---|---|---|---|---|---|
| `ForgetWord` | `Control+7` | no | pinyin engine | Keys · learning | R |
| `PrevPage` | `minus`, `Up`, `KP_Up`, `Page_Up` | no | pinyin engine | Keys · paging | R |
| `NextPage` | `equal`, `Down`, `KP_Down`, `Page_Down` | no | pinyin engine | Keys · paging | R |
| `PrevCandidate` | `Shift+Tab` | no | pinyin engine | Keys · candidate navigation | R |
| `NextCandidate` | `Tab` | no | pinyin engine | Keys · candidate navigation | R |
| `CurrentCandidate` | `space`, `KP_Space` | no | pinyin engine | Keys · candidate navigation | R |
| `CommitRawInput` | `Return`, `KP_Enter`, `Control+Return`, `Control+KP_Enter`, `Shift+Return`, `Shift+KP_Enter`, `Control+Shift+Return`, `Control+Shift+KP_Enter` | no | pinyin engine | Keys · commit | R for the unmodified entries; **C** for the modified and `KP_Enter` entries (§2.6) |
| `SecondCandidate` | **empty list** | **no** | pinyin engine | Keys · candidate selection | R |
| `ThirdCandidate` | **empty list** | **no** | pinyin engine | Keys · candidate selection | R |
| `UseKeypadAsSelection` | bool · `false` | — | pinyin engine | Keys · candidate selection | R |
| `ChooseCharFromPhrase` | `[` | **yes** — the N-th binding selects the N-th character of the current candidate (upstream tooltip) | pinyin engine | Keys · phrase selection | R; order is semantic, never sorted or de-duplicated |
| `ChooseLastCharFromPhrase` | `]` | no (single `Key`, not a list) | pinyin engine | Keys · phrase selection | R |
| `BackSpaceToUnselect` | bool · `true` | — | pinyin engine | Keys · phrase selection | R |
| `FilterByStroke` | `grave` | no | pinyin engine | Keys · more | R |
| `QuickPhraseKey` | `;` | no (single `Key`) | quickphrase addon | Keys · quick phrase | R |
| `VAsQuickphrase` | bool · **`true`** | — | quickphrase addon | Keys · quick phrase | R |

`SecondCandidate`/`ThirdCandidate` are **not** "N-th key selects N-th candidate" mappings; they are the key lists for the
2nd and 3rd candidate actions and are empty by default. Only `ChooseCharFromPhrase` is positional.

### 2.4 Advanced values

| Path | Type · default · constraints | New location | Disposition |
|---|---|---|---|
| `Number of sentence` | int · `2` · 1–3 | Candidates and input · More | R — the stored key is literally `Number of sentence`; `nbest` is only the C++ member name |
| `WordCandidateLimit` | int · `15` · ≥0 (`0` = all) | Candidates and input · More | R |
| `LongWordLengthLimit` | int · `4` · 0–10 | Candidates and input · More | R |
| `QuickPhraseTriggerRegex` | `std::vector<std::string>` of regular expressions · defaults `.(/\|@)$`, `^(www\|bbs\|forum\|mail)\.`, `^(http\|https\|ftp\|telnet\|mailto):` · no constraint beyond `RegexConstrain` per entry | Keys · quick phrase · trigger patterns (one expression per line) | R — a list of patterns, not a key list; each entry is validated as a regular expression and order is preserved |
| `FirstRun` | hidden bool · `true` | — | H |

### 2.5 External and sub-configuration links (exact URIs)

| Path | Kind | URI | Runtime availability | Disposition |
|---|---|---|---|---|
| `DictManager` | `ExternalOption` | `fcitx://config/addon/pinyin/dictmanager` | addon `pinyin` packaged | X — link only; the dictionary tool is not implemented on Windows |
| `CustomPhrase` | `ExternalOption` | `fcitx://config/addon/pinyin/customphrase` | addon `pinyin` packaged | X — link only; tool not implemented |
| `QuickPhrase` | `ExternalOption` | `fcitx://config/addon/quickphrase/editor` | addon `quickphrase` packaged | X — link only; editor not implemented |
| `CloudPinyin` | `SubConfigOption` | `fcitx://config/addon/cloudpinyin` | addon `cloudpinyin` **not** in the packaged addon set | X — link only; the sub-page is unreachable in this runtime and is shown as unavailable |
| `Punctuation` | `SubConfigOption` | `fcitx://config/addon/punctuation/punctuationmap/zh_CN` | addon `punctuation` packaged | X — link only; the punctuation map is a separate schema page |
| `Chttrans` | `SubConfigOption` | `fcitx://config/addon/chttrans` | **not packaged** (`data/fcitx5/addon` has no `chttrans.conf`) | X with a stated reason: shown as unavailable in this runtime, not silently dropped |

A link is not an option value: opening one leaves the current draft and opens another configuration surface.

### 2.6 Measured Windows caveat (carried over, not changed here)

From the earlier key-path measurement (`windows-architecture.md` §22, R36):

- `KP_Enter`-named entries never match, because the TSF path delivers the main `Return` keysym.
- `Control+…` / `Alt+…` entries are not delivered: `routes_key` refuses modified keys, so those bindings are stored but
  cannot fire on this path.
- An idle `Return` is left to the application (R36), so `CommitRawInput`'s plain `Return` only acts while composing.

The editor shows the lists exactly as the engine stores them and must not claim the affected entries take effect.

## 3. Global configuration — `fcitx://config/global`

The schema is two **containers**, not a flat list: `Hotkey` and `Behavior`. Each child below is one option; the containers
are not counted as settings.

### 3.1 `Hotkey` (17 children)

| Path | Default · constraints | Consumer | Delivery on this path | Disposition |
|---|---|---|---|---|
| `TriggerKeys` | `Control+space`, `Zenkaku_Hankaku`, `Hangul` | `instance.cpp:805` key handler | modified keys are not delivered to the engine | C |
| `EnumerateWithTriggerKeys` | bool · `true` | `instance.cpp` trigger path | same | C |
| `ActivateKeys` | `Hangul_Hanja` | `instance.cpp:813` | same | C |
| `DeactivateKeys` | `Hangul_Romaja` | `instance.cpp:816` | same | C |
| `ConsumeRedundantActivateKeys` | bool · `false` | `instance.cpp` activate path | same | C |
| `AltTriggerKeys` | `Shift_L` | `instance.cpp:810` | modifier-only keys are not delivered | C |
| `EnumerateForwardKeys` | empty | `instance.cpp:819` | modified keys | C |
| `EnumerateBackwardKeys` | empty | `instance.cpp:822` | same | C |
| `EnumerateSkipFirst` | bool · `false` | `instance.cpp` enumerate | same | C |
| `EnumerateGroupForwardKeys` | `Super+space` | `instance.cpp:825` | Windows consumes Super+Space | C |
| `EnumerateGroupBackwardKeys` | `Super+Shift+space` | `instance.cpp:828` | same | C |
| `PrevPage` (fallback) | `Up` | `clipboardtempmode.cpp:71`, `imselectortempmode.cpp:116` | unmodified key; only in those temporary modes | R, **separate from the Pinyin `PrevPage`** — not an inheritance layer |
| `NextPage` (fallback) | `Down` | `clipboardtempmode.cpp:86`, `imselectortempmode.cpp:124` | same | R |
| `PrevCandidate` (fallback) | `Shift+Tab` | core temporary-mode consumers | same | R |
| `NextCandidate` (fallback) | `Tab` | core temporary-mode consumers | same | R |
| `TogglePreedit` | `Control+Alt+P` | `instance.cpp` key handler | modified keys | C |
| `ModifierOnlyKeyTimeout` | int · −1–5000 (schema default) | modifier-only hotkey path | no modifier-only delivery | C |

### 3.2 `Behavior` (17 children)

| Path | Default · constraints | Consumer | Disposition |
|---|---|---|---|
| `ActiveByDefault` | bool · `false` | `instance.cpp` activation policy | NV — the TIP's compartment state also decides; interaction not measured |
| `resetStateWhenFocusIn` | enum · `No` | `instance.cpp` focus-in policy | NV |
| `ShareInputState` | enum · `No` on desktop | `instance.cpp` property propagation | NV |
| `PreeditEnabledByDefault` | bool · `true` | `instance.cpp` / input context | NV |
| `ShowInputMethodInformation` | bool · `true` | notifications module | C — `notifications` is not in the packaged addon set |
| `showInputMethodInformationWhenFocusIn` | bool · `false` | notifications module | C — same |
| `CompactInputMethodInformation` | bool · `true` | notifications module | C — same |
| `ShowFirstInputMethodInformation` | bool · `true` | notifications module | C — same |
| `DefaultPageSize` | int · **`5`** · **1–10** | core default page size (input context / temporary modes) | NV — the Pinyin page has its own `PageSize`; the two are different options |
| `OverrideXkbOption` | bool · `false` | XKB path | H — `ConditionalHidden<!hasKeyboard>`; this runtime has no keyboard layouts |
| `CustomXkbOption` | string · empty | XKB path | H — same |
| `EnabledAddons` / `DisabledAddons` | hidden string lists | addon manager | H — upstream hides them |
| `PreloadInputMethod` | hidden bool · `true` | instance | H |
| `AllowInputMethodForPassword` | bool · `false` | `instance.cpp` password detection | NV — the product's password gate is the TIP hard policy; the two were not measured together |
| `ShowPreeditForPassword` | bool · `false` | same | NV |
| `AutoSavePeriod` | int · **`30`** · 0–1440 | `instance.cpp:1336`, `:2134` (instance timer + reload) | R — an actual timer consumer exists; **not** "ineffective because global". Not measured in the product (NV) |

`AutoSavePeriod`, `ActivateKeys`/`DeactivateKeys`/enumeration and the fallback paging options are the corrections to the
earlier blanket claim: each has a real consumer, and only the *event delivery* is limited where stated.

## 4. Cross-cutting rules the client must honour

1. **One draft per underlying option**, keyed by the **addon identity** `(addon:pinyin | addon:<other>, path)` — never by the
   public URI alone. Pinyin and Shuangpin publish different URIs over one configuration object; a stale second copy would revert
   the other view's save. The same rule applies to any future method that shares an addon.
2. **Lists are ordered.** Only `ChooseCharFromPhrase` is positional; the others are ordered alternatives. No sorting or
   de-duplication anywhere.
3. **Empty ≠ default.** `SecondCandidate`/`ThirdCandidate`/`EnumerateForwardKeys` are empty by default; reset restores the
   schema default, clearing writes an empty list.
4. **Only changed options are submitted**; the Host merges into the full `save()` tree. Unknown keys are dropped by Fcitx
   itself (measured in Stage A) — the client must not claim otherwise.
5. **Comma/period presets** edit the official list after explicit confirmation; they are not a boolean and do not merge.
6. **Warnings are computed from the values shown**; no invented priority rules, no silent rewriting.
7. **Per-URI partial success** is reported honestly; no atomic claim across `global` and `pinyin`.
8. **No-change save writes nothing**, so no configuration or learning file is touched.
9. **Links (X) leave the draft**; they never fabricate a value in the parent schema.

## 5. Coverage (recomputed)

`Items` counts every schema field or operation listed above, hidden ones included. The five disposition columns are
**mutually exclusive** and sum to `Items`: X = link to another surface, H = hidden by upstream, C = conditional limit
stated in the row, NV = reachable in the design but explicitly unvalidated, R = reachable with no stated limit.

| Group | Items | X | H | C | NV | R |
|---|---|---|---|---|---|---|
| Input-method operations | 5 | 0 | 0 | 0 | 0 | 5 |
| Pinyin engine / candidates / preedit | 21 | 0 | 0 | 4 | 0 | 17 |
| Fuzzy children | 21 | 0 | 0 | 0 | 0 | 21 |
| Pinyin key options | 16 | 0 | 0 | 1 | 0 | 15 |
| Advanced values | 5 | 0 | 1 | 0 | 0 | 4 |
| External / sub-config links | 6 | 6 | 0 | 0 | 0 | 0 |
| Global `Hotkey` children | 17 | 0 | 0 | 13 | 0 | 4 |
| Global `Behavior` children | 17 | 0 | 5 | 4 | 8 | 0 |
| **Total** | **108** | **6** | **6** | **22** | **8** | **66** |

Reading the totals:

- **6 links**: `DictManager`, `CustomPhrase`, `QuickPhrase`, `Punctuation`, `Chttrans`, `CloudPinyin`. Of these, `Chttrans`
  and `CloudPinyin` point at addons that are **not** in the packaged set and are shown as unavailable.
- **6 hidden**: `FirstRun`; `OverrideXkbOption`, `CustomXkbOption` (conditional on a keyboard layout this runtime lacks);
  `EnabledAddons`, `DisabledAddons`, `PreloadInputMethod` (hidden upstream).
- **22 conditional**: 4 CloudPinyin values (missing addon), `CommitRawInput` (partly: modified and `KP_Enter` entries are
  not delivered), 13 `Hotkey` children (modified keys, Super+Space or modifier-only keys are not delivered), 4
  notifications-related `Behavior` children (addon not packaged).
- **8 not validated**: `ActiveByDefault`, `resetStateWhenFocusIn`, `ShareInputState`, `PreeditEnabledByDefault`,
  `DefaultPageSize`, `AllowInputMethodForPassword`, `ShowPreeditForPassword`, and `AutoSavePeriod` — each has a consumer,
  none was measured in the product.
- Containers (`Fuzzy`, `Hotkey`, `Behavior`) are **not** counted as items; the input-method group is counted as 5
  operations, not as configuration fields.

The headline total is a bookkeeping check, not a product feature: completeness here means every schema item listed above
has a disposition.

## 6. What this inventory still does not establish

- Whether any value takes effect in the product: no runtime test was executed for this client (NV rows).
- Whether the conditional rows can be made to work: that needs either a different delivery path (a TSF routing change,
  out of scope) or the missing addons.
- Editor behaviour: the client is not compiled (`blocker.md`).
