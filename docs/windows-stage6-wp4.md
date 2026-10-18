# Windows Stage 6 WP4 — packaging and lifecycle (R6, R9, R23, R26; U2)

Date: 2026-10-18. Owner decision recorded: **U2 = (b) Accepted 2026-10-18** (§7.1 of the plan).

**Result: WP4 is implemented and locally verified; the owner walkthrough (B3/B4) is pending.** B1, B2 and the
walkthrough dry run are complete on the branch head below. The lifecycle evidence that requires an interactive owner
(install, sign-out/sign-in, upgrade with a running application, uninstall) is not claimed here. Three owner attempts
are recorded below: `wp4-run-20261018-115501` (the walkthrough quoted a parameter *name*), `wp4-run-20261018-124402`
(install.ps1 read `$LASTEXITCODE` after a `.ps1` call and aborted a *successful* registration) and
`wp4-run-20261018-170730` (**step (a) installed and asserted correctly for the first time**, then aborted on a Notepad
check that inspected the launcher process instead of the window's). All three are harness defects, all three are fixed,
and each fix is proven without UAC; the rerun is pending.

## Candidate, runtime and tooling identities

| item | value |
|---|---|
| fusion-windows base | `572fdb9` (PR #16 merge, `main`) |
| branch / head | `feature/stage6-wp4-packaging` — `7cba293` (implementation), `faeb266` (package file selection), `b050292` (walkthrough launcher), `8b0d9ec` (installer verification mode), `e6f131e` (cancelled-logoff fix, session-end test, machine identity in the manifest, walkthrough steps c/c2), `0d1a1c4` (ignore the local agent workspace), `53597d7` (harness fix after the 11:55 abort), `f63aa4f` (skip the Notepad check when an earlier assertion failed), `ae54e70` (elevated-path fix + full script audit), `2a8b0bb` (forward `-TranscriptPath` to the launched script), `97f90f7` (check the Notepad window's process, not the launcher's PID) |
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
   expects the Pinyin default with no WP3 learning to reuse.
3. **`reg.exe` is blocked on the 5700X** by its Security Center program blacklist, so `VsDevCmd.bat` cannot detect the
   Windows SDK and produces `LIB`/`INCLUDE` without SDK paths (link fails with `LNK1104: kernel32.lib`). The build was
   driven by a local wrapper outside the repository that supplies the SDK `INCLUDE`/`LIB`/`PATH` and then calls the
   repository script unmodified. This is a machine condition, not a repository change.
4. **clang-format available on the 5700X is 23.1.3** (WP3 used 22.1.3). It reports formatting differences in files this
   WP did not touch (`src/tsf/service.hpp`, `src/tsf/diag.cpp`), so the check was scoped to the files this WP changed.

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
stop and waits. The walkthrough's step (c2) checks the product behaviour by cancelling a real logoff.

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
unregister and remove the `CTF\TIP` / COM keys that the COM unregister leaves → delete the task → remove version
directories no process holds → remove the install root if empty. `%APPDATA%\Fcitx5` is kept unless `-RemoveUserData` is
given, which removes only the current user's `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5`.

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
against a 300 ms bound); with the exit already signalled the same call returns immediately. A real session end is still
only exercised by the walkthrough's step (c) and step (c2).

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

Both packages were rebuilt from the branch head `2a8b0bb` with `tools/package/build-package.ps1`; every hash below is
taken from `b2-manifests.txt` (written from the zips' own `package.json`), never transcribed from a console. The
manifest records `builtOn: 5700X`, `runtime.machine: 5700X` and the note that the runtime is a rebuild from the WP3
pins.

| item | wp4.1 | wp4.2 |
|---|---|---|
| file | `fusion-0.0.0-wp4.1+2a8b0bb.zip` | `fusion-0.0.0-wp4.2+2a8b0bb.zip` |
| zip SHA-256 | `0CBF47ADD4D7EC722F94E2A4C3A0429A243F5DCDDD5F5846FC6502E5A9004624` | `01EECAF1BDD9323F903F8EC39148AD2EE8EB976D91F8529D823B8E994D32B0D8` |
| size | 39 461 225 B | 39 461 225 B |
| files / uncompressed | 101 / 53 119 874 B | 101 / 53 119 874 B |
| source commit | `2a8b0bbbc3f1e7e733e9440408a79c3ea735f8a7` (clean) | same |
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
| `install.ps1` | `D9146ED22914A721C518FCC692D18894595A989F4763DEB5A115F6F8888B9033` |
| `uninstall.ps1` | `AAE7749439F3DAE00F93006CA97FD89F4AD903406C2BB2E14F612E76D023A14F` |
| `host-state.ps1` | `93DB1C3D9CBBA98688C5ED218252F82C40668311C35091EFD59DF50680728362` |
| `tools/register.ps1` | `7534A2CC10F4DA0818D5F030D2E91D001BA6E6B345EFB11D4EEB8547978891C5` |
| `tools/unregister.ps1` | `6F4FB97581E0C2E3D30F3E0A92953F568C98FCDB146087D0119F6F1A8DF47922` |
| `tools/check-state.ps1` | `F019F0AA447FA2E080C77F529663FC666A00030341279E7712F58E3FC4CF728B` |

## B3/B4 — owner walkthrough (NOT RUN)

`tools/package/verify-wp4.ps1` performs every action itself; the owner never edits a file or types a command. Prompts
accept Enter / y / n, or the one short answer a prompt asks for (a pinyin string, a rank such as `5→1`), and every
answer is written to the run log — a reported observation is never reduced to "Enter". Before and after each step it
records the install root, the HKLM/HKCU `CTF\TIP` and COM keys, the scheduled task, running `FusionHost` processes
(path, user, integrity, session), `%APPDATA%\Fcitx5` (count/size and size+mtime+hash of the key files) and the machine
and user `PATH`. It records the development registration found before the run and restores it at the end.

| step | what the owner does | what the script records |
|---|---|---|
| a | allow UAC, close every Notepad, check the taskbar input indicator (Win+Space only if Fusion is not already selected), type a few letters in the Notepad the script opened, press Enter | install exit code (fatal if non-zero), Host pid/session/user/integrity, HKLM/HKCU registration, which Notepad process was checked and why (window owner vs launcher), its loaded DLL + hash, `你好` yes/no, "Pinyin default, no WP3 learning" yes/no |
| b | nothing (the script starts the task again and `FusionHost.exe` directly) | Host count after each attempt, the direct start's exit code and stderr, typing still works |
| c | type a pinyin string, select a candidate, report the rank change; then sign out | Host running before the learning prompt, the pinyin string, `before→after` ranks, `%APPDATA%\Fcitx5` snapshot; then the exact `-Resume` command |
| c (resume) | sign in, close every Notepad, type the same string in the Notepad the script opened, report the rank | the same assertions as step a, typing works, the current rank, whether the learning survived |
| c2 | leave an unsaved Notepad, start a logoff, **cancel** it | Host pid before and after, "still the same Host", typing still works |
| d | keep a Notepad open, allow UAC for the upgrade, then check both Notepads and Notepad A–E | the old Notepad's loaded DLL before and after, the same assertions as step a against the wp4.2 directory, whether the wp4.1 directory was removed, behaviour of the old and the new Notepad, A–E answers |
| e | allow UAC for the uninstall | exit code, no TIP/COM/task/Host, install root, `%APPDATA%\Fcitx5` kept, `check-state.ps1` output |
| f | close every Notepad, allow UAC twice | reinstall and `-RemoveUserData` exit codes, user data removed, nothing else left |
| end | — | the development registration restored and re-snapshotted |

The learning used by steps (c)/(d) is produced inside the walkthrough with the Host already running (step (c) refuses
to continue if no Host is running), so the owner is never asked to do something the machine cannot do at that moment.

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
are the package binaries. Step (a) records the version directories before and after the install, so the run will show
whether `install.ps1` removed that directory as an older, unused version; nothing holds its DLL, so it should be
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
step (a)'s own subject, and the next run asserts it on the right process.

**Owner observation for R15 (evidence only).** In the 17:07 run, with the *installed* TIP and the Host running, the
owner reports that Chinese input worked **and the candidate popup was visible** in the Notepad window at 175 % display
scaling on a single monitor — something the development DLL never showed. That is evidence against R15's "tiny or
misplaced above 100 %" hypothesis at 175 %/one monitor, but it does not close R15: WP6 still owes 100 % and 150 % and a
two-monitor check, and "visible" is not "correctly positioned".

**Residue of this abort.** Unlike the earlier runs it leaves a working install: the version directory, the logon task
and the development registration all remain (the cleanup stops the Host and restores registration only). The next run's
step (a) installs over the same version directory and refreshes the task, and step (e) uninstalls. Recorded as a known
limit below rather than cleaned up by hand.

**Pending.** B3a–B3f, B4 (Notepad A–E on the installed wp4.2) and the per-step results are not recorded here. Until the
owner runs the script, WP4 is not closed: the exit gate needs that evidence.

## Known limits at this scope

- An old version directory stays locked until every application that loaded its DLL exits; a later install or uninstall
  removes it. It is reported, never force-deleted.
- An install that fails after the copy leaves its version directory behind (the 12:44 run did). It is harmless — a later
  install merges over it and uninstall removes it — but "install root absent" is then no longer true before the next run;
  the walkthrough records the directories it finds.
- An aborted walkthrough does not uninstall: the cleanup stops the Host and restores the development registration only,
  so a version directory and the logon task stay until the next run's step (a) or step (e). The 17:07 abort left exactly
  that (installed `0.0.0-wp4.1+2a8b0bb`, task `FusionHost` last result 0, no Host, development DLL registered).
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
| B2 packages and manifests | `packages\fusion-0.0.0-wp4.{1,2}+2a8b0bb.zip`, `b2-manifests.txt` (UTF-8 copy `b2-manifests-utf8.txt`), `package-wp4.{1,2}.log` |
| B3 dry run (post-fix) | `wp4-run-20261018-172101\verify-wp4.log`, `wp4-run-20261018-172101\snapshots\01-before.txt` |
| B3 aborted run 11:55 (invalid) | `wp4-run-20261018-115501\verify-wp4.log`, `install-wp4.1.log`, `elevated-install-wp4.1.ps1`, `snapshots\02-after-install-wp4.1.txt` |
| B3 aborted run 12:44 (invalid) | `wp4-run-20261018-124402\verify-wp4.log`, `install-wp4.1.log` (the `$LASTEXITCODE` failure), `snapshots\*` |
| B3 aborted run 17:07 (install OK, Notepad check invalid) | `wp4-run-20261018-170730\verify-wp4.log`, `install-0.0.0-wp4.1+2a8b0bb.log`, `snapshots\*` |
| Notepad topology probe | `notepad-topology.txt`, `probe-notepad-topology.ps1` |
| Notepad resolution proof (pass/fail, on the owner's window) | `proof-notepad-owner.txt`, `check-owner-notepad.ps1` |
| harness proof without UAC (round 2) | `harness-proof.txt` |
| script-call semantics probe | `probe-semantics.ps1` (its output is quoted in the 12:44 section) |
| script-call / VerifyOnly / broken-package proof (round 3) | `proof-scripts.txt` (UTF-8 copy `proof-scripts-utf8.txt`), `proof-scripts\verifyonly-0.0.0-wp4.{1,2}+2a8b0bb.log`, `proof-scripts\verifyonly-broken.log` |
| runtime product | `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018\` (Debug + Release) |
