# Windows Stage 6 — Plan and Acceptance Gates (PROPOSED)

> **Status: PLAN PROPOSED — NOT accepted.** Current state (2026-10-18): WP1 is merged and closed within its accepted scope (fusion-windows PR #13, controller PR #32; R1–R3 in §4, §9). WP2 implementation is merged (fusion-windows PR #14 → `main` `559d9ce`, controller PR #33), and its final-candidate technical checks are complete on `559d9ce`: Notepad A–E PASS; the Edge and Firefox password checks PASS for the U5 minimum outcome, with both blocked by the existing keyboard-disabled hard-policy gate before admission; the InputScope probe read Unknown, so Sensitive classification is not demonstrated (`docs/windows-stage6-wp2.md`). U3 (a)/U5 (a)/U1 are owner-accepted (2026-10-18; §7.1); WP2 is closed within its scope, and WP3 is closed within its scope after the owner's walkthrough on `runtime-wp3-20261018` (B3a–B3g and Notepad A–E PASS, `docs/windows-stage6-wp3.md`). The Firefox diagnosis confirmed R24 (fixed by fusion-windows PR #15, merged as `main` `0cd0374`) and recorded R25. PR #15 changed only `SemanticController::routes_key`: it removed the semantic-readiness and backend-health (`healthy()`) terms and left key-class eligibility unchanged. The T-path decision that produced the Firefox password-field result (`mapped && hard_policy_allows(...)` in `OnTestKeyDown`) and the K path are unchanged, and Notepad and Edge did not deliver T in the recorded runs. The U5 minimum-outcome and Notepad A–E conclusions therefore apply to `0cd0374` by source analysis; they were not re-run on `0cd0374`. U2 (b) is owner-accepted (2026-10-18; §7.1) and WP4 is implemented and locally verified on `feature/stage6-wp4-packaging` (B1 70/70 Mock-only, 78/78 Debug, 78/78 Release; two packages; A1–A6 evidence; `docs/windows-stage6-wp4.md`), **not merged**: its owner walkthrough (B3/B4) is pending, so WP4 is not closed. No later work package is authorized; U4, U6, U7 stay open.
> This is the single canonical Stage 6 planning location. It proposes a target scope, work order and gates; it is
> not an Accepted Decision, does not change `docs/REQUIREMENTS.md`, and does not supersede D072 / D082 / D083 or
> `docs/windows-architecture.md`. Merging this plan authorizes no implementation: every work package, including the
> recommended first one (WP1), starts only on its own explicit, bounded execution instruction after the plan has
> been reviewed. Proposed boundaries need explicit owner acceptance before they become product requirements.
> Stage 5 stays VERIFIED / COMPLETE within its recorded scope (architecture §22/§25) and is not reopened. The Stage 5
> deferred-capability table stays in architecture §25 and is referenced, not duplicated.
>
> Baseline inspected (2026-10-17): fusion-controller `main` `56eae64` (PR #30); fusion-windows `main` `0f724af`
> (tree `ee2f5536…`, identical to the reviewed/smoked `670dfc0`). Source was read read-only; no builds, tests,
> registration changes or experiments were run for this plan.

## 1. Goal

Define the minimum, evidence-driven path from the verified Windows development build (Debug, manually registered,
manually started Host, Pinyin-only, Notepad-verified) to a **reviewable release candidate** of a proposed target
scenario — without promising universal Windows compatibility and without silently resolving P0-A / P0-B / P0-D.

## 2. Verified starting point (state before WP1, 2026-10-17)

| Area | Current fact | Evidence (status) |
|---|---|---|
| Product path | Thin TSF DLL (`FusionTsf.dll`, client-only) → per-user/per-session Named Pipe → `FusionHost.exe` → pinned Fcitx5 + chinese-addons + LibIME | arch §1–§3, §13, §25; Stage 4/5 closure (CONFIRMED, Notepad) |
| Verified apps | Notepad only on the out-of-process product path (Stage 4 + Stage 5). Firefox/Edge were physically verified only in the Stage 3C in-process era; Firefox has one incidental Stage 4B reconnect observation | arch §25 Stage 3C/4 records (CONFIRMED, scoped) |
| Input method | The Host enables only the `pinyin` addon group and forces input method `pinyin` on every context (`backend.cpp`: `setCurrentInputMethod(context, "pinyin", true)`; group built with `pinyin` only). chinese-addons ships Shuangpin as a separate input method (`shuangpin`, same addon) whose scheme is a pinyin configuration option (`ShuangpinProfile`) | `src/fcitx/backend.cpp`; chinese-addons `im/pinyin/shuangpin.conf.in`, `pinyin.h` (CONFIRMED) |
| User data | Host sets `SKIP_FCITX_USER_PATH=1` and `SKIP_FCITX_SYSTEM_PATH=1`: no persisted user configuration, no persisted learning / user dictionary | `src/fcitx/backend.cpp` (CONFIRMED) |
| Build configuration | All Stage 3C–5 product-path evidence is **Debug** (debug CRT). A Release native runtime exists (`runtime-release`) and Stage 3B ran Release tests; Stage 4/5 were never validated in Release | build logs / `dumpbin` (CONFIRMED) |
| Architecture | x64 only | build scripts (CONFIRMED) |
| Registration | `tools/tsf/register.ps1` / `unregister.ps1` = elevated `regsvr32` (HKLM COM + CTF TIP + categories incl. display-attribute provider); `check-state.ps1` read-only inventory; `FusionTsfProfile.exe on/off/query` = development session activation | source (CONFIRMED); arch §14 |
| Host start | **Manual.** The DLL never starts the Host (P0-D, arch §25 Stage 4). With no Host, a key is not owned (fail open **before ownership**) once any unresolved prior EFFECT has been drained or the key cancelled (§8.2 rule 1), and reconnect is armed on the key path | `src/tsf/service.cpp`, arch §13/§25 (CONFIRMED) |
| Failure mechanisms | Implemented and verified at Stage 4 scope: request write certainty (a request that provably wrote no byte may leave the key caller-owned/Forward; an uncertain write or post-write response loss → Reject, no blind replay/retry); disconnect keeps unresolved ownership/uncertainty and the undelivered materialization queue; `probe_pass`/watermark stop later keys overtaking an unresolved EFFECT; Host incarnation + connection-generation fencing of stale results; teardown audit (`CONTEXT_TEARDOWN`) | arch §13, §25 Stage 4A/4B/4C (CONFIRMED, Notepad + deterministic real-process tests) |
| Pipe identity | `FusionHost-v1-<user SID>-<session id>`; DACL grants only the current user SID; client checks server session | `src/ipc/pipe_security.cpp` (CONFIRMED) |
| Runtime layout | Host needs Fcitx/LibIME DLLs, addon libs, `data/`, `lib/libime` beside its output (CMake post-build copy) | `cmake/FcitxBackend.cmake` (CONFIRMED) |
| Request bound | One backend request waiting > 2 s marks the backend session permanently unhealthy until the Host process restarts | `src/fcitx/backend.cpp` (CONFIRMED) |
| Diagnostics | TSF diagnostic logging is **off by default**. Explicit opt-in through `FUSION_TSF_DIAGNOSTICS` records technical lifecycle/ownership data only; key identifiers and key/text-bearing details are redacted before emission. It never records key content, preedit, commit, candidate, clipboard, credential or user-dictionary data. | `src/tsf/diag.cpp`, `service.cpp` (CONFIRMED) |
| Native runtime build | `tools/fcitx/build-native.ps1` (Debug/Release) applies `cmake/patches/fcitx5-msvc.patch` and `chinese-addons-msvc.patch` to pinned sources | source (CONFIRMED) |
| Local tests | Mock-only 64/64; real-Fcitx 70/72 with `stage3c_faithful_frontend`, `fcitx_msvc_empty_range_guards` aborting (`xstring(54)`) | Stage 5 closure (CONFIRMED) |


> The rows above are the state **before WP1** (2026-10-17) and are kept as the plan's starting point. Later stages changed several of them; the current outcomes are in `docs/windows-stage6-wp2.md` (Release configuration, diagnostics, InputScope/U5), `docs/windows-stage6-wp3.md` (input-method selection, per-user user data, learning persistence) and `docs/windows-stage6-wp4.md` (Host start mechanism, windowless Host, package layout, install/upgrade/uninstall).
## 3. Target scope versus temporary test limitations

**Target release-candidate scope (proposed; product capabilities are not optional here):**
- Windows 11 x64, one interactive console session, standard user at Medium IL.
- **Pinyin and Shuangpin** through the pinned Fcitx5 + chinese-addons engine (both are first-class project support),
  with the Stage 5 composition/candidate UI.
- **Fcitx-owned learning and user dictionary, persisted per user with Fcitx's existing mechanisms** (D082: learning
  semantics belong to Fcitx; Windows only provides the storage location and lifecycle).
- **Selected input method/scheme and options persist per user** across Host restarts.
- Representative application matrix: Notepad, Word (Office 16 x64), Firefox and Edge (address bar, web
  `<input>`/`<textarea>`). Availability on the development machine (read-only, 2026-10-17): Notepad 11.2607,
  Word 16.0.17932 x64, Firefox 159.0, Edge 155.0.4283 — other machines must re-check (dependency).

**Temporary limitation of interim diagnostic/test artifacts (WP1–WP2, local developer machine only):** Pinyin only,
no persisted user data. This bounds what those artifacts prove; it is **not** a scope reduction of the target, and
it ends at WP3 (§6).

**Explicit exclusions from the target (proposed):** universal Windows compatibility; 32-bit apps; ARM64;
AppContainer/UWP/LPAC apps; elevated (High-IL) apps; RDP, multiple simultaneous sessions, Fast User Switching;
**automatic** restoration of an in-flight composition after a Host failure (the minimum recovery representation and
responsibility accounting are *not* excluded — §5); input methods or schemes beyond Pinyin/Shuangpin; a broad
settings UI (configuration uses Fcitx's existing configuration files/options); MoQi, Voice and other Android-line
features; all Stage 5 DEFER / NEEDS SEPARATE items (arch §25); exact-caret anchoring unless §7.4 makes it necessary.

## 4. Findings register

Disposition: **REQUIRED** (for the target release candidate), **CONDITIONAL** (only if its trigger fires),
**DEFERRED** (after the target release candidate). WP numbers refer to §6; the mapping from the first revision of
this plan is in §10.

| ID | Finding / gap | Source & evidence status | Correctness / remaining hypothesis | Disposition | Owner · trigger · dependencies | Validation that could change the decision | Durable authority |
|---|---|---|---|---|---|---|---|
| R1 | The two baseline aborts (`stage3c_faithful_frontend`, `fcitx_msvc_empty_range_guards`) | Before WP1: test logs (abort, base = HEAD); installed header without the guard; timestamps only. **WP1 (observed):** the old source checkout matches the older patch state `d476f29` (whitespace-insensitive reverse-apply), not `127912b`; both old installed `utf8.h` (Debug and Release, identical) equal that checkout and lack guard A (`lengthValidated` empty range) and guard B (`UTF8StringViewIter::update`); the old Debug runtime still aborts both tests. Runtime rebuilt from fresh pinned clones (`a7f2528` / `61474bd` / `171edcf`) with the current patches: both guards present in sources and installed headers; `stage3c_faithful_frontend` passes; guard sections A/B pass | **Cause attributed to the missing guards:** supported by the old checkout/header state and the observed old-abort / rebuilt-pass difference. Complete build provenance of the old binaries remains **UNKNOWN** (their exact construction is not proven). The test's section C then failed on a stale pre-Stage-5 oracle (Forward batches now carry the empty `CandidateSnapshot`), corrected in fusion-windows PR #13 (exactly one empty-panel snapshot) | **RESOLVED on the rebuilt runtime** `runtime-wp1-20261017` (observed; Mock-only 64/64, real-Fcitx Debug 72/72 and Release 72/72 with PR #13) | Windows pinned-runtime compatibility follow-up · WP1 execution instruction · `build-native.ps1`, pinned sources | Done (WP1 evidence package); reopen only if a later runtime rebuild reintroduces an abort | this plan + arch §25 runtime bullet |
| R2 | Punctuation (`handlePunc` empty `puncAfter`, guard A) and the QuickPhrase trigger path (guard B) on the product path | **WP1 Notepad product path (CONFIRMED):** old Debug runtime — idle `,`, `nihao,` and `www.` each block the Host on a Debug CRT assertion dialog (Host alive but hung); the TIP sees a post-write failure → Reject (comma lost; `nihao,` leaves raw `ni hao`); `;` QuickPhrase prompt unaffected. Rebuilt Debug and Release runtimes: `，`, `你好，`, QuickPhrase `www.` and `;` correct, Host healthy. Old Release runtime: same inputs observed correct | **CONFIRMED on the old Debug runtime; attributed to the missing guards (R1).** Old Release: unguarded code not observed to misbehave on these inputs (observation, not a proof of safety) | **RESOLVED on the rebuilt runtime** (observed) | as R1 | — | this plan |
| R3 | Stroke-filter MSVC Debug assertion (string_view end dereference on a stroke key) | Originally reported by the PR #12 implementation pass (claimed base reproduction, raw log not read). **WP1 (CONFIRMED, independent):** the base-API probe (`nihao`, `` ` ``, `h`) on the old Debug runtime reproduces "cannot dereference end string_view iterator" on the stroke key (exit `0xC0000409`, raw log kept); on the rebuilt Debug and Release runtimes (and old Release) the probe completes, all keys consumed | **Reproduced on the old Debug runtime; does not reproduce on the rebuilt runtime.** Guard B (`UTF8StringViewIter::update`, same failing site) is a plausible attribution, not an independently isolated cause | **RESOLVED on the rebuilt runtime** (observed; history kept) | pinned-runtime follow-up · WP1 (no later than the release gate, per arch §25) · independent reading of the raw probe/log | Done (independent reproduction on the old runtime, none on the rebuilt one); reopen only on a new reproduction with the rebuilt runtime | arch §25 runtime bullet (updated by WP1) + this plan |
| R4 | Historical diagnostics emitted keystroke-adjacent data in every TIP host process | Historical source (CONFIRMED); repaired source and policy tests (CONFIRMED) | The candidate now defaults off and redacts prohibited details when explicitly opted in | REPAIRED in WP2; U3 (a) accepted 2026-10-18; WP2 closed within its scope | Windows TSF/diagnostics · U3 · — | Default-off and opt-in redaction tests | this plan → product source |
| R5 | Split into R5a–R5c (first revision: "no persisted user data/config", optional via U1) | — | — | see R5a–R5c | — | — | — |
| R5a | **Input-method/scheme selection** is hard-coded to `pinyin`; Shuangpin cannot be selected | Source (CONFIRMED); repaired in WP3 | Correct. Opening the user path alone does not make Shuangpin selectable | **RESOLVED in WP3**: the profile's input-method list selects the method in use (`Items/0`), and the WP1 runtime's missing `inputmethod\shuangpin.conf` is now generated and copied, so Shuangpin is a real entry | Windows Host · WP3 · Fcitx input-method manager/group API; U1 (how the user selects) | Verified on the product path: Pinyin first run and Shuangpin both commit 你好 (B3a/b, `docs/windows-stage6-wp3.md`) | this plan; `docs/windows-stage6-wp3.md` |
| R5b | **Persisted per-user configuration** (selected input method, Shuangpin profile, pinyin options) absent (`SKIP_FCITX_USER_PATH`) | Source (CONFIRMED); repaired in WP3 | Correct | **RESOLVED in WP3**: the product Host enables Fcitx's standard per-user path and Fcitx's own load/save does the persistence (no custom serialization); `PageSize=9` survived a Host restart | Windows Host · WP3 · Fcitx standard paths (AppData), U1 | Verified on the product path (B3c); a file edited while the Host runs is overwritten by its next save (B3f, R27) | this plan; `docs/windows-stage6-wp3.md` |
| R5c | **Persisted learning / user dictionary** absent | Source (CONFIRMED); repaired in WP3 | Correct; learning semantics are Fcitx's (D082) | **RESOLVED in WP3**: learning persists across an orderly restart and across a console close; forced termination loses changes since the last save (Fcitx's own semantics) | Windows Host · WP3 · Fcitx/LibIME existing save mechanisms, U5 (no learning in sensitive fields) | Verified on the product path (B3d, B3g pass; B3e observation); upgrade/uninstall data policy is U1, uninstall is WP4 | this plan; D082; `docs/windows-stage6-wp3.md` |
| R6 | No Host start mechanism; the DLL must not start the Host | Source + arch (CONFIRMED) | Without one, users get silent not-owned typing | **ADDRESSED in WP4 (implemented, not merged):** a per-user logon scheduled task starts the windowless Host at Medium integrity (U2 (b)); the DLL still never starts it. The start-once / second-start / sign-out behaviour is walkthrough evidence B3b/B3c and is **not yet run** | Windows packaging · WP4 walkthrough · U2 | B3b/B3c results | arch §14 P0-D, this plan, `docs/windows-stage6-wp4.md` |
| R7 | 2 s request bound → permanent backend-session fence until the Host **process** restarts | Source (CONFIRMED) | Correct. An alive-but-unhealthy Host is not restarted by a logon-start mechanism | REQUIRED (WP5 evidence; recovery path decision) | Windows Host + P0-A owner · WP5 · U2, U7 | WP5 timeout-fence scenario (§8.2) and soak frequency (WP7) | arch §12/§13, this plan |
| R8 | x64 only; 32-bit apps unsupported | Build (CONFIRMED) | Correct | DEFERRED (exclusion) | Windows build · owner requests 32-bit or a target app is x86 · x86 runtime | Matrix app found to be x86 | this plan |
| R9 | Persistent registration requires elevation (HKLM) | arch §14 (CONFIRMED, scoped) | Correct for cross-process TIP load | **ADDRESSED in WP4 (implemented, not merged):** `install.ps1`/`uninstall.ps1` refuse to run unelevated, verify the package manifest, and register from the installed version directory; the Host itself stays at Medium integrity. Install/uninstall results are walkthrough evidence B3a/B3e/B3f and are **not yet run** | Windows packaging · WP4 walkthrough · — | B3a/B3e/B3f results | arch §14, `docs/windows-stage6-wp4.md` |
| R10 | Pipe DACL = user SID → AppContainer/LPAC clients cannot connect; High-IL / RDP / multi-session / FUS untested | Source (CONFIRMED) + arch §14 (OPEN) | AppContainer exclusion follows from the ACL; others UNKNOWN | DEFERRED (exclusions); CONDITIONAL if a matrix field needs them | P0-D owner · a required app needs it · P0-D evidence | Per-scenario P0-D evidence | arch §14 |
| R11 | Password / private InputScope admission needed a TSF-side boundary | Historical gap plus repaired source (CONFIRMED); final-candidate browser runs 2026-10-18: Edge password field blocked (no composition, no backend admission, no key content logged) through the existing `GUID_COMPARTMENT_KEYBOARD_DISABLED`/empty-context gate with InputScope read as Unknown; Firefox (fresh process, effective control) blocked pre-admission by the same gate, evaluated at T (`HARD_POLICY guid-blocked`; 0 `KEY_ARRIVAL`/`FCITX_BATCH`); an earlier INCONCLUSIVE Firefox run is consistent with R24 (its specific trigger is inferred). `INPUT_SCOPE class=unknown` observed in Notepad, Edge and Firefox — a confirmed observation with an undiagnosed cause (app metadata absence and probe/read failure are hypotheses) | The bounded policy probes the current selection with synchronous `TF_ES_SYNC|TF_ES_READ`, holds the live entry with generation fencing, blocks Sensitive, and records Unknown without heuristic classification | REPAIRED implementation; U5 minimum outcome observed in Edge and Firefox; the Sensitive classification path is not demonstrated in either browser; U5 (a) accepted with this limitation (§7.1) | arch §10 hard-policy owner · U5 · browser sessions; Firefox control diagnosed (R24) | Future bounded validation of Sensitive classification on one minimal case with a known sensitive InputScope, reusing existing smoke/diagnostic facilities | arch §25 table row (long-term capability; separate decision) + `docs/windows-stage6-wp2.md` |
| R12 | Multi-application out-of-process re-verification (Stage 4 VALIDATION DEBT) | arch §25 Stage 4 (CONFIRMED debt) | Debt, not a known failure | REQUIRED (WP6) | WP6 · after WP4 (may overlap WP5, as WP6 permits) | Matrix results | arch §25 |
| R13 | Chromium/Edge P0-C census DEFERRED UNKNOWN | arch §7.6/§18 | UNKNOWN | REQUIRED in matrix (Edge); a failure is classified, not assumed | WP6 | Edge results | arch §7.6/§18 |
| R14 | Exact-caret popup anchoring | arch §25 DEFER row (CONFIRMED source) | Limitation at current scope | CONDITIONAL (§7.4); collapsed caret/selection measurement preserved as a candidate approach, not accepted | Windows composition/candidate UI follow-up · its §25 trigger · WP6 results | Popup detached from / obscuring the edit point in a matrix field | arch §25 table row (unchanged) |
| R15 | Popup geometry/font are fixed pixel constants, not DPI-scaled | Source (CONFIRMED) | Likely tiny/misplaced at >100 % or mixed-DPI — **hypothesis** | REQUIRED check (WP6); fix CONDITIONAL | Windows candidate UI · WP6 | DPI 100/150 % and two-monitor checks | this plan |
| R16 | Stage 4/5 Release product-path evidence was incomplete | WP2 Release builds and 66/66 Mock plus real-Fcitx smoke exit 0,0,0 are recorded; final candidate (fresh Release build of `559d9ce`) CTest 74/74 and Notepad A–E PASS (2026-10-18) | Release product path evidenced for Notepad | Done for WP2 (other apps: WP6) | Windows build · WP2 · rebuilt runtime | Release suites + final-candidate Notepad A–E | this plan |
| R17 | Windows latency unmeasured (A′ sync EFFECT makes it user-visible); also unmeasured: end-to-end key → document-settlement latency and synchronous EditSession refusal/fallback frequency | arch §15/§21 (CONFIRMED gap) | Unknown | REQUIRED bounded measurement (WP7) using existing diagnostics, tied to the §15 Exit Gate signal | WP7 · after WP2 (opt-in diagnostics) | Recorded P50/P95/P99/max and refusal/fallback counts; thresholds stay hypotheses until the owner accepts them | arch §15/§21 |
| R18 | IME open/close compartment observed **closed** in a new Notepad context; user toggled with Ctrl+Space | Stage 5 smoke run 1 (CONFIRMED, one app) | Per-app default unknown | REQUIRED to observe (WP6) then decide (U6) | Windows TSF · WP6 · §10 hard policy | Per-app observation | arch §10 |
| R19 | Modifier combinations never reach Fcitx | arch §25 NEEDS SEPARATE row | Affects mode toggles / some engine bindings | DEFERRED (row unchanged) unless U1/U6 needs a Fcitx-side toggle | as recorded | — | arch §25 table row |
| R20 | No signing, installer, release configuration or channel | Repository (CONFIRMED absence) | — | Proposed and pending (U4); no signing rule is accepted by this plan | owner · WP8 · certificate, channel | — | this plan |
| R21 | Other Stage 5 deferred/separate capabilities | arch §25 table | unchanged | DEFERRED; owners/triggers as recorded | as recorded | as recorded | arch §25 table (not duplicated) |
| R22 | **Failure-path accounting is unverified beyond Notepad/Stage 4 scope**, and the user-visible recovery representation/notification is undecided (P0-A §7.3 contract, P0-B representation) | arch §7.3, §7.5, §12, §25 Stage 4 (CONFIRMED: mechanisms exist; user-visible disposition deliberately left open). WP1 incidental evidence (old Debug runtime, R2): a hung Host produced a post-write failure → Reject, no PASS, raw composition text left in the document — an input for WP5/U7, not an acceptance | Mechanisms correct at Stage 4 scope; behaviour in other apps and under the timeout fence unverified | REQUIRED (WP5 evidence; U7 decision before the release candidate) | P0-A/P0-B owner (Windows architecture) · WP5 evidence · existing ledger/fencing/teardown audit | §8.2 failure scenarios per matrix app | arch §7.3/§12; this plan §5 |
| R23 | `tools/fcitx/build-native.ps1` fails under Windows PowerShell 5.1 on a fresh source checkout: its "patch already applied?" probe `git apply --reverse --check … 2>$null` raises a terminating `NativeCommandError` with `$ErrorActionPreference = 'Stop'` | WP1 (CONFIRMED; no PowerShell 7 on the machine) | Tooling defect, not a runtime/product defect. WP1 workaround: apply the same two repository patches to the fresh checkout with `git apply` first (the probe then succeeds); rebuild otherwise unchanged | **FIXED in WP4.** `Invoke-Native` and the probe now relax `$ErrorActionPreference` for the native call and decide on `$LASTEXITCODE`. The defect class was broader than the probe: on this machine a CMake *warning* on stderr aborted the Debug runtime build at the same place | Windows build tooling · done · — | Verified in isolation under PowerShell 5.1 on a fresh checkout: the old probe form raises, the fixed probe returns false→apply→true and is idempotent (`docs/windows-stage6-wp4.md`, A5) | `docs/windows-stage6-wp4.md`, this plan |
| R24 | TSF test-key eligibility (`routes_key`) required semantic readiness and backend health (`healthy()`). Both are re-established only on the K (`OnKeyDown`) path. Applications that consult `OnTestKeyDown` and skip K after FALSE (Firefox; Word per arch §8) therefore never recover a non-ready context, e.g. after a Host restart or a bind on a stale link | Source plus Firefox runs on 2026-10-18. CONFIRMED: readiness-dependent T eligibility prevents K-path recovery; reproduced on `559d9ce` twice after a Host restart, and the fixed build recovers. INFERRED (bind result not directly logged), consistent with the confirmed defect: the first INCONCLUSIVE WP2 Firefox run was triggered by a stopped previous Host incarnation and a failed initial bind | Product defect in the TSF adapter. The fix needs no change to ownership, PASS ordering, Fcitx semantics or privacy | FIXED: fusion-windows PR #15, merged (`main` `0cd0374`). The fix removed both terms, so T eligibility is key-class only; K is unchanged. Fix evidence (2026-10-18, branch head `7a6542b`, the same tree as the merge): recovers after a Host restart in Firefox; CTest 74/74 | Windows TSF · done · — | WP5 scenarios in T-consulting apps (Firefox, Word): (a) Host restart; (b) Host stopped or hung. Because T now routes eligible keys to K regardless of readiness, each key in a non-ready context performs one bounded reconnect attempt (`kLinkTimeout`). Measure the per-key latency with the Host process absent and with it alive but unresponsive | this plan + fusion-windows PR #15 (merged) |
| R25 | Firefox: the candidate popup is not shown while composition and commit work | Firefox runs on 2026-10-18 (CONFIRMED observation: hidden in 3 of 4 runs). Source: the popup is hidden whenever the `GetTextExt` anchor probe fails, and that failure is not logged | Cause undiagnosed. Hypothesis: `GetTextExt` fails (e.g. `TS_E_NOLAYOUT`) during composition; arch §18 lists this behaviour as unsurveyed. While the popup is hidden, pointer selection of candidates is unavailable | REQUIRED in the WP6 matrix (Firefox); may start earlier on owner instruction | Windows candidate UI · WP6 · R14 anchoring | Content-free logging of the anchor-probe result in Firefox | this plan |
| R26 | The **logoff/shutdown save** of the Host depends on how it is started (a console close is handled and saves; a logoff or shutdown that gives the process less time, or a start mechanism without a console, may not) | WP3 orderly-stop evidence (CONFIRMED for console close and Ctrl+Break); deployment-dependent for logoff/shutdown | Correct for the current start mechanism; unresolved for the future one | **ADDRESSED in WP4 (implemented, not merged):** the packaged Host has no console, so a hidden top-level window handles `WM_QUERYENDSESSION`/`WM_ENDSESSION` and waits, bounded at 4 s, for the ordered shutdown including Fcitx's save; the console path is unchanged. The save itself is proven for the stop-event and `CTRL_BREAK` paths on this machine (`docs/windows-stage6-wp4.md`, A1/A2/A3); the **sign-out/sign-in** half is walkthrough evidence B3c and is **not yet run** | Windows packaging · WP4 walkthrough (B3c) · U2 | B3c result | `docs/windows-stage6-wp3.md` (B3g, shutdown ordering); `docs/windows-stage6-wp4.md` |
| R27 | A Fcitx configuration file edited **while the Host runs** is overwritten by the Host's next save, because Fcitx saves its in-memory configuration on instance exit | WP3 B3f (CONFIRMED observation) | Correct Fcitx behaviour, not a defect | DOCUMENTED limitation; the Host must be stopped before configuration files are edited | Windows Host · owner of the user-facing procedure · a user-visible settings surface (none at this scope) would remove the hazard | Documented in the selection procedure; revisit when a settings surface exists | `docs/windows-stage6-wp3.md` (B3f; User-facing selection procedure) |
| R28 | **An upgrade or uninstall cannot remove a previous version directory while a running application still holds that version's `FusionTsf.dll`.** The installer leaves the directory in place and reports it; a later install or uninstall removes it. The old application keeps using the old DLL against the new Host (same pipe, unchanged protocol) | WP4 design and local verification (CONFIRMED by construction: the DLL is mapped by the application process); the concrete lock behaviour is walkthrough evidence B3d/B3e | Expected Windows loader behaviour, not a defect. Force-deleting or replacing a mapped DLL is excluded by decision | DOCUMENTED limitation; no fix scheduled | Windows packaging · WP8 (release candidate) or the second upgrade on one machine · nothing | B3d/B3e record which directory stayed and which could be removed | `docs/windows-stage6-wp4.md` (A4, Known limits) |

## 5. Open architecture items: what blocks the target scenario

| P0 | Sub-question | Blocks the target? | Proposed handling | Evidence/decision needed |
|---|---|---|---|---|
| P0-D | Host **startup/lifecycle** (who starts `FusionHost.exe`, per user × session) | **Yes** (R6) | U2 (b) accepted: per-user logon task, implemented in WP4 as the baseline for the §3 scope (arch §14); "the DLL does not start the Host" unchanged; Host topology stays UNFROZEN | WP4 walkthrough evidence (B3b/B3c) |
| P0-D | Installer privilege and registration scope | **Yes** (R9) | Elevated per-machine TIP registration; Host per user at Medium IL (current baseline); implemented in WP4 (`install.ps1`/`uninstall.ps1` refuse to run unelevated) | WP4 walkthrough evidence (B3a/B3e/B3f) |
| P0-D | Medium-IL single session, user-SID pipe ACL | No — implemented baseline | Keep | — |
| P0-D | AppContainer/LPAC, High IL, x86, multi-session, RDP, FUS, runas | No — excluded | Each needs its own evidence when lifted | Per-scenario |
| P0-A | §7.3 recovery contract: user-visible disposition of an owned key / prior EFFECT that cannot reach its required terminal (Host crash, uncertain write, timeout fence) | **Yes for the release candidate** | Existing mechanisms must be shown to give explicit terminals (§8.2); the user-visible behaviour/notification is an owner decision (U7) on that evidence — no loss-of-input policy is accepted by this plan | WP5 evidence → U7 |
| P0-A | Recovery key buffer / journal | No | Deferred | — |
| P0-B | Minimum recovery **representation**: what remains auditable/visible about unresolved responsibility (composition identity, committed prefix, uncertain EFFECT) after a failure | **Yes for the release candidate** (minimum only) | Owner: P0-A/P0-B owner; work pointer: WP5; trigger: WP5 evidence; consequence: no release candidate without an accepted minimum representation/notification (U7) | WP5 evidence → U7 |
| P0-B | Automatic restoration of an in-flight composition after Host restart | No | Deferred (exclusion) | When composition restoration becomes a requirement |

## 6. Work packages (ordered) and dependencies

Execution rule: no work package starts because this plan was merged. Each starts only on its own explicit, bounded
execution instruction after the plan review (WP0).

| WP | Content | Depends on | Entry gate | Exit gate |
|---|---|---|---|---|
| WP0 | Owner review of this plan; decisions U2, U3, U5 (others may follow) | — | this PR | Plan accepted or amended |
| WP1 | **Pinned-runtime compatibility (R1–R3).** Record raw identity first (pinned source SHAs, patch-applied state, installed headers, build logs, binary hashes of the current runtime); rebuild Debug and Release runtimes with the current patches via `build-native.ps1` into a new versioned directory; rerun the existing suites and Notepad A–E; punctuation and `;` QuickPhrase on the product path (old vs new); independently verify R3 | WP0 + explicit WP1 instruction | Clean fusion-windows `main`; pinned revisions unchanged | R1 closed or re-classified with a root cause; R2 resolved; R3 dispositioned with evidence kept. **Executed 2026-10-17:** R1/R2/R3 RESOLVED (§4); new runtime `runtime-wp1-20261017` (Debug + Release); fusion-windows PR #13 (test oracle) merged; WP1 closed within its scope |
| WP2 | **Release configuration and distribution prerequisites.** Release build of DLL + Host on the rebuilt runtime; diagnostics per U3; sensitive/password-field policy per U5 with its minimum validation (R11); artifact identity manifest | WP1, U3, U5 | WP1 exit; builds use the rebuilt runtime (identity recorded) | Release suites and diagnostics pass; InputScope repair is implemented; final candidate `559d9ce`: Notepad A–E PASS, Edge and Firefox password checks PASS for the U5 minimum outcome (technical checks complete). These conclusions apply to `0cd0374` (PR #15) by source analysis and were not re-run there (see the status header). U3/U5 decided (a) on 2026-10-18 (§7.1); WP2 closed within its scope; distribution stays gated by WP4 and U4. |
| WP3 | **Input methods and user data (R5a–R5c).** Pinyin/Shuangpin selection (replacing the hard-coded `pinyin`), per-user configuration persistence, learning/user dictionary persistence through Fcitx's existing mechanisms, upgrade/uninstall data policy (U1). Registering Shuangpin as a second input method fires the trigger of the unchanged arch §25 `overlayMessage` DEFER row ("Host registers a second IM / IM switching reachable"); WP3 records whether IM switching is reachable on the product path and, if so, dispositions that row per its own terms (it depends on the already implemented aux row). Other §25 rows are not pulled in | WP2, U1 | WP2 exit | **Closed within its scope (2026-10-18).** Pinyin and Shuangpin each selectable with representative input committing correctly (B3a/b); `PageSize=9` survived a Host restart (B3c); learning survived an orderly restart (B3d) and a console close (B3g); forced termination loses changes since the last save (B3e, observation); a config file edited while the Host runs is overwritten by its next save (B3f, observation); U1 data policy documented and applied; no learning in U5-sensitive fields (B5, source); the §25 `overlayMessage` trigger check recorded — switching is **not** reachable, so that row stays DEFER (B6). Record: `docs/windows-stage6-wp3.md`; candidate `a720319` on `runtime-wp3-20261018` |
| WP4 | **Packaging and lifecycle.** Package layout (TIP DLL; Host + runtime + `data/` + `lib/libime` + user-data location from WP3); elevated install/uninstall on the existing `register.ps1`/`unregister.ps1`/`check-state.ps1`; Host start per U2; upgrade while the DLL is loaded in running apps (stage, re-register, require app restart/sign-out); test-package delivery | WP3, U2 | WP3 exit | **Implemented and locally verified; owner walkthrough pending; not closed.** U2 (b) accepted: a per-user logon task starts the windowless Host at Medium integrity; the Host saves on session end through a hidden top-level window and is stopped orderly through a named per-user stop event. Two packages built from `faeb266`; B1 70/70 Mock-only, 78/78 Debug, 78/78 Release; A1–A6 evidence and the walkthrough dry run in `docs/windows-stage6-wp4.md`. Exit gate items still open: B3a–B3f (install, second start, sign-out/sign-in, upgrade with Notepad open, both uninstall paths) and B4 (Notepad A–E on the installed wp4.2) |
| WP5 | **Failure and recovery acceptance (R7, R22).** The §8.2 failure scenarios on the packaged build with existing mechanisms; evidence for U7 | WP4 | WP4 exit | §8.2 failure evidence recorded; U7 decided before WP8 |
| WP6 | **Application compatibility and positioning.** Matrix §8.3; positioning/DPI/monitor checks §7.4; U6 observation | WP4 (WP5 may overlap) | WP4 exit | §8.3 gate |
| WP7 | **Stability, resources, latency** §8.4 | WP4 (overlaps WP6) | WP4 exit | §8.4 numbers recorded for owner decisions (R7, R17, U7) |
| WP8 | **Release candidate.** Signing/channel per U4 (pending decision); release notes with scope/exclusions; final identity manifest; independent review | WP5, WP6, WP7, U4, U7 | all prior exits | Reviewable release candidate (not a public release) |

## 7. Owner decisions, scope requirements and positioning

### 7.1 Owner decisions (U1, U2, U3 and U5 owner-accepted 2026-10-18; the others are not assumed accepted)

| ID | Decision | Alternatives | Recommendation (with reason) | Consequence if undecided |
|---|---|---|---|---|
| U1 | **Implementation and data-lifecycle choices** for the required Pinyin/Shuangpin selection, configuration and learning persistence — not whether to keep those capabilities | user-facing selection method (Fcitx configuration file vs a minimal existing switch); storage location (Fcitx AppData standard paths vs an explicit Fusion location); upgrade (keep user data) and uninstall (keep vs remove vs ask) | **Accepted 2026-10-18.** Selection only through Fcitx's existing configuration files: the Fcitx profile's input-method list selects Pinyin or Shuangpin (`Items/0` is the method in use; `DefaultIM` is only the fallback), and the pinyin addon's own config holds the Shuangpin scheme and pinyin options — no Fusion-specific file, switch or registry key. Storage in Fcitx's own standard per-user paths as the pinned Fcitx resolves them (`%APPDATA%\Fcitx5\{config,data}`); no Fusion-specific data location. Upgrade keeps user data; uninstall leaves it unless the user chooses removal (WP4 implements uninstall) | Decided — WP3 closed within its scope; WP4 implements the uninstall choice |
| U2 | Host start mechanism — candidate approaches: (a) per-user logon **Run** entry; (b) **scheduled task** at logon; (c) another minimal mechanism if evidence requires it | (a)/(b)/(c) | **Accepted (b) 2026-10-18.** A per-user scheduled task at logon: principal = the installing user, interactive token, `RunLevel Limited` (Medium integrity, never elevated), no execution time limit, `MultipleInstances = IgnoreNew`, no restart-on-failure. The task starts the Host; the DLL still never starts it. The windowless Host saves on logoff/shutdown through a hidden top-level window, and install/upgrade/uninstall stop it through a named per-user stop event instead of terminating it (`docs/windows-stage6-wp4.md`, A2/A4). (a) is **not** implemented and its gaps are recorded: an elevated installer cannot start a Run entry non-elevated for the current user without a sign-out, and nothing restarts a Run entry that exits. Neither (a) nor (b) restarts a crashed Host or recovers an alive-but-unhealthy one (R7) — that stays WP5 evidence | Decided — WP4 implements (b); U4/U6/U7 remain open |
| U3 | Diagnostics in packages that leave the local developer machine | (a) off by default, opt-in, no key/text content; (b) on without content; (c) unchanged | **Accepted (a) 2026-10-18.** (a) — removes keystroke-adjacent data and disk growth; local controlled development diagnostics stay available | Decided — no longer a WP2 gate; distribution stays gated by WP4/U4 |
| U4 | Signing, distribution channel, test package vs release candidate | certificate available or not; channel | Proposal only: owner-only test packages first; signing decided before any external release candidate. Not an accepted signing rule | No external release candidate |
| U5 | Password/private InputScope minimum policy | (a) fail open (TIP does not own keys) for password/private scopes; (b) pass `Password`/`Sensitive` to Fcitx | **Accepted (a) 2026-10-18.** (a) for the first packages — smallest change, no composition/learning in password fields; (b) remains the faithful long-term path (§25 row). Known limitation, accepted with this decision: in the Edge and Firefox runs the minimum outcome came from the existing `GUID_COMPARTMENT_KEYBOARD_DISABLED`/empty-context hard-policy gate, and the InputScope probe read Unknown in Notepad, Edge and Firefox, so the Sensitive-classification path is not demonstrated and a field that neither trips that gate nor yields a readable InputScope is unverified. The R11 follow-up (bounded validation on one minimal case with a known sensitive InputScope) remains open and is not scheduled by this record | Decided — no longer a WP2 gate; distribution stays gated by WP4/U4 |
| U6 | IME open/close default for a new app context | (a) keep Windows' compartment value; (b) declare open on activation (Activate already writes `KEYBOARD_OPENCLOSE=1`, yet run 1 observed closed) | Observe per app in WP6 first; compartments stay Windows authority (§10) | Users may need Ctrl+Space first (documented) |
| U7 | P0-A/P0-B minimum for the release candidate: user-visible disposition/notification of unresolved responsibility after a failure | decided on WP5 evidence | Decide after WP5; no loss-of-input policy is accepted in advance | No release candidate |

### 7.2 Target scope requirements recap
Pinyin and Shuangpin selection, per-user configuration persistence, and learning/user-dictionary persistence are
target requirements scheduled in WP3; only the interim WP1–WP2 artifacts are Pinyin-only without persistence.

### 7.3 U2 evidence (proportionate)
For the chosen start mechanism: Host starts once per user × session at logon; a second start attempt does not create
a competing server for the same pipe name (singleton / session identity); sign-out/sign-in; behaviour after a Host
crash (does anything restart it, and how the TIP reconnects); behaviour with an alive-but-unhealthy Host after the
2 s fence (R7). Results feed U2 and U7; they do not freeze Host topology.

**As evidenced in WP4** (`docs/windows-stage6-wp4.md`): the mechanism is (b), a per-user logon task with an
interactive token and limited run level (`docs/windows-stage6-wp4.md`, A4). The singleton half is implemented and
covered by tests (`HostClaim` refused a second Host; `stage4a wp3singleton`, `wp4stopevent`); the logon, sign-out and
second-start halves are the walkthrough's B3b/B3c and are **not yet run**. Nothing restarts a crashed Host (no
restart-on-failure setting), and recovery of an alive-but-unhealthy Host stays WP5/U7 evidence.

### 7.4 Positioning: release requirement vs polish (proposed)
- **Required for the release candidate:** in every matrix field the popup is on the same monitor as the edit point,
  fully on screen, does not cover the composition it serves, follows the composition after window move/scroll before
  the next keystroke, and is legible and clickable at 100 % and 150 % scaling and on a secondary monitor with a
  different scale (R15).
- **Conditional:** exact-caret anchoring (R14) — required only if a matrix field shows the popup detached from or
  obscuring the edit point because of the composition-edge anchor.
- **Polish (deferred):** per-monitor font tuning beyond legibility, hover, animation, theme/dark mode.

## 8. Validation and gates

### 8.1 Mandatory vs conditional validation
- **Mandatory:** existing Mock-only and real-Fcitx suites on every candidate (Debug; Release from WP2); Notepad A–E on
  every packaged candidate; WP3 input-method/user-data exits; WP4 lifecycle evidence; WP5 failure scenarios; the
  application matrix; positioning checks; bounded stability/latency; identity verification of the DLL actually loaded
  by the target process.
- **Conditional:** exact-caret implementation (R14); DPI fix (R15); recovery changes if WP5 shows missing terminals or
  R7 occurrences; P0-D evidence when an exclusion is lifted.

### 8.2 Failure and recovery acceptance evidence (WP5, existing mechanisms)

Each scenario records, per key and per prior EFFECT, the disposition from the target process's diagnostics and the
Host log, plus the user-visible document state. Required invariants in every scenario (arch §7.3–§7.5, §20):
explicit terminal disposition for every owned key; no cancelled key represented as a genuine PASS; consumed STATE →
EFFECT watermark and causal order preserved (no later key overtakes an unresolved prior EFFECT); no blind replay or
retry after an indeterminate mutation; no duplicate irreversible EFFECT; no cross-context EFFECT; no stale/late result
applied after incarnation/connection fencing; no silent loss of an owned key.

The dispositions below are the existing mechanisms (arch §7.1–§7.5, §13, §25 Stage 4A/4B; fusion-windows `0f724af`
`service.cpp` `OnKeyDown`, `semantic.cpp` `begin_real_key`, `ipc_backend.cpp` `send`), applied in this order to
every key; WP5 evidences them, it does not redesign them:

1. **Prior-EFFECT ordering first, independent of Host state.** Before admission is considered, an unresolved prior
   EFFECT is probed (`probe_pass`): drain-before-PASS where lawful; if the required terminal is not reached the
   current key is cancelled with an explicit `CANCELLED_RECOVERY` terminal (eaten, never a PASS). Host absence, crash
   or an unhealthy session does not by itself bypass this obligation.
2. **Pre-ownership admission.** Only after (1) is clear: hard policy, semantic readiness (link + incarnation +
   binding), backend health, capacity and `probe_pass` decide whether the TIP takes ownership. If not, the key is not
   owned (fail open before ownership) and it still reaches the application only through the PASS path, which again
   requires drain + `finish_pass`/watermark before returning it as a genuine PASS.
3. **Request write certainty (an owned stimulus sent to the Host).** No established link at send time, or a request
   write that provably accepted no byte (`NotWritten`), leaves the physical key caller-owned and lawfully answered as
   **Forward** — still subject to the PASS-path conditions of (2). An `Uncertain` write, or any failure after the
   request is on the wire (missing/mismatched/undecodable response), is **Reject**: explicit cancelled terminal, no
   retry, no replay.
4. **Responsibility already accepted (owned records).** Callbacks admitted from a valid batch are owned STATE/EFFECT
   records and must each reach an explicit terminal; on disconnect they are retained (not dropped, not converted to
   PASS) together with any undelivered materialization queue, and results of a fenced incarnation/connection are not
   applied.
5. **Document mutation outcomes are distinct.** A synchronous EditSession refusal (no session ran) leaves the work
   queued for the async fallback — no mutation, not an uncertain write; asynchronous work still pending is unresolved
   (`TF_S_ASYNC` is admission, not completion) and keeps blocking later PASS per (1); `RefusedNoMutation` observed
   inside a session is a definite no-effect terminal; only `Indeterminate` (mutation attempted, result not provable,
   e.g. failed `SetText` or readback mismatch) is uncertain, recorded as such and never blindly retried.

| Scenario | Key classes to account for | Expected disposition with existing mechanisms (to be evidenced) |
|---|---|---|
| Host absent before input | Keys arriving with no prior unresolved EFFECT; keys arriving while an earlier EFFECT is still unresolved (e.g. Host lost mid-session) | Rule (1) first: an unresolved prior EFFECT is drained or the key is cancelled (`CANCELLED_RECOVERY`), never passed ahead of it. Otherwise rule (2): not owned; returned through the PASS path (drain + `finish_pass`) as a genuine PASS; reconnect attempted on later keys; no owned records created |
| Host crash during composition | Keys after the crash; the key whose request was being sent; owned records from earlier keys; unresolved prior EFFECT; late results | Keys after the crash: rules (1) and (2) — not owned only when no unresolved prior EFFECT forbids it. The in-flight key: rule (3) — `NotWritten` → Forward (via the PASS-path conditions), `Uncertain`/post-write → Reject, no retry. Owned records: rule (4) — retained to an explicit terminal, teardown audit recorded. Results of the dead incarnation: fenced. Composition text left in the document recorded as the user-visible state (U7) |
| Backend timeout → permanent unhealthy session (R7) | The owned key whose request timed out; later keys while the Host process is alive but unhealthy; unresolved prior EFFECT | Timed-out key: post-write failure → rule (3) Reject (explicit terminal, not PASS). Later keys: rule (1) first, then rule (2) — not owned while the session stays unhealthy, passed only via the PASS-path conditions. Evidence must show how long this lasts and what restores service (U2/U7) |
| Document mutation refused, pending or indeterminate | Synchronous refusal; async work still pending; `RefusedNoMutation`; `Indeterminate` | Rule (5): refusal → queued for async fallback (later keys follow rule (1)); pending → unresolved, blocks later PASS until a terminal; `RefusedNoMutation` → definite no-effect terminal; `Indeterminate` → uncertain, never blindly retried, reported by the teardown/audit; user-visible representation per U7 |

### 8.3 Application matrix (bounded)

Fields: Notepad document; Word document body; Firefox address bar and a web `<input>`/`<textarea>`; Edge address bar and
a web `<input>`/`<textarea>`; one password field per browser (U5 check).

| Scenario (each field; Pinyin and Shuangpin where marked) | Pass condition |
|---|---|
| Continuous input (Pinyin and Shuangpin) | Exactly the committed text; no duplicate/lost characters; no stale preedit |
| Partial selection + continued composition (Pinyin and Shuangpin) | Committed text equals the engine's selection sequence |
| Punctuation (`,` `.` `;` quotes) and QuickPhrase prompt | Fcitx punctuation output exactly once; no Host death (R2) |
| Keyboard candidate selection (digits, Space, `-`/`=`, PageDown/PageUp, arrows) | Selected index matches the highlighted/numbered candidate |
| Pointer candidate selection and paging | Clicked displayed index committed; popup does not take focus |
| Focus/context changes during composition | Explicit terminal, no cross-context commit, no crash |
| Learning visible after Host restart (one representative field) | A learned phrase ranks as Fcitx ranked it before the restart (WP3 mechanisms) |
| Failure scenarios (§8.2) in at least Notepad and one browser | §8.2 invariants |
| Positioning (§7.4) | §7.4 required items |

A field that cannot be tested (app missing, sandboxed field) is recorded as **NOT RUN (dependency)**, never PASS. A
failure is classified as product defect, app-specific TSF behaviour (P0-C/§18 evidence) or harness problem before any
fix.

### 8.4 Stability, resources, latency (bounded, tied to decisions)
- **Soak:** one continuous scripted session of representative input across the matrix apps (proposed 2 hours,
  engineering hypothesis) plus one sign-out/sign-in. **Decision fed:** R7 recovery need, U7.
- **Resources:** Host RSS, handles, threads at start/mid/end; TSF host app handle count before/after. **Decision:**
  whether a leak blocks the release candidate (monotonic growth is investigated; no numeric limit accepted in advance).
- **Latency and refusal frequency** (existing opt-in diagnostics on the Release build): per-key Host round trip,
  end-to-end key → document settlement (`EFFECT_APPLIED`), and synchronous EditSession refusal / async-fallback counts
  (P50/P95/P99/max where applicable). **Decision:** §15 Exit Gate signal evaluation and U7. Numbers are recorded;
  thresholds remain hypotheses until the owner accepts them (arch §21). No separate benchmark is created.

### 8.5 Stop conditions
**Hard Stop (stop and report):** a D072/D082/D083 boundary would have to change (e.g. the DLL would start or host
Fcitx, or synthesize keys); a fix would need a P0-D scenario outside §3; the rebuilt runtime changes engine
semantics; user data would be persisted against the U1/U5 decisions; a package would leave the local developer
machine with keystroke-adjacent logging (U3) or without the U5 minimum.
**Not a Hard Stop:** ordinary, explainable build, test-harness or environment failures (missing dependency, harness
bug, registration mistake, flaky fixture) — fix within the work package's scope and continue; an unexplained
Release-vs-Debug difference is investigated before it is classified.

## 9. First execution task (WP1) — executed 2026-10-17 under its own owner authorization

- **Goal:** Establish first-hand identity of the pinned native runtime and make it match the repository's current
  MSVC patches; settle R1–R3 with raw evidence.
- **Allowed scope:** record current runtime identity (pinned source SHAs, patch-applied state, installed headers,
  build logs, binary hashes); run `tools/fcitx/build-native.ps1` (Debug and Release) into a new versioned runtime
  directory; build against it; run the existing Mock-only/real-Fcitx suites and the existing Notepad A–E harness;
  one product-path punctuation and `;` QuickPhrase check (old vs new); independently read/re-run the stroke-filter
  probe on old and new runtime; record identities. Product changes only if a guard is missing from
  `cmake/patches/*.patch` (patch-only, smallest guard, reviewed separately).
- **Must preserve:** pinned upstream revisions; Fcitx/chinese-addons/LibIME semantics (D082); Stage 5 behaviour and
  tests; registration conventions; R3's history (no silent deletion).
- **Expected behavioural delta:** none in product semantics; expected effect is that the two baseline aborts
  disappear and punctuation/QuickPhrase cannot abort a Debug Host. Any other semantic change is a stop condition.
- **Result (merged; WP1 closed within its scope):** expected delta met with no semantic difference. Old source checkout and installed
  headers match the older patch state `d476f29` and lack both guards (complete old-binary provenance UNKNOWN); rebuilt Debug/Release runtimes from fresh pinned clones + current patches (`runtime-wp1-20261017`,
  old runtime preserved). Mock-only 64/64; real-Fcitx Debug 72/72, Release 72/72 (new and old Release runtime),
  given the stale test-oracle fix in fusion-windows PR #13 (https://github.com/choicky/fusion-windows/pull/13); Notepad A–E PASS with the
  existing Stage 5 registered `FusionTsf.dll` and the rebuilt Debug Host/runtime (fresh Notepad; loaded TIP, Host and
  runtime paths/hashes recorded) — compatibility evidence for that combination, not a smoke of the whole newly built
  binary set; product-path punctuation and
  QuickPhrase correct on the rebuilt runtimes; the stroke assertion reproduces on the old Debug runtime only. No
  `cmake/patches` change was needed. PR #13 later tightened its oracle to exactly one empty-panel snapshot and reran
  only that test (Debug and Release, rebuilt runtime: pass); the full suites and Notepad A–E were not rerun after
  that test-only change. Next boundary at the time: WP2 needed its own authorization (since given; see the WP2 row).

## 10. Accounting and reverse-omission

**Identifier mapping from the first revision (`d4a37a1`):**
R1–R4 and R6–R21 keep their IDs (texts corrected where noted: R1 provenance; R7 alive-but-unhealthy; R11 distribution
prerequisite; R17 adds settlement latency and refusal frequency; R20 signing as proposal). R5 → R5a (scheme
selection), R5b (configuration persistence), R5c (learning/user dictionary), all REQUIRED. New R22 (failure-path
accounting and minimum recovery representation). U1 → implementation/data-lifecycle choices only; U2 → candidate
approaches without equivalence claims; U3/U5 → distribution prerequisites; U4 → proposal; U6 unchanged; U7 → P0-A/P0-B
minimum on WP5 evidence. Old WP1 → WP1; old WP2 → WP2 (+U5 minimum); new WP3 (input methods/user data); old WP3 → WP4;
new WP5 (failure/recovery); old WP4 → WP6; old WP5 → WP7; old WP6 → WP8.

**Reverse-omission:**
- Discovered gaps without scheduled work: none — R1–R22 each have a WP or an explicit DEFERRED/CONDITIONAL disposition;
  the §25 `overlayMessage` row whose trigger WP3 fires (second input method) is accounted in WP3.
- Useful recommendations preserved: collapsed caret/selection measurement (R14, candidate approach); stroke finding
  verification (R3); Stage 4 multi-app debt (R12); Chromium census (R13); WP1 raw identity evidence.
- Resolved matters still labelled unresolved: none; Stage 5 remains CLOSED / VERIFIED; R1–R3 recorded RESOLVED on
  the rebuilt runtime with history kept; R23 (build-script tooling) added, scheduled and now FIXED in WP4.
- ROADMAP: Phase 8 points here as "Stage 6 plan proposed (not accepted); WP1 closed; WP2 closed within its scope (U3/U5 decided (a) 2026-10-18); WP3 closed within its scope (U1 accepted 2026-10-18); WP4 implemented and locally verified, owner walkthrough pending (U2 (b) accepted 2026-10-18)".

**WP1 delivery accounting (2026-10-17):** changed register entries — R1, R2, R3 RESOLVED on the rebuilt runtime
(observed; R1 old-binary provenance UNKNOWN; R3 guard-B attribution plausible, not isolated); R16 partially informed
(Release product path stays WP2); R22 gains incidental hung-Host/residual-text evidence for WP5/U7; R23 new
(CONDITIONAL, tooling). All other R-entries (R4, R5a–R5c, R6–R15, R17–R21), U1–U7 and the P0-A/P0-B/P0-D
subquestions (§5) are unchanged by WP1 and keep the status, owner, trigger and authority recorded above. WP1
boundaries kept: the Notepad A–E run used the existing Stage 5 registered TIP with the rebuilt Host/runtime; WP2 is
not started. Reverse-omission for WP1: no discovery without scheduled work (the PS 5.1 script defect is R23, the
hung-Host evidence is under R22); no hypothesis dropped (R3 attribution and old-binary provenance kept open as
stated); nothing resolved still marked unresolved; ROADMAP Phase 8 reflects WP1.

**WP4 delivery accounting (2026-10-18):** changed register entries — R6 ADDRESSED (implemented, walkthrough evidence
B3b/B3c pending); R9 ADDRESSED (implemented, B3a/B3e/B3f pending); R23 FIXED and verified in isolation; R26 ADDRESSED
for the stop-event and session-end paths (implemented and locally evidenced), with its sign-out/sign-in half pending
as B3c; R28 new (DOCUMENTED limitation: a previous version directory stays while an application holds its DLL). U2
moved from open to Accepted (b) with (a)'s gaps recorded. U4, U6, U7 and R7, R11 (follow-up), R12–R22, R24, R25, R27
are unchanged and keep the owner, trigger and authority recorded above. The P0-A/P0-B subquestions (§5) are unchanged;
P0-D gains the implemented startup baseline (arch §14) and stays OPEN for the excluded scenarios. WP4 boundaries kept:
no MSI/signing/channel (U4), no restart-on-failure (WP5/U7), no PATH change, no force-delete, and no change to Fcitx
semantics or TSF admission. Reverse-omission for WP4: no discovery without scheduled work (the locked-directory
behaviour is R28, the Run-entry gaps are recorded under U2); no hypothesis dropped (the walkthrough items are marked
NOT RUN, not passed); nothing resolved still marked unresolved (R23 closed, R6/R9/R26 explicitly bounded by the
walkthrough); ROADMAP Phase 8 reflects WP4.
