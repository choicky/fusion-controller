# Settings-UI spike — durable evidence record (2026-10-20)

Bounded Windows build-and-transport spike for the official Fcitx settings UI (D085), run on
machine 7700. This directory is the durable, reviewable record for the measured result. The
experiment's working tree (sources, built binaries, patched upstream checkouts, disposable
profiles) stays in the recovery workspace
`C:\Users\choicky\worktmp\fusion-windows\official-settings-ui\` and is **not** committed.

Status: **experimental feasibility proven for the Pinyin configuration page; product integration
not started.** The settings item stays open under D085; the EXE installer remains the following
separate item.

## What was measured

| Question | Result |
| --- | --- |
| Does the retained upstream widget subset build on Windows? | Yes — see `build-and-run.md` |
| Can an adapted provider drive it without a session D-Bus? | Yes, but **not** via a synthesized `QDBusPendingCall` — see `transport-probe.md` |
| Does one real UI round trip work through the official widget layer? | Yes, Pinyin `PageSize` 7 → 9 — see `ui-roundtrip.log` |
| Does the value persist across an orderly engine restart? | Yes — `ui-roundtrip.log`, "orderly restart" section |
| Is a no-change apply safe? | Yes — identical value tree and identical profile-file fingerprint, `values-comparison.md` |

Demonstrated behaviour is limited to the Pinyin `PageSize` option. The run also *rendered* the
Pinyin page from the official schema (43 value keys, 2 config types, 62 renderable options), but
no other option was driven through a round trip. Pinyin/Shuangpin **selection**, `External`
actions, dictionary/`setSubConfig` tooling and production Host behaviour were **not** validated.

## Pinned versions and toolchain

| Component | Revision / version |
| --- | --- |
| `fcitx5-configtool` | `9bd6022c6f5f768d32b6dbe2012358e4cdb1116b` (5.1.16) |
| `fcitx5-qt` | `0285a5d18367d8f3af80dab7e4819d5555982340` (5.1.16) |
| fcitx5 runtime (pinned, reused, not rebuilt) | `a7f2528773ebfec98a20c49b6eea2333110b1c88` (5.1.23), `runtime-wp3-20261018/runtime-release` |
| Qt | 6.8.3 `win64_msvc2022_64`, `qtbase` archive only (via `aqtinstall` 3.3.0) |
| ECM | 6.10.0 (built locally from the KDE release tarball; build-time only) |
| Compiler / SDK | MSVC 14.51.36231 (VS 18 Community), Windows SDK 10.0.26100.0 |
| Build generator | CMake + Ninja from the Visual Studio installation |

`fcitx5-macos` was inspected read-only at `d31733edee0975619e753350d323757f595b7734` (shallow
clone) as a source reference only; it was not built or modified.

## Evidence index

| File | Contents |
| --- | --- |
| `build-and-run.md` | Prerequisites, exact configure/build/run commands, built-artifact inventory |
| `transport-probe.md` | The QtDBus probe's four strategies and their measured results |
| `ui-roundtrip.log` | Full console transcript of the UI round trip (baseline → UI apply → re-read → restart → no-change) |
| `values-comparison.md` | What the no-change comparison covered, and its result |
| `patch-inventory.md` | The exact upstream delta: files, diffstat, SHA-256, provenance — **patches themselves are local-only** |
| `settings-window.png` | Screenshot of the experimental window (1140×1080) |

## What this record does **not** contain, and why

- **The upstream patch files.** Their content is retained upstream source under GPL-2.0-or-later
  (configtool) / LGPL-2.1-or-later (fcitx5-qt widgetsaddons). Committing or publishing them
  requires the owner's licence decision, which has not been made. `patch-inventory.md` therefore
  records an inventory with hashes and provenance, **not** a reviewable patch.
  Consequence, stated plainly: **this directory alone cannot reproduce the patched build.** The
  patches and the patched checkouts exist only in the recovery workspace.
- **Binaries, dependencies, disposable profiles.** Excluded by design (repository lifecycle rules).
  Their names, sizes and hashes are recorded in `build-and-run.md` as build evidence.
- **The experimental harness sources** (`wire.h`, the engine service, the pipe CLI, the settings
  app, the probe). These are this task's own code, but they are not product code and not part of
  the feasibility claim; they remain in the recovery workspace. `build-and-run.md` describes their
  structure precisely enough to be reviewed against the results.

## Independent-review limitations

- The result was produced and recorded by the same session that built it; no second reviewer has
  reproduced it. Reproduction requires the local patches and the task-local toolchain.
- Only one option, on one page, in one URI form was driven end to end. The other two URI forms
  (`global`, `addon/<name>`) are implemented in the harness but were not exercised through the
  widget layer.
- The engine process is an isolated test process built from the pinned runtime — not the
  production Host, and not the product's TIP/TSF path.
- `External`-type options render but cannot launch anything on Windows (no wrapper binaries exist);
  that is upstream's own fallback behaviour, not a validated feature.
- The screenshot proves a rendered window; it does not by itself prove the values shown, which is
  what `ui-roundtrip.log` and `values-comparison.md` are for.

## Relation to the documents

`docs/windows-settings-ui-feasibility.md` §4–§5 and §9–§10 carry the reconciled conclusion and link
here. The historical record of the failed synthesized-reply approach is kept in §4 as experimental
evidence; the asynchronous adapter is the current approach.
