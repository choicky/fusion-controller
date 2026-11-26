# Requirements

## 1. 项目目标

开发/研究一套以 **Android 为第一目标平台**的中文输入方案。优先复用 Fcitx5 生态已有能力，在源码研究和最小 PoC 证明现有接口不足前，不重新实现成熟基础设施。

核心能力：

- 高质量 Pinyin / Shuangpin；
- 按需 Auxiliary Filter（当前重点为 MoQi）；
- 高质量中文语音输入；
- 可配置 ASR 与可选 LLM 后处理；
- 数据流透明、可审计、可配置。

## 2. 中文主输入

- Pinyin / Shuangpin 为主输入方式。
- 优先使用 `fcitx5-chinese-addons` + LibIME。
- 继续利用 LibIME 的候选、语言模型、词典、用户学习和 partial selection。
- Rime / rime-frost 是成熟参考与备选，不是硬依赖。
- 除非 PoC 证明现有能力不足，不重新实现拼音解码器，不修改 LibIME 核心。

### 2.1 Mixed Input architecture boundary (D074 / D075)

> **澄清注记（2026-10-11；澄清既有措辞，不改变规范强度、不改变任何裁定）：** 本条 unified
> bounded mixed search 的规范范围是混输候选 **generation**（Layer 1）。最终组装（Layer 2）中
> PinyinEngine 经既有排序/组装机制（customCandidateMap / CandidateOrder / inplace_merge）把混输
> 候选与 LibIME classical 候选及其他既有来源做**候选级 composition**。“非 late merge”禁止的是
> **两个独立整句 decoder 各自产出最终句后在末端朴素合并**的架构，不得被读为禁止或否定上述
> 候选级 composition。另：classical 路径对大写/raw literal（如用户键入的 “Apple”）的保留，不使
> 这些候选成为 English Core 拥有的词汇假设（无 English sourcing / lexicon ownership /
> learning ownership）。以上读法经产品基线源码复核确认（fcitx5-chinese-addons @ 644205f：
> im/pinyin/pinyin.cpp:816-993、im/pinyin/pinyincandidate.h/.cpp；LibIME 只读核实）。

中英混输保持 Architecture A 上位结构：Pinyin/Shuangpin Chinese hypotheses（LibIME-derived，
含 Chinese LM、UserDict/history 能力）与 English Core hypotheses 进入**同一个 bounded mixed
hypothesis graph / bounded search**，产出统一候选表示并排序后进入 CandidateList；
Auxiliary Filter 继续位于主候选生成之后（downstream）。

“中文/英文独立”的规范含义是 **candidate sourcing、lexicon ownership、learning ownership、
resource responsibility 独立**；它不意味着两个完整 decoder 必须分别完成整句解码，也不意味着
Chinese/English 先各自选出最终结果再 late merge。不得为了 mixed input 把 English resources
无原则混入 Chinese dictionary。

每次 input update 使用一个 bounded mixed search；LM state 是 search path state 的组成部分，
Chinese transition 按该 path 当前 LM state 做 context-aware scoring（以 pinned LibIME
`LanguageModel::score` 语义为基准，只允许源码核实/已验证的最小调整）；搜索必须 bounded 并保留
有限的多路径 diversity；不得在 cross-language context 有机会发挥作用前，把 Chinese span 不可逆
固定成 Top-1；A′ per-gap ChineseGapSolver 及其 Chinese (start,end) Top-1 contract 不是产品架构
（historical/reference，D074/D075）。不得直接比较/相加未经校准的 heterogeneous Chinese/English
raw scores。当前产品架构 baseline 为 M2+/r4（D075 Accepted；A′ 与 M2+ 的架构竞争已 CLOSED，
A′ 不再是 competing accepted product architecture）。

transitionAcrossEnglish 的规范含义：架构要求存在**一个集中式的 English transition 策略**；它
**不**永久要求 preserve——“LM state 是 search-path state”不蕴含该状态必须跨越 English
transition 被保留。当前接受且经验证的 r4 产品 baseline 使用 **reset**（D075 §3：validated
implementation policy，非永久架构不变量）。preserve 或其他策略可作为后续研究（deferred /
unverified），不是当前 requirement，也不是 productization blocker。

Pinyin/Shuangpin 都是一等支持；raw↔output alignment、partial selection、composition
preservation 语义必须保持。LibIME 原则上不修改；只有实现过程中真实源码事实证明所需能力无法
经现有接口获得时，才按 Hard Stop 规则重新评估。M3-B Graph-aware Parallel Mixed Recall
（proposed D073，未接受）保持 deferred research candidate，不是当前 product target。

HR1/M2+ 各轮实验参数（beam B、English LM state 测试策略含 reset 默认、计数/降幅判定线、
语料规模、评测矩阵与 ASan 子集等）属于 experimental decision rule，**不是永久产品需求**，
只在 ROADMAP 实验记录中管理；reset 本身也不因此升级为永久 Requirement。

未来独立分发的英文词库资源（D078 §4，forward requirement）：引入可下载/可更新的 English
Lexicon 时，该资源必须携带显式的 runtime/scoring compatibility generation/version 标记
（字段名与编码属实现细节），应用只启用与其所支持的 scoring/runtime contract 兼容的资源
版本；改变 ranking 相关数据的资源 artifact 分发前必须通过相应的 mixed-input 回归门；不得
绕过 compatibility gate 改变 ranking 语义（英文词库的 prior/tier 分布与已校准 mixed-search
scoring 直接交互）。本条不是当前实现 English Lexicon 下载/更新能力的授权。

## 3. Auxiliary Filter

Auxiliary code 是候选过滤手段，不是主输入编码。

统一模型：

```text
Pinyin / Shuangpin
       ↓
LibIME Candidates
       ↓
Auxiliary Filter Trigger
       ↓
Configured Auxiliary Filter
       ↓
Disabled / Stroke / MoQi
```

默认 Trigger 为反引号 `` ` ``。Trigger 仅表示“进入当前配置的 Auxiliary Filter”，不得硬绑定 Stroke 或 MoQi。

当前不得为未来 Filter 建立复杂 Plugin Framework。

### 3.1 复用 Stroke Filter

应复用并最小泛化 `fcitx5-chinese-addons` 已有 Stroke Filter 基础设施，包括：

- `FilterByStroke`
- `handleStrokeFilter()`
- `PinyinTabbedCandidateList`
- filter mode / buffer
- `CommonCandidateList::setFilter()`
- Backspace / Escape
- candidate selection
- tab actions
- composition 协作

不得在可复用该基础设施时继续维护独立平行的 Stroke/MoQi trigger 与 mode 状态机。

具体算法保持分离：

```text
Auxiliary Filter
├── Stroke → reverseLookupStroke() → filterByStroke()
└── MoQi   → reverseLookupMoQi()   → filterByMoQi()
```

### 3.2 MoQi 交互

采用早期墨奇的按需逐字/词辅助筛选模型：

1. 正常 Pinyin/Shuangpin 输入；
2. 出现歧义时进入 Auxiliary Filter；
3. 输入 MoQi code；
4. 候选减少；
5. partial selection；
6. composition 保留；
7. 继续输入；
8. 后续可再次使用 Auxiliary Filter。

MoQi 不得天然触发整句 commit。Backspace 应撤销辅码/过滤状态，Escape 应退出 Auxiliary Filter。

### 3.3 MoQi target semantics

MoQi V1 过滤目标是 **current selection frontier 后的目标字符**：

```text
selected prefix | unselected composition
                ^
          selection frontier
```

不得照搬 Stroke 当前“候选 phrase 中任意字符匹配即可保留”的语义。

### 3.4 Partial selection

优先复用：

- `PinyinContext::selectedLength()`
- `candidatesToCursor()`
- `selectCandidatesToCursor()`
- `selectCustom()`
- `cancel()`
- `ChooseCharFromPhrase`

过滤并选择后应保留 selected prefix，继续解码剩余 Pinyin/Shuangpin，并允许再次进入 Auxiliary Filter。

## 4. MoQi 码表

当前固定来源：

- repository: `gaboolic/moqima-tables`
- commit: `6d8ba8f1c57466f358e682baefe11bbd0fe389ab`
- table: `moqima_gb18030.txt`

V1 runtime 主要需要“汉字 → MoQi code”。测试值必须来自固定码表，不得猜测或凭记忆填写。许可证和再分发要求必须保留。

## 5. Android Auxiliary Filter 配置

优先使用 Fcitx generic configuration：

```text
Auxiliary Filter:
- Disabled
- Stroke
- MoQi
```

`fcitx5-android` 已有 ConfigEnum/ConfigKey 通用 UI，应优先复用；除非实际验证不足，不增加 MoQi-specific Android settings UI 或修改 candidate frontend protocol。

### 5.1 成对标点目标（D051）

成对标点复用现有 Fcitx5 Chinese Addons 的 punctuation setting/mechanism。启用成对标点行为时，当前用户可见目标集合严格为以下 19 组；不得静默扩展：

1. （）
2. “”
3. ‘’
4. ()
5. []
6. {}
7. ""
8. 【】
9. 「」
10. 『』
11. 《》
12. 〈〉
13. ［］
14. ｛｝
15. 〖〗
16. 〔〕
17. «»
18. <>
19. ‹›

这 19 组是本项目的 authoritative target set。扩展到集合之外必须另行作出明确决定。

实现方向仍复用现有上游 paired-punctuation mechanism；不得因此引入 structured candidate API、Pinyin/Table paired-punctuation plumbing、新 punctuation state machine、LibIME/Fcitx core/Android candidate protocol 修改。19 组目标及其历史实验保留为产品边界参考；D066 已将实验扩展移出当前首发范围并恢复发布 pin 的 upstream 基线。未来若重新开启该扩展，必须另行作出决定并完成适用的 source/runtime 验证；冻结首发范围见 `docs/first-release-baseline.md` §2/§6；首发后的共享引擎映射记录见 [ROADMAP](ROADMAP.md#current-state)。

## 6. 词库与语言模型

- 词库质量优先。
- 允许联网更新词库。
- 词库更新与上传用户输入数据完全解耦。
- 词库/LM 负责词语、词频和排序；MoQi table 负责辅助筛选。
- 优先利用 LibIME 已有用户学习能力。

## 7. Voice Trigger

Voice Trigger 与 ASR Provider 必须解耦。

默认入口为独立麦克风按钮；同时允许用户把长按 Space 配置为 Voice Input：

```text
Microphone Button ─┐
                   ├→ same Voice Trigger → Voice Input
Long-press Space ──┘
```

两种入口共用同一个 Voice Input 会话及其 start/stop/cancel 操作：

- 麦克风按钮：点击开始；active session 显示共享 Voice Session Panel，并提供“取消 / 完成”；“完成”停止录音并等待最终识别结果，“取消”立即放弃本次输入；
- 空格键：按住开始，正常松开停止并等待最终识别结果；按住期间上滑越过阈值进入 cancel-armed，滑回阈值内恢复正常 finish 状态，越界状态下松开才取消；
- active Voice session 期间，Voice Session Panel 覆盖主键盘按键区域但不得销毁正在持有 Space gesture 的 keyboard/gesture owner；状态提示放在手指不会遮挡的位置；
- stop 表示结束录音并等待 final transcript；cancel 表示放弃本次输入，清除临时 partial transcript，不提交文本，并丢弃迟到的回调；
- Direct/Fcitx-owned Audio Capture 能提供 microphone input level 时，Panel 显示由真实 PCM 音量级驱动的实时波形/电平；UI 只消费归一化 level，不取得或拥有 PCM/AudioRecord。System ASR 等不能提供 level 的 backend 使用不依赖 PCM 的静态 Listening indicator。

长按 Space 作为 `SpaceLongPressBehavior.VoiceInput` 接入统一 Voice Input flow，不建立第二套 pipeline。Phase 4B.2 已实现 gesture Down/Move/Up：长按阈值达到 → start，正常松开 → stop，按住上滑越过阈值 → cancel；未显式保存该设置的新安装默认使用 VoiceInput，已有用户已保存的选择保持不变。麦克风与 Space 共用同一个 `VoiceInputSession` 状态：Listening/Recording、Cancel-armed、Processing/Recognizing 均应提供明确反馈。实现提交与真机状态见 ROADMAP。

## 8. Android Voice Input

优先研究和复用 `fcitx5-android` 现有麦克风 UI 及 upstream WIP PR #899 SpeechRecognizer voice-input 工作，而不是重新实现已有 Android voice 基础设施。

Android System ASR 路径：

```text
Fcitx5 Android
↓
SpeechRecognizer
↓
device default RecognitionService
↓
speech implementation
```

该路径是一个 ASR implementation/provider path，不是整个 Voice 架构，也不得成为正式版唯一可用的 Voice 路径。真机 PoC 已证明不同 OEM 的默认 `RecognitionService` 行为可能不同：vivo X100 Pro 可用，而 Redmi K90 Pro Max 的 Xiaomi AsrService 在实际 session 中返回权限不足（error 9）。因此“存在/可发现 RecognitionService”不能等同于“实际 session 一定可用”。

Voice layer 应负责 start/stop、权限、lifecycle、partial/final transcript、取消、错误处理和 UI 状态。正式版还必须至少提供一条由 Fcitx5 控制、与 OEM/system `RecognitionService` 解耦的 ASR 路径；System ASR 不可用或运行失败时，不得导致 Voice 功能整体不可用。

`RecognitionService` 是 Android System ASR 的标准边界，但不是项目内部 ASR backend/Provider abstraction 本身。

### 8.1 Voice 架构（Architecture A，D027）

```text
Voice Input Flow（麦克风 / 长按 Space → 同一 VoiceInputSession）
↓
internal VoiceBackend
├─ SystemAsrBackend → Android SpeechRecognizer → OEM/system RecognitionService
└─ DirectAsrBackend → Fcitx-owned Audio Capture（AudioRecord）→ configured cloud/local/self-hosted ASR
↓
Raw Transcript → IME
```

- **System ASR**：由 OEM/system `RecognitionService` 自行采集音频并识别；复用 upstream PR #899 思路；可选，行为随 OEM 而异。
- **Direct/Fcitx-controlled ASR**：由 Fcitx5 在 IME 进程内用 `AudioRecord` 采集音频，再交给所配置的 ASR；这是正式版可移植性的基础。
- Direct 采集必须有产品侧的硬性安全上限（与自动停用的端点判定相互独立）：到达上限等同于用户停止——正常收尾、交付最终结果并释放麦克风，不是取消、不是错误，也不触发 D035 回退；当现行模型/引擎/协议有更低的上限时采用较低值。语义见 D081。
- `VoiceBackend` 是内部最小 session/backend 边界（start/stop/cancel 与 partial/final/error 事件），不是公开插件框架，也不是以 PCM 为中心的接口；System ASR 不接收 Fcitx5 PCM。
- 当前 Direct ASR 不以自定义 `RecognitionService` 作为边界；仅当需要通过 Android RecognitionService API 对外提供本项目识别器时再评估。

### 8.2 Capture-only AudioRecord 硬关口

接入任何真实 Direct ASR Provider 之前，必须先完成 capture-only `AudioRecord` PoC（不含 ASR、不联网、不持久化音频），并在 **vivo X100 Pro 与 Redmi K90 Pro Max** 上都证明：

- Fcitx5 IME 进程在已授予 `RECORD_AUDIO` 时能打开 `AudioRecord` 并开始录音；
- 采集到的是**真实非静音音频**，而不仅是 `AudioRecord.read()` 返回成功——Android 在并发/后台策略下可能向应用返回静音而非报错；API 29+ 以 client-silenced 状态作为辅助证据；
- stop / cancel / release 在各 lifecycle 路径下正确，系统麦克风占用指示及时消失。

若 Redmi 上无法获得真实非静音音频（被拒绝或被静音），则停止，不接入真实 Direct ASR Provider，先重新评估。

## 9. ASR Provider

ASR Provider 位于 VoiceBackend 之后（见 8.1）：System ASR 由 `SystemAsrBackend` 承载；Local、Managed Cloud、Self-hosted Provider 由 `DirectAsrBackend` 使用 Fcitx5-owned Audio Capture 驱动。Provider 按部署位置/数据去向分为四个逻辑类别（D033；逻辑分类，不是物理模块/APK 拆分）：

```text
ASR Provider
├─ System
│  └─ Android SpeechRecognizer / RecognitionService
├─ Local
│  └─ On-device ASR
├─ Managed Cloud
│  ├─ Doubao / Seed-ASR
│  ├─ Alibaba / Qwen ASR
│  ├─ Tencent Realtime ASR
│  └─ future providers
└─ Self-hosted
   ├─ FunASR 2-pass / Paraformer
   ├─ Fun-ASR-Nano Server
   ├─ sherpa-onnx Server
   └─ Custom / OpenAI-compatible or future protocol adapters
```

在 VoiceBackend 中的位置：

```text
Voice Input Flow
↓
internal VoiceBackend
├─ SystemAsrBackend → SpeechRecognizer → RecognitionService
└─ DirectAsrBackend → Fcitx5-owned Audio Capture
                      ├─ Local（如 sherpa-onnx）→ on-device engine
                      ├─ Managed Cloud（BYOK）→ Provider API
                      └─ Self-hosted（含 OpenAI-compatible 等协议适配）→ 用户配置的 endpoint
↓
Raw Transcript
```

允许云端、本地、自建和 OpenAI-compatible Provider，包括但不限于豆包、阿里云、腾讯、讯飞、FunASR/SenseVoice、sherpa-onnx 等。

所有 Provider 走同一逻辑流程；Self-hosted 不建立独立 Voice pipeline：

```text
Mic / Long-press Space → VoiceInputFlow → Configured ASR Provider → Raw Transcript
→ Optional Text Post Processor / LLM → Final Transcript → IME
```

实现、设备验收、服务凭据和发布状态属于
`docs/first-release-baseline.md`、`docs/ROADMAP.md`、`docs/DECISIONS.md` 和证据文档；本节只规定
Provider 分类与架构边界，不复制时点性 PoC/设备状态。

正式版的可移植性要求：

- OEM/system `RecognitionService` 不得是唯一 ASR 路径；
- 至少一条 ASR 路径必须由 Fcitx5 控制 Audio Capture，并与 OEM RecognitionService 解耦；
- System ASR 可保留为低成本、系统集成良好的 Provider，但是否作为默认 Provider 必须由后续跨设备 PoC 与产品可用性验证决定；
- capability detection 既要考虑静态可用性，也要处理实际 session failure；不得仅因 `RecognitionService` 存在就认定可用；
- 目标是覆盖代表性 Android/OEM 设备并避免单一 OEM speech service 使 Voice 整体失效；不作“所有 Android 设备 100% 可用”的不可验证承诺。

PoC 使用某个 Provider 不得使 Voice Trigger、Audio Capture 或 IME 层绑定该 Provider。对于 Android System ASR，不强制要求 Fcitx5 提供 PCM；对于 direct cloud/local Provider，使用最小的 Fcitx5-owned Audio Capture 边界，不提前建立复杂 Provider framework。

Provider 选择遵循 D028/D033：Provider-specific backend 复用统一 Voice flow；云端采用 BYOK，凭据独立安全存储，切换 Provider 不删除或复用其他 Provider 凭据；Local 不需要云端凭据；ASR 与 LLM 的 Provider/credential 完全分离。维护者凭据不得进入 APK、仓库、CI 或 release。当前产品 Provider/model 状态见 current baseline。

### 9.1 正式 Android Provider 设置与 Local Model Manager

正式 Android UI 应提供一等的 ASR Provider 选择入口，而不是依赖 Developer debug 开关。面向用户的名称优先使用“语音识别服务”；设置页按四类展示：系统自带 / System、设备端 / Local、第三方云端 / Managed Cloud、自建云端 / Self-hosted。每个已接入的具体服务可以独立配置和启用，允许同时启用多个；“当前使用”只选择一个已启用的具体服务，不提供长期 Auto 选项。顶层设置方向（D034，非最终 UI 规格）：

```text
语音输入
├─ 系统自带：Android 系统语音识别
├─ 本地集成（不联网）：正式 Local 模型
├─ 第三方云端 ASR：已接入的服务
├─ 自建云端 ASR：已配置的服务
├─ 当前使用：已启用的具体服务
└─ 麦克风按钮、本地模型管理等其他设置
```

每个 Provider 只显示其所需的配置；云端 Provider 可包含 API Key、model、endpoint/resource 等，Local Provider 不显示 API Key。配置、启用和“当前使用”分别持久化；禁用当前服务时需提示重新选择，不得静默选中另一家云端服务。可提供 Provider-specific“测试配置”，但不得为了测试凭据而未经明确告知上传用户录音。

Local ASR 的 **Provider / runtime / model** 必须区分：例如 Local Provider 可使用 sherpa-onnx runtime，而 Zipformer、FunASR Nano 等是具体 model；runtime 可用不等于任一模型已获准打包或再分发，模型许可须逐一核对。

正式版应提供 Local Model Manager/Downloader，至少覆盖 model catalog、大小/版本/License 展示、下载/重试（是否支持暂停/断点续传按实现验证）、完整性校验、原子安装、更新与删除。大型 Local 模型原则上不因启用 Voice 而强制内置 APK；模型安装完成后，Local ASR 的日常识别应能完全离线工作。Downloader/Model Manager 不属于 4B.3b 最小 PoC，须在实际 runtime/model 的文件结构、加载方式和许可确认后设计。

4B.3b A/B PoC 的模型均不打包进 APK，也不实现下载器；使用固定模型文件/hash，通过 `adb` 放入测试设备可访问的应用目录。正式产品不要求用户使用 adb，而是在后续 Model Manager/Downloader 中按需获取已通过许可审查的模型。A 的模型权重许可未明确前，不得进入正式模型目录、release artifact 或由项目提供下载。第三方 runtime、模型与训练数据许可必须分层记录；Local ASR 引入前建立并维护 `docs/THIRD_PARTY_LICENSES.md`。

### 9.2 首次推荐具体服务（D034）

产品目标为“开箱即用优先”。首次安装/首次启用语音输入时运行一次推荐规则，将结果保存为“当前使用”的具体服务：

1. 已安装且健康的 Local 模型 → Local；
2. 否则 System ASR 可用：用户已授权 System ASR → System；尚未授权 → 使用前先显示一次性披露/授权（不授权则按第 3 步处理）；
3. 否则提示当前没有可用的识别服务，并提供配置入口：安装 Local 模型、配置 Managed Cloud、配置 Self-hosted。

首次推荐不得静默选择 BYOK Managed Cloud，也不得静默选择用户配置的 Self-hosted endpoint。之后可用性变化不重新运行推荐规则，也不悄悄更改“当前使用”；早期故障按 9.3 处理。修订后的具体服务选择、首次推荐、Voice Settings 与迁移逻辑已在当前 Android 产品线实现；设备验收仍按对应验收文档区分。

System ASR 授权：System ASR 与 Local ASR 在隐私上不等价，显式选为“当前使用”或首次推荐时，使用前需要用户事先一次性授权；启用条目不应绕过披露。设置应披露该服务由 Android/设备系统服务提供、语音数据处理取决于该系统服务且可能涉及远程处理；最终文案与 UI 未冻结。已有授权不意味着同意将 System 用作自动回落。

首次推荐与 Local 模型按需下载必须遵循 D034/D050；推荐资格、配置持久化与 runtime readiness 是独立维度。

### 9.3 自动 fallback（D035）

核心隐私规则：未经用户事先明确授权，自动 fallback 不得扩大可能接收用户语音数据的参与方/处理方集合。Local ASR 是 Fcitx 控制的设备端处理；System ASR 是独立的信任/数据处理边界，由 Android/OEM/system `RecognitionService` 实现，可能在本地或远程处理，Fcitx 不得假定其仅在本地处理。

自动 fallback 默认开启。Managed Cloud 与 Self-hosted 同级，不在两者之间自动切换：

- 当前使用 Managed Cloud / Self-hosted：该具体服务 →（启动/早期技术失败）已安装、启用且健康的正式 Local → 识别失败；Local 不可用时直接报告失败；
- 当前使用 Local 或 System：其失败直接报告失败。

**System 即使已授权也永不进入自动 fallback 链**；它只在用户显式选为当前服务，或首次推荐经披露/授权选定为当前服务时使用。不得静默发生 Doubao → Alibaba、Alibaba → Tencent、Managed Cloud ↔ Self-hosted、Local → 云端、任何服务 → System 等。

V1 只处理启动/早期技术失败：Provider 不可用、无网络/连接失败、endpoint 不可用、认证/服务初始化失败、可用识别会话建立前的早期超时。早期边界是“可用识别会话已建立”，现有 `onStarted` 太早，不能用它判定。V1 不做会话中途跨 Provider PCM replay/迁移、不因质量差自动重识别、不做双 Provider 同时识别、不把已采集音频静默重放给其他第三方；用户停止/取消后不切换，会话中途失败报告失败/允许重试。没有正式 Local 模型时不把研究/调试模型当作产品回落目标。

## 10. ASR 与 LLM 解耦

```text
Audio
↓
ASR Provider
↓
Raw Transcript
↓
Optional Text Post Processor
↓
Final Transcript
```

LLM 必须可以完全关闭。ASR Provider 与 LLM Provider 分别选择和配置。LLM 可用于纠错、标点、断句、格式化、口语整理、翻译等。

## 11. 语音隐私与数据流

必须能够明确：

- 何时开始/停止录音；
- 谁打开 microphone；
- 当前 ASR Provider；
- 上传什么、发送到哪个 endpoint、何时停止上传；
- ASR 返回的 Raw Transcript；
- 是否继续发送给 LLM；
- LLM Provider/endpoint；
- 最终提交给 IME 的文本；
- 是否发生自动 fallback，以及实际使用的 ASR Provider。

启用词库更新不等于上传用户输入；启用 Voice Trigger 不等于选择某家云 ASR；启用 ASR 不等于把 transcript 自动发送给 LLM；首次推荐与自动 fallback 不得静默把音频发送给用户未选择的 Managed Cloud 或 Self-hosted 服务；System ASR 只在用户显式选择为“当前使用”或首次推荐经披露/授权后使用，从不自动回落到它（D034/D035）。

## 12. 最小修改边界

### MoQi / Auxiliary Filter

优先仅修改 `fcitx5-chinese-addons`。当前不修改：

- LibIME；
- Android candidate frontend protocol；
- MoQi-specific Android UI。

### Voice

后续优先复用 `fcitx5-android` 现有能力、upstream SpeechRecognizer 工作（System ASR backend）和 Android `SpeechRecognizer/RecognitionService`；Direct ASR 所需的 Audio Capture 与 VoiceBackend 边界保持在 `fcitx5-android` 的 voice 模块内部、最小化。Provider-specific 实现与 Voice Trigger 分离。

## 13. Phase 2 PoC Exit Criteria

必须端到端验证：

```text
Pinyin/Shuangpin
→ candidates
→ Auxiliary Filter Trigger
→ configured MoQi
→ real MoQi code
→ candidate filtering
→ partial selection
→ composition preserved
→ continue input
→ Auxiliary Filter Trigger again
→ second filtering/selection
```

同时验证：

1. Pinyin；
2. Shuangpin；
3. Disabled / Stroke / MoQi；
4. 真实固定 MoQi table；
5. Backspace；
6. Escape；
7. 不强制整句 commit；
8. Stroke regression；
9. continued composition；
10. repeated Auxiliary Filter use。

PoC 证明现有接口不足前，不修改 LibIME。
