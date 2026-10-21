# Stage A handoff — global settings + input-method management (Windows validation executed)

Scope: D082 / D085 / D086. Base governance: `fusion-windows/AGENTS.md`, `fusion-controller`
`windows-architecture.md` (D072 §4/§5/§9/§13/§15/§22/§23/§25), `windows-stage6-plan.md`,
`windows-stage6-wp3.md`, `provider-settings-acceptance.md`, `docs/windows-settings-ui-feasibility.md`.
Bounded contract: `change-contract.md` (same directory). Capability mapping and the Shift/raw-input
account: `capability-mapping.md`.

Status: **implemented, built in Release on Windows 7700, validated against an isolated Host + GUI and
packaged.** This file started as a minis-side handoff and is now the closing record; §6 states which
Windows checks were executed and which were not. It is **not** an acceptance: no merge was performed
on any repository, nothing was installed or registered, and the owner's production Host, tray,
browser and profile were not modified. The live key-down/key-up check through a TSF-registered
application was **not run**.

## 1. Repositories, branches, bases, heads, PRs

| Repo | Role | Branch | Base commit | Head commit | Draft PR |
|---|---|---|---|---|---|
| `choicky/fusion-windows` | product | `feature/stage-a-global-settings-im-methods` | `fe23f07611707c2d70aaf16a2ece282057ff8097` (origin/main, PR #28 merged) | `fe65d08621fb84f47ef83acfc3e50835217af462` (feature `0a965e2` + correction `fb62348` + packaging-pin `2e28a88` + closure `fe65d08`) | https://github.com/choicky/fusion-windows/pull/29 |
| `choicky/fcitx5-configtool` (owner fork) | reusable widget-layer contract | `windows/fusion-settings` | `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b` (upstream base) | `abc2c6644b030d761d3d5e7c27c888af019e63f0` (unchanged this batch — no new commit) | https://github.com/choicky/fcitx5-configtool/pull/2 |
| `choicky/fusion-controller` | docs/authority + this evidence | `feature/stage-a-global-settings` | `96fb28c2bca1cc400361fc2377409e668201054f` (origin/main, PR #51 merged) | PR #52 branch tip (this evidence pack; `d25c39c` = first commit, `8ca1282` = Draft index, this batch on top) | https://github.com/choicky/fusion-controller/pull/52 |

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

## 4. Shift / raw-input commit — what the source does

`CommitRawInput` is a **per-input-method** Pinyin `KeyListOption` on
`fcitx://config/inputmethod/pinyin`, **not** a global option. This stage reuses that official option
as delivered; the Pinyin page already routes its Shift+Return / plain-Return family to the engine
while a composition is active.

Three separate things were conflated in the earlier drafts of this record, and are kept apart here:

- **`Shift+Return` commits raw input** because the option lists it. It is **not** a lone `Shift`
  trigger and must never be described as one.
- **Lone `Shift` is NOT implemented.** A bare modifier never reaches the engine: modified keys are
  routed as Pass by the D072 §5 UNFROZEN "production effect-capable key classification", and no
  Shift state machine was added. Nothing in `src/tsf/` changed in this batch
  (`git diff fe23f07..HEAD -- src/tsf/` is empty).
- **Numpad Enter is not a separate case.** Windows delivers it as `VK_RETURN`, so it matches the
  ordinary `Return` family and the `KP_Enter` entries in the key lists never match. No keysym
  translation was added, and none is needed for the option to behave as documented.

`Control`/`Alt`/`Win`-modified keys share the same Pass routing. Ordinary Shift+letter, key-up and
modifier handling are unchanged. Routing a modifier-held key or a bare-Shift trigger to the engine
would be the D072 §5 / WP3-B6 (R19) boundary and, under D082, needs evidence that mature Fcitx
behaviour is otherwise a defect — not implementation here. Details in `capability-mapping.md` §1.
The Windows key-down/key-up check through a real TSF-registered application was **not run**, so this
section is a source reading, not a measurement.

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

Qt / Fcitx / Host / fork files could not be built on minis (Qt 6.8 + Fcitx headers + MSVC). That
limitation was recorded as source-reviewed/compile-unverified and is now **superseded**: those files
were compiled in Release and exercised on Windows 7700 (§6).

## 6. Windows validation checks — results on machine 7700

The list below was the handoff; each item now carries its result. Linux/mock results are not Windows
validation and are not counted here.

1. MSVC **Release** build of the settings executable + Host (fork pinned at `abc2c66`) — **EXECUTED**:
   `FusionSettings.exe`, `FusionHost.exe`, `FusionTray.exe`, `FusionTsf.dll` and
   `FusionFcitxHost.dll` built with MSVC 14.51.36231, SDK 10.0.26100.0, Qt 6.8.3 and
   `runtime-wp3-20261018`; `ninja` then reports no work to do.
2. Default **and** explicit isolated per-user pipe connection — **EXECUTED** on an explicit unique
   pipe (`fusion-stage-a-20261021k`); the default-endpoint normalization was corrected and validated
   in PR #28 and is unchanged here.
3. Global page: load / apply / read-back, and persistence across an orderly Host restart —
   **EXECUTED and passing** (a `Behavior/DefaultPageSize` probe value applied and read back; the
   restart check re-read all three pages from the profile).
4. No-change Apply short-circuit; Cancel/reload; unknown + learning-data preservation — **EXECUTED**, split
   by what each check actually proves:
   - **No-change Apply:** the profile tree hash was identical either side of the pass, and the list
     apply reported no re-save.
   - **Cancel/Reload on the Global page:** Cancel discarded an unsaved edit, restored the Host's value
     and refreshed the baseline; Reload fetched a value a second client had changed on the Host behind
     the page (7 → 9 while the page was dirty) and refreshed the baseline; switching tabs kept an
     ordinary unsaved edit. These are the assertions the buttons' own code path performs.
   - **Configuration the apply does not submit:** seeded `Behavior/ShowInputMethodInformation=False`
     and pinyin `SpellEnabled=False` both survived. A key no schema knows
     (`FusionStageAUnknownProbe`) did **not** survive in either file: Fcitx's own save writes the
     schema, so unknown keys are dropped. That is Fcitx semantics, and "unexposed values are
     preserved" was only ever true of schema-known options this UI does not submit.
   - **Learning data:** `pinyin/user.dict` (339 B) and `user.history` (61 B), produced by the
     integration run in the same disposable profile, were byte-identical (SHA-256) before and after
     the settings-only session. File existence is not used as evidence anywhere in this record.
5. Input-method management: add / remove / reorder / select persistence; invalid-choice
   (empty / unavailable) handling — **EXECUTED and passing** (reorder → remove → add-back ended on a
   changed order that survived the restart; empty and `fusion-unavailable-probe` were both refused
   with the documented message).
6. Pinyin/Shuangpin shared-value regression — **EXECUTED and passing**, as Pinyin/Shuangpin. The
   scenario loads both per-input-method pages, applies a change on one, and asserts the other is not
   reverted and shows the new value once applied or reloaded. It is **not** a Global-page-versus-input-
   method test: nothing in this batch exercises that pairing, and no scenario was invented to justify
   the name.
7. Composition refusal / retry on any apply/switch during an active composition — **NOT RUN**; it
   needs a live composing context and was not driven through a TSF-registered application.
8. Shift / raw-input commit and **ordinary Shift+letter** through the real Windows key-down/key-up
   path — **NOT RUN**. No keyboard behaviour is claimed as measured. Lone `Shift` remains
   unimplemented (§4).

`ctest` over `stage_a_settings` (protocol/bounds/validator), `wp3_input_methods`, `rg0` and `stage2`
is **37 of 37 passed**, with `stage4a_ipc_integration` reported by CTest as **skipped**. That mode now
requires an explicit isolated endpoint (`FUSION_STAGE4A_INTEGRATION_PIPE`) and prints `SKIP` instead of
connecting without one, because it types into whichever Host owns the endpoint it is given and the
default endpoint is the installed product Host; the CTest entry carries `SKIP_REGULAR_EXPRESSION` so a
skip is reported as a skip rather than as a successful integration run. Run through
`tools/settings/run-isolated-stage-a.ps1` — which starts a Host on a unique pipe with a disposable
profile and hands the endpoint over — the unchanged assertions pass:
`PASS integration with real Fcitx over IPC` and
`PASS integration candidate select committed the projected candidate`, in 0.4 s. Its earlier failure
was the production endpoint it was fixed to, not a product defect, and against a controlled profile it
passes; that is the result this record keeps. The script also proves the safe default (no endpoint →
`SKIP`, exit 0), the refusal of this user's production endpoint in both the logical-name and full-path
spellings before any request, the refusal of a malformed endpoint, and — by fingerprinting the owner's
learning files around those attempts — that the refusal never reached the production Host.

Windows entry points: Host handlers `handle_settings_get_im_list` / `handle_settings_set_im_list`
(`src/host/host_server.cpp`, gated by `settings_capable`, `close_after_settings_request = true`,
`envelop_response` incarnation fencing); backend `settings_get_im_list` / `settings_set_im_list` +
global branch (`src/fcitx/backend.cpp`); GUI Global tab + IM-management panel (`src/settings/main.cpp`);
payload codecs (`include/fusion/settings_protocol.hpp`, `src/settings/dbusprovider.cpp`).

Running the real-engine test, and the safe default: `stage4a integration` reads
`FUSION_STAGE4A_INTEGRATION_PIPE`, prints `SKIP integration (...)` and exits 0 when it is unset, and
throws before the first request when the endpoint is this user's production pipe. `ctest -R
stage4a_ipc_integration` therefore reports *skipped* unless the build was configured with
`-DFUSION_STAGE4A_INTEGRATION_PIPE=<isolated name>`. The supported way to run it for real is
`tools/settings/run-isolated-stage-a.ps1 -BuildDir <build> -ProfileRoot <disposable> -PipeName <unique>`,
which starts the Host itself, passes the endpoint, stops it through the named stop event and prints a
JSON result.

## 7. Packaging and remaining work

- **Packaged.** `fusion-0.0.0-settings+fe65d08.zip`, 60,085,623 bytes, SHA-256
  `5db4ab21c80a9ffd6aa68c880312a1c68c4b1c5f1a6fb71f78869aecb1d95e5d`, built by the committed packaging
  script from the clean committed source `fe65d08621fb84f47ef83acfc3e50835217af462`
  (`sourceDirty=false`). All 148 manifest entries were re-verified after extraction, along with the
  Qt platform plugin, the executable/DLL identities, the zh_CN catalogs, the non-nested
  `data\locale` layout and the notice/corresponding-source set. The manifest's settings block reads
  its repository, branch and commit from `cmake/settings-ui/versions.cmake` instead of a literal,
  which is what had left a stale `0234c64` in it. This batch changed `src/settings/main.cpp`, which
  ships inside `FusionSettings.exe`, so the package was rebuilt from the new head rather than reused;
  `fusion-0.0.0-settings+2e28a88.zip` (60,076,754 bytes) and `+5271848` (60,052,053 bytes) are retained
  for rollback. **Not installed, not registered, not owner-accepted.**
- **EXE installer** remains a separate later item (D085 / U4). U4 signing + distribution channel,
  U6 real-window confirmation, U7 tray hang-detection follow-up, WP5/WP7/P0-A/P0-B/P0-D are not
  closed by this stage.
- The D072 §5 modifier/bare-Shift classification decision is the only functional boundary left open
  by the raw-input request; it needs owner evidence/decision, not implementation here.

## 8. Unrelated state preserved

No commit to `main`; `feature/*` naming per D076. Nothing installed/registered; production services
untouched. Every settings write in this batch went through a unique pipe to an isolated Host with a
disposable profile, so no Stage A apply targeted the owner's configuration, and no production
Host/tray/browser was stopped or restarted.

**A test side effect that already happened, and the fix.** Until this batch the repository's
`stage4a_ipc_integration` connected to the default endpoint, so running the suite typed through the
installed Host and appended to that user's pinyin history. It did so during the previous batch on this
machine. The side effect is recorded rather than erased: the owner's
`%APPDATA%\Fcitx5\pinyin\user.dict` and `user.history` carry those entries and were **not** restored,
because they may also contain legitimate owner changes. The mode now requires an explicit isolated
endpoint, skips without one, and refuses this user's production endpoint in both spellings before
sending anything; `tools/settings/run-isolated-stage-a.ps1` is the supported way to run it, and it
fingerprints the owner's learning files around the refusal checks to show they stayed untouched.

**One observation without an attribution.** The owner's `pinyin.conf` held `PageSize=9` where an
earlier state held 7. Nothing in this batch establishes what wrote it: no Stage A code writes 9 outside
a disposable profile, and the installed settings window, an earlier package and the owner's own use are
all equally consistent with the file. It is recorded as unattributed and was not modified.

Only the three intended task branches were pushed.

## 9. Artifacts in this directory

- `change-contract.md` — bounded Stage A change contract, with the executed outcome appended.
- `correction-inventory.md` — C1–C9 (the consolidated correction batch), C10/C11 (this closure batch),
  and what was deliberately not changed.
- `capability-mapping.md` — global/IM option-by-option mapping and the Shift/raw-input account.
- `isolated-stage-a-result.json` — the JSON `tools/settings/run-isolated-stage-a.ps1` printed for the
  final run (`fusion-stage-a-20261021u`, profile `profile-stage-a-20`): integration PASS, the four
  Cancel/Reload assertions, the skip/refusal results, and the before/after fingerprints.
- `diffs/fusion-windows-stage-a.diff` — cumulative product diff `fe23f07..fe65d08` (the real diff, not
  a summary).
- `diffs/fusion-windows-commits.txt` — product commit list.
- `diffs/fcitx5-configtool-stage-a.diff` — the Stage A increment in the fork, `0234c64..abc2c66`
  (header-only). The full upstream-to-fork adaptation is the 10-file range `9bd6022..abc2c66` on
  fork PR #2; this batch added nothing to the fork.
- The cumulative controller diff is saved outside the repository, under the persistent task workspace
  (`C:\Users\choicky\worktmp\fusion-windows\official-settings-ui\`), because a diff of this pack
  cannot contain itself.
- `handoff.md` — this record.
