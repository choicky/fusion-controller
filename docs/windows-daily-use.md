# Windows daily use (WP7) — the one entry point

This is the owner-facing guide for the Stage 6 WP7 daily-use period. The durable decisions are in
`docs/DECISIONS.md` (D084) and `docs/windows-architecture.md` (§10, §14); the acceptance policy is in
`docs/windows-stage6-plan.md` §8.1. **Daily use is not complete until the owner closes the period** (see §7).

## 1. Install, upgrade, uninstall

**The installer is one EXE.** Download `fusion-<version>-setup.exe` and run it; nothing has to be unpacked first and
nothing is downloaded while it installs — the whole product travels inside the EXE. It is owner-only and **unsigned**, so
Windows shows the usual "unknown publisher" prompt, and it asks for administrator rights once because the persistent
registration of the input-method component writes `HKLM`. The wizard is in Chinese, shows the version it is about to
install, and reports what happened at the end.

| step | what to do |
|---|---|
| install | run `fusion-<version>-setup.exe`, accept the elevation prompt, follow the wizard |
| upgrade | run the newer `fusion-<version>-setup.exe`; it names the version already installed, then closes the tray, stops the running Host in order, installs the new version directory, re-registers the TIP, refreshes the logon tasks and starts the new Host and tray |
| same version again | run it again; it says this version is already installed and re-applies the install. Files whose SHA-256 already match are skipped, so nothing is rewritten |
| uninstall, keeping user data | **设置 → 应用 → 已安装的应用 → Fusion 输入法 → 卸载** (or `C:\Program Files\Fcitx5 Fusion\uninstall.exe`), answering 否 to the user-data question |
| uninstall, removing user data | the same uninstaller, answering 是 — it also deletes `%APPDATA%\Fcitx5` and `%LOCALAPPDATA%\Fcitx5` |

The wizard runs the package's own `install.ps1` / `uninstall.ps1`; the progress page shows the same phase-by-phase output
those scripts print, and both write a transcript to `%LOCALAPPDATA%\fusion-installer\`. The progress page ends with
`清理临时安装文件…`, which deletes the payload the wizard unpacked (about 285 MB in 658 files) **before** the 完成 page
appears, so 完成 closes the wizard instead of leaving it deleting files behind a page that already says the install
finished. A failed install aborts the wizard with the exit code and that path instead of finishing as if it had worked, and
re-running the EXE retries it from the start; a failed uninstall keeps the "已安装的应用" entry so it can be retried.
Neither path modifies `PATH`, force-deletes a directory a process holds, or touches `%APPDATA%\Fcitx5` unless removal was
chosen. A version directory held by a running application stays until the next uninstall after sign-out (R28); both paths
report the holders.

The first real attempt (2026-10-23, `+c2a5ee1`) failed exactly this way: it stopped at exit 1 in `Add-Type` — NSIS had
unpacked its own native `System.dll` into the working directory the installer gave PowerShell, and `Add-Type` resolved the
default reference to that file — and, as the paragraph above says, nothing on the machine changed: no version directory, no
entry, no registration. **The next attempt, `+3325514`, is the accepted installer build:** installing (over the
zip-installed `0.0.0-winui3+43a832f`), Chinese input in Notepad, the "已安装的应用" entry with that version,
uninstalling from it while keeping user data, and a reinstall that kept learning data and settings were all carried
out and all passed. The build named in §7 is newer than it and is the one to install now.

**The zip is the fallback, not the normal route.** Every version is also published as `fusion-<version>.zip` for the cases
the EXE does not serve: a machine where an unsigned EXE may not run, a scripted or unattended install, and rolling back to
an older version. Extract it and run `install.ps1` **elevated** (`-PackageDir <extracted folder>`, default is the script's
own folder); run `uninstall.ps1` **elevated** from the installed version directory to remove it, with `-RemoveUserData` to
delete the user data as well, and `-VerifyOnly` to check the manifest and hashes without changing anything. Two differences
are worth knowing: a zip-based install does **not** create the "已安装的应用" entry, and a zip-based upgrade *after* an EXE
install leaves that entry naming the version the EXE last installed — its uninstall command still removes every version.

**Rollback:** run the previous version's `install.ps1` elevated (it installs side by side and re-registers that version),
or uninstall and install the earlier package. User data is kept either way.

**After an upgrade, restart the applications that were already running.** The install replaces the input-method files
and re-points the registration, but a process keeps the version it loaded, so an application started before the
install — Explorer is the one that matters most, because it lives for days — keeps using the previous build. When
that build is from a different protocol version, it cannot talk to the new Host at all: the input method simply does
nothing in that process (no candidates, no composition, keys typed raw) until it is restarted, and Ctrl+Space does not
help because nothing is listening for it. Signing out and back in, or restarting Explorer, is what loads the new one.
R48 records the measurement behind that: the mismatch is confirmed, and the owner restarted Explorer on 2026-10-26 and reports 双拼 working again and the machine usable overall. The occasional stalls were not reproduced and the restart is not evidence about them — their cause is still undetermined.

**Installing registers and enables Fusion; it does not necessarily make it the selected input method.** After install (and
after every upgrade), check the Windows input indicator in the taskbar. If Fusion is not the current input method, press
**Win+Space** until it is, then type. Judge U6 only with Fusion already selected — otherwise "letters appear raw" means the
IME was not selected, not that a new window failed to open.

## 2. Opening Fusion Settings

The package installs **Fusion → Fusion Settings** in the Start Menu, and the tray opens the same executable from its
**Settings** command and from a left click. The client lives in `bin\settings\` because it carries its own .NET and
Windows App SDK runtimes; nothing else needs to be installed for it to start.

Fusion Settings talks to the current user's and current session's `FusionHost` over the existing user/session-scoped pipe.
It does not start a second engine, restart the Host, or register anything. If the Host is absent the window says so and
stays usable; if it disconnects, times out or refuses the settings API version, the operation is reported as unsuccessful
rather than as done.

**The five pages.**

- **输入方式** — the ordered list of methods, with the first entry marked **当前** because that is the method a bound
  context follows. **上移 / 下移** reorder, **移除** leaves the group, and **添加** takes a method from the available list.
  Removing an entry never deletes a method definition or learning data. The fallback method (`DefaultIM`) is shown but not
  edited here. The 正在编辑 selector at the bottom chooses *which* method's settings you are editing — it does **not** switch
  the active input method.
- **按键** — one row per action, each holding the alternative bindings for it. Three tags are shown with **N 个更多** for the
  rest; **编辑** reveals the per-tag **替换** and remove buttons; **添加按键** records a new binding (press the keys, **Esc**
  cancels and changes nothing). **恢复默认** writes the Host's own default for that action into the draft — nothing reaches the
  Host until Save. A binding this build cannot interpret is shown in italics and kept as it is, not replaced. A repeated or
  cross-action binding is reported in a warning line and left alone: which one wins is the engine's business. The **，/。**
  preset edits the same `PrevPage`/`NextPage` lists and asks whether to replace or extend them. **系统快捷键** and
  **系统回退翻页** are separate sections: the first is the system's own shortcuts, the second is the core's fallback paging,
  and neither is the Pinyin engine's paging keys. **辅助筛选** holds **辅助筛选触发键** — the keys that start filtering a
  candidate list with 辅助码; whether that filtering is 笔画 or 墨奇 is chosen on 候选与输入, so the key has one home and
  the page states which one it is.
- **候选与输入** — the candidate window itself, then candidates, **英文候选与混输**, **辅助筛选**, preedit, learning,
  模糊音, the Shuangpin-only fields, and 系统输入行为. The **候选窗口** section at the top holds **候选排列方向** — **竖排** (one candidate per row), **横排** (this
  page's candidates on one row) or **自动卷轴** (below), shown in Chinese rather than as the stored
  `Vertical`/`Horizontal`/`AutoScroll` — plus **候选字体** and
  **字号（磅）** — 候选字体 is a searchable list of the fonts this Windows session has; opening it puts the font in
  effect at the top **and marks that row — a tinted background and a short accent bar at its left edge**, so the list
  starts at the current family instead of at "A" and says which one it is. The same mark, in the same place, is what
  every other selection list in this window uses for the value in effect: the closed-set dropdowns (候选排列方向,
  上屏方式, 候选字号 …) and the 拼音/双拼 selector draw it the same way, and it moves when the selection changes. The
  mark follows what the control holds — an unsaved choice included — it is a row style rather than part of the name,
  and opening any of these lists changes nothing. Any name the font list does not contain can still be typed — with a small preview of the candidate window's own typography and **恢复默认字体** to put both back to the
  Host schema's defaults — and it now rebuilds the page so the two editors show the reset values straight away, not only the preview. They save with the same **保存** button as everything else — there is no separate apply — and they
  take effect from the next candidate page, so the popup already on screen does not move under the pointer. A profile that
  never touched them keeps the vertical list and the Microsoft YaHei UI look this product has always drawn, so upgrading
  changes nothing. The size is in **points**, so the same setting stays the same physical size on a 150 % or 200 % display;
  the list is the system's own font collection, so a name you pick is one the popup can draw, and a name that is *not*
  installed is never rewritten on your behalf: the stored name stays, the row says Windows will substitute its default, and
  missing characters are filled in by the system font linking. When an
  input method asks for a particular direction for one list (quick phrase, Unicode and the input-method selector do), the
  input method's choice wins over this setting. The 外部工具 section lists the upstream tools this package does not ship,
  with the reason — except **词库管理**, which is implemented in this window and opens its own page (see below).

  **自动卷轴** is the third arrangement, and it is about how much of the list is on screen at once rather than about a
  direction. A new word starts on **one line**, exactly like 横排. The first time you press the configured "next page" key
  the panel grows: each row is one page of candidates — the number of columns is the input method's own page size, so a
  page of seven gives seven columns — and the rows you get are the rows your screen holds, up to five. A short screen simply
  turns shorter pages (three visible rows page three at a time), so "next page" always means the next row you can see, and
  the font is never shrunk to make more fit. From there each "next page" moves down **one row** and **keeps the column you
  were on**; when the next page has fewer candidates than that, the highlight lands on its last candidate instead of jumping
  back to the first. On the last row of what is on screen it moves to the first row of the next screenful, "previous page"
  goes back the same way, into the last row of the previous screenful, and it stops at the ends instead of wrapping around.
  **Each column is as wide as its own widest candidate**, so a page with one long candidate and single characters
  beside it draws one wide column and narrow ones instead of widening all of them; the columns still line up from row
  to row, and the widths are taken from every row of the scroll window so the grid does not shift while you move.
  The **numbers** are drawn only on the row the keys act on — that row is the engine's current page — so pressing `3`
  selects the candidate labelled 3 on that row and nothing else — and with no candidate list on screen a number key is
  just a number, on the top row and on the keypad alike, instead of being claimed and dropped; the same holds for the
  **symbols** — `/`, `-`, `+`, `=`, `@` and the keypad operators type through when the input method has nothing to do
  with them, while the punctuation it does convert (`,`, `.`, `?` …) still comes out as `，` `。` `？`; the **mouse**
  can click any visible row, and that candidate is what gets committed. **Left and right arrows** move the highlight between candidates in this window — the highlight is
  the engine's own cursor — **space** commits the one it is on, and up/down keep the row navigation described above. At the
  first or the last candidate the arrows stop instead of wrapping, and there the key goes back to the input method, which is
  how the caret inside the pinyin string is moved; so editing what has been typed stays possible, and 竖排 and 横排 keep the
  arrow keys they have always had. Type anything new, pick a partial candidate, or switch input method, and the panel is back to one
  line. If an input method does not publish its pages (or asks for a particular direction itself), this setting simply
  leaves its usual arrangement in place; and if the list cannot be read in full — a page whose candidates exceed what one
  update may carry, or a list that refuses a candidate — the window stays the ordinary single row or column rather than
  showing a grid with a hole in it.

  A candidate is drawn whole whenever it fits — one character, two characters and ordinary short words are no longer
  shortened (an earlier build measured them one way and drew them another, so every 横排 cell lost a little text and the
  full-text box never appeared for the case that needed it). A cell shows at most **eight characters' worth of
  candidate text** — eight 汉字 measured in your candidate font, with the number (`1.`) and the cell's own padding
  added on top rather than taken out of the eight — and only a candidate genuinely longer than that is shortened
  with an ellipsis instead of widening the window past the screen. The eight belongs to the **cell**, not to the
  column or the row: a candidate on an unnumbered row of the grid (the rows above and below the engine's own page)
  gets the same eight, without the numbering allowance a numbered cell carries — hover it,
  or leave the keyboard highlight on it, and the full text appears beside the panel. That popup never takes focus and never
  changes what is selected, and what is committed is always the engine's own full string, not the shortened one. If the text
  is taller than the box, the box scrolls: move the pointer *into* it — across the gap as well, since a pointer on its way
  from the candidate to the box no longer dismisses it — then use the wheel and the end of the text is reachable, with a
  scrollbar showing where you are. The box closes when the pointer leaves both the candidates and the box, and a pointer that
  simply stops beside them leaves nothing behind. The box is placed in the largest space beside the panel that does not
  overlap it, so a scrollable full-text box never covers the candidates; if no such space exists at all it is not shown
  rather than shown over them. A text short enough to fit does not scroll and lets the pointer through, so it can never get in the
  way of a click on a candidate. The panel itself opens below the caret and moves above it when there is no room, staying
  inside the display's work area; moving the caret to another monitor re-measures for that monitor's scaling instead of
  reusing the old size. A vertical list stays **one column**: when it is taller than the space it is given, the list keeps
  the highlighted candidate in view and the rows before it as context, a shaded bar at the top or the bottom of the candidate
  area shows that more candidates are hidden that way, and the paging keys walk the list one row at a time — no candidate is
  dropped, the font is never shrunk, and the list never continues into a second column. **Implemented and verified in
  isolation** (fusion-windows PR #36, head `ff3bd8f`: real-window geometry checks including the
  continuous candidate → gap → box path, the measure-at-the-width-it-is-drawn check and the never-overlap-the-panel cases,
  and 恢复默认字体 refreshing both font editors); **not yet verified in a real application**, and the non-default-DPI visual
  check is still unobserved (`WM_DPICHANGED` is unit-tested only); owner acceptance is pending (plan R39). Three
  arrangements are implemented in isolation and verified against the real engine and the real popup, and **not yet in a
  real application**: 竖排 and 横排 are D088 (PR #33, documented in PR #56), the geometry, font, DPI and full-text work is
  R39 (PR #36), and 自动卷轴 is R40 (PR #37: real-engine one-line → expansion, row and scroll-page steps with the column
  kept, a click on another row committing that candidate). The check counts for each round are in the plan's round
  paragraphs rather than repeated here. **Install the build this batch packages** — the package and single EXE named in
  the plan's *State by level* table, not repeated here — before checking any of it: the version installed on this
  machine predates it, and a settings window opened before that upgrade keeps showing the build it was started from —
  and note that installing is not acceptance. The 自动卷轴 pass is
  yours to make: select it, type a syllable, press the configured next-page once and check the panel grows, that the
  numbers sit on the row the keys act on, that clicking another row commits that candidate, that a page with one long
  candidate beside single characters draws one wide column instead of widening all of them (R42), and that a new word
  starts on one line again.
- **English words and 墨奇 in the candidate list (new in this build).** The Chinese engine is now the same one the
  Android product runs, so a Latin word can be a candidate in its own right: type `hello` and the list offers `hello`
  beside the Chinese candidates, and selecting it commits it as typed. Typing one string that mixes both gives a
  candidate carrying both, and it commits as the whole mixed string: `woxiangmaiiphone` offers `我想买iPhone`, and in
  双拼 both `woiphone` → `我iPhone` and `woxdmliphone` → `我想买iPhone` are on the first screen, right behind the
  Chinese head. It is not required to be the first entry — the Chinese reading keeps row 0 — and other fused
  readings can still sit behind the Chinese list, so page on if the one you meant is not there. **显示英文候选**
  (`SpellEnabled`, 候选与输入 → 英文候选与混输) is the only switch and it is on by default; turn it off and no English
  candidate appears, while pinyin, its candidates and 双拼 behave exactly as before — there is **no separate “mixed
  input” switch**, and **混输候选自动加空格** (`MixedAutoSpacing`, same group) stays off unless you turn it on: it puts one
  space where one candidate's own Chinese and English segments meet, so the committed string carries it, and it does not
  touch Chinese or English candidates or text already in the document. **辅助筛选方式** (`AuxiliaryFilter`, 候选与输入 →
  辅助筛选) still defaults to 笔画: with a candidate list on screen press the auxiliary-filter key — **辅助筛选触发键**, bound
  on the 按键 page in the same 辅助筛选 group — and the band above the candidates says `[Stroke Filtering]`. Set the option
  to **墨奇** and the same key says `[MoQi Filtering]`; then type the two letters of a character’s 墨奇 code — `nz` for 好,
  `rx` for 你 — and the list narrows to what that code names, with the first candidate committing. Both the table and the
  English word list travel inside the package, so this works with no network. The English, 墨奇 and mixed candidates above
  are verified on the real engine in isolation. **The owner reports the mixed-input fix passing the test it was given** —
  双拼 mixed candidates on the first screen — and that is the whole confirmed scope: it is not an acceptance of 墨奇 inside
  a running application, and it says nothing about the Android build.
- **词库管理** — reached from 候选与输入 › 外部工具 › **词库管理**, and from the left navigation. The list is what the engine
  itself can load: every `.dict` in your own data folder (**用户词库**) or in the package (**随包提供**), whether it is
  enabled, and its file size — no word counts, because nothing on disk knows one without loading the dictionary.
  **导入词库…** takes several files at once. A `.dict` is loaded first to prove it is a pinyin dictionary; a `.txt` is a
  text dictionary in libime's own format — `汉字 拼音 [权重]` per line, e.g. `墨珂 mo'ke 1000` — and is converted before it is
  placed. Either way the file is validated first — and a file that holds no real entry at all is refused: an empty file, or
  ordinary prose, is not an empty dictionary, so the window says why instead of placing it, while a line libime cannot read
  beside valid ones still imports. The validated file is then moved into your dictionary folder, so a file that fails
  leaves the dictionaries you already had untouched. When the name is already taken the window asks **覆盖** or **跳过**;
  nothing is replaced silently. **停用** writes the engine's own `<file>.dict.disable` marker and **启用** removes it — a
  dictionary that came with the package can be disabled but not deleted, and **删除** (after a confirmation) removes only a
  user dictionary and its marker, never `user.dict`, the learning history or the system dictionary. Each change reloads the
  engine through its own dictionary-manager entry, so a new word appears without restarting anything; while a word is being
  composed the change is refused and nothing is written — finish or cancel it and repeat. **重新加载** re-reads the folder,
  which is also what to press after dropping a file there by hand. 搜狗 `.scel` cannot be imported: its converter is not
  part of this Windows runtime. 拼音 and 双拼 share one set of dictionaries; there is no second copy to keep in step.
- **关于与诊断** — version, endpoint, connection state, the known limits, and a technical trace.

**Saving.** Edits are held in a draft, one per underlying configuration: 拼音 and 双拼 share one because they are one
configuration, and the global configuration has its own. The bar at the bottom always shows whether anything is pending.
**保存** sends only what changed, one request per configuration, and reports each result separately — a refusal on one does
not discard the other, and an unapplied edit stays in the draft. **放弃修改** throws the draft away. Closing the window with
unsaved edits asks first. Changing pages never loses edits.

A save that changes nothing sends no request at all. Clearing every binding of an action, or emptying a rule list, is a
change and saves like any other.

**Saving while you are still typing.** You never have to finish a word twice. The **候选窗口** page (排列方向, 字体, 字号) and
the **全局** page are written immediately: the panel already on screen keeps its size and place, and the new value is used from
the next candidate page. The **拼音/双拼** configuration and the **输入方式列表** cannot be applied without ending the
composition that is running — the input method's own setter clears its buffer, and reordering the list switches the method a
context uses — so the Host refuses that save and says it wrote nothing. The window then does the waiting for you: the bar says
`尚未保存，等待当前输入结束或 Host 空闲后自动重试`, keeps your edit, and submits it again by itself as soon as the word is
finished. 停止等待 stops that; the edit then stays unsaved until you save again. Only a refusal is retried — a timeout, a
dropped connection or an uncertain write is reported and left to you, and a page that already saved is not sent again.

If a write's outcome is uncertain — the reply was lost, the connection dropped, or the Host said
so — the window says the change **may have applied**, re-reads the configuration and shows what the Host actually has; edits
the reread does not confirm stay pending. It never silently retries.

**While a request is running** the pages are read-only and the buttons are disabled, so nothing can change under a request
that has already been sent. Typing in a field commits when you leave it, and both Save and closing the window commit it for
you, so the value you just typed is the value that is saved — and the question "close without saving?" is only asked after
that last edit has been counted. Closing while a save is in progress is refused with a message rather than discarding the
draft under it.

**If the Host is not there.** At startup the window says so and offers **重试连接**. Once connected, the same **重试连接**
appears if the Host goes away mid-session, and **重新加载** re-reads everything and asks first if anything is unsaved. A
retry keeps your pending edits; only an explicit reload discards them. A part of the window you had not edited follows the
Host instead: if the input-method order changed elsewhere, a retry shows the Host's order rather than presenting the old one
as an unsaved change you would later save back.

**What the pages do not claim.** A binding stored here is a configuration value; whether that key reaches the engine depends
on the input path. Combinations with Ctrl/Alt/Win, modifier-only keys and keypad Enter are not delivered on this path, so
those bindings can be saved but will not fire. The 按键 page says so where it matters. The 关于与诊断 page repeats it.

Packages before this one shipped the Qt client with a different layout (Pinyin/Shuangpin/Global pages and an apply-per-page
button). This package does **not** contain it: the build has the Qt client switched off, so the only settings entry is the
one described above, and going back to the Qt client means going back to an earlier package.

The current package is an owner-test artifact only. Implementation, isolated validation, packaging, owner installation and
owner acceptance are separate status claims, and this guide does not imply the last two.

## 3. Restarting the Host

Start menu → **Fusion → Restart FusionHost** (`restart-host.ps1` in the installed version directory), or the tray icon's
**Restart FusionHost** command (§5, "The Fusion tray") — both run the same tool. It:

1. asks the running Host for an **orderly** stop through the product's own stop event and waits up to 20 s;
2. if it is still alive, force-stops **only** the instance verified as this user's and this session's, and says so — that
   instance does not run Fcitx's own save;
3. starts the installed Host through the existing logon task (Medium integrity);
4. verifies the executable path, the integrity level and that exactly one Host is running.

An absent Host is a normal restart. Another user's or another session's Host is reported and never touched. Nothing here
enables automatic restart or changes the task's restart-on-failure setting.

## 4. Selecting the backup input method

Press **Win+Space** to cycle input methods; the indicator shows the current one. Use it if Fusion is not selected, or to
compare against another IME. Installing does not force Fusion to be the selected method; the Windows indicator is the
authority. The Fusion tray icon (§5, "The Fusion tray") reports whether this user's Host is running; it is not an
input-method selector — the Windows indicator remains the authority for which method is current.

## 5. Diagnostics and the tray

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

R25 was **reported recovered by the owner on 2026-10-20** (see §7), so this capture is no longer a required step. Keep it
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
     `C:\Program Files\Fcitx5 Fusion\0.0.0-winui3+43a832f\bin\FusionTsf.dll` (the running Host and tray are that build),
     and after installing the package in §7 whatever version directory
     `HKLM\SOFTWARE\Classes\CLSID\{8B7C4A90-5E2D-4B1F-9A31-6C0F42D1E7A4}\InprocServer32` names.
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
| `ANCHOR_PROBE stage=text-ext source=composition hr=0x80040206` | `TS_E_NOLAYOUT` — neither the composition range nor the caret fallback had layout for that keystroke, so the page is hidden and stays hidden until the next key re-probes it (R46). Measured 2026-10-28 in the owner's browser log: every `hidden reason=no-anchor` sits directly after one of these, 142 of 151 right after `COMPOSITION_START` and 9 after a mid-composition batch |
| `ANCHOR_PROBE stage=no-active-view` / `stage=no-range` | the context had no active view, or the selection returned no range — the anchor was not even attempted |
| `CANDIDATE_VIEW hidden reason=no-anchor` / `no-page` | nothing to anchor to (R46's symptom — the failing probe is above it) / no candidate page to show |
| `stage=ok` and still no visible window | the anchor is fine and the defect is in placement or visibility. R49's step 4 measured exactly this in the taskbar search box (`stage=ok source=composition rect=1024,2101,1039,2132 clipped=0` with nothing visible): the popup has no owner window and is not topmost, so the flyout covers it; the fix is to own the popup to the document's window and make it topmost (the owner's decision of 2026-10-28, to be recorded as D090) |

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

**What it does.** In the merged tray update (fusion-windows PR #30, `main` `504f48c`; **not yet in an installed package**) a **left click**
opens the settings window, or raises the one that is already open instead of starting a second copy; if Windows refuses to bring
it forward the taskbar button flashes. Until that update is installed, use **right-click → Settings**. Right-click keeps every
command below, and the menu stays keyboard reachable:

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

## 6. Reporting a problem

Only when something goes wrong. Include:

- **time** (local, with date);
- **application** and window/field (e.g. Notepad, Firefox address bar);
- **symptoms** (what appeared, what was expected);
- **reproduction** if known (keys/actions in order);
- **candidate identity** — the version directory name (`C:\Program Files\Fcitx5 Fusion\<version>`) or the package name;
- **logs**: `%LOCALAPPDATA%\fusion-host\` (if diagnostics were enabled) and `%LOCALAPPDATA%\fusion-tsf\` if present.

No daily report is required, and no feedback is needed for a day without problems. Resource observations stay incidental
(Task Manager's Host memory/handle count if something looks wrong); there is no mandatory benchmark or monitoring.

## 7. Known limitations at this scope

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
  - **Remaining limit (R46 — cause now measured, fix in progress):** in web content there are two moments when neither
    the composition range nor the caret has layout yet — the **first keystroke of a composition**, and the **first
    keystroke after you pick a candidate mid-word** — and at those moments the popup disappears for the rest of that
    word until you type one more key, which brings it back. It is not only a one-key gap: measured 2026-10-28 on the
    installed `+51467cf`, it happened 151 times in one browser process's log. The fix keeps the popup at the last
    position that did resolve and re-places it as soon as the page lays out — the same PR that owns the popup to the
    document's window. Meanwhile: one more keystroke brings the window back.
  - **If a field ever shows no popup again:** restart the browser first (see the next bullet) and then, if you want to
    report it, run the optional capture in §5 "R25 browser capture" — it gives the exact steps, the log location and
    what to report.
- **After installing a new package, close and reopen the applications you want to test.** A process keeps the
  `FusionTsf.dll` it loaded at startup, so an already-running browser or editor can still show the previous build's
  behaviour — and only that build's diagnostics — until it is restarted. This is the same loader fact recorded as R28.
  Measured on 2026-10-20: a browser started three days earlier still held a development build, and the Edge process used
  for that day's report held one built from source that has no candidate-window code at all (`stage3c1`, base commit
  `5794a36`), so its "no popup in the address bar either" result described that old build rather than the installed
  package.
- **The taskbar search box: 双拼 activates and composes there on `0.0.0-winui3+51467cf`, but the candidate window is
  not visible (R49).** The box's text field belongs to `SearchHost.exe`, which runs in an AppContainer, and Windows
  offers a text service to such a process only when the service declares that it runs in one. Measured on 2026-10-26:
  declaring the capability made the box activate Fusion, and two barriers then stood in order — the Host's pipe refused
  that process with `ERROR_ACCESS_DENIED`, and with a grant in place the identity check failed because it opens the
  Host *process*, which an AppContainer may not. **Both are fixed in fusion-windows PR #55** (D089): the data pipe
  names that one package with client-only rights — no right to create a second pipe instance, no administrative
  right — and an AppContainer client verifies the pipe object (its session, its owner, and that it is not a
  low-integrity object) instead of the Host process. The claim and stop events stay user-only, and no other
  AppContainer or LPAC application is supported.
  **The owner ran the checklist on the installed build on 2026-10-28 (the 5700X,
  `fusion-0.0.0-winui3+51467cf-setup.exe`, 105 279 868 B, SHA-256
  `E94DE186A041F4C32EA152D4EED25433442CCCF9A1DF48C5D0DB46BD4871484A`, unsigned, so Windows may show SmartScreen
  first):** the box activates 双拼, shows an underlined preedit, and Space commits the first candidate into it — the
  box's own log records the link succeeding and the anchor resolving. **What does not work is the candidate window:
  nothing appears**, so a search whose answer is not the first candidate cannot be finished in the box yet. The log
  says the window was created and placed, so the popup exists where the flyout is drawn and the owner sees only the
  flyout: Fusion's popup has
  no owner window, is not topmost, and every raise uses `HWND_TOP`. **The owner's decision of 2026-10-28 fixes it** —
  the popup is owned by the document's own window and kept topmost, as Microsoft's SampleIME and CorvusSKK do —
  implemented in fusion-windows `fix/candidate-owner-layout`, to be recorded as D090. Until that build is installed and
  the box checked on it, use the box only where the first candidate is the one you want, or type in an application
  window where the popup is visible. Win+R, the editors and the browsers are unaffected either way.
  **One session is reported where the box went quiet while typing.** No cause is established for it, and the log named
  with that report belongs to a non-AppContainer process rather than to `SearchHost`; the R49 row of
  `docs/windows-stage6-plan.md` keeps the measurements.
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
- **WeChat: Backspace, Delete and the arrow keys stopped working after Chinese text was committed** (R37, owner report
  2026-10-22). They neither deleted nor moved the caret while Fusion was the Chinese method, and worked again after
  `Win+Space` to another input method or `Ctrl+Space` to English. What is confirmed is the Fusion side: with nothing
  composing, the TIP claimed those keys although the engine only hands them back, and a claim made at that point cannot
  be taken back later in the same key press — which is what an application deciding on that first answer would drop.
  What WeChat itself does was **not** captured, so treat the fix as implemented, not as a proven cure. fusion-windows
  PR #34 claims those keys only while they really are engine work (a composition, a candidate page, unfinished work) and
  keeps them engine-owned while composing; every other key keeps the admission it had before, including the function
  keys you may have bound as Fcitx global hotkeys — the first version of the fix would have blocked those.
  **Owner-confirmed on 2026-10-22:** after installing the package built from that change, the owner reports the WeChat
  editing keys work again. That confirmation covers these keys in WeChat; it is not a claim about every key in every
  application, and it does not turn the WeChat-side mechanism into a measured fact — it was never captured. If a key is
  ever lost again, take the ordinary per-process capture from §5 and read its log: a key the TIP still claims is followed
  by a `CLASSIFY` line, while a key left to the application shows `TEST_KEY` alone.
- A new application window should start with Fusion open (U6, owner decision 2026-10-19; implemented in fusion-windows
  PR #20 and **awaiting its first real-window confirmation**). Check with Fusion already selected (see §1). If a new
  window still starts closed, press Ctrl+Space once and report it with the time and the application.
- **The tray reports a process, not a health state** (U7). `Fusion: Host running (pid N)` means this user's Host process
  exists in this session; it does not mean the Host is responsive, and the tray does not detect an alive-but-unresponsive
  Host. It also does not restart anything by itself: **Restart FusionHost** is always your action. Its first installed
  run, the fault balloon and the install/upgrade/uninstall handling are described in §5 "The Fusion tray", including what
  has not been observed yet.
- **Fusion Settings changes less than the schema shows.** 全局设置 renders Fcitx's whole global schema, but the hotkey,
  input-status/panel and password-field categories have no effect on this product (Fusion never gives a
  `Ctrl`/`Alt`/`Win` combination to the engine, and a lone modifier only as the 临时切换输入法 binding below, `Ctrl+Space`
  is Windows input switching, there is no Fcitx panel, and the
  Windows input-scope policy applies first) — the page says so in one sentence. `Shift`+`Return` commits raw input only
  because the Pinyin `CommitRawInput` option lists it. **A lone `Shift` is the "temporarily switch input method" key.** That
  is the official setting "系统快捷键 → 临时切换输入法" (`Hotkey/AltTriggerKeys`, `Shift_L` unless you change it): tap it
  once and the keys go to the application — English, nothing composed — and tap it again and you are back in the input
  method you were using, Shuangpin or Pinyin, without changing any scheme. It is not a way to rotate between schemes and
  not the Windows input-method switch (`Win+Space`, or `Ctrl+Space` where Windows does it); those stay as they were.
  `Shift`+key combinations are ordinary text input and never toggle, a right `Shift` only does something if you bind the
  setting to it, and holding `Shift` down does not repeat the switch. **Implemented and verified against the engine in isolation,
  not yet verified in a real application; owner acceptance is pending** (plan R38): install the current package (`+ff3bd8f`),
  click into a text field and tap the key the settings page shows. Whether a window is in the switched-off (English) state is
  judged by **that window's own input context**: a background window that is switched off passes its typed characters straight
  to the application even while another window is still composing in Chinese, and it is not read off whichever window most
  recently had focus. In the English state your other configured keys keep working: a key you bound
  to 切换输入法 (for example `F6`) still reaches Fcitx and turns Chinese back on — the state only stops *text* from being
  composed. A lone `Shift` does not commit raw input either, and
  numpad Enter behaves as ordinary `Return` because Windows delivers it as the same virtual key. The list manages the two
  methods this runtime defines (pinyin, shuangpin); dictionaries, addons and `External` tools are not included.
  **Known behaviour:** every Shift key-down is consumed by the TSF service so the engine can see a lone press; a Shift release
  that is not admissible (because document work is outstanding) is not claimed by the TIP — it is passed to the application and
  never reaches the engine, so the lone-Shift toggle does not fire and needs a second press.
- The TIP is a development build: its description and identifiers are the development ones (R29); the installer is
  **unsigned**, and signing and a distribution channel are not decided (U4).
- An alive-but-unhealthy Host (2 s request bound, R7) is not restarted automatically and the tray does not detect it; use
  the restart shortcut or the tray's **Restart FusionHost**.
- Multi-account machines: the logon task name is global (R32, hypothesis); this guide covers one interactive account.

**No zero-loss, zero-corruption or automatic-recovery guarantee is made.** The mechanisms give explicit terminals for the
scenarios in `docs/windows-stage6-plan.md` §8.2; a problem is assessed and repaired by impact under the daily-use policy.

## 8. Acceptance

WP5 and WP6 are accepted through this daily-use period: the owner reports a problem when one occurs, it is repaired by
impact, and **closing the period without reported problems is the acceptance** (owner decision 2026-10-19, D084). No
separate fault-injection or application-matrix campaign is required. Daily use is **not** complete until the owner says so;
unexecuted scenarios are never recorded as PASS.
