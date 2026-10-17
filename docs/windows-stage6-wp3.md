# Windows Stage 6 WP3 — input methods and user data (R5a–R5c, U1)

Date: 2026-10-18. Owner decision recorded: **U1 = Accepted 2026-10-18** (§7.1 of the plan).

**Result: WP3 is closed within its scope.** B1–B6 passed, and the owner's product-path walkthrough
(B3a–B3g and Notepad A–E) passed on the Release candidate of `a720319` against `runtime-wp3-20261018`.

## Candidate and runtime identities

| item | value |
|---|---|
| fusion-windows base | `0cd0374` (PR #15 merge) |
| branch / head | `feature/stage6-wp3-input-methods` — `a720319` (PR #16: `f27c754`, `6831064`, `a720319`) |
| Release candidate | `stage6-wp3\build-release-wp3fix\bin` — `FusionTsf.dll` `A0C13C0AC580869D4BD9E39CFA4A2EC6550240A3374B602217032E770C41BD80`, `FusionHost.exe` `6B74E700749A93651FCED28CBB452AF79962A3736B814649A9981A67D3737637` |
| runtime | `runtime-wp3-20261018` (Debug + Release) — fcitx5 `a7f2528` + `fcitx5-msvc.patch`, fcitx5-chinese-addons `61474bd` + `chinese-addons-msvc.patch`, libime `171edcf` unmodified (kenlm `4cb443e6`), vcpkg `20ceb8ad` |
| previous runtime | `runtime-wp1-20261017` — untouched |
| changed files | 10 (`CMakeLists.txt`, `include/fusion/backend.hpp`, `src/fcitx/backend.cpp`, `src/fcitx/startup_group.hpp` (new), `src/host/host_server.{hpp,cpp}`, `src/host/main.cpp`, `tests/stage4a_ipc.cpp`, `tests/wp3_input_methods.cpp` (new), `tools/fcitx/build-native.ps1`) |

## Why the runtime changed

`runtime-wp1-20261017` shipped only `data\fcitx5\inputmethod\pinyin.conf`. The pinyin addon defines
Shuangpin as a second input method of the same addon (`im/pinyin/shuangpin.conf.in`; upstream's
`im/pinyin/CMakeLists.txt` installs both files), but `tools/fcitx/build-native.ps1` never asked the build for
the generated `shuangpin.conf` and never copied it, and Fcitx builds its input-method entries from
`inputmethod/*.conf` (the addon does not list input methods programmatically). Without the definition no
`shuangpin` entry exists, so no profile could select it and R5a's Shuangpin half was unreachable.

The fix is the smallest one: one more generated target plus one more copy, mirroring `pinyin.conf`. R23 was
**not** fixed; the recorded WP1 workaround was used (both repository patches applied with `git apply` before
running the script).

Old/new comparison per configuration (`runtime-diff.txt`, full manifest in `runtime-manifest.txt`):

| configuration | files | added | removed | size-changed |
|---|---|---|---|---|
| `runtime-debug` | 377 → 378 | `data\fcitx5\inputmethod\shuangpin.conf` | none | `lib\fcitx5\pinyin.lib`, `lib\fcitx5\punctuation.lib` (rebuilt import libraries) |
| `runtime-release` | 377 → 378 | `data\fcitx5\inputmethod\shuangpin.conf` | none | none |

Restricted to `data\`, the added definition file is the only difference in both configurations, so the
"unexpected difference" stop condition was not triggered.

**WP2's conclusions were obtained on `runtime-wp1-20261017` and were not re-run on the new runtime.** What
changed between the two runtimes is one input-method definition file; every WP2 statement about admission,
diagnostics and password fields is unaffected by it, but this record does not claim a re-run.

## B1 — builds and tests (measured)

All trees configured fresh in a `chcp 65001` console against `runtime-wp3-20261018`:

| tree | build | CTest |
|---|---|---|
| Mock-only Debug | 0 | **69/69** |
| Debug + real Fcitx | 0 | **77/77** |
| Release + real Fcitx | 0 | **77/77** |

`clang-format --dry-run --Werror` and `git diff --check` clean.

## B2 — identity and user data (measured)

Recorded by the walkthrough itself (`verify-wp3-resume-20261018-031519.log`): commit `a720319` (working tree
clean), both candidate hashes above, runtime input methods `pinyin.conf, shuangpin.conf`, and the loaded TIP
verified by path and hash in the repository's own diagnostics. Before the first attempt neither
`%APPDATA%\Fcitx5` nor `%LOCALAPPDATA%\Fcitx5` existed; after the run the user directory holds the files
listed below, and it is kept as the owner requires.

## B3 — product-path persistence (owner-confirmed, all measured on the candidate)

| step | result | what was observed |
|---|---|---|
| B3a | **PASS** | first run, no profile: Pinyin default, `nihao` + Space committed `你好` |
| B3b | **PASS** | Shuangpin first in `Items/0`; the Host printed `[shuangpin,pinyin] current=shuangpin default=pinyin profile=present source=loaded`; keys **`nihk`** + Space committed `你好`; Shuangpin scheme = Fcitx default (`ShuangpinProfile` not set in the addon config) |
| B3c | **PASS** | `PageSize=9` written while the Host was stopped; after the orderly restart the popup showed **9** candidates |
| B3d | **PASS** | Shuangpin `ui`, selected the 4th candidate → rose to rank 1; orderly stop with Notepad connected (CTRL_BREAK, exit 0, `Save before exiting...`), restart: still rank 1 |
| B3e | observation (expected) | Shuangpin `vi`, 5th candidate → rank 1; after forced termination and restart it was back at rank 5 — learning since the last save is lost (Fcitx's own semantics) |
| B3f | observation | a profile swap written while the Host ran did **not** survive: Fcitx's save on stop rewrote the profile from memory (`DefaultIM=pinyin`, `Items/0=shuangpin`). Users must stop the Host before editing Fcitx configuration files |
| B3g | **PASS** | Pinyin `chi`, original rank 4 → rank 2 after learning → rank 1 after closing the Host console window with the X and restarting; the Host log shows `Save before exiting...`, the addon unloads and `shutting down`. The extra promotion (2 → 1) is most likely a second selection of the same character before the close and is recorded as such, not as a persistence discrepancy |
| B4 | **PASS** | Notepad A–E on the candidate |

Every orderly stop in the run delivered a real `GenerateConsoleCtrlEvent(CTRL_BREAK)`, reported
`Host exited=True code=0` and `save line present: True` (`verify-wp3-resume-20261018-031519.log`).

**Resolved user paths** (measured; Fcitx's own package directories, no Fusion-specific location):

| item | path |
|---|---|
| input-method profile | `%APPDATA%\Fcitx5\config\fcitx5\profile` |
| pinyin addon config (scheme, `PageSize`, options) | `%APPDATA%\Fcitx5\config\fcitx5\conf\pinyin.conf` |
| learning (user dictionary / history) | `%APPDATA%\Fcitx5\pinyin\user.dict`, `user.history` |
| data root | `%APPDATA%\Fcitx5\data` (resolved by Fcitx; not created in this run) |
| cache | `%LOCALAPPDATA%\Fcitx5\cache` (resolved by Fcitx; not created in this run) |

## B5 — no learning in password/private fields (source argument, unchanged)

WP3 leaves `src/tsf/service.cpp`, `semantic.cpp` and `key_map.cpp` byte-identical to base (checked with
`git diff --quiet`), so the T/K admission, the hard policy and the InputScope classification are exactly as
WP2 merged them. A key the hard policy refuses returns before a `BackendStimulus` exists
(`begin_real_key`: `!policy_allows` and `!routes_key` both return FailOpen), and `handle_pass_key` performs
projection cleanup only, never `send()`. Fcitx therefore never sees a blocked keystroke and cannot learn
from one; WP2's product evidence (Edge and Firefox password fields: `HARD_POLICY guid-blocked` →
`OWN_FAIL_OPEN` → genuine PASS, 0 `FCITX_BATCH`/`COMPOSITION_START`/`EFFECT_APPLIED`) stands. No new browser
run was needed. U5's accepted limitation (Sensitive classification not demonstrated; R11 follow-up open) is
unchanged.

## B6 — §25 `overlayMessage` trigger (source argument)

Modifier combinations never reach Fcitx: `routes_key` requires `!action.modified`, the classifier marks
`ctrl || alt || win` as a modified `Pass`, and `begin_real_key` returns before building a stimulus for a
modified `Pass`. No other trigger exists on the product path (no UI, no IPC message, no Fusion switch — U1
forbids one), so **switching input methods at runtime is not reachable** and the §25 `overlayMessage` row
stays **DEFER** per its own terms. No §25 edit.

## Fcitx semantics found while implementing (recorded because they shape the procedure)

1. **`Items/0` is the input method in use; `DefaultIM` is the fallback.**
   `InputMethodGroup::setDefaultInputMethod` rewrites a default equal to `Items/0` to `Items/1`, and
   `Instance::inputMethod` returns `defaultInputMethod()` only for an *active* state while an inactive
   context gets `Items/0`. A bound context follows `Items/0`; editing `DefaultIM` alone does not switch.
2. **Fcitx writes user data on load** (`conf/pinyin.conf`, `conf/punctuation.conf`), not only on save — which
   is why the singleton is claimed before the backend exists.
3. **Fcitx saves on instance exit** (`Instance` exit event → `save()`), which is what the Host's orderly stop
   relies on; a forced termination keeps Fcitx's own semantics (changes since the last save are lost).

## Harness defects recorded (script only; no product change)

The walkthrough script was rewritten after the aborted first attempt; the defects below are recorded for the
next harness, not fixed in product code:

1. The first attempt's orderly stop called `CloseMainWindow()` on a Host started with redirected stderr —
   that Host has no window, so nothing was delivered and the Host was force-killed after 8 s
   (`verify-wp3-20261018-025927.log`). Diagnosed as a delivery failure, not a product defect
   (`diagnose-orderly-stop.log`): a real `CTRL_BREAK` stops the same Host with exit code 0 and the save line,
   even with a client connected (`console-stop-with-client.log`).
2. B3g asked for new learning before starting its Host, while no Host was running; the owner produced the
   learning after the Host window opened.
3. The B3d/B3g prompts accepted only Enter, so the ranks in this record are the owner's reported
   observations rather than script-captured values.
4. "Collecting diagnostics" failed with "cannot overwrite the item with itself"
   (`host-b3b-before-stderr.log`); it ran after B4, so no result was lost. Cleanup still ran: the leftover
   Host (pid 556) was terminated, the pre-run registration restored, the Fusion profile deactivated, and the
   user data kept.
5. The script's backup name reused the run stamp, so the three profile backups collapsed into one file
   (`profile.20261018-031519.bak`); the content written at each step is in the run log.

## Known limits

- No hotkey switching between Pinyin and Shuangpin: modifier combinations do not reach Fcitx (R19), so
  switching is configuration plus a Host restart (B6).
- A configuration file edited while the Host runs is overwritten by its next save (B3f): stop the Host first.
- Forced termination loses changes since the last save (B3e) — Fcitx's semantics, documented, not fixed.
- Logoff/shutdown saving depends on how the Host is started; that is WP4/WP5 work, recorded as R26.
- Pinyin and Shuangpin only; the runtime defines no other input method. No UI and no Fusion-specific
  configuration file.

## User-facing selection procedure

1. Stop the Host **orderly** (Ctrl+C/Ctrl+Break in its console, or close its console window) so Fcitx writes
   its configuration.
2. Edit `%APPDATA%\Fcitx5\config\fcitx5\profile` and put the wanted input method **first** in the group's
   item list (`[Groups/0/Items/0] Name=...`). `Items/0` is the input method in use; `DefaultIM` is only
   Fcitx's fallback. The Shuangpin scheme and the other pinyin options live in
   `%APPDATA%\Fcitx5\config\fcitx5\conf\pinyin.conf`.
3. Start the Host. Its log line `input method group: [...] current=... default=... profile=... source=...`
   states what Fcitx resolved.

## Evidence locations

All under `C:\Users\choicky\worktmp\fusion-windows\stage6-wp3\` unless noted:

| what | where |
|---|---|
| final walkthrough log and per-Host logs | `verify-wp3-resume-20261018-031519.log`, `wp3-resume-20261018-031519\host-*-stderr.log` |
| aborted first attempt | `verify-wp3-20261018-025927.log`, `wp3-run-20261018-025927\` |
| orderly-stop diagnosis and console-stop checks | `diagnose-orderly-stop.log`, `diagnose-case1-host.log`, `diagnose-case2-host.log`, `console-stop-with-client.log`, `console-stop-verification.log` |
| user-directory accounting | `user-dir-after-abort.txt`, `user-dir-accounting.txt` |
| B1 logs | `tests-off.log`, `tests-debug-wp3fix.log`, `tests-release-wp3fix.log` |
| runtime identity and comparison | `runtime-identity.md`, `runtime-diff.txt`, `runtime-manifest.txt` |
| candidate identity | `candidate-identity.txt` |
| B5/B6 arguments | `b5-b6-arguments.md` |
| change contract / validation / checklist | `change-contract.md`, `validation.md`, `CHECKLIST.md` |
| runtime product | `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018\` (identity and manifest above) |
