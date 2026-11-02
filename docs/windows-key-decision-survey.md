# How Windows IMEs decide "is this key mine?" — survey for Fusion R53

Date: 2026-10-30. Read by the reviewer at the commits listed (shallow clones of each default
branch).

> Committed here on 2026-11-01 with the R53 measurement. Two of the claims below were re-read
> against the sources in this round and are confirmed; the note under each one says what was
> read. Everything else is the reviewer's text, unchanged, including the commit ids. The
> measurement the last section asks for now exists: `docs/windows-key-matrix.md` (the full
> matrix) and `docs/windows-r53-pairing-and-soak.md` (the T/K pairing counters and the soak).

## The question

TSF asks a text service twice per key: `OnTestKeyDown` (T: "will you eat this?") and
`OnKeyDown` (K: "process it"). Some hosts (Explorer rename, Notepad++, Notepad--,
Chrome/Electron, WeChat) honour T alone: if T says TRUE the key never reaches the application
natively, whatever K later says. So the rule every working IME follows is: **never answer TRUE
in T for a key that K will then hand back** ("eat then spit" — WindInput names it explicitly).

Fusion's family of key-loss bugs (R36 Return, R37 editing keys in WeChat, R44 digits, R47
symbols, R53 F-keys, and the 2026-10-30 Notepad++ English-mode stall) all break that rule. T is
answered by a local key-class table (`routes_key` + `engine_has_work_for`,
src/tsf/semantic.cpp) that does not know Fcitx's state or bindings. Each bug was fixed by adding
one more key class to the table.

## What each project does

| Project (commit) | Engine location | How T is answered |
|---|---|---|
| rime/weasel (82b50c9) | out-of-process server (librime) | **Asks the engine in T**: `_ProcessKeyEvent` sends the key to the server, which really processes it. The answer is cached (`_fTestKeyDownPending`) and K returns it without processing again. A repeated T for the same key returns the cached TRUE. |
| google/mozc (d766d25) | out-of-process server | **Asks the engine in T, side-effect free**: `TEST_SEND_KEY` / `TestSendKeyWithContext` — the server evaluates the key without changing state. K then sends the real key. |
| EasyIME/PIME + libIME2 (3697f88 / 287c24b) | out-of-process server (Python/Go/Node backends) | **Asks the engine in T**: `filterKeyDown` RPC; each backend answers from its own full state. K calls `onKeyDown` RPC inside a sync edit session. |
| gaboolic/moqi-im-windows (13af5bc) + moqi-ime | out-of-process (Go) on libIME2 | Same as PIME (`filterKeyDown` RPC in T). Its fcitx5 backend is still a mock (`filterKeyDown` = `onKeyDown`, simulated candidates). |
| tsaanghwang/Yime (5f98f28) | libIME2 / PIME launcher | Same family as PIME. |
| chewing/windows-chewing-tsf (dc6d82f) | host process (`chewing_tip_host`, named pipe) | **Rule in the engine host with the engine's state**: keyboard closed → no; English + half-width → no; Ctrl/Alt chords → no (except composing digits); not composing and key not printable → no; configured keybindings → yes. |
| huanfeng/WindInput (765292c) | out-of-process server | **Local rule fed by server-pushed state**: hotkey whitelist, Chinese/English mode, composing, candidates, password and InputScope masks synced from the server. Comment: T=TRUE then K=FALSE "吃了再吐" makes Chrome/Electron drop the key. |
| qingjian-team/qingjian (6935ab9) | out-of-process server | **Hybrid**: a local `would_eat` from composing/translating state and settings. For the one ambiguous class (digits with nothing composing) it **asks the server in T** and caches the outcome for K (`pending_digit`). |
| metasequoiaime/msime-windows (e255232) | in-proc composition logic + out-of-process server | **Local rule** (`_IsKeyEaten`, SampleIME lineage) over its own engine state, plus a deferred-key queue behind a barrier. |
| jrywu/DIME (2788155), shenmin/cassotis-ime (6104654) | in-process engine | **Local rule** over the in-process engine's own state (`_IsKeyEaten` / `on_test_key_down_core`). |

## Patterns

1. **Ask the engine in T** (Weasel, Mozc, PIME family, Qingjian for digits). The decision is the
   engine's, so it cannot drift from what K will do. Weasel processes for real and caches; Mozc
   uses an engine-side test mode.
2. **Local rule, exact because it owns the engine** (chewing, WindInput, msime, DIME, cassotis).
   The rule is written by the people who wrote the engine and is fed the engine's live state
   (mode, composing, bindings).
3. Everyone answers FALSE in T when the keyboard is closed or disabled, in English/ASCII mode
   with nothing to do, and for a non-printable key with nothing composing.

Fusion is in neither pattern: its engine is foreign (Fcitx, with addons and user-configurable
hotkeys) and its rule is local and stateless. That is why the table can never be complete.

## The two Weasel failure modes the owner reported

The owner reports, from long use of Weasel, slowdowns after long uptime and lost words in Word.
The reviewer traced two likely causes in Weasel's source.

**1. Blocking pipe I/O on the application's UI thread with no deadline** —
`WeaselIPC/PipeChannel.cpp`.

> **Checked here, 2026-11-01, against `rime/weasel@82b50c9`.** `PipeChannelBase::_WritePipe` is
> `::WriteFile(...)` followed by `::FlushFileBuffers(pipe)` and `_Receive` is a plain
> `::ReadFile(pipe, msg, rec_len, &lread, NULL)` — a synchronous read on a `PIPE_WAIT` handle,
> with no `OVERLAPPED` and no timeout anywhere in the file. `_Connect` is
> `while (_Invalid(pipe = _TryConnect())) ::WaitNamedPipe(name, 500);`, an unbounded retry loop
> that only ends when a server appears. There is no deadline to expire: a Host that stops
> answering blocks the calling thread for as long as it takes. Confirmed as stated.

**2. Async edit sessions that read the *latest* server response** — `WeaselTSF/EditSession.cpp`.

> **Checked here, 2026-11-01, against `rime/weasel@82b50c9`.** `WeaselTSF::DoEditSession` begins
> with `bool ok = m_client.GetResponseData(std::ref(parser));` — the session asks the client for
> the response *at the moment the session runs*, not for the response to the key that requested
> it. With an asynchronously granted session that is a different key's answer whenever the user
> has typed again in between, which is the word-loss the owner sees in Word. The flag itself
> (`TF_ES_ASYNCDONTCARE`) is passed at the request site, which is `WeaselTSF::_RequestEditSession`
> in `WeaselTSF.cpp`, not in this file; the read-back shape the reviewer described is confirmed
> as stated.

## Options for Fusion

**A. Weasel model — process in T, cache for K.** T sends the key to the Host, Fcitx processes it
for real, and the TIP caches the batch; K applies the cached batch inside its edit session. The
answer is Fcitx's own (faithful, D082). To settle: T called more than once for one key (Word); T
answered TRUE but K never arrives (engine moved, document did not), which needs a reconcile;
latency (the same single round trip, moved from K to T); interaction with the D072
ownership/effect machinery; key-up pairs (Shift toggle).

**B. Mozc model — side-effect-free test.** Needs a dry-run in Fcitx, which upstream does not
offer. Not viable without patching Fcitx, which D082 rules out.

**C. Chewing/WindInput model — local rule fed by Host-pushed state.** The Host pushes
active/inactive, composing, page-present and Fcitx's configured hotkeys (trigger, alt-trigger,
enumerate, punctuation toggles, quick-phrase trigger…). It is cheaper, but it is still a
prediction of a foreign engine and its addons, so it can drift again.

Reviewer's leaning: **A**, measured first.

## Measured facts to collect before deciding (key-matrix tool)

- For every key × state × modifier: today's T answer, Fcitx's real answer, and the mismatches
  (the bug list).
- Under option A: how often hosts call T twice, or T without K, per application (Notepad,
  Notepad++, Explorer, Edge, Firefox, WeChat, Word, VS Code, search box); the T latency
  distribution; and whether any host breaks.

> **Status, 2026-11-01.** The first item is done: `tests/key_matrix.cpp` measures 4 824 cells and
> the result is in `docs/windows-key-matrix.md`. The second item is only half done: the T/K
> pairing counters are implemented and unit-tested (`tests/key_pairing.cpp`,
> `src/tsf/key_pairing.cpp`) and write one content-free `KEY_PAIRING` line per 200 keys and on
> deactivate, but the per-application numbers need a build installed with
> `FUSION_TSF_DIAGNOSTICS=1` and one log per application — that is the owner step in the PR, not
> something this machine can produce.
