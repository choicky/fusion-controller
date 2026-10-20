# Roadmap

> This file is the compact delivery map. Durable requirements are in `docs/REQUIREMENTS.md`; accepted decisions are in `docs/DECISIONS.md`; `docs/first-release-baseline.md` is the frozen D068 first-release matrix, not the post-release current-next authority. Detailed execution evidence remains in the referenced evidence/checkpoint files and Git history.

## Current state

The first Fusion Enhanced release is published. This table records post-D068 current state; the frozen D068 release matrix remains in `docs/first-release-baseline.md` (D060/D066/D068).

| Area | Current status | Durable pointer |
|---|---|---|
| MoQi Auxiliary Filter | COMPLETE / productized. Auxiliary Filter only; Pinyin/Shuangpin and LibIME remain the input path. Fixed MoQi table: `gaboolic/moqima-tables@6d8ba8f1c57466f358e682baefe11bbd0fe389ab` / `moqima_gb18030.txt`. | D002–D008, D024, D052–D053 |
| Voice core and UI | COMPLETE / device accepted. Mic and optional Space trigger share one Voice flow; Toolbar and Voice Settings regression/defect closure is complete. D081's bounded capture-duration policy is an accepted policy; its Android implementation is not yet in the shipped product and lands via the paired Android PR #16 (merge order: fcitx5-android PR #16 → this fusion-controller PR #17). | D012–D018, D026–D031, D046–D049, D055–D057, D081 |
| Local ASR | COMPLETE / DEVICE VERIFIED. Priority is X-ASR offline INT8 → X-ASR 960 ms streaming INT8 → FunASR Nano. All three are production/recommendation-eligible; core release-device lifecycle and Model Manager abnormal/error paths are complete. | D050, D056, D059 |
| Managed Cloud ASR | COMPLETE for current scope: Doubao, Qwen, Tencent. Basic real-service/device recognition is accepted; no broader provider expansion is planned. | D028–D033 |
| D035 fallback | COMPLETE / DEVICE VERIFIED for the core gate: selected external early technical/service failure before session establishment may fall back once to eligible Local. System, quality failures, chaining, and PCM replay remain excluded. | D035 |
| Self-hosted ASR | IMPLEMENTED boundary; device checkpoint DEFERRED and non-blocking. | D033, `docs/network-asr-checkpoint.md` |
| Text post-processing / LLM | Optional provider-neutral seam implemented; configured processor is null/disabled. Tested Local-LLM route STOPPED after quality failure; Cloud LLM is not implemented/planned. | D017, Android PR #15 closure evidence |
| Dictionary Manager | Productized through Phase 5D and included in the release line. Preserve the catalog/version, staged update, atomic replacement, failure safety, and enabled-state boundaries. | D039–D040, D080 |
| Mixed Input | M2+/r4 is the accepted architecture baseline. Phase 11 productization and Android integration are VERIFIED / COMPLETE; A' and earlier Architecture A/A' validation are historical/reference. | D069–D075, Phase 11 evidence |
| Toolbar / punctuation | Current product semantics and release boundaries are settled; old implementation transcripts are historical. | D047–D051, D066, D080 |

## Completed phases

### Phase 0 — Requirements and architecture
COMPLETE. Android-first scope, Pinyin/Shuangpin, Auxiliary Filter, Voice, ASR/LLM separation, privacy boundaries, and minimal-fork principles are captured in `REQUIREMENTS.md` and D001–D021.

### Phase 1 — Upstream research
COMPLETE. Fcitx5/LibIME candidate, filter, Android configuration, and SpeechRecognizer boundaries were established. No LibIME fork or parallel decoder was accepted.

### Phase 2 — MoQi Auxiliary Filter V1
COMPLETE. Shared Stroke/MoQi infrastructure, separate predicates, selection-frontier semantics, partial selection, composition preservation, continued typing, and fixed-table behavior are product boundaries. Detailed evidence: Phase 2/3 history and D002–D008.

### Phase 3 — Android MoQi integration
COMPLETE. Configure-time fixed-table retrieval and config-component packaging are productized; Android product path and release behavior were accepted. Upstream contribution work is closed/non-blocking. Detailed evidence: `docs/phase3-device-verification.md`, D024, D052–D053.

### Phase 4 — Voice input
COMPLETE. VoiceBackend, shared session lifecycle, Mic/Space triggers, provider selection, Local Model Manager, Managed Cloud adapters, D035, Toolbar/Voice Settings, and release/runtime closure are complete at the core-gate level. Extended validation debt is not a current product task.

### Phase 5 — ASR provider/productization
COMPLETE for current product scope. Local order, fixed-upstream download policy, provider-specific BYOK, D035 bounded fallback, and current Managed Cloud providers are settled. Self-hosted device testing is deferred rather than a release blocker.

### Phase 5D — Dictionary Manager
COMPLETE. Productized install identity, update detection, staged download/checksum, atomic replacement, failure/cancel preservation, migration, and compatibility gate are accepted in D080.

### Phase 6 — Optional LLM post-processing
STOPPED/DEFERRED. The seam remains available, but no concrete processor is configured. The tested Local-LLM route is closed; no Cloud LLM task is current.

### Phase 7 — Android product integration
COMPLETE for the accepted Fusion Enhanced release/product line. Product boundaries and release identity are governed by D054, D058, D063–D068, and D080.

### Phase 8 — Additional platforms
Windows is a separate research/implementation track governed by D072, D082, D083 and `docs/windows-architecture.md`; it is outside this Android/Fusion roadmap. It pursues an advanced Windows-native architecture and a faithful Fcitx5 port together: Fcitx remains the default semantic authority, Windows owns TSF adaptation and reliability, and departures from mature Fcitx behavior need D082 evidence. Stage 3C (faithful Fcitx frontend path) is VERIFIED / COMPLETE via merged fusion-windows PR #7. Stage 4 — production out-of-process Host and TSF lifecycle integration — is COMPLETE / VERIFIED via merged fusion-windows PR #8 (4A), #9 (4B) and #10 (4C); its closure scope, the real-product-path evidence and the explicitly not-claimed multi-application re-verification are recorded in `docs/windows-architecture.md` §22/§25 and D083. Stage 5 (composition/candidate UI) is VERIFIED / COMPLETE within the scope recorded in `docs/windows-architecture.md` §25: fusion-windows PR #11 (first candidate projection/action seam, `main` `3e7fc35`) and PR #12 (faithful preedit projection, aux/panel projection with whole-InputPanel hide, candidate comment presentation, and the candidate popup pointer-delivery fix) are merged, with PR #12 merge `0f724af` carrying exactly the tree of the independently reviewed, exact-HEAD Notepad A–E-smoked candidate `670dfc0`; local validation was Mock-only 64/64 and real-Fcitx 70/72 with two documented pre-existing baseline aborts; fusion-controller PR #29 is merged. The closure is scoped to the Notepad product path; multi-application compatibility, exact-caret popup anchoring and the other deferred or separate-decision Stage 5 capabilities keep the owners and triggers recorded in §25, and the reported stroke-filter MSVC Debug assertion has since been independently reproduced on the old pinned runtime and does not reproduce on the Stage 6 WP1 rebuilt runtime (see below). `research/windows-stage5-candidate-contract-inputs.md` remains research input, not an accepted contract. The next stage is Stage 6 (packaging / release / soak): a plan is proposed in `docs/windows-stage6-plan.md` (not accepted). WP1 (pinned-runtime compatibility) is merged and closed within its accepted scope (R1–R3 no longer reproduce on the rebuilt runtime; fusion-windows PR #13). WP2 (Release configuration, default-off diagnostics, bounded InputScope admission policy) is merged (fusion-windows PR #14). Its final-candidate technical checks are complete: Notepad A–E, and the Edge and Firefox password checks for the U5 minimum outcome. The Firefox diagnosis confirmed R24, which is fixed by merged fusion-windows PR #15. WP2 is closed within its scope after the owner's U3 (a) / U5 (a) decisions (2026-10-18). WP3 (input methods and user data, R5a–R5c) is closed within its scope after the owner's U1 decision and product-path walkthrough: selection only through Fcitx's own configuration files, storage in Fcitx's standard per-user paths, learning and options persist across an orderly restart, verified on the data-complete runtime `runtime-wp3-20261018` (fusion-windows PR #16, `docs/windows-stage6-wp3.md`). WP4 (packaging and lifecycle, R6/R9/R23/R26, U2) is **CLOSED within its scope on 2026-10-19** (`feature/stage6-wp4-packaging` — U2 (b) accepted: a per-user logon task starts the windowless Host at Medium integrity, which saves on session end and is stopped orderly through a named per-user stop event; two owner-only test packages, B1 71/71 Mock-only / 79/79 Debug / 79/79 Release, A1–A6 evidence and both walkthrough runs in `docs/windows-stage6-wp4.md`). The 2026-10-18 run (`wp4-run-20261018-185004`) met the exit gate except B3c2, which failed — a cancelled logoff ended the Host and nothing restarted it; the 2026-10-19 run (`wp4-run-20261019-130037`) passed every step on the fixed packages, including the cancelled logoff (c2), the new cancelled restart (c3) and the upgrade with a running Host. **R34 is RESOLVED — verified 2026-10-19 (run `wp4-run-20261019-130037`): cancelled logoff (c2) and cancelled restart (c3) restore a Host in the same session; cancelled shutdown covered by the 1073 measurement.** **R30 is IMPLEMENTED** (fusion-windows PR #19: opt-in by `FUSION_HOST_DIAGNOSTICS`, a technical-only log in `%LOCALAPPDATA%\fusion-host\` with a 2 MiB cap and five-file retention; Fcitx's own log excluded by owner decision 2026-10-19). **Current direction: WP7 daily use** (owner decision 2026-10-19, D084). WP5 and WP6 are accepted through daily-use experience and owner closure — a reported problem is assessed and repaired by impact, and no report is required when nothing goes wrong; no separate fault-injection or application-matrix campaign is required, and no pre-WP8 crash/hang session remains mandatory. The WP7 preparation batch provides the daily-use package, the manual Host restart tool (Start Menu shortcut) and the explicit Host-diagnostics opt-in/opt-out (`docs/windows-daily-use.md`). **U6 is decided** (a new application window starts with Fusion open; an explicit Ctrl+Space close stays effective) and **implemented** in fusion-windows PR #20; **real-window confirmation is pending** the owner's first installed window check. **U7 presentation is decided** (tray status, fault balloon, restart/open-log actions; no automatic restart) and **the tray is implemented** in fusion-windows PR #26 (head `0094269`, after the independent-review correction batch): one icon per user × session, one balloon per `running → absent` transition, Restart FusionHost / Open log folder / Exit, its own logon task, and install/upgrade/uninstall/restart-tool integration — built and validated with isolated test processes and a stubbed restart tool, without installing it, so the balloon, the session-end suppression and the installer path are **not** yet observed end to end; **hang detection remains a distinct future piece, pending, and is not a WP8 prerequisite**, with its owner and trigger in the plan's U7 row. **The next work is decided (2026-10-20, D085):** the **official settings UI** first, reusing upstream Fcitx's settings UI and configuration semantics where feasible (Windows dependencies and Host communication to be inspected; feasibility not proven), then the **EXE installer** that packages it (a single EXE; framework not chosen; signing and channel still undecided under U4). **First daily-use sessions (2026-10-19): three owner-reported incidents.** Notepad-- 3.4 crashed on Win+Space before any key because the TIP bound to that application's own older `MSVCP140.dll`; cause confirmed by WER records and a reproduction at the same fault offset, fixed in fusion-windows PR #21 and **owner-confirmed** on the repair package (R35 — that confirmation covers the activation crash, not universal Notepad-- compatibility). Idle Enter in the same application then inserted no newline: the shell claimed a key the engine only forwards and answered `eaten=FALSE` in `OnKeyDown`, so an application that treats the test callback as the decision inserts nothing; fixed in fusion-windows PR #23 (merged as `main` `2d9db81`) and **owner-confirmed on 2026-10-20 with `0.0.0-wp7+24914ff`** (R36). The browser candidate popup (R25) is **RECOVERED — owner-confirmed on 2026-10-20** with the installed `0.0.0-wp7+995e5a5` (source `995e5a5`): after the browsers were restarted with that package, the owner reported "firefox 和 edge 目前正常", which closes the persistent candidate-invisibility incident at current-use scope (not a per-field, per-site guarantee). Measured on 2026-10-20 with the diagnostics PR #24 landed: Firefox answers `TS_E_NOLAYOUT` for the composition range on every keystroke, which hid the popup; fusion-windows PR #25 falls back to the caret range that the same context does answer, and in fresh isolated processes with that package the popup appears — from the first keystroke in the Firefox and Edge address bars, from the second in a page textarea and the Gemini composer. The owner's earlier 2026-10-20 field table (Gmail compose, ChatGPT editor, Gemini editor, all Edge fields) described browser processes that had never loaded that package and were still running older builds, one of which was built from source with no candidate-window code at all; it is retained as a measurement of those builds and its individual cells were not re-checked. In web content the first keystroke of a composition still cannot be anchored — a one-key gap, not an explanation for a popup that never appears; the optional capture in `docs/windows-daily-use.md` §4 stays available for a field that misbehaves. The next work is authorized and bounded (D085): the official settings UI, then the EXE installer that packages it; nothing else is promoted ahead of those two. U4 stays open; P0-A/P0-B/P0-D remain OPEN.

### Phase 9 — Mixed Input Architecture A
HISTORICAL / SUPERSEDED as the current architecture. Preserve stable boundaries and evidence, but do not treat Architecture A/A' device smoke or old candidate text as current work.

### Phase 10 — M2+ architecture evaluation
COMPLETE. HR1 held-out evaluation and the M2+/r4 decision are closed by D074/D075. No new architecture competition, r5/r6 experiment, or A' parity project is current.

### Phase 11 — M2+/r4 productization and Android integration
VERIFIED / COMPLETE. The accepted r4 behavior, bounded mixed search, separate Chinese/English sourcing and learning ownership, alignment, selection frontier, Pinyin/Shuangpin support, and LibIME-unmodified boundary are preserved. Productized addons and the merged Android integration are the current baseline.

## Current next

There is no new product implementation task created by this cleanup. The only explicitly open/deferred items are:

- Self-hosted real endpoint/device checkpoint, deferred and non-blocking until suitable environment/conditions exist.
- Optional realtime partial-transcript preedit UX, future and not a current requirement.
- Larger or exhaustive mixed-input corpora / cross-English-LM research, future unless a specific accepted gate reopens it.
- Release- or feature-specific work only when authorized by a new decision or concrete regression; do not reopen closed Local, Model Manager, Voice, Toolbar, D035, Qwen/Tencent, MoQi, Dictionary Manager, or Phase 11 gates.

These are not release blockers. Stopped Local-LLM work, generic Voice revalidation, generic Managed Cloud expansion, MoQi upstream contribution, and old Architecture A/A' or Android-integration “NEXT” entries are not current tasks.

## Authority and evidence pointers

- Current product matrix: `docs/first-release-baseline.md`.
- Accepted decisions and supersession: `docs/DECISIONS.md`.
- Requirements and architecture contracts: `docs/REQUIREMENTS.md`.
- Voice/ASR evidence: `docs/voice-closure-checkpoint.md`, `docs/asr-productization-acceptance.md`, `docs/network-asr-checkpoint.md`.
- Local model provenance and historical research: `docs/local-asr-checkpoint.md`, `docs/THIRD_PARTY_LICENSES.md`.
- Mixed Input authority/evidence: D074/D075 and the Phase 11 evidence pointers.
- Branch/version/release governance: `docs/branch-policy.md`, D076/D077, D080.

Historical checkpoints remain available for provenance; they do not override this current map.
