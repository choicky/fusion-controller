# Windows daily use (WP7) — the one entry point

This is the owner-facing guide for the Stage 6 WP7 daily-use period. The durable decisions are in
`docs/DECISIONS.md` (D084) and `docs/windows-architecture.md` (§10, §14); the acceptance policy is in
`docs/windows-stage6-plan.md` §8.1. **Daily use is not complete until the owner closes the period** (see §6).

## 1. Install, upgrade, uninstall

The package is owner-only and unsigned. Everything except the last step is elevated.

| step | command |
|---|---|
| install | extract the zip, then run `install.ps1` **elevated** (`-PackageDir <extracted folder>`, default is the script's own folder) |
| upgrade | extract the new zip and run its `install.ps1` elevated; it closes the running tray, stops the running Host in order, copies the new version directory, re-registers the TIP, refreshes the logon tasks and starts the new tray |
| uninstall (keep user data) | run `uninstall.ps1` **elevated** from the installed version directory |
| uninstall (remove user data) | `uninstall.ps1 -RemoveUserData` — also deletes `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5` |

`install.ps1 -VerifyOnly` checks the manifest and hashes without changing anything. The installer never modifies `PATH`,
never force-deletes a directory a process holds, and never touches `%APPDATA%\Fcitx5` unless `-RemoveUserData` is given.
A version directory held by a running application stays until the next uninstall after sign-out (R28); the installer
reports the holders.

**Rollback:** run the previous version's `install.ps1` elevated (it installs side by side and re-registers that version),
or uninstall and install the earlier package. User data is kept either way.

**Installing registers and enables Fusion; it does not necessarily make it the selected input method.** After install (and
after every upgrade), check the Windows input indicator in the taskbar. If Fusion is not the current input method, press
**Win+Space** until it is, then type. Judge U6 only with Fusion already selected — otherwise "letters appear raw" means the
IME was not selected, not that a new window failed to open.

## 2. Restarting the Host

Start menu → **Fusion → Restart FusionHost** (`restart-host.ps1` in the installed version directory), or the tray icon's
**Restart FusionHost** command (§4, "The Fusion tray") — both run the same tool. It:

1. asks the running Host for an **orderly** stop through the product's own stop event and waits up to 20 s;
2. if it is still alive, force-stops **only** the instance verified as this user's and this session's, and says so — that
   instance does not run Fcitx's own save;
3. starts the installed Host through the existing logon task (Medium integrity);
4. verifies the executable path, the integrity level and that exactly one Host is running.

An absent Host is a normal restart. Another user's or another session's Host is reported and never touched. Nothing here
enables automatic restart or changes the task's restart-on-failure setting.

## 3. Selecting the backup input method

Press **Win+Space** to cycle input methods; the indicator shows the current one. Use it if Fusion is not selected, or to
compare against another IME. Installing does not force Fusion to be the selected method; the Windows indicator is the
authority. The Fusion tray icon (§4, "The Fusion tray") reports whether this user's Host is running; it is not an
input-method selector — the Windows indicator remains the authority for which method is current.

## 4. Diagnostics and the tray

### Diagnostics (off by default)

The product default stays **off**. To enable the Host's technical log for this user:

```
powershell -ExecutionPolicy Bypass -File "<version dir>\host-diagnostics.ps1" -Enable
```

It sets the user environment variable `FUSION_HOST_DIAGNOSTICS=1`, restarts the Host and **verifies that the launch path
received it** (the new pid's log file must exist; a launcher with a cached environment is reported). Disable with
`-Disable`; show the state with `-Status`.

- Log: `%LOCALAPPDATA%\fusion-host\fusion-host-<pid>.log`; five files retained, 2 MiB cap.
- Content is technical only: no key codes, pinyin, preedit, candidate, document or password text.
- The TSF log (`%LOCALAPPDATA%\fusion-tsf\`) stays a separate developer opt-in (`FUSION_TSF_DIAGNOSTICS=1`).
- Fcitx's own log is **not** part of this; at debug level it can contain key events.

### R25 browser capture (optional troubleshooting, one-off per process)

R25 was **reported recovered by the owner on 2026-10-20** (see §6), so this capture is no longer a required step. Keep it
for a browser field that shows no candidate list: it checks that field against the **installed** package. It uses the
existing `FUSION_TSF_DIAGNOSTICS` opt-in for one process, so nothing is added to the user or machine environment.

1. **Save your work and exit every process of the browser you will test** — all windows, plus any background
   instance. A running browser keeps the `FusionTsf.dll` it loaded at startup, and a new launch would only hand the
   window to it.
2. **Start the browser from a Command Prompt**, so the variable lives in that window only. Run the two lines for your
   browser:

   ```
   set FUSION_TSF_DIAGNOSTICS=1
   "C:\Program Files\Mozilla Firefox\firefox.exe"
   ```

   ```
   set FUSION_TSF_DIAGNOSTICS=1
   "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
   ```

   Use your normal profile. Do not use `setx` or the System/User environment dialog: the capture needs the variable in
   one process tree only, and a permanent value would silently cover every later process. The prompt may return
   immediately even when a new browser process starts correctly, so prompt timing says nothing about the launch —
   step 3 establishes which process is under test.
3. **Establish that this is a fresh process running the installed build.** Focus a text field in the browser and make
   sure Fusion is the selected input method (Win+Space) first: the DLL is loaded when the TIP is activated in that
   process, so a window with no focused field can legitimately show `-`. Then check, read-only:

   ```powershell
   Get-Process firefox,msedge -ErrorAction SilentlyContinue | ForEach-Object {
     $dll = try { ($_.Modules | Where-Object ModuleName -eq 'FusionTsf.dll').FileName } catch { '<no access>' }
     if (-not $dll) { $dll = '-' }
     '{0}  {1:HH:mm:ss}  {2}' -f $_.Id, $_.StartTime, $dll
   }
   ```

   - **Identity:** the pid and start time of the window-owning process must be newer than the launch. Only that
     process maps the TIP; its content/renderer children do not.
   - **Loaded build:** compare the path with the **intended installed version** — currently
     `C:\Program Files\Fcitx5 Fusion\0.0.0-wp7+995e5a5\bin\FusionTsf.dll`, and after any later upgrade whatever
     version directory `HKLM\SOFTWARE\Classes\CLSID\{8B7C4A90-5E2D-4B1F-9A31-6C0F42D1E7A4}\InprocServer32` names.
     A *different* version directory means that process still holds an older build: exit it and repeat from step 1.
     `-` only means no `FusionTsf.dll` was observed in that process — it does not establish an old build or a product
     failure. `<no access>` is inconclusive. Re-focus the field and check again before concluding anything.
   - **Fresh diagnostic records for that pid:** `%LOCALAPPDATA%\fusion-tsf\fusion-tsf-<pid>.log` must have a last-write
     time after the launch, and its lines carry `pid=<pid>`. A file of that name that already existed is an older
     session's log and is not evidence for this one:

     ```powershell
     Get-Item "$env:LOCALAPPDATA\fusion-tsf\fusion-tsf-<pid>.log" -ErrorAction SilentlyContinue |
       Select-Object LastWriteTime, Length
     ```

4. **Firefox, two surfaces.** Address bar: type `nihao`, press `Esc` (this is the control). Then open ChatGPT, click
   its editor, type `nihao`, press `Esc`. Do not send.
5. **Edge, two surfaces.** Address bar: type `nihao`, press `Esc`. Then Gmail → **Compose** (empty draft), type
   `nihao` in the body, press `Esc`, and close the draft without saving. Do not send.
6. **Report per surface** whether the candidate list was visible and from which keystroke, plus the pid and the
   version directory. Then close the browser processes and the Command Prompt window from step 2 (see Cleanup).

`nihao` is the only thing to type — nothing private, no existing mail or conversation, nothing sent, and no screenshot
of a mailbox or a chat.

**Logs:** `%LOCALAPPDATA%\fusion-tsf\fusion-tsf-<pid>.log`, one per process. A log written after the launch, with
lines carrying that pid, is the proof that step 2's variable reached this browser process; a file that already existed
is not. Content is technical only — stage names, HRESULTs, rectangles, window state; no typed text, candidate text,
URLs or page content. The lines that decide the next step:

| line | meaning |
|---|---|
| `ANCHOR_PROBE stage=ok source=composition` | the composition range had layout and supplied the anchor |
| `ANCHOR_PROBE stage=ok source=composition-caret` | the composition range had no layout; the caret supplied the anchor (PR #25) |
| `ANCHOR_PROBE stage=text-ext source=composition hr=0x80040206` | `TS_E_NOLAYOUT` — no range had layout for that keystroke, so the page stays hidden |
| `ANCHOR_PROBE stage=no-active-view` / `stage=no-range` | the context had no active view, or the selection returned no range — the anchor was not even attempted |
| `CANDIDATE_VIEW hidden reason=no-anchor` / `no-page` | nothing to anchor to / no candidate page to show |
| `stage=ok` and still no visible window | the anchor is fine and the defect is in placement or visibility |

**Cleanup:** exit every process of the diagnostic browser (including a background instance it leaves behind) **and**
close the Command Prompt window that set the variable. Exiting the browser does **not** clear `FUSION_TSF_DIAGNOSTICS`
in its parent Command Prompt, so that window would keep the variable for anything else started from it. To reuse the
window, clear the variable first with `set FUSION_TSF_DIAGNOSTICS=` (an empty value is off). If a permanent variable
was set by mistake, remove it at User scope and confirm it is empty:

```powershell
[Environment]::SetEnvironmentVariable('FUSION_TSF_DIAGNOSTICS',$null,'User')
[Environment]::GetEnvironmentVariable('FUSION_TSF_DIAGNOSTICS','User')
```

### The Fusion tray (U7 — installed as `0.0.0-wp7+0094269`)

The decided presentation (D084) now exists as `FusionTray.exe` in the installed version directory. It is started at logon by
its own scheduled task (`FusionTray`, Medium integrity, interactive token) — not from inside the Host (a crashed Host
cannot report itself) and not once per application's TSF DLL. One instance per user × session; a second start exits
quietly, so an upgrade cannot leave two icons.

**What it shows.** Hover the icon: `Fusion: Host running (pid N)` or `Fusion: Host not running`. "Running" is a process
fact, **not** a health claim: an alive-but-unresponsive Host still reads as running (see the hang-detection note below).
On the first installed package (`0.0.0-wp7+0094269`, 2026-10-20) the owner confirmed this tooltip read
`Fusion: Host running (pid 41396)`. That is the whole of the installed confirmation so far: the commands below, the
balloon and the session-end behaviour were **not** exercised.

**What it does.** Right-click the icon:

| command | effect |
|---|---|
| **Restart FusionHost** | runs `restart-host.ps1` from the installed version directory — the same tool as §2 |
| **Open log folder** | opens `%LOCALAPPDATA%\fusion-host` (created if missing). It does **not** enable logging |
| **Exit Fusion tray** | removes the icon. The Host keeps running |

**When it notifies.** One balloon per `running → absent` transition, never per key, and never for a transition it did not
witness (a tray that starts while the Host is already absent says so in the tooltip and stays quiet). A **manual
restart** from the tray, the Start Menu or the installer, and a **session end**, are recognised as intentional and never
produce the balloon. Recovery (`absent → running`) is silent; the tooltip changes back. A replacement Host under the same
running state is not a transition either: the tooltip's pid refreshes and nothing is reported.

**A cancelled session end.** The tray exits when the session really ends, so a logoff or restart that another application
then cancels could leave no tray behind. Its own task carries the same User32 1077/1073 cancellation trigger the Host
task uses, so the tray is started again in the same session; a tray that was still running when the end was cancelled
resumes normal reporting instead of staying silent, and does not replay the Host stop that happened during the attempt.
A second start is harmless either way: one tray per user × session is enforced by the tray's own single-instance claim.

**What it does not do.** No automatic restart (not authorized), no hang detection, no per-key or per-application
notification, no focus stealing, no input content, no log of its own, and it never stops the Host — not even when the
tray exits. Diagnostics remain default-off; the tray neither reads nor enables them.

**Why a separate logon task.** A task instance stays alive while any of its actions runs, so a long-lived tray action on
the `FusionHost` task would keep that task in the Running state and `restart-host.ps1`'s start (`MultipleInstances =
IgnoreNew`) would be silently ignored — a manual restart would appear to do nothing. The tray therefore has its own task,
which install creates/refreshes and uninstall deletes. **Install and upgrade** stop the tray before the Host (orderly
close, reported if it has to be forced) and start the new tray afterwards; **uninstall** stops the tray and does not
start it again — the task, the icon and the shortcut go away. The manual restart tool writes a short-lived marker so its
own Host stop is not reported as a failure. The marker is honoured for five minutes, so one left behind by a crash cannot
silence a later real exit.

**Only the product's tray is touched.** The installer, the uninstaller and the restart tool act on a tray only when it is
this user's, in this session, and running from `<install root>\<version>\bin\FusionTray.exe`; any version directory
counts, so an upgrade can stop the previous version's tray. A process that merely shares the name `FusionTray` is
reported as `left alone (identity not verified)` and is never signalled.

**Not yet observed on this machine.** The tray was built and validated with isolated test processes and a stubbed restart
tool, without installing it and without stopping the running Host. So: the `running → absent` balloon itself, the
session-end suppression and its cancellation recovery, and the install/upgrade/uninstall integration have **not** been
exercised end to end here; the transition rules are covered by unit tests and the scoped inventory, the window ownership
and both `WM_ENDSESSION` cases by an isolated harness. The two assumptions that remain open are that a Host killed during
logoff always ends before the tray sees the session end, and that no application blocks the session end long enough to
race the marker. Hang detection (alive but unresponsive) is the remaining design work and is **not** in this version.

## 5. Reporting a problem

Only when something goes wrong. Include:

- **time** (local, with date);
- **application** and window/field (e.g. Notepad, Firefox address bar);
- **symptoms** (what appeared, what was expected);
- **reproduction** if known (keys/actions in order);
- **candidate identity** — the version directory name (`C:\Program Files\Fcitx5 Fusion\<version>`) or the package name;
- **logs**: `%LOCALAPPDATA%\fusion-host\` (if diagnostics were enabled) and `%LOCALAPPDATA%\fusion-tsf\` if present.

No daily report is required, and no feedback is needed for a day without problems. Resource observations stay incidental
(Task Manager's Host memory/handle count if something looks wrong); there is no mandatory benchmark or monitoring.

## 6. Known limitations at this scope

- **Browsers: the candidate popup now appears in normal use** (R25 — **recovered, owner-confirmed 2026-10-20** on package
  `0.0.0-wp7+995e5a5`, source `995e5a5`). The owner reported "firefox 和 edge 目前正常" after installing that package and
  restarting the browsers, so the persistent "no candidate popup in a browser field" incident is closed at this
  current-use scope; it is not a claim about every field on every site.
  - **Measured cause (2026-10-20):** Firefox answers `ITfContextView::GetTextExt` with `TS_E_NOLAYOUT` (`0x80040206`) for
    the whole composition on every keystroke, so nothing could be anchored and the page stayed hidden; composition and
    commit never used the anchor and kept working. The same context answers the selection/caret range from the second
    keystroke, which is what fusion-windows PR #25 falls back to (`source=composition-caret`).
  - **Historical measurements, unchanged:** in fresh isolated processes with the installed package the popup appeared
    from the first keystroke in the Firefox and Edge address bars and from the second in a page textarea and the Gemini
    composer. The owner's earlier per-field table (Gmail compose, the ChatGPT editor, the Gemini editor, all Edge cells)
    described browser processes that had never loaded the installed build — one of them built from source with no
    candidate-window code at all — and is kept as a measurement of those builds, not of this package. The individual
    fields of that table were **not** re-checked one by one; the recovery rests on the owner's report for normal Firefox
    and Edge use.
  - **Remaining limit:** in web content the first keystroke of a composition has no layout in any range, so the popup
    appears from the second keystroke there; the address bars anchor from the first. That is a one-key gap, not an
    explanation for a popup that never appears.
  - **If a field ever shows no popup again:** restart the browser first (see the next bullet) and then, if you want to
    report it, run the optional capture in §4 "R25 browser capture" — it gives the exact steps, the log location and
    what to report.
- **After installing a new package, close and reopen the applications you want to test.** A process keeps the
  `FusionTsf.dll` it loaded at startup, so an already-running browser or editor can still show the previous build's
  behaviour — and only that build's diagnostics — until it is restarted. This is the same loader fact recorded as R28.
  Measured on 2026-10-20: a browser started three days earlier still held a development build, and the Edge process used
  for that day's report held one built from source that has no candidate-window code at all (`stage3c1`, base commit
  `5794a36`), so its "no popup in the address bar either" result described that old build rather than the installed
  package.
- **Notepad-- 3.4 crashed on Win+Space with the 76bc009 package** (R35). The cause was the TIP binding to Notepad--'s
  own older `MSVCP140.dll`, not the input behaviour; fusion-windows PR #21 removed the CRT mutex that triggered it, and
  the owner confirmed on the repair package that switching to Fusion no longer crashes. That covers the activation crash
  in this application, not universal Notepad-- compatibility. Other applications that ship their own older
  `MSVCP140.dll` would have shown the same crash.
- **Notepad--: Enter did not start a new line** while Fusion was the Chinese input method, after committing with Space
  (R36). The TIP claimed the key and then handed it back, and that application had already stopped handling it;
  fusion-windows PR #23 (merged) leaves an idle Enter to the application. **Confirmed by the owner on 2026-10-20 with
  `0.0.0-wp7+24914ff`:** the `nihao` → Space → Enter sequence inserts the newline again. Enter during a composition,
  and candidate selection, are unchanged.
- A new application window should start with Fusion open (U6, owner decision 2026-10-19; implemented in fusion-windows
  PR #20 and **awaiting its first real-window confirmation**). Check with Fusion already selected (see §1). If a new
  window still starts closed, press Ctrl+Space once and report it with the time and the application.
- **The tray reports a process, not a health state** (U7). `Fusion: Host running (pid N)` means this user's Host process
  exists in this session; it does not mean the Host is responsive, and the tray does not detect an alive-but-unresponsive
  Host. It also does not restart anything by itself: **Restart FusionHost** is always your action. Its first installed
  run, the fault balloon and the install/upgrade/uninstall handling are described in §4 "The Fusion tray", including what
  has not been observed yet.
- The TIP is a development build: its description and identifiers are the development ones (R29); signing, installer
  packaging and a distribution channel are not decided (U4).
- An alive-but-unhealthy Host (2 s request bound, R7) is not restarted automatically and the tray does not detect it; use
  the restart shortcut or the tray's **Restart FusionHost**.
- Multi-account machines: the logon task name is global (R32, hypothesis); this guide covers one interactive account.

**No zero-loss, zero-corruption or automatic-recovery guarantee is made.** The mechanisms give explicit terminals for the
scenarios in `docs/windows-stage6-plan.md` §8.2; a problem is assessed and repaired by impact under the daily-use policy.

## 7. Acceptance

WP5 and WP6 are accepted through this daily-use period: the owner reports a problem when one occurs, it is repaired by
impact, and **closing the period without reported problems is the acceptance** (owner decision 2026-10-19, D084). No
separate fault-injection or application-matrix campaign is required. Daily use is **not** complete until the owner says so;
unexecuted scenarios are never recorded as PASS.
