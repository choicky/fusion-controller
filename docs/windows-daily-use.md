# Windows daily use (WP7) — the one entry point

This is the owner-facing guide for the Stage 6 WP7 daily-use period. The durable decisions are in
`docs/DECISIONS.md` (D084) and `docs/windows-architecture.md` (§10, §14); the acceptance policy is in
`docs/windows-stage6-plan.md` §8.1. **Daily use is not complete until the owner closes the period** (see §6).

## 1. Install, upgrade, uninstall

The package is owner-only and unsigned. Everything except the last step is elevated.

| step | command |
|---|---|
| install | extract the zip, then run `install.ps1` **elevated** (`-PackageDir <extracted folder>`, default is the script's own folder) |
| upgrade | extract the new zip and run its `install.ps1` elevated; it stops the running Host in order, copies the new version directory, re-registers the TIP and refreshes the logon task |
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

Start menu → **Fusion → Restart FusionHost** (`restart-host.ps1` in the installed version directory). It:

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
authority. There is **no Fusion tray icon yet** (U7 is decided but not implemented) — the indicator is the only status
surface in this build.

## 4. Diagnostics (off by default)

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

### U7 minimal design (presentation decided, implementation pending — nothing here exists yet)

Owner process: a small per-user tray process started at logon (its own scheduled task, Medium integrity), not inside the
Host (a crashed Host cannot report itself) and not one instance per application's TSF DLL. Single instance per user ×
session via a named object.

Detection: a low-frequency timer (≈5 s) enumerates this user's/this session's `FusionHost` processes through the same
helper the restart tool uses. State transitions are `running → absent` (notify) and `absent → running` (clear). An
intentional shutdown is recognised by the session ending (the tray receives `WM_QUERYSESSION`/`WM_ENDSESSION` itself) and
by a short-lived "restart in progress" marker written by `restart-host.ps1`; those transitions never notify.

Distinguishing crash from timeout: a missing process is an exit (crash or kill); an alive process with failing requests is
**not** proven to be a crash. The minimal version therefore reports only "the Host is not running". Hang detection
(alive but unresponsive) needs an additional probe over the existing pipe (or the Host log's last timestamp) and is
explicitly **not** in the minimal version — it is the remaining design work.

Notification: one balloon per transition, suppressed for a window after the first (deduplication), never per key and
never for a transition the tray itself caused. Actions: **Restart Host** (runs `restart-host.ps1`) and **Open log
folder** (`%LOCALAPPDATA%\fusion-host`). Automatic restart is **not** authorized.

Unconfirmed assumptions: that a Host killed during logoff always ends before the tray sees the session end, and that no
application blocks the session end long enough to race the marker. Both are to be validated with the implementation.

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

- **Browsers: the candidate popup was missing in some fields** (R25). Firefox and Edge both showed the symptom on
  2026-10-20 — no popup in the affected field — while `nihao` + Space still committed 你好. Measured that day: a fresh
  Edge process shows the popup, while Firefox answers the anchor query with `TS_E_NOLAYOUT` for the whole composition,
  which hid it. fusion-windows PR #25 falls back to the caret position and shows the popup; that needs the next package,
  and owner confirmation is pending. The first letter of a composition still cannot be anchored in either browser, so
  the popup appears from the second key. The address bar and the Google homepage search field showed the popup in the
  2026-10-19 observation and were not retested since; the ChatGPT conversation editor was the field reported broken
  that day.
- **After installing a new package, close and reopen the applications you want to test.** A process keeps the
  `FusionTsf.dll` it loaded at startup, so an already-running browser or editor can still show the previous build's
  behaviour — and only that build's diagnostics — until it is restarted. This is the same loader fact recorded as R28,
  and it is why a 2026-10-20 Edge report described the old build rather than the installed package.
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
- The TIP is a development build: its description and identifiers are the development ones (R29); signing, installer
  packaging and a distribution channel are not decided (U4).
- An alive-but-unhealthy Host (2 s request bound, R7) is not restarted automatically; use the restart shortcut.
- Multi-account machines: the logon task name is global (R32, hypothesis); this guide covers one interactive account.

**No zero-loss, zero-corruption or automatic-recovery guarantee is made.** The mechanisms give explicit terminals for the
scenarios in `docs/windows-stage6-plan.md` §8.2; a problem is assessed and repaired by impact under the daily-use policy.

## 7. Acceptance

WP5 and WP6 are accepted through this daily-use period: the owner reports a problem when one occurs, it is repaired by
impact, and **closing the period without reported problems is the acceptance** (owner decision 2026-10-19, D084). No
separate fault-injection or application-matrix campaign is required. Daily use is **not** complete until the owner says so;
unexecuted scenarios are never recorded as PASS.
