# Third-Party License Inventory

本文件用于跟踪项目直接使用、修改、分发或计划引入的第三方代码、数据和模型。它是工程合规清单，不替代最终发行前的法律审查。

原则：

- runtime license、model weights license、training-data provenance 分层记录，不互相推导；
- “可使用”不等于“可随 APK/模型目录再分发”；
- release 前按实际 SBOM、最终 artifact 与上游原始 LICENSE/NOTICE 再核验；
- 未核实项明确标记，不以平台 metadata 代替最终原始许可证文件。

| Component / artifact | Role | Current license evidence | Distribution status / obligation | Verification |
|---|---|---|---|---|
| fcitx5-android | Android IME base / modified fork | LGPL-2.1-or-later | 商业使用允许；分发时履行 LGPL，对修改的 covered code 提供对应源码并保留 notices/license | 已知；release 前复核最终版本 |
| fcitx5 | core | LGPL-2.1-or-later | 同上 | 已知；release 前复核 |
| `FusionSettings.exe` retained configtool subset | Windows upstream-derived settings executable / derivative UI adapter | **GPL-2.0-or-later** for this executable and derivative settings code only (D086; upstream copyright/SPDX notices preserved) | Owner-test package must ship scoped notices, corresponding source/build instructions, and exact source pins; this does **not** relicense FusionHost/TIP/tray or the repository | Implemented in `fusion-windows` settings PR; fork source `choicky/fcitx5-configtool` PR #1; final release/legal review remains required |
| fcitx5-chinese-addons | Pinyin/Shuangpin/Auxiliary Filter / modified fork | LGPL-2.1-or-later | 同上；当前 MoQi/Unified Auxiliary Filter 修改属于 fork 修改 | 已知；release 前复核 |
| LibIME | Pinyin/LM runtime | LGPL-2.1-or-later | 商业使用允许；按实际链接/分发方式履行 LGPL | 已知；release 前复核 |
| gaboolic/moqima-tables / moqima_gb18030.txt | MoQi table | MIT；固定 commit 6d8ba8f1c57466f358e682baefe11bbd0fe389ab | 可商业使用/再分发；保留 copyright + MIT license | 已验证项目基线 |
| Rime Frost / `rime-frost.dict` | Phase 5B third-party dictionary | GPL-3.0-only；固定 commit `211de1ca927b6c876e384c6de42e1cc8af868c68`，LICENSE SHA-256 `3972dc9744f6499f0f9b2dbf76696f2ae7ad8af9b23dde66d6af86c9dfb36986` | Phase 5B release set；保留 GPL license 与 source revision attribution | 已核固定提交/LICENSE |
| Wanxiang `jichu` / `rime-wanxiang.dict` | Phase 5B third-party dictionary | CC-BY-4.0；固定 commit `94f1e8d7b6d1267a9c8752a2e62145705dd1fb92`，LICENSE SHA-256 `9e5f1b3c610b9c2da5c313bf81d577a7d1acec686bdb0384edefa6df0f90cd94` | Phase 5B release set；保留 attribution、license 与 source revision | 已核固定提交/LICENSE |
| Rime Ice / CustomPinyinDictionary | Phase 5B excluded candidates | Ice: GPL-3.0-only but unresolved phrase-level pronunciation semantics; Custom: no verifiable redistribution license and README identifies multiple third-party data sources | 不进入 release artifact 或公开下载目录；仅在固定 manifest 中记录 provenance | Phase 5A evidence + Phase 5B audit |
| sherpa-onnx v1.13.8 | Local ASR runtime candidate | Apache-2.0 | PoC runtime；正式分发时保留适用 license/copyright/NOTICE | 4B.3b-0 已核 |
| ONNX Runtime used by sherpa-onnx | inference runtime | MIT（checkpoint 中尚未直接读取最终引入版本 LICENSE） | PoC 前/正式引入时核最终 artifact 与 LICENSE | 待最终 artifact 复核 |
| A: sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30 | Local ASR model A / OnlineRecognizer | **模型权重未声明明确 license**；训练数据含若干 non-commercial 条款，条款对权重的法律效果未判定 | **research/device-evaluation only**；许可澄清前不得进入公开发布、正式模型目录，项目不打包/镜像/托管其权重。2026-09-28 所有者个人测试决定（D037 修订）：测试（debug）构建的 Model Manager 从 HF 转换仓库固定 revision `ad658fa0` 下载并校验 SHA-256；该仓库公开、无访问门槛、无 license 元数据；原始检查点 `yuekai/icefall-asr-multi-zh-hans-zipformer-large` 在 HF 设访问门槛（同意分享联系方式）且未声明 license | **公开发布：BLOCKED（许可未解决）**；个人测试构建：可下载（debug-only） |
| B: sherpa-onnx-funasr-nano-int8-2025-12-30 | Current Local ASR / OfflineRecognizer | Fun-ASR-Nano HF metadata Apache-2.0；FunASR 模型协议 v1.1；第三方 ONNX export metadata Apache-2.0；导出者仓库无 LICENSE，转换/NOTICE 链仍待补证 | D050 当前产品模型；Model Manager 从固定 HF revision `6f16bd378457e13f36ccf3910df9017f96c346fb` 逐文件下载并校验 SHA-256；项目不托管、不镜像、不内置 APK。应用内固定上游按需下载是所有者确认的项目发布口径，不等于独立法律审计 PASS；34–39 s 空 final 限制保留 | 逐模型证据未完全闭合；不再写成“尚未批准因此阻止 APK 发布” |
| C: sherpa-onnx-streaming-zipformer-bilingual-zh-en-2023-02-20 | Local ASR 候选 C / OnlineRecognizer（评估中） | HF 镜像元数据 `apache-2.0`；上游 `pfluo/k2fsa-zipformer-chinese-english-mixed` 元数据与 README 均为 `apache-2.0`；训练数据未公开（UNVERIFIED） | Model Manager 从 HF 固定 revision `98590b7e` 逐文件下载并校验 SHA-256，项目不托管；非正式、实验性候选，待双机设备结果 | 元数据验证；训练数据来源未核实 |
| X-ASR offline INT8 / `sherpa-onnx-x-asr-zipformer-transducer-zh-en-punct-int8-2026-06-03` | Current Local ASR / OfflineRecognizer | Author card/source repository declare Apache-2.0; exact archive has no embedded LICENSE/NOTICE; exporter/checkpoint mapping remains incomplete | Fixed upstream asset `460927314`, archive SHA-256 `5d02c36d7b44e886b7c8f0d8e051f8713acab96c264bb6ef9e718be39a6a2224`; project does not host/mirror/embed weights; evidence gap remains | 应用内固定上游按需下载允许（所有者确认口径）；独立法律审计未完成 |
| X-ASR 960 ms streaming INT8 / `sherpa-onnx-x-asr-960ms-streaming-zipformer-transducer-zh-en-punct-int8-2026-06-05` | Current Local ASR / OnlineRecognizer | Same Apache-2.0 author/source evidence; exact archive has no embedded LICENSE/NOTICE; exporter/checkpoint mapping remains incomplete | Fixed upstream asset `460927089`, archive SHA-256 `0a92b798bd6801c333c7ce8aebf5ba769bfe7f3f3511699a67837b2288428603`; project does not host/mirror/embed weights; evidence gap remains | 应用内固定上游按需下载允许（所有者确认口径）；独立法律审计未完成 |
| Doubao Seed-ASR 2.0 API | Direct Cloud ASR PoC | 服务/API条款，不是 OSS model/runtime license | BYOK；维护者 credential 不进入 repo/APK/CI/release；正式产品按届时服务条款复核 | PoC only |

## Phase 4B.3b PoC policy

- A/B 模型均不打包进 APK。
- 使用固定模型文件/hash，通过 `adb` 放入测试设备可访问的应用目录。
- 当前不实现 Model Manager/Downloader。（历史记录；此后 D037 与其 2026-09-28 修订引入 Model Manager，见上表。）
- A 的许可 blocker 仍是旧研究候选的历史限制；A 已从 D050 当前支持集删除。
- B/Nano 的逐项 archive、转换和 NOTICE 证据仍待补齐，但当前接受固定上游按需下载口径；不得把缺口写成独立法律审计 PASS，也不得笼统宣称阻止 APK 发布。
- 正式版仍须保留精确来源、revision、SHA-256、归属和可获得的许可证/NOTICE 证据。

## Release gate

## Current distribution interpretation (D050, 2026-10-02)

- 当前接受的项目发布方式是 APK 内应用从固定上游按需下载 FunASR Nano 与两个 X-ASR。
- 项目不托管、不镜像模型权重，模型权重不内置 APK；这三种形态不得混写。
- 两个 X-ASR 保留上游 Apache-2.0 声明；Nano 保留 Apache-2.0、FunASR 模型协议 v1.1 和署名/模型名称证据。
- 精确来源、revision、URL、SHA-256、archive LICENSE/NOTICE 与转换来源缺口继续逐项记录；本段是所有者确认的项目发布口径，不等同于独立法律审计，也不把待补证自动写成 APK 发布阻断。

### Four-rights closure (D059, 2026-10-05)

上表逐模型许可证据不变；D059 在此之上把“分发”拆成四项彼此独立的权利并给出当前三模型的权威答案：
① bundle 进 APK = 项目主动不做（许可证据不足以肯定授予再分发）；② mirror/re-host = 项目主动不做；
③ 应用内 fixed-upstream 下载入口 = 所有者批准（D050），当前 release 已实际提供；④ manual import = 用户
自身合法取得的文件，非项目分发行为。“独立法律审计是否完成”不 gate ③，仅作如实 audit-pending 披露。
因此本清单中 Nano/X-ASR 的未闭合 provenance/license/NOTICE 条目仍是证据状态，不能被 D050/D059
改写为审计 PASS；同时它们也不应被扩大为固定上游下载或首发发布的笼统阻断项。D059 的六项 reopen
conditions 仅在 `docs/DECISIONS.md` 中规定的实质新事实出现时适用。
Android 侧陈旧字段 `distributionApproved`（原唯一作用是给产品模型显示“research only”横幅,从不 gate 下载）
已重命名为 `licenseAuditComplete`。首发下载 UI 判定 **GO**。reopen conditions 见 `docs/DECISIONS.md` D059；
除其列出的情形外不重开 A/B/C 研究阶段的分发结论。

正式发行前至少完成：

1. 从最终依赖/SBOM 反查实际版本与 artifact；
2. 保存并核对各组件原始 LICENSE / NOTICE / copyright；
3. 核 LGPL covered code 的 source availability 与修改源码对应关系；
4. 对每个随包或由产品提供下载的 ASR model 单独核权重 license 与 redistribution；
5. 对许可证不明确的模型从 release artifact、catalog 与项目托管下载中剥离。
| Self-hosted 服务器（sherpa-onnx、FunASR、Fun-ASR-Nano） | 用户自建的识别服务 | 本项目只实现客户端协议；服务器、模型由用户自行部署与取得许可 | 不随 APK 分发；互通测试所用的上游模型仅用于本机测试，未提交、未分发 | 不适用于分发 |
| Qwen（阿里云百炼）、腾讯云实时语音识别 | Managed Cloud 服务 | 服务条款，不是开源许可 | BYOK；维护者凭据不进入仓库/APK/CI/release | 服务条款未逐条复核 |
