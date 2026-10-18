# Windows Stage 6 WP4 — packaging and lifecycle (R6, R9, R23, R26; U2)

Date: 2026-10-18. Owner decision recorded: **U2 = (b) Accepted 2026-10-18** (§7.1 of the plan).

**Result: WP4 is implemented and locally verified; the owner walkthrough (B3/B4) is pending.** B1, B2 and the
walkthrough dry run are complete on the branch head below. The lifecycle evidence that requires an interactive owner
(install, sign-out/sign-in, upgrade with a running application, uninstall) is not claimed here.

## Candidate, runtime and tooling identities

| item | value |
|---|---|
| fusion-windows base | `572fdb9` (PR #16 merge, `main`) |
| branch / head | `feature/stage6-wp4-packaging` — `7cba293` (implementation), `faeb266` (package file selection) |
| Release candidate | `stage6-wp4\build-release\bin` — `FusionHost.exe` `8C341D17B1DE7CD708A030E724C60D1274BAA7C322EAF5D5AE41341E5BDAF141`, `FusionTsf.dll` `22D41116DECAD2E7A30A46B44E675CDF36FEACB4D2B50191F5593BDFA0AB0EB9`, `FusionFcitxHost.dll` `01AA0CF48B501832C66D0D4FB35688C3C0FEB40A58A849EBA0E0F7D19963D3D9` |
| runtime | `runtime-wp3-20261018` (Debug + Release), rebuilt on this machine from the WP3 pins — fcitx5 `a7f2528` + `fcitx5-msvc.patch`, chinese-addons `61474bd` + `chinese-addons-msvc.patch`, libime `171edcf` unmodified, vcpkg `20ceb8ad`; Release `Fcitx5Core.dll` `98675392F46067DB13E442A5ED26EF64C719692069B1174BBE8478E2B4C662AB` |
| changed files | 12 (`CMakeLists.txt`, `README.md`, `src/host/main.cpp`, `src/host/host_server.{hpp,cpp}`, `tests/stage4a_ipc.cpp`, `tools/fcitx/build-native.ps1`, `tools/package/{build-package,install,uninstall,host-state,verify-wp4}.ps1`) |

### Environment facts that differ from the WP3 record

Recorded because they bound what these results mean, not as product decisions:

1. **The runtime directory did not exist.** `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018` was absent
   at WP4 start (only an older Debug-only prefix from the WP1 era remained). The runtime was therefore rebuilt from
   the same pinned sources and patches with `tools/fcitx/build-native.ps1` into that exact directory. Source identity
   is identical to WP3's record; binary hashes are this machine's build and are recorded above, not inherited.
2. **No WP3 user data was present.** `%APPDATA%\Fcitx5` did not exist before the WP4 run. The A1 console check (below)
   ran without `FUSION_FCITX_SKIP_USER_PATH` and created a first-run default configuration plus an empty user
   dictionary at 09:59. That is recorded as an unintended touch of real user data by a WP4 check (no pre-existing data
   was modified or removed), and it is the state the walkthrough's "before" snapshot contains. Step (a)'s "existing
   user data is reused" therefore means "the first-run default configuration is reused", not WP3 learning.
3. **`reg.exe` is blocked** by this machine's Security Center program blacklist, so `VsDevCmd.bat` cannot detect the
   Windows SDK and produces `LIB`/`INCLUDE` without SDK paths (link fails with `LNK1104: kernel32.lib`). The build was
   driven by a local wrapper outside the repository that supplies the SDK `INCLUDE`/`LIB`/`PATH` and then calls the
   repository script unmodified. This is a machine condition, not a repository change.
4. **clang-format available is 23.1.3** (WP3 used 22.1.3). It reports formatting differences in files this WP did not
   touch (`src/tsf/service.hpp`, `src/tsf/diag.cpp`), so the check was scoped to the files this WP changed.

## A1 — windowless by default, console mode kept

`FusionHost.exe` is now a Windows-subsystem binary (`dumpbin /headers` → `subsystem (Windows GUI)`), so the packaged
process has no console window and no console at all. The console path is kept for development and tests behind
`--console`:

* `--console` attaches to the caller's console explicitly (`AttachConsole(ATTACH_PARENT_PROCESS)`) because a
  GUI-subsystem process does not inherit one, and installs the existing control handler. A redirected stderr is left
  untouched, so the WP3 harness pattern (redirect stderr, then deliver `CTRL_BREAK`) still works.
* The windowless Host writes no log file (U3) and adds no new logging mechanism; it keeps writing to stderr, which a
  redirecting caller still captures.

Evidence (`a1-console.txt`, `a1-console-host-stderr.log`): the child appears in `GetConsoleProcessList` for the
caller's console, a real `GenerateConsoleCtrlEvent(CTRL_BREAK_EVENT, <pid>)` reaches it, the Host exits 0 and the log
contains `Save before exiting...` followed by the addon unloads and `shutting down`.

## A2 — session end and the named stop event

| mechanism | what it does |
|---|---|
| hidden top-level window (`WS_POPUP`, `WS_EX_TOOLWINDOW`, not `HWND_MESSAGE`) | receives `WM_QUERYENDSESSION` / `WM_ENDSESSION`, requests the stop and waits, bounded at 4 s (the console path's bound), for the ordered shutdown; `WM_ENDSESSION` with `wparam == FALSE` (cancelled) does nothing |
| named stop event `Local\FusionHost-stop-<user SID>-<session id>` | created by the Host before the backend exists, with the pipe's own user-only DACL; `install.ps1` / `uninstall.ps1` open and signal it instead of terminating the process |
| console control handler | unchanged, console mode only |

All three paths only *request* the stop. `main()` remains the single thread that runs `stop()` →
`shutdown_backend()` → Fcitx's save on instance exit, and the `HostClaim` is held until the process exits. The
stop-event waiter is one thread that waits on the stop event and the exit signal and never drains, joins or touches the
backend. The window lives on its own thread so the accept loop keeps running on the main thread; `WM_CLOSE` ends it
after the exit signal is set.

Fail-closed choices: a Host that cannot create the stop event, or (windowless) cannot create the session-end window,
refuses to run — otherwise it could be neither stopped orderly nor save on logoff (R26).

## A3 — runtime location

The packaged layout is `bin\` + `data\` + `lib\libime\`, and Fcitx resolves its data from the parent of the directory
that holds `Fcitx5Utils.dll`, which it requires to be named `bin`. Every DLL sits beside the executable, which is the
first entry of the default DLL search order, so nothing depends on `PATH`.

Evidence (`a3-smoke.txt`):

* `dumpbin /dependents FusionHost.exe` → `FusionFcitxHost.dll` plus system DLLs only; the Fcitx runtime is loaded from
  the same directory.
* `dumpbin /dependents FusionTsf.dll` → `ole32`, `ADVAPI32`, `USER32`, `GDI32`, `KERNEL32`, `OLEAUT32` and the VC/UCRT
  DLLs. **Zero Fcitx/LibIME dependencies**, so the thin-DLL boundary holds.
* The staged layout was started with `PATH=%SystemRoot%\system32;%SystemRoot%` (no runtime on PATH) and
  `FUSION_FCITX_SKIP_USER_PATH=1`: `Fcitx backend ready`, then `stage4a integration` **PASS** end-to-end against it,
  then the stop event stopped it with exit 0 and `Save before exiting...`.

## A4 — package, install, upgrade, uninstall

**Package form.** A zip whose root maps onto `C:\Program Files\Fcitx5 Fusion\<version>\`: `bin\` (Host, TSF DLL,
`FusionFcitxHost.dll`, 21 runtime DLLs), `data\` (fcitx5 addon/input-method/pinyin data, `libime` dictionaries),
`lib\libime\zh_CN.lm`, `tools\` (`register.ps1`, `unregister.ps1`, `check-state.ps1` — reused, not duplicated),
`install.ps1`, `uninstall.ps1`, `host-state.ps1`, `package.json`. Test executables and `FusionTsfProfile.exe` are not
packaged. No MSI, no installer toolchain, no signing (U4 stays open; decided with WP8).

**Manifest.** `package.json` records package name, version, source commit and dirty flag, build time, configuration,
runtime identity (label, pinned fcitx5 / chinese-addons / libime commits, patch names) and `size` + `sha256` for every
file. `install.ps1` verifies every hash before copying and refuses on a mismatch.

**Install.** Elevated (refuses otherwise). Order: verify → ask a running Host for an orderly stop → copy into the
version directory → register the TIP from that directory (`tools\register.ps1`) → create or update the logon task →
start the Host for the current interactive user → remove older version directories that are not locked and are not the
registered one.

**Scheduled task** (`FusionHost`), verified values recorded by `install.ps1`:

| property | value |
|---|---|
| trigger | at logon of the installing user |
| principal | that user, `LogonType Interactive`, `RunLevel Limited` (Medium integrity, never elevated) |
| action | `<version>\bin\FusionHost.exe`, working directory `<version>` (no `--console`) |
| execution time limit | none (`PT0S`) |
| multiple instances | `IgnoreNew` (the `HostClaim`, not the task, remains the authority for one Host per user × session) |
| restart on failure | **not** configured (crash restart is WP5/U7 evidence) |

**"Start now" from the elevated installer** uses `Start-ScheduledTask`; an interactive-token, limited-run-level task
starts in the user's own session with the user's limited token. `install.ps1` then waits (bounded) for a `FusionHost`
process whose path is under the version directory and reports its pid, session, user and integrity level; a Host that
does not appear is reported as an installation that is complete but not running (exit 3), not as success.

**Upgrade.** A new version directory; the previous Host is stopped through the stop event before any file is copied;
registration and the task are re-pointed at the new directory; the previous directory is removed only if no running
application still holds its DLL. A locked directory is left in place and reported, never force-deleted.

**Uninstall.** Elevated. Order: ask the Host for an orderly stop (abort if it does not exit within the bound) →
unregister and remove the `CTF\TIP` / COM keys that the COM unregister leaves → delete the task → remove version
directories that are not locked → remove the install root if empty. `%APPDATA%\Fcitx5` is kept unless
`-RemoveUserData` is given, which removes only the current user's `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5`.

**Not implemented, as decided.** The logon **Run** entry (U2 (a)) is not implemented. Its recorded gaps: an elevated
installer cannot start a Run entry non-elevated for the current user without a sign-out, and nothing restarts a Run
entry that exits.

## A5 — R23 (build script probe)

`tools/fcitx/build-native.ps1` now relaxes `$ErrorActionPreference` around every native tool call and decides on
`$LASTEXITCODE` alone. The patch probe uses the same rule. The fix is not cosmetic: on this machine a CMake *warning*
on stderr aborted the Debug runtime build at the same place (`C1090`-era rebuild attempt), so the defect class covers
every native invocation, not only the probe.

Evidence (`a5-probe.txt`), Windows PowerShell 5.1.26100, fresh clone of fcitx5 at `a7f2528` (clean tree):

* the old probe form (`git apply --reverse --check … 2>$null` under `Stop`) **raises** `RemoteException: error: patch
  failed: src/lib/fcitx/instance.cpp:795` — R23 reproduced;
* the fixed probe returns `False` without a terminating error and leaves the tree clean;
* `Apply-NativePatch` applies the patch (9 files changed) and the probe then returns `True`;
* a second `Apply-NativePatch` changes nothing (same file set) — idempotent.

The functions were extracted verbatim from the script by the harness, so the test runs the shipped code.

## A6 — automated coverage

`stage4a wp4stopevent` (registered as `stage4a_ipc_wp4stopevent`, runs in every Windows/MSVC build): creates the stop
event with a test scope, checks that a second create for the same scope is refused, starts a real `HostServer` with a
connected client, has an outside caller open the event **by name** and signal it, and then checks that the connection
is still owned and the backend untouched until the owning thread runs `stop()` → `shutdown_backend()`, that the backend
is destroyed exactly once and never during a call. "The stop event is refused to another principal" is not tested: it
needs a second token, which this scope does not have.

## B1 — builds and tests (measured)

Every tree configured fresh in a `chcp 65001` console against `runtime-wp3-20261018`:

| tree | build | CTest |
|---|---|---|
| Mock-only Debug | 0 | **70/70** |
| Debug + real Fcitx | 0 | **78/78** |
| Release + real Fcitx | 0 | **78/78** |

`stage4a_ipc_integration` is the operator-driven mode that requires an already-running Host; the suites above use its
documented `FUSION_STAGE4A_SKIP_INTEGRATION=1` switch, and the mode was run separately against a started Release Host
(`a3-smoke.txt`, PASS). `clang-format --dry-run --Werror` is clean on every file this WP changed; `git diff --check` is
clean. (See environment fact 4 for why the formatter check is scoped to the changed files.)

## B2 — packages (measured)

Both packages were built from the same commit `faeb266` with `tools/package/build-package.ps1`; full manifests are in
`b2-manifests.txt`.

| item | wp4.1 | wp4.2 |
|---|---|---|
| file | `fusion-0.0.0-wp4.1+faeb266.zip` | `fusion-0.0.0-wp4.2+faeb266.zip` |
| zip SHA-256 | `94854AA0201CB7587B490F1DAEA247372EF5AA3D348ABAD9DF0B1978013F6A5C` | `1E34D1874B121B420FF19063E3B73C41FAC988E154C4CD69DA7BFEE01A7DAD0B` |
| size | 39 457 424 B | 39 457 426 B |
| files / uncompressed | 101 / 53 109 154 B | 101 / 53 109 154 B |
| source commit | `faeb266517f4c33d6a496054aea43a58b7a54041` (clean) | same |
| runtime | `runtime-wp3-20261018` | same |

The only difference between the two packages is the version string, so the two zips differ in hash and bytes while
their binaries are identical — which is what an upgrade of the same commit is supposed to exercise.

## B3/B4 — owner walkthrough (NOT RUN)

`tools/package/verify-wp4.ps1` performs every action itself and accepts only Enter / y / n. It records, before and after
each step, the install root, the HKLM/HKCU `CTF\TIP` and COM keys, the scheduled task, running `FusionHost` processes
(path, user, integrity, session), `%APPDATA%\Fcitx5` (count/size and size+mtime+hash of the key files) and the machine
and user `PATH`. It records the development registration found before the run and restores it at the end. Steps (a)–(f)
are as specified in the WP4 instruction; step (c) ends the run and asks the owner to sign out and re-run with
`-Resume`.

**Dry run (done).** With `-DryRun` the script read both packages, extracted them, recorded the pre-run snapshot
(`wp4-run-20261018-100135/snapshots/01-before.txt`), printed step (a)'s prompt and stopped before any elevated action.
The snapshot shows: install root absent, no task, no Host, the development registration at
`evidence\fusion-windows-stage3c-faithful-frontend-v2\deployment\bin\FusionTsf.dll`, and the first-run
`%APPDATA%\Fcitx5` from environment fact 2.

**Pending.** B3a–B3f, B4 (Notepad A–E on the installed wp4.2) and the per-step results are not recorded here. Until the
owner runs the script, WP4 is not closed: the exit gate needs that evidence.

## Known limits at this scope

- An old version directory stays locked until every application that loaded its DLL exits; a later install or uninstall
  removes it. It is reported, never force-deleted.
- Nothing restarts the Host after a crash, and an alive-but-unhealthy Host is not restarted (WP5/U7).
- The task, pipe, claim and stop event are single user × session; a machine with several interactive users needs the
  install per user, and an installer elevated as a *different* administrator cannot open the owner's stop event (it
  reports that instead of killing the Host).
- The windowless Host writes no log file, so its stderr evidence only exists when a caller redirects it.
- `--console` is required by anything that still expects the console subsystem (the WP3 harness pattern); a caller that
  passes no switch gets the windowless Host.
- No MSI, no signing, no distribution channel (U4, WP8); versions are provisional test identifiers.

## Evidence locations

All under `C:\Users\choicky\worktmp\fusion-windows\stage6-wp4\` unless noted:

| what | where |
|---|---|
| A1 console mode / CTRL_BREAK | `a1-console.txt`, `a1-console-host-stderr.log` |
| A3 runtime location and DLL dependencies | `a3-smoke.txt`, `a3-smoke-host-stderr.log` |
| A5 / R23 probe | `a5-probe.txt` |
| B1 build and test logs | `build-mock.log`, `build-debug.log`, `build-release.log`; native runtime logs `native-debug.log`, `native-release.log` |
| B2 packages and manifests | `packages\fusion-0.0.0-wp4.{1,2}+faeb266.zip`, `b2-manifests.txt`, `package-wp4.{1,2}.log` |
| B3 dry run | `wp4-run-20261018-100135\verify-wp4.log`, `wp4-run-20261018-100135\snapshots\01-before.txt` |
| runtime product | `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018\` (Debug + Release) |
