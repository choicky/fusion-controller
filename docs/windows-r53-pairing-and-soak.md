# R53 measurement — T/K pairing, the two Weasel failure modes, and the soak

Three measurements that the R53 key decision needs and that no earlier round produced:

1. the content-free T/K pairing counters that ship in the TIP;
2. Weasel's two failure modes — a shared "latest response" and a blocking call with no deadline —
   checked against Fusion's own source and, for the first one, against the real engine;
3. a long-run soak of the product Host.

The key matrix itself is separate: `docs/windows-key-matrix.md` and `.csv`. The survey that frames
the decision is `docs/windows-key-decision-survey.md`.

Everything here is measurement. **Nothing in this round changes what T or K answers.**

---

## 1. The T/K pairing counters

`src/tsf/key_pairing.{hpp,cpp}` is a pure state machine — no TSF, no Windows types — and
`tests/key_pairing.cpp` unit-tests it. `src/tsf/service.cpp` feeds it from the four key callbacks
and writes one summary line. It is behind the existing `FUSION_TSF_DIAGNOSTICS` opt-in.

### What is on each line

| event | fields |
|---|---|
| `TEST_KEY` | `mod=<position>` for a modifier, then `down`, `eligible=0\|1`, `seq=N`, `sent=0`, `result=none`; for every other key `seq=N`, `eligible=0\|1`, `sent=0`, `result=none` |
| `KEY_ARRIVAL` | `mod=<position>` or nothing, `ctx=<pointer>`, `cid=N`, `seq=N` |
| `TEST_KEY_UP` | `mod=<position>` or nothing, `up`, `eligible=0\|1`, `seq=N`, `sent=0`, `result=none` |
| `BACKEND` (key-up dispatch) | `mod=<position>` or nothing, `up`, `sent=0\|1`, `result=`, `route=`, `ops=`, `seq=N` |
| `KEY_PAIRING` | `t=N t_true=N k=N t_true_without_k=N t_repeated=N k_without_t=N async_effects=N` |

`seq` is one per-thread counter; T and K for the same physical key share it, and a new key starts
the next one. A key is identified for counting purposes by its scan code and the message time of
the event (`GetMessageTime()`), never by its virtual key.

### What each counter means

| field | meaning |
|---|---|
| `t` | T calls on this thread |
| `t_true` | … that answered TRUE |
| `k` | K calls |
| `t_true_without_k` | T answered TRUE and no K followed before the next key — the claim the host did not follow through, which is the one shape the Weasel model cannot tolerate |
| `t_repeated` | a second T for the same physical key (same scan code and message time) |
| `k_without_t` | K arrived with no T for that key |
| `async_effects` | document writes that ran in an asynchronously granted edit session |

The line is written when `k` reaches each multiple of 200 and once more from `Deactivate()`, which
also closes an open pair first (`flush()`), so the last key of a session is not lost. Counters are
cumulative: consecutive lines differ by one window.

### The privacy property, and the one place the old shape changed

The log must never name a key that could carry content. Until this round the non-modifier lines
carried `vk=0x…`, which `diag::event` reduces to `privacy=redacted` — so those lines carried
*nothing at all*, not even the eligibility answer. The line now carries `seq` and `eligible` and
**no key identity**: dropping `vk=` is what lets the line be written verbatim, and the pairing is
done by `seq`. Modifier lines keep the position name (`lshift`, `rctrl`, …), which is not content.

`tests/diag_policy.cpp` is extended to assert all of it: the ordinary-key line and the
`KEY_PAIRING` line are written verbatim, a line carrying `vk=` is still reduced to
`privacy=redacted`, and no line in the file contains `vk=` or `text=`.

---

## 2. Weasel's failure mode 1 — a shared "latest response" (the Word word-loss)

### The shape Fusion must not have

Weasel's `WeaselTSF::DoEditSession` asks the client for the response *when the session runs*
(`m_client.GetResponseData(parser)`), so a session granted asynchronously can apply a later key's
answer to an earlier key's commit. Fusion has no such read: **a commit is bound to the record of
the key that produced it, at dispatch time, and the text travels with that record.**

### The trace

| step | where | what it establishes |
|---|---|---|
| the batch is validated and each operation is admitted into its own Core record | `src/tsf/semantic.cpp:658-670` | one record per operation, owned at dispatch time; the operation is copied into `decision.frontend` with the key it was admitted under |
| the effect's state dependency is captured at that moment | `src/tsf/semantic.cpp:674` | `propose_effect(key, state_dependency_)` binds the commit to the STATE watermark that preceded it, not to "whatever is current later" |
| the text of a STATE is the batch's own projection | `src/tsf/semantic.cpp:677` | `buffer_ = operation.text` — the shell's projection, never re-read from the engine |
| the decision's frontend list is immutable afterwards | `SemanticDecision::frontend` (`src/tsf/semantic.hpp:56`) | the callback list is carried in the decision value, not in a shared buffer |
| the TSF side copies each callback and its text into the ordered queue | `src/tsf/service.cpp:2333-2335` | `enqueue_effect(work.key, work.operation.text)` / `enqueue_state(...)`: the queue entry carries the key *and* the text |
| the queue is drained front to back, stopping at the first refusal | `src/tsf/service.cpp:2290-2316` | `apply_frontend` reads `entry.semantic.pending().front()` each time and settles it through `open_effect`/`settle_front_state` |
| an effect cannot even be opened out of order | `src/core.cpp:29` (`prior_pending`), `:96` (`begin_effect`) | a ticket is refused while any earlier record in the same binding is pending or uncertain |
| the async session uses the same door | `src/tsf/service.cpp:513` (`AsyncPumpSession::DoEditSession` → `apply_frontend`) | an asynchronously granted session applies the queue, not a response |

There is no per-thread "last batch" anywhere on this path: the only thing that outlives a
dispatch is the ordered queue, and every entry in it is immutable.

### The test

`tests/stage3c.cpp`, the section *"Word burst: deferred commits are bound to their key and written
in order"*:

- three words are typed and committed through the real engine (`nihao`, `zhongwen`, `shijie`);
- **every** frontend callback is deferred into the queue exactly as `settle_frontend_work` defers
  it, and nothing is settled;
- four further keys are then typed, so the engine's latest batch is a *different* one — the batch a
  "latest response" implementation would read back;
- the queue is checked first: every queued commit still carries the text its own key produced;
- then the deferred session runs, draining in order, and the document must equal the three commits
  concatenated.

Result: `deferred queue: 30 entries, 3 commits`, and `deferred document == expected`. Nothing was
overwritten, duplicated or reordered.

**What this test does not do, and why.** The instruction asked for a fixture text store that grants
only asynchronous edit sessions. This repository has no TSF text-store fixture at all — the TSF
service is not unit-tested against a fake `ITfContext` anywhere — and building one is a
multi-thousand-line harness, not a test-fixture change. The test therefore exercises the queue at
the layer where the invariant lives (`SemanticController` + `Core` + the real engine), with the
settlement deferred exactly as the async session defers it. The remaining gap is the TSF glue
between `RequestEditSession` and `apply_frontend`, which is 30 lines and reads only
`pending().front()`.

---

## 3. Weasel's failure mode 2 — a blocking call with no deadline

Every call the TIP makes to the Host **on the application's thread**, with its deadline and what
happens when it expires. There is no call without a deadline and no unbounded loop on this path.

| call | site | deadline | on expiry |
|---|---|---|---|
| `IpcBackend::ensure_link` | `src/ipc/ipc_backend.cpp:79` | `200 ms` (`kLinkTimeout`, `src/tsf/semantic.cpp:18`) for the transport connect **and** the same again for the Hello round trip | returns false; `begin_real_key` answers `FailOpen`, the key stays the application's |
| `IpcBackend::send` | `src/ipc/ipc_backend.cpp:181` | write is overlapped and bounded; the response wait is `kRequestTimeout = 2000 ms` (`src/ipc/pipe_client.hpp:17`) | `NotWritten` → `Forward` (caller-owned); `Uncertain` → `Reject`; no response → `Reject` and `drop_link()`, which the controller turns into the cancelled-recovery terminal |
| `IpcBackend::bind` / `reset` / `unbind` | `src/ipc/ipc_backend.cpp:115-119` | same `kRequestTimeout` | the call reports failure; `ensure_semantic_ready` leaves readiness false and the next key retries the bounded connect |
| `link_status` / `healthy` / `session_id` | `src/ipc/ipc_backend.hpp:77-79` | none needed — local atomics, no I/O | — |
| settings calls | not on the key path (separate connection, `src/host/host_server.cpp:389`) | `5000 ms` on the settings connection | the settings window reports an error; the key path is untouched |
| `FcitxBackend::invoke` (in-process, tests and the Host's own backend) | `src/fcitx/backend.cpp:2234` | `request_wait = 2000 ms` (`src/fcitx/backend.cpp:80`) | sets `healthy_ = false`, which is a permanent fence for that backend session |

Two observations worth recording rather than fixing:

- **The bound is real but large.** A key that finds the pipe present but unserved can spend
  `200 + 200 + 2000 ≈ 2.4 s` on the application's thread before failing open. Weasel's problem is
  that there is *no* bound; Fusion's is that the bound is generous. That is a tuning question for
  the redesign, not a correctness one.
- **No retry loop exists on the TIP side.** `ensure_link` is one attempt per call; the only loop in
  the transport is `PipeClient::connect`'s wait for `ERROR_PIPE_BUSY`, which is bounded by the same
  timeout argument. Weasel's `_Connect` has `while (…) ::WaitNamedPipe(name, 500);` with no exit.

---

## 4. The soak

### 4.1 The isolation that failed, and how it was contained

**This harness wrote into the owner's real Fcitx data on its first two runs, despite the hard
constraint. This section records it rather than hiding it, because the mechanism is a property of
the runtime that every later harness needs to know.**

What was done. The first version of `tests/soak.cpp` set `FCITX_CONFIG_HOME` and `FCITX_DATA_HOME`
to a throwaway directory under the task workspace and left `FUSION_FCITX_SKIP_USER_PATH` unset, so
that the engine would really persist — the point being to measure the autosave path.

What actually happened. This runtime does not honour either variable on Windows. Both variables
exist as strings in `Fcitx5Utils.dll`, but the Windows build resolves its per-user directories
itself, and a run with both set still wrote into `%APPDATA%\Fcitx5`. Two one-minute smoke runs
(2026-11-02, 10:56–11:01) therefore wrote to the owner's profile.

How it was detected. The harness scans the profile directory it *believes* it is using and reports
the file count per window; that count stayed **0** while `%APPDATA%\Fcitx5` showed a file written
during the same minute. Reading `pinyin/mixed_english_user.tsv` then showed **18 entries with
timestamps 10:57:17–10:58:12 whose words are the harness's own random letters** (`aa`, `hig`,
`hin`, `ih`, `inn`, `inns`, `ino`, `io`, `wig`, `wii`, `wie`, `ova`, `na`, `ok`, `joe`, `gio`, `ie`,
`seo`) — none of them anything the owner had typed. The owner's own twelve entries
(`codex`, `firefox`, `etf`, `dns`, `cd`, `im`, `tm`, `vix`, `wez`, `windows`, `wok`, `word`) are
untouched and were not modified.

What was also rewritten in that window, and what cannot be told. `pinyin/user.dict`,
`pinyin/user.history`, `config/fcitx5/conf/pinyin.conf`, `config/fcitx5/conf/punctuation.conf` and
`config/fcitx5/profile` were rewritten at 10:56:52 in the same window. `user.dict` and
`user.history` are binary, and no copy of them from before the run exists, so **whether their
content changed cannot be determined from the files**; the sizes at the next save (13:26:51) are
identical to the ones observed at 10:56:52, which is consistent with a re-save of unchanged
content but is not proof of it. The only content change that can be shown is the 18 entries above.

How it was contained. `FUSION_FCITX_SKIP_USER_PATH=1` is now set unconditionally by the harness —
the backend then sets `SKIP_FCITX_USER_PATH` inside the Host process itself, so the engine has no
user path at all. Before the three-hour run, a one-minute control run was compared against the real
profile file by file on last-write time: **unchanged**. Every later window of the three-hour run was
checked the same way, and `%APPDATA%\Fcitx5` shows no write between 11:00:16 and the production
Host's own save at 13:26:51.

The cost. With no user path the engine persists nothing, so **the autosave / user-data-write series
this round was meant to record cannot be measured here**. The CSV carries the columns
(`profile_files`, `profile_bytes`, `profile_written`, `max_us_near_write`) as zeros rather than
guesses, and this document does not claim a number for it.

**The rule this leaves for every later harness.** A harness that starts the Host must either set
`FUSION_FCITX_SKIP_USER_PATH=1`, or use an isolation it has *proven* by comparing
`%APPDATA%\Fcitx5` before and after a control run. Setting `FCITX_CONFIG_HOME` /
`FCITX_DATA_HOME` is not isolation on this platform. The harness should also refuse to start if it
cannot show the profile directory it scans is the one the Host writes.

A read-only backup of the owner's profile was taken before any further work
(`~/worktmp/fusion-windows/r53-key-matrix/fcitx5-backup-20261102`, 7 files, 20 267 bytes), and the
owner decided the cleanup on 2026-11-02.

What was removed, and the proof. The 18 lines whose **first-seen** timestamp falls in
10:57:17–10:58:12 were deleted from `pinyin/mixed_english_user.tsv` — those words and no others,
matched on the word **and** the timestamp so that nothing the owner typed could be caught by
accident. Nothing else in that file changed: it went from 31 lines (header + 30 entries) to 13
(header + 12), `diff -u` against the backup shows exactly 18 deletions and no addition or
modification, and every surviving line is byte-identical to the backup. The file's SHA-256 was
`1A00857E5596E3BDFBF953041234CDFD16C34558700F50B7CF735512338D058C` before and
`6672927FD0D2552E3AB08D9897A3F48E46A87814C74ABCF3FB32007006F3CD14` after (415 B). The owner's own
twelve entries — `cd`, `codex`, `dns`, `etf`, `firefox`, `im`, `tm`, `vix`, `wez`, `windows`, `wok`,
`word` — are unchanged, and the file still sorts and still starts with `#schema=1`.

No other file was touched. `mixed_english_user.tsv` is the only entry under `%APPDATA%\Fcitx5`
whose last-write time is 18:03:02; the other six carry the production Host's own periodic save at
17:56:51, with exactly the sizes they had at 10:56:52. That save is written from the state the
production Host loaded at **10:26:45**, before the incident, so those files were never exposed to
the harness's writes and need no action.

### 4.2 The run

Three hours, four concurrent client connections, on the release build of this branch, with
`FUSION_FCITX_SKIP_USER_PATH=1`. Each client types a mix of pinyin words committed with Space,
candidate digits, punctuation, English letters and abandoned compositions, at roughly 2.7 keys per
second. Every ten minutes the harness samples the Host's private bytes, handle count and thread
count, the per-key round-trip latency of that window (p50/p99/max), the profile directory, and one
settings read on its own connection.

The CSV is `~/worktmp/fusion-windows/r53-key-matrix/soak3h/soak.csv`; the numbers below are copied
from it.

| minute | private bytes | handles | threads | keys | p50 | p99 | max |
|---|---|---|---|---|---|---|---|
| 10 | 83,013,632 | 227 | 11 | 6,489 | 5 094 µs | 35 669 µs | 330 861 µs |
| 20 | 78,798,848 | 229 | 10 | 12,970 | 5 252 µs | 36 209 µs | 56 658 µs |
| 30 | 78,278,656 | 229 | 9 | 19,475 | 5 018 µs | 29 955 µs | 52 953 µs |
| 40 | 86,736,896 | 229 | 10 | 25,985 | 4 555 µs | 29 446 µs | 54 736 µs |
| 50 | 89,190,400 | 229 | 9 | 32,482 | 5 276 µs | 33 536 µs | 57 770 µs |
| 60 | 91,078,656 | 229 | 9 | 38,947 | 5 957 µs | 40 865 µs | 99 662 µs |
| 70 | 89,075,712 | 229 | 10 | 45,437 | 5 320 µs | 35 260 µs | 72 039 µs |
| 80 | 85,557,248 | 229 | 9 | 51,944 | 5 040 µs | 29 806 µs | 51 669 µs |
| 90 | 87,941,120 | 229 | 9 | 58,440 | 5 143 µs | 36 791 µs | 62 885 µs |
| 100 | 87,179,264 | 229 | 9 | 64,913 | 6 253 µs | 40 990 µs | 69 686 µs |
| 110 | 91,353,088 | 228 | 9 | 71,388 | 5 723 µs | 40 930 µs | 68 990 µs |
| 120 | 87,584,768 | 229 | 9 | 77,872 | 5 705 µs | 38 443 µs | 66 946 µs |
| 130 | 92,983,296 | 229 | 9 | 84,365 | 5 151 µs | 35 822 µs | 60 102 µs |
| 140 | 92,930,048 | 229 | 9 | 90,844 | 5 918 µs | 37 534 µs | 64 759 µs |
| 150 | 90,312,704 | 229 | 9 | 97,324 | 5 720 µs | 38 382 µs | 72 297 µs |
| 160 | 88,379,392 | 229 | 9 | 103,788 | 6 133 µs | 43 778 µs | 65 795 µs |
| 170 | 85,245,952 | 229 | 9 | 110,279 | 5 598 µs | 38 286 µs | 69 043 µs |
| 180 | 87,871,488 | 229 | 9 | 116,759 | 5 255 µs | 37 594 µs | 80 571 µs |

**What the series says.**

- **Memory does not grow.** Private bytes moved between 78.3 MB and 93.0 MB for three hours and
  ended at 87.9 MB, between the two extremes and 4.9 MB above the first window. The movement is
  oscillation, not a trend: it rises for three windows and falls for three. There is no monotonic
  slope to extrapolate, which is what a leak would show over 18 samples.
- **Handles and threads are flat.** Handles sat at 229 for seventeen of eighteen windows (227 in
  the first, 228 in the eleventh — both one-off), and threads went 11 → 10 → 9 and stayed at 9 for
  the last eleven windows. Nothing accumulates per key over 116 759 keys.
- **Latency is flat, and the level is the finding.** p50 stayed in 4.6–6.3 ms (median of the
  medians 5.3 ms) and p99 in 29–44 ms (median 37 ms) with no upward drift — window 18 looks like
  window 3. The absolute numbers are worth recording: **a key costs about 5 ms at the median and
  37 ms at the 99th percentile**, which is the pipe round trip plus the engine's own event-loop
  hand-off, with four connections sharing one Host. No key in any window came near the 2 s request
  timeout, so no dispatch took the fail-open or recovery path by timeout.
- **The settings read worked in every window** (18 of 18), so the separate settings connection
  stayed serviceable throughout.
- **The write columns are zero because of §4.1**, not because nothing was written: the engine had
  no user path. `%APPDATA%\Fcitx5` was checked afterwards: the only writes between 11:00 and the
  end of the run are the production Host's own saves at 13:26:51 and 13:56:51, both with the same
  sizes as its 10:56:52 save, and my Host wrote nothing anywhere.

**What it does not say.** One Host, one machine, one configuration, four synthetic connections. It
is evidence about this build under this load for three hours; it is not a general claim about
uptime. The one measurement it was meant to make and could not — how long an autosave blocks a key —
is unmeasured, and §4.1 says why.

### 4.3 R48

R48 records an Explorer stall whose cause was never determined, and it stayed open partly because
"slowdowns after long uptime" had no measurement behind it. This run is that measurement for the
Host half: **three hours, 116 759 keys, no growth in memory, handles or threads, and no drift in
p50 or p99.** It does not explain the Explorer stall — the stall is in the TIP inside Explorer, not
in the Host, and this run did not have a TIP attached to an application — but it removes the
"the Host degrades with uptime" hypothesis from the list, which is the part of R48 the Host can
answer.
