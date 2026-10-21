# Accepted Decisions

This file records durable accepted product/architecture decisions and concise
supersession boundaries. Detailed experiment transcripts, benchmark tables,
CI logs, device checklists, and release execution evidence remain in the
referenced evidence documents or Git history. Historical decisions are retained
when their rationale or supersession remains useful; a superseded clause is
not current authority.

## D001–D021 — project and initial architecture

### D001 — Android first
Android is the first target platform; other platforms are separate tracks.

### D002 — Pinyin/Shuangpin primary input; MoQi auxiliary
Pinyin and Shuangpin remain primary input. MoQi is an Auxiliary Filter, not a
decoder or replacement input engine.

### D003 — MoQi interaction
MoQi uses the bounded on-demand per-character/word interaction: filter,
partial-select, preserve composition, continue typing, and permit another
filter pass. It must not force whole-string commit.

### D004 — shared Auxiliary Filter infrastructure
Reuse and minimally generalize the upstream Stroke Filter infrastructure.

### D005 — trigger/implementation separation
The Auxiliary Filter trigger enters the configured implementation. It is not
hard-bound to Stroke or MoQi and no general plugin framework is introduced.

### D006 — MoQi target semantics
MoQi filters the target character after the current selection frontier. It does
not inherit Stroke’s arbitrary-character-in-phrase predicate.

### D007 — LibIME boundary
Reuse LibIME partial-selection and composition APIs. LibIME remains unmodified
unless a new, concrete source/API conflict is accepted.

### D008 — fixed MoQi table
The product table is `gaboolic/moqima-tables@6d8ba8f1c57466f358e682baefe11bbd0fe389ab`,
file `moqima_gb18030.txt`, MIT. The pin, digest, and configure-time packaging
are the single source for the product table.

### D009 — Rime reference
Rime is a reference/alternative, not a hard dependency.

### D010 — Android generic configuration
Reuse the existing generic Fcitx configuration UI and descriptor path.

### D011 — data/update separation
Dictionary updates and user-input upload are independent. Enabling an update
does not authorize uploading user input.

### D012 — default Voice trigger
The microphone button is the default Voice trigger.

### D013 — optional Space trigger
Long-press Space may trigger the same Voice Input flow; it does not create a
second pipeline.

### D014–D015 — System ASR boundary
Reuse Android SpeechRecognizer/upstream-compatible work. System
SpeechRecognizer/RecognitionService is one optional ASR backend, not the whole
Voice architecture and not the internal Provider abstraction.

### D016 — Provider independence
Voice flow, ASR backend, and concrete Provider remain separate. Provider changes
must not change the trigger/session contract.

### D017 — ASR and post-processing separation
ASR ends at Raw Transcript. Optional TextPostProcessor is independently
configurable and fully disableable. The provider-neutral seam is implemented
and retained by Android PR #15; configured processor/provider is NONE/null/
disabled, so the current product path is ASR → Raw Transcript → IME.
The tested 0.6B–1B Local-LLM route is STOPPED after the quality gate failed;
llama.cpp/JNI/GGUF runtime was removed. Cloud LLM is not implemented/planned.
A concrete processor requires a separate decision/task.

### D018 — auditable Voice data flow
Voice routing, lifecycle, receiving party, fallback, and termination must be
auditable without default transcript/audio/credential/endpoint-content logging.
Content-level transcript logging requires a separate explicit privacy decision.

### D019–D021 — repository and scope boundaries
Fusion is the controller repository; upstream forks remain separate. Do not
choose a repository-wide license prematurely. Prefer minimal, reviewable
changes and existing upstream infrastructure over parallel abstractions.

## D022–D040 — productization decisions

### D022 — batched validation
Validation is staged after source/diff review and uses the applicable local,
CI, device, and service route; a lower gate never substitutes for a higher one.

### D023 — configuration persistence
Auxiliary Filter persistence follows the upstream path; unit tests need not
claim filesystem persistence they cannot exercise.

### D024 — MoQi Android packaging
The addon retrieves the fixed table at configure time and installs it through
the config component; Android requires no MoQi-specific product source change.

### D025 — historical MoQi release identity
The old `.moqi` package/signing line is historical and separate from Fusion
Enhanced. It is not the current signer or migration path.

### D026 — no single OEM System ASR dependency
System RecognitionService cannot be the only usable Voice path.

### D027 — Architecture A VoiceBackend
Voice uses an internal VoiceBackend boundary:
SystemAsrBackend → SpeechRecognizer/RecognitionService, and DirectAsrBackend →
Fcitx-owned AudioRecord → configured Local/Managed/Self-hosted Provider.
This is not a public plugin framework.

### D028 — Provider categories and scope
The logical categories are System, Local, Managed Cloud, and Self-hosted.
Provider-specific implementations are acceptable; no generic protocol/plugin
framework is required. Current Managed Cloud scope is Doubao, Alibaba/Qwen, and
Tencent.

### D029 — Provider-specific settings
Provider credentials/configuration are independent, securely stored, and never
cross-reused. Local models are separately managed from the Local runtime.
Maintainer credentials never enter source, CI, APK, or release artifacts.

### D030 — Voice visibility
Voice trigger visibility follows configured-backend semantics and availability,
not a hardcoded System-ASR assumption.

### D031 — shared Voice panel
Mic and Space sessions use shared lifecycle/panel semantics; Direct capture may
provide a real level visualization, while System uses a non-PCM indicator.

### D032 — historical Local A/B study
The first A/B Local comparison was a bounded research/device study with external
models. Its evidence remains historical and does not select the current model
set.

### D033 — four Provider categories
All Providers use the same Voice flow. Self-hosted does not create a separate
pipeline. Historical checkpoint claims that Qwen/Tencent real-device work was
pending are superseded by later owner acceptance; Self-hosted device testing is
DEFERRED/non-blocking.

### D034 — selection and first recommendation
Users configure/enable concrete services; “current service” selects one. First
recommendation is one-time and Local-first, then disclosed/authorized System,
then configuration guidance. It never silently selects a cloud/self-hosted
service or rewrites the user’s selection.
Disabling the current service must prompt for a new choice rather than silently
selecting another cloud service; System authorization and disclosure remain
required before use. The durable selection/recommendation contract is also in
REQUIREMENTS.md §9.1–9.2.

### D035 — bounded runtime fallback
Only a selected external/network ASR may fall back on early technical/service
failure before usable session establishment. The target is one eligible,
installed, enabled, runtime-ready production Local ASR. System is never an
automatic target; there is no external-to-external chaining, quality-triggered
fallback, post-establishment migration, or PCM replay. `onStarted` is not
session establishment. The core early-failure path is DEVICE VERIFIED /
COMPLETE; edge/race permutations are not a reopened core gate.
User stop/cancel ends the session without fallback; Local/System failures do
not chain; no external-to-external, Local-to-cloud, or any-to-System fallback
is permitted. The detailed trigger, privacy, and session boundary contract is
in REQUIREMENTS.md §9.3.

### D036–D038 — Local model selection
The old A/B checkpoint remains historical. Current product models and order are
fixed by D050. Each Local model is an independently enabled/selectable service;
installed, enabled, current, production, recommendation eligibility, and
runtime readiness remain separate dimensions.

### D039–D040 — dictionary distribution
Dictionary distribution uses fixed, reproducible, license-reviewed candidates
and preserves the required license/provenance boundaries. D080 closes the
productized Dictionary Manager update path.

## D045–D055 — retained boundaries and Voice architecture

### D045 — current Local recommendation
The current Local model set and recommendation semantics are governed by D050;
the old Zipformer research collection is not restored.

### D046–D049 — UI boundaries
Toolbar configuration is canonical state projected by the UI; runtime width does
not rewrite it. Voice Settings is presentation/navigation and does not change
Provider, dictionary, or lifecycle semantics. Toolbar/Voice Settings device
regression and defect closure are COMPLETE.

### D050 — current Local ASR release line
Current production/recommendation-eligible Local order is:
X-ASR offline INT8 → X-ASR 960 ms streaming INT8 → FunASR Nano.
Models are downloaded from fixed upstream sources on demand; weights are not
bundled, mirrored, or project-hosted. Core release-device lifecycle and
abnormal/error Model Manager paths are COMPLETE / DEVICE VERIFIED. Fixed-
upstream download/distribution approval is distinct from independent
provenance, license, and NOTICE evidence: the latter remains itemized and
incomplete where `docs/THIRD_PARTY_LICENSES.md` says so. This evidence debt is
not a blanket download or release blocker; do not start a new audit in this
status-reconciliation task.

### D051 — paired punctuation
The experimental paired-punctuation extension is PAUSED and excluded from the
first-release product scope by D066. Do not treat its historical experiment as
a current product task.

### D052–D053 — MoQi downstream boundary
MoQi remains downstream-only. Upstream contribution work is closed/non-blocking;
the product table and Auxiliary Filter semantics remain.

### D054 — Fusion Enhanced identity
The current Fusion Enhanced application identity is distinct from historical
MoQi identity and does not perform old-install migration.

### D055 — top-level Voice providers
External Android Voice Input, System ASR, Local, Managed Cloud, and Self-hosted
are top-level provider choices with the existing ownership/selection semantics.

## D056–D068 — release and authority closure

### D056 — Local release JNI closure
The sherpa-onnx R8/JNI runtime issue was fixed with the accepted package keep
boundary and closed by release/minified three-engine device verification.

### D057 — System ASR diagnostics
System ASR runtime failure must be visible/diagnosable within the accepted
SpeechRecognizer boundary. Do not add ROM-specific service selection or rewrite
system settings.

### D058 — release signing identity
Fusion Enhanced has its own permanent release signing identity, separate from
historical MoQi and debug identities. The canonical public certificate
fingerprint is `A5:15:B7:4A:C4:C3:84:51:54:E1:7C:AD:D1:3D:02:75:BE:70:01:DD:CF:2B:97:2A:4F:2F:06:1A:FD:26:89:0C`;
all `.fusionenhanced` release artifacts must verify against it or stop. Private
keys, passwords, and secret material never enter Git, `/tmp`, logs, or chat;
`.moqi` and `.fusionenhanced` identities remain permanently separate.

### D059 — Local distribution four-rights closure
Bundle, mirror/re-host, fixed-upstream in-app download, and manual import are
separate rights. The fixed-upstream download path is approved; project
mirroring/bundling is not. `licenseAuditComplete` is disclosure state, not a
download gate. Independent license/provenance/NOTICE evidence remains an
honest, per-model evidence status and is not converted into an audit PASS by
download approval or used as a blanket release blocker. Reopen only when:
(1) a model artifact, revision, URL, or SHA-256 changes; (2) an upstream
license or authorization changes or is clarified; (3) the download host is no
longer the author/original host; (4) the project starts mirroring or
re-hosting weights; (5) the project bundles weights in the APK; or (6)
concrete legal evidence contradicts this record. These conditions are
exhaustive for reopening the A/B/C distribution record; per-model evidence
remains in `docs/THIRD_PARTY_LICENSES.md`.

### D060 — stable baseline and authority
`docs/first-release-baseline.md` is the frozen D068 first-release matrix and
historical release authority. Other documents must not rewrite its rows with
post-D068 status; current post-release state and next work belong in
`docs/ROADMAP.md` and later accepted decisions.

### D061 / D066 — punctuation and RC supersession
D061’s acceptance of the expanded punctuation pin was superseded by D066:
D051 is outside first-release scope, the release pin is `47401b04`, and RC.1
was rejected. These are historical release-boundary decisions, not current
punctuation work.

### D063–D064 — signed RC and debug identity
Signed RC production uses the existing signing route or its CI fallback, puts
one SHA-256-recorded artifact in a draft release, and promotes that exact
artifact only after the applicable device gate; publishing must not rebuild
it. The D058 certificate is a hard gate. Never expose or copy private signing
material, passwords, or GitHub secrets. Debug identity is separate from
release identity and is shared by local/CI Debug/Test builds; do not mix
identities or rebuild a tested release artifact. Reopen D063 only for a
signing identity/material change, a second/bypass signing path, CI secret or
permission changes, a regression invalidating the accepted artifact, a change
to draft exact-artifact promotion, an owner-approved release-policy change, or
a changed frozen baseline premise. Ordinary inability to access local signing
credentials is not a reopen condition.
The accepted Debug identity is the permanent Fusion Enhanced Debug/Test
identity (certificate SHA-256
`DC:49:71:82:C5:48:66:34:FD:68:E5:71:B8:78:A3:3E:CE:A6:8B:DA:D2:05:C3:D6:1A:ED:09:B9:83:92:A5:6A`),
shared by local and CI Debug/Test builds and never used for Release/Signed RC.

### D068 — first release
RC.2 was exactly promoted and the first Fusion Enhanced release was published.
Release evidence is historical; post-release development state is tracked by
the current baseline/ROADMAP.

## D069–D080 — Mixed Input, governance, lexicon, and latest release

### D069–D071 — Mixed Input foundations
Preserve the accepted bounded mixed-input architecture: Chinese and English
hypotheses share one bounded search, while sourcing, lexicon, learning, and
resource ownership remain separate. Preserve raw/output alignment, selection
frontier, partial selection, canonical/literal coexistence, and the rule that
uncalibrated heterogeneous raw scores are not directly compared. Auxiliary
Filter remains downstream of primary candidate generation. These decisions
are not licenses to revive superseded Architecture A' or old ranking
experiments; the normative architecture contract is in REQUIREMENTS.md §2.1.

### D072 — Windows boundary
Windows Fusion is a separate product/research track with authority in
`docs/windows-architecture.md`. D082 later sets the combined target
(Windows-native architecture + faithful Fcitx5 port), separates Fcitx WHAT
from Windows HOW, and makes a scoped supersession; D072's Windows mechanisms
are otherwise unchanged. D083 closes Windows Stage 4 and partially supersedes
the §9 open scope for the popped context-stack member.

### D073–D075 — M2+/r4 closure
D073’s M3-B proposal was not accepted. D074 preserves the mixed-input contract
and explicitly supersedes A' as product architecture. D075 accepts M2+/r4 as
the current Mixed Input architecture baseline; A' is historical/reference,
architecture competition is CLOSED, and no r5/r6 or new held-out competition is
planned. The validated r4 implementation uses `ENG_STATE=reset`; that is not
a permanent architecture invariant. Preserve a centralized English transition
strategy, Pinyin/Shuangpin support, the LibIME-unmodified principle, and the
bounded-search/alignment/selection contract in REQUIREMENTS.md §2.1. M3-B and
cross-English LM-state preservation remain deferred research, not
productization blockers. Phase 11 productization and merged Android
integration are VERIFIED / COMPLETE. Reopen only on concrete conflicting
source/API/device/CI/release evidence or applicable accepted Decision
conditions; more testing alone is not a reopen trigger.

### D076 — branch governance
Canonical branches, PR/review flow, gitlink reachability, and deletion
preconditions are governed by `docs/branch-policy.md`.

### D077 — versioning
Fusion versions use the canonical upstream-version + `-fusion.N` policy, the
sole primary authority for version/tag semantics; `docs/branch-policy.md` §9
points to it. Preserve the complete upstream release version. The upstream
component may advance only after that upstream baseline passes the branch
policy integration/validation flow and enters permanent `fusion` trunk; an
upstream release alone does not change the Fusion baseline. Fusion revisions
increase monotonically within one upstream baseline and reset to 1 only when
the accepted upstream baseline changes. Release tags are
`v<complete-upstream-version>-fusion.<N>` with optional `-rc.K` after the
Fusion revision; do not invent alpha/beta syntax without a future accepted
decision. Android `versionCode` is independent and strictly monotonic across
installable Fusion releases. Historical tags and release records are never
rewritten. Reopen only for an unforeseen version representation ambiguity or
a real release-infrastructure conflict, resolved by a new Accepted Decision.

### D078–D079 — English lexicon
The current English Core uses one bundled system lexicon generated from the
D079 Ice/Frost/curated replacement; Android Spell remains separate and is not
merged into Fusion’s mixed-input lexicon. Do not restore the retired SCOWL
comparison, a second runtime lexicon, external raw frequency/ranking metadata,
or an uncalibrated ranking dimension. Preserve the M2+/r4 bounded-search,
sourcing/ownership, and OOV boundaries in REQUIREMENTS.md §2.1. Any future
downloadable/updatable English resource must carry a runtime/scoring
compatibility generation and pass the corresponding mixed-input regression
gate; this is not current implementation authorization. The D079 curated
replacement supersedes only D078’s SCOWL-source wording; the remaining D078
architecture and forward-compatibility rules remain valid.

### D080 — v0.1.3-fusion.7 and Dictionary Manager closure
The v0.1.3-fusion.7 exact-artifact release, addons pin advancement, and
productized Dictionary Manager Phase 5D are recorded as COMPLETE / historical
release facts. Do not treat the old Phase 5D forward pointer or release
execution transcript as current NEXT.

### D081 — Bounded ASR capture-duration policy
Auto-stop (endpointing) remains the normal Microphone-session termination;
the watchdog endpointing values recorded in `docs/voice-closure-checkpoint.md`
are unchanged. Independently of endpointing, every Fcitx-owned Direct PCM
capture path keeps a mandatory hard safety ceiling: it is the only bound on
sessions that never auto-stop, such as a held Space gesture. The
common Direct ceiling is 180 s where no lower applicable bound exists, and the
effective bound is `min(common ceiling, applicable model/engine/protocol lower
limit)`. Lower limits honored here: the FunASR Nano current pin gets a
temporary 30 s protective ceiling (dual-device empty finals at ~34–39 s for
that pinned export; revisiting it is a re-pin decision, not a policy change);
selecting the Tencent Hy-ASR-3.0-preview engine honors its documented 60 s
per-session limit; OpenAI-compatible whole-file self-hosted capture stays at
60 s because its post-stop upload/transcription timeout is premised on at most
60 s of audio. Ceiling expiry finalizes the session exactly like a user stop
and releases the microphone: it is not a cancel, not an error, and never
triggers D035 fallback. System ASR remains service-controlled and gains no
Fcitx duration timer. A longer ceiling is a safety bound, not a claim that
recognition quality at 180 s is device-verified; X-ASR lengths beyond the old
60 s remain post-implementation validation debt.

## D082 — Windows-native architecture carrying a faithful Fcitx5 port (scoped supersession of one implementation direction)

### D082 — Advanced Windows-native architecture + faithful Fcitx5 port
The Windows line pursues both an advanced, reliable, Windows-native input-method
architecture and a faithful Windows port of Fcitx5; the two are complementary.

**Fcitx WHAT.** Fcitx5 framework/engines (with chinese-addons and LibIME) are
the default semantic authority for mature ordinary IME behavior: Pinyin/
Shuangpin, engine key-event handling, consume/forward semantics, punctuation,
QuickPhrase, composition, candidate generation/selection/partial selection,
commit/preedit behavior and ordering, frontend callbacks/InputContext contract,
and user learning. Mature Fcitx functionality, semantics, frontend contracts,
event behavior and implementation mechanisms are preserved and reused by
default and are not changed merely to fit a Windows-specific abstraction.

**Windows HOW.** Windows owns the platform adaptation and reliability TSF
requires: TSF hard policy and ownership disposition, context/document
identity, DLL/Host isolation and transport, causal ordering/barriers,
generation fencing, lawful EditSession materialization, recovery, bounded
ledger/reliability mechanisms, Host/link health, observability, and
candidate/preedit/UI projection. These mechanisms may be sophisticated and
robust; they exist to carry Fcitx faithfully and reliably on Windows, not to
become an independent input-method semantic authority.

**Evidence threshold.** A mature Fcitx behavior or implementation mechanism is
changed only when concrete evidence establishes at least one of: a correctness
defect; a serious portability defect; a major product-relevant defect; a
genuine Windows/TSF incompatibility that cannot be solved faithfully in the
Windows adaptation layer; or an existing Fcitx API/frontend boundary that is
demonstrably insufficient for a required TSF capability. "The Windows
implementation would be easier if Fcitx behaved differently" is not
sufficient. Such changes stay minimal and scoped to the evidenced defect.

**Scoped supersession.** Superseded only to that extent: (1) the Stage 3C
working direction of first expanding an independent Windows key-admission/
ownership semantic model and then adapting or selectively exposing Fcitx
behavior through it — a Windows-side classifier must not keep an input from
Fcitx merely because Windows predicts the engine's behavior, absent a proven
TSF constraint requiring pre-admission; (2) D072 wording that frames
Fcitx5/chinese-addons/LibIME as a provisional v0.x reference backend or the
Host as an independent semantic authority — the Host hosts Fcitx semantics, and
replacing or departing from Fcitx requires this Decision's evidence threshold
(D072 §15 Exit Gate signals are such evidence candidates). D072 otherwise
remains accepted: thin TSF DLL/out-of-process Host separation, A′ ordering,
drain-before-PASS, watermarks, EVENT/STATE/EFFECT, ownership/terminal
disposition, ledgers, fencing, recovery, observability and the SendInput-replay
prohibition are valid Windows HOW mechanisms, not invalidated for being
Windows-specific.

**Stage 3C framing and closure.** Historical pre-Stage-3C evidence that the
Windows path could keep ordinary keys such as comma from reaching Fcitx,
because Windows predicted semantic ownership first, remains valid as history.
One Fcitx key dispatch may legitimately produce an ordered sequence (e.g.
`CommitText("你好")`, `SetPreedit("")`, `CommitText("，")`); the Windows port
preserves that semantics and order unless a real TSF constraint proves a
specific adaptation necessary. Merged fusion-windows PR #7 closed Stage 3C:
the faithful frontend carries those ordered callbacks through the accepted
Windows HOW machinery as separate ordered records/settlements, without a
one-key-one-Effect invariant, a new FcitxTransaction semantic abstraction, or
any change to mature Fcitx/chinese-addons/LibIME semantics. D082 is therefore
implementation-supported, not revised. The earlier Change-Contract proposals
(merging consumed + CommitText into one Effect, one physical key = one
semantic Effect, folding ordered Fcitx callbacks to fit the envelope, a new
FcitxTransaction abstraction) remain rejected. Current normative wording is in
`docs/windows-architecture.md` (header, §1–§2, §15, §22–§23, §25). No product
requirement change (REQUIREMENTS.md unchanged); no runtime behavior changed by
this Decision.

## D083 — Windows Stage 4 closure and the popped-context lifecycle boundary (partial supersession of the D072 §9 open scope)

### D083 — Stage 4 closure and OnPopContext scope

Merged fusion-windows PR #8 (Stage 4A), #9 (Stage 4B) and #10 (Stage 4C) close
Windows Stage 4 as recorded in `docs/windows-architecture.md` §22/§25. Stage 4 is
COMPLETE / VERIFIED over the boundary it implemented and validated on the real
product path: a thin client-only TSF DLL plus out-of-process `FusionHost.exe` over
a per-user/per-session Named Pipe, the D072 §13 reconnect ladder with per-context
semantic readiness, Host-incarnation/connection-generation fencing, and the TSF
lifecycle obligation table.

**Accepted lifecycle boundary (the part that was previously OPEN).** The popped
context-stack member is reconciled/torn down on `OnPopContext`: the Fusion semantic
incarnation for that popped context is audited, fenced to explicit terminals,
unbound from the Host, and its entry generation advanced so late edit sessions and
pumps can no longer match. This is **partial** supersession of D072 §9's statement
that which TSF lifecycle events constitute a semantic discontinuity remains OPEN:

- it does **not** mean the whole `ITfDocumentMgr` is destroyed — document-manager
  destruction is the separate `OnUninitDocumentMgr` path;
- it does **not** imply teardown of a different context (an underlying/main context
  returning to the top of the stack);
- a later push or use of an `ITfContext` establishes or binds a Fusion semantic
  incarnation under the current Fusion lifecycle contract and must not resurrect
  stale Host state — the Stage 4B incarnation/connection-generation fence carries
  that barrier;
- lifecycle sub-boundaries not actually resolved by Stage 4 evidence (transitory
  contexts, context reuse/discontinuity, natural destruction while an async request
  is pending, and every application other than Notepad) remain **OPEN / UNFROZEN**.

**Evidence.** Microsoft TSF's context-stack contract (applications and text services
both add and remove stack members through `ITfDocumentMgr::Push`/`Pop`, and a text
service's own modal UI, such as a word list, enters and leaves the stack the same
way — so popping one stack member is neither document destruction nor removal of
the stack); mature TSF precedent such as Mozc establishing per-context private
state on push and removing it on pop, carried here as independently reviewed
rationale rather than first-hand verification by this Decision; and Fusion's own
Stage 4C implementation and product-path evidence, where Notepad pops the context
before `DOCMGR_UNINIT`, every teardown records an explicit `CONTEXT_TEARDOWN`
audit, and closing a document that held an open composition delivers **no**
`OnCompositionTerminated` callback.

**Not superseded, not claimed.** D072's remaining §9 principle stands unchanged
(`focus ≠ semantic incarnation`; focus loss alone neither advances the epoch nor
terminates a composition), as do A′, drain-before-PASS, the consumed-STATE →
EFFECT watermark, EVENT/STATE/EFFECT, the product replay prohibition, and the
thin-DLL/out-of-process Host separation. P0-A, P0-B and P0-D remain OPEN,
including the user-visible disposition of work that a teardown or a Host crash
leaves unresolved. This closure claims **no** multi-application out-of-process
re-verification: the real-product-path application evidence is Notepad, Firefox
contributes one incidental reconnect observation only, and Chromium/Edge remain
DEFERRED UNKNOWN (`docs/windows-architecture.md` §7.6/§18); that breadth is
recorded as VALIDATION DEBT for the Stage 6 application gate, not as a Stage 4
mechanism failure. No product code, requirement or runtime behavior changes under
this Decision; current normative wording is in `docs/windows-architecture.md`
(header, §9, §13, §18, §22–§25).

## D084 — Windows Stage 6: WP7 daily-use acceptance, U6 open-on-new-window, U7 presentation

### D084 — WP7 daily use carries WP5/WP6 acceptance
The owner directs Stage 6 to proceed to **WP7 daily use** (2026-10-19). WP5 (failure and recovery) and WP6 (application
compatibility) are accepted through daily use: the owner reports a problem when one occurs, it is assessed and repaired
by impact, and **closing the daily-use period without reported problems is acceptance**. This supersedes the separate
WP5/WP6 acceptance campaigns, deliberate crash/hang injection for stage acceptance, and the mandatory pre-WP8 fault-test
session; none of them is a release blocker. Unexecuted scenarios are never recorded as PASS, and daily use is not
complete until the owner closes it. The Stage 6 plan's §8.2 and §8.3 remain the **analysis reference** for a reported
problem, not a mandatory campaign.

**U6 decided:** a newly opened application window starts with Fusion open, while an explicit Ctrl+Space close remains
effective (before or after the first key) and ordinary focus changes are not overridden. Implementation and validation
status are tracked in the Stage 6 plan (U6 row) and in the fusion-windows PR, not in this decision; the evidence that
motivated it is that the keyboard open/close compartment reads open at activation and closed at the first key of a new
context, and that attributing the change to the application or to Windows is an unconfirmed hypothesis.

**U7 presentation decided:** a persistent tray status, a fault balloon notification, and actions to restart the Host and
open its log directory; no focus stealing, no input content and no repeated per-key notifications. Automatic restart is
**not** authorized. The minimal design is in `docs/windows-daily-use.md` §4. What the presentation deliberately excludes
is part of the decision: no automatic restart and no hang detection. Implementation, validation and package state are
**not** part of this record — they are tracked in the Stage 6 plan's U7 row, `docs/windows-daily-use.md` §4 and the
fusion-windows PR, so a moving implementation head never has to be edited into an accepted decision.

**Not changed:** U2's scheduled-task start mechanism and its no-automatic-restart policy; U3 (diagnostics off by default,
technical content only); U4 (signing/channel) stays open; P0-A/P0-B/P0-D remain OPEN. Current normative wording is in
`docs/windows-architecture.md` (§10, §14) and `docs/windows-stage6-plan.md`.

## D085 — Windows next work after the minimal tray: official settings UI, then the EXE installer (partial supersession of U1's selection method and of the broad-settings-UI exclusion)

Decided by the owner on 2026-10-20.

**Order of the next work.** After the minimal U7 tray batch is closed, the next work is the **official settings UI**, and
after it the **installer that packages it**. Nothing else is promoted ahead of those two, and neither starts without its
own bounded execution instruction.

**The installer deliverable is an EXE.** The Windows installer that carries the product to a user is a single **EXE** —
not an MSI, and not a zip the user unpacks and runs a script from. The framework is **not** chosen by this decision and
must not be assumed by the implementation.

**The settings UI reuses upstream first.** The UI prefers **reusing Fcitx5's own settings UI** and its configuration
semantics over writing a Fusion-specific configuration surface. Whether that is feasible on Windows is **not
established**: the upstream UI's Windows dependencies and the way it would communicate with this product's Host must be
inspected before an implementation approach is chosen. A Fusion-written UI is a fallback, not the default, and it is not
   authorized here. *(Partially superseded 2026-10-21 by D087: the presentation layer is now a Fusion-owned C# + WinUI 3 client.
   Everything else in this decision — official configuration formats and storage, no additional configuration store
   *(narrowed for Windows presentation settings by D088, 2026-10-22)*, the user-data policy,
   the settings-before-EXE-installer order, the EXE deliverable form and U4 remaining open — remains effective.)*

**Partial supersession.** Exactly two accepted statements are narrowed:

- **U1's "selection only through Fcitx's existing configuration files"** was a *no-UI* decision. It remains the
  configuration model — the UI is a front end to those same files — but it no longer excludes a UI, and "no
  Fusion-specific file, switch or registry key" now means no *additional* configuration store, not the absence of a UI.
  *(That phrase is narrowed once more on 2026-10-22 by **D088**, for Windows presentation settings only: one Fusion-owned
  Fcitx configuration may exist for how the Windows candidate window presents itself. It duplicates no engine or global
  value, and every other constraint in this decision — official formats, per-user paths, no Fusion-specific data
  location, the user-data policy — remains effective.)*
- **The Stage 6 plan's exclusion of "a broad settings UI"** is superseded for this settings-UI work and only for it:
  the UI is the product's own settings surface, not an attempt at a general desktop-configuration product.

Everything else U1 fixed is **unchanged**: official Fcitx configuration formats, storage in Fcitx's standard per-user
paths (`%APPDATA%\Fcitx5\{config,data}`), no Fusion-specific data location, upgrade keeps user data, and uninstall
leaves it unless the user chooses removal.

**Not decided.** Signing and the distribution channel stay **undecided** (U4 remains open), the installer framework is
undecided, and the UI's exact scope is undecided beyond "official settings for what the product actually exposes".

**Not changed.** Overall WP7 acceptance is unchanged — daily use still closes it, and the tray's first installed run is
still the owner's check. U6's decision is unchanged. The minimal tray's scope is unchanged: no automatic restart, no hang
detection, no new lifecycle framework. No implementation of the settings UI or of the EXE installer is authorized by this
record; the work-package ordering is updated in `docs/windows-stage6-plan.md` and `docs/ROADMAP.md`.

## D086 — scoped Windows settings executable licence and upstream integration

Accepted by the owner on 2026-10-20 through the bounded implementation instruction for the upstream-derived Windows settings UI.

**Scope of the licence authorization.** The separate Windows settings executable (`FusionSettings.exe`) and source code derived from the retained `fcitx5-configtool` 5.1.16 widget/configtool sources may be distributed under **GPL-2.0-or-later**. Preserve all upstream copyright/SPDX notices and the full GPL text, and provide the complete corresponding source, exact upstream revision, maintained patch/source integration, and build instructions with the owner-test package. This authorization is limited to the settings executable and its derivative source; it does **not** select a repository-wide license and does **not** apply GPL to `FusionHost.exe`, `FusionTray.exe`, `FusionTsf.dll`, the existing Host/backend/runtime, or unrelated repository content.

**Other components retain their own licenses.** `fcitx5-qt` modules retain their per-file licenses (LGPL-2.1-or-later for widgetsaddons and BSD-3-Clause for dbusaddons); the Qt shared libraries/plugin use their applicable upstream open-source license terms (the owner-test build selects the LGPL-3.0-only shared-library path and supplies the license texts/source-access instructions); pinned Fcitx/chinese-addons/LibIME runtime components retain their own upstream LGPL-2.1-or-later terms. The implementation must preserve these notices and check the actual linked/distributed dependency set; this decision does not blanket-license the product.

**Relationship to D085.** D085 remains unchanged and continues to govern work order, Fcitx configuration semantics/data paths, and the separate EXE installer. This later scoped authorization removes D085's previously identified *licence prerequisite* for the bounded settings implementation; it does not itself authorize a public release, machine installation/registration, merge, selection of U4 signing/channel, or implementation of the EXE installer. D085's official-settings-UI-before-EXE-installer order remains current.

**Upstream maintenance.** The owner-maintained `choicky/fcitx5-configtool` fork is the source home for reusable configtool UI adaptations, on a dedicated branch based on the pinned upstream 5.1.16 commit. The Windows product repository owns FusionHost transport, tray, packaging and source-pin/build integration. The single `fcitx5-qt` MSVC path-portability fix remains a separate, reviewable patch; no fork was authorized or created for fcitx5-qt by this decision.

Implementation/package/validation state is tracked in the Stage 6 plan and product PRs, not in this Accepted Decision. U4, U6, WP7 and P0-A/P0-B/P0-D remain unchanged.

## D087 — Windows settings UI: Fusion-owned C# + WinUI 3 presentation (supersedes D085's presentation clause only)

Decided by the owner on 2026-10-21.

**Presentation technology.** The Windows settings surface is a **Fusion-owned C# + WinUI 3** application. This
supersedes exactly one clause of D085 — "The settings UI reuses upstream first … A Fusion-written UI is a fallback, not
the default, and it is not authorized here." That clause is replaced: the **presentation layer is Fusion-written**, while
upstream Fcitx remains the authority for **configuration semantics** (schema, defaults, storage, apply path), not for the
widget technology.

**Accepted with it.** The task-oriented navigation (Input methods · Keys · Candidates and input · About and diagnostics),
lower-frequency settings under "More settings" sections rather than one advanced dump, and the interaction rules the
owner stated for the key editor, shared Pinyin/Shuangpin state and saving.

**Unchanged.** D085's **work order** (settings UI, then the EXE installer), its **EXE** deliverable form, official Fcitx
configuration formats, storage in Fcitx's standard per-user paths, the prohibition on an additional Fusion configuration
store *(narrowed on 2026-10-22 by **D088** for Windows presentation settings only; it stands unchanged for every engine
and global value)*, the user-data policy (upgrade keeps data; uninstall leaves it unless chosen), and every other U1
constraint. U4 (signing/channel) stays open; the installer framework stays undecided; no merge, installation,
registration or publication is authorized by this record.

**D086 is not changed.** Its scoped GPL-2.0-or-later authorization remains attached to the retained upstream-derived
`FusionSettings.exe` and the `fcitx5-configtool`-derived sources. The new client is Fusion-owned code and does **not**
inherit that authorization, and this decision selects **no repository-wide licence**. While the upstream-derived
executable continues to ship, its notices, corresponding source and build instructions remain required; whether it is
retained or retired is a packaging decision taken with its own evidence.

**Out of scope.** No engine-semantic change, no new configuration store beyond the single Windows presentation
configuration D088 authorizes, no TSF key-routing policy (the bare-Shift
boundary stays unresolved), no EXE installer implementation, no machine installation.

**Status.** The presentation change is **accepted**; implementation state is tracked in the Stage 6 plan and the product
PRs, not here. The option-coverage design is `evidence/settings-winui3-2026-10-21/migration-inventory.md`.

## D088 — Windows candidate-presentation configuration (narrow supersession of D085/D087's "no additional configuration store", for presentation only)

Accepted by the owner on 2026-10-22 through the bounded instruction for the horizontal/vertical candidate-layout setting.

**The decision.** A **Windows-specific presentation setting may have its own configuration**, while **official input
semantics must not acquire duplicate stores**. A setting that decides how the Windows candidate window presents itself has
no official Fcitx configuration to live in in this runtime, and inventing a second copy of an *input semantic* is not an
acceptable substitute. The owner therefore authorized exactly one narrowly scoped configuration for presentation.

**What it is.** `fcitx://config/candidate-presentation` — one Fcitx `FCITX_CONFIGURATION`
(`CandidatePresentationConfig`) holding the candidate layout only, served over the existing `GetConfig`/`SetConfig`
messages with the same `dumpDescription` / `safeSave` / reload semantics as the three official URIs, and stored through
Fcitx's own standard per-user configuration path (`…/Fcitx5/config/fcitx5/conf/candidate-presentation.conf`, measured on
this runtime beside `pinyin.conf`). It is **not** an additional configuration *system*: schema, serializer, storage
location, apply path and per-user policy are all Fcitx's. The Host owns the stored value and is its only reader and
writer; the WinUI client edits it through the Host; the candidate renderer consumes it.

**Why not the official option.** Upstream's only candidate-layout option is `verticalCandidateList`, a member of
`ClassicUIConfig` owned by the classicui addon (`classicui.h:129`, stored in `conf/classicui.conf`, read by that addon's own
`reloadConfig()` at pinned fcitx5 `a7f2528`). This runtime does not build, ship or load classicui, and the Host serves only
`global`, `inputmethod/pinyin` and `inputmethod/shuangpin`; `GlobalConfig` and the shipped addons have no layout option.
Reusing that key would mean loading the Linux classic UI (explicitly excluded) or storing a value with no reader. The new
configuration therefore uses its own namespace and never classicui's.

**Narrow supersession.** Exactly one statement is narrowed, and only for Windows presentation settings: D085's
"*no Fusion-specific file, switch or registry key* … means no *additional* configuration store" and D087's unchanged
"prohibition on an additional Fusion configuration store". For presentation, a Fusion-owned Fcitx configuration in Fcitx's
own storage is permitted. **Everything else stands unchanged**: official Fcitx configuration formats, storage in Fcitx's
standard per-user paths, no Fusion-specific *data* location, no duplicate store of any engine or global value, the
user-data policy (upgrade keeps it; uninstall leaves it unless chosen), D087's presentation-technology decision, D086's
scoped licence, and the settings-before-EXE-installer order. No engine-semantic change, no new key-routing policy, no
matrix layout, no theme or font work, and no general configuration-framework expansion is authorized.

**Precedence.** The stored preference is a **default**; an engine `CandidateLayoutHint` that names a layout wins. This is
classicui's own rule (`inputwindow.cpp:598-603`, again at `938-943`) adopted verbatim, so a semantic hint (clipboard,
unicode, imselector) keeps working and no stored value can silently override one. The default is **vertical**, which
preserves the presentation shipped before the option existed — deliberately the opposite of classicui's default, because
preserving existing behaviour outranks copying upstream's default.

**Status.** Accepted. Implementation, compilation, isolated validation and packaging are tracked in the Stage 6 plan and
the product PRs, not here. The option-coverage record is `evidence/settings-winui3-2026-10-21/migration-inventory.md`.

## Current authority rule

For current post-release product state, use the compact current-state/current-
next sections in `docs/ROADMAP.md` and later accepted decisions. Use
`docs/first-release-baseline.md` for the frozen D068 first-release matrix. For
requirements use `docs/REQUIREMENTS.md`; for evidence use the referenced
checkpoint/report documents. Historical PENDING/NOT RUN rows do not reopen
later CLOSED, VERIFIED, STOPPED, or DEFERRED facts.
