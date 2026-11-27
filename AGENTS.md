# AGENTS.md

本文件适用于本仓库及其全部子目录。参与本项目的 agent 应遵守以下长期工程规则。

## 事实来源与设计核对

- 开始设计或实现前，先按任务分类，识别相关需求/决策/当前路线图章节或主题专属权威，再阅读这些相关章节、指针及所涉及上游仓库的最新源码、公开接口和现有测试。完整历史决策、证据或研究仅在需要溯源、冲突仲裁或验证时按需读取；不得将本条机械解释为每个任务都必须读完整 `REQUIREMENTS.md`、`DECISIONS.md`、`ROADMAP.md` 与全部研究历史。
- 判断当前预期行为时，按以下权威顺序读取：`REQUIREMENTS.md` 当前有效需求、`DECISIONS.md` 中最新且未被 superseded 的 Accepted decision、`ROADMAP.md` 当前 Phase/checkpoint，最后才参考历史记录。首个 Fusion Enhanced 发布的冻结范围和功能状态矩阵以 `docs/first-release-baseline.md`（D060/D068）为准；首发后的实现状态和下一步见 `docs/ROADMAP.md`。
- Windows 线的当前规范性架构权威为 `docs/windows-architecture.md`（D072）；`researching-notes` 中的 Windows 架构文档仅为研究叙述与历史，不作为当前权威。
- 分支治理（canonical branches、命名空间、上游集成模型、gitlink 可达性与删除前置条件）的当前规范性权威为 `docs/branch-policy.md`（D076）；`evidence/branch-inventory-2026-10-11/` 与 `evidence/branch-governance-gate1-2026-10-11/` 仅为历史清点与迁移时点证据，不作为当前 policy 权威。
- 不得把 `SUPERSEDED`、`PARTIALLY SUPERSEDED`、`HISTORICAL` 或 `DEPRECATED` 条目实现为当前行为；仅部分 supersede 旧决定时，旧决定原位置必须标明仍有效和不再规范的部分。看似 ACTIVE 的冲突不得自行推断。
- 设计假设必须以当前源码和可复现证据验证；不要仅凭文档、历史结论、记忆或接口名称推断行为。
- 若已接受的设计与最新源码、公开 API 或实际测试结果冲突，立即停止相关实现。记录并报告冲突位置、源码版本或提交、相关调用链、测试结果及影响范围，等待设计被明确修订后再继续。
- 不得把推测、仅有代码但未验证的行为、跳过的测试或失败的 CI 写成已完成或已验证。

## 已收口证据与重开规则（CLOSED EVIDENCE RULE）

- 被当前权威记录标记为 CLOSED / VERIFIED 的事项，**不得仅因** agent 找到更旧的 roadmap 条目、checkpoint、
  字段名、commit message、TODO 或被 superseded 的 Decision 就重开或降级。历史证据保留但不覆盖当前权威。
- 仅在出现**实质性新事实**时重开：相关产品行为/源码变化；依赖/artifact/版本变化；上游 API/许可/行为变化；
  新设备证据与已接受结果矛盾；CI/release 回归与之矛盾；某 Decision 专有的 reopen 条件被命中；新的可靠证据直接矛盾当前权威。
- 专项重开边界（各自为准，不重复泛化）：D059 模型分发四项权利、D056 release/JNI ABI、D057 System-ASR 边界、D063 签名产物与精确晋升策略，均以其条目内列明的 reopen 条件为准。

## 状态分类：VALIDATION DEBT vs RELEASE BLOCKER

- **CLOSED / VERIFIED**：当前证据接受，无故不重开。
- **RELEASE BLOCKER**：首发前必须解决——需**具体证据**表明当前首发行为破损、违反隐私/安全、数据完整性风险、
  实际分发缺所需许可、构建/签名/产物失败、架构违反已接受需求等。
- **VALIDATION DEBT**：已实现且无当前矛盾证据，只是额外测试仍可进行；**可推迟**。
- **FUTURE / DEFERRED**：非首发所需。
- “还能测更多 / 矩阵不穷尽 / 某 ROM 未测 / 缺自建服务器或真实凭据 / 长语音或扩展边界用例”**本身不构成 RELEASE BLOCKER**；
  晋升为 blocker 必须附上述具体证据。

## 修改边界

- **上游优先。** 上游实现或公开 API 能满足需求时直接复用，包括已有状态机、配置、UI 和测试基础设施。能力不足时，说明具体缺口，优先做最小的上游兼容扩展；复用与扩展均无法满足需求时，才采用最小替代实现。改动限于已有证据支持的层级和范围，避免平行状态机和过度抽象。fork 保持最小、聚焦且便于审查；扩展长期 fork 范围前，说明上游限制、替代方案和维护成本。
- **复用项目已有能力。** 优先复用上游和项目已有功能。在可行时，将确实等价的行为归并到现有公开接口，并保持调用方可见行为和依赖方向。无关行为各自保留。
- **单一权威与模块职责。** 每项事实、政策、配置值和实现职责各有一处权威归属。可调值放在所属配置中，以单一来源为准，并在读取或加载边界校验。每个模块承担一项内聚职责，模块间通过公开接口交互，接口须说明输入、输出和失败行为。采用前导下划线标识私有名称的语言中，这类名称限所属包内使用。
- **就近组织。** 相关代码、配置和文档放在同一处或相邻位置。其他位置需要同一事实时，简洁引用权威来源。
- **依赖方向。** 依照受 MIT Axiomatic Design 独立公理（Independence Axiom）启发的规则，层级顺序只在适用的架构权威中登记一次。跨层实现依赖向下，禁止任何长度的环，包括同层环。函数内的运行时导入计入依赖；仅供类型检查且运行时从不执行的导入不计入。可复用功能放在所有使用者都能到达的最低适用层；禁止为复用向上导入或通过函数内导入绕过依赖方向。
- **渐进整理。** 新增顶层包时，在权威层级清单中登记。按实际职责边界拆分大文件，将结构移动和行为修改分开。仅移动结构时须保持行为；有适用的确定性回放设施时，验证输出逐字节一致；否则使用现有产品路径验证，并说明实际证据和验证局限。
- **性能与稳定性。** 用代表性工作负载的测量指导性能优化和稳定性维护。性能改动须记录基线、指标、工作负载、测量结果及相关正确性检查。测量定位到算法瓶颈时，优先改进算法。
- **兼容代码退役。** 在未来有明确范围的改动中，退役仅服务历史格式、路径或名称的兼容代码。改变受支持的持久化格式时，先定义一次性迁移与明确的完成条件，迁移完成后再移除旧读取器、转发路径或别名。在获授权迁移或退役前，保护现有用户数据和已接受的外部兼容承诺。
- 保持触发入口、可配置实现和具体算法之间的既有架构边界；除非已有决定被正式修订，不把某个入口硬绑定到单一实现或供应商。
- 引入第三方数据或代码时，固定并记录来源版本，保留许可证和再分发信息；测试数据必须来自已确认的固定来源。
- Local ASR 模型的分发资格以 `DECISIONS.md` D059 的四项权利（bundle / mirror-rehost / 应用内 fixed-upstream 下载 / manual import）为单一权威记录，除非命中 D059 reopen conditions，不重开 A/B/C 研究阶段的分发结论或对当前三模型重复调查。

## 文档职责

- `docs/ROADMAP.md` 管理阶段、进度、待办、验证状态和下一步。进度变化应在这里更新。
- `docs/DECISIONS.md` 只记录已经接受的重要技术决定，不用于记录临时计划、未决方案、实现进度或未经验证的结论。
- `docs/REQUIREMENTS.md` 描述需求和约束；发现需求、决定、路线图与源码事实不一致时，应明确指出，不得静默选择其中之一。
- 文档中的完成状态必须有对应源码、测试或其他可核查证据。
- 固化新事实/决定/状态遵循下节「文档一致性维护（Documentation Reconciliation）」：先定位并分类旧语句，
  就地更新 canonical current state、协调交叉引用，不机械追加矛盾段落。详细执行证据放在 PR/证据文件，
  不在 ROADMAP/DECISIONS 重复堆叠。

## 文档一致性维护（Documentation Reconciliation）

文档维护不是只追加书写。这是跨项目长期规则，适用于 fusion-controller 及其管辖的每个产品仓库。
修改任何权威或 current-state 文档前，先定位并阅读现有支配同一 fact / status / decision / requirement /
procedure / architecture / current phase-stage / next step 的语句，再对每条受影响语句分类并相应处理：

- **CURRENT AND CORRECT**：保留。不得仅为新增一段更晚的话而在别处重复同一事实。
- **STATE CHANGED**：原地更新 canonical/current 语句；不得既保留旧的 active 语句，又追加一个与之矛盾的新语句。
- **OBSOLETE**：从 current-state / operational authority 移除。仅当丢失会导致重复调查、失去 provenance/可审计性、
  引发未来错误结论或抹掉有用历史证据时，才在历史/证据材料中保留；否则删除。
- **PARTIALLY SUPERSEDED**：显式标明仍有效的部分、被 supersede 的部分，以及管辖被 supersede 部分的新权威/决定；
  不得静默改写历史证据。
- **DETAILED EVIDENCE / TEST / PROVENANCE**：细节留在相应的证据/历史位置。current authority 通常只应含简洁的
  当前结论、current operational consequence，以及在有用时指向细节的指针。
- **DUPLICATED CURRENT FACT**：选定唯一 canonical current 权威；把其他 current-state 副本转为简洁指针。
- **MISLEADING TO CURRENT ACTION**：若旧措辞会导致今天执行错误动作，更新或删除它；在 stale 可执行文本下面
  追加一条免责声明并不足够。

历史证据与当前权威的治理性原则（governing principle）：

  Historical evidence preserves what was concluded at that time.
  Current authority is reconciled in place.
  Superseded or stale current-state statements must not remain active merely
  because they appeared earlier in a document.

编辑文档后，必须在受影响的权威面上搜索 stale 或矛盾的 current-state 措辞并就地协调，包括（where relevant）
旧 NEXT/PENDING/OPEN/CLOSED 标签、旧 stage/phase 状态、过时分支名、过时 current SHA/pin、被 supersede 的架构
语句、过时 next step、重复的 current-state 事实。不得机械改写历史证据。

## 实现与验证流程

- 修改应组成逻辑完整、范围清晰的小批次；避免把无关重构、格式化或文档调整混入同一批次。
- 修改完成后审查完整 diff，确认没有意外文件、无关改动、调试残留、生成物或敏感信息。
- 重要批次须保留累计实际 diff 交付物，并明确回答除请求行为外是否有额外行为变化；发现未经授权的业务、provider、fallback、lifecycle、migration、隐私或 trigger 变化时停止。

## Change Contract 与自主 CI 边界

- 重要任务先记录 `Goal`、`Allowed scope`、`Must preserve`、`Expected behavioral delta`；文档职责按需更新，不机械同步全部文档。
- 提交/推送前审查完整累计 actual diff、negative diff 和范围边界；diff 交付物必须对应 base→final，不能只交摘要。
- 只有根因明确、修复机械、仍在 Change Contract 内且不改变业务/架构/provider/fallback/lifecycle/隐私语义的 CI 失败，才可自主最小修复并复跑 CI。
- 若需扩大范围、削弱测试、绕过检查、改变行为或根因不确定，必须停止并请求审查；CI 绿不替代累计 diff 审查，也不授权发布。

## 三层验证与状态措辞

每个需要编译或测试的代码批次必须区分以下三层，不能用较低层级的结果代替较高层级：

1. **Gate 1 — Source / Semantic Review**：审查架构、范围、API 边界、行为语义、回归推理以及源码/调用链，回答“设计和实现方向从源码层面是否合理”。
2. **Gate 2 — Mechanical / Static Review**：审查完整实际 diff、`git diff --check`、本地可用的 formatter、namespace、include、声明/定义一致性、访问控制、const correctness 和明显类型/签名问题，回答“未执行真实 compiler/test 时是否发现明显机械问题”。
3. **Gate 3 — Real Validation**：按适用范围运行真实 compiler/build、unit/integration/product-path tests、GitHub CI 或 device tests，回答“是否经过真实工具链/产品路径验证”。

Gate 1 或 Gate 2 通过时，若没有实际运行 compiler/build，必须写明 `STATIC REVIEW PASS; COMPILE UNVERIFIED`（或等价表述）。人工 compile-oriented audit 不能替代 compiler，也不能升级为 `compile validated`、`compile-safe` 或 `build verified`。

本地能够验证的内容优先在本地验证；但必须先核对当前环境能力。缺少真实 build environment 时，不得假装完成 build validation，也不得为一次小任务盲目搭建重型环境，应明确记录 `COMPILE UNVERIFIED` 并选择可用的验证路由。按任务核对本机、Windows 开发机和适用 CI 的实际工具链与权限。

CI 只能作为 source review、actual diff review 和可用本地验证之后的 staged/final validation，不得作为逐个发现基础编译错误的猜测性试错工具。CI 失败只有在根因明确、修复机械且仍在 Change Contract 内时才可自主修复；formatter-only repair、未完成 CI 或 static review 均不得被写成 Gate 3 PASS，也不得改变尚未完成的阶段结果。
- 运行仓库可用且与改动相关的格式检查、静态检查、构建和测试；优先使用项目现有脚本与 CI 等价命令。
- 如因环境、依赖、权限、平台或时间限制未运行某项检查，必须如实列出未运行项目和原因。失败、跳过或未运行不等于通过。
- 不使用反复提交或触发远端 CI 的方式猜测性调试；先在本地完成可行的源码核对和验证。

## 提交与交付

- 按逻辑小批次提交，每个提交应可独立理解和审查，提交信息应准确描述实际改动。
- 提交或推送前重新检查状态与完整 diff，并确认目标远端分支未发生未处理的前移；不得强推，除非用户明确要求并已核对风险。
- 交付时报告实际修改、验证结果、未运行或失败项目、剩余风险，以及准确的提交号和推送状态。

## Workspace artifact lifecycle

Choose storage by the artifact's lifecycle, not by historical habit. This
section supersedes any active rule that describes `/tmp` as a combined raw
evidence and recovery workspace; historical evidence and reports are not
rewritten.

- `~/worktmp/<project>/<task>/` is the durable recovery workspace for
  CHECKLIST/recovery state, important raw evidence, diffs, patches, reports,
  results, or release evidence that must survive reboot, session loss,
  interruption, or loss of disposable scratch. It is not a permanent evidence
  repository and does not replace Git, a PR, or authoritative project docs.
- `/tmp/<project>/<task>/` is for disposable or regenerable build scratch,
  extraction, inspection copies, intermediate generation, diagnostics, and
  other short-lived material. Persistent NVMe does not make `/tmp` a recovery
  authority.
- `/dev/shm/<project>/<task>/` is only for small, short-lived,
  fully-disposable material when RAM-backed storage is specifically useful; it
  is not the default workspace.
- The Git repository, commit, PR, and accepted project documents remain the
  durable authority and normal code-review handoff. Source review uses the
  real pushed branch/PR/diff, not scratch artifacts.

## Execution Policy

- 对已批准且范围明确的工作，agent 应自主完成逻辑批次：源码/文档与完整 diff review、适用验证、必要修复、commit、push 及 CI/status 检查；不因 commit 或 push 另行请求许可。
- 仅在真实架构冲突、范围实质扩大、破坏性或高风险操作、凭据/秘密/签名/支付问题、未解决回归，或需要用户决定的架构选择时停止并报告。
- 本地 commit 不视为交付完成（push 可用时）；不得未经明确授权 merge `main`。
- 需要签名产物时按 D063 执行：普通 agent 会话无法安全访问本地发布凭证时记录
  `LOCAL_SIGNING = UNAVAILABLE_IN_CURRENT_SESSION`（正常、受支持状态），直接选择 CI fallback
  （`contribution/signed-rc-workflow` 的 dispatch+draft 路径），并以已记录 SHA-256 的同一产物做设备验收与
  晋升；不得反复尝试读取、导出或注入私有签名材料，也不得为此削弱任何安全约束。
- Debug/Test 签名按 D064 执行：本地使用 `DEBUG_SIGN_KEY_FILE` + `DEBUG_SIGN_KEY_PWD_FILE`（只传路径，
  口令与密钥内容不得出现在命令行、shell 历史、属性文件或输出中）；CI 继续使用 `DEBUG_SIGN_KEY_BASE64/PWD/ALIAS`
  Secrets。D058 Release 身份与 Debug 身份永不互用。
