# Stage A handoff — global settings + input-method management (Windows validation pending)

Scope: D082 / D085 / D086. Base governance: `fusion-windows/AGENTS.md`, `fusion-controller`
`windows-architecture.md` (D072 §4/§5/§9/§13/§15/§22/§23/§25), `windows-stage6-plan.md`,
`windows-stage6-wp3.md`, `provider-settings-acceptance.md`, `docs/windows-settings-ui-feasibility.md`.
Bounded contract: `change-contract.md` (same directory). Capability mapping and the Shift/raw-input
resolution: `capability-mapping.md`.

Status: **implementation complete on the Linux dev machine (minis); Windows/MSVC build and all
real-Host validation NOT RUN — this is a handoff for Windows validation on machine 7700, not an
acceptance.** No merge performed on any repository. Nothing installed or registered. The owner's
live input-method configuration was not touched.

## 1. Repositories, branches, bases, heads, PRs

| Repo | Role | Branch | Base commit | Head commit | Draft PR |
|---|---|---|---|---|---|
| `choicky/fusion-windows` | product | `feature/stage-a-global-settings-im-methods` | `fe23f07611707c2d70aaf16a2ece282057ff8097` (origin/main, PR #28 merged) | `0a965e20957022f578526cc86a3314b9fb3be717` | https://github.com/choicky/fusion-windows/pull/29 |
| `choicky/fcitx5-configtool` (owner fork) | reusable widget-layer contract | `windows/fusion-settings` | `0234c64e93f1d61badc1fd3eaad2ba786ca63850` (merged PR #1 base) | `abc2c6644b030d761d3d5e7c27c888af019e63f0` | https://github.com/choicky/fcitx5-configtool/pull/2 |
| `choicky/fusion-controller` | docs/authority + this evidence | `feature/stage-a-global-settings` | `96fb28c2bca1cc400361fc2377409e668201054f` (origin/main, PR #51 merged) | PR #52 branch tip (this evidence pack; `d25c39c` = first commit) | https://github.com/choicky/fusion-controller/pull/52 |

Dependency order followed: fork commit (`abc2c66`) first, then product re-pin + commit (`0a965e2`),
then controller docs/evidence. The product `cmake/settings-ui/versions.cmake`
`FUSION_SETTINGS_CONFIGTOOL_COMMIT` is re-pinned to `abc2c6644b030d761d3d5e7c27c888af019e63f0`
(`FUSION_SETTINGS_CONFIGTOOL_BASE_COMMIT` stays `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b`).

Pinned upstream sources used for the mapping (unchanged, reference only): Fcitx5 core
`a7f2528773ebfec98a20c49b6eea2333110b1c88`, fcitx5-qt `0285a5d…`, chinese-addons `61474bd…`,
libime `171edcf…`. Local pinned core checkout: `/home/choicky/coding-projects/fcitx5-upstream-empty-range-preflight`.

## 2. What this stage delivers (vs the prior Pinyin/Shuangpin subset delivery)

Prior delivery (credited, unchanged): the official-widget settings surface over the per-user named
pipe serving the Pinyin/Shuangpin **per-input-method** configuration subset, owner-test package
`fusion-0.0.0-settings+5271848.zip`.

Stage A adds two capabilities on that same surface, reusing official Fcitx schemas/widgets/config
and management APIs — **not** a new settings framework:

1. **Global configuration** (`fcitx://config/global`).
2. **Input-method list management** over the methods the packaged runtime actually defines
   (currently pinyin, shuangpin): ordered enabled list, add-from-available, remove, reorder
   (front = active), with the active/fallback semantics driven by Fcitx's group model.

The two-method dropdown is **replaced** by the management panel (no competing manager). No new
input engine, no general addon/dictionary management, no cosmetic redesign, no EXE installer.

## 3. Source-backed design decisions

- **Global served over the existing GetConfig/SetConfig messages** — zero new message types for the
  global page. The Host branches on the URI (`is_global_config_uri`): global read =
  `globalConfig().save(values)` + `dumpSettingsDescription(global.config())`; global apply =
  read → `merge_schema_values` → **no-change short-circuit** (no setter call, no learning touch) →
  `has_active_composition_on_loop` Busy guard → `settings_value_fill_raw_config` →
  `globalConfig().load(raw, /*partial=*/true)` + `safeSave()` + `reloadConfig()` → re-read. This is
  exactly Fcitx's own global-apply semantics. XKB and other Linux-only options are already
  self-hidden upstream via `ConditionalHidden<!hasKeyboard>` / `PreloadInputMethod` etc.; the official
  schema is reused, they are not re-exposed.
- **Separate `ImList` model + additive messages** (`SettingsGetImListRequest` 0x15 / `…Response`
  0x16, `SettingsSetImListRequest` 0x17 / `…Response` 0x18) rather than mutating the tested
  `SettingsSelection`, so the existing MSVC round-trip test cannot break and the new serialization is
  Linux-testable. `settings_api_version` 1 → 2; `max_message_type` now `SettingsSetImListResponse`.
- **`ImList`** = `{active, fallback, enabled[], available[]}` with `active = enabled.front()`
  (Items/0), `fallback = DefaultIM`, remove ≠ delete (never removes a method file or learning data).
- **Pure validator** `src/fcitx/im_list.hpp` `plan_im_list_update(...)`: rejects empty, rejects any
  name not in `available`, de-duplicates keeping first order, and passes the previous fallback
  straight through to `setDefaultInputMethod` — the validator deliberately does **not** reimplement
  Fcitx's own `Items/0 == DefaultIM` rewrite invariant (confirmed at pinned core
  `inputmethodgroup.cpp:93-114`), per D082 "reuse, don't reimplement". Official `setGroup` silently
  drops unknown names and does not reject empty; the validator closes exactly those two gaps.
- **Host IM-list path** over `currentGroup()`/`setGroup()`/`save()`: rebuild the group item list
  preserving per-method layout via `current_group.layoutFor(name)`, apply the validated order, set
  fallback, save, re-read. No-change short-circuit + composition guard + all engine touch scheduled
  onto the Fcitx event loop through the existing `invoke_settings`.
- **Available set** = `{pinyin, shuangpin} ∩ defined-entry`, matching the existing selection path,
  rather than `foreachEntries` (which would leak keyboard-layout entries). Refuse-unavailable is the
  safety net for a stale/Linux-authored profile naming a method this runtime does not define.
- **Fork scope** is header-only (the shared `ControllerProxy`/`PendingCall`/`SettingsImList`
  declarations). The product compiles its own `src/settings/dbusprovider.cpp` and only includes the
  fork header, so there is no duplicate-symbol conflict; the fork's merged `dbusprovider.cpp` is not
  a product-compiled unit.

## 4. Shift / raw-input commit — resolved against source, not assumed

`CommitRawInput` is a **per-input-method** Pinyin `KeyListOption` on
`fcitx://config/inputmethod/pinyin`, **not** a global option. This stage reuses that official option
as delivered; the Pinyin page already routes its Shift+Return / plain-Return family to the engine
while a composition is active.

**Hard-stop boundary (reported, not implemented):** routing a **modifier-held** key
(Ctrl/Alt/Win + key, including `Control+Return`), a **numpad Enter** variant, or a **bare-Shift**
trigger to the engine is the D072 §5 UNFROZEN "production effect-capable key classification" / WP3-B6
(R19) boundary and, under D082, would require evidence that mature Fcitx behavior is otherwise a
defect. This stage does **not** change `routes_key`/`classify_key` ownership, does not add a Shift
state machine, and leaves ordinary Shift+letter, key-up and modifier handling unchanged. Details in
`capability-mapping.md` §1.

## 5. Exact tests run on minis (Linux) and results

Executed and passing here (pure-logic, non-`semantic.cpp`, non-Fcitx-header family):

- `tests/stage_a_settings.cpp` — **new**, builds under `-Isrc -Iinclude -Wall -Wextra -Wpedantic -Werror`
  (GCC), registered as CTest `stage_a_settings` with three modes:
  - `protocol` — ImList field round-trip; SetImList empty + round-trip; static assert
    `max_message_type == SettingsSetImListResponse`. **PASS.**
  - `bounds` — over-bound list write refused; forged count fails; trailing-byte detectable;
    truncated read fails. **PASS.**
  - `validator` — reorder + de-dupe accepted; empty refused; unavailable refused; single-method
    accepted. **PASS.**
- `wp3_input_methods`, `rg0`, `stage2` — still **PASS** (regression check, unchanged).
- `FusionCore`, `MockBackend`, `FusionMock` — still **compile** on minis.

Pre-existing minis validation debt (NOT a Stage A change, NOT fixed here): the shared
`semantic.cpp` / `stage5_candidates` family fails to build on this machine's GCC 15.2 with
`-Werror=missing-field-initializers` at base `fe23f07` (MSVC-tuned code). Recorded only.

Qt / Fcitx / Host / fork files are **STATIC REVIEW PASS / COMPILE UNVERIFIED** on minis: they require
Qt 6.8 + Fcitx headers + MSVC and are not built here.

## 6. Required Windows validation checks — NOT RUN here (machine 7700)

All of the following are the Windows/MSVC handoff. They must be run before this lands; Linux/mock
results above are **not** Windows validation.

1. MSVC **Release** build of the settings executable + Host (fork re-pinned to `abc2c66`).
2. Default **and** explicit isolated per-user pipe connection.
3. Global page: load / apply / read-back, and persistence across an orderly Host restart.
4. No-change Apply short-circuit; Cancel/reload; unknown + learning-data preservation.
5. Input-method management: add / remove / reorder / select persistence; invalid-choice
   (empty / unavailable) handling.
6. Cross-page stale-value regression (global page vs IM page).
7. Composition refusal / retry on any apply/switch during an active composition.
8. Shift / raw-input commit and **ordinary Shift+letter** through the real Windows
   key-down/key-up path.

Windows entry points: Host handlers `handle_settings_get_im_list` / `handle_settings_set_im_list`
(`src/host/host_server.cpp`, gated by `settings_capable`, `close_after_settings_request = true`,
`envelop_response` incarnation fencing); backend `settings_get_im_list` / `settings_set_im_list` +
global branch (`src/fcitx/backend.cpp`); GUI Global tab + IM-management panel (`src/settings/main.cpp`);
payload codecs (`include/fusion/settings_protocol.hpp`, `src/settings/dbusprovider.cpp`).

## 7. Packaging / remaining work (not in this stage)

- **Packaging pending.** No new package identity; the current available owner-test package remains
  `fusion-0.0.0-settings+5271848.zip`. A Stage A package is produced only after Windows validation.
- **EXE installer** remains a separate later item (D085 / U4). U4 signing + distribution channel,
  U6 real-window confirmation, U7 tray hang-detection follow-up, WP5/WP7/P0-A/P0-B/P0-D are not
  closed by this stage.
- The D072 §5 modifier/bare-Shift classification decision is the only functional boundary left open
  by the raw-input request; it needs owner evidence/decision, not implementation here.

## 8. Unrelated state preserved

No commit to `main`; `feature/*` naming per D076. Nothing installed/registered; production services
untouched; owner's live IM config untouched. Only the three intended task branches were pushed.

## 9. Artifacts in this directory

- `change-contract.md` — bounded Stage A change contract.
- `capability-mapping.md` — global/IM option-by-option mapping + the Shift/raw-input resolution.
- `diffs/fusion-windows-stage-a.diff` — cumulative product diff `fe23f07..0a965e2` (this is the real diff, not a summary).
- `diffs/fusion-windows-commits.txt` — product commit list.
- `diffs/fcitx5-configtool-stage-a.diff` — cumulative fork diff `0234c64..abc2c66`.
- `handoff.md` — this record.
