# Change contract — Windows settings UI, C# + WinUI 3 (2026-10-21)

> **Scope of this record.** It bounded the **first** batch of this task: the D087 decision, the migration inventory, and
> the first implementation attempt. The statements below that say "nothing is compiled" and "only source-level gates can
> run" describe **that batch**, not the current state. Current outcome: the client is **implemented, compiled, tested and
> packaged** in fusion-windows PR #32 — core, WinUI views, 515 checks with an isolated Host on a writable disposable profile,
> and the corrected save path exercised through the packaged executable. It is **not installed and not owner-accepted**. See
> `migration-inventory.md` §6 and the Stage 6 plan's settings section for the per-level state. The contract's scope,
> must-preserve list and hard stops still govern the remaining work.

**Goal.** Replace the presentation layer of the Windows settings surface with a Fusion-owned **C# + WinUI 3** client
that drives the same Host settings protocol and the same official Fcitx configuration, without adding a second
configuration store or changing engine semantics.

**Deliverable (this batch).** (a) The accepted decision that supersedes D085's presentation clause only (D087).
(b) A reviewable migration inventory covering every exposed option, not only the ones the design prototype shows.
(c) The WinUI 3 client implementation and its packaging — **blocked in that batch** on the missing package feed, see
`blocker.md`. In that batch nothing was compiled, packaged, installed or accepted; the core has since been implemented and
tested, and packaging, installation and acceptance remain open.

**Authority / inputs.**
- Owner instruction of 2026-10-21 (C# + WinUI 3 preferred; task-oriented navigation; preserve official capabilities
  and configuration semantics; UX/correctness/maintainability over one-time coding cost).
- D085 (order: settings UI, then EXE installer; official configuration semantics and storage) — presentation clause
  superseded by D087 only.
- D086 (scoped GPL-2.0-or-later for `FusionSettings.exe` and configtool-derived sources) — **unchanged**; it applies to
  the retained upstream-derived executable, not to the new client.
- `docs/windows-architecture.md` §22 settings paragraph; `docs/windows-daily-use.md` §1/§6;
  `docs/windows-settings-ui-feasibility.md`; `evidence/stage-a-global-settings-2026-10-20/`.
- Product source at `fusion-windows` `085326f` (`include/fusion/settings.hpp`, `settings_protocol.hpp`,
  `src/host/host_server.cpp`); pinned upstream `fcitx5` `a7f2528`, `fcitx5-chinese-addons` `61474bd`.

**Allowed scope.** New client source and its build definition; packaging scripts for the client; tray/Start-Menu entry
points; the documents above; the evidence pack under `evidence/settings-winui3-2026-10-21/`.

**Must preserve.**
- Fcitx configuration formats, storage in `%APPDATA%\Fcitx5\{config,data}`, no Fusion-specific configuration store,
  upgrade keeps user data.
- Official option semantics: enums, optional values, numeric bounds, defaults/reset, key-list ordering and
  multiple bindings, schema-known unsubmitted options.
- Host protocol invariants: endpoint normalization, user/session checks, capability negotiation, incarnation fencing,
  request bounds, timeouts, composition protection, `settings_api_version` handshake.
- The settings-before-EXE-installer order (D085) and the unresolved U4 signing/channel.
- Existing TSF key routing: this task does not add a key-routing policy (bare Shift stays unresolved).

**Expected behavioural delta.** A new settings client with the accepted navigation (Input methods · Keys · Candidates
and input · About and diagnostics), one canonical draft per underlying option, shared Pinyin/Shuangpin editing state,
explicit Save/Discard with unsaved-edit guards, and honest partial-success reporting. Engine behaviour, configuration
files and learning data are unchanged by the UI itself.

**Hard stop (not hit).** No new configuration store; no second materialization authority; no engine-semantic change;
no repository-wide licence selection.

**Validation.** See §5 of the instruction. In the first batch only source-level gates could run (see `blocker.md`); the
core now builds and its checks run. Anything not executed is recorded as NOT RUN, never as passed — in particular no WinUI
compilation, no packaging and no product-path acceptance.

**Commit / push / PR policy.** One coherent PR per independently reviewable task on `feature/*` or `docs/*` branches from
`main`; no merge, no CI dispatch, no installation.
