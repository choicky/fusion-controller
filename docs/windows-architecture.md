# Fcitx5 Fusion for Windows — Current Architecture

> **状态：CURRENT NORMATIVE WINDOWS ARCHITECTURE AUTHORITY（D072，2026-10-07 起）**
>
> 本文回答“Windows 架构现在是什么、哪些已接受、哪些仍 OPEN/UNFROZEN、下一阶段做什么”。
> 接受该方向的决定见 `docs/DECISIONS.md` D072；阶段与进度见 `docs/ROADMAP.md` Phase 8。
> 本文只约束 Windows 线，不改变 Android 首发基线（D060 `docs/first-release-baseline.md`）。
>
> **Authority 迁移（D072）：** 本文取代 `researching-notes` 的
> `windows-ime/fcitx5-windows-current-architecture.md`（snapshot 2026-10-07，提交 `855ec99`），后者现为
> **SUPERSEDED / HISTORICAL**，保留原文不删除。本文 §1–§26 的编号沿用该 snapshot，便于逐节对照；
> 未在本文修订的条款按原意迁入并在此继续有效。

> **架构方向（D082；接受的决定见 `docs/DECISIONS.md` D082）：** Windows 线同时追求**先进、可靠的
> Windows-native 输入法架构**与**对 Fcitx5 的 faithful Windows port**，二者互补而非竞争。
>
> - **Fcitx WHAT（默认语义权威）：** Fcitx5 framework/engines（含 chinese-addons、LibIME）决定成熟的普通
>   输入法行为——Pinyin/Shuangpin、引擎 key-event 处理、consume/forward 语义、标点、QuickPhrase、
>   composition、候选生成/选择/部分选择、commit/preedit 行为及其顺序、前端回调/InputContext 契约、用户学习。
>   成熟的 Fcitx 功能、语义、前端契约、事件行为与实现机制默认保留并复用，不得仅为适配 Windows 抽象而改变。
>   Faithful port 不要求复制 Linux/Android/macOS 前端源码结构，而要求忠实实现 Fcitx 前端/InputContext 契约。
> - **Windows HOW（平台适配与可靠性）：** TSF hard policy 与 ownership 处置、context/document identity、
>   DLL/Host 隔离与 transport、causal ordering/barriers、generation fencing、合法 EditSession
>   materialization、recovery、bounded ledger/reliability、Host/link health、observability、
>   candidate/preedit/UI projection。本文 §1–§25 的 EVENT/STATE/EFFECT、A′、drain-before-PASS、watermark、
>   ledger、settlement 等机制属于 Windows HOW，可以且应当做得先进、稳健；它们承载 Fcitx 语义，约束其在 TSF 上
>   **如何**安全表示，不独立定义输入法**是什么**。“Windows 前端必须始终最薄”不是本文的原则；边界是 Windows
>   的复杂性不得在无证据时替代成熟 Fcitx 语义。
> - **偏离 Fcitx 的证据门槛：** 仅当具体证据证明至少一项——correctness defect、严重 portability defect、与
>   产品相关的重大缺陷、无法在 Windows 适配层内忠实解决的真实 Windows/TSF 不兼容、或既有 Fcitx API/前端边界
>   对所需 TSF 能力被证明不足——才改变 Fcitx 行为或实现机制，且改动限于该缺陷。“Fcitx 换种行为 Windows 实现
>   更容易”不构成证据。
>
> **Scoped supersession（仅此范围）：** “先扩展独立的 Windows key-admission / ownership 语义模型，再经该模型
> 适配或选择性暴露 Fcitx 行为”的实现方向，在其使 Windows 代码预测或替代 Fcitx 引擎语义的范围内 SUPERSEDED：
> Windows 侧分类器不得仅凭对引擎行为的预测阻止 Fcitx 看到一个输入，除非真实 TSF 约束要求 pre-admission 且其
> correctness boundary 已有证据。D072 原文中把 Fcitx5 / chinese-addons / LibIME 表述为临时 v0.x reference
> backend、把 Host 表述为独立语义权威的措辞已按 D082 在 §1、§2、§15、§25 原位收窄（§23 同步）。D072 的其余内容——thin TSF
> DLL / out-of-process Host 分离、上列 Windows HOW 机制、通用 SendInput replay 禁令——仍 ACCEPTED，不因其
> Windows 特有而失效。Stage 3C 前的证据保留为历史证据；Stage 3C 结果、Stage 4 与 Stage 5 收口及下一阶段
> （Stage 6）见 §22/§25。

## 0. 证据基础与证据等级

P0-C 收口的事实基础是 `fcitx5-windows-tsf-census`（本地 evidence 仓库，branch `master`）在
`328f5e6bbec33e05dd10f7066b470b529316a5fa` 时点的源码、原始 trace、文档截图与分析器输出。2026-10-07 已对其做
独立核查：在 Linux 上对存储的原始 trace 重跑全部 analyzer，摘要逐字节一致再生成；29 项测试 OK
（2 项需要 Windows 构建产物的 PE 检查 skipped）；报告提交未改动原始证据。核查未在 Windows 上重建或重跑实验，
DLL `.text` 一致性记录（`results/stage05d/observed-code-identity.json`）未独立复核。

| 阶段 | 提交 | 证据范围 | 主要结论（严格限定范围） |
|---|---|---|---|
| 0.5B | `419b73e`、`00c2150` | REAL TSF，自建 Win32 宿主中的 Microsoft RichEdit（`msftedit.dll`）+ 真实 `ITfContext` + COM 加载 TIP；按键为投递到自身消息队列并经 `ITfKeystrokeMgr` 路由的消息，非硬件/SendInput | 10/10：Space 回调内 `TF_ES_SYNC\|TF_ES_READWRITE` 写入并读回（outer/inner `S_OK`），回调返回后 Enter `OnKeyDown` FALSE（PASS），文档恰为 `P0C_EFFECT\r\n` |
| 0.5C | `30bbbf6`、`140b404` | 同上受控 RichEdit；C–H 为 harness 注入失败；另有 30 条 native 模型路径 | 27/27 新进程；真实 `TF_E_LOCKED`（`0x80040500`）×3 后 `TF_S_ASYNC`（`0x00040300`）async fallback 恰写一次；真实 `TF_E_DISCONNECTED`（`0x80040504`）×24；不确定写入 fenced、无 retry；未观察到 `TF_E_SYNCHRONOUS` |
| 0.5D | `8cbe13f`、`328f5e6` | REAL THIRD-PARTY：Notepad 11.2607.14.0、Word 16.0.17932.21000 的真实文档；测试驱动 `SendInput` 仅为测试输入 | 合格运行 4 次（每 target×策略 1 次）；初始 EFFECT 均为**刻意诱导**的 `TF_E_LOCKED`；drain 请求 3/3 outer/inner `S_OK`；EFFECT 3、真实 PASS 3、eaten 1、retained 0、replayed 0 |

证据等级区分（下文沿用）：

- **CONFIRMED EVIDENCE**：上表原始 trace/文档/截图直接支持的、限定于所列 target 与条件的观察；
- **ACCEPTED DECISION**：项目在证据之上作出的架构选择（D072），不是证据本身；
- **DEFERRED / UNKNOWN**：未观察；零计数表示“无观察”，不是“零风险”或“失败”。

**Stage 0.5D 自身的历史分类是 NOT-CLOSED**（`results/stage05d/target-census.json`），原因是该实验要求的收口范围包含
Chromium，而 Chromium 未取得合格运行，且缺少回调不可用/drain 未决时的已接受 recovery contract。该分类在其实验范围内
**仍然有效，不得改写为 CLOSURE-A**。P0-C 的关闭是 D072 的**项目决定**（见 §7），不是 0.5D 证据自身达成的收口。

## 1. 当前总体架构

```text
                    Windows Application
                            │
                            ▼
                ┌───────────────────────┐
                │ Thin Native TSF DLL   │
                │ Windows hard policy   │
                │ key ownership         │
                │ context identity      │
                │ ordering/barriers     │
                │ EditSession control   │
                │ projection            │
                │ bounded causal ledger │
                │ Host/link health      │
                └───────────┬───────────┘
                            │
                     bounded causal IPC
                            │
                            ▼
                ┌───────────────────────┐
                │ Fusion Input Host     │
                │ Fcitx5 hosting        │
                │ context routing       │
                │ backend integration   │
                │ recovery              │
                │ observability         │
                │ candidate data relay  │
                └───────────┬───────────┘
                            │
                     backend boundary
                            │
                ┌───────────▼───────────┐
                │ Fcitx5                │
                │ chinese-addons        │
                │ LibIME                │
                └───────────────────────┘
```

Fcitx5 + chinese-addons + LibIME 是 Windows 输入语义的**默认权威基础**（D082 faithful port）；替换或偏离只能满足
D082 证据门槛（§15 Exit Gate 信号属于此类证据候选），不是预设。

## 2. Authority model

```text
TSF DLL
    = Windows authority
    + ownership authority
    + projection controller
    + Windows-boundary causal ordering enforcement
    + 唯一的 document materializer（Host 只产生 proposal）

Fusion Host
    = session/context-routing authority
    + Fcitx5 宿主（承载而非重定义 Fcitx 语义）

Fcitx5
    = 成熟普通输入法行为的默认语义权威（D082 Fcitx WHAT）

LibIME
    = Chinese decoding / language-model authority

Application Document
    = materialized text authority
```

```text
Application Document = 已实际 materialize 的文本权威
Application Document ≠ 完整 semantic recovery log
```

## 3. Thin DLL 的边界

“Thin”不等于 stateless。DLL 可以且必须持有 Windows correctness 所需的最小状态：context identity；ownership
state；ordering/effect barrier；input watermark；bounded per-composition causal ledger/journal；pending
EditOperations；projection state；Host/link health；minimal recovery/accounting state。

DLL **不得**承载 dictionary/LM/candidate ranking/LibIME decoder/Fcitx addon business logic/ASR/networking/model
management，也不得演化成第二个输入引擎。

## 4. Correctness invariants

必须保持：

- zero silent loss of owned keys；
- zero silent duplicate materialization；
- zero duplicate irreversible EFFECT；
- zero cross-context EFFECT；
- zero stale-state resurrection；
- bounded queues / bounded waits（仅限 client 可控部分，见 §8）；
- deterministic failure behavior；
- no progressive degradation during long-running use；
- Host crash 不得导致目标应用崩溃；
- ownership 前允许 fail-open；ownership 后必须明确结算责任；
- Host restart 是 transport replay barrier；
- **A PASS key must never overtake an unresolved prior Fusion-owned document EFFECT in the same live causal
  context.**
- **Every owned key must reach an explicit terminal disposition.**

## 5. Ownership contract

> **TSF ownership != Fcitx event.accepted().**

```text
EAT(key)
⇒ Fusion owns responsibility for resolving that physical key.
```

Host/backend 对已经 owned 的 key 意外 reject 属于 **contract violation**，不等同于普通 TSF PASS，也不得被当作 PASS
处理。

通用 `EAT → backend reject → SendInput replay` 继续禁止作为恢复机制（产品 replay 政策见 §7.4）。

“任何情况下都 zero synchronous Host decision”已 **REJECTED as invariant**。D072 已接受：effect-capable
key 的 EFFECT 以 bounded synchronous materialization 为优先主路径（§7.1）；normal state-only typing 尽量异步。
**哪些 key 属于 effect-capable/boundary class 的精确分类仍 UNFROZEN**，在 Stage 1 core/simulator 中定义并由测试约束。

## 6. EVENT / STATE / EFFECT

### EVENT
Key/focus/context/reset 等因果事件。owned KeyEvent 不得 silent drop/coalesce。

### STATE
composition/preedit/candidate/mode/UI 等可替代状态。

- **async where possible**（D072 ACCEPTED）；在 causally safe 条件下可 stale-drop/coalesce/latest-wins；
- 必须保留 **consumed STATE → EFFECT watermark**：EFFECT 只能在其依赖的 STATE 已被实际消费（真实 edit 成功）后
  admission；watermark 初始为 0，EFFECT 不得依赖未消费的 STATE；EFFECT 之后旧 STATE 不得复活 composition。
  理由（CONFIRMED BY API CONTRACT + 证据）：TSF 同步请求先于 pending 异步请求执行，FIFO async 调度本身不能保护
  STATE 因果；Word 中一次 STATE 同步请求被拒（§7.6）时，watermark gate 正确拒绝了后续 EFFECT。
- 注意：0.5B–0.5D 的所有 STATE 都是**同步写入的实验校准**，“STATE async where possible”是 ACCEPTED DECISION，
  未在第三方应用中被实测证明。

### EFFECT
commit/不可逆 document mutation/destructive semantic action。

- ordered、uniquely identified、non-coalescible、never latest-wins；
- 必须到达 explicit terminal disposition；
- APPLIED 只在所需 mutation / composition 结束 / selection / readback 条件全部满足后记录；
- 新 generation 到来不得使 EFFECT 静默消失。

EVENT/STATE/EFFECT 是**语义类别**，不要求物理上三条 IPC channel。

## 7. P0-C — Key / Effect Ordering：A′ ACCEPTED，CLOSED by project decision（D072）

典型反例仍是 correctness contract 的定义来源：

```text
nihao → Space → Enter
Space 已导致 "你好" commit EFFECT，但 EFFECT 尚未 materialize；
若 Enter 直接 PASS，应用可能先收到 Enter，再收到 "你好"。
```

### 7.1 A′（ACCEPTED DECISION）

- **STATE**：async where possible；可在语义合法时 stale-drop/coalesce；保留 consumed STATE → EFFECT watermark。
- **EFFECT**：ordered / unique / non-coalescible；**优先主路径为 bounded synchronous TSF materialization（环境允许时）**：
  eligibility/admission → 校验 context/epoch/sequence 与 consumed STATE → 一次 `TF_ES_SYNC | TF_ES_READWRITE`
  → 实际 mutation + readback → explicit APPLIED → 回调返回 owned。成功的同步主路径在一个 key 回调内闭合因果链，
  正常路径无需 deferred later-PASS 存储。
- **Fallback / recovery：MANDATORY。** 同步请求被拒、上下文失效、不确定结果等必须进入显式 async/failure fallback
  与 recovery 语义（§7.3–§7.5）。A′ 简化正常路径，不删除任何 correctness 机制（§7.5）。

“bounded synchronous” **不表示** client 能在 N 毫秒后抢占一个正在执行的同步 TSF 调用。只对 client 可控部分设界：
admission、Host wait、queue size、fallback wait、operation count/payload、环境 eligibility / disable policy。
已开始的同步调用若超时或 pump 出重入工作，仍须由 partial-mutation / indeterminate 处理兜底。实验中的
30 ms / 150 ms / 500 ms / 15 s / 30 s 等数值均为 fixture 调度/收集参数，**不是**生产预算。

### 7.2 Later PASS while prior EFFECT is unresolved：drain-before-PASS（ACCEPTED，where lawful）

```text
prior EFFECT unresolved
    → later physical PASS key reaches TIP
    → callback detects prior unresolved EFFECT
    → drain prior EFFECT（同步 write session）
    → if APPLIED: return current original key as genuine PASS（not eaten）
    → application receives its original key naturally
```

- **K / `OnKeyDown`** = 在已测 Notepad + Word 总体上**最小已证实策略**（Notepad K 1/1、Word K 1/1 drain 成功）。
- **T / `OnTestKeyDown`** = **opportunistic**，仅在实际被投递时使用。Notepad 在观察到的路由中**完全不投递**
  `OnTestKeyDown`（trace 中无任何 Test 回调入口事件）；Word 投递，且 Word T drain 1/1 成功——Word 中
  `OnTestKeyDown` 返回 FALSE 后不再投递该 Enter 的 `OnKeyDown`。**correctness 不得依赖 T。**
- **不存在普遍回调保证**：不得表述为 “K always exists” 或 “sync EditSession always succeeds”。

### 7.3 回调不可用 / drain 无法在 PASS 前到达所需 terminal

若所选回调不可用，或合法 drain 不能在当前 key 可 PASS 前把 prior EFFECT 带到所需 terminal：

- 必须给出 **explicit recovery / ownership disposition**（例如 prior EFFECT `FAILED_NO_EFFECT` 或
  `INDETERMINATE`、当前 key 显式 `CANCELLED_RECOVERY`、epoch fence）；
- **不得**把被取消的 key 伪装为真实 PASS；
- 0.5D 证据只证明该处置**可观察、可审计**（Notepad T-only 1 次），不证明其用户体验可接受。

**具体 recovery contract（用户可见行为、通知、何时可恢复正常 ownership、与 P0-A 的 key policy 如何组合）未在本文冻结**，
分配给 **P0-A Recovery Key Policy**（§12）定义，并由 **Stage 1** sans-IO core/simulator 以 RG-0 用例（§20）实现与验证。

### 7.4 Replay policy（ACCEPTED）

- 无通用产品级 `SendInput` replay；
- 无通用 `PostMessage` replay；
- 不以合成 `WM_CHAR` / newline 重建作为正常 PASS 机制。

实验中的测试驱动 `SendInput`（census `src/stage05d/input_driver.cpp`，标注 TEST AUTOMATION ONLY）**不是**产品 replay。
0.5C 中在自建应用里保留原始 Enter MSG 并在释放后 dispatch 的做法依赖**合作宿主**，不是第三方应用中可用的产品机制，
不得作为产品 key 保留/释放能力的证据。

### 7.5 必须保留的公共机制（A′ 不删除）

context identity；lifetime generation；Host epoch；sequence/response validation；consumed STATE → EFFECT
watermark；explicit terminal disposition；duplicate/late/stale fencing；bounded admission/waits；reentrancy
protection（未观察到 key 重入不代表 Windows/TSF 保证不重入）；Host restart/reset policy；需要时的 pending ownership
ledger；proposal ticket/lifetime validation（ticket 创建时捕获、回调入口重新校验）；indeterminate mutation
reconciliation；**不确定的不可逆 mutation 之后不得盲目 retry**（失败的 edit 不是 rollback）。

**FORCE_ASYNC** 保留为 diagnostic/development capability：用于 CI、simulator/model、soak、recovery 与兼容性工作，
使 fallback/recovery、pending ownership、fencing 不成为未测冷路径。它**不是**生产正常路径；strict FORCE_ASYNC 从不
静默同步 drain。`ASYNC_THEN_DRAIN` 是另一个独立命名的 race 模式。

### 7.6 Compatibility evidence（at architecture-freeze time）

| Target | 状态 | 依据与限制 |
|---|---|---|
| Microsoft RichEdit（自建宿主） | **demonstrated, controlled real-TSF** | 0.5B 10/10 + 0.5C 27/27 + 0.5D fixture 24/24；非第三方产品 |
| Windows Notepad 11.2607.14.0 | **demonstrated third-party drain-before-PASS（K）** | 1 次 K 成功；T 回调不投递；单样本，非总体证明 |
| Microsoft Word 16.0.17932.21000 | **demonstrated third-party drain-before-PASS（T 与 K）** | T 1 次、K 1 次成功；单样本 |
| Chromium-family（Edge/Chrome 等） | **DEFERRED UNKNOWN** | 0 次合格运行，无任何 TSF trace；Edge 尝试因自动化工具 URL-policy 检查在 TSF 实验前停止。这是工具限制，**不是 Chromium TSF 失败**，也不证明不支持 drain-before-PASS |
| VS Code、Windows Terminal、Excel、Firefox、WinUI 3、UWP/AppContainer、classic EDIT、RDP | **DEFERRED UNKNOWN** | 未取得观察（VS Code 未找到；Terminal 未选定无执行风险的文本面；其余未尝试） |

另须保留的观察（不属于合格 EFFECT/drain census）：Word 首次激活/预检时，一次 **STATE** 同步写请求返回 outer
`S_OK`、inner `0x80040208`（`TF_E_SYNCHRONOUS`），未执行 DoEditSession，watermark 保持 0，后续 EFFECT 因 watermark
被拒，Enter 显式 recovery（census `results/stage05d/word-initial-state-rejection.jsonl`）。它**证明**：Word 至少在一种
（焦点/路由未合格的）状态下会拒绝 key 回调内的同步写会话，且 watermark fencing 按设计工作；它**不证明**：Word 正文的
同步拒绝率、EFFECT 写入在 Word 中失败、或其原因（可能是 Search 焦点）。0.5D 的 pending 频率（4/4）是刻意诱导的，
不是自然拒绝率；自然拒绝/fallback 频率 UNKNOWN。

禁止表述：“all Windows applications supported”、“Chromium supported”、“K always exists”、
“sync EditSession always succeeds”。Chromium 等兼容性可在后续真实 Windows TIP 实现/集成阶段（Stage 2.5/4）、
在更便宜且更具代表性时重新收集。

## 8. EditSession model

不假设 `1 Host operation = 1 async RequestEditSession`。每 context 保存 ordered immutable EditOperation intents；
EditSession 是**执行机会**：

```text
Host output → enqueue EditOperation → RequestEditSession → grant
  → revalidate context / generation / epoch / applicability
  → drain causally applicable FIFO operations → apply
```

- A′ 下 EFFECT 的正常路径是 key 回调内的同步 grant（§7.1）；排队与异步 grant 是 fallback/recovery 路径与 STATE 路径；
- 同时检查 `RequestEditSession` 的 outer 与 inner（`phrSession`）结果：outer `S_OK` 不足以证明执行；
  `TF_S_ASYNC` 是 admission，不是 completion；outer 失败时 inner 只作诊断；
- STATE 可在合法条件下变 no-op；EFFECT 不得因新 generation 静默消失；
- 不在等待 IPC 时持有 TSF document lock（能先取得决定时）。

## 9. Focus / Epoch / composition termination — 部分已接受（D083），其余 OPEN

```text
focus ≠ semantic incarnation
```

focus loss 本身不自动 increment Epoch，也不等于 composition termination。`OnCompositionTerminated` 属于外部
authority event，必须触发 projection/pending operation/semantic binding 的 reconciliation；只有真正 semantic
discontinuity 才推进/重置 epoch。该原则已由 Stage 4C 在真实产品路径上实测确认（Notepad：focus out/in 与
thread foreground 不结束、不重置、不重绑 Fusion context）。

**已接受（D083，Stage 4 收口）——被 pop 的 context-stack 成员。** `OnPopContext` 结束/对账的是**该被 pop 的
context 所对应的当前 Fusion semantic incarnation**：进入 Stage 4C 的 teardown 路径（先审计仍未决的责任，再把
ledger fence 到显式 terminal、unbind Host context、递增 entry generation 使迟到的 edit session/pump 无法匹配，
并留下一条 `CONTEXT_TEARDOWN` 审计记录）。该决定的定界：

- **不**表示整个 DocumentMgr 被销毁——DocumentMgr 销毁是另一条 `OnUninitDocumentMgr` 路径；
- **不**隐含对另一个 context（回落到栈顶的 underlying/main context）做 teardown；
- 同一 `ITfContext` 之后被再次 push/使用时，按 Fusion 当前生命周期契约建立/绑定新的 semantic incarnation，
  不得复活 stale Host state（由 Stage 4B 的 Host incarnation / connection generation fencing 承担该屏障）；
- 证据：Microsoft TSF 的 context-stack 契约（应用与文本服务都通过 `ITfDocumentMgr::Push`/`Pop` 增删栈成员，
  文本服务的模式 UI 也经自己的 context 进出栈，因此 pop 一个栈成员不等于销毁文档或整个栈）、成熟 TSF 实现
  先例（Mozc 在 Push 建立、在 Pop 移除 per-context 私有状态）作为经独立评审的支持理由，以及 Fusion 自身的
  Stage 4C 实现与产品路径证据（fusion-windows PR #10：Notepad 实测 `CONTEXT_POP` 先于 `DOCMGR_UNINIT`、
  每次 teardown 记 `CONTEXT_TEARDOWN … clean=1`，且关闭持有 open composition 的文档时**不**投递
  `OnCompositionTerminated`）。

**仍 OPEN / UNFROZEN。** 除上述被 pop 的栈成员边界之外，哪些 TSF lifecycle event 构成 semantic discontinuity
仍须由真实执行取证：transitory context、context reuse/discontinuity、async 请求 pending 时的自然销毁/停用，
以及 Notepad 之外应用的 lifecycle 序列均未取证。0.5B–0.5D 只观察到受控的 deactivation fence、销毁 context 上的
`TF_E_DISCONNECTED` 与一次 Word `composition_terminated`（§18）。

## 10. Windows hard policy

至少纳入 `GUID_COMPARTMENT_EMPTYCONTEXT`、`GUID_COMPARTMENT_KEYBOARD_DISABLED`、
`GUID_COMPARTMENT_KEYBOARD_OPENCLOSE`、`GUID_COMPARTMENT_KEYBOARD_INPUTMODE_CONVERSION`、context validity、
input scope/private/password constraints、preserved/system keys、composition validity、Host/link health。
backend 不得覆盖 Windows hard policy。

**U6（2026-10-19 决定）：** 新打开的应用窗口默认以 Fusion 打开。实现只在**新 context 首次出现**时把该线程的
`GUID_COMPARTMENT_KEYBOARD_OPENCLOSE` 声明为开（每次 context 一次），不在每次按键或焦点切换时重写，因此用户在同一
context 内用 Ctrl+Space 关闭仍然有效。诊断证据：激活时的写入会在 Windows/应用为新 context 重新初始化该 compartment 后
丢失（`KEYBOARD_OPEN declared` 之后、新 context 的第一个按键读到 `HARD_POLICY openclose=closed`，中间没有本服务写入或
`Deactivate`）。compartments 仍是 Windows 的权威；本项只是声明新 context 的默认值。

## 11. Context identity — wire set UNFROZEN

`ITfContext` incarnation，而非 HWND，是 context identity 的基本对象。A′ 要求以下**概念**存在并被校验：context identity、
lifetime generation、Host epoch、input sequence、response identity、consumed STATE watermark；其余可能需要的因果身份包括
profile/context/composition/result generation、effect_id、operation_id、HostInstanceId、ConnectionGeneration、
ClientInstanceId、LocalContextId。

以下仍 **UNFROZEN**：permanent identifiers / numeric IDs、exact wire identifier set、final wire format、尚未决定的
Host topology 细节、production timing thresholds、UI。原则：先保留因果区别，待证据证明可合并；不过早冻结过度复杂的
wire schema。

## 12. Recovery — P0-A / P0-B — OPEN

### P0-A Recovery Key Policy — OPEN

候选：immediate fallback；tiny bounded recovery buffer；bounded per-composition causal journal；hybrid。不得凭直觉冻结
8 keys / 200 ms / 500 ms 等常数。

**新增分配（D072）：** P0-A 负责定义 §7.3 的 recovery contract——所选回调不可用或合法 drain 无法在 PASS 前到达所需
terminal 时，当前 key 与 prior EFFECT 的用户可见处置；不得把取消的 key 伪装为 PASS，不得使用通用 replay（§7.4）。

### P0-B Recovery Representation — OPEN

至少评估 Pinyin、Shuangpin、MoQi auxiliary filtering、partial selection、未来 mixed Chinese-English。可能需要
composition identity、recoverable raw semantic input、cursor、selected/committed prefix boundary、remaining raw
input、mode/scheme metadata、checkpoint/input watermark。不得序列化任意 Fcitx/LibIME object graph。

## 13. Host reconnect

```text
DISCONNECTED → CONNECT → HELLO / NEGOTIATE → PROFILE SYNC
  → CONTEXT REBIND → RECOVERY RECONCILIATION → READY
```

pipe connected ≠ semantic ready；readiness 可以 per-context。Host incarnation change 继续作为 transport replay barrier，
不盲目 replay 旧 incarnation 中状态不确定的 KeyEvent/EFFECT。

**Stage 4B 已把该阶梯接上真实产品路径（fusion-windows PR #9）。** client 侧传输状态显式化为
`Disconnected → Connected → Established`（transport up / handshake+incarnation known），"semantic ready" 是其上
一个独立的 **per-context** 条件：绑定必须属于当前 Host incarnation *且* 当前 connection generation，因为 Host
会在连接断开时 unbind 该连接的全部 context，并把新连接重映射为新的 Host context id。重连由既有 key/context
执行路径驱动，bounded 且幂等（transport connect 加一次 Hello 往返），无后台线程、无 busy loop；连接失败即
FailOpen，key 保持应用所有，因此 Host 崩溃不能崩溃目标应用。§11 早已命名的两个因果身份——Host incarnation 与
ConnectionGeneration——由此成为可测的 fencing 身份（backend `session_id` 在每个 Host 进程都为 1，不足以区分
incarnation）。断开时未决的 ownership/uncertainty 记账与从未下发的 materialization queue 被刻意保留，Core 的
`probe_pass`/watermark 仍阻止后来者越过未决 EFFECT。Host restart 因此不再只是模型级验证：真实 `FusionHost.exe`
kill/restart 的确定性测试与 Notepad 产品路径运行均已取得。最终用户可见的 recovery 语义仍属 P0-A / P0-B。

## 14. P0-D — Windows Trust / IPC Topology — OPEN

必须实测/设计：Medium IL；High IL；x86/x64；AppContainer；LPAC where relevant；multiple
interactive sessions；RDP；Fast User Switching；runas/elevation transition。

`one Host per user × session` 为 **baseline, not frozen**；Named Pipe 为 **v0.x baseline, not permanently frozen**
（需考虑 explicit SD、local-only、session/logon isolation、message size limits、parser fuzzing、quota、client/server
identity verification、pipe squatting protection、AppContainer compatibility）。Supervisor/broker/multiple trust-class
Hosts 仍为 experiment，仅在具体场景证明简单 topology 不足时引入。

**已核查的注册/提权观察（0.5B/0.5D，scoped）：** 非管理员 token 下 HKLM `CTF\TIP` create access 为
`0x80070005`，默认作用域/持久 `RegisterProfile` 与 legacy `Register` 返回 `E_FAIL`（`0x80004005`）；HKCU COM +
process-local `RegisterProfile(TF_RP_LOCALPROCESS)` / `TF_IPPMF_FORPROCESS` 只在自身进程内可用，不会让 Notepad 等其他
进程加载。经用户授权的 UAC 提权后，公开 API 的持久注册（RegisterProfile、EnableLanguageProfile、category 注册）+
HKLM/HKCU COM 使跨进程加载生效。这是安装/拓扑输入，**不是**“所有 TIP 都必须提权”的结论；P0-D 仍 OPEN。

**启动机制：已实现基线（WP4，U2 (b) Accepted 2026-10-18）。** 在 §3 目标场景（Windows 11 x64、单个交互会话、标准
用户 Medium IL）内，`FusionHost.exe` 由**每用户登录计划任务**启动：principal 为安装用户、interactive token、
`RunLevel Limited`（Medium IL，从不提权）、无执行时限、`MultipleInstances=IgnoreNew`、**不**设置崩溃重启。任务只是
启动者；DLL 仍然从不启动 Host（P0-D 的该子问题不变）。打包 Host 无控制台（Windows 子系统），因此登录/注销/关机的
保存由隐藏的**顶层**窗口的 `WM_QUERYENDSESSION`/`WM_ENDSESSION` 触发（message-only 窗口收不到会话结束广播），
在 4 s 界限内等待既有的有序关闭（request stop → stop → shutdown_backend → Fcitx save）；安装/升级/卸载通过
`Local\FusionHost-stop-<user SID>-<session id>` 命名停止事件（与 pipe、HostClaim 相同的 user-only DACL）请求同一
有序停止，而不是终止进程；`WM_QUERYENDSESSION` 只回答 TRUE 而不停止任何东西（注销仍可能被取消），只有
`WM_ENDSESSION(TRUE)` 才请求停止。三条路径都只**请求**停止，`main()` 仍是唯一执行有序关闭的线程，`HostClaim` 持有到进程
退出；`one Host per user × session` 的权威仍是 `HostClaim`，不是任务设置。开发/测试用的控制台模式（`--console`，
含 control handler）保留。证据与边界见 `docs/windows-stage6-wp4.md`；U2 (a)（登录 Run 项）未实现，其缺口已记录。
**自动启动：**登录时启动已验证（2026-10-18 18:58:43，以及 2026-10-19 走查（run `wp4-run-20261019-130037`）的 (c) 登录）。
**取消注销/重启（R34）：已解决 —— 2026-10-19 走查验证（run `wp4-run-20261019-130037`）：取消注销 (c2) 与取消重启 (c3) 均在同一会话内恢复 Host；取消关机由 1073 测量覆盖。
2026-10-18 的走查中，注销被取消（User32 1077，19:03:20）后 Host 已经结束、没有任何东西重启它；修复是任务上的 1077（取消注销）或 1073（取消重启/关机）事件触发器，已在上述走查中验证。本段描述的是启动机制的设计与配置。

**手动重启与诊断（WP7，2026-10-19）：** 包内 `restart-host.ps1` 只重启**当前用户 × 当前会话**的 Host：先经命名停止事件请求
有序停止并在界限内等待；仍未退出时**仅**强制结束已核验为本用户/本会话的实例并明确报告（该实例不会执行 Fcitx 自身的保存）；
随后用既有登录任务启动（Medium IL），再核验可执行文件路径、完整性与"恰好一个实例"。它不是新的启动机制，不启用自动重启，
也不改变任务的崩溃重启设置；其他用户/会话的进程永不触碰。安装创建开始菜单快捷方式（`Fusion\Restart FusionHost`），卸载删除。
`host-diagnostics.ps1` 是 R30 的显式 opt-in/opt-out（用户级 `FUSION_HOST_DIAGNOSTICS`），并**验证实际启动路径是否收到该设置**
（重启后检查新 pid 的日志文件），产品默认仍为关闭。日常使用说明见 `docs/windows-daily-use.md`。

**P0-D 仍然 OPEN 的部分：** 上述只是 §3 目标场景内的已实现基线，不是 Host topology 的冻结。High IL、x86/x64 混合、
AppContainer/LPAC、RDP、多会话、Fast User Switching、runas/提权转换、崩溃重启与 alive-but-unhealthy 恢复（R7/WP5）
都仍未取证；每个被解除的排除项都需要自己的证据。计划任务/停止事件/pipe 的“单用户 × 单会话”范围意味着多交互用户
机器**当前不受支持**：任务名 `FusionHost` 是全局的，第二个账户安装会覆盖第一个账户的任务（R32，Hypothesis，
CONDITIONAL；候选修法是按 SID 的每用户任务名）。以*另一个*管理员账户提权的安装器也打不开属主的停止事件——它如实
报告，而不是杀进程。

## 15. Backend policy

Fcitx5 是 faithful port 的语义基础（D082；Pinyin、candidate behavior、punctuation、QuickPhrase、configuration、MoQi、
lifecycle、user dictionary、prediction、mode behavior）。

**Entry Gate**：能否在 Host 中干净嵌入，同时满足 Fusion ownership/lifecycle/ordering/reliability contract？

**Exit Gate**：若出现以下证据，应认真转向 Native Fusion Runtime + chinese-addons/LibIME：大量 Fcitx internal state 被迫
泄漏到 DLL；normal typing 经常依赖 synchronous Fcitx decision；Fcitx lifecycle 无法可靠映射 TSF context；为维持 Fusion
authority 需要反复 patch/deception/bypass；Fcitx 明显恶化 tail latency；soak degradation 根因位于 framework；
Windows correctness 与 Fcitx 结构性冲突。上述信号是 D082 证据门槛下的证据候选，须由具体证据确立并逐项判断，
不是默认方向。

A′ 的同步 EFFECT 主路径使 effect-capable key 的 backend 尾延迟成为直接约束：Windows 上 backend 延迟尚未测量
（census 中的 Linux 后端数据为所有者提供、未在 Windows 复测）。LibIME 的调整同样服从 D082 证据门槛。

## 16. Candidate UI — UNFROZEN

candidate renderer process/location 未冻结；至少比较 Host-owned native window、TSF UIElement、minimal specialized
in-process presentation surface。不得为了 UI 简单而把 backend semantics 搬入 TSF DLL。

Stage 5 契约输入（research 级，非 Accepted Decision）：`research/windows-stage5-candidate-contract-inputs.md`
——一手 Fcitx5 frontend 契约（5.1.23-11）与 fcitx5-android / fusion-windows `CandidateSnapshot` 的字段与生命周期
mapping；D082 下 Windows 只运输/投影 Fcitx 已定义语义，候选 select/page 等动作须经非键 semantic stimulus 回送
引擎，不得以合成键 replay 模拟。

fusion-windows PR #11（已合并，`main` `3e7fc35`）实现了首个最小 in-process presentation 接缝（candidate popup 经
`ITfContext::GetActiveView` 取权威 active view 锚定；焦点丢失仅隐藏 presentation）。该接缝是本节允许的
"minimal specialized in-process presentation surface" 的一个候选实现，**不决定也不冻结** renderer 的最终
进程/位置；§16 仍为 UNFROZEN。

## 17. State representation — UNFROZEN

旧 flat FSM（DISABLED / PASS_THROUGH / READY / COMPOSING / RECOVERING / DEGRADED_PASS）不永久冻结；应与
`Mode × Composition State × Link/Host Health` 比较，重点是暴露非法组合。

## 18. Stage 0.5 — TSF Behavior Census

**解决 P0-C 所需的 Stage 0.5 工作：COMPLETE（0.5B / 0.5C / 0.5D，§0）。** 这**不是**“全部 Stage 0.5 census 完成”。

不属于已接受 P0-C 决定所需、仍为 **DEFERRED / UNKNOWN** 的 census 项：

- 应用：Chromium-family（**DEFERRED UNKNOWN**）、Excel、Firefox、VS Code/Electron、WinUI 3、UWP/AppContainer test
  host、classic EDIT、Windows Terminal、RDP；
- 行为：OnTestKeyUp/OnKeyUp 全面分布、GetTextExt / TS_E_NOLAYOUT、transitory contexts、context
  reuse/discontinuity、async 请求 pending 时的自然销毁/停用、自然同步拒绝率与 fallback 频率、wider
  latency/document 分布、key 回调重入。
- 行为（Stage 0.5 census 时未测，现已在 **Notepad 产品路径**取得，见 §9/§25；其余应用仍未测）：
  `OnPushContext`/`OnPopContext`/`OnSetFocus`（含 thread foreground 变体）与 TSF compartments 的实际路由
  （Stage 4C：push 早于任何 context 存在、pop 先于 `DOCMGR_UNINIT`、focus out/in 不结束也不重绑 context、
  hard policy 为 per-context 且 closed 模式一律 FailOpen）；`OnCompositionTerminated` 的**外部终止**分支
  已取得，并证明它在 context 销毁时**不保证**投递（关闭持有 open composition 的文档只到达 pop/document
  teardown），其在其余应用上的正常路径分布仍未测。

这些剩余项在后续真实 Windows 兼容性工作（Stage 6 的应用广度门）或 §9 / P0-D 需要时收集；Stage 2.5 / 3 / 4
已合并的事实不改变它们仍未取证。每条证据继续记录 application、Windows build、
architecture、callback sequence、thread、timestamp、EditSession mode/HRESULT、context identity、focus/composition/
compartment state；不得从单个应用行为推断普遍 TSF 规律。

## 19. Portable correctness core（Stage 1）

```text
FusionCore    protocol semantics / causal ordering / reliability state / watermarks /
              ledger/journal / bounded queues / reconnect/recovery logic
FusionMock    deterministic simulator / mock backend / fault scheduler / property tests
FusionWindows TSF / Win32 adapter
```

portable abstraction 不得抹掉真实 Windows/TSF semantics（outer/inner HRESULT、`TF_S_ASYNC` ≠ completion、
同步请求先于 pending 异步请求、回调可用性因应用而异）。

## 20. Deterministic oracle / RG-0

建立 deterministic synchronous-reference oracle，与 production async model 对照。RG-0 至少验证：

```text
silent owned-key loss = 0
duplicate irreversible EFFECT = 0
cross-context EFFECT = 0
stale-state resurrection = 0
illegal causal reorder = 0
queue bound violation = 0
PASS overtaking unresolved prior EFFECT = 0
owned key without terminal disposition = 0
cancelled key reported as PASS = 0
blind retry after indeterminate mutation = 0
```

fault schedule 覆盖 ownership→IPC→Host→output→EditSession→document mutation→completion 各窗口，以及 Host
crash/restart、focus/context churn、external termination、sync EditSession refusal（`TF_E_LOCKED`、
`TF_E_SYNCHRONOUS`、`TF_E_DISCONNECTED`）、compartment changes、**T 回调缺失 / 仅 K 可用**、**drain 未决**、
FORCE_ASYNC 路径。

## 21. Observability

从 PoC 第一日 trace：physical key arrival → ownership → terminal disposition → IPC enqueue → Host receive → backend
begin/end → Host output/watermark → EditOperation enqueue → EditSession request（outer/inner）/grant → revalidation →
document mutation → readback → EFFECT_APPLIED；并记录 drain 尝试与所用回调（T/K）、disconnect/restart/rebind/recovery、
external termination、compartment changes。测 P50/P95/P99/P99.9/max 与 queue high-water、pending effects/ops、Host
restarts、protocol mismatch、recovery outcomes、unexpected owned-key failures、stale drops、cross-epoch rejects、duplicate
prevention、RSS/handles/threads。具体 latency/buffer/timeout 数值是 engineering hypothesis，必须测量后冻结。

## 22. Stage order 与状态

```text
Stage 0    Architecture closure                         COMPLETE（D072）
Stage 0.5  TSF Behavior Census — P0-C 所需部分          COMPLETE（0.5B/C/D）
           其余 census 应用/行为                         DEFERRED / UNKNOWN（§18）
Stage 1    Portable sans-IO FusionCore                  COMPLETE（fusion-windows PR #1，已合并）
           + deterministic simulator + RG-0
Stage 2    Mock Backend + Host 边界                      COMPLETE（fusion-windows PR #2，已合并）
Stage 2.5  Real TSF + Mock Backend（representative Windows applications；Chromium 可在此重新取证）  COMPLETE（fusion-windows PR #4，已合并；Edge 尝试记为 HARNESS/TOOL INVALID）
Stage 3    Real Fcitx5 + chinese-addons + LibIME（faithful port：经 Fcitx 前端/InputContext 契约承载引擎语义）
           ├─ 3B Real-Fcitx5 backend PoC / Option-A frontend 边界验证  COMPLETE（fusion-windows PR #5，已合并）
           └─ 3C Faithful Fcitx Windows frontend 路径  VERIFIED / COMPLETE（fusion-windows PR #7，已合并；
              物理按键验证范围见 §25）
Stage 4    Production Host / TSF lifecycle integration      COMPLETE / VERIFIED（4A PR #8、4B PR #9、
           4C PR #10 均已合并；exit gate 的实际证据范围与明确未主张项见 §25）
Stage 5    Composition / candidate UI                         VERIFIED / COMPLETE（在 §25 记录的范围内；
           PR #11 与 PR #12 均已合并，canonical `main` `0f724af` 的 tree = 经评审与 exact-HEAD
           Notepad A–E smoke 的 `670dfc0`；controller PR #29 已合并；未主张项与 DEFER 项见 §25）
Stage 6    Packaging / release / soak                       PLAN PROPOSED — 未接受；WP1 已合并并在其范围内收口；
           WP2、WP3 已在其范围内收口（所有者 2026-10-18 决定 U3 (a)、U5 (a)、U1；WP3 运行于新 runtime
           `runtime-wp3-20261018`）；WP4 已在 5700X 上实现并本地验证（U2 (b) Accepted 2026-10-18；B1 71/71 Mock-only、
           79/79 Debug、79/79 Release；两个测试包；A1–A6 与走查试运行见 `docs/windows-stage6-wp4.md`；
           该机的 runtime-wp3-20261018 是按 WP3 pins 重建的 5700X 版本，与 7700 构建非字节相同），
           但**所有者走查 B3/B4 未运行**，故 WP4 未收口；后续工作包未授权
           （`docs/windows-stage6-plan.md`、`docs/windows-stage6-wp3.md`、`docs/windows-stage6-wp4.md`）
P0-A / P0-B / P0-D 仍 OPEN（§24）
```

**下一阶段边界。** Stage 6 仍未接受为计划，WP4 之后没有授权的工作包。WP4 的收口只差其所有者走查（安装、第二次启动、
注销/登录、在记事本保持打开时升级、两条卸载路径、已安装 wp4.2 上的 Notepad A–E）；这些是 §14 启动基线与 R26 的
部署侧证据，不是新的架构决定。WP5（失败与恢复，R7/R22）只有在 WP4 走查收口后按自己的执行指令开始，并且它是 U7 的
证据来源；WP6/WP7 依赖 WP4 退出；WP8 还依赖 U4（签名/渠道）与 U7。
## 23. Decision classification

### ACCEPTED（D072）

- A′：STATE async where possible；EFFECT bounded-sync preferred primary where lawful；fallback/recovery mandatory；
- consumed STATE → EFFECT watermark；
- drain-before-PASS where lawful；K 为最小已证实策略，T opportunistic；
- no generic product SendInput / PostMessage replay；no synthetic WM_CHAR/newline as normal PASS；
- FORCE_ASYNC as diagnostic/development capability；
- 公共机制全部保留（§7.5）。

### ACCEPTED（D083，Stage 4 收口）

- 被 pop 的 context-stack 成员的处置与定界（§9）：`OnPopContext` 结束/对账该栈成员对应的当前 Fusion
  semantic incarnation，不销毁 DocumentMgr、不 teardown 其他 context，之后同一 `ITfContext` 再使用时不得
  复活旧 Host state；
- Stage 4 exit gate 的评估结论（§22/§25）：产品拓扑、重连/fencing、生命周期与 write-certainty 边界在
  Notepad 真实产品路径上成立；多应用 out-of-process 再验证**不**在主张范围内。

这是对 D072 §9 “哪些 TSF lifecycle event 构成 semantic discontinuity 仍 OPEN” 的**部分** supersession：
仅被 pop 的栈成员边界被接受，§9 列出的其余子边界仍 OPEN。

### KEEP / strengthen

TSF primary；Thin TSF DLL + out-of-process Host；Host crash must not crash app；TSF ownership != Fcitx accepted；
fail-open before ownership；explicit responsibility after ownership；EVENT/STATE/EFFECT；document = materialized-text
authority；Host 承载 Fcitx 语义（D082）；per-context causal serialization；bounded waits/queues；Host restart = transport
replay barrier；zero duplicate irreversible effect；zero cross-context effect；zero stale resurrection；no PASS overtakes
unresolved prior document EFFECT；every owned key reaches terminal disposition。

### UNFROZEN / OPEN

exact causal identifier set / permanent numeric IDs；exact flat FSM；candidate renderer location / UI；one Host per user
× session 与其他未决 Host topology；exact effect-capable key classification；exact recovery representation（P0-B）；
RECOVERING new-key policy 与 §7.3 recovery contract（P0-A）；lifecycle discontinuity（§9；D083 只接受被 pop 的
栈成员边界，transitory context、context reuse 等其余子边界仍在此列）；final wire protocol；final IPC
topology（P0-D）；numerical timeout/buffer/latency/admission thresholds；environment eligibility/disable policy 的具体规则。

## 24. P0 set

```text
P0-A  Recovery Key Policy             OPEN（含 §7.3 recovery contract）
P0-B  Recovery Representation         OPEN
P0-C  Key / Effect Ordering Contract  CLOSED by project decision（D072；A′ ACCEPTED；Chromium DEFERRED UNKNOWN）
P0-D  Windows Trust / IPC Topology    OPEN（含 §14 注册/提权观察）
```

## 25. 当前允许与禁止的工作

**Stage 3C 结果（VERIFIED / COMPLETE，fusion-windows PR #7）：** Stage 1–3B（PR #1 / #2 / #4 / #5）与 Stage 3C
（PR #7）均已合并。PR #7 经完整独立评审；real-Fcitx Debug 测试 42/42、Mock-only 39/39 通过；所有者在 Notepad、
Firefox 与 Edge 的地址栏及文本框中完成物理按键验证（例如 `nihao,` → 恰为 `你好，`，无重复 ASCII 逗号、丢标点、
preedit 残留或崩溃；代表性真实 Forward 恰好送达应用一次）。该范围**不**推广为通用 Windows 应用兼容性。

- **历史（Stage 3C 前）：** Windows 路径曾在 Fcitx 看到按键之前预测语义归属，可使逗号等普通按键到达不了 Fcitx；
  PR #7 在已实现的 faithful frontend 路径上消除了这一已知偏离。
- **已确立：** 普通的、经翻译的 IME 相关按键到达真实 Fcitx，由 Fcitx 的 consumed/Forward 结果决定语义处置；
  Windows 保留 hard-policy / routing eligibility 职责。一次 key dispatch 的有序回调（如 `CommitText("你好")` →
  `SetPreedit("")` → `CommitText("，")`）经既有 D072 机制作为独立的有序 Core 记录/结算承载；不需要一键一 Effect，
  不需要新的 FcitxTransaction 语义抽象；真实 Forward 不经通用 SendInput/PostMessage replay 恰好送达一次；未修改
  Fcitx / chinese-addons / LibIME 语义。该实现证据支持 D082 边界，而非修订它。
- **仍被拒绝（历史）：** 先前 Stage 3C Change Contract 的提议（consumed + CommitText 合并为一个 Effect、一个物理键
  对应一个语义 Effect、为适配 envelope 折叠有序 Fcitx 回调、新的 FcitxTransaction 抽象）以及 Windows-first
  key-admission 语义模型（L1/L2/L3 分类、comma ownership policy、Forward+effects、Mechanism A/B 等），仅在满足
  D082 证据门槛时才可作为条件性平台工具重新考虑。

**Stage 4 — Production Host / TSF lifecycle integration：COMPLETE / VERIFIED（fusion-windows PR #8 / #9 / #10 均已
合并；收口时点 `main` = `11ef4b2`）。**

- **目标（已达成）：** 把已验证的 faithful frontend 路径从开发期的同进程布局迁移到 D072 产品拓扑——thin TSF DLL +
  out-of-process Fusion Host——并接通真实 TSF 生命周期，使 D072 的隔离、重连与 fencing 保证在真实产品路径上成立。
  Stage 4 不重做 Stage 1–3C 的交付，只把它们搬到产品拓扑上并保持行为不变。
- **已完成、因此不属于 Stage 4：** portable Core / RG-0（PR #1）；`fusion::Host` 边界（PR #2）；真实 TSF 注册、
  key sink、EditSession materialization、drain-before-PASS、compartment hard policy 与 composition self-end/外部终止
  处理（PR #4）；Host 边界内的真实 Fcitx5 backend（PR #5）；faithful key routing、有序回调承载与 Forward 恰好一次
  （PR #7）。Stage 4 不重做这些，只把它们搬到产品拓扑上并保持行为不变。
- **范围与状态（依据 fusion-windows `main` `11ef4b2`，三个阶段均已合并并各自经独立评审）：**
  1. **4A — Out-of-process Host + bounded IPC（PR #8，已合并）。** TSF DLL 为 client-only，不链接
     Fcitx / chinese-addons / LibIME（PR #8 `dumpbin /dependents` 核查，收口时对合并后的 4C `FUSION_WITH_FCITX5=ON`
     构建重新核查），经 per-user/per-session Named Pipe 连接外部 `FusionHost.exe`（§1–§3；§14 baseline：
     one Host per user × session、Named Pipe、Medium IL）。IPC 协议与标识符是工程常量，**不冻结**拓扑或 wire 格式；
     Host 启动/生命周期属 P0-D，DLL 不自动拉起 Host。该通用边界承载后续 projection 所需的既有 frontend 数据
     （含已有 CandidateSnapshot），不引入新的 candidate 语义或 candidate 专用传输子系统。write-certainty 边界：
     definitely NotWritten 可依法保持 caller-owned / Forward；Uncertain write 或 post-write response loss 为 Reject；
     不盲目 replay/retry；不引入通用合成键注入。
  2. **4B — Host 失效 / 重连 / semantic readiness / incarnation fencing（PR #9，已合并；§13）。** §13 阶梯在
     真实产品路径上实现：client 传输状态 `Disconnected → Connected → Established`，semantic readiness 是其上独立的
     **per-context** 条件（pipe connected ≠ semantic ready）；重连 bounded、幂等、由既有 key/context 路径驱动，
     无后台线程与 busy loop；Host incarnation 与 connection generation 提供可测的 fencing 身份；旧 incarnation 的
     STATE/结果不能改写新 incarnation 或跨 context materialize；未决/不确定 EFFECT 责任显式保留，既不被转成普通
     PASS，也不被盲目 replay；Host 崩溃不得使应用崩溃（ownership 前 FailOpen）。Stage 4A write-certainty 语义保持。
  3. **4C — TSF 生命周期接线与 context reconciliation（PR #10，已合并；§9）。** 生命周期义务表按真实观察接通：
     只有 teardown 事件（`Deactivate` / `OnPopContext` / `OnUninitDocumentMgr`）结束 Fusion context，并在 fencing 前
     审计仍未决的责任；外部 `OnCompositionTerminated` 对账 projection/pending 而保持 context identity 与 Host
     binding；focus（两个变体）、`OnPushContext`、`OnInitDocumentMgr` 记录但不改变任何东西（§9：focus 不是 semantic
     incarnation）。被 pop 的栈成员处置已由 **D083** 接受并定界（§9）；其余 discontinuity 子边界仍未取证。
- **Gate 评估（Stage 4 PASS 实际依据的证据）：** 原 gate 措辞要求"在 Stage 3C 已验证的应用集合（Notepad、Firefox、
  Edge 地址栏与文本框）上，Stage 3C smoke 用例经 out-of-process 路径保持通过"。已实际取得的真实应用产品路径证据是
  **Notepad**（本仓库既定的最小 owner-visible 应用）：`nihao` + Space → 恰为 `你好` 经 out-of-process 路径在 Host
  kill / restart 前后均通过（4B run 3 的目标实例自身 per-key 日志；4C 六步运行在重启前后各取得一次提交），
  且不经切换输入法即完成"检测 → 重连到新 incarnation（connection generation 递增）→ semantic readiness 重建 →
  新输入再次工作"；另有真实 `FusionHost.exe` kill/restart 与 silent-death 的确定性测试同覆盖同一边界。其余 gate
  条款均已满足：TSF DLL 不再链接 Fcitx / chinese-addons / LibIME；输入过程中强制终止并重启 Host 时应用不崩溃；
  重连与 incarnation fencing 生效；无 silent owned-key loss、重复不可逆 EFFECT、跨 context EFFECT、stale state 复活、
  不确定 mutation 后的盲目 replay/retry 或 PASS 越过未决 EFFECT（RG-0 不变量由确定性真实进程测试与产品路径
  运行共同约束）；仍属 P0-A/P0-B 的未决恢复责任被显式记账并可观察（`CONTEXT_TEARDOWN`、`COMPOSITION_TERMINATED`、
  `LINK` 诊断）；focus/context 切换与 composition 外部终止产生显式终态；现有 Core / Stage 测试保持通过
  （Mock-only 54/54；real-Fcitx 58/60，两项失败在 base 同一环境复现，为 pinned-runtime 既有问题，非本阶段回归）。
- **本收口不主张的内容（范围澄清，而非证据主张）：** 上述 gate 条款的**应用广度**未重新普查——Firefox 仅有一次附带
  观察（4B run 3 中一个 Firefox TIP 实例随其他实例连回新 incarnation），不构成该应用上的 smoke 用例；Edge/Chromium
  未作 out-of-process 复验，仍为 §7.6 / §18 **DEFERRED UNKNOWN**。该定界与本节既有的"更广的应用兼容性 → Stage 6 或
  §18 DEFERRED"条款一致：**多应用兼容性不是 Stage 4 机制收口的前提**。因此 Stage 4 记为 COMPLETE / VERIFIED 的范围是
  其已实现并已验证的产品拓扑、重连/fencing 与生命周期边界；多应用 out-of-process 再验证记为 **VALIDATION DEBT**
  （可推迟的验证债，不是机制失效），归 Stage 6 的应用广度门。若日后真实应用暴露与已接受结论矛盾的观察，属 §9.7
  意义上的实质性新事实，按其规则重开对应事项；本条不预先否认当前结论。
- **仍 OPEN、不因 Stage 4 关闭：** P0-A 最终 recovery key policy、P0-B recovery representation、P0-D 除 §14 baseline
  之外的场景（High IL、AppContainer、多 session、RDP 等）、§9 未取证的 lifecycle 子边界、final wire protocol、
  permanent identifiers、timing/threshold 数值、effect-capable key 分类。Host 崩溃后旧 incarnation 状态与未决 EFFECT
  的用户可见表示，被刻意留白（仅要求可检测、可审计）。
- **不属于 Stage 4（历史定界）：** candidate projection / renderer 与其进程/位置选择（§16）、composition 显示与 UI
  打磨 → Stage 5；安装器、持久注册/提权路径、签名、soak、延迟/阈值测量与冻结、更广的应用兼容性 → Stage 6 或
  §18 DEFERRED。

**Stage 5 — Composition / candidate UI（含首个 candidate renderer，§16）：VERIFIED / COMPLETE（在下文 closure
gate 记录的范围内；fusion-windows PR #11 与 PR #12 均已合并）。** 下列目标、必须保持与边界是 Stage 5 的契约，
收口后继续约束后续工作。

- **目标（已达成）：** 让真实 Fcitx composition / 候选 UI 经既有 D072 通道进入 Windows 产品路径，并以**非键** semantic
  stimulus 把候选动作回送引擎。
- **必须保持：** Stage 3C 的有序回调与独立结算；Stage 4A write-certainty；Stage 4B incarnation /
  connection-generation fencing 与 per-context semantic readiness；Stage 4C 生命周期义务表与 §9 被 pop 栈成员的
  处置（D083）。
- **边界：** 不引入 candidate 专用传输子系统（扩展既有 `BackendOperation` / `BackendStimulus` 数据面）；不在 TSF
  DLL 内建立第二候选语义模型或新的语义 decoder；候选 select / page 等动作不得以合成键 replay 模拟（§7.4、D082）；
  候选快照属 STATE（可 latest-wins），但候选动作引发的 EFFECT 仍走既有顺序结算与 consumed STATE → EFFECT
  watermark；§16 renderer 的进程/位置仍 UNFROZEN，首个变更只选取所需的最小实现接缝。
- **契约输入：** `research/windows-stage5-candidate-contract-inputs.md` 为 research / evidence，**不是** Accepted
  Decision，不冻结 wire schema，本身不构成实现授权；其中的 Fcitx 契约事实须经一手源码核对后方可依赖。

**PR #11 — 首个 Stage 5 candidate projection/action 接缝（已合并；canonical fusion-windows `main` `3e7fc35`，
其 tree 与经评审的 PR head `d8bc8c0` 一致）。**

- **PR #11 是 Stage 5 的第一个接缝，不是 Stage 5 closure。** 其已实现、因此不得再被描述为未解决的能力：
  有界的 candidate page 投影（STATE，index 与 `CandidateList::candidate(i)` 对齐，含 `layoutHint()` 的
  `CandidateLayout` 投影并被 renderer 消费、pager 真值与 `truncated` 边界旗标，以及截断后 cursor 永不指向
  保留前缀之外的规范化；label/text/comment 显示字段在投影处经引擎自身 `Instance::outputFilter` 过滤，
  与成熟 classic-ui 显示路径同一调用形状，Windows 不建立第二过滤权威）；显式**非键**候选选择
  （`SelectCandidate`，复用真实按键 dispatch 的准入、有序 Core 记录与结算）；显式翻页
  （`PageNext` / `PagePrev`）；针对 context / Host incarnation / connection generation 的 stale-action fencing
  （`PageFence` 失配即 fail-open 拒绝，不发 dispatch）；首个最小 candidate presentation 接缝
  （non-activating owned popup，经 `ITfContext::GetActiveView` 锚定 active view，焦点丢失仅隐藏
  presentation，不触碰语义 incarnation）。
- **snapshot 排序/围栏的实现处置（研究契约建议的对账）：** 候选快照与候选动作的因果/状态身份由
  **复用既有单调 stimulus identity**（`PageFence` 记录产生该 page 的 dispatch 的 `stimulus_id`，连同
  binding / backend session / Host incarnation / connection generation）满足，而非新增独立的 candidate
  revision/generation 计数器；接缝未引入第二 candidate 账本。这是对既有建议的收口记账，不构成新的
  Accepted Decision。
- **剩余能力逐项处置（closure accounting，2026-10-17，依据 canonical `main` `3e7fc35` 与一手 fcitx5
  `a7f2528` / chinese-addons `61474bd` 源码）。** 背景事实：Host 静态加载 pinyin / punctuation / pinyinhelper /
  quickphrase / spell / clipboard，frontend 只声明 `Preedit`；Windows 构建下 pinyin 的编译期默认为
  `PreeditMode=ComposingPinyin`、`PreeditCursorPositionAtBeginning=true`——client preedit 为
  `[HighLight|Underline：引擎光标之前][Underline：其后]`、cursor=0，HighLight 边界是引擎光标的唯一显示。
  分类是对当前事实的处置记账，不是新的 Accepted Decision；分类不冻结 §16、wire、标识符或数值。

  | 能力 | 处置 | 阻塞收口 | 理由 / owner · trigger · 依赖 |
  |---|---|---|---|
  | 1. faithful preedit projection = 显示的 preedit 文本在正确的 Fcitx frontend 边界经 `Instance::outputFilter` + 保留/投影所需 `ui::Text` 分段与 format flags + 引擎 byte cursor + 合法换算到 TSF UTF-16 caret 与相应 display attribute | **IMPLEMENT** | **是** | 现状把 client preedit 扁平为 `clientPreedit().toString()`：未过 `outputFilter`（dbusfrontend 对 client preedit 调用之）、分段/flags 丢失、caret 恒置 composition 末尾；pinyin 在默认键（Left/Right/Home/End）上移动/编辑引擎光标、Left 在已选边界撤销部分选择，Windows 无从显示——偏离 D082 composition 语义，TSF 无限制（composition 内可置 selection，样式经 display attribute）。四个组成部分缺一即不满足本项；仅完成 cursor/format 而缺 `outputFilter` 不构成关闭 |
  | 2. aux/panel projection = `auxUp`/`auxDown` 及本项实际投影的其他 panel 文本（server-side `preedit()` 行）经相应 `Instance::outputFilter` + `auxUp`/`auxDown` 投影 + 隐藏条件 = 整个 `InputPanel::empty()` | **IMPLEMENT** | **是** | pinyin 笔画筛选（默认键 `` ` ``）所输入的笔画**只**显示在 auxUp，QuickPhrase 模式提示亦在 auxUp；现状不投影 aux，且候选页为空即隐藏——引擎吞键而用户无可见输入，隐藏条件窄于 Fcitx 契约。classic-ui 与 dbusfrontend 均承载，且两者都先经 `outputFilter` 再显示 aux。三个组成部分缺一即不满足本项 |
  | 3. 候选 comment 呈现 | **IMPLEMENT** | **是** | comment 已投影（过 `outputFilter`）但 renderer 只画 label+text；pinyin 用 comment 区分同字异读候选（`(full pinyin)`；同字候选可对应不同读音与选择长度）、`(Half)` 标点与笔画候选读音 |
  | label / placeholder；候选 label/text/comment 经 `outputFilter` 的传输 | 已解决（PR #11） | 否 | — |
  | `customLabel` | DEFER WITH REASON | 否 | 固定源码中无 producer。trigger：Host 加载的引擎调用 `setCustomLabel`；做法：非空时优先（同 classic/dbus） |
  | `spaceBetweenComment` | DEFER WITH REASON | 否 | 默认 `true`，唯一 producer 为 table IM（未加载）。trigger：Host 加载 table IM 或其他置 `false` 的引擎；依赖 comment 呈现 |
  | `overlayMessage` | DEFER WITH REASON | 否 | 唯一 producer 为切换输入法时的 IM 信息；Host 仅一个 IM，修饰键切换不到达。trigger：Host 注册第二个 IM / IM 切换可达；依赖 aux 行 |
  | bulk candidate access / windowing | DEFER WITH REASON | 否 | 分页契约（`hasPrev/hasNext` + next/prev，已实现）是桌面成熟前端路径；bulk 仅服务展开式网格（Android）。owner：§16 renderer 决定；trigger：采用展开/滚动候选 UI |
  | `ActionableCandidateList` / `TabbedCandidateList` | DEFER WITH REASON | 否 | 可选能力接口；pinyin 的 action = 忘词/置顶/删除自定义短语（用户词库管理），tab = 音节/笔画筛选；classic-ui 与 dbusfrontend 均不消费。owner：§16 renderer 决定 + 所有者产品请求；依赖：在同一 PageFence/stimulus identity 下扩展非键 stimulus，禁止合成键 |
  | preedit 点击 / `InvokeActionEvent` | DEFER WITH REASON | 否 | 依赖上面的 preedit 光标投影与 composition range 上的 TSF mouse sink。trigger：preedit 投影合并后的所有者请求 |
  | `cursorRect` / scale-factor | DEFER WITH REASON | 否 | 当前 presentation 在 TSF 侧经 `GetActiveView`/`GetTextExt` 锚定，Host 内无 Fcitx UI 消费者。owner：§16；trigger：选定 Host 侧 renderer 或 Host 模块消费 `cursorRect` |
  | 键盘-候选 UX（无修饰键） | 已解决（Stage 3C 路由 + PR #11 投影） | 否 | 数字/Space 选择、`-`/`=`/PageUp/PageDown 翻页、方向/Home/End、Tab、Esc、Backspace 均以 keysym 到达 Fcitx，高亮/页状态以 STATE 回投 |
  | 修饰键组合（Ctrl/Alt/Win）引擎绑定 | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 现为平台策略 Pass，如 Ctrl+7 忘词、Ctrl+←/→ 永不到达 Fcitx。问题（D082 key admission）：哪些组合是真实 TSF / §10 preserved-key 约束，哪些只是被预先扣留；证据：TSF preserved-key/系统快捷键 census 对照 pinyin 默认绑定 |
  | `commitStringWithCursor` / `InputContextV2` | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 仅非默认 `TypePairedPunctuationsTogether` 可达；无该 capability 时 pinyin 退回 commit + 合成 `forwardKey(Left)`，Windows 视为非原始 Forward 而 fail-closed（CancelledRecovery，无 replay）。问题：是否升 V2 并把 commit + 文档内 caret 作为一个 EFFECT；证据：所有者是否在 Windows 支持成对标点，及目标应用同一 EditSession 内插入后置 caret 的可靠性 |
  | surrounding text | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 维持 fence 为合法默认。问题：是否以 TSF 只读 readback + 有序 EFFECT 的 `deleteSurroundingText` 声明 `SurroundingText`；证据：Host 引擎依赖面与目标应用的合法 readback/delete（§8） |
  | `StatusArea` | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 模式/状态 UI，非 composition/候选语义。问题：Fcitx status action 与 active 状态如何投影到 TSF（language bar / Host UI / §10 conversion-mode compartment）；依赖修饰键路由决定 |
  | `ClientUnfocusCommit`（研究 §7.3，原列表遗漏） | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 与 §9 focus/termination 语义绑定，§9 未取证子边界仍 OPEN；取证前不声明 |
  | 客户端字段类型 capability（Password/Sensitive 等；原列表遗漏） | NEEDS SEPARATE EVIDENCE/DECISION | 否 | 属 §10 hard policy（input scope/private/password）；pinyin 学习以 `PasswordOrSensitive` 为门，故含隐私面。Stage 6 WP2（fusion-windows PR #14）已实现**有界的 InputScope admission 策略**：以同步读取当前 selection 的 InputScope，Password/Private/PIN 类 scope 在 backend admission 前阻断；metadata 不可得时记为 Unknown，按普通字段处理，不做启发式判定（已知限制）。最终候选浏览器验证（2026-10-18）：Edge 密码框无 composition、无 backend admission，但拦截来自既有的 keyboard-disabled/empty-context compartment，InputScope 读为 Unknown；Firefox（新进程，普通框对照有效）同样在接管前被该 gate 拦截（拦截发生在 T 路径）；test-key 资格依赖就绪状态会阻止 K 路径恢复，此缺陷已确认（复现，修复后恢复），并由 fusion-windows PR #15 修复（已合并，plan R24）。此前一次 Firefox 运行不确定的具体触发（先前的 Host incarnation 已停止、初次 bind 失败）属推断：bind 结果未被直接记录，但与该缺陷一致。Notepad 与两种浏览器均观察到 Unknown，原因未诊断（应用不提供 metadata、探测/读取失败均为假设）。Sensitive 分类路径尚未在浏览器中得到证明；U5 的最低政策已由所有者于 2026-10-18 决定为 (a)（密码/私有 scope 下 fail open、不接管），该决定接受此已知限制，不表示接受忠实的 capability 路径。**忠实的 Password/Sensitive capability（把字段类型传给 Fcitx）仍延后**，本行处置不变。问题：长期是 FailOpen 还是传 capability 给 Fcitx |
  | 精确 caret 锚定（popup 锚定到 TSF selection/caret 本身） | DEFER WITH REASON | 否 | `AnchorProbeSession` 以 composition range 的 `GetTextExt` 右/下边界为锚点，而非精确的 TSF selection/caret；当前 Stage 5 门的 B 项（锚定到 active text context）在 Notepad 成立。重要但非阻塞的 UI 限制。owner：Windows composition/candidate UI 跟进；trigger：Stage 6 定位/应用兼容性工作、在宣称精确 caret 定位兼容性之前，若当前门的真实失败证明有必要则提前；建议（候选方案，需相称验证）：测量折叠的 caret/selection range |
  | 悬停高亮 | DEFER WITH REASON | 否 | 纯外观（classic-ui 本地 hover，不改引擎 cursor）；随 §16 renderer 打磨 |
  | UI suspend/resume、OSK `VirtualKeyboardUserInterface`、`FCITX_NO_PREEDIT_APPS` | NOT APPLICABLE ON WINDOWS | 否 | Host 内单一 frontend；Windows 触摸键盘是 OS 表面；按程序名关闭 preedit 是 X11/Wayland 变通，TSF composition 是逐应用契约 |

  DEFER 项的 owner 默认是 Stage 5 收口之后的 Windows composition/candidate UI 增量，按所列 trigger 启动；
  NEEDS SEPARATE EVIDENCE/DECISION 项各归其所列权威（§9 / §10 / D082 key admission / §16），不在 Stage 5 内
  隐式决定。逐项一手源码依据见 fusion-controller PR（本次 closure accounting）描述。
- **Stage 5 closure gate（唯一一条）：已满足（2026-10-17）。** (1) 剩余能力逐项分类——已满足（上表）；(2) PR #11
  接缝合并进 canonical fusion-windows `main`——已满足（`3e7fc35`）；(3) 上表三项 **IMPLEMENT** blocker 按项目常规
  流程收口——已满足：fusion-windows PR #12 已合并（merge `0f724af1a912daaf83bad65f9deb92b779352f54`），其 tree
  `ee2f5536e684406a3b9db174277dbe7a681221cb` 与经独立定向复审并通过 exact-HEAD Notepad A–E smoke 的 PR HEAD
  `670dfc03cd6992681ed800570413ddd9a789582f` 完全一致，即被验证的产品 tree 已在 canonical `main` 上；独立评审门与
  exact-HEAD smoke 门均已满足。smoke 的 A–E：A 输入 `nihao` 出现可见候选 popup；B popup 锚定到 active TSF text
  context；C 点击某个显示位解析到正确的显示 index；D 点击经显式非键候选动作到达 Fcitx；E 选择/提交在应用中正确结算。
  - **收口证据（`670dfc0`）：** 新 Notepad 进程实际加载的 `FusionTsf.dll` 与该 HEAD 的构建哈希一致；A 可见 popup；
    B 锚定到 active text context（当前 Stage 5 范围）；C 点击解析为显示 index 2；D `CANDIDATE_ACTION action=0
    index=2 route=Effect`，无合成键；E `EFFECT_APPLIED`，文档读回为该候选文本。本地验证：Mock-only 64/64；
    real-Fcitx 70/72——`stage3c_faithful_frontend`、`fcitx_msvc_empty_range_guards` 两项为既有 `xstring(54)` 中止
    （已记录的基线，与 base 相同），并非全部测试通过。
  - **本收口主张的范围：** 三项投影 blocker（faithful preedit、aux/panel + 整个 InputPanel 隐藏、候选 comment）
    与候选 popup 指针送达/选择，在 Notepad（既定最小 owner-visible 应用）真实产品路径上成立。**不主张：** 多应用
    兼容性（仍归 Stage 6 应用广度门，Chromium/Edge 仍为 §7.6/§18 DEFERRED UNKNOWN）；精确 caret 锚定（上表 DEFER
    项）；上表其余 DEFER / NEEDS SEPARATE EVIDENCE/DECISION 能力；P0-A / P0-B / P0-D 的任何结论；§16 renderer 进程/
    位置或 wire schema 的冻结；上述两项基线中止的修复。
  - **历史（PR #12 期间，保留为证据）：** 首个集成候选 `36912ff` 的 Notepad A–E smoke 为 A PASS、B PASS、**C FAIL**
    （对可见、已锚定 popup 的点击未产生 `CANDIDATE_HIT`），D/E 因此未被到达（C 的下游后果，不是独立缺陷结论）。
    最小消息顺序诊断（临时 instrumentation，未提交）显示物理点击与注入点击行为一致：`WM_MOUSEACTIVATE` 交给
    `DefWindowProc` 后 popup 被点击激活（WA_CLICKACTIVE），夺走应用焦点，应用终止 composition、popup 被隐藏，选择用
    的 button-up 未到达。修复 `6d54592`：popup 以 `MA_NOACTIVATE` / `PA_NOACTIVATE` 拒绝激活（Win32 实现细节，不是
    Accepted Decision）；修复后的诊断运行基于 dirty 诊断构建，仅为诊断证据，收口证据为上面的 `670dfc0` exact-HEAD
    smoke。`670dfc0` 另把四个 `stage5_candidates` 模式（utf16/preedit/panel/comment）注册进 CTest。
- **pinned-runtime 兼容性发现（Stage 6 WP1 已核验，2026-10-17）：** 历史：PR #12 实现工作曾报告 pinyin 笔画筛选模式下
  输入笔画键会在固定的 MSVC Debug runtime 中触发 string_view 迭代器断言，并声称 base 可复现；当时未经独立核验。
  WP1 观察：(1) 旧源码检出与已安装头文件均对应 `fcitx5-msvc.patch` 的早期状态 `d476f29`，缺少两处空区间 guard
  （`lengthValidated` 与 `UTF8StringViewIter::update`）；两项基线中止（`stage3c_faithful_frontend`、
  `fcitx_msvc_empty_range_guards`）在旧 runtime 上出现、在重建 runtime 上消失，归因于缺失的 guard；旧二进制的完整
  构建来源仍为 UNKNOWN，不视为已证明。(2) 产品路径上
  旧 Debug runtime 对 `,`、`nihao,` 与 QuickPhrase 触发 `www.` 会使 Host 停在 Debug CRT 断言对话框（应用不崩溃，
  TIP 按 post-write failure 作 Reject）。(3) 笔画键断言经独立探针在旧 Debug runtime 上复现。以上三项在以当前补丁、自固定版本全新检出重建的 Debug / Release runtime（`runtime-wp1-20261017`）上均
  不再出现；笔画断言与 guard B 位置一致，guard B 是合理归因，但未独立隔离。WP1 的 Notepad A–E 使用既有 Stage 5 已注册的
  `FusionTsf.dll` 加重建的 Host/runtime，是该组合的兼容性证据，不是整套新构建二进制的 smoke。未修改 `cmake/patches` 或固定版本；fusion-windows
  PR #13 仅修正一个早于 Stage 5 的测试判定。证据与处置见 `docs/windows-stage6-plan.md` R1–R3。

**下一阶段：Stage 6 — Packaging / release / soak（PLAN PROPOSED，未接受；WP1 已合并收口；WP2、WP3 已在其范围内收口（所有者 2026-10-18 决定 U3 (a)、U5 (a)、U1）；后续工作包未授权）。** 唯一的 Stage 6
规划位置是 `docs/windows-stage6-plan.md`：提议的目标场景（Pinyin 与 Shuangpin、Fcitx 学习的持久化为目标要求）、过渡期
测试产物的限制与排除项、发现登记表、P0 子问题与故障恢复取证、待所有者决定的事项、有序工作包与门；每个工作包须
另有明确的执行指令才可开始。该计划不是 Accepted Decision，不改变本文的 OPEN/UNFROZEN 项，也不复制上表；其输入包括
多应用 out-of-process 兼容性验证债（§25 Stage 4 收口、§18）、上表以 Stage 6 为 trigger 的 DEFER 项（如精确 caret
锚定）、上面的 pinned-runtime 发现核验（已由 WP1 完成），以及安装/注册/签名/soak/阈值测量（§21、§25 Stage 4"不属于"条款）。
WP1 已合并并在其范围内收口；WP2 已在其范围内收口（所有者 2026-10-18 决定 U3 (a)、U5 (a)），最终候选 Notepad A–E 与 Edge 密码框检查通过，Firefox 密码框检查也已通过（最低保护结果），技术检查已完成，U5 的已知限制（InputScope 读为 Unknown、Sensitive 分类未证明）随该决定一并接受（`docs/windows-stage6-wp2.md`；`docs/windows-stage6-plan.md` §7.1）。WP3 亦已在其范围内收口：选择只经 Fcitx 既有配置文件，存储落在 Fcitx 标准每用户路径，学习与选项跨重启持久（B3a–B3g、Notepad A–E 通过，候选 `a720319`，运行于新 runtime `runtime-wp3-20261018`；WP2 结论仍取自已记录的 `runtime-wp1-20261017`，未在新 runtime 上复跑），已知限制为无热键切换（R19）、强制终止丢失未保存改动、登出/关机保存取决于启动方式（R26）（`docs/windows-stage6-wp3.md`）；后续工作包仍须各自的明确授权，U2、U4、U6、U7 仍待决定。

**暂不应做：** freeze final wire protocol；freeze permanent identifiers；freeze recovery/timing thresholds；freeze
未决 Host topology；无证据即冻结 §16 candidate renderer 的进程/位置；借其他阶段的工作偷跑 P0-A / P0-B 的最终
recovery policy 或 recovery representation；未经 Stage 6 自身的任务/契约就开始 Stage 6 实现；无 D082
证据而偏离或替换 Fcitx 语义；在 TSF DLL 建立第二候选语义模型或 candidate 专用传输；broad product UI；
为 P0-C 继续 Chromium 实验。

## 26. 文档 authority 与仓库职责

| 仓库 | 职责 |
|---|---|
| `fusion-controller` | normative architecture（本文）/ requirements / decisions（D072）/ roadmap |
| `researching-notes` | research narrative、证据解读、architecture history（`windows-ime/fcitx5-windows-architecture-evolution.md`，Stage L 记录本次迁移）、supporting reviews |
| `fcitx5-windows-tsf-census` | 可复现的低层 TSF 实验实现、raw traces、HRESULT census、实验脚本/构建输出 |
| `fusion-windows` | Windows 产品实现 |

读取顺序：本文与 D072（**现在是什么**）→ researching-notes evolution（**为什么变成这样**）→ researching-notes initial
architecture 与已 SUPERSEDED 的 2026-10-07 snapshot（**历史状态**）→ census 仓库（**证据从哪里来**）。历史文档与本文在
“当前状态”上冲突时，以本文为准；本文的 OPEN/UNFROZEN 项不得被历史文档措辞擅自冻结。

**“B” 消歧（仅本文所述三处，互不相同）：**

- **Recommendation B（已 SUPERSEDED by A′）**：census `docs/p0c-automated-evidence.md` 与
  `docs/real-tsf-runtime-evidence.md` 中“keep bounded-sync EFFECT and fully-async EFFECT equally open”的历史建议。
  不得据此把项目回退为“两者并列开放”。
- **Model B**：census `docs/protocol-findings.md` 中的抽象协议模型（primary live-context callback 串行化 later PASS），
  是 A′ 同步主路径所由发展的模型，不是被 supersede 的建议。
- **Option B（evolution Stage H）**：关于 upstream fcitx5-windows frontend 复用策略的选项，与 P0-C EFFECT 调度无关。

**外部评审：** researching-notes `windows-ime/reviews/claude-2026-10-07.md` 的 §2（effect-capable key 同步 materialize）
与随后经证据核查的 A′ 方向一致、对其起支持作用；它仅是 supporting review input。A′ 的接受来自项目证据与项目决定（D072），
外部 AI 评审不是 architecture authority，也不是强制 gate 或常设流程。
