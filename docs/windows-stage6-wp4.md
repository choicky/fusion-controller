# Windows Stage 6 WP4 — packaging and lifecycle (R6, R9, R23, R26; U2)

Date: 2026-10-18. Owner decision recorded: **U2 = (b) Accepted 2026-10-18** (§7.1 of the plan).

**Result: WP4 is CLOSED within its scope on 2026-10-19.** The 2026-10-18 run (`wp4-run-20261018-185004`) met the exit gate except B3c2, which FAILED — a cancelled logoff ended the Host and nothing restarted it; the 2026-10-19 run (`wp4-run-20261019-130037`) then passed every step on the fixed packages: install, second start, sign-out/sign-in, the cancelled logoff (c2), the new cancelled restart (c3), the upgrade with a **running** Host, both uninstall paths and the development-registration restore. **R34** is RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement. Logon autostart is verified (the task started the Host at 18:58:43 and again at the 2026-10-19 sign-in).
*Corrected 2026-10-18 by the event-log evidence (`r34-evidence.txt`): this paragraph said the task "did not start the
Host at the second sign-in". There was no second sign-in — the only sign-out/sign-in of the run is 18:58:39→18:58:43 —
and (c2) was not "not exercised": the logoff was cancelled (User32 1077 at 19:03:20) and the Host had already ended.*
Three earlier attempts are kept below as history — `wp4-run-20261018-115501` (the walkthrough quoted a parameter
*name*), `wp4-run-20261018-124402` (install.ps1 read `$LASTEXITCODE` after a `.ps1` call and aborted a *successful*
registration) and `wp4-run-20261018-170730` (step (a) installed and asserted correctly for the first time, then aborted
on a Notepad check that inspected the launcher process). All three were harness defects, all three were fixed and each
fix proven without UAC; the same root cause — every Notepad window lives in one packaged process — also made step (d)
unsatisfiable, so its order was corrected before the run.

## Candidate, runtime and tooling identities

| item | value |
|---|---|
| fusion-windows base | `572fdb9` (PR #16 merge, `main`) |
| branch / head | `feature/stage6-wp4-packaging` — `7cba293` (implementation), `faeb266` (package file selection), `b050292` (walkthrough launcher), `8b0d9ec` (installer verification mode), `e6f131e` (cancelled-logoff fix, session-end test, machine identity in the manifest, walkthrough steps c/c2), `0d1a1c4` (ignore the local agent workspace), `53597d7` (harness fix after the 11:55 abort), `f63aa4f` (skip the Notepad check when an earlier assertion failed), `ae54e70` (elevated-path fix + full script audit), `2a8b0bb` (forward `-TranscriptPath` to the launched script), `97f90f7` (check the Notepad window's process, not the launcher's PID), `7fd83bf` (step (d) order for the single-process Notepad), `9c7ceec` (harness-owned sign-out blocker; per-user input state removed by uninstall), `e85d497` (hash-skip copy, per-holder reporting, corrected prompts) |
| machine | **5700X** (this WP4 work, its packages and the walkthrough). WP1–WP3 work and evidence, including the WP3 walkthrough and the original `runtime-wp3-20261018` binaries, were produced on the owner's **7700** |
| Release candidate | `stage6-wp4\build-release\bin` — `FusionHost.exe` `BE7FBC4E95706DB4F447A90E27EB56FC8C4B6378680B6E13C2F6DA81A3218CF2`, `FusionTsf.dll` `FEACFCEDBE083976DB9A7DEF72C80337FFB64669BD09FFBAF763CA6C990052E9`, `FusionFcitxHost.dll` `B1F3A8A34664F988CD2CE35D39CE9648BE07A7D31B38B579F0DC38BDF63F085F` |
| runtime | `runtime-wp3-20261018`, **5700X rebuild** (Debug + Release) — same pins as the 7700 build: fcitx5 `a7f2528` + `fcitx5-msvc.patch`, chinese-addons `61474bd` + `chinese-addons-msvc.patch`, libime `171edcf` unmodified, vcpkg `20ceb8ad`; Release `Fcitx5Core.dll` `98675392F46067DB13E442A5ED26EF64C719692069B1174BBE8478E2B4C662AB`. **Not byte-identical to the 7700 build** (see the machine section below) |
| changed files | 16 (`.gitignore`, `CMakeLists.txt`, `README.md`, `src/host/main.cpp`, `src/host/host_server.{hpp,cpp}`, `tests/stage4a_ipc.cpp`, `tools/fcitx/build-native.ps1`, `tools/package/{build-package,install,uninstall,host-state,walkthrough-assertions,verify-wp4}.ps1`, `tools/tsf/{register,unregister}.ps1`) — **no C++/CMake change after `0d1a1c4`**, the commit whose binaries are in both packages |

### Machines and the two `runtime-wp3-20261018` builds

The runtime directory keeps its name because its CMake package files carry absolute paths, but the name alone is not an
identity. There are two builds from the same pins:

| | WP3 (owner's **7700**) | WP4 (this **5700X**) |
|---|---|---|
| source pins | fcitx5 `a7f2528` + `fcitx5-msvc.patch`; chinese-addons `61474bd` + `chinese-addons-msvc.patch`; libime `171edcf`; vcpkg `20ceb8ad` | identical |
| `FusionHost.exe` (candidate) | `6B74E700749A93651FCED28CBB452AF79962A3736B814649A9981A67D3737637` | `BE7FBC4E95706DB4F447A90E27EB56FC8C4B6378680B6E13C2F6DA81A3218CF2` |
| `FusionTsf.dll` (candidate) | `A0C13C0AC580869D4BD9E39CFA4A2EC6550240A3374B602217032E770C41BD80` | `FEACFCEDBE083976DB9A7DEF72C80337FFB64669BD09FFBAF763CA6C990052E9` |
| runtime DLL hashes | not recorded in the WP3 record | in the 5700X package manifests (`b2-manifests.txt`, `package.json`) |

The candidate binaries differ for two reasons that must not be conflated: WP4 changed the Host (windowless mode, stop
event, session-end handling), and the two builds were produced on different machines. Nothing in this record, the plan,
the architecture text or `package.json` claims byte identity with the WP3 build; `package.json` now records
`builtOn` and `runtime.machine` and states that the runtime is a rebuild from the WP3 pins.

### Environment facts that differ from the WP3 record

Recorded because they bound what these results mean, not as product decisions:

1. **The runtime directory did not exist on the 5700X.** `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018`
   was absent at WP4 start (only an older Debug-only prefix from the WP1 era remained). The runtime was therefore
   rebuilt on the 5700X from the same pinned sources and patches with `tools/fcitx/build-native.ps1` into that exact
   directory. Source identity matches WP3's record; the binaries are the 5700X build recorded above, not WP3's.
2. **No WP3 user data was present on the 5700X.** `%APPDATA%\Fcitx5` did not exist before the WP4 run. The A1 console
   check (below) ran without `FUSION_FCITX_SKIP_USER_PATH` and created a first-run default configuration plus an empty
   user dictionary at 09:59. That is recorded as an unintended touch of real user data by a WP4 check (no pre-existing
   data was modified or removed), and it is the state the walkthrough's "before" snapshot contains. Step (a) therefore
   only confirms that the current input method is Pinyin; it no longer asks about learning history, because the 17:07
   Host had already saved the owner's input.
3. **`reg.exe` is blocked on the 5700X** by its Security Center program blacklist, so `VsDevCmd.bat` cannot detect the
   Windows SDK and produces `LIB`/`INCLUDE` without SDK paths (link fails with `LNK1104: kernel32.lib`). The build was
   driven by a local wrapper outside the repository that supplies the SDK `INCLUDE`/`LIB`/`PATH` and then calls the
   repository script unmodified. This is a machine condition, not a repository change.
4. **clang-format available on the 5700X is 23.1.3** (WP3 used 22.1.3). It reports formatting differences in files this
   WP did not touch (`src/tsf/service.hpp`, `src/tsf/diag.cpp`), so the check was scoped to the files this WP changed.
   (The R34 change used the VS-bundled 22.1.3, which is clean on its files.)
5. **Task Scheduler history is disabled on the 5700X** (`Microsoft-Windows-TaskScheduler/Operational`), so no run can
   show *which* trigger started the task. The walkthrough therefore decides from the System log (User32 1077 for a
   cancelled logoff, User32 1073 for a cancelled restart or shutdown — both read without elevation), the session's WTS
   logon time and the Host's own process and save state — never from task history.

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
| hidden top-level window (`WS_POPUP`, `WS_EX_TOOLWINDOW`, not `HWND_MESSAGE`) | `WM_QUERYENDSESSION` answers TRUE and stops nothing; `WM_ENDSESSION` with `wParam` TRUE requests the stop and waits, bounded at 4 s (the console path's bound), for the ordered shutdown; `WM_ENDSESSION` with `wParam` FALSE does nothing |
| named stop event `Local\FusionHost-stop-<user SID>-<session id>` | created by the Host before the backend exists, with the pipe's own user-only DACL; `install.ps1` / `uninstall.ps1` open and signal it instead of terminating the process |
| console control handler | unchanged, console mode only |

All three paths only *request* the stop. `main()` remains the single thread that runs `stop()` →
`shutdown_backend()` → Fcitx's save on instance exit, and the `HostClaim` is held until the process exits. The
stop-event waiter is one thread that waits on the stop event and the exit signal and never drains, joins or touches the
backend. The window lives on its own thread so the accept loop keeps running on the main thread; `WM_CLOSE` ends it
after the exit signal is set.

**A cancelled logoff must not stop the Host** (review finding, fixed before the walkthrough). A logoff or shutdown can
still be cancelled *after* `WM_QUERYENDSESSION` — an application asks to save and the user cancels. Requesting the
stop from the query would leave the session running with no Host until the next logon. The decision now lives in
`fusion::ipc::SessionEndHandler` (host_server), separate from the window procedure so it can be driven without a real
session end: the query only answers TRUE, and only `WM_ENDSESSION(TRUE)` — the session is really ending — requests the
stop and waits.
*Corrected 2026-10-18 by the event-log evidence: this paragraph used to say step (c2) "checks the product behaviour by
cancelling a real logoff", as if the handler made the Host survive one. It does not. The walkthrough cancelled a real
logoff (User32 1077, 19:03:20) and the Host had already ended — Windows delivers `WM_ENDSESSION(TRUE)` application by
application, so an application processed before the blocking one is gone before the Cancel takes effect (R34). The
decision above is still right; surviving the cancel is the task's 1077 event trigger.*

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
`install.ps1`, `uninstall.ps1`, `host-state.ps1`, `package.json`. The owner-test settings package additionally carries
`bin\FusionSettings.exe`, Qt 6.8.3 shared DLLs and `bin\plugins\platforms\qwindows.dll`, `bin\qt.conf`, the pinned
Chinese locale catalogs, scoped settings license notices, corresponding-source archives for the pinned Fcitx runtime,
and the exact fcitx5-qt patch/source instructions. The installer creates the `Fusion Settings.lnk` Start Menu entry;
the tray exposes Settings. Test executables and `FusionTsfProfile.exe` are not packaged. No MSI, no installer toolchain,
no signing (U4 stays open; decided with WP8). The owner-maintained configtool fork source is pinned by exact commit and
its fork PR remains separately reviewable.

**Manifest.** `package.json` records package name, version, source commit and dirty flag, build time, configuration,
runtime identity (label, pinned fcitx5 / chinese-addons / libime commits, patch names) and `size` + `sha256` for every
file. `install.ps1` verifies every hash before copying and refuses on a mismatch.

**Install.** Elevated (refuses otherwise). Order: verify → ask a running Host for an orderly stop → copy into the
version directory, **touching only what differs**: a file whose SHA-256 already matches the manifest is skipped (so a
reinstall over a directory whose identical DLL is still mapped by shell processes does not try to overwrite it), and a
file that must change but is in use fails the step naming the file and every process holding it → register the TIP from
that directory (`tools\register.ps1`) → create or update the logon task →
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
does not appear fails the step and names it, rather than reporting success.

**Failure reporting.** Both scripts take `-TranscriptPath` and write their own transcript (a runas launch cannot redirect
output), set `$currentStep` before every phase, and print `INSTALL FAILED at step '<x>': <message>` (or `UNINSTALL
FAILED …`) with the script position before exiting 1. Every failure — including the integrity check, a Host that will not
stop, and a Host that does not start — leaves through that one path, so a failed step always names itself.

**Upgrade.** A new version directory; the previous Host is stopped through the stop event before any file is copied;
registration and the task are re-pointed at the new directory; the previous directory is removed only if no process has
a module loaded from it (`Test-DirectoryInUse`) and it is not the registered one. A directory a process holds is left in
place and reported, never force-deleted and never partially deleted. A reinstall over an existing version directory
merges the package contents into it instead of nesting them (`bin\bin`).

**Uninstall.** Elevated. Order: ask the Host for an orderly stop (abort if it does not exit within the bound) →
unregister and remove the `CTF\TIP` / COM keys that the COM unregister leaves → **remove this user's per-user input
state** (`HKCU\SOFTWARE\Microsoft\CTF\TIP\{CLSID}`, i.e. the `EnableLanguageProfile` flag, and the language-list tip if
this user has one) → delete the task → remove version directories no process holds → remove the install root if empty.
`%APPDATA%\Fcitx5` is kept unless `-RemoveUserData` is given, which removes only the current user's `%APPDATA%\Fcitx5`
and `%LOCALAPPDATA%\Fcitx5`.

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

`stage4a wp4sessionend` (registered as `stage4a_ipc_wp4sessionend`): drives the session-end decision without a real
session end. `WM_QUERYENDSESSION` returns TRUE and requests nothing; `WM_ENDSESSION(FALSE)` (a cancelled logoff)
requests nothing; `WM_ENDSESSION(TRUE)` requests the stop exactly once and waits for the exit, bounded (measured
against a 300 ms bound); with the exit already signalled the same call returns immediately. A real session end is
exercised by the walkthrough's step (c) (sign-out and sign-in, PASS) and by step (c2): the logoff there was cancelled
(User32 1077) and the Host had already ended — the decision under test is right, but it does not make the Host survive a
cancelled end (R34).
*Corrected 2026-10-18: this paragraph said (c2) "was meant to exercise a cancelled end but the harness could not hold the
sign-out". It did hold it; the Host died anyway.*

## B1 — builds and tests (measured)

Every tree configured fresh in a `chcp 65001` console against the 5700X `runtime-wp3-20261018`:

| tree | build | CTest |
|---|---|---|
| Mock-only Debug | 0 | **71/71** |
| Debug + real Fcitx | 0 | **79/79** |
| Release + real Fcitx | 0 | **79/79** |

`stage4a_ipc_integration` is the operator-driven mode that requires an already-running Host; the suites above use its
documented `FUSION_STAGE4A_SKIP_INTEGRATION=1` switch, and the mode was run separately against a started Release Host
(`a3-smoke.txt`, PASS). `clang-format --dry-run --Werror` is clean on every file this WP changed; `git diff --check` is
clean. (See environment fact 4 for why the formatter check is scoped to the changed files.)

## B2 — packages (measured)

Both packages were rebuilt from the branch head `e85d497` with `tools/package/build-package.ps1`; every hash below is
taken from `b2-manifests.txt` (written from the zips' own `package.json`), never transcribed from a console. The
manifest records `builtOn: 5700X`, `runtime.machine: 5700X` and the note that the runtime is a rebuild from the WP3
pins.

| item | wp4.1 | wp4.2 |
|---|---|---|
| file | `fusion-0.0.0-wp4.1+e85d497.zip` | `fusion-0.0.0-wp4.2+e85d497.zip` |
| zip SHA-256 | `84B97968691DA58BC05C83759EC1B49D4DA626B9FE103FF7DF3A4E32824429A9` | `ED4D9280F25A351AADC8E25D3713AE36BA2CC7FFA62EBB69BC2782A89686C470` |
| size | 39 463 487 B | 39 463 487 B |
| files / uncompressed | 101 / 53 127 565 B | 101 / 53 127 565 B |
| source commit | `e85d497af2c61abf84f69f6df4cecedb44e7d4c4` (clean) | same |
| runtime | `runtime-wp3-20261018` (5700X rebuild) | same |

The only difference between the two packages is the version string, so the two zips differ in hash and bytes while
their binaries are identical — which is what an upgrade of the same commit is supposed to exercise.

**Product binaries are unchanged by the rebuild.** `bin\FusionHost.exe`
`BE7FBC4E95706DB4F447A90E27EB56FC8C4B6378680B6E13C2F6DA81A3218CF2`, `bin\FusionTsf.dll`
`FEACFCEDBE083976DB9A7DEF72C80337FFB64669BD09FFBAF763CA6C990052E9` and `bin\FusionFcitxHost.dll`
`B1F3A8A34664F988CD2CE35D39CE9648BE07A7D31B38B579F0DC38BDF63F085F` are byte-identical to the `0d1a1c4` packages,
because nothing after that commit touched C++ or CMake. Only the scripts inside the package and the manifest's
`sourceCommit` differ, which is why the zip hashes changed while the binaries did not.

**The packaged scripts are the audited ones.** `b2-manifests.txt` compares the six scripts inside each zip with the
repository files at the same head; all twelve comparisons are `MATCH`:

| file in the package | SHA-256 (both packages) |
|---|---|
| `install.ps1` | `7EF82FFCAE2470E41C17C0A84736F9B738F866CB115B933B17D101B9032876D1` |
| `uninstall.ps1` | `9244AAC73EE3D270D784E7DAD2A4976EBA288153E1FE66F5F05E34F67A918582` |
| `host-state.ps1` | `BABA5C0986C6568011086D721773983A01A2B3B94D3F3D514660DB279AF61FF7` |
| `tools/register.ps1` | `7534A2CC10F4DA0818D5F030D2E91D001BA6E6B345EFB11D4EEB8547978891C5` |
| `tools/unregister.ps1` | `6F4FB97581E0C2E3D30F3E0A92953F568C98FCDB146087D0119F6F1A8DF47922` |
| `tools/check-state.ps1` | `F019F0AA447FA2E080C77F529663FC666A00030341279E7712F58E3FC4CF728B` |

`install.ps1`, `uninstall.ps1` and `host-state.ps1` changed in this round (the hash-skip copy, the holder reporting
and the per-user input-state removal), which is why the packages were rebuilt; the other three are unchanged.

## B3/B4 — owner walkthrough (2026-10-18, run `wp4-run-20261018-185004`)

`tools/package/verify-wp4.ps1` performs every action itself; the owner never edits a file or types a command. Prompts
accept Enter / y / n, or the one short answer a prompt asks for (a pinyin string, a rank such as `5→1`), and every
answer is written to the run log — a reported observation is never reduced to "Enter". Before and after each step it
records the install root, the HKLM/HKCU `CTF\TIP` and COM keys, the scheduled task, running `FusionHost` processes
(path, user, integrity, session), `%APPDATA%\Fcitx5` (count/size and size+mtime+hash of the key files) and the machine
and user `PATH`. It records the development registration found before the run and restores it at the end.

| step | what the owner does | what the script records |
|---|---|---|
| a | allow UAC, close every Notepad (the script waits for that), check the taskbar input indicator (Win+Space only if Fusion is not already selected), type a few letters in the Notepad the script opened, press Enter | install exit code (fatal if non-zero), Host pid/session/user/integrity, HKLM/HKCU registration, "no Notepad window and no process holding a foreign Fusion DLL", which Notepad process was checked and why (window owner vs launcher), its loaded DLL + hash, `你好` yes/no, **"the current input method is Pinyin (not Shuangpin)" yes/no** |
| b | nothing (the script starts the task again and `FusionHost.exe` directly) | Host count after each attempt, the direct start's exit code and stderr, typing still works |
| c | type a pinyin string, select a candidate that is **not** rank 1 before learning, report `N→M` (a single number is rejected); then sign out | Host running before the learning prompt, the pinyin string, the `N→M` pair, `%APPDATA%\Fcitx5` snapshot; then the exact `-Resume` command |
| c (resume) | sign in, close every Notepad (the script waits), type the same string in the Notepad the script opened, report the rank | the same assertions as step a, typing works, the current rank, whether the learning survived |
| c2 | start a logoff and **cancel** it at Windows' "these apps are preventing you from signing out" screen — the blocker is the walkthrough's own window, titled `WP4 test: click Cancel`; the terminal may close even though the cancel worked | the blocker (handle, block reason read back with `ShutdownBlockReasonQuery`, FALSE to a synthetic `WM_QUERYENDSESSION`); **session id, WTS logon time and the newest User32 1077 before and after**; the verdict from that evidence ("cancelled" = same session/logon + a newer 1077, "signed out" = new session/logon); a Host in this session within 60 s, with the old/new pids and the save time |
| c3 | start a **restart** and cancel it the same way — the blocker is a second one in the same process, titled `WP4 test: cancel the restart`; the terminal may close here too | the blocker's second start (handle, block reason), FALSE to `WM_QUERYENDSESSION`; **session id, WTS logon time and the newest User32 1073 before and after**; the verdict from that evidence; a Host in this session within 60 s, with the old/new pids and the save time |
| d | keep a Notepad open — **or, if none is open, open one and type a few letters with Fusion first** — allow UAC for the upgrade, type Chinese in that same window, then close every Notepad **including each tab** and type in the fresh one; Notepad A–E there | the pre-upgrade process + its loaded wp4.1 DLL; install exit code and how many files were copied vs skipped as identical; Host + registration for wp4.2 (**no Notepad assertion yet**); the old window's y/n, its DLL (still wp4.1), whether the process is alive, whether the wp4.1 directory exists and **every process holding it** (shell processes are expected); the "close all + poll" result; the fresh process's pid + loaded wp4.2 DLL + hash; A–E; whether the wp4.1 directory is free now (not removed here) |
| e | allow UAC for the uninstall (the script asks for Notepads to be closed first) | exit code, no TIP/COM/task/Host, install root, **the holders of the wp4.2 directory before and after** (shell processes are expected and are not a failure), **HKCU `CTF\TIP` and the language-list tips before and after**, `%APPDATA%\Fcitx5` kept, `check-state.ps1` output |
| f | close every Notepad (the script waits), allow UAC twice | **whether the wp4.2 directory was still held before the reinstall** (identical files are skipped, so the reinstall succeeds over a held directory), reinstall and `-RemoveUserData` exit codes, user data removed, nothing else left, **HKCU `CTF\TIP` and the language-list tips before and after** |
| end | — | the development registration restored and re-snapshotted |

The learning used by steps (c)/(d) is produced inside the walkthrough with the Host already running (step (c) refuses
to continue if no Host is running), so the owner is never asked to do something the machine cannot do at that moment.

### Results

Machine 5700X, Windows 11, 175 % scaling, one monitor. Packages `fusion-0.0.0-wp4.1+e85d497.zip` (SHA-256
`84B97968…29A9`) and `fusion-0.0.0-wp4.2+e85d497.zip` (`ED4D9280…C470`). **Both packages contain identical binaries**
(`FusionHost.exe` `BE7FBC4E…`, `FusionTsf.dll` `FEACFCED…052E9`, `FusionFcitxHost.dll` `B1F3A8A3…F085F`), so the upgrade
exercises **version-directory switching, not a binary change** — that is exactly what this run can prove about upgrade.

| step | verdict | evidence |
|---|---|---|
| a | **PASS** | install exit 0; Host, registration and the Notepad module asserted; Pinyin confirmed |
| b | **PASS** | a second Host was refused; typing still worked |
| c | **PASS** | learning survived sign-out/sign-in — the owner reports `shi` still rank 1 after re-logon (note below). The mtime that advanced at 19:03:13 belongs to (c2)'s cancelled logoff, not to this sign-out (which happened at 18:58:39) |
| c2 | **FAIL** (product) | the logoff **was cancelled** — User32 1077 "user 5700X\choicky attempted to log off computer 5700X and failed" at 19:03:20 — and no Host was running afterwards: it had saved at 19:03:13 and ended. R34 |
| d | **PASS** | upgrade with a Notepad open — details below |
| e | **PASS** | Host 8124 stopped in order and Fcitx saved on stop (`user.dict` 406 → 768 B, `user.history` 170 → 309 B, mtime 19:16:45); HKLM COM/TIP and the task removed, no Host, `%APPDATA%\Fcitx5` kept, current-user HKCU `CTF\TIP` enable flag 1 → none |
| f | **PASS** | the reinstall succeeded **while the wp4.2 directory was held by `explorer`** (hash-skip copy), task Running, Host 14940; the uninstall stopped it, `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5` were deleted, registration and task removed |
| end | **PASS** | development registration restored (`register.ps1` exit 0); HKCU enable flag back to 1 |

**This table is the 2026-10-18 run and stays as history.** The 2026-10-19 run (`wp4-run-20261019-130037`) re-ran
every step on the fixed packages and passed; its results are the verification block in the R34 section below.

**(c) harness note.** The prompted word `shuangpin` was already rank 1 on this machine, so the owner substituted `shi`.
Provenance of the ranks: the owner reports `shi` was 4th before learning and 1st after learning, and still 1st after
sign-out/sign-in; the script's rank prompt (example `5→1`) recorded only `1` in `verify-wp4.log`, so the `4→1` pair is
the owner's report, not the log's.

**(c2) what happened.** The logoff was cancelled — the terminal closing does **not** mean the session ended — and the
Host did not survive it: Windows delivers `WM_ENDSESSION(TRUE)` to applications one after another during a logoff, so an
application processed before the blocking one has already ended when the Cancel takes effect. That is **R34** — RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement. The fix is an event trigger on User32 1077 (cancelled logoff) or 1073 (cancelled restart/shutdown) that starts the Host again.
*Corrected 2026-10-18 by the event-log evidence (`r34-evidence.txt`): this section recorded (c2) as "not exercised —
the blocker died with its terminal". The blocker did hold the logoff (1077 at 19:03:20) and the session never ended
(`query user`: session 2, logon time 18:58, still active at 20:06).*

**(d) upgrade with a Notepad open.**

- The pre-upgrade Notepad (pid 11884) had the wp4.1 DLL. After the upgrade it was still alive, still on the **wp4.1**
  DLL, and typed Chinese through the new **wp4.2** Host — old DLL + new Host: PASS.
- Host after the upgrade: pid 8124, wp4.2 path, Medium integrity, session 2. HKLM at the wp4.2 DLL, HKCU none.
- The wp4.1 directory was retained while in use (holders: `firefox`, Notepad) — expected.
- A fresh Notepad (pid 13484, launcher 34592) loaded the wp4.2 DLL with the manifest SHA-256.
- **Correction (R18).** The recorded answers "new Notepad can type → n" and "A → n" were caused by **R18** — a new
  Notepad window starts with the IME closed. After Ctrl+Space the owner confirmed `nihao`+Space commits 你好, so both are
  **PASS** with that note; the raw answers stay visible as "recorded n, corrected: R18".
- Firefox: `nihao`+Space → 你好, PASS.
- B, D, E: PASS. C: PASS — `,` and `.` give Chinese punctuation; `;` and `www.` show the QuickPhrase prompt, which is
  Fcitx behaviour (`;` is the QuickPhrase trigger, matching the WP1 record). The question's wording ("`;` gives Fcitx
  punctuation") was wrong — a harness follow-up, not a product result.

**Leftover after the run.** `C:\Program Files\Fcitx5 Fusion\` remains with both version directories because shell and
browser processes still hold the DLLs; it is to be removed after a restart (manually, because step (f) already removed
the task and the registration). That is the documented behaviour of R28 — report the holders, remove after sign-out —
not a defect.

### R34 — a cancelled logoff, restart or shutdown ends the Host (RESOLVED — verified 2026-10-19)

**RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement.**

The 2026-10-18 run's (c2) is a product failure, and the event log says why. `r34-evidence.txt` (owner's read-only collection,
2026-10-18 20:06, elevated):

| time | event |
|---|---|
| 18:58:39 → 18:58:43 | the run's **only** sign-out/sign-in (LocalSessionManager 23/21, Winlogon 7002/7001, Security 4647 then 4624 type 2); the task started the Host at 18:58:43 |
| 19:03:13 | the Host saved (`user.dict`/`user.history`/`profile`/`pinyin.conf` mtimes) and ended |
| 19:03:20 | System, User32 **1077**: "user 5700X\choicky attempted to log off computer 5700X and failed" — the Cancel worked |
| 20:06 | `query user`: session 2, console, logon time 18:58, still active — the session never ended |

**Which event a cancelled attempt writes** (owner's measurement, 2026-10-19, `b1b-restart.txt` and `b1b-shutdown.txt`;
the shutdown's terminal was closed by the attempt, so its events were collected afterwards from the baseline record):

| attempt | User32 event | evidence |
|---|---|---|
| cancelled logoff | **1077** | 2026-10-18 19:03:20, RecordId 104140, `Security/@UserID` = this user |
| cancelled restart | **1073** | 2026-10-19 11:16:08, RecordId 104461 (after User32 1074 "restart requested" at 11:16:03), same SID |
| cancelled shutdown | **1073** | 2026-10-19 11:23:50, RecordId 104462, same SID |
| completed restart | **none** | EventLog 6006 at 11:24:41, Kernel-General 13 at 11:24:47, boot 11:25:02 — no 1073 |

Both cancel events carry the user's SID in `System/Security/@UserID`, and neither is written when the attempt completes.
**The order in which Windows closes applications varies between attempts**: the terminal running the probe survived the
cancelled restart and was closed by the cancelled shutdown. That is direct support for the inference below — whether the
Host survives a cancel is luck, not design.

**Mechanism (inference, not a proven cause):** Windows delivers `WM_ENDSESSION(TRUE)` to applications one after another
during a logoff, so an application processed before the blocking one has already ended when the Cancel takes effect. The
Host's only save trigger without a stop-event signal is `WM_ENDSESSION(TRUE)`, which it received; `LastTaskResult`
`0x40010004` is consistent with the process then being terminated. The handler's decision stays correct — the wrong
assumption was that a cancelled end only ever delivers `ENDSESSION(FALSE)`.

**Fix (direction A, owner-accepted 2026-10-18 as an implementation detail of U2 (b)):** the task gains a System-log
`User32` **1077 or 1073** event trigger scoped to the installing user's SID (`Security[@UserID='<SID>']`), delayed
`PT10S` so the old Host has released `HostClaim`; the logon trigger and every other setting stay, and the install fails
if the read-back does not carry both ids. A latent defect found in the same review is fixed too: an external `WM_CLOSE`
(`taskkill` without `/f`) used to destroy the hidden session-end window and leave the Host running with no logoff path;
it now requests the orderly stop and keeps the window.

**Scope: all three cancelled ends.** The walkthrough verifies the cancelled **logoff** (c2) and the cancelled **restart**
(c3); the cancelled **shutdown** is covered by the event measurement above — the same id, the same SID scoping — rather
than by its own step, because a cancelled shutdown closes the terminal the walkthrough runs in more often than a restart
does and the id is identical to the restart's.

**Review defect found in the first version of the trigger (fixed in #18).** The subscription used `@UserID` on
`System`, but `UserID` is an attribute of `System/Security`, so the XPath selected nothing and the trigger could never
fire; the `QueryList` was also missing `</Query>`, which `Get-WinEvent` tolerates and the task registration does not.
The query is now `Security[@UserID='<SID>']` over `(EventID=1077 or EventID=1073)` and is proven against the records
(`proof-a1.txt`, five queries): the exact subscription string that `install.ps1` builds returns 104140 (1077,
2026-10-18 19:03:20), 104461 and 104462 (1073, 2026-10-19 11:16:08 and 11:23:50); the same query bounded to
11:24:00–11:26:00 (the completed restart) returns nothing; another SID returns nothing; and the old `@UserID` form
returns nothing. `install.ps1` validates the query before registering — a valid query with no match is accepted, a
malformed one fails the install at step 'task' (`proof-xpath-validation.txt`).

**Harness defects found while adding (c3)** (all fixed in #18, all proven without a restart):

1. **The shutdown blocker could not start twice in one process.** `RegisterClassW` failed with
   `ERROR_CLASS_ALREADY_EXISTS` on the second start and the static state was not reset, so `Start` returned false. (c3)
   needs a second blocker in the same process when the terminal survives (c2). The class is now registered once and its
   re-registration is tolerated, every field is reset on `Start`, and `Stop` leaves the class reusable
   (`proof-a2-a4.txt`: start → stop → start both report `started=True` with a registered block reason).
2. **`probe-b1b.ps1` could not run and could lose its events.** It did not dot-source `host-state.ps1`
   (`Get-FusionHostState` was undefined), and it had no way to read the log after the attempt killed its terminal. It now
   dot-sources the helper, writes the baseline record before the blocking prompt, and takes
   `-CollectOnly -OutFile <file>` to append every event newer than that baseline from a fresh terminal — proven against
   the 11:16 restart file (`probe-collectonly.txt`), and its append keeps the file's own encoding.
3. **`Get-SessionIdentity` read the wrong field — twice.** WTS class 24 is `WTSSessionInfo`, i.e. a `WTSINFOW`; reading
   its first 8 bytes as a FILETIME returned State + SessionId, so every session reported `1601-01-01 00:07:09`. The
   first fix parsed the structure but omitted the six DWORD counters that follow `SessionId` (incoming/outgoing bytes
   and frames, incoming/outgoing compressed bytes), which shifted every later field by 24 bytes — the field named
   `LogonTime` read `ConnectTime`. The two are equal on a console session, which is why that first proof matched
   `query user`, and differ after a disconnect/reconnect (fast user switching, RDP). The counters are now declared in
   order, the Unicode field sizes and the sanity checks are unchanged, and `FusionWts.Query` exposes both timestamps
   and the two sizes so the layout is checkable. Proof (`proof-wts.txt`): `Marshal.SizeOf` **216** equals the **216**
   bytes WTS returned; `ConnectTime` 2026-10-19 11:25:12 and `LogonTime` 11:25:13 are distinct, and `LogonTime` matches
   `query user` (11:25).

**Verification — owner walkthrough, run `wp4-run-20261019-130037` (2026-10-19).** Machine 5700X, Windows 11, 175 %
scaling, one monitor. Scripts from fusion-windows branch head `3279293`; packages `fusion-0.0.0-wp4.1+5f357b2.zip`
(`6FE297B4…59E4`) and `fusion-0.0.0-wp4.2+5f357b2.zip` (`FB5FA711…2247`); installed `FusionTsf.dll` SHA-256
`1BC7907B…0F83`.

| step | verdict | evidence |
|---|---|---|
| a | **PASS** | install exit 0; task triggers `Logon;Event` (count 2), subscription `User32 (1077 or 1073)` scoped to `S-1-5-21-…-1001`; Host 18324, Medium, session 1; HKLM at wp4.1, HKCU none; the Notepad module is the installed DLL; 你好; Pinyin. **The two leftover directories from the 2026-10-18 run (`0.0.0-wp4.1+e85d497`, `0.0.0-wp4.2+e85d497`) were removed by this install** (`still-present=none`) — the first observed instance of R28's "a later install removes it" |
| b | **PASS** | re-running the task kept 1 Host; the direct start exited 1 ("another Host already owns this pipe name; exiting without touching user data"); typing OK |
| c | **PASS** | a real sign-out/sign-in (session 1 → 2, logon 13:34:44); the logon task started Host 30376 (Medium, session 2); `shi` 6→4 before sign-out (the enforced `N→M` pair), rank 1 after sign-in, and the owner confirmed the learning survived |
| c2 | **PASS** | logoff cancelled — User32 1077 record 104831 at 13:37:09, same session and logon time; the old Host 30376 saved (13:37:04) and ended; a new Host 14524 (Medium) ran in the same session within the bound; typing OK; blocker closed |
| c3 | **PASS** | restart cancelled — User32 1073 record 104834 at 13:38:02, same session; the old Host 14524 saved (13:37:57) and ended; a new Host 32528 ran in the same session; the blocker's second start in the same process worked (A2 on the product path) |
| d | **PASS** | upgrade with a **running** Host — the old Host 32528 stopped in order (save advanced 13:37:57 → 13:39:06), none still alive, new Host 35252 from wp4.2; the old Notepad 36220 stayed on the wp4.1 DLL and typed through the new Host; wp4.1 held by `explorer` + Notepad (expected); a fresh Notepad 28360 loaded the wp4.2 DLL with the manifest hash; new-Notepad typing and A–E all PASS |
| e | **PASS** | uninstall exit 0; HKLM COM/TIP, the task and the Host gone; `%APPDATA%\Fcitx5` kept; the HKCU TIP enable flag removed; directories retained because held (wp4.1 by `explorer`, wp4.2 by `firefox`) — R28 expected |
| f | **PASS** | reinstall exit 0 over a held wp4.2 directory, the event trigger present after the reinstall; the `-RemoveUserData` uninstall exited 0; `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5` deleted; nothing else left |
| end | **PASS** | the development registration restored; the HKCU enable flag back to 1 |

**(c) rank note.** The owner reports `shi` 6→4 before sign-out and rank 1 after sign-in, and confirms the learning
survived. The further improvement after sign-in most likely comes from the owner selecting the candidate again during the
post-sign-in check; recorded as reported, conclusion "learning retained (not back to 6)".

**The cancelled shutdown has no step of its own.** It is covered by the 2026-10-19 measurement above — the same event
id 1073, record 104462, matched by the subscription proof in `proof-a1.txt` — not by a walkthrough step.

**Verification without UAC:** Mock 71/71, Debug 79/79, Release 79/79; `wp4sessionend` covers the close decision; a real
staged Release Host exits 0 in 0.3 s on `taskkill /pid` (no `/f`) with "Save before exiting" and "shutting down"; the
pre-fix package leaves the Host running under the same test. The packages are
`fusion-0.0.0-wp4.1+5f357b2.zip` (`6FE297B4…59E4`) and `fusion-0.0.0-wp4.2+5f357b2.zip` (`FB5FA711…2247`), 101 files
each, all twelve packaged-script hashes `MATCH`, `-VerifyOnly` exits 0 ("101 files match"), and the dry run is
`wp4-run-20261019-121935`. They were built from `5f357b2`; the only later change is
`tools/package/walkthrough-assertions.ps1`, which `build-package.ps1` does not package (it packages `install.ps1`,
`uninstall.ps1` and `host-state.ps1`, plus `register.ps1`/`unregister.ps1`/`check-state.ps1`), so the packages and their
twelve hashes still hold and the walkthrough runs from the branch head `3279293`.

### Harness follow-ups from the WP4 run (fixed in the R34 change)

1. **(c)** the prompted word may already be rank 1 — it was here (`shuangpin`), and the owner had to substitute `shi`.
   The rank prompt also accepts a single number, so a before→after pair is not enforced.
2. **(c2)** the verdict was taken from the tester's impression ("the terminal closed, so the session ended"). It must be
   taken from evidence — session id, logon time and the User32 1077 record — and a cancelled logoff ends the Host, so a
   PASS needs a Host running afterwards, not the same pid.
3. **(d)** question C's wording is wrong: `;` is the QuickPhrase trigger, not punctuation.
4. **(d)** R18 makes "new Notepad" questions fail unless the IME is opened with Ctrl+Space, and the harness does not tell
   the tester — it recorded two false "n" answers because of that.

All four are fixed in the R34 change (fusion-windows #18): the (c2) verdict is now evidence-based, `Ask-RankChange` enforces `N→M`, C is reworded, and the tester is told about Ctrl+Space.

### Harness follow-ups from the 2026-10-19 run (recorded, not fixed)

1. **`Get-FusionSaveStamp` prints UTC while the session logon times print local time**, and neither value says which clock
   it is (`LastWriteTimeUtc` vs `ToLocalTime`). Display only — every comparison uses one clock, so no result depends on
   it. Owner: whoever next touches `verify-wp4.ps1`; trigger: WP6/WP7 harness work.
2. **(c) accepts any `N→M` before sign-out, but the post-sign-in rank can improve further** if the tester selects the
   candidate again while checking (this run's `6→4` became rank 1). Testers should only observe after sign-in, and the
   prompt should say so. Owner and trigger as above.

### What the 2026-10-18 run did not exercise

The upgrade path "stop the **running** old Host before switching versions" was not exercised in the 2026-10-18 run, because the cancelled logoff of (c2) had already ended the Host and nothing restarted it (R34). The same ordered-stop mechanism was exercised in (e) and (f). The 2026-10-19 run exercised it directly in its (d): the old Host was stopped in order, the save advanced, and a new Host started from wp4.2.

### The 11:55 run — aborted, invalid (harness defect, not a result)

`wp4-run-20261018-115501` is **not** a walkthrough result. Step (a) failed before touching anything:

| observation | evidence |
|---|---|
| the elevated transcript runs `& install.ps1 '-PackageDir' '<path>'` | `install-wp4.1.log` |
| `TerminatingError(): "package.json not found in -PackageDir"`, exit 1 | same; `[a] install.ps1 退出码 => 1` |
| nothing installed: no `C:\Program Files\Fcitx5 Fusion`, no task, no Host | `snapshots/02-after-install-wp4.1.txt`, registry/task queries |
| Notepad kept the previously registered development DLL | HKLM `InprocServer32` and the module list of the running Notepad both `evidence\…-v2\deployment\bin\FusionTsf.dll`, sha256 `03DEDD4CB251D82D4E6B674449067D763E1D6E0D266913EB75856B6F09935B92` |
| that DLL needs no Host | `dumpbin /dependents`: `FusionFcitxHost.dll` + debug CRT → Fcitx runs in-process; it has no candidate surface |

**Root cause.** `Invoke-ElevatedScript` single-quoted every element of the argument list, including the parameter
*name*. A quoted `-PackageDir` is a string, so PowerShell bound it positionally to `$PackageDir` and the real path was
dropped. Reproduced in isolation: quoted → `PackageDir=[-PackageDir]`; unquoted → `PackageDir=[C:\some\path]`. The
script then continued to its typing questions, which is why the owner's "y" for step (a) means nothing.

Two things this run proves about the record, not the product: a **profile name never identifies a build** (the
registered description is "Fusion Stage 3B (Real Fcitx5 PoC)", and the loaded module was the Stage 3C deployment), and
an owner's yes/no cannot substitute for an assertion on the loaded module.

**Harness fixes** (`53597d7`; harness and installer message only, no C++/CMake change, packages from `0d1a1c4` stay
valid):

1. `Format-ArgumentList` (new `walkthrough-assertions.ps1`) emits parameter names bare and quotes only values; every
   elevated call site passes a hashtable, and the dry-run command text uses the same formatter.
2. **Fail fast**: any step whose command exits non-zero stops the walkthrough at once — it records the step, the
   command and the exit code, runs the safe cleanup (orderly stop of an installed Host; restore the development
   registration only if the run changed it; never `%APPDATA%`), and asks no further question. `check-state.ps1` is
   covered too.
3. **Assertions before every typing question** (step a, after sign-in, after the upgrade): a `FusionHost` from
   `<install>\<version>\bin` running as the owner at Medium integrity in this session; HKLM `InprocServer32` pointing
   at `<install>\<version>\bin\FusionTsf.dll` with no HKCU override; and a Notepad that really loaded that DLL
   (module list + the manifest's SHA-256). A fresh Notepad loads the TIP only once Fusion is active in it, so a
   "not loaded" result asks the owner to switch with Win+Space and re-checks, bounded.
4. `install.ps1` names the `-PackageDir` value it received when `package.json` is missing.

**Proof without UAC** (`harness-proof.txt`): the new form binds `-PackageDir` to the real path and `-VerifyOnly`
verifies 101 files for **both** packages (exit 0); the old quoted form reproduces `package.json not found in
-PackageDir` (exit 1). Against a Host started from the extracted wp4.1 package and a Notepad the proof started:
`Test-FusionHostFrom` passes for that directory (`medium`, session 1, `5700X\choicky`) and returns null for the wp4.2
directory; `Test-RegistrationPointsTo` passes for the registered development DLL and fails for the package DLL with the
exact path mismatch; `Test-NotepadLoadedDll` passes on the application that has the DLL loaded and fails when expecting
the package DLL, while a freshly started Notepad reports "not loaded" (the case the walkthrough now retries on). The
aggregate `Test-InstalledTipInUse` therefore returns `Ok=false` with both failures named — the correct answer on a
machine where nothing is installed — and the Host was stopped through its own stop event (exit after 130 ms). The
machine was left exactly as before: HKLM still points at the development DLL, `%APPDATA%\Fcitx5` keeps the same five
files with the same hashes and mtimes, no install root, no task, no Host.

**Dry run (done, post-fix).** With `-DryRun` the script read both packages, extracted them, recorded the pre-run snapshot
(`wp4-run-20261018-165349/snapshots/01-before.txt`), printed step (a)'s prompt — `install.ps1 -PackageDir '<path>'` —
and stopped before any elevated action. The snapshot shows: no task, no Host, the development registration at
`evidence\fusion-windows-stage3c-faithful-frontend-v2\deployment\bin\FusionTsf.dll`, the first-run `%APPDATA%\Fcitx5`
from environment fact 2, and — as expected after the 12:44 abort — the leftover version directory
`0.0.0-wp4.1+0d1a1c4\` with 102 files whose `bin\FusionHost.exe` (`BE7FBC4E…`) and `bin\FusionTsf.dll` (`FEACFCED…`)
are the package binaries. Step (a) records the version directories before and after the install, so the run recorded
whether `install.ps1` removed that directory as an older, unused version; nothing held its DLL, so it should have been
removed.

**Installer integrity gate exercised before the owner run** (introduced at `8b0d9ec`, re-proven at `2a8b0bb`):
`install.ps1 -VerifyOnly` is read-only and needs no elevation. On an untouched extracted package it verifies all 101
files and exits 0; after appending one byte to `bin\FusionHost.exe` it exits 1 and reports
`INSTALL FAILED at step 'verify': Package integrity check failed: … bin/FusionHost.exe (sha256 mismatch)`. Both install
and uninstall refuse to run unelevated (exit 2).

**Harness defect found and fixed before the owner run** (`b050292`): the script launched its elevated steps through
`Start-Process`, which builds an environment dictionary and fails when both `http_proxy` and `HTTP_PROXY` are defined —
it would have stopped the walkthrough at its first elevated action. The launcher now uses `ProcessStartInfo` with
`UseShellExecute` + `runas`. The dry run above is the post-fix run; the packages were rebuilt from `b050292` so that
"the packages come from the PR head" stays true (the packaged files are unchanged; only `package.json`'s source commit
differs, which is why the zip hashes changed).

### The 12:44 run — aborted, invalid (defect in the elevated span)

`wp4-run-20261018-124402` is **not** a walkthrough result either. The command was correctly formed this time
(`install.ps1 -PackageDir '<path>'`), the owner approved UAC, and the install got most of the way through — then died on
its own strict-mode reading of a variable:

| observation | evidence |
|---|---|
| verification, orderly-stop check, copy and registration all succeeded | `install-wp4.1.log`: `101 files verified`, `no running Host`, `102 files installed`, `Registered (elevated): …\0.0.0-wp4.1+0d1a1c4\bin\FusionTsf.dll` |
| then `TerminatingError(): 检索不到变量"$LASTEXITCODE"，因为未设置该变量。` at `install.ps1:110`, exit 1 | same transcript |
| the failed install left its version directory behind | `C:\Program Files\Fcitx5 Fusion\0.0.0-wp4.1+0d1a1c4\`, 102 files; `bin\FusionTsf.dll` `FEACFCED…052E9`, `bin\FusionHost.exe` `BE7FBC4E…18CF2`, both matching the package |
| no task, no Host, registration back at the development DLL | abort cleanup ran (`register.ps1` exit 0); HKLM `InprocServer32` = `evidence\…-v2\deployment\bin\FusionTsf.dll`; task absent |
| `%APPDATA%\Fcitx5` untouched | five files, same sizes, mtimes 09:59:09 and hashes as before the run |

**Root cause.** `install.ps1:110` read `$LASTEXITCODE` after calling `register.ps1`; `uninstall.ps1:65` had the identical
pattern after `unregister.ps1`. A `.ps1` call sets `$LASTEXITCODE` only when the callee calls `exit`, and
`Set-StrictMode -Version 2.0` turns a read of the unset variable into a terminating error — so a **successful**
registration aborted the install, before the task and the Host start.

Measured, because the details matter more than the summary:

| call | `$?` | `$LASTEXITCODE` |
|---|---|---|
| script succeeds, no `exit` | `True` | **unset → the read throws** |
| script `exit 1` | `False` | `1` |
| script `throw` (caught) | `False` | unchanged |
| script `Write-Error` with `$ErrorActionPreference='Stop'` | `False` | unchanged |

The variable also cannot be cleared before the call: it lives in the global scope, so a `Remove-Variable` in the script
scope leaves it visible — a stale `1` from an earlier child was read as a *successful* call's status. `$?` is therefore
the only sound contract, with `try`/`catch` for the message.

**Why it was not caught.** The only thing that had ever run was `install.ps1 -VerifyOnly`, non-elevated. Everything after
verification — copy, register, task, start, cleanup — had never executed once, and the defect sat exactly there.

### Audit of the package scripts

One complete pass over the scripts as they run when copied into the package, because the elevated span can only be
executed by the owner. Findings:

| # | finding | files | verdict |
|---|---|---|---|
| 1 | `$LASTEXITCODE` read after a `.ps1` call | `install.ps1:110`, `uninstall.ps1:65` | **fixed** — `Invoke-PackageScript` (in `host-state.ps1`) uses `$?` + `try`/`catch` only |
| 2 | the elevated wrapper derived its exit code from `$LASTEXITCODE` after `& script` | `walkthrough-assertions.ps1` (`Invoke-ElevatedScript`) | **fixed** — `$?` decides success; `$LASTEXITCODE` only refines a non-zero code |
| 3 | `(… Where-Object …).Prop` on an empty pipeline throws under strict mode | `install.ps1` (cleanup), `uninstall.ps1` (unregister), `verify-wp4.ps1` ×4 (abort cleanup, pre-run, step e, step f) | **fixed** — `Get-FusionRegisteredDll`, or `@(…)[0]` after a count check |
| 4 | `.Count` / indexing on a value that can be a scalar or `$null` | `install.ps1` (file count, `$started`), `uninstall.ps1` (`$kept`, root emptiness), `verify-wp4.ps1` (`$zips`, `$hosts`, `$before`, `$remaining`), `host-state.ps1` (`$result`, `$pids`) | **fixed / OK** — each is wrapped in `@()` or guarded by a count check; the ones left are arrays by construction |
| 5 | `$process.StartTime` throws for a process that just exited | `host-state.ps1` (`Get-FusionHostState`) | **fixed** — guarded, recorded as `null` |
| 6 | `Start-ScheduledTask` on a missing task throws a raw error instead of a named step | `verify-wp4.ps1` step (b), `install.ps1` | **fixed** — step (b) stops through `Stop-Walkthrough`; `install.ps1` throws inside its own step |
| 7 | `$task.Execute` / `LastResult` on a `$null` task | `install.ps1` | **fixed** — the task is read back and checked before use |
| 8 | a synchronous `ReadToEnd` on the second Host start can block forever | `verify-wp4.ps1` step (b) | **fixed** — asynchronous read, bounded `WaitForExit(15000)`; a still-running process is asked to stop through its own event, never killed |
| 9 | `Remove-Item -Recurse -Force` on a directory holding a loaded DLL deletes part of the tree before failing | `install.ps1` cleanup, `uninstall.ps1` removal | **fixed** — `Test-DirectoryInUse` first; a directory a process holds is kept and reported, never partially deleted |
| 10 | `Copy-Item <dir> -Destination <existing dir> -Recurse` nests the source (`bin\bin`) on a reinstall | `install.ps1` copy | **fixed** — the contents are copied into the target |
| 11 | `$ErrorActionPreference` not restored if the native `check-state` call throws | `verify-wp4.ps1` step (e) | **fixed** — `try`/`finally` |
| 12 | `Resolve-Path` throws on a missing DLL, bypassing the `exit 2` below it | `tools/tsf/register.ps1`, `tools/tsf/unregister.ps1` | **fixed** — `Test-Path` first; both scripts now always `exit` (0/1/2), so the caller's contract is unambiguous |
| 13 | native stderr becoming a terminating error (the R23 class) | `build-package.ps1` (git), `verify-wp4.ps1` (check-state) | **OK** — `git`'s exit code is checked directly after the native call (the correct use of `$LASTEXITCODE`); `check-state` is called with the preference lowered |
| 14 | `Start-Process` / `runas` failing in this environment | `verify-wp4.ps1` | **OK** — `ProcessStartInfo` + `UseShellExecute` + `runas`, as fixed at `b050292` |
| 15 | paths with spaces passed unquoted to native tools | `install.ps1`, `uninstall.ps1`, `register.ps1`, `unregister.ps1` | **OK** — `regsvr32` gets `"<path>"`; no `schtasks`/`reg` is used (the `*-ScheduledTask` cmdlets take the path as an argument) |
| 16 | anything else that would throw only in the elevated span | `install.ps1`, `uninstall.ps1` | **fixed by construction** — every phase sets `$currentStep`, and one `catch` reports it |

**Self-describing failures.** `install.ps1` and `uninstall.ps1` now take `-TranscriptPath` and start their own transcript
in a `try`/`finally`, so a failure inside the elevated span leaves a log written by the failing process itself (a runas
launch cannot redirect output). Every phase sets `$currentStep` — `manifest`, `verify`, `stop`, `copy`, `register`,
`task`, `start`, `cleanup` for install; `stop`, `unregister`, `task`, `remove`, `userdata` for uninstall — and the single
`catch` prints `INSTALL FAILED at step '<x>': <message>` (or `UNINSTALL FAILED …`) with the script position before
exiting 1. Integrity, stop and start failures now `throw` instead of exiting with their own code, so **every** failure
leaves through that one path. The walkthrough passes `<runDir>\install-<version>.log` and
`<runDir>\uninstall-<version>.log` and starts no second (nested) transcript for those two calls.

### Proof without UAC (third round)

`proof-scripts.txt` (UTF-8 copy: `proof-scripts-utf8.txt`). Nothing here installs anything.

**The contract, with stub scripts under the same strict mode** (`Invoke-PackageScript`, the function `install.ps1` now
uses):

| stub | `Ok` | `Threw` | `Error` |
|---|---|---|---|
| succeeds, no `exit` | `True` | `False` | — |
| `exit 0` | `True` | `False` | — |
| `exit 1` | `False` | `False` | — |
| `throw "stub boom"` | `False` | `True` | `stub boom` |
| `Write-Error "stub boom"` (preference `Stop`) | `False` | `True` | `stub boom` |

And the old form, in a fresh process exactly as `install.ps1:110` had it: exit 1 with
`检索不到变量"$LASTEXITCODE"` pointing at the `if ($LASTEXITCODE -ne 0)` line — the 12:44 failure reproduced in four
lines.

**`install.ps1 -VerifyOnly -TranscriptPath`, through the walkthrough's own `Invoke-ElevatedScript`** (run as the current
user, since `-VerifyOnly` needs no elevation):

| package | exit | transcript | content |
|---|---|---|---|
| `0.0.0-wp4.1+2a8b0bb` | 0 | `verifyonly-0.0.0-wp4.1+2a8b0bb.log`, 1167 B | `101 files verified`, `Package verified: 101 files match the manifest.` |
| `0.0.0-wp4.2+2a8b0bb` | 0 | `verifyonly-0.0.0-wp4.2+2a8b0bb.log`, 1167 B | same |

This proof found a real defect in the fix itself: the helper accepted `-TranscriptPath` for its own bookkeeping but never
added it to the argument list it builds, so the script wrote no transcript at all (`exit 0`, no log). Fixed at `2a8b0bb`
by forwarding it; the table above is the post-fix run.

**A deliberately broken package** (one byte appended to `bin\FusionHost.exe`): exit 1, transcript 1425 B,
`INSTALL FAILED at step 'verify': Package integrity check failed:` followed by `bin/FusionHost.exe (sha256 mismatch)`
and the script position.

**PSScriptAnalyzer** is not installed on the 5700X and was not installed for this run (optional, and installing a module
is a machine change); the audit above is the substitute.

**The machine is unchanged**: not elevated, HKLM `InprocServer32` still the development DLL, HKCU COM key absent, no
task, no `FusionHost`, `%APPDATA%\Fcitx5` the same five files with the same sizes, mtimes (09:59:09) and hashes. The only
residue of the 12:44 run is the version directory it left, recorded in the pre-run snapshot below.

### The 17:07 run — step (a) installed, aborted on the Notepad check (harness defect)

`wp4-run-20261018-170730` is the first attempt in which **the install itself succeeded end to end**:

| observation | evidence |
|---|---|
| `install.ps1` exit 0 | `verify-wp4.log`: `[a] install.ps1 退出码 => 0`; its own transcript `install-0.0.0-wp4.1+2a8b0bb.log` |
| files installed | `C:\Program Files\Fcitx5 Fusion\0.0.0-wp4.1+2a8b0bb\`, 102 files |
| the leftover `0d1a1c4` directory was removed as an older, unused version | `[a] 安装后是否清掉了旧版本目录 => leftover-before=0.0.0-wp4.1+0d1a1c4 still-present=none` |
| registration | `HKLM=C:\Program Files\Fcitx5 Fusion\0.0.0-wp4.1+2a8b0bb\bin\FusionTsf.dll`, `HKCU=(none)` |
| Host | `pid=3528 path=…\0.0.0-wp4.1+2a8b0bb\bin\FusionHost.exe integrity=medium session=1 user=5700X\choicky` |
| task | `FusionHost`, `choicky`, `Interactive`, `Limited`, `IgnoreNew`, `PT0S` |
| then | `[a] 断言：记事本模块 => pid=10648 loaded= …` and `ABORT … detail=Notepad: FusionTsf.dll is not loaded in the process` (exit 8) |
| abort cleanup | Host stopped through its own stop event (`signalled=True exited=True after 124 ms`), development registration restored (`register.ps1` exit 0) |

**Root cause: the process inspected was the launcher, not the window owner.** On this machine `notepad.exe` is the
packaged Windows 11 Notepad (tabs), and `Process.Start('notepad.exe')` returns an app-execution-alias launcher that
hands the window to another process and exits. Measured twice, read-only:

| launch | `Process.Start` PID | alive after 2 s | window owner |
|---|---|---|---|
| probe 1 | 42868 | **no** (dead within 1 s) | pid 7508, `C:\Program Files\WindowsApps\Microsoft.WindowsNotepad_11.2607.14.0_x64__8wekyb3d8bbwe\Notepad\Notepad.exe`, class `Notepad`, title `无标题 - Notepad`, foreground |
| probe 2 | 45008 | yes (the launcher lingers) | pid 7508 again — the packaged app opens a **tab** in the existing process, so no new window is created |

In both cases `Test-NotepadLoadedDll` on the launcher PID reports `FusionTsf.dll is not loaded in the process` — it never
had it. The check could therefore never pass on this machine, whatever the installed TIP did. A profile name or a
description ("Fusion Stage 3B") would not have told us that; the module path and hash do.

**Fix** (`97f90f7`, harness only — no C++/CMake, and neither changed file is inside the package, so the `2a8b0bb`
packages stay valid and were not rebuilt):

1. `Get-NotepadWindows` enumerates visible top-level windows (`EnumWindows`) and keeps the `Notepad` window class,
   mapping each to its owning process (id, image path, title, start time). The title fallback requires a `- Notepad`
   suffix, so another editor called `Notepad--` can never be chosen.
2. `Resolve-NotepadWindow` picks the newest visible window, preferring one created at or after the launch, and waits
   for it, bounded (15 s).
3. `Start-NotepadForCheck` returns **both** PIDs and states which one owns the window, so the log says why a process
   was inspected and a launcher that exits is no longer a failure.
4. `Test-InstalledTipInUse` checks the window owner; the reused-PID retry path rebuilds the same object shape (a
   `Process` has `MainWindowTitle`, not `Title` — reading `.Title` there would have thrown under strict mode).
5. The "not loaded" prompt now gives the owner the order that works: check the taskbar input indicator first, press
   Win+Space **only** if Fusion is not already selected (it cycles), type a few letters in that Notepad, then press
   Enter. Step (d) resolves the pre-upgrade Notepad the same way instead of taking the first `notepad` process.

**Proof without UAC** (`proof-notepad-owner.txt`, `notepad-topology.txt`), with the development DLL registered and the
owner typing in a Notepad window:

| check | result |
|---|---|
| window owner resolution | pid 21668, `…\WindowsApps\Microsoft.WindowsNotepad_11.2607.14.0_x64__8wekyb3d8bbwe\Notepad\Notepad.exe`, title `*你好 - Notepad` |
| modules of that process | `FusionTsf.dll` and `FusionFcitxHost.dll` loaded from `evidence\…-v2\deployment\bin\` |
| pass case (expected = that development DLL) | `ok=True`, loaded path identical, sha256 `03DEDD4CB251D82D4E6B674449067D763E1D6E0D266913EB75856B6F09935B92` |
| fail case (expected = the installed wp4.1 DLL) | `ok=False`, `loaded …evidence\…-v2…, expected …\0.0.0-wp4.1+2a8b0bb\bin\FusionTsf.dll` |
| old behaviour (launcher PID) | `ok=False`, `FusionTsf.dll is not loaded in the process` — the 17:07 failure |

The development DLL is what is registered right now **because the abort cleanup restored it**; that is also why the
owner saw Chinese input but no candidate popup. The installed TIP plus the Host (the state that shows the popup) is
step (a)'s own subject, and the walkthrough run asserted it on the right process (step (a) PASS, see the results).

**Owner observation for R15 (evidence only).** In the 17:07 run, with the *installed* TIP and the Host running, the
owner reports that Chinese input worked **and the candidate popup was visible** in the Notepad window at 175 % display
scaling on a single monitor — something the development DLL never showed. That is evidence against R15's "tiny or
misplaced above 100 %" hypothesis at 175 %/one monitor, but it does not close R15: WP6 still owes 100 % and 150 % and a
two-monitor check, and "visible" is not "correctly positioned".

**Residue of this abort.** Unlike the earlier runs it leaves a working install: the version directory, the logon task
and the development registration all remain (the cleanup stops the Host and restores registration only). The walkthrough
run then did exactly what this note predicted — step (a) installed over the same directory and refreshed the task, and
step (e) uninstalled. Recorded as a known limit below rather than cleaned up by hand.

### Step (d) with the single-process Notepad (same root cause, different consequence)

Every Notepad window on this machine lives in **one** packaged process: a second launch adds a tab, it does not create a
process. Step (d) as first written kept the wp4.1 Notepad open, upgraded, then opened "a fresh Notepad" and asserted the
wp4.2 DLL in it — but that window was in the same process, which still had the wp4.1 DLL loaded, so the assertion could
only fail. That would have aborted the walkthrough after the owner had already done (a)–(c2), including a sign-out.

**New order** (`7fd83bf`):

1. Before the upgrade: resolve and record the window-owning process and its loaded DLL (wp4.1).
2. Upgrade with that window open; fail fast as before.
3. Assert **Host and registration only** (`Assert-HostAndRegistration` → `Test-InstalledTipInUse -SkipNotepad`): a
   `FusionHost` from the wp4.2 `bin` directory at Medium in this session, HKLM at the wp4.2 DLL, no HKCU COM.
4. The still-open window against the new Host: the owner's y/n, that process's loaded DLL again (still wp4.1), whether
   it is still alive, whether the wp4.1 directory exists and whether a process holds it.
5. Close **every** Notepad, wait (bounded) until no window is open and no process holds a Fusion DLL that is not the
   wp4.2 one — then open one and assert the wp4.2 DLL with the manifest SHA-256, with the indicator-first prompt.
6. Notepad A–E in that fresh window.
7. Record whether the wp4.1 directory is free now. It is **not** removed here: step (e)'s uninstall or the next install
   cleans it up.

**The same assumption elsewhere**, checked step by step:

| step | assumption | verdict |
|---|---|---|
| a | a new launch gives a new process | **was false** — a leftover window holding the development DLL made the check fail for the wrong reason. Fixed: `Confirm-NoNotepad` before the check, which waits until no Notepad window is open and no process holds a Fusion DLL other than the expected one |
| b | none (starts the task and the Host directly) | OK |
| c | none (asks the owner to type in an existing window) | OK |
| c (resume) | a new launch gives a new process | **was false** — same fix as (a) |
| c2 | the process it started owns the window | **was false** — the recorded PID was the launcher. Fixed: `Start-UnsavedNotepad` resolves the window owner |
| d | a new launch gives a new process, and the old window releases the DLL | **was false** — the new order above |
| e | an open Notepad releases the DLL so the directory can be removed | **not guaranteed** — it now asks for Notepads to be closed, records whether the wp4.2 directory is still held before uninstalling, and treats a held directory as "kept" (non-fatal, never forced) |
| f | same as (e), plus "a loaded DLL can be overwritten" | **was false** — `Confirm-NoNotepad -RequireNoFusionDll` before the reinstall, because a copy over a mapped DLL would fail the install |

**One more measured detail**: `WM_CLOSE` (and `CloseMainWindow`) closes a single-tab Notepad and the process exits, but
with two tabs the window stays open — the packaged app wants the tabs closed or confirmed. The prompt therefore says to
close each tab (Ctrl+W) or confirm "close all tabs".

**Proof without UAC** (`proof-step-d.txt`), development DLL registered:

| fact | measurement |
|---|---|
| a second launch while a window is open | first window pid 44828; `Process.Start` returned 45796; the resolved window owner was **44828** — same process, launcher not the owner |
| the helper does not call that state clear | `Wait-NotepadStateClear` → `Clear=False`, 1 window |
| after the window closes | `Clear=True`, 0 Notepad processes, the old PID gone |
| a launch after that | resolved pid 29584 ≠ 45208 — a genuinely new process |
| machine afterwards | development DLL still registered, `%APPDATA%\Fcitx5` unchanged, 0 Notepad processes left |

The proof closes its own empty Notepad with `WM_CLOSE` (what the owner does) and terminates the empty Notepad it started
between the two facts to reset; the walkthrough itself never kills anything.

**Packages**: no packaged script changed (`install.ps1`, `uninstall.ps1`, `host-state.ps1`, `tools\*.ps1` are untouched
since `2a8b0bb`), so the packages were not rebuilt and their script hashes still match the repository.

### Step (c2): a sign-out blocker the harness owns

Step (c2) needs a sign-out that can be cancelled. The first version relied on Notepad asking to save an unsaved
document — but the packaged Windows 11 Notepad **restores unsaved content and does not prompt on sign-out**, so nothing
would have held the session and the owner would have been signed out for real, after (a)–(c) and before (d).

The blocker is now a harness object: a visible top-level window on its own thread (`FusionShutdownBlocker`) that calls
`ShutdownBlockReasonCreate("WP4 test: click Cancel")` and answers **FALSE** to `WM_QUERYENDSESSION`, so Windows shows its
"these apps are preventing you from signing out" screen. The step:

1. starts the blocker and requires `alive`, a non-empty block reason read back with `ShutdownBlockReasonQuery`, and
   FALSE from a synthetic `WM_QUERYENDSESSION` — otherwise it stops before asking the owner to do anything;
2. saves the state as `c2-signout` **before** the request, so a sign-out that is not held leaves a resumable run;
3. tells the owner exactly what to expect and to click **Cancel** (not "sign out anyway");
4. after the cancel: same Host PID, still from the wp4.1 `bin` directory, typing works (y/n), blocker closed;
5. if the owner *was* signed out, `-Resume` records `sign-out was not blocked (blocker ineffective)` — neither a pass
   nor a product failure — and continues to (d).

**Proof without UAC** (`proof-a1-a2.txt`): the window is visible with the expected title and class, the reason reads
back as `WP4 test: click Cancel`, `WM_QUERYENDSESSION` returns FALSE (the window procedure saw the query), and `Stop`
closes it. No sign-out was performed.

**What happened in the run.** The blocker verified and Windows did show the blocking screen; the owner clicked Cancel
and Windows recorded it (User32 1077 at 19:03:20), so the logoff **was** cancelled. The walkthrough's terminal still
closed with the logoff attempt, and the Host had already saved and ended — which is R34, not a harness failure.
*Corrected 2026-10-18 by the event-log evidence: this section used to say the blocker died with its terminal so "Cancel
had nothing left to cancel". The cancel worked; the Host did not survive it.*

### Uninstall and the per-user input state (A2)

Snapshots showed `HKCU\Software\Microsoft\CTF\TIP\{CLSID}` present after installs while `uninstall.ps1` removed only
the HKLM keys. Established read-only first (`a2-hkcu-tip.txt`):

| question | answer |
|---|---|
| what does the HKCU key hold? | only `LanguageProfile\0x00000804\{profile}` with `Enable = 1`; no values on the parent keys, no `Description`, **no COM `InprocServer32`** (so it is not an override) |
| who writes it? | `EnableLanguageProfile` (called by `src/tsf/register.cpp`), i.e. per-user profile enablement — not registration and not the language list |
| is the TIP in the user's language list? | **no**: `Get-WinUserLanguageList` → zh-Hans-CN lists `{86598FB9…}{607FDF85…}` and `{81D4E9C9…}{FA550B04…}` only |

So an uninstall that removed only HKLM would leave an **enable flag for a profile whose DLL no longer exists** — an entry
the user can still see and select in the input indicator. The uninstaller therefore now removes, for the current user
only:

- `HKCU\SOFTWARE\Microsoft\CTF\TIP\{CLSID}` (recorded with the `Enable` value it held), and
- the language-list tip `0804:{CLSID}{profile}`, **only if this user actually has it** — `InputMethodTips` is a get-only
  `StringCollection`, so it is edited rather than assigned, and a failure there warns instead of failing the uninstall.

`Get-FusionUserInputState` records the state and `Remove-FusionUserInputState` performs the removal (both in
`host-state.ps1`, so the packaged uninstaller and the proof exercise the same code). Steps (e) and (f) record HKCU
`CTF\TIP` and the language-list tips before and after, and the end-of-run restore records them again.

**Proof without UAC** (`proof-a1-a2.txt`): against a synthetic `HKCU\…\CTF\TIP\{FFFFFFFF-…}` key of the same shape,
the helper reports `EnableWas=1`, removes the key, leaves the real key and the language list untouched, and emits no
warning; the language-list filter is exercised on a `StringCollection` of the same type (one matching entry removed, the
unrelated one kept). Machine state before and after is identical.

## Walkthrough harness rules (carry forward)

Seven rules, each paid for by a failed attempt on 2026-10-18. They apply to every later owner walkthrough (WP5, WP6,
WP8), and §8.1 of the plan points here.

| rule | why it exists (what failed without it) | where it lives |
|---|---|---|
| **Prove the elevated span, or make every failure name its step and line.** The walkthrough cannot test an elevated script's internals; the script must therefore write its own transcript, set `$currentStep` before each phase and print `INSTALL FAILED at step '<x>': <message>` with its position | the 12:44 run aborted on a strict-mode read inside the elevated span and reported only an exit code | `install.ps1` / `uninstall.ps1` `-TranscriptPath` + `$currentStep`; the wrapper never starts a second transcript |
| **Assert Host, registration and the loaded module before any typing question** — the module by path **and** hash, in the process that owns the target's window | the 11:55 run asked typing questions while the *development* DLL was registered; a profile name identified nothing | `Assert-InstalledTipInUse` / `Assert-HostAndRegistration` |
| **Fail fast**: any non-zero step stops the run, records the step/command/exit code, runs the safe cleanup and asks nothing further | the 11:55 run continued to typing questions after `install.ps1` exited 1 | `Assert-StepExit` / `Stop-Walkthrough` / `Invoke-SafeRestore` |
| **The owner never types commands or edits files**; prompts accept Enter / y / n or the one short answer they ask for, and every reported observation is captured | earlier harnesses relied on the owner transcribing results, and one "y" was recorded for a step that never ran | `Ask-Enter` / `Ask-YesNo` / `Ask-Text`, `Add-Result` |
| **Account for packaged-application process topology**: a launched executable may be a launcher, several windows may share one process, and an open window keeps its DLL — resolve the window owner, and close every window of the app before asserting a fresh load | the 17:07 run checked the launcher PID; step (d) would have asserted the new DLL inside the process still holding the old one | `Get-NotepadWindows` / `Resolve-NotepadWindow` / `Wait-NotepadStateClear` / `Confirm-NoNotepad` (R31) |
| **Save state before any sign-out request**, with a resume stage that can record "the session ended anyway" as neither pass nor failure | (c2) asks for a real sign-out; if it is not held, the run must survive it | `$state.stage = 'c2-signout'` before the prompt; the `c2-signout` branch records "blocker ineffective" |
| **Prefer a blocker the harness owns over application behaviour**: a visible top-level window with a registered shutdown block reason that answers FALSE to `WM_QUERYENDSESSION` | the packaged Notepad restores unsaved text and never prompts, so it could not hold the sign-out — the owner would have been signed out for real | `FusionShutdownBlocker` / `Start-ShutdownBlocker` / `Test-ShutdownBlockerBlocksSessionEnd` |

## Known limits at this scope

- **A cancelled logoff, restart or shutdown ends the Host (R34; RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement).** At 19:03:20 on 2026-10-18 Windows logged User32 1077 "user 5700X\choicky attempted to log off computer 5700X and failed" — the Cancel worked — and the Host had already saved (19:03:13) and ended; no Host ran afterwards. Windows delivers `WM_ENDSESSION(TRUE)` to applications one after another, so an application processed before the blocking one ends before the Cancel takes effect (inference, consistent with all evidence; not a proven cause). The fix is a 1077/1073 event trigger that starts the Host again — the handler itself is right, the survival assumption was wrong — and the 2026-10-19 run verified it for a cancelled logoff (c2) and a cancelled restart (c3).
- **During an upgrade there is a gap** between the old Host's orderly stop and the new Host's start: the TIP is still
  registered and keys pass to the application unmediated (no candidate, no composition). It is bounded by the install
  (seconds) and it is the same "Host absent" state WP5 §8.2 exercises deliberately; the walkthrough records it only
  implicitly (step (d) types in the old window *after* the upgrade, not during the gap).
- **Multiple accounts on one machine**: the TIP is machine-wide but the task name `FusionHost` is global, so a second
  account's install overwrites the first account's task, and an account without its own task types with no Host (fail
  open). Hypothesis from source, not exercised (one interactive account here); CONDITIONAL, candidate fix is a per-user
  task name (R32).
- **The package is visibly a development build**: the TIP description is "Fusion Stage 3B (Real Fcitx5 Backend PoC)"
  and the CLSID/profile are the development GUIDs, so they appear in the input indicator and the language settings.
  Correct for an owner-only test package; the product name and permanent identifiers are decided before any package
  leaves the owner (R29, WP8/U4), and changing them later needs the development registration removed first.
- An old version directory stays locked while **any** process still holds its `FusionTsf.dll`, and the holders are not
  only the application that typed: the input method is global, so `FusionTsf.dll` stays loaded in shell processes
  (`explorer.exe`, `WindowsTerminal.exe`, …) until sign-out. The installer and the uninstaller report every holder; the
  directory is removed by running the uninstaller again after sign-out or a restart. Never force-deleted. **Future
  protocol changes**: an upgrade that changed `protocol_version` would leave those processes on the old DLL and unable
  to use the new Host until sign-out (fail open), so such a change needs a sign-out requirement or a compatibility
  window (R28). The walkthrough left exactly that: both version directories remained (holders `firefox`, Notepad,
  `explorer`), and `C:\Program Files\Fcitx5 Fusion\` is to be removed after a restart (manually, because step (f)
  already removed the task and the registration).
- An install that fails after the copy leaves its version directory behind (the 12:44 run did). It is harmless — a later
  install merges over it and uninstall removes it — but "install root absent" is then no longer true before the next run;
  the walkthrough records the directories it finds.
- An aborted walkthrough does not uninstall: the cleanup stops the Host and restores the development registration only,
  so a version directory and the logon task stay until the next run's step (a) or step (e). The 17:07 abort left exactly
  that (installed `0.0.0-wp4.1+2a8b0bb`, task `FusionHost` last result 0, no Host, development DLL registered).
- Nothing restarts the Host after a crash, and an alive-but-unhealthy Host is not restarted (WP5/U7).
- The task, pipe, claim and stop event are single user × session, and a machine with several interactive users is
  **not** supported today: the task name `FusionHost` is global, so a second account's install overwrites the first
  account's task (R32, hypothesis from source, CONDITIONAL). An installer elevated as a *different* administrator cannot
  open the owner's stop event — it reports that instead of killing the Host.
- The windowless Host writes no log file **unless the opt-in is set**: with `FUSION_HOST_DIAGNOSTICS=1` it writes
  `%LOCALAPPDATA%\fusion-host\fusion-host-<pid>.log` (technical only, 2 MiB cap, five files retained); without it, stderr
  evidence still exists only when a caller redirects it. Recorded as **R30**, **IMPLEMENTED** in fusion-windows PR #19
  (WP5's entry condition); Fcitx's own log is excluded by owner decision 2026-10-19.
- `--console` is required by anything that still expects the console subsystem (the WP3 harness pattern); a caller that
  passes no switch gets the windowless Host.
- No MSI, no signing, no distribution channel (U4, WP8); versions are provisional test identifiers.

## Evidence locations

All under `C:\Users\choicky\worktmp\fusion-windows\stage6-wp4\` unless noted:

| what | where |
|---|---|
| **owner walkthrough, 2026-10-18** | run `wp4-run-20261018-185004` — its directory (`verify-wp4.log`, `snapshots\*`, `install-*.log`, `uninstall-*.log`) is **kept locally by the owner**; the values quoted above come from that log |
| A1 console mode / CTRL_BREAK | `a1-console.txt`, `a1-console-host-stderr.log` |
| A3 runtime location and DLL dependencies | `a3-smoke.txt`, `a3-smoke-host-stderr.log` |
| A5 / R23 probe | `a5-probe.txt` |
| B1 build and test logs | `build-mock.log`, `build-debug.log`, `build-release.log`; native runtime logs `native-debug.log`, `native-release.log` |
| B2 packages and manifests | `packages\fusion-0.0.0-wp4.{1,2}+2a8b0bb.zip`, `b2-manifests.txt` (UTF-8 copy `b2-manifests-utf8.txt`), `package-wp4.{1,2}.log` |
| B3 dry run (post-fix) | `wp4-run-20261018-183630\verify-wp4.log`, `wp4-run-20261018-183630\snapshots\01-before.txt` |
| B3 aborted run 11:55 (invalid) | `wp4-run-20261018-115501\verify-wp4.log`, `install-wp4.1.log`, `elevated-install-wp4.1.ps1`, `snapshots\02-after-install-wp4.1.txt` |
| B3 aborted run 12:44 (invalid) | `wp4-run-20261018-124402\verify-wp4.log`, `install-wp4.1.log` (the `$LASTEXITCODE` failure), `snapshots\*` |
| B3 aborted run 17:07 (install OK, Notepad check invalid) | `wp4-run-20261018-170730\verify-wp4.log`, `install-0.0.0-wp4.1+2a8b0bb.log`, `snapshots\*` |
| Notepad topology probe | `notepad-topology.txt`, `probe-notepad-topology.ps1` |
| Notepad resolution proof (pass/fail, on the owner's window) | `proof-notepad-owner.txt`, `check-owner-notepad.ps1` |
| single-process / close-and-relaunch proof | `proof-step-d.txt`, `proof-step-d.ps1` |
| A1/A2 proof (blocker, per-user input state) | `proof-a1-a2.txt`, `proof-a1-a2.ps1` |
| A2 investigation (HKCU CTF\TIP, language list) | `a2-hkcu-tip.txt`, `probe-a2.ps1` |
| hash-skip copy proof | `proof-copy.txt`, `proof-copy.ps1` |
| R34 event-log evidence (owner-collected, elevated) | `r34-evidence.txt` — kept locally by the owner; the times quoted above come from it |
| 2026-10-19 walkthrough run | `wp4-run-20261019-130037` (under `worktmp\fusion-windows\stage6-wp4\`) — kept locally by the owner; the values quoted above come from its `verify-wp4.log` |
| R34 cancel-event measurement (2026-10-19) | `b1b-restart.txt`, `b1b-shutdown.txt`, `b1b-shutdown.txt.after.txt` — kept locally by the owner; the RecordIds and times in the table above come from them |
| R34 WM_CLOSE proof and negative control | `proof-wmclose.txt` / `proof-wmclose.ps1`, `proof-wmclose-old.txt` / `proof-wmclose-old.ps1` (under `worktmp\fusion-windows\stage6-r34\`) |
| R34 event-query proof (the five XPath queries) | `proof-a1.txt` / `proof-a1.ps1` (same directory) |
| R34 event-query validation | `proof-xpath-validation.txt` / `proof-xpath-validation.ps1` (same directory) |
| R34 blocker-restart and session-identity proofs | `proof-a2-a4.txt` / `proof-a2-a4.ps1`, `proof-wts.txt` / `proof-wts.ps1` (the WTSINFOW layout), plus `probe-mode1.txt` and `probe-collectonly.txt` for the probe's two modes (same directory) |
| harness proof without UAC (round 2) | `harness-proof.txt` |
| script-call semantics probe | `probe-semantics.ps1` (its output is quoted in the 12:44 section) |
| script-call / VerifyOnly / broken-package proof (round 3) | `proof-scripts.txt` (UTF-8 copy `proof-scripts-utf8.txt`), `proof-scripts\verifyonly-0.0.0-wp4.{1,2}+2a8b0bb.log`, `proof-scripts\verifyonly-broken.log` |
| runtime product | `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018\` (Debug + Release) |
