# Stage A Change Contract — global settings + input-method management

Per `fusion-windows/AGENTS.md` §9.2. Bounded contract for the Stage A implementation instruction.
Base: `fusion-windows` origin/main `fe23f07` (PR #28 merged); `fusion-controller` origin/main
`96fb28c` (PR #51 merged); fork `choicky/fcitx5-configtool` `windows/fusion-settings` `abc2c66`
(fork PR #2; the pin the build and the package use).

## Goal
Extend the existing official-widget settings surface (`FusionSettings.exe` → `FusionHost` over the
per-user named pipe) from the previously delivered Pinyin/Shuangpin per-input-method configuration
subset to (1) **global configuration** (`fcitx://config/global`) and (2) **input-method list
management** over the methods the packaged runtime actually defines, reusing Fcitx5's official
schemas, widget, configuration and management semantics. Add the directly-necessary Windows event
adaptation for the reused `CommitRawInput` option only where it does not touch the UNFROZEN key
classification.

## Deliverable
- Product `fusion-windows`: Host handlers + Fcitx backend + settings protocol + GUI shell that serve
  the global page and a coherent input-method list-management page; pure-logic tests that build and
  run on minis; README + doc reconciliation.
- Fork `fcitx5-configtool`: the reusable provider/model declarations the new surface needs
  (`ControllerProxy` method + the list-management reply type) on `windows/fusion-settings`; then the
  product re-pins the exact fork commit.
- Controller `fusion-controller`: this evidence pack, plus reconciled Stage-6/settings status.
- One Draft PR per repository actually changed. No merge (owner decision). Windows build/validation
  and packaging are **NOT RUN** on minis.

## Authority / Inputs
D072 (`windows-architecture.md`, §4/§5/§9/§13/§15/§22/§23/§25), D082, D085, D086, WP3
(`windows-stage6-wp3.md`), `windows-settings-ui-feasibility.md`, `provider-settings-acceptance.md`,
`evidence/settings-ui-spike-2026-10-20/`, `AGENTS.md` (both). Pinned source as listed in
`capability-mapping.md`. `settings_api_version` and the settings-message set in
`include/fusion/ipc_protocol.hpp` + `settings_protocol.hpp`.

## Allowed Scope
- `include/fusion/settings.hpp`, `settings_protocol.hpp`, `ipc_protocol.hpp` (settings messages),
  `include/fusion/backend.hpp` (settings virtuals), `src/fcitx/backend.cpp` (global read/apply +
  IM-list get/set), a new pure-logic `src/fcitx/im_list.hpp`, `src/host/host_server.{hpp,cpp}`
  (dispatch + capability/api-version gate), `src/settings/{main.cpp,dbusprovider.cpp}` (Global tab +
  IM-management page + new controller ops), `cmake/settings-ui/versions.cmake` (fork re-pin),
  `tests/` (new pure-logic tests + their registration).
- Fork: `src/lib/configlib/dbusprovider.h` (+ the matching `.cpp` default) and, if the list type is
  reused by ConfigWidget signals, the minimal model header — nothing else in the widget layer.
- Docs: this evidence pack; `windows-settings-ui-feasibility.md` status pointer;
  `windows-stage6-plan.md` settings/IM row; `fusion-windows/README.md` settings section.

## Must Preserve
- Endpoint normalization (`logical_pipe_name`/`compute_pipe_name`), user/session identity, Host-
  incarnation + connection-generation fencing on every settings reply (`envelop_response`).
- TIP compatibility; the settings capability handshake and explicit `settings_api_version`/protocol
  version handling; one-settings-request-per-connection; async GUI thread (GUI never blocks on pipe).
- Unknown/unexposed value preservation and the **no-change Apply** short-circuit (no setter call, no
  learning-file touch) already in `apply_settings_on_loop`.
- Composition/pending-state protection (`has_active_composition_on_loop`) before any apply/switch.
- Every engine touch scheduled onto the Fcitx event loop via the existing `invoke_settings`.
- The official Fcitx semantics: global apply = `globalConfig().load(raw,true)` + `safeSave()` +
  `reloadConfig()`; IM list = `currentGroup()`/`setGroup()`/`save()`, `Items/0`=active, `DefaultIM`=
  fallback, unknown IMs filtered by core.
- Fcitx's own self-hidden Linux-only global options (XKB via `ConditionalHidden<!hasKeyboard>`,
  `EnabledAddons/DisabledAddons/PreloadInputMethod`) — reuse the official schema, do not re-expose.
- Ordinary Shift+letter, key-up and modifier handling unchanged; no new Shift state machine; the
  D072 §5 UNFROZEN effect-capable key classification is not redefined; no product replay.
- The current available owner-test package `fusion-0.0.0-settings+5271848.zip` stays the artifact;
  no new package identity; the previous Pinyin/Shuangpin subset delivery remains the credited prior
  work.

## Expected Behavioral Delta
1. A **Global** tab appears; it loads/edits/`fcitx://config/global` through the official ConfigWidget
   against the real Host, applies on the Fcitx loop with `load/safeSave/reloadConfig`, persists across
   an orderly Host restart, respects no-change and composition guards.
2. The two-method dropdown is replaced by an **input-method management** page over the methods the
   runtime defines (currently pinyin, shuangpin): the ordered enabled list, add from available,
   remove from enabled, reorder (front = active), and it writes the group through official
   `setGroup/save`; empty-list and unavailable-selection are refused; removing an entry never deletes
   the method file or learning data.
3. `CommitRawInput` (Pinyin page) already delivers its Shift+Return / Return family to the engine
   while a composition is active; this stage keeps that and documents that Ctrl-modified and numpad
   Enter variants and any bare-Shift trigger need the D072 §5 classification decision (below).

## Hard Stop (report, do not silently implement)
Routing a **modifier-held** key (Ctrl/Alt/Win + key, incl. `Control+Return`) or a **bare-modifier**
(Shift-only) event to the engine is the D072 §5 UNFROZEN "production effect-capable key
classification" / the WP3-B6 (R19) boundary and, under D082, requires evidence that the mature Fcitx
behavior is otherwise a defect. This stage does **not** change `routes_key`/`classify_key` ownership.
Reported as the precise boundary in `capability-mapping.md` §1 and in the handoff; the requested
behaviour is delivered through the already-working Shift+Return path + option reuse.

## Validation
- On minis (build/test infra that already works — the pure-logic, non-`semantic.cpp` test family):
  settings protocol round-trip + bounds for the new/changed messages and list state; global
  apply merge/delta + no-change; IM-list validation, ordering, empty/unavailable guards,
  remove-≠-delete. Registered as a new CTest entry that builds under the existing non-MSVC
  `-Wall -Wextra -Wpedantic -Werror` flags (pattern: `wp3_input_methods`).
  NOTE (recorded, not fixed here): the shared `semantic.cpp`/`stage5_candidates` test family already
  fails to build on this machine's GCC 15.2 with `-Werror=missing-field-initializers` at base
  `fe23f07` (pre-existing, MSVC-tuned code); that is minis VALIDATION DEBT, not a Stage A change.
- Windows checks prepared, **NOT RUN** on minis (Stage B / machine 7700): MSVC Release build; default
  + explicit isolated pipe connection; global load/apply/read-back + restart persistence; no-change
  Apply + Cancel/reload + unknown/learning preservation; add/remove/reorder/select persistence +
  invalid-choice handling; cross-page stale-value regression; composition refusal/retry; Shift/raw-
  input + ordinary Shift+letter through the real Windows key-down/key-up path.
- Gate discipline: minis = Gate 1 (source/semantic) + Gate 2 (actual+negative diff, `git diff
  --check`) + the pure-logic Gate 3 above. Qt/Windows/Gate-3 product build was NOT RUN on minis,
  which is the whole reason those files were handed off; the minis-side classification was
  "source-reviewed, compile unverified" and is **superseded** by the Windows execution recorded in
  the Outcome section below.

## Required Artifacts
Product Draft PR (task branch off `fe23f07`), fork Draft PR off `0234c64` then product re-pin,
controller Draft PR with this evidence pack + reconciled status. Cumulative diffs saved under the
persistent task workspace `~/worktmp/...` (a `/tmp` copy is convenience only). Handoff record with
paths/branches/bases/heads, pins, changes, exact tests + results, NOT-RUN Windows checks, and the
D072 §5 boundary.

## Commit / Push / PR policy
Follow D076 branch namespaces (`feature/*`, `fix/*`, `docs/*`). No commit to `main`. Push task
branches, open Draft PRs, **do not merge**, do not install/register, do not touch the owner's live
input-method configuration or production services. Fork commit precedes the product pin update.

## Outcome (2026-10-21) — what actually happened, by state

The contract above is the plan of record for the batch; this section is its result. The states are
kept apart on purpose: implemented ≠ compiled ≠ tested ≠ packaged ≠ installed ≠ accepted.

- **Implemented.** Product `fusion-windows` `feature/stage-a-global-settings-im-methods` off `fe23f07`:
  feature commit `0a965e2`, one consolidated correction `fb62348` (C1–C9), packaging-pin correction
  `2e28a88`, and the closure batch `fe65d08` (C10 test isolation, C11 evidence accuracy). Controller
  `fusion-controller` `feature/stage-a-global-settings` off `96fb28c`.
  **Fork: no new commit** — the build and the package pin `abc2c66`, and inspection showed nothing in
  the fork needed changing (`configlib` links only `launchexternal.cpp`/`logging.cpp`/`font.cpp`;
  `configwidgetslib` calls only `GetConfig`, `SetConfig` and `PendingSignal::finished`; no fork
  translation unit switches on `Operation`).
- **Compiled.** Release build on Windows 7700 of `FusionSettings.exe`, `FusionHost.exe`,
  `FusionTray.exe`, `FusionTsf.dll`, `FusionFcitxHost.dll` with MSVC 14.51.36231, SDK 10.0.26100.0,
  Qt 6.8.3 and `runtime-wp3-20261018`; `ninja` reports no work to do afterwards. The Windows-side
  compile that this contract could only mark unverified is therefore **verified**.
- **Tested.** `ctest` over `stage_a_settings` (protocol/bounds/validator), `wp3_input_methods`, `rg0`
  and `stage2`: **37 of 37 pass**, with `stage4a_ipc_integration` **skipped** — that mode now requires
  an explicit isolated endpoint and connects to nothing without one, so CTest reports a skip instead of
  a successful integration run. Through `tools/settings/run-isolated-stage-a.ps1` (a real Host on a
  unique pipe, disposable profile, endpoint passed explicitly) the unchanged assertions **pass**:
  `PASS integration with real Fcitx over IPC`, `PASS integration candidate select committed the
  projected candidate`. Its earlier failure was the production endpoint it was fixed to, not a product
  defect.
  In the same isolated session: three pages load, a changed value applies and reads back, an orderly
  restart restores the value, the enabled order `shuangpin,pinyin` and the pinyin learning files; Cancel
  discards an unsaved Global edit and refreshes the baseline; Reload fetches a value a second client
  changed on the Host behind the page; switching tabs keeps an unsaved edit; a no-change Apply leaves
  the profile byte-identical; empty/unavailable list entries are refused with the documented message.
  Preservation was measured on content: the schema-known but unsubmitted `ShowInputMethodInformation` and
  pinyin `SpellEnabled` survived, a key no schema knows was dropped by Fcitx's own save, and
  `user.dict`/`user.history` were byte-identical across the settings-only session.
- **Packaged.** `fusion-0.0.0-settings+fe65d08.zip`, 60,085,623 bytes, SHA-256
  `5db4ab21c80a9ffd6aa68c880312a1c68c4b1c5f1a6fb71f78869aecb1d95e5d`, from clean committed source
  `fe65d08621fb84f47ef83acfc3e50835217af462` with `sourceDirty=false`. All 148 manifest entries were
  re-verified after extraction. The package was rebuilt rather than reused because C10 changes
  `src/settings/main.cpp`, which ships in `FusionSettings.exe`;
  `fusion-0.0.0-settings+2e28a88.zip` and `+5271848` are retained for rollback.
- **Not installed, not registered, not accepted.** No package was installed, no TIP was registered,
  and the owner's production Host, tray, browser and profile were not modified. The live
  key-down/key-up check through a TSF-registered application was **not run**, so the `Shift`/`Return`
  account rests on source reading only; **lone Shift is not implemented** and numpad Enter matches the
  `Return` family because it arrives as `VK_RETURN`.
