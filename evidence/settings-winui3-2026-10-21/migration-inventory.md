# Migration inventory — Windows settings (C# + WinUI 3)

Status: **design inventory, corrected 2026-10-21.** It records what the schema actually contains and where each item is
meant to live in the new navigation.

**What exists.** The client's **core** is implemented, compiled and tested (fusion-windows
`src/settings-winui3/FusionSettings.Core`): framing, codecs, endpoint naming, connection and handshake, the shared
draft/baseline model, key chords and lists, the key-editor state machine, and the input-method draft. The **WinUI views are
not written**, because `Microsoft.WindowsAppSDK` cannot be restored here (`blocker.md`). So a "disposition" below is a
design decision plus, where the core implements it, a tested behaviour — never a claim that a settings window shows it.

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
external/sub-configuration link to another configuration surface · **H** hidden by upstream, not exposed · **V** present
in the value tree but in no schema, so there is no editable location and the client never writes it · **NV** not validated
(no runtime test executed; the default for anything not measured).

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

**How the core implements it.** `SettingsDraft` holds the **values** — a confirmed baseline and a set of pending edits —
and no schema. Each edit takes the requesting view's snapshot, so the editable-field check is explicitly that view's, and
adopting or refreshing a reply from either view cannot replace the other view's schema. One draft, two schemas: a
Shuangpin-specific field is editable through the Shuangpin view and refused by the Pinyin view, while both share one
pending set and one change set. A field that no view describes (the four `CloudPinyin*` values, `FirstRun`) is therefore not
writable from anywhere.

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
| `PreeditMode` | enum · `ComposingPinyin` on desktop | pinyin engine | Candidates and input · preedit mode | R |
| `PreeditCursorPositionAtBeginning` | bool · `true` | pinyin engine | Candidates and input · preedit cursor | R |
| `PinyinInPreedit` | bool · `false` | pinyin engine | Candidates and input · show full pinyin | R |
| `Prediction` | bool · `false` on desktop | pinyin engine | Candidates and input · prediction | R |
| `KeepCurrentContext` | bool · `true` | pinyin engine | Candidates and input · prediction · more | R |
| `PredictionSize` | int · `49` · 3–100 | pinyin engine | Candidates and input · prediction size | R |
| `BackspaceBehaviorOnPrediction` | enum · `BackspaceWhenNotUsingOnScreenKeyboard` | pinyin engine | Candidates and input · prediction · more | R |
| `SwitchInputMethodBehavior` | enum · `CommitPreedit` | pinyin engine | Keys · action when switching input method | R |
| `Learning` | bool · `true` | pinyin engine | Candidates and input · learning | R |

**Value-tree-only fields (no editable UI location).** Five keys are present in the configuration's value tree but are
described by **no** schema in this runtime, so no view can display them and the core refuses to write them. They are
recorded here so a later schema change is noticed, not to give them a screen:

| Path | Type · default | Why it is not exposed | Disposition |
|---|---|---|---|
| `CloudPinyinEnabled` | bool · `false` | `cloudpinyin` addon is not in the packaged set, so the engine still stores the value but exposes no option for it | **V** — value-tree only; never written |
| `CloudPinyinIndex` | int · `2` · 1–10 | same | **V** |
| `CloudPinyinAnimation` | bool · `true` | same | **V** |
| `KeepCloudPinyinPlaceHolder` | bool · `false` | same | **V** |
| `FirstRun` | bool · `true` | hidden upstream (`HiddenOption`); the first-run flow is not part of this product | **V** |

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
| `NextPage` | `equal`, `Down`, `KP_Down`, `Next` | no | pinyin engine | Keys · paging | R |
| `PrevCandidate` | `Shift+Tab` | no | pinyin engine | Keys · candidate navigation | R |
| `NextCandidate` | `Tab` | no | pinyin engine | Keys · candidate navigation | R |
| `CurrentCandidate` | `space`, `KP_Space` | no | pinyin engine | Keys · candidate navigation | R |
| `CommitRawInput` | `Return`, `KP_Enter`, `Control+Return`, `Control+KP_Enter`, `Shift+Return`, `Shift+KP_Enter`, `Control+Shift+Return`, `Control+Shift+KP_Enter` | no | pinyin engine | Keys · commit | R for the unmodified entries; **C** for the modified and `KP_Enter` entries (§2.6) |
| `SecondCandidate` | **empty list** | **no** | pinyin engine | Keys · candidate selection | R |
| `ThirdCandidate` | **empty list** | **no** | pinyin engine | Keys · candidate selection | R |
| `UseKeypadAsSelection` | bool · `false` | — | pinyin engine | Keys · candidate selection | R |
| `ChooseCharFromPhrase` | `bracketleft` (the source literal is `[`) | **yes** — the N-th binding selects the N-th character of the current candidate (upstream tooltip) | pinyin engine | Keys · phrase selection | R; order is semantic, never sorted or de-duplicated |
| `ChooseLastCharFromPhrase` | `bracketright` (source literal `]`) | no (single `Key`, not a list) | pinyin engine | Keys · phrase selection | R |
| `BackSpaceToUnselect` | bool · `true` | — | pinyin engine | Keys · phrase selection | R |
| `FilterByStroke` | `grave` | no | pinyin engine | Keys · more | R |
| `QuickPhraseKey` | `semicolon` (source literal `;`) | no (single `Key`) | quickphrase addon | Keys · quick phrase | R |
| `VAsQuickphrase` | bool · **`true`** | — | quickphrase addon | Keys · quick phrase | R |

`SecondCandidate`/`ThirdCandidate` are **not** "N-th key selects N-th candidate" mappings; they are the key lists for the
2nd and 3rd candidate actions and are empty by default. Only `ChooseCharFromPhrase` is positional.

**Wire shape (measured, 2026-10-21).** A `List|Key` value is **not** a space-separated string. The Host sends it as an
indexed map — `PrevPage` arrives as `{"0":"minus","1":"Up","2":"KP_Up","3":"Page_Up"}` — and an empty list arrives as the
empty **scalar** `""`. Fcitx marshals a vector as numbered keys, so a client must preserve index order and must accept both
shapes. Key names are X11 keysym names, not source literals: `[` is `bracketleft`, `]` is `bracketright`, `;` is
`semicolon`, and `Page_Down` is `Next` (the same keysym under its canonical name). A UI that compared the source literal
with the wire name would show a binding as changed when the user had not touched it.

### 2.4 Advanced values

| Path | Type · default · constraints | New location | Disposition |
|---|---|---|---|
| `Number of sentence` | int · `2` · 1–3 | Candidates and input · More | R — the stored key is literally `Number of sentence`; `nbest` is only the C++ member name |
| `WordCandidateLimit` | int · `15` · ≥0 (`0` = all) | Candidates and input · More | R |
| `LongWordLengthLimit` | int · `4` · 0–10 | Candidates and input · More | R |
| `QuickPhraseTriggerRegex` | `std::vector<std::string>` of regular expressions · defaults `.(/\|@)$`, `^(www\|bbs\|forum\|mail)\.`, `^(http\|https\|ftp\|telnet\|mailto):` · no constraint beyond `RegexConstrain` per entry | Keys · quick phrase · trigger patterns (one expression per line) | R — a list of patterns, not a key list; each entry is validated as a regular expression and order is preserved |

### 2.5 External and sub-configuration links (exact URIs)

| Path | Kind | URI | Runtime availability | Disposition |
|---|---|---|---|---|
| `DictManager` | `ExternalOption` | `fcitx://config/addon/pinyin/dictmanager` | addon `pinyin` packaged | X — link only; the dictionary tool is not implemented on Windows |
| `CustomPhrase` | `ExternalOption` | `fcitx://config/addon/pinyin/customphrase` | addon `pinyin` packaged | X — link only; tool not implemented |
| `QuickPhrase` | `ExternalOption` | `fcitx://config/addon/quickphrase/editor` | addon `quickphrase` packaged | X — link only; editor not implemented |
| `CloudPinyin` | `SubConfigOption` | `fcitx://config/addon/cloudpinyin` | addon `cloudpinyin` **not** in the packaged addon set, so the option is **not in the runtime schema at all** (measured 2026-10-21) | X — recorded, but there is nothing to show: this view does not expose it |
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

### 3.2 `Behavior` (12 exposed children)

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
| `AllowInputMethodForPassword` | bool · `false` | `instance.cpp` password detection | NV — the product's password gate is the TIP hard policy; the two were not measured together |
| `ShowPreeditForPassword` | bool · `false` | same | NV |
| `AutoSavePeriod` | int · **`30`** · 0–1440 | `instance.cpp:1336`, `:2134` (instance timer + reload) | R — an actual timer consumer exists; **not** "ineffective because global". Not measured in the product (NV) |

`AutoSavePeriod`, `ActivateKeys`/`DeactivateKeys`/enumeration and the fallback paging options are the corrections to the
earlier blanket claim: each has a real consumer, and only the *event delivery* is limited where stated.

**Five more `Behavior` fields exist in the source but are not in the runtime schema** (measured 2026-10-21):
`OverrideXkbOption` and `CustomXkbOption` (`ConditionalHidden` — this runtime has no keyboard layouts),
`EnabledAddons`/`DisabledAddons` and `PreloadInputMethod` (hidden upstream). A client never sees them, so there is nothing
to show and nothing to write; they are recorded here so a later schema change is noticed rather than guessed at.

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

## 5. Coverage (recomputed from the runtime schema)

The earlier totals were counted from the source and did not match what a Host actually returns. These are counted from a
schema dump of a running Host (2026-10-21, isolated pipe, `FUSION_FCITX_SKIP_USER_PATH=1`).

`Items` counts every field or operation a client must give a disposition to. Containers (`Fuzzy`, `Hotkey`, `Behavior`) are
**not** items: they are navigation, and the rows below already include their children. The four disposition columns are
mutually exclusive and sum to `Items`: X = link to another surface, C = conditional limit stated in the row, NV = reachable
but explicitly unvalidated, R = reachable with no stated limit.

| Group | Items | X | C | NV | R |
|---|---|---|---|---|---|
| Input-method operations | 5 | 0 | 0 | 0 | 5 |
| Pinyin/Shuangpin options (union of both views) | 37 | 0 | 1 | 0 | 36 |
| Fuzzy children | 21 | 0 | 0 | 0 | 21 |
| External / sub-configuration links | 5 | 5 | 0 | 0 | 0 |
| Global `Hotkey` children | 17 | 0 | 13 | 0 | 4 |
| Global `Behavior` children | 12 | 0 | 4 | 8 | 0 |
| **Total** | **97** | **5** | **18** | **8** | **66** |

Reading the totals:

- **The two Pinyin views expose different schemas over one value tree.** The pinyin view describes 41 options, the
  shuangpin view 43 (it adds `ShuangpinProfile` and `ShowShuangpinMode`), and both return the **same 43-key value tree**.
  The union is 43, of which 5 are links and 1 is the `Fuzzy` container, leaving 37 fields. This is why a draft is keyed by
  the addon, not by the URI: two URI-keyed drafts of one value would diverge.
- **5 links**: `DictManager`, `CustomPhrase`, `QuickPhrase`, `Punctuation`, `Chttrans`. `Chttrans` is described by the
  schema even though its addon is not packaged, so it is shown as unavailable rather than dropped. `CloudPinyin` is **not**
  in the runtime schema at all, so it is not one of these five.
- **1 conditional Pinyin field**: `CommitRawInput` — the modified and `KP_Enter` entries are stored but not delivered on
  this path (§2.6).
- **13 conditional `Hotkey` children**: modified keys, `Super+space` and modifier-only keys are not delivered.
- **4 conditional `Behavior` children**: the `notifications` addon is not packaged.
- **8 not validated**: `ActiveByDefault`, `resetStateWhenFocusIn`, `ShareInputState`, `PreeditEnabledByDefault`,
  `DefaultPageSize`, `AllowInputMethodForPassword`, `ShowPreeditForPassword`, `AutoSavePeriod` — each has a consumer, none
  was measured in the product.

**Five keys are in the value tree but in no schema** and are therefore **not counted** in the table: the four
`CloudPinyin*` values and `FirstRun`. They carry disposition **V** and are listed in §2.1 with no editable location. The
client must never write them — they are not visible in any view, so a write would change a setting the user could not see.
The draft layer enforces this structurally: an edit is only accepted against the requesting view's schema, so a field no
view describes is refused by every view.

The row counts add up as they should: §2.1 has 17 option rows, §2.3 has 16, §2.4 has 4 — 37 in the Pinyin/Shuangpin group.

The headline total is a bookkeeping check, not a product feature: completeness here means every item above has a
disposition.

## 6. Coverage as implemented

**Completion level of the client, by layer:**

| Layer | State |
|---|---|
| design (this document) | done |
| core (`src/settings-winui3/FusionSettings.Core`) | implemented, compiled, tested |
| WinUI views (`src/settings-winui3/FusionSettings.App`) | **implemented, compiled, launched** — four pages, every item below has a location |
| packaged | yes — `fusion-0.0.0-winui3+67f771b.zip` |
| installed | **no** |
| owner-accepted | **no** |

**Where each group landed.** The accepted navigation is Input methods · Keys · Candidates and input · About and
diagnostics, and every item in §1–§3 has a place in it:

| Group | Items | Where it is now |
|---|---|---|
| Input-method operations | 5 | Input methods: the enabled list with ↑/↓/remove, the available list with add, `Items/0` marked 当前 |
| Pinyin/Shuangpin options | 37 | Keys: 翻页 · 候选导航 · 候选选择 · 上屏 · 短语选字 · 快捷短语 · 学习（忘记词语）· 更多设置. Candidates and input: 候选 · 预编辑 · 学习 · 双拼 · 更多设置 |
| Fuzzy children | 21 | Candidates and input → 模糊音, one section for the nested container |
| External links | 5 | Candidates and input → 外部工具, listed with the reason they are unavailable |
| Global `Hotkey` children | 17 | Keys → 系统快捷键 and 系统回退翻页 |
| Global `Behavior` children | 12 | Candidates and input → 系统输入行为 |
| Value-tree-only (`V`) | 5 | no location, by design — no view describes them and the draft refuses to write them |

- **The engine's paging keys and the system's fallback paging keys are separate sections** with the difference stated in
  the UI, because they are two different options with two different consumers.
- **The table is enforced, not advisory.** Each page renders only the rows whose placement names it; an option the table
  does not place falls back to 更多设置 of its own configuration, so a Host that adds a field never becomes invisible and is
  never duplicated. Before the correction round the page was not consulted at all, so every page rendered every root option.
- **Editable means editable through the asking view's schema.** A row's setter is bound to the configuration that page
  renders, so a Global control cannot reach the Pinyin draft, and the save splits the shared draft across the URIs whose
  schemas describe its fields.
- **Checks**: **422** with an isolated Host (347 without one). The workspace path is driven against a real Host — Global
  routing, a Shuangpin-only save, clearing the last binding, the method list, a failed start and its retry — and one edit was
  driven through the compiled window: a control toggled, Save pressed, and the Host read back independently.
- **A successful global save is NOT RUN on this Host.** The isolated Host runs without a user config path, and the Host's
  global apply ends in `safeSave()`, which needs one; it answers "Fcitx could not save the global configuration; reload
  before retrying." What is checked instead is that the client reports that failure and keeps the edit.
- **Not established**: whether any individual value changes product behaviour (the NV rows). That needs the client used
  against the product Host, which is an owner step.
- **Not established**: whether the conditional rows can be made to work. That needs either a different delivery path (a TSF
  routing change, out of scope here) or the missing addons.
- **Not claimed**: that storing or rereading a paging binding proves the key pages candidates through a TSF application.
  Those are different claims and only the first was checked. Also not claimed: that a stable-incarnation check exercises
  every fencing failure, or that an in-memory reread is restart persistence.
