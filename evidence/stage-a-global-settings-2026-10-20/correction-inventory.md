# Stage A correction inventory (Windows 7700, 2026-10-21)

Consolidated review of the three Stage A heads before editing. Every entry is traced to the
committed source at product `0a965e2`, fork `abc2c66`, controller `8ca1282`. This is the single
list the correction batch applies; nothing else is changed.

## Confirmed by review and in scope

| ID | Finding | Evidence | Disposition |
|---|---|---|---|
| C1 | `src/settings/main.cpp` assigns `QListWidget::addItem(const QString &)` to `auto *` and dereferences it at four sites (`168`, `183`, `633`, `640`). That overload returns `void`, so `FusionSettings.exe` cannot compile. | Qt 6 `QListWidget` API | Construct a `QListWidgetItem` with the target list, so ownership stays with the list and `takeItem()` remains correct. No model layer. |
| C2 | `reloadAll()` → `loadBothPages()` reloads only Pinyin and Shuangpin. Once the Global page has loaded, Cancel and Reload leave its edits and `baseline_` untouched while reporting success. `globalLoaded_` is never reset. | `main.cpp:314-347` | Extend the serialized loader to every page the button claims to reload, reset each loaded flag, and report success only when all pages loaded. Keep the Busy protection and the tab-switch unsaved-edit behaviour. |
| C3 | `capability-mapping.md` §2 says the inert global options are "explained (GUI limitations label + this record)". The only label is a generic sentence about composition timing and dictionary management; nothing identifies the unsupported categories. | `main.cpp:145-149` vs `capability-mapping.md:116-117` | Add one precise explanatory label on the Global page listing the categories that do not work on the Windows path. Do not disable or hide supported configuration and do not invent semantics. Reconcile the mapping to point at the actual label. |
| C4 | The Shift/Enter account contradicts itself. `capability-mapping.md` §1 says numpad Enter is "in scope" and "fixable by keysym translation only"; the handoff and the plan say it belongs to the D072 §5 boundary and is not implemented. No `key_map.cpp` change exists in the diff: `kVkReturn` (0x0D, which is also what a numpad Enter produces) maps only to `0xff0d`. The plan also calls the Shift item "resolved against source". | `git diff fe23f07..0a965e2 -- src/tsf/` is empty; `key_map.cpp:51-53` | Documentation only. Describe what the source does: `Return`/`Shift+Return` are delivered while a composition is active; numpad Enter arrives as `VK_RETURN` and therefore matches the `Return` family, never the `KP_Enter` entries; `Control`/`Alt`/`Win` and lone `Shift` never reach the engine. Lone Shift stays explicitly unimplemented. Remove "resolved"/"STATIC REVIEW PASS" claims the evidence does not support. |
| C5 | The settings handshake compares `api == settings_api_version` on both sides. Version 2 is strictly additive over version 1 (no existing message changed shape), so an exact match rejects the compatibility the wire protocol actually has — including the already-shipped `+5271848` client against a v2 Host. | `host_server.cpp:548-560`, `dbusprovider.cpp:521` | Host accepts any client API in `[1, settings_api_version]`; the client requires `api >= settings_api_version` so a newer Host stays usable. The advertised version is still reported verbatim. |
| C6 | `third_party/settings-ui/SOURCE-ACCESS.txt` names fork commit `d440322d…` twice while the build pins `abc2c66…`. | `SOURCE-ACCESS.txt:9-10`, `versions.cmake:4` | Correct the notice to the pinned revision. |
| C7 | `tools/settings/prepare-settings-sources.ps1` builds its pin regex as `^set\(NAME "([0-9a-f]{40})"\)$` under `(?m)`. With a CRLF checkout the `\r` sits before the anchor and every pin lookup throws, so a normal fresh Windows checkout cannot prepare sources. | `prepare-settings-sources.ps1:13-19`; reproduced in the previous batch | Make the pattern line-ending tolerant rather than depending on a specially normalized worktree. |
| C8 | `windows-stage6-plan.md` carries the earlier settings-status paragraph, the earlier validation paragraph, and a new multi-paragraph Stage A narrative side by side, so the current scope/completion state is stated three times with different tense. | `windows-stage6-plan.md:188-260` | Consolidate into one status paragraph, one validation paragraph and one limitations paragraph covering the subset delivery plus Stage A. No appended "latest update" block. |

## Added by this review, same paths

| ID | Finding | Evidence | Disposition |
|---|---|---|---|
| C9 | The only real-product GUI harness (`--self-test`) exercises Pinyin/Shuangpin only. Nothing in it reaches the Global page or the list-management surface, so the new capability has no Host+GUI evidence path. | `main.cpp:452-592` | Extend the existing self-test with a Global step (render, change a supported option, apply, read back from the Host) and a list step (reorder + remove/add, apply, read back), then add a bounded `--self-test-verify` mode that asserts the applied values after an orderly Host restart. The existing cross-page scenario and its exit codes stay unchanged. |

## Checked and deliberately not changed

- **Fork header vs. fork targets.** `src/lib/configlib/dbusprovider.h` is the only fork change. The
  fork compiles `configlib` (`launchexternal.cpp`, `logging.cpp`, `font.cpp`) and
  `configwidgetslib` (`configwidget.cpp`, …). `configwidgetslib` calls `ControllerProxy::GetConfig`,
  `SetConfig` and `PendingCall::finished`; neither target defines or calls `PendingCall::complete`,
  and no fork translation unit switches on `ControllerProxy::Operation`. The new declarations are
  satisfied by the product's own `src/settings/dbusprovider.cpp`, which is the only compiled
  definition. Declarations and definitions are therefore coherent for every target the fork builds,
  and the fork needs no further commit.
- **`configwidgetslib` does not use the list-management API.** `GetImList`/`SetImList` are consumed by
  the product GUI only, so no widget-layer model is required.
- **Global schema nesting.** `merge_schema_values` already recurses through
  `find_config_type(types, option->type)`, which is how the per-input-method pages handle their own
  nested types; `GlobalConfig`'s `Hotkey`/`Behavior` groups take the same path, and the apply starts
  at `types.front().name` exactly as the per-IM apply does.
- **`DefaultIM` handling.** `plan_im_list_update` passes the current fallback straight to
  `setDefaultInputMethod` and does not reimplement Fcitx's own `Items/0` rewrite. Left as is.
- **Two apply buttons (page Apply vs. list Apply).** This is one management surface with its own
  commit action, not a second manager. Left as is; the daily-use guide states the scope of each.
- **README.** `fusion-windows/README.md` has no settings section to reconcile; the change contract's
  "README settings section" line is not applicable and no file is invented for it.

## Applied

All nine entries were applied in one batch, then built and exercised.

| ID | Where it landed |
|---|---|
| C1 | product `src/settings/main.cpp` — four `new QListWidgetItem(text, list_)` sites; `<QListWidgetItem>` included |
| C2 | product `src/settings/main.cpp` — `ReloadChain` walks `{pinyin_, shuangpin_, global_}`, every loaded flag reset, success only when all three loaded |
| C3 | product `src/settings/main.cpp` — `globalPageNote()` attached to the Global tab through `makePage(uri, page, note)` |
| C4 | controller `capability-mapping.md` §1, `windows-stage6-plan.md`, `handoff.md` §4 — numpad Enter as `VK_RETURN`, lone Shift explicitly not implemented, "resolved" and "STATIC REVIEW PASS" claims removed |
| C5 | product `tools/settings/prepare-settings-sources.ps1` and `tools/package/build-package.ps1` — `[ \t\r]*$` on every pin anchor |
| C6 | product `.gitattributes` (`*.patch -text`) plus normalized patch bytes, so a CRLF checkout stays byte-exact |
| C7 | product `third_party/settings-ui/SOURCE-ACCESS.txt` — fork PR #2 and `abc2c66` |
| C8 | controller `windows-stage6-plan.md` — one consolidated settings section replaces four overlapping status/validation/limitations paragraphs; `windows-daily-use.md`, `windows-architecture.md`, `windows-settings-ui-feasibility.md`, `ROADMAP.md`, `THIRD_PARTY_LICENSES.md` reconciled |
| C9 | product `src/settings/main.cpp` — Global and list steps in `--self-test`, plus read-only `--self-test-verify` with `--expect-im-list` |

Two further corrections were needed once the packaging ran and are recorded with C5/C7 rather than as
new findings: the manifest's `settings.commit`/`repository`/`branch` were literals (the commit was
stale at `0234c64`), so they now come from `cmake/settings-ui/versions.cmake`, and that file gained
`FUSION_SETTINGS_CONFIGTOOL_BRANCH` so the branch has one declared source.

Result: product head `2e28a88` (Release build clean, 36/37 focused CTest, isolated Host + GUI
validation passing, package `fusion-0.0.0-settings+2e28a88.zip`), controller head on PR #52, fork
unchanged at `abc2c66`. The one failing test and the two unrun checks are named in `handoff.md` §6;
none of them is presented as passing.
