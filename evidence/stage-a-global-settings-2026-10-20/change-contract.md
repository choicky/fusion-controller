# Stage A Change Contract — global settings + input-method management

Per `fusion-windows/AGENTS.md` §9.2. Bounded contract for the Stage A implementation instruction.
Base: `fusion-windows` origin/main `fe23f07` (PR #28 merged); `fusion-controller` origin/main
`96fb28c` (PR #51 merged); fork `choicky/fcitx5-configtool` `windows/fusion-settings` `0234c64`.

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
  --check`) + the pure-logic Gate 3 above. Qt/Windows/Gate-3 product build is NOT RUN on minis;
  recorded as `STATIC REVIEW PASS; COMPILE UNVERIFIED` for the Qt/fork files.

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
