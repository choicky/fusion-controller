# Windows Stage 6 — Plan and Acceptance Gates (PROPOSED)

> **Status: PLAN PROPOSED — NOT accepted.** Current state (2026-10-19): WP1 is merged and closed within its accepted scope (fusion-windows PR #13, controller PR #32; R1–R3 in §4, §9). WP2 implementation is merged (fusion-windows PR #14 → `main` `559d9ce`, controller PR #33), and its final-candidate technical checks are complete on `559d9ce`: Notepad A–E PASS; the Edge and Firefox password checks PASS for the U5 minimum outcome, with both blocked by the existing keyboard-disabled hard-policy gate before admission; the InputScope probe read Unknown, so Sensitive classification is not demonstrated (`docs/windows-stage6-wp2.md`). U3 (a)/U5 (a)/U1 are owner-accepted (2026-10-18; §7.1); WP2 is closed within its scope, and WP3 is closed within its scope after the owner's walkthrough on `runtime-wp3-20261018` (B3a–B3g and Notepad A–E PASS, `docs/windows-stage6-wp3.md`). The Firefox diagnosis confirmed R24 (fixed by fusion-windows PR #15, merged as `main` `0cd0374`) and recorded R25. PR #15 changed only `SemanticController::routes_key`: it removed the semantic-readiness and backend-health (`healthy()`) terms and left key-class eligibility unchanged. The T-path decision that produced the Firefox password-field result (`mapped && hard_policy_allows(...)` in `OnTestKeyDown`) and the K path are unchanged, and Notepad and Edge did not deliver T in the recorded runs. The U5 minimum-outcome and Notepad A–E conclusions therefore apply to `0cd0374` by source analysis; they were not re-run on `0cd0374`. U2 (b) is owner-accepted (2026-10-18; §7.1) and WP4 is **CLOSED within its scope on 2026-10-19** (branch `feature/stage6-wp4-packaging`; B1 71/71 Mock-only, 79/79 Debug, 79/79 Release; A1–A6 evidence, both walkthrough runs and the harness follow-ups in `docs/windows-stage6-wp4.md`). The 2026-10-18 run (`wp4-run-20261018-185004`) met the exit gate except B3c2, which failed; the 2026-10-19 run (`wp4-run-20261019-130037`) passed every step on the fixed packages, including the cancelled logoff (c2), the new cancelled restart (c3) and the upgrade with a running Host. **R34** is RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement. **R30 is IMPLEMENTED** (fusion-windows PR #19: opt-in by `FUSION_HOST_DIAGNOSTICS`, a technical-only log in
`%LOCALAPPDATA%\fusion-host\`, 2 MiB cap, five-file retention; Fcitx's own log excluded by owner decision 2026-10-19).
**Current direction: WP7 daily use** (owner decision 2026-10-19). WP5 and WP6 are accepted through daily-use experience and
owner closure — a reported problem is assessed and repaired by impact, and no report is required when nothing goes wrong;
no separate fault-injection or application-matrix campaign is required, and no pre-WP8 crash/hang session remains
mandatory. **U6 is decided** (a new application window starts with Fusion open) and **U7 presentation is decided**
(persistent tray status, fault balloon, restart-Host and open-log-directory actions), with the **tray implemented in
fusion-windows PR #26**; hang detection is a separate pending follow-up and **not** an entry condition for WP8. The
owner decided the next work on 2026-10-20 (D085): the **official settings UI** first, then the **EXE installer** that
packages it. The settings-UI feasibility investigation D085 required is complete (2026-10-20, §7): the unmodified
upstream build cannot be configured here, but the dependency trace shows a **minimal upstream-UI adaptation** is the
right route, and a bounded build-and-transport spike has **proven it on Windows** (built, transport replaced at the
provider seam, one real UI round trip through the official widget layer) — Qt6 is required, and the transport, X11 and
KF6 requirements are separable features rather than the whole stack. The UI is now **implemented on
`feature/official-settings-ui-product`** (fusion-windows PR #27) under D086's scoped licence and packaged for owner
testing; it is **not installed or owner-accepted**, and the real product GUI round trip has not been executed
(`docs/windows-settings-ui-feasibility.md`). Signing and the distribution channel stay open (U4). **Three owner-reported incidents from the first daily-use sessions are tracked as R35, R36 and
R25:** Notepad-- 3.4 crashed on Win+Space because the TIP bound to that application's own older `MSVCP140.dll` (fixed in
fusion-windows PR #21 and **owner-confirmed** on the repair package), idle Return in the same application inserted no newline
because the shell claimed a key the engine only forwards (fixed in fusion-windows PR #23, merged, and **owner-confirmed on
2026-10-20 with `0.0.0-wp7+24914ff`**), and the browser candidate popup is **recovered, owner-confirmed on 2026-10-20**
with the installed `0.0.0-wp7+995e5a5`: measured on 2026-10-20 as `TS_E_NOLAYOUT` on Firefox's composition range, fixed in
fusion-windows PR #25 with a caret fallback and verified in fresh isolated processes; the owner's earlier 2026-10-20 field
table came from browser processes that had never loaded that package (one of them built from source that has no
candidate-window code at all) and is retained as a measurement of those builds, and after restarting the browsers the
owner reported Firefox and Edge normal — closing the incident at current-use scope, with the one-key gap in web content
unchanged and the optional capture in `docs/windows-daily-use.md` §4 still available. None changes U6, and none is a
WP5/WP6 campaign.
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
settings UI **as a general desktop-configuration product** — *partially superseded 2026-10-20 (D085):* the owner
scheduled an official settings UI that reuses Fcitx's existing configuration files and options, so this exclusion now
means no second configuration model, not the absence of a UI; MoQi, Voice and other Android-line
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
| R6 | No Host start mechanism; the DLL must not start the Host | Source + arch (CONFIRMED) | Without one, users get silent not-owned typing | **ADDRESSED in WP4 (closed 2026-10-19; fusion-windows PR #17 merged):** a per-user logon scheduled task starts the windowless Host at Medium integrity (U2 (b)); the DLL still never starts it. B3b (a second start is refused) and B3c (learning survives sign-out/sign-in) **pass** in both runs, logon autostart is verified (18:58:43 and the 2026-10-19 sign-in), and B3c2 now **passes**: a cancelled logoff and a cancelled restart each restore a Host in the same session — **R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement | Windows packaging · WP4 walkthrough · U2 · **R34** | B3b/B3c/B3c2 results in both runs | arch §14 P0-D, this plan, `docs/windows-stage6-wp4.md` |
| R7 | 2 s request bound → permanent backend-session fence until the Host **process** restarts | Source (CONFIRMED) | Correct. An alive-but-unhealthy Host is not restarted by a logon-start mechanism | OBSERVED IN DAILY USE (owner policy 2026-10-19): the fence stays; a reported hang is investigated then, with the Host log (R30). No deliberate hang injection is scheduled | Windows Host + P0-A owner · WP7 daily use · U2, U7 | A reported hang, or the WP7 closure | arch §12/§13, this plan |
| R8 | x64 only; 32-bit apps unsupported | Build (CONFIRMED) | Correct | DEFERRED (exclusion) | Windows build · owner requests 32-bit or a target app is x86 · x86 runtime | Matrix app found to be x86 | this plan |
| R9 | Persistent registration requires elevation (HKLM) | arch §14 (CONFIRMED, scoped) | Correct for cross-process TIP load | **ADDRESSED in WP4 (closed 2026-10-19; fusion-windows PR #17 merged):** `install.ps1`/`uninstall.ps1` refuse to run unelevated, verify the package manifest, and register from the installed version directory; the Host itself stays at Medium integrity. The install/uninstall results **pass** — B3a (install), B3e (uninstall keeping data) and B3f (reinstall + `-RemoveUserData`) in run `wp4-run-20261018-185004` | Windows packaging · WP4 walkthrough · — | B3a/B3e/B3f **pass** | arch §14, `docs/windows-stage6-wp4.md` |
| R10 | Pipe DACL = user SID → AppContainer/LPAC clients cannot connect; High-IL / RDP / multi-session / FUS untested | Source (CONFIRMED) + arch §14 (OPEN) | AppContainer exclusion follows from the ACL; others UNKNOWN | DEFERRED (exclusions); CONDITIONAL if a matrix field needs them | P0-D owner · a required app needs it · P0-D evidence | Per-scenario P0-D evidence | arch §14 |
| R11 | Password / private InputScope admission needed a TSF-side boundary | Historical gap plus repaired source (CONFIRMED); final-candidate browser runs 2026-10-18: Edge password field blocked (no composition, no backend admission, no key content logged) through the existing `GUID_COMPARTMENT_KEYBOARD_DISABLED`/empty-context gate with InputScope read as Unknown; Firefox (fresh process, effective control) blocked pre-admission by the same gate, evaluated at T (`HARD_POLICY guid-blocked`; 0 `KEY_ARRIVAL`/`FCITX_BATCH`); an earlier INCONCLUSIVE Firefox run is consistent with R24 (its specific trigger is inferred). `INPUT_SCOPE class=unknown` observed in Notepad, Edge and Firefox — a confirmed observation with an undiagnosed cause (app metadata absence and probe/read failure are hypotheses) | The bounded policy probes the current selection with synchronous `TF_ES_SYNC|TF_ES_READ`, holds the live entry with generation fencing, blocks Sensitive, and records Unknown without heuristic classification | REPAIRED implementation; U5 minimum outcome observed in Edge and Firefox; the Sensitive classification path is not demonstrated in either browser; U5 (a) accepted with this limitation (§7.1) | arch §10 hard-policy owner · U5 · browser sessions; Firefox control diagnosed (R24) | Future bounded validation of Sensitive classification on one minimal case with a known sensitive InputScope, reusing existing smoke/diagnostic facilities | arch §25 table row (long-term capability; separate decision) + `docs/windows-stage6-wp2.md` |
| R12 | Multi-application out-of-process re-verification (Stage 4 VALIDATION DEBT) | arch §25 Stage 4 (CONFIRMED debt) | Debt, not a known failure | OBSERVED IN DAILY USE (owner policy 2026-10-19): multi-application behaviour is accepted through WP7 daily use; a reported problem is classified then. No matrix sweep | WP7 daily use | A reported multi-application problem | arch §25 |
| R13 | Chromium/Edge P0-C census DEFERRED UNKNOWN | arch §7.6/§18 | UNKNOWN | OBSERVED IN DAILY USE (owner policy 2026-10-19): Edge/Chromium behaviour is accepted through daily use; a reported problem is classified (P0-C census only if it is needed to explain one) | WP7 daily use | A reported Edge/Chromium problem | arch §7.6/§18 |
| R14 | Exact-caret popup anchoring | arch §25 DEFER row (CONFIRMED source) | Limitation at current scope | CONDITIONAL (§7.4); collapsed caret/selection measurement preserved as a candidate approach, not accepted | Windows composition/candidate UI follow-up · its §25 trigger · WP6 results | Popup detached from / obscuring the edit point in a matrix field | arch §25 table row (unchanged) |
| R15 | Popup geometry/font are fixed pixel constants, not DPI-scaled | Source (CONFIRMED). Owner observation 2026-10-18 (WP4 walkthrough step a): with the **installed** TIP and the Host running, the candidate popup was visible in Notepad at 175 % scaling on a single monitor — something the development DLL never showed. That is evidence against the "tiny/misplaced above 100 %" hypothesis at 175 %/one monitor, and it is not a measurement of geometry | Likely tiny/misplaced at >100 % or mixed-DPI — **hypothesis, partially weakened at 175 %/one monitor**; 100 %, 150 % and two-monitor setups are untested | OBSERVED IN DAILY USE at the owner's 175 %/one monitor (owner policy 2026-10-19); a reported positioning problem triggers the 100 %/150 %/two-monitor checks. Fix stays CONDITIONAL | Windows candidate UI · WP7 daily use | A reported positioning problem; the DPI checks above | this plan |
| R16 | Stage 4/5 Release product-path evidence was incomplete | WP2 Release builds and 66/66 Mock plus real-Fcitx smoke exit 0,0,0 are recorded; final candidate (fresh Release build of `559d9ce`) CTest 74/74 and Notepad A–E PASS (2026-10-18) | Release product path evidenced for Notepad | Done for WP2 (other apps: WP6) | Windows build · WP2 · rebuilt runtime | Release suites + final-candidate Notepad A–E | this plan |
| R17 | Windows latency unmeasured (A′ sync EFFECT makes it user-visible); also unmeasured: end-to-end key → document-settlement latency and synchronous EditSession refusal/fallback frequency | arch §15/§21 (CONFIRMED gap) | Unknown | OBSERVED INCIDENTALLY during WP7 daily use; no mandatory benchmark (owner policy 2026-10-19). A reported latency problem triggers the bounded measurement with the existing diagnostics | WP7 · R30 implemented | Recorded P50/P95/P99/max and refusal/fallback counts if a problem is reported; thresholds stay hypotheses until the owner accepts them | arch §15/§21 |
| R18 | IME open/close compartment observed **closed** in a new Notepad context; user toggled with Ctrl+Space | Stage 5 smoke run 1 (CONFIRMED, one app) | Per-app default unknown | **U6 decided 2026-10-19:** a new application window starts with Fusion open. Implemented in fusion-windows PR #20 (declare the compartment open once per new document manager, never from a key path, so an explicit Ctrl+Space close still holds); real-window confirmation pending the owner's installed check. Per-app observation is now incident-triggered | Windows TSF · WP7 daily use · §10 hard policy | The owner's first installed window check | arch §10 |
| R19 | Modifier combinations never reach Fcitx | arch §25 NEEDS SEPARATE row | Affects mode toggles / some engine bindings | DEFERRED (row unchanged) unless U1/U6 needs a Fcitx-side toggle | as recorded | — | arch §25 table row |
| R20 | No signing, installer, release configuration or channel | Repository (CONFIRMED absence) | — | Proposed and pending (U4); no signing rule is accepted by this plan | owner · WP8 · certificate, channel | — | this plan |
| R21 | Other Stage 5 deferred/separate capabilities | arch §25 table | unchanged | DEFERRED; owners/triggers as recorded | as recorded | as recorded | arch §25 table (not duplicated) |
| R22 | **Failure-path accounting is unverified beyond Notepad/Stage 4 scope**, and the user-visible recovery representation/notification is undecided (P0-A §7.3 contract, P0-B representation) | arch §7.3, §7.5, §12, §25 Stage 4 (CONFIRMED: mechanisms exist; user-visible disposition deliberately left open). WP1 incidental evidence (old Debug runtime, R2): a hung Host produced a post-write failure → Reject, no PASS, raw composition text left in the document — an input for WP5/U7, not an acceptance | Mechanisms correct at Stage 4 scope; behaviour in other apps and under the timeout fence unverified | U7 **decided** 2026-10-19 (presentation); recovery behaviour is observed through WP7 daily use and a reported problem is assessed then (owner policy 2026-10-19) | P0-A/P0-B owner (Windows architecture) · WP7 daily use · existing ledger/fencing/teardown audit | A reported recovery problem; the WP7 closure | arch §7.3/§12; this plan §5 |
| R23 | `tools/fcitx/build-native.ps1` fails under Windows PowerShell 5.1 on a fresh source checkout: its "patch already applied?" probe `git apply --reverse --check … 2>$null` raises a terminating `NativeCommandError` with `$ErrorActionPreference = 'Stop'` | WP1 (CONFIRMED; no PowerShell 7 on the machine) | Tooling defect, not a runtime/product defect. WP1 workaround: apply the same two repository patches to the fresh checkout with `git apply` first (the probe then succeeds); rebuild otherwise unchanged | **FIXED in WP4.** `Invoke-Native` and the probe now relax `$ErrorActionPreference` for the native call and decide on `$LASTEXITCODE`. The defect class was broader than the probe: on this machine a CMake *warning* on stderr aborted the Debug runtime build at the same place | Windows build tooling · done · — | Verified in isolation under PowerShell 5.1 on a fresh checkout: the old probe form raises, the fixed probe returns false→apply→true and is idempotent (`docs/windows-stage6-wp4.md`, A5) | `docs/windows-stage6-wp4.md`, this plan |
| R24 | TSF test-key eligibility (`routes_key`) required semantic readiness and backend health (`healthy()`). Both are re-established only on the K (`OnKeyDown`) path. Applications that consult `OnTestKeyDown` and skip K after FALSE (Firefox; Word per arch §8) therefore never recover a non-ready context, e.g. after a Host restart or a bind on a stale link | Source plus Firefox runs on 2026-10-18. CONFIRMED: readiness-dependent T eligibility prevents K-path recovery; reproduced on `559d9ce` twice after a Host restart, and the fixed build recovers. INFERRED (bind result not directly logged), consistent with the confirmed defect: the first INCONCLUSIVE WP2 Firefox run was triggered by a stopped previous Host incarnation and a failed initial bind | Product defect in the TSF adapter. The fix needs no change to ownership, PASS ordering, Fcitx semantics or privacy | FIXED: fusion-windows PR #15, merged (`main` `0cd0374`). The fix removed both terms, so T eligibility is key-class only; K is unchanged. Fix evidence (2026-10-18, branch head `7a6542b`, the same tree as the merge): recovers after a Host restart in Firefox; CTest 74/74 | Windows TSF · done · — | **No mandatory measurement** (owner policy 2026-10-19): the fixed behaviour is observed in daily use. If a latency problem is reported, measure the per-key cost with the Host absent and with it alive-but-unresponsive (each eligible key performs one bounded reconnect attempt) | this plan + fusion-windows PR #15 (merged) |
| R25 | **Firefox and Edge: the candidate popup is not shown in some fields** while composition and commit work — **RECOVERED, owner-confirmed 2026-10-20** on the installed `0.0.0-wp7+995e5a5` (source `995e5a5`): after the browsers were restarted with that package the owner reported "firefox 和 edge 目前正常", which closes the persistent candidate-invisibility incident at current-use scope, not as a per-field or per-site guarantee | Firefox runs on 2026-10-18 (CONFIRMED: hidden in 3 of 4 runs). **Owner report 2026-10-19 (CONFIRMED per field):** visible in the address bar and in the Google homepage search field, invisible in the ChatGPT conversation editor (contenteditable), where composition and commit still work. **Owner report 2026-10-20 (Windows 7700, package `0.0.0-wp7+24914ff`):** no popup in Firefox *and* Edge affected fields while `nihao` + Space still committed 你好. **Owner table 2026-10-20, reported after installing `0.0.0-wp7+995e5a5`:** Firefox address bar visible; Firefox Gmail compose, ChatGPT editor and Gemini editor invisible; all four Edge cells invisible. **Process identity measured 2026-10-20 03:06 — that table cannot be attributed to the installed package:** Firefox pid 59488 (started 2026-10-18) had `…\stage6-wp3\build-release-wp3fix\bin\FusionTsf.dll` mapped, a build with the candidate window but **without** the R25 diagnostics and **without** the caret fallback; Edge pid 19712 (started 2026-10-16) had `…\stage3c1-ordered-settlement-poc\build-on\bin\FusionTsf.dll`, whose source has **no candidate-window code at all** (`BASE_HEAD.txt` beside the build records base commit `5794a36`; `src/tsf` at that commit has no `candidate_view.*` and only the `CandidateSnapshot` backend operation), so no Edge cell in that table — the address bar included — could have shown a popup. Neither process had loaded `0.0.0-wp7+995e5a5`. This is R28's loader fact, not a product result. | **Cause (Firefox), measured 2026-10-20 in isolated processes:** `ITfContextView::GetTextExt` answers `TS_E_NOLAYOUT` (`0x80040206`) for the composition range on every keystroke, so `candidate_anchor_valid` stays false and the page is hidden; composition and commit do not use the anchor and are unaffected. The same context answers the **selection (caret)** range with a valid rectangle from the second keystroke on, while a collapsed clone of the composition range fails identically (`hr=0x80040206`). **Edge is not a separate conclusion:** the earlier "Edge not affected" result came from a fresh isolated process on the local fixture and on its address bar; the owner's Edge has not been exercised with the installed build, so Edge stays inside the open incident. | **FIX in fusion-windows PR #25** (merged, `main` `2244729`): when the composition probe fails, the probe retries with the default selection and anchors to that rectangle, logging `source=composition-caret`; no mouse-cursor placement, no second pipeline, and when both ranges fail the page is still hidden with the unchanged failure line. **Verified with the installed `0.0.0-wp7+995e5a5` in fresh isolated processes on 2026-10-20:** the popup is visible: the Firefox and Edge address bars anchor on every keystroke (`stage=ok source=composition`, five anchors for five keystrokes), while a real third-party page textarea (`json.cn`) and the Gemini composer anchor from the second keystroke through the caret (Firefox logs `stage=ok source=composition-caret`). **Then owner-confirmed on 2026-10-20** after restarting the browsers with that package: Firefox and Edge are normal, which is the closure of the incident at current-use scope; the individual fields named in the table above were not re-checked one by one. **Remaining limit:** in web content the first keystroke of a composition has no layout in any range, so the popup appears from the second keystroke; the address bars do have layout from the first. That is a one-keystroke gap and does **not** explain a popup that never appears. | Windows candidate UI · WP7 daily use · R14 anchoring | **Closed (owner-confirmed 2026-10-20):** Firefox and Edge were restarted with the installed package and reported normal, so the previously pending per-field check is no longer an open action, and the individual cells of the 2026-10-20 table (Firefox Gmail compose, ChatGPT editor, Gemini editor; all Edge cells) were **not** re-checked one by one. If a field misbehaves again, the optional capture is `docs/windows-daily-use.md` §4 "R25 browser capture" (per-process opt-in, installed-DLL check, log lines, cleanup; local convenience launchers in `worktmp\fusion-windows\browser-candidate-popup-real-pages\`), and the log decides the branch: `stage=text-ext hr=…` vs `no-active-view` / `no-range` vs `hidden reason=no-page` vs `stage=ok` with no visible window | this plan; `docs/windows-daily-use.md` §6 |
| R26 | The **logoff/shutdown save** of the Host depends on how it is started (a console close is handled and saves; a logoff or shutdown that gives the process less time, or a start mechanism without a console, may not) | WP3 orderly-stop evidence (CONFIRMED for console close and Ctrl+Break); deployment-dependent for logoff/shutdown | Correct for the current start mechanism; unresolved for the future one | **ADDRESSED in WP4 (closed 2026-10-19; fusion-windows PR #17 merged):** the packaged Host has no console, so a hidden top-level window handles `WM_QUERYENDSESSION`/`WM_ENDSESSION` and waits, bounded at 4 s, for the ordered shutdown including Fcitx's save; the console path is unchanged. The save itself is proven for the stop-event and `CTRL_BREAK` paths on this machine (`docs/windows-stage6-wp4.md`, A1/A2/A3); the **sign-out/sign-in** half is B3c and **passes**; the **cancelled-logoff** half (B3c2) **failed in the 2026-10-18 run and passes in the 2026-10-19 run** (**R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement) while the handler's decision (query answers TRUE only; the stop is requested from `WM_ENDSESSION(TRUE)`) is unchanged and correct (test `stage4a wp4sessionend`). The save itself still runs whenever the session really ends | Windows packaging · WP4 walkthrough (B3c/B3c2) · U2 · R34 | B3c/B3c2 results in both runs | `docs/windows-stage6-wp3.md` (B3g, shutdown ordering); `docs/windows-stage6-wp4.md` |
| R27 | A Fcitx configuration file edited **while the Host runs** is overwritten by the Host's next save, because Fcitx saves its in-memory configuration on instance exit | WP3 B3f (CONFIRMED observation) | Correct Fcitx behaviour, not a defect | DOCUMENTED limitation; the Host must be stopped before configuration files are edited | Windows Host · owner of the user-facing procedure · a user-visible settings surface (none at this scope) would remove the hazard | Documented in the selection procedure; revisit when a settings surface exists | `docs/windows-stage6-wp3.md` (B3f; User-facing selection procedure) |
| R28 | **An upgrade or uninstall cannot remove a previous version directory while any process still holds that version's `FusionTsf.dll`.** The installer leaves the directory in place and reports it; a later install or uninstall removes it. The old application keeps using the old DLL against the new Host (same pipe, unchanged protocol). The holders are **not only the application that typed**: on Windows 11 the input method is global, so `FusionTsf.dll` stays loaded in shell processes (`explorer.exe`, `WindowsTerminal.exe`, …) until sign-out | WP4 design and local verification (CONFIRMED by construction: the DLL is mapped by the application process); the concrete lock behaviour is walkthrough evidence B3d/B3e | Expected Windows loader behaviour, not a defect. Force-deleting or replacing a mapped DLL is excluded by decision | DOCUMENTED limitation: reported with every holding process; removed by running the uninstaller again after sign-out or a restart. No force-delete and no reboot-pending deletion. **Constraint for future protocol changes:** an upgrade that changed the wire protocol (`protocol_version`) would leave those processes on the old DLL and unable to use the new Host until sign-out (fail open), so a protocol change needs either a sign-out requirement or a compatibility window | Windows packaging · WP8 (release candidate) or the second upgrade on one machine · nothing | B3d/B3e record which directory stayed and which holders kept it | `docs/windows-stage6-wp4.md` (A4, Known limits) |
| R29 | **The packaged TIP is user-visibly a development build.** The profile description is "Fusion Stage 3B (Real Fcitx5 Backend PoC)" (`src/tsf/register.cpp`) and the CLSID/profile GUIDs are the development ones (`src/tsf/guids.hpp`); both appear in the input indicator, the language settings and the CTF keys | Repository plus WP4 snapshots (CONFIRMED) | Correct for an owner-only test package; wrong for anything a user is meant to see as a product | **DEFERRED**: the product name and the permanent identifiers are decided before any package leaves the owner (WP8/U4). Changing identifiers later also requires removing the development-identity registration first (a migration step, not an in-place rename) | owner · WP8/U4 · the release package | The WP8 package shows the decided name and identifiers; the migration note is written with it | **this plan** (WP8/U4 decides the name and identifiers); arch §11 (wire/causal identifiers UNFROZEN) is a related reference only; `docs/windows-stage6-wp4.md` |
| R30 | **The packaged Host had no diagnostic log at all.** It is windowless, so stderr exists only when a caller redirects it, and a failure leaves nothing to read — the 2026-10-18 diagnosis had only the walkthrough's own transcript | WP4 walkthrough and diagnosis (CONFIRMED; it materially hampered the diagnosis) | A real evidence gap for every failure scenario §8.2 requires | **IMPLEMENTED (fusion-windows PR #19).** Opt-in `FUSION_HOST_DIAGNOSTICS` = `1`, `true` or `on` (case-insensitive), read once at Host start; unset or any other value means no directory, no file and no behaviour change (U3 (a)). Sink `%LOCALAPPDATA%\fusion-host\fusion-host-<pid>.log`, `ts= pid= tid= <event> <details>` lines, 2 MiB cap recreated empty on overflow, retention keeps the newest five files and deletes only `fusion-host-*.log`. Content is technical only: every `[FusionHost]` stderr message, client connected/disconnected (connection id, no payload), Hello/request failures (kind + Win32 error, `timeout=1` for `ERROR_TIMEOUT`), the stop request and its path (stop event, `WM_ENDSESSION(TRUE)`, external `WM_CLOSE`, console control), the session-end decision, ordered-shutdown begin, backend shutdown/save begin and end with elapsed ms, and exit. No key codes, pinyin, preedit, candidate or commit text and no per-key lines; the TSF privacy-marker redaction applies as a safety net. Fail-open and thread-safe; it never holds a lock across the backend or the session-end bounded wait and never changes `HostClaim`, the stop ordering or the exit code. Enable with `setx FUSION_HOST_DIAGNOSTICS 1`; it takes effect at the next Host start. **Fcitx's own log is deliberately excluded** (owner decision 2026-10-19): at debug level it can contain key events, so including it would break U3; a warning-level-only option is a separate decision (owner: Stage 6 implementation; trigger: a WP5 scenario that needs it) | Windows Host · WP5 entry · U3 (accepted 2026-10-18) · fusion-windows PR #19 | Unit tests (off / opt-in / redaction / cap / retention) and a real windowless Host run stopped through the stop event. **No mandatory capture** (owner policy 2026-10-19): when a problem is reported, enable the opt-in (`host-diagnostics.ps1 -Enable`) and collect `%LOCALAPPDATA%\fusion-host\` | this plan (§8.2); `docs/windows-stage6-wp4.md` (Known limits) |
| R31 | **"A new window" is not "a new process".** A launched executable may be a launcher (the packaged Windows 11 Notepad hands its window to another process and exits), several windows may share one process (tabs), an open window keeps its DLL mapped, and — because the input method is global — the target process may already hold an **older** DLL from an earlier version before the check even starts | WP4 walkthrough 2026-10-18 (CONFIRMED and measured: launcher 45008 vs window owner 7508; a second launch only added a tab) | Harness-methodology defect, not a product defect — but it invalidated step (a) once and would have invalidated step (d) | **DOCUMENTED rule for WP6/WP7 harnesses**: resolve the process that owns the target's window, and require every window of that application to be closed before asserting a fresh load (closing the windows is necessary but not sufficient when the DLL is loaded globally — check the loaded path and hash, never the process age) | Windows test harnesses · WP6/WP7 · any packaged-application assertion | The WP6 matrix harness applies the rule; `docs/windows-stage6-wp4.md` "Walkthrough harness rules (carry forward)" is the reference | this plan §8.1; `docs/windows-stage6-wp4.md` |
| R32 | **Multiple accounts on one machine.** The TIP registration is machine-wide, but the scheduled task name `FusionHost` is global: a second account's install overwrites the first account's task, and an account without its own task types with no Host (fail open) | Source analysis (HYPOTHESIS — not exercised; this machine has one interactive account) | Correct for the single-user scope; wrong for a shared machine | **CONDITIONAL**: when a second account is in scope, or at WP8. Candidate fix: a per-user task name (for example by SID). Not fixed now | Windows packaging · WP8 or a second-account requirement · U2 | A two-account install-and-type check on one machine | this plan; `docs/windows-stage6-wp4.md` (Known limits) |
| R33 | **The build environment is not reproducible from the repository on the 5700X.** `reg.exe` is blocked by the local security software, so `VsDevCmd.bat` cannot detect the Windows SDK and a build needs an uncommitted local wrapper supplying `INCLUDE`/`LIB`; the available clang-format (23.x) flags files that were formatted with 22.x | WP4 builds and format checks (CONFIRMED on this machine) | Machine condition, not a repository defect | **CONDITIONAL tooling**: WP8 release build (a documented, reproducible build environment) or a future format gate (a pinned formatter version) | Windows build · WP8 · the release build | A documented build recipe that succeeds where `reg.exe` is blocked, or a pinned formatter in CI | `docs/windows-stage6-wp4.md` (Environment facts 3 and 4) |
| R34 | **A cancelled logoff, restart or shutdown ends the Host and nothing restarts it.** The WP4 walkthrough's step (c2) cancelled a logoff: Windows logged User32 1077 ("user 5700X\choicky attempted to log off computer 5700X and failed") at 19:03:20, the Host had already saved (19:03:13) and ended, and no Host ran afterwards; `query user` still shows the same session (2, logon 18:58), so the session never ended. A cancelled restart/shutdown writes **1073** instead (measured 2026-10-19), and a completed restart writes neither | Event-log evidence (`r34-evidence.txt`, 2026-10-18 20:06, elevated) plus the run's user-data mtimes. **The mechanism is an inference**: Windows delivers `WM_ENDSESSION(TRUE)` application by application, so applications processed before the blocking one end before the Cancel takes effect | A user who cancels a logoff is left with no Host (silent not-owned typing) until the next logon. The merged WP4 record attributed this to a "second sign-in" that never happened (corrected in `docs/windows-stage6-wp4.md`) | **RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement** (direction A, owner-accepted 2026-10-18 as an implementation detail of U2 (b), not a new start mechanism): a System-log User32 1077/1073 event trigger, scoped to the installing user's SID, starts the Host again 10 s later. **Scope: all three cancelled ends.** Measured 2026-10-19: a cancelled logoff writes 1077, a cancelled restart or shutdown writes **1073**, and a completed restart writes neither (`b1b-restart.txt`, `b1b-shutdown.txt`). The walkthrough verifies the logoff (c2) and the restart (c3); the shutdown is covered by that measurement — same id, same SID scoping. | Stage 6 implementation · fusion-windows PR #18 · U2 (b), R6 | Verified in run `wp4-run-20261019-130037`: (c2) and (c3) each left a Host in the same session within the bound, and (d) started from a running Host | this plan; `docs/windows-stage6-wp4.md` |
| R35 | **An application that ships an older local MSVCP140 crashes when Fusion is activated.** Notepad-- 3.4 (2025-06-28, x64, Qt 6.6 — the installed `Notepad--.exe` carries a stale `3.2.0.0` version resource, and its own `Qt6Core.dll` is 6.6.0) ends on Win+Space before any character is typed: seven Windows Application Error records on 2026-10-19 (19:38:01-19:43:44) all name `C://Program Files\Notepad--\MSVCP140.dll` 14.32.31326.0+0x13278, `0xc0000005` followed by `0xc000041d`; `FusionTsf.dll` from the installed 76bc009 package is the last module loaded | WER `Report.wer` records (no dump: LocalDumps is not configured for that executable) + the installed DLL hash matching the package manifest + `dumpbin -imports` + a hostile-CRT harness that reproduces the exact fault offset with the installed DLL and stops reproducing with the fix (**CONFIRMED**) | **Cause: CRT version mixing, not U6.** Windows binds the TIP's `MSVCP140` import to the copy the application already loaded (14.32); the TIP is built with the 14.51 toolset and imports `_Mtx_lock`/`_Mtx_unlock` but not `_Mtx_init_in_situ`, so its `std::mutex` relies on the newer header initialisation and the older `_Mtx_lock` dereferences a field that initialisation never filled. The first diagnostic write is in the text service's own constructor, which is why the crash precedes any key. The U6 lifecycle writes were checked as a hypothesis and are not involved | **FIXED in fusion-windows PR #21**: the three CRT mutexes compiled into the TIP DLL become SRW locks, and the rebuilt DLL imports no `_Mtx_*` symbols. **Owner-confirmed on the repair package (2026-10-19): Win+Space in Notepad-- no longer crashes.** That confirmation covers the activation crash in this application; it is **not** a claim of universal Notepad-- compatibility, and idle Enter is a separate defect (R36) | Windows TSF · WP7 daily use · PR #21 | Done (owner, 2026-10-19); a new WER record at a different offset reopens it | this plan; `docs/windows-daily-use.md` §6 |
| R36 | **Idle Return is lost in Notepad--: after committing with Space, Enter inserts no newline** while Fusion is the Chinese/Pinyin method (English mode and punctuation are unaffected) | Owner report 2026-10-19 (**symptom confirmed**). A content-free TSF capture from the owner's own process (pid 23196) carries the whole shell-side path of that Enter: `CLASSIFY type=4` (Pass) → `FCITX_BATCH ops=1` → `CANDIDATE_VIEW hidden`, with no edit session and no `CANCELLED_RECOVERY`. Re-executed on the repair branch against the real Fcitx engine (`stage3c`, 2026-10-20): after `nihao` + Space the commit emits `CommitText("你好")` and then `SetPreedit("")`, so the projection is empty and no page is displayed; the idle Return is answered with `ForwardKey` alone (one operation, zero document callbacks). What the application does after `eaten=FALSE` in `OnKeyDown` is **inferred** from that path, not captured | **Cause: the Windows adapter claimed a key it could only hand back.** `OnTestKeyDown` answered TRUE because Return is T-routable whenever the engine is ready; the engine then answered `ForwardKey` and the shell answered `eaten=FALSE` in `OnKeyDown`. An application that treats the test callback as the decision has already stopped handling the key at that point, so nothing inserts the newline — consistent with the symptom and with engine-consumed keys and engine-idle typing being unaffected. Not a Fcitx defect and not U6 | **FIXED in fusion-windows PR #23** (head `1b2dab8`): `OnTestKeyDown` leaves Return unclaimed when the engine has no work for it (no raw input, no displayed candidate/panel page, no unsettled work, clear pass barrier); composing, QuickPhrase and unresolved work keep engine ownership, and nothing is synthesized, blanket-passed or application-specific. Verified on the real Fcitx engine. **Owner-confirmed on 2026-10-20 on Windows 7700 with the installed package `0.0.0-wp7+24914ff`:** Notepad-- works normally, including the previously failing `nihao` → Space → Enter sequence. That confirmation is scoped to that application and that package, not to Notepad-- generally. The same claim-then-hand-back shape still applies to other keys the engine only forwards (for example `/` and `@` in Pinyin mode); it is unchanged here and would be assessed on report | Windows TSF · WP7 daily use · PR #23, merged as `main` `2d9db81` · confirmed | **Done** — no further action. If the symptom recurs, the expectation is unchanged: with `FUSION_TSF_DIAGNOSTICS=1` set for that application's process, its log (`%LOCALAPPDATA%\fusion-tsf\fusion-tsf-<pid>.log`) shows `TEST_KEY ... vk=0x00D eligible=0` for the Enter and **no** `CLASSIFY` line for it | this plan; `docs/windows-daily-use.md` §6 |

## 5. Open architecture items: what blocks the target scenario

| P0 | Sub-question | Blocks the target? | Proposed handling | Evidence/decision needed |
|---|---|---|---|---|
| P0-D | Host **startup/lifecycle** (who starts `FusionHost.exe`, per user × session) | **Yes** (R6) | U2 (b) accepted: per-user logon task, implemented in WP4 as the baseline for the §3 scope (arch §14); "the DLL does not start the Host" unchanged; Host topology stays UNFROZEN. Logon autostart is verified (18:58:43 and the 2026-10-19 sign-in); a **cancelled logoff or restart ended the Host** and is **R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement | WP4 walkthrough evidence (B3b/B3c/B3c2 in both runs) |
| P0-D | Installer privilege and registration scope | **Yes** (R9) | Elevated per-machine TIP registration; Host per user at Medium IL (current baseline); implemented in WP4 (`install.ps1`/`uninstall.ps1` refuse to run unelevated) | WP4 walkthrough evidence (B3a/B3e/B3f) |
| P0-D | Medium-IL single session, user-SID pipe ACL | No — implemented baseline | Keep | — |
| P0-D | AppContainer/LPAC, High IL, x86, multi-session, RDP, FUS, runas | No — excluded | Each needs its own evidence when lifted | Per-scenario |
| P0-A | §7.3 recovery contract: user-visible disposition of an owned key / prior EFFECT that cannot reach its required terminal (Host crash, uncertain write, timeout fence) | **Yes for the release candidate** | Existing mechanisms must be shown to give explicit terminals (§8.2); the user-visible behaviour/notification is an owner decision (U7) on that evidence — no loss-of-input policy is accepted by this plan | **U7 decided 2026-10-19** (presentation: tray status, fault balloon, restart/open-log actions; no automatic restart). Recovery behaviour is observed through WP7 daily use; a reported recovery problem is assessed then (owner policy 2026-10-19) | WP7 daily use → a reported problem; U7 tray implemented in fusion-windows PR #26, hang detection pending |
| P0-A | Recovery key buffer / journal | No | Deferred | — |
| P0-B | Minimum recovery **representation**: what remains auditable/visible about unresolved responsibility (composition identity, committed prefix, uncertain EFFECT) after a failure | **Yes for the release candidate** (minimum only) | Owner: P0-A/P0-B owner; work pointer: the U7 tray/balloon implementation; trigger: U7 implementation (presentation decided 2026-10-19) — **the tray half is implemented in fusion-windows PR #26**; consequence: no release candidate without the accepted minimum representation/notification | U7 tray implemented (PR #26); the residual representation work continues under P0-A/P0-B ownership; no WP5 campaign |
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
| WP4 | **Packaging and lifecycle.** Package layout (TIP DLL; Host + runtime + `data/` + `lib/libime` + user-data location from WP3); elevated install/uninstall on the existing `register.ps1`/`unregister.ps1`/`check-state.ps1`; Host start per U2; upgrade while the DLL is loaded in running apps (stage, re-register, require app restart/sign-out); test-package delivery | WP3, U2 | WP3 exit | §7.3 startup / singleton / session-identity evidence; install → uninstall removes the TIP/COM/start entry and every file not in use (in-use version directories are reported and removed by a second uninstall after sign-out), user data per U1; upgrade evidence. **Status: CLOSED within its scope on 2026-10-19.** The 2026-10-18 run (`wp4-run-20261018-185004`) met the gate except B3c2; the 2026-10-19 run (`wp4-run-20261019-130037`) passes every step on the fixed packages. U2 (b) accepted: a per-user logon task starts the windowless Host at Medium integrity; the Host saves on session end through a hidden top-level window and is stopped orderly through a named per-user stop event. Two packages built from the branch head; B1 71/71 Mock-only, 79/79 Debug, 79/79 Release; A1–A6 evidence, the walkthrough results and its harness follow-ups in `docs/windows-stage6-wp4.md`. **Exit gate:** B3a, B3b, B3c, B3d, B3e, B3f and B4 (Notepad A–E on the installed wp4.2) **pass**; B3c2 **passes** in the 2026-10-19 run, as does the new cancelled-**restart** check (c3) — **R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement; the upgrade with a **running** old Host also **passes** (old Host stopped in order, save advanced, new Host from wp4.2) |
| WP5 | **Failure and recovery acceptance (R7, R22) — accepted through WP7 daily use.** No separate campaign: the owner reports a problem when it occurs, it is repaired by impact, and closing the daily-use period without reports is acceptance (owner decision 2026-10-19, §8.1) | WP4 (closed 2026-10-19); R30 implemented (fusion-windows PR #19: opt-in `FUSION_HOST_DIAGNOSTICS`, log in `%LOCALAPPDATA%\fusion-host\`) so a reported problem has Host-side evidence. R34 is **not** a dependency | WP4 exit **and R30 implemented** — both met; acceptance closes with the owner's WP7 daily-use closure | No separate evidence campaign; §8.2 is the reference for a reported problem |
| WP6 | **Application compatibility and positioning — accepted through WP7 daily use.** §8.3 is the reference for a reported problem; no matrix sweep (owner decision 2026-10-19, §8.1). U6 is decided and implemented in the WP7 preparation batch | WP4 (closed 2026-10-19); U6 decision | WP4 exit | Acceptance closes with the owner's WP7 daily-use closure |
| WP7 | **Daily use (current).** The owner uses the prepared package; problems are reported and repaired by impact; stability, resources and latency are observed incidentally (§8.4); no report is required when nothing goes wrong | WP4 (closed), R30 implemented, the daily-use package and restart tool from the WP7 preparation batch | WP4 exit | The owner closes the daily-use period; absence of reported problems is WP5/WP6 acceptance under the 2026-10-19 policy |
| WP8 | **Release candidate.** Signing/channel per U4 (pending decision); release notes with scope/exclusions; final identity manifest; independent review | WP7 daily-use closure (which carries WP5/WP6 acceptance), U4, **U7** (presentation implemented; hang detection is a separate pending follow-up and is **not** a WP8 entry condition) | all prior exits | Reviewable release candidate (not a public release) |

## 7. Owner decisions, scope requirements and positioning

**Next work after the minimal tray (owner decision 2026-10-20, D085; licence scope D086).** Two items remain scheduled ahead of anything else,
in this order: (1) the **official settings UI** implementation/integration and its owner-test package — delivered and packaged, with its
current status consolidated immediately below; (2) the **EXE installer** that packages it.
The feasibility prerequisite is closed by the three-round evidence and the bounded product implementation. The owner has authorized
GPL-2.0-or-later only for the separate upstream-derived `FusionSettings.exe` and derivative code, preserving upstream notices and
corresponding source; unrelated FusionHost/TIP/tray/repository code keeps its own component licensing. `fusion-windows` owns Host,
pipe, tray and package integration; reusable configtool adaptations live in the owner fork `choicky/fcitx5-configtool` branch
`windows/fusion-settings` (fork PR #2). The installer remains a single EXE, framework not chosen, and signing/channel remain U4.
The release candidate (WP8) keeps its entry conditions: U4 decided and WP7 daily-use closure. The installer item is the vehicle for
WP8 packaging, not a replacement for those conditions, and WP5/WP6/WP7 acceptance is unchanged. Product implementation and
owner installation/acceptance are separate states; no installation is claimed here.

**Settings UI — consolidated current status (2026-10-21; D085/D086, presentation superseded by D087).** This is the single current record.
**Direction change (2026-10-21, D087).** The owner has replaced the *presentation technology only*: the Windows settings surface is to be a
Fusion-owned **C# + WinUI 3** client with task-oriented navigation (Input methods · Keys · Candidates and input · About and diagnostics),
while official Fcitx configuration semantics, storage, defaults and the settings-before-installer order stay as they are. D086's scoped
licence remains attached to the retained upstream-derived executable and does not extend to the new client. The Qt client recorded below is
still the **shipped** settings surface — nothing has replaced it.

**WinUI 3 client — state by level (2026-10-21, second round).**

| Level | State |
|---|---|
| designed | done — navigation, option coverage, shared-state model (`evidence/settings-winui3-2026-10-21/migration-inventory.md`) |
| implemented | **core layer only** — `src/settings-winui3/FusionSettings.Core`: framing, codecs, endpoint naming, connection/handshake, draft/baseline model, key-chord and key-list model, key-editor state machine, input-method draft. The WinUI views are **not written**. |
| compiled | core **yes** (.NET SDK 10.0.402, zero package references); views **no** — `Microsoft.WindowsAppSDK` cannot be restored |
| tested | **215 checks pass**, of which 35 run against a real Host on an isolated pipe with `FUSION_FCITX_SKIP_USER_PATH=1` (read/apply/read-back, key-list round trip, shared-configuration claim, IM list and selection, incarnation stability, unknown-URI refusal) |
| packaged | no — a settings package without a settings executable would misrepresent the product |
| installed | no |
| owner-accepted | no |

The remaining blocker is one dependency, not the design: no reachable feed serves `Microsoft.WindowsAppSDK` and
`Microsoft.Windows.SDK.BuildTools` (`evidence/settings-winui3-2026-10-21/blocker.md`). The .NET SDK half of that record is
**resolved** — 10.0.402 is installed in the task-local toolchain with its SHA-512 verified against the official sidecar.

The second round also corrected the inventory against a running Host rather than the source alone: a `List|Key` value is an
indexed map (or the empty scalar), not a space-separated string; `NextPage`'s fourth default is `Next`, not `Page_Down` (one
keysym, two names); `[`/`]`/`;` arrive as `bracketleft`/`bracketright`/`semicolon`; `Behavior` exposes 12 children, not 17;
`CloudPinyin` is not in the runtime schema at all. Coverage is recomputed at 97 items. Separately, the **tray entry point** is **merged** (fusion-windows PR #30, `main` `504f48c`): left click opens the settings window or raises the
existing one; identity is the exact executable path, this session and the production endpoint, so an isolated `--pipe-name` instance or an
older copy is never raised; a found window whose foreground activation is refused is flashed instead of launching a second copy; and a
launch guard stops repeated clicks from stacking windows during startup. Right-click keeps the full menu and the launch path is unchanged.
**Validated at the unit level only** (lookup/decision/launch-guard tests plus a cross-process lookup against child processes); a real
installed tray click was **not** performed, and the change is not in an installed package. The three-round feasibility
investigation stays historical evidence in `docs/windows-settings-ui-feasibility.md`, and the per-correction detail lives in
`evidence/stage-a-global-settings-2026-10-20/` (`change-contract.md`, `correction-inventory.md`, `capability-mapping.md`, `handoff.md`).
Implementation, compilation, isolated validation, packaging, owner installation and owner acceptance are separate states; the first four
are claimed below, the last two are **not**.

**Delivered scope.** One Qt process (`FusionSettings.exe`) hosts the official schema-driven `ConfigWidget` over the existing product pipe
with Chinese Apply/Cancel/Reload, in three tabs. Pinyin and Shuangpin use the official per-input-method schemas. A third **Global** tab
renders `fcitx://config/global` and applies it with Fcitx's own sequence (`globalConfig().load(raw, partial=true)` → `safeSave()` →
`reloadConfig()`, mirroring `dbusmodule.cpp`), reusing the existing `GetConfig`/`SetConfig` messages and adding none of its own.
Input-method selection became list management over the methods the packaged runtime defines (pinyin, shuangpin): the ordered enabled
list (top row = `Items/0` = the method a bound context follows; `DefaultIM` is only the fallback), add from available, remove from
enabled, and reorder, written back through the official `currentGroup()`/`setGroup()`/`save()`. Removing an entry only drops it from the
group's list; it never deletes a method definition file or learning data. The empty list and any name outside the defined set are
refused by a pure product-side validator (`src/fcitx/im_list.hpp`) that closes the two gaps the core leaves open (`setGroup` silently
drops unknown names; nothing rejects an empty list) instead of persisting them. The list transport is four additive messages behind a
`settings_api_version` **1 → 2** bump (`SettingsGet/SetImListRequest`/`Response`, `0x15`–`0x18`), reusing the capability handshake,
one-request-per-connection, bounded payloads and Host-incarnation fencing already in place; a Host accepts any client API in `[1, 2]`
and refuses a newer one.

**Invariants the three tabs share.** Options an apply does not submit survive it (`merge_schema_values` recurses through nested
list/optional/external types, and the page submits only the top-level options it changed) — measured in this batch on a seeded profile;
keys the schema does not know are a different case and are **not** preserved, because Fcitx's own save writes the schema, and that was
measured too rather than assumed. A no-change Apply performs no setter call; a changed Apply and a list change refuse visibly while an
active composition or pending engine state makes them unsafe; unrelated groups and dictionaries/learning data are untouched; every
Fcitx access is scheduled on the Host event loop; and Reload re-reads **all three** pages — the earlier build reloaded only Pinyin and
Shuangpin, so Cancel and Reload left the Global page's edits and baseline untouched while reporting success. `External`/subconfiguration
actions are disabled, and Tray Settings plus a Start Menu shortcut ship in the package.

**Global options Windows does not honour are explained, not hidden.** The Global tab carries one sentence per category: global hotkeys
(trigger/activate/deactivate/page/candidate-move/`ModifierOnlyKeyTimeout`) — Fusion never hands a lone modifier or a Ctrl/Alt/Win
combination to the engine, and Ctrl+Space belongs to Windows input switching; input-status and panel options — this product has no
Fcitx panel, the candidate window is its own projection; password-field options — the Windows input-scope policy applies first;
`ActiveByDefault` and `resetStateWhenFocusIn` only partly. Linux-only options Fcitx already self-hides stay hidden, and the
supported-versus-inert mapping is recorded in the evidence rather than hard-coded into the widget.

**Keyboard boundary (corrected against source).** The owner's "Shift (or raw input) commits" request maps to the pinyin addon's
per-input-method `CommitRawInput` `KeyListOption` on `fcitx://config/inputmethod/pinyin` — not a global setting, and already rendered on
the Pinyin page. It makes `Shift+Return` and the plain `Return` family commit raw input; it does **not** make a lone `Shift` commit, and
**lone Shift is not implemented**. A bare modifier never reaches the engine, because modified keys are routed as Pass under the D072 §5
**UNFROZEN** "production effect-capable key classification". Numpad Enter arrives on this path as `VK_RETURN`, so the `KP_Enter` entries
in the key lists never match, and no `src/tsf/` change was made for them. `Control`/`Alt`/`Win`-modified keys share the same Pass
routing. Ordinary `Shift`+letter, key-up and modifier handling are unchanged, and no key-routing code changed in this batch. Changing
that classification is a D082 decision with evidence requirements, not this stage.

**Executed validation on Windows 7700.** Release build of `FusionSettings.exe`, `FusionHost.exe`, `FusionTray.exe`, `FusionTsf.dll` and
`FusionFcitxHost.dll` with MSVC 14.51.36231, SDK 10.0.26100.0, Qt 6.8.3 and the pinned `runtime-wp3-20261018`. `ctest` over
`stage_a_settings` (protocol/bounds/validator), `wp3_input_methods`, `rg0` and `stage2` is **37 of 37 passed**, with
`stage4a_ipc_integration` reported by CTest as **skipped**: that mode now requires an explicit isolated endpoint and connects to
nothing without one, because it types into whichever Host owns the endpoint it is given and the default endpoint belongs to the
installed product Host. The same test is **executed and passing** through `tools/settings/run-isolated-stage-a.ps1`, which starts a
Host on a unique pipe with a disposable profile and hands the endpoint over: `PASS integration with real Fcitx over IPC` and
`PASS integration candidate select committed the projected candidate`, in 0.4 s. Its earlier failure was that production endpoint, not a
product defect: against a controlled profile the unchanged assertions pass. The script also proves the safe default (no endpoint →
`SKIP`, exit 0), the refusal of this user's production endpoint in both the logical-name and full-path spellings before any request
(with the owner's learning files fingerprinted unchanged across the attempt), and the refusal of a malformed endpoint.

**Isolated session results.** In that one isolated session: the three pages load over the real pipe and a changed value applies and
reads back from the Host; an orderly restart returns the value, the enabled order (`shuangpin,pinyin`) and the pinyin learning files
from the profile; **Cancel** discards an unsaved Global edit, restores the Host's value and refreshes the baseline; **Reload** fetches a
value a second client changed on the Host behind the page (7 → 9 with the page dirty) and refreshes the baseline; switching tabs keeps
an ordinary unsaved edit; a no-change Apply leaves the profile byte-identical; empty and unavailable list entries are refused with "The
input-method list must contain at least one method this Host defines, and every entry must be available."; reorder → remove → add-back
ends on the reordered list. Preservation was measured on content, not existence: the seeded schema-known but unsubmitted
`ShowInputMethodInformation=False` and pinyin `SpellEnabled=False` both survived, while a key no schema knows
(`FusionStageAUnknownProbe`) was dropped by Fcitx's own save in both files; the engine's `user.dict` (339 B) and `user.history` (61 B)
were byte-identical (SHA-256) before and after the settings-only session. No production Host/tray/browser/profile was changed. The live
key-down/key-up test through a TSF-registered application was **not run**, so no keyboard claim rests on it.

**Package.** `fusion-0.0.0-settings+fe65d08.zip`, 60,085,623 bytes, SHA-256
`5db4ab21c80a9ffd6aa68c880312a1c68c4b1c5f1a6fb71f78869aecb1d95e5d`, built by the committed packaging script from the clean
committed product source `fe65d08621fb84f47ef83acfc3e50835217af462` with `sourceDirty=false`, the fork pinned at
`abc2c6644b030d761d3d5e7c27c888af019e63f0` (`windows/fusion-settings`, fork PR #2) and the pinned runtime identity. All 148 manifest
entries were re-verified after extraction, together with the Qt platform plugin, the executable/DLL identities, the zh_CN catalogs,
the non-nested `data\locale` layout and the notice/corresponding-source set. The package is **not installed and not registered**. It was
rebuilt rather than reused because this batch changes `src/settings/main.cpp`, which ships inside `FusionSettings.exe`; the previous
`fusion-0.0.0-settings+2e28a88.zip` (60,076,754 bytes) is retained for rollback, and `+1f4b749` stays superseded.

**Corrections in this batch.** `correction-inventory.md` records C1–C9: the list surface did not compile
(`QListWidget::addItem(QString)` returns `void`, not a `QListWidgetItem *`); Reload and Cancel skipped the Global page; unsupported
global options carried no explanation; the keyboard claims above were misstated; the packaging manifest named a stale fork commit and
the pin readers broke on a CRLF checkout; the patch files were EOL-converted on checkout; and the fork notice cited the wrong PR and
revision. The fork itself needed no new commit — `configlib` links only `launchexternal.cpp`/`logging.cpp`/`font.cpp`,
`configwidgetslib` calls only `GetConfig`, `SetConfig` and `PendingSignal::finished`, and no fork translation unit switches on
`Operation`.

**Still unsupported or unclaimed at this scope.** Lone `Shift`; numpad-Enter-specific handling; the D072 §5 modifier-held-key
classification; general addon and dictionary management; the single-EXE installer (D085 item 2); owner installation, owner acceptance
and owner daily use of this package; and any statement that the settings surface removes the configuration-file caveat in
`docs/windows-daily-use.md` before an owner has installed it.

### 7.1 Owner decisions (U1, U2, U3 and U5 owner-accepted 2026-10-18; the others are not assumed accepted)

| ID | Decision | Alternatives | Recommendation (with reason) | Consequence if undecided |
|---|---|---|---|---|
| U1 | **Implementation and data-lifecycle choices** for the required Pinyin/Shuangpin selection, configuration and learning persistence — not whether to keep those capabilities | user-facing selection method (Fcitx configuration file vs a minimal existing switch); storage location (Fcitx AppData standard paths vs an explicit Fusion location); upgrade (keep user data) and uninstall (keep vs remove vs ask) | **Accepted 2026-10-18.** Selection only through Fcitx's existing configuration files: the Fcitx profile's input-method list selects Pinyin or Shuangpin (`Items/0` is the method in use; `DefaultIM` is only the fallback), and the pinyin addon's own config holds the Shuangpin scheme and pinyin options — no Fusion-specific file, switch or registry key. Storage in Fcitx's own standard per-user paths as the pinned Fcitx resolves them (`%APPDATA%\Fcitx5\{config,data}`); no Fusion-specific data location. Upgrade keeps user data; uninstall leaves it unless the user chooses removal (WP4 implements uninstall). **Selection method partially superseded 2026-10-20 (D085):** an official settings UI is now scheduled and is a front end to these same files; the configuration model, storage paths and user-data policy above are unchanged | Decided — WP3 closed within its scope; WP4 implements the uninstall choice; the settings UI (D085) reads and writes the same Fcitx configuration |
| U2 | Host start mechanism — candidate approaches: (a) per-user logon **Run** entry; (b) **scheduled task** at logon; (c) another minimal mechanism if evidence requires it | (a)/(b)/(c) | **Accepted (b) 2026-10-18.** A per-user scheduled task at logon: principal = the installing user, interactive token, `RunLevel Limited` (Medium integrity, never elevated), no execution time limit, `MultipleInstances = IgnoreNew`, no restart-on-failure. The task starts the Host; the DLL still never starts it. The windowless Host saves on logoff/shutdown through a hidden top-level window, and install/upgrade/uninstall stop it through a named per-user stop event instead of terminating it (`docs/windows-stage6-wp4.md`, A2/A4). (a) is **not** implemented and its gaps are recorded: an elevated installer cannot start a Run entry non-elevated for the current user without a sign-out, and nothing restarts a Run entry that exits. Neither (a) nor (b) restarts a crashed Host or recovers an alive-but-unhealthy one (R7) — that stays WP5 evidence. **Verification:** B3b (a second start is refused) and B3c (learning survives sign-out/sign-in) **pass** in both runs; logon autostart is verified (the task started the Host at 18:58:43 and again at the 2026-10-19 sign-in); the **cancelled-logoff** and **cancelled-restart** halves **pass** in the 2026-10-19 run, (c2) and (c3) — **R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement. The decision is unchanged | Decided — WP4 implements (b); U4 pending; U6 decided (implementation in fusion-windows PR #20, real-window confirmation pending); U7 tray implemented (fusion-windows PR #26, head `0094269`), hang detection a separate pending follow-up |
| U3 | Diagnostics in packages that leave the local developer machine | (a) off by default, opt-in, no key/text content; (b) on without content; (c) unchanged | **Accepted (a) 2026-10-18.** (a) — removes keystroke-adjacent data and disk growth; local controlled development diagnostics stay available | Decided — no longer a WP2 gate; distribution stays gated by WP4/U4 |
| U4 | Signing, distribution channel, test package vs release candidate | certificate available or not; channel | **Deliverable format decided 2026-10-20 (D085):** the installer is a single **EXE** (not an MSI, and not a zip plus a script the user runs); the installer framework is **not** chosen. Owner-only test packages first; **still undecided: signing and the distribution channel**, both settled before any external release candidate | No external release candidate; no installer framework is assumed before this row is decided |
| U5 | Password/private InputScope minimum policy | (a) fail open (TIP does not own keys) for password/private scopes; (b) pass `Password`/`Sensitive` to Fcitx | **Accepted (a) 2026-10-18.** (a) for the first packages — smallest change, no composition/learning in password fields; (b) remains the faithful long-term path (§25 row). Known limitation, accepted with this decision: in the Edge and Firefox runs the minimum outcome came from the existing `GUID_COMPARTMENT_KEYBOARD_DISABLED`/empty-context hard-policy gate, and the InputScope probe read Unknown in Notepad, Edge and Firefox, so the Sensitive-classification path is not demonstrated and a field that neither trips that gate nor yields a readable InputScope is unverified. The R11 follow-up (bounded validation on one minimal case with a known sensitive InputScope) remains open and is not scheduled by this record | Decided — no longer a WP2 gate; distribution stays gated by WP4/U4 |
| U6 | IME open/close default for a new app context | (a) keep Windows' compartment value; (b) declare open on activation (Activate already writes `KEYBOARD_OPEN=1`, yet run 1 observed closed) | **Decided (b) 2026-10-19: a newly opened application window starts with Fusion open; an explicit Ctrl+Space close stays effective, before or after the first key.** Evidence: the compartment reads open at activation and closed at the first key of a new context (fusion-tsf log, 2026-10-19); **attribution is a hypothesis** — something re-initialises it in between (the application's context initialisation or Windows), and the current evidence does not separate them. Implementation: declare it open once per **new document manager** (the lifecycle event for a new window), never from a key path and never per focus change, so a close is never overwritten | **Implementation status: in fusion-windows PR #20.** **Validation status: real-window confirmation pending** the owner's first installed window check; no claim of verified behaviour | Compartments stay Windows authority (§10); the change only declares the default for a new window |
| U7 | P0-A/P0-B minimum for the release candidate: user-visible disposition/notification of unresolved responsibility after a failure | decided on WP5 evidence | **Presentation decided 2026-10-19:** a persistent tray status, a fault balloon notification, and actions to restart the Host and open its log directory; no focus stealing, no input content, no repeated per-key notifications; automatic restart is **not** authorized. The minimal design is in `docs/windows-daily-use.md` §4 | **Piece (1) IMPLEMENTED** in fusion-windows PR #26 (head `0094269`, after the independent-review correction batch on top of `2f9c178`): the tray process, one balloon per `running → absent` transition, Restart FusionHost / Open log folder / Exit, one instance per user × session, its own `FusionTray` logon task, and install/upgrade/uninstall/restart-tool integration. The review batch scoped the tray inventory and shutdown to this user × this session × `<install root>\<version>\bin\FusionTray.exe` (a same-named executable elsewhere is reported and left alone), made a cancelled session end resume reporting and gave the tray task the same User32 1077/1073 cancellation trigger the Host task uses, refreshed the tooltip on a Host pid change without reporting an absence, and stopped an uninspectable Host process from becoming a verified absence. Validated with isolated test processes and a stubbed restart tool (40/40 harness checks, `tray_state` unit test with 12 scenarios, CTest 80/82 with the two known Host-singleton failures unchanged). **Installed on 2026-10-20 as `0.0.0-wp7+0094269`: the installation succeeded and the owner confirmed the hover tooltip `Fusion: Host running (pid 41396)`.** The menu commands, the `running → absent` balloon, the session-end suppression and the installer path were **not** exercised, so they remain unobserved end to end. **Piece (2) hang detection (alive-but-unresponsive) is pending:** it needs an additional probe and is **not** part of the minimal version, and it is **not** a WP8 entry condition (owner: Stage 6 implementation; trigger: a reported hang, or a later batch that takes it up). The next work after this batch is the official settings UI, then the EXE installer (D085) |

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
covered by tests (`HostClaim` refused a second Host; `stage4a wp3singleton`, `wp4stopevent`); the session-end decision
is covered by `stage4a wp4sessionend`. **Result of the 2026-10-18 run (`wp4-run-20261018-185004`):** the second-start
half (B3b) and the sign-out/sign-in half (B3c) **pass**; the logon half is **verified** (the task started the Host at 18:58:43 and again at the 2026-10-19 sign-in); the **cancelled-logoff** half (B3c2) **failed in the 2026-10-18 run** — the logoff was cancelled (User32 1077) and the Host had already ended — and **passes in the 2026-10-19 run**, which also adds a cancelled restart as (c3) (**R34** RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement). Nothing restarts a crashed Host (no
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
  every packaged candidate; WP3 input-method/user-data exits; WP4 lifecycle evidence; identity verification of the DLL
  actually loaded by the target process — which means resolving the process that **owns the target's window** rather than
  the PID a launcher returned, and closing every window of that application before asserting a fresh load (R31;
  `docs/windows-stage6-wp4.md`, "Walkthrough harness rules (carry forward)").
- **Daily-use acceptance (owner decision 2026-10-19, superseding the separate WP5/WP6 campaigns):** WP5 (failure and
  recovery) and WP6 (application compatibility) are accepted through **WP7 daily use**: the owner reports a problem when
  one occurs, it is assessed and repaired by impact, and **no report is required when nothing goes wrong**; closing the
  daily-use period without reported problems is sufficient acceptance. §8.2 and §8.3 are the **analysis reference** for a
  reported problem, not a mandatory campaign: no deliberate crash/hang injection, no application-matrix sweep and no
  pre-WP8 fault-test session is required, and unexecuted scenarios must never be recorded as PASS. Positioning (§7.4) and
  stability/latency (§8.4) are observed incidentally; a reported positioning or latency problem is investigated then.
- **Conditional:** exact-caret implementation (R14); DPI fix (R15); recovery changes if a reported problem shows missing
  terminals or an R7 occurrence; P0-D evidence when an exclusion is lifted.

### 8.2 Failure and recovery reference (WP5, existing mechanisms)

**This section is the analysis reference for a problem reported during WP7 daily use, not a mandatory test campaign**
(owner decision 2026-10-19; §8.1). Each scenario records, per key and per prior EFFECT, the disposition from the target process's diagnostics and the
Host log, plus the user-visible document state. Required invariants in every scenario (arch §7.3–§7.5, §20):
explicit terminal disposition for every owned key; no cancelled key represented as a genuine PASS; consumed STATE →
EFFECT watermark and causal order preserved (no later key overtakes an unresolved prior EFFECT); no blind replay or
retry after an indeterminate mutation; no duplicate irreversible EFFECT; no cross-context EFFECT; no stale/late result
applied after incarnation/connection fencing; no silent loss of an owned key.

**Capturing the Host log (R30, implemented in fusion-windows PR #19).** Set `FUSION_HOST_DIAGNOSTICS=1` (user scope, e.g.
`setx FUSION_HOST_DIAGNOSTICS 1`), restart the Host so it reads the opt-in at startup, run the scenario, then collect
`%LOCALAPPDATA%\fusion-host\` (the newest `fusion-host-<pid>.log`; five files are retained). The log is technical only —
no key or text content (U3) — so it supplies the Host-side disposition and timings while the target process's own
diagnostics supply the document-side half.

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

**Reference for classifying a reported compatibility problem during WP7 daily use; not a mandatory sweep** (§8.1).
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

### 8.4 Stability, resources, latency (observed incidentally during daily use)

**No mandatory benchmark or monitoring project** (§8.1): daily use is the observation. A reported stability, resource or
latency problem triggers the bounded measurement below, using the existing opt-in diagnostics.
- **Soak:** if a soak is ever needed for a reported problem, one continuous scripted session of representative input across
the matrix apps plus one sign-out/sign-in. **Decision fed:** R7 recovery need, U7.
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
- ROADMAP: Phase 8 points here as "Stage 6 plan proposed (not accepted); WP1 closed; WP2 closed within its scope (U3/U5 decided (a) 2026-10-18); WP3 closed within its scope (U1 accepted 2026-10-18); WP4 closed within its scope on 2026-10-19 (exit gate passed in run wp4-run-20261019-130037; R34 RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement), R30 implemented in fusion-windows PR #19, WP7 daily use is the current direction (WP5/WP6 accepted through daily use, D084), U6 and U7 decided (U2 (b) accepted 2026-10-18); U7 tray implemented in fusion-windows PR #26; R25 recovered, owner-confirmed 2026-10-20".

**WP1 delivery accounting (2026-10-17):** changed register entries — R1, R2, R3 RESOLVED on the rebuilt runtime
(observed; R1 old-binary provenance UNKNOWN; R3 guard-B attribution plausible, not isolated); R16 partially informed
(Release product path stays WP2); R22 gains incidental hung-Host/residual-text evidence for WP5/U7; R23 new
(CONDITIONAL, tooling). All other R-entries (R4, R5a–R5c, R6–R15, R17–R21), U1–U7 and the P0-A/P0-B/P0-D
subquestions (§5) are unchanged by WP1 and keep the status, owner, trigger and authority recorded above. WP1
boundaries kept: the Notepad A–E run used the existing Stage 5 registered TIP with the rebuilt Host/runtime; WP2 is
not started. Reverse-omission for WP1: no discovery without scheduled work (the PS 5.1 script defect is R23, the
hung-Host evidence is under R22); no hypothesis dropped (R3 attribution and old-binary provenance kept open as
stated); nothing resolved still marked unresolved; ROADMAP Phase 8 reflects WP1.

**WP4 delivery accounting (2026-10-18):** changed register entries — R6 ADDRESSED (implemented; B3b/B3c **pass**, logon
autostart passed once at 18:58:43, and B3c2 **failed** — a cancelled logoff ends the Host, R34); R9 ADDRESSED
(implemented; B3a/B3e/B3f **pass**); R23 FIXED and verified in isolation;
R26 ADDRESSED for the stop-event and session-end paths (implemented and locally evidenced), with its sign-out/sign-in
half **passing as B3c** and the cancelled-logoff half **failing** (R34 — the handler is right, the survival assumption
was wrong); R28 new
(DOCUMENTED limitation: a previous version directory stays while an application holds its DLL). U2
moved from open to Accepted (b) with (a)'s gaps recorded. U4, U6, U7 and R7, R11 (follow-up), R12–R22, R24, R25, R27
are unchanged and keep the owner, trigger and authority recorded above. The P0-A/P0-B subquestions (§5) are unchanged;
P0-D gains the implemented startup baseline (arch §14) and stays OPEN for the excluded scenarios. WP4 boundaries kept:
no MSI/signing/channel (U4), no restart-on-failure (WP5/U7), no PATH change, no force-delete, and no change to Fcitx
semantics or TSF admission.

**Consolidated repair accounting (2026-10-18, PR #17 head `9c7ceec`):** register entries added or changed — **R29**
(development identity, DEFERRED to WP8/U4), **R30** (no opt-in Host log; was REQUIRED before WP5 and is an explicit WP5
entry dependency — implemented in fusion-windows PR #19), **R31** (target-process identity verification; a documented rule for WP6/WP7 harnesses, also stated
in §8.1), **R32** (multiple accounts; CONDITIONAL), **R33** (build environment; CONDITIONAL tooling). **R15** keeps its
REQUIRED check and gains the owner's 175 %/one-monitor observation as evidence; it is not closed. **R28** now names the
real holders (shell processes, because the input method is global) and adds the `protocol_version` constraint, and
**R31** carries the same caveat (the target process may already hold an older DLL). The final round (PR #17 head
`e85d497`) adds no register entry: the hash-skip copy and the per-holder reporting implement R28's disposition, and the
corrected (a)/(d) prompts remove a question the 17:07 Host had already invalidated. No other entry changes: R6/R9/R26
stay ADDRESSED, and WP4's status is the walkthrough result below.

**Walkthrough accounting (2026-10-18, run `wp4-run-20261018-185004`):** B3a–B3f and B4 pass except **B3c2**, which failed: a cancelled logoff ends the Host and nothing restarts it — **R34** (event trigger on User32 1077 or 1073). The upgrade with a running Host was not exercised because the cancelled logoff had ended the Host.
*Corrected 2026-10-18 by the event-log evidence (`r34-evidence.txt`): the first version of this paragraph said autostart
failed at a "second sign-in" and that B3c2 was not exercised. There was no second sign-in — logon autostart passed once
at 18:58:43 — and B3c2 was exercised and failed.*
*Extended 2026-10-19: the owner measured which event each cancelled attempt writes (1077 logoff, 1073 restart and shutdown, none for a completed restart), so R34's trigger covers all three; the R34 walkthrough verifies the logoff in (c2) and adds a cancelled restart as (c3), and the shutdown rests on that measurement. Three harness defects found while adding (c3) — the blocker's second start, the probe's missing helper and collection mode, and the wrong WTS logon-time field — are fixed and proven in the same PR.*

**Walkthrough accounting (2026-10-19, run `wp4-run-20261019-130037`):** every step **passes** on the fixed packages — (a) install (task triggers `Logon;Event`, subscription `User32 (1077 or 1073)` scoped to the user's SID, Host Medium in session 1, and the two leftover directories from the 2026-10-18 run removed), (b) a second start refused, (c) sign-out/sign-in with the learning retained, (c2) cancelled logoff, (c3) cancelled restart, (d) upgrade with a **running** Host, (e) uninstall keeping data, (f) reinstall + `-RemoveUserData`, and the development-registration restore. WP4 is **CLOSED within its scope**; **R34** is RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement. The 2026-10-18 results stay above as history.

Reverse-omission for WP4: no discovery without scheduled work (the locked-directory behaviour is R28, the Run-entry gaps are recorded under U2, the cancelled-end failure is R34, and R29–R34 each carry a disposition and a trigger); no hypothesis dropped (the 2026-10-18 item the run could not exercise — the upgrade with a running Host — is marked "not exercised" there and **passed** in the 2026-10-19 run); nothing resolved still marked unresolved (R23 closed, R6/R9/R26 verified by the 2026-10-19 run, R34 RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement); ROADMAP Phase 8 reflects WP4 closed, R30 implemented and WP5 as the next work item.

**R30 delivery accounting (2026-10-19, fusion-windows PR #19):** R30 moves from REQUIRED-before-WP5 to **IMPLEMENTED**;
its entry condition for WP5 is met once the PR merges. U3 (a) is preserved and made checkable: off by default (a unit test
asserts no directory and no file without the opt-in), technical fields only, and the TSF privacy-marker redaction applied
as a safety net. The Fcitx-log exclusion is a recorded owner decision (2026-10-19) with its own owner (Stage 6
implementation) and trigger (a WP5 scenario that needs Fcitx's own log), so it is scheduled rather than dropped. Nothing
else changed: U4 stays open, WP5/WP6 are accepted through WP7 daily use, and no other register entry moved.

**WP7 preparation accounting (2026-10-19, fusion-windows PR #20):** the owner's daily-use policy is recorded as D084 and
applied here: §8.1 gains the daily-use acceptance rule; §8.2/§8.3 are reclassified from mandatory campaigns to the
analysis reference for a reported problem; §8.4 becomes incidental observation; WP5/WP6/WP7/WP8 rows, R7/R17/R22 and the
status line follow. U6 is decided and implemented (the compartment is declared open at activation and once per new
document manager, released when that manager is destroyed; real-window confirmation is the owner's first installed
check), U7 presentation is decided with the tray implemented in fusion-windows PR #26 and installed as
`0.0.0-wp7+0094269` (the owner confirmed the hover tooltip; the menu commands and the balloon were not exercised;
hang detection pending) and its
minimal design in `docs/windows-daily-use.md` §4. The daily-use package, the restart tool and the diagnostics opt-in/out are in
PR #20; nothing was installed by this batch, and no acceptance is claimed before the owner closes the daily-use period.

**U7 correction accounting (2026-10-20, fusion-windows PR #26 head `0094269`):** an independent review of the tray batch
raised six findings; this batch closes four in code and two in documentation. Code: the tray inventory and shutdown are
scoped to this user × this session × `<install root>\<version>\bin\FusionTray.exe` (any version, so an upgrade can stop
the previous tray; a same-named executable elsewhere is reported and never signalled), `WM_CLOSE` goes only to a window
owned by a verified pid, and identity is re-read before any forced stop; a cancelled session end
(`WM_ENDSESSION` with `wParam` 0) clears the suppression and re-baselines, and the tray task now carries the same User32
1077/1073 cancellation trigger the Host task uses, so a tray that had already exited for a cancelled session end is
started again; a Host pid change refreshes the tooltip without reporting an absence; and a Host process that cannot be
inspected makes the observation unresolved instead of a verified absence. Documentation: the claim that hang detection is
a WP8 prerequisite is removed (it stays a separate pending follow-up), the tray's implementation head and validation
numbers moved here and out of D084, the daily-use statement that uninstall restarts the tray is corrected (uninstall
removes it), and D085 records the owner's next-work decisions. Validation for this batch: `tray_state` 12 scenarios,
the isolated harness 40/40 (scoped inventory with a same-named decoy left running, window ownership, `WM_ENDSESSION` 0
and 1, trigger equality with the installed Host task, marker lifecycle), and CTest 80/82 with the two known `stage4b_*`
Host-singleton failures unchanged. The tray was then **installed on 2026-10-20 as `0.0.0-wp7+0094269`**: the
installation succeeded and the owner confirmed the hover tooltip `Fusion: Host running (pid 41396)`. The menu
commands, the `running → absent` balloon, the session-end suppression and the installer path were **not** exercised,
so they remain unobserved end to end.
