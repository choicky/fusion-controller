# ASR 服务产品化 — 设备 / 凭据 / 服务器验收脚本

对象：`fcitx5-android` `phase4-voice-poc` 最新已通过 CI 的提交（见工作日志最后一条；记录实际 SHA）；debug APK = CI artifact `moqi-debug-apk`。Candidate B 历史基线 `a8a0e1b3` 不变；此前 `fb3b0c26` 的设置基础验收不覆盖本批。

**2026-10-13 later owner-reported acceptance addendum:** The 2026-09-28 tables below are historical records and remain unchanged. After that snapshot, the owner reported that all three production Local ASR models completed the release product-path lifecycle (download, pause, resume, cancel, enable, and actual speech recognition); Alibaba/Qwen ASR and Tencent ASR both succeeded on Android with real services and real credentials; and the core D035 early external/service-failure fallback to an installed, enabled, runtime-ready production Local ASR continued the voice operation. These reports close only those core gates. Long sessions, exhaustive resource/latency measurements, every interruption/cancellation/retry race, quota/auth/provider-error cases, and Self-hosted device coverage remain open validation debt. This addendum is owner-reported evidence, not an independent rerun by this documentation task.

**2026-10-25 host-side fusion.14 evidence (device pending):** A fixed corpus was built from the four Apache-2.0 `test_wavs` in the pinned X-ASR archives (Mandarin and Chinese-English code switching), two 1 s-silence concatenations (14.5 s and 16.2 s), and 1 s of empty audio. Baseline and candidate ran through the real sherpa-onnx 1.13.8 adapter configuration (numThreads=2, 20 ms chunks), not a separate pipeline. The lexical reference is the offline X-ASR final — a reference output, not a human transcript — so the figures are agreement with that reference, not accuracy against ground truth. Streaming greedy with the old 960 ms tail scored mean 0.0767 and 0/7 terminal punctuation; beam-4 with 960 ms scored 0.0624 and 1/7; beam-4 with 1920 ms (the published fusion.14 configuration) scored 0.0584 and 1/7. One intermediate greedy run exists in the evidence (`x-960-greedy-tail2000-corpus.json`, 0.0726), but its recorded tail length is inconsistent between the result filename and an earlier summary and no per-run command log exists; it is not used as a data point. The 16.2 s clip lost its last two characters with 960 ms and was complete with 1920 ms. Offline X-ASR emitted terminal punctuation on all six speech clips. The streaming export's final token sequence ends without a punctuation token, so the missing mark is model output, not finalization or editor loss; the release does not append a period or infer a question mark, and reliable streaming terminal punctuation is not claimed. FunASR Nano returned nonempty, punctuated finals on all six speech clips with no truncation, and all six installed files matched the catalog sizes and SHA-256 values; the earlier device baseline used the same revision and adapter, so no model-identity or configuration regression was reproduced host-side, but the owner's Nano quality concern remains unresolved. The D081 30 s Nano ceiling is unchanged. These are host-side measurements and identity checks, not Android capture/JNI, latency, resource or device-quality acceptance; device acceptance remains pending and was explicitly not a publication gate for v0.1.3-fusion.14 (release-specific owner authorization recorded under D063–D064). Shared-engine pin for this release: `2145c4b` (same as fusion-windows); the packaged data files match that tree, and the native library revision rests on the build record rather than an independent binary comparison.

**Historical snapshot status:** The 2026-09-28 “部分执行” wording and empty detailed rows describe the evidence available at that time; they must not be read as the current authority after the addendum above. The test script remains useful for genuinely unexercised cases.

## 0. 已有的非设备证据（供对照）

| 项 | 证据 | 级别 |
|---|---|---|
| 选择/推荐/迁移/fallback 纯逻辑 | 本机 JVM 95 项测试 + CI | LOCAL-JVM + CI |
| sherpa-onnx 客户端 | 上游 v1.13.8 Python streaming_server 本机互通（3 个 wav） | INTEROP |
| FunASR 2-pass 客户端 | 上游 FunASR 1.4.16 Python 2pass 服务器本机互通（3 个 wav，含标点修正） | INTEROP |
| OpenAI-compatible 整段转写 | 上游 FunASR 1.4.16 `funasr-server --model sensevoice --device cpu` 本机互通（2 个 wav，stop→final 2.1/3.0 s） | INTEROP |
| Qwen / Tencent / Fun-ASR-Nano 客户端 | 按文档/源码的本地协议仿真器；Tencent 签名与独立 Python 实现一致 | EMULATION（非真实服务） |
| B 模型下载 | 真实上游 HF 固定 revision 下载 1,009,605,061 字节，6 文件 SHA-256 匹配，原子安装 | JVM + 真实上游 |
| A、C 模型下载 | 真实上游 HF 固定 revision 下载并 SHA-256 校验、原子安装（A 167,360,920 字节）；A 下载进程在约 98 MB 处被 SIGKILL 后重跑，经 HTTP Range 续传完成（HF CDN 返回 206） | JVM + 真实上游 |
| 断点续传 / 校验失败 / 取消 / 旧 adb 目录删除 / 用户地址策略 | `LocalModelInstallerTest`、`ModelSourcesTest`（JDK HttpServer + 真实 OkHttp：Range、无 Range、416、错误文件、404） | LOCAL-JVM + CI |
| FunASR 2-pass 首包顺序 | 慢握手本机服务器复现：修复前首帧为音频，修复后为配置 JSON；`FunAsr2PassClientTest` | LOCAL + CI |
| 错误详情脱敏、last-error 移出 prefs | `ErrorRedactionTest`、`VoicePrefsTest` | LOCAL-JVM + CI |
| Android UI、Keystore、AudioCapture 路径、真实云端 | — | UNTESTED |

## 1. 设置结构与迁移（vivo + Redmi）

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 1.1 | 从 `fb3b0c26` 升级安装（不清数据），原设置为“自动”且已允许 System | 当前服务 = Android 系统语音识别；不出现“自动”选项 | | |
| 1.2 | 原设置为“本地”，Developer 研究模型 = B | 当前服务 = 本地 · B；B 已启用（`7ed0fa78` 起每个模型单独一项） | | |
| 1.3 | 全新安装，打开语音输入设置 | 分组：当前使用 / 系统自带 / 本地集成 / 第三方云端 / 自建云端 / 其他；当前 = 未选择；出现“一键推荐（仅首次）” | | |
| 1.4 | 点“一键推荐”（System 可用、未授权） | 弹披露；允许 → 当前 = 系统，推荐项消失；不允许 → 当前仍未选择，推荐项消失 | | |
| 1.5 | 启用豆包但不填凭据，选为当前 | 当前服务摘要显示“尚未填写凭据”；麦克风按钮隐藏；长按空格 → 提示并打开设置；**不**改选其他服务 | | |
| 1.6 | 停用当前服务 | 摘要显示“已停用”；不改选 | | |
| 1.7 | 当前服务选择列表 | 只列出已启用的服务；未接入的服务不出现 | | |

## 2. Local Model Manager：A/B/C 下载（个人测试 checkpoint，D037 2026-09-28 修订）

范围：debug（测试）构建。A、B、C 都是研究模型：不推荐、不作 fallback；A 的下载仅限个人测试，许可对公开发布仍未解决。两台手机各执行一遍；PC 为 Windows，PowerShell。

### 2.0 准备（PowerShell）

```powershell
# 0) 工具：adb（platform-tools）、gh（GitHub CLI，已登录）、python（仅 2.5/2.6 需要）
adb devices -l                                   # 记下两台手机的 serial
$vivo  = "<vivo serial>"
$redmi = "<redmi serial>"
$pkg   = "org.fcitx.fcitx5.android.debug"

# 1) 取最新的已通过 CI 的 debug APK（HEAD 见工作日志最后一条；两种方式任选其一）
#    a. 下载 CI 产物
gh run list -R choicky/fcitx5-android -b phase4-voice-poc -w "MoQi test APK" -L 3
gh run download <run id> -R choicky/fcitx5-android -n moqi-debug-apk -D .\moqi-debug-apk
$apk = (Get-ChildItem .\moqi-debug-apk\*.apk | Select-Object -First 1).FullName
#    b. 或本机构建（仓库已按 CI 的方式切换 chinese-addons 子模块到 feature/moqi-filter）
#    git fetch origin; git checkout phase4-voice-poc; git pull --ff-only
#    git submodule update --init --recursive
#    $env:BUILD_ABI = "arm64-v8a"; .\gradlew.bat :app:assembleDebug
#    $apk = (Get-ChildItem .\app\build\outputs\apk\debug\*.apk | Select-Object -First 1).FullName

# 2) 安装（保留数据）
adb -s $vivo  install --no-streaming -r $apk
adb -s $redmi install --no-streaming -r $apk
adb -s $vivo shell dumpsys package $pkg | Select-String versionName   # 记录版本

# 3) 看日志（另开一个窗口；每台手机一个）
adb -s $vivo logcat -c; adb -s $vivo logcat | Select-String "Local model|Local ASR|capture failed"
```

打开：Fcitx5 设置 → 语音输入 → “本地集成”分组中的 A、B、C 三行。

若手机网络无法访问 huggingface.co：对该模型用“从其他地址下载…”，把地址中的 `https://huggingface.co` 换成你选择的镜像（例如第三方镜像 `https://hf-mirror.com`，路径保持 `/<仓库>/resolve/<完整 revision>`）。这是你主动选择的来源，App 不会自动切换；文件仍逐个按固定 SHA-256 校验，不符即不安装。记录实际使用的地址。

### 2.0a 设备观察记录（所有者报告）

| 日期 | 设备 | APK | 观察 | 结论 |
|---|---|---|---|---|
| 2026-09-28 | vivo X100 Pro | `7af0cc16` debug | 语音识别服务选择正常；豆包 BYOK 识别正常（见 3.1） | PASS（所有者报告） |
| 2026-09-28 | vivo X100 Pro | `7af0cc16` debug | A/B/C 三行可见；点按任一行，对话框只显示模型说明与“取消”，**没有**下载、从其他地址下载、导入、使用、删除等操作 | **FAIL**——模型下载 checkpoint 被阻塞；下载、安装、本地识别均**未测** |
| 2026-09-28 | vivo X100 Pro | `6007c8ca` debug | A、B、C 各自下载完成，之后显示为已安装 | 下载与安装完成：PASS（所有者观察）；校验失败、中断续传、空间不足等未单独测 |
| 2026-09-28 | vivo X100 Pro | `6007c8ca` debug | 任一模型下载期间，语音设置页在正常内容与大部分空白之间快速闪烁；下载完成后立即停止 | **FAIL**（设备观察的 UI 失败；历史证据，保留）；已由 `7ed0fa78` 修复，双机复测通过（见下行） |
| 2026-09-28 | vivo X100 Pro、Redmi K90 Pro Max（两台结果相同） | `7ed0fa78` debug | APK 升级后豆包 API Key 仍保留 | PASS（所有者报告） |
| 2026-09-28 | 同上 | `7ed0fa78` debug | 分别下载 A、B、C 期间设置页不闪烁 | PASS（A、B、C 三者；所有者报告）——`6007c8ca` 闪烁 FAIL 的修复在两台手机上通过 |
| 2026-09-28 | 同上 | `7ed0fa78` debug | A、B、C 可各自启用、各自被选为当前服务 | PASS（所有者报告） |
| 2026-09-28 | 同上 | `7ed0fa78` debug | 切换当前识别服务 | PASS（所有者报告） |
| 2026-09-28 | 同上 | `7ed0fa78` debug | 实际本地识别：A 中文好、英文差；B、C 中文与英文均可；三者延迟主观可接受 | 观察记录（非性能 PASS）：未报告是否断网、录音时长、实测 stop→final/RTF/PSS；**不据此选定正式/默认 Local 模型** |
| — | vivo / Redmi | — | 断网确认、长语音（约 40 s 以上）、实测延迟/RTF/PSS、取消/继续、中断续传、校验失败、删除、旧设置精确迁移（1.1/1.2/2.0c.9）、Qwen/腾讯/自建/fallback | 未测 |

原因（源码）：`modelActions()` 在同一个 AlertDialog 上同时 `setMessage()` 与 `setItems()`，AlertDialog 只有在没有 message 时才把列表放进对话框。修复：Android `3116a7b8`（操作列表对话框不再带 message；详情作为单独一项）。此后 `6007c8ca`（vivo）与 `7ed0fa78`（两台）上已通过该对话框下载并启用模型，间接说明操作可见；§2.0b 各项的具体内容未逐项报告，仍留空。

### 2.0b 修复后先确认操作可见（每台手机）

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.0b.1 | 未安装的 A/B/C 行 | 行内第一行为状态与下一步（如“未安装 · 约 168 MB · 点按下载”），第二行为一句简短说明；不再列出完整来源 URL 与长许可说明 | | |
| 2.0b.2 | 点按未安装的模型 | 列表中可见：下载（约 N MB）、从其他地址下载…、导入模型文件…、详情：来源、许可、限制；以及“取消”按钮 | | |
| 2.0b.3 | 点“详情” | 显示版本、大小、SHA-256 校验说明、来源（仅可下载的构建）、完整许可与限制说明 | | |
| 2.0b.4 | 下载中点按该行 | 显示“取消”；行内显示百分比与 MB | | |
| 2.0b.5 | 下载完成后点按 | 显示“启用并用于语音输入”（已启用时为“用于语音输入”）、“删除”、“详情”（`7ed0fa78` 起的语义见 §2.0c） | | |

可选的仪器测试（不清除 App 数据；需本机构建，CI 只编译不运行）：

```powershell
.\gradlew.bat :app:assembleDebug :app:assembleDebugAndroidTest
$apk  = (Get-ChildItem .\app\build\outputs\apk\debug\*.apk | Select-Object -First 1).FullName
$tapk = (Get-ChildItem .\app\build\outputs\apk\androidTest\debug\*.apk | Select-Object -First 1).FullName
adb -s $vivo install --no-streaming -r $apk
adb -s $vivo install --no-streaming -r -t $tapk
# 找到测试 APK 注册的 instrumentation（形如 <测试包>/androidx.test.runner.AndroidJUnitRunner）
$inst = ((adb -s $vivo shell pm list instrumentation | Select-String "target=$pkg\)").ToString() -split ' ')[0] -replace '^instrumentation:', ''
adb -s $vivo shell am instrument -w -e class org.fcitx.fcitx5.android.ui.main.settings.behavior.ActionListDialogTest $inst
adb -s $vivo uninstall ($inst -split '/')[0]    # 只卸载测试 APK，App 数据保留
```

预期输出 `OK (2 tests)`。不要用 `gradlew connectedDebugAndroidTest`：它在结束时会卸载 App，清除设置与凭据。

### 2.0c 下载期间的页面稳定性与“安装 / 启用 / 当前使用”（`7ed0fa78` 起）

闪烁原因（源码）：进度每变化 1% 就触发 `VoiceSettingsFragment.render()`，整页删除并重建全部 Preference；`PreferenceGroupAdapter` 的稳定 id 来自每个 Preference 对象，于是每次重建全部 id 改变，RecyclerView 默认动画把整页当作删除+插入淡出淡入。修复：进度只更新该模型那一行的摘要（同一 Preference），状态变化（开始/完成/失败/取消）才整页重建一次，且该列表关闭条目动画。

设计变更（所有者要求，2026-09-28）：A/B/C 各自是一个服务，像云端服务一样单独启用；安装、启用、当前使用三者分开。取消“启用本地语音识别”总开关与单独的“本地识别模型”选择。

`f651a082` 起行文字与布局已变（开关在模型行内、状态文字更短），以 §2.6 为准；本表的语义（安装/启用/当前分开、不自动改选）不变。

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.0c.1 | 下载 C（约 199 MB）期间停留在语音设置页，上下滚动 | 页面不闪烁、不变空白；只有 C 行的百分比/MB 变化；滚动位置保持；其他开关可正常操作 | 不闪烁（摘要，`7ed0fa78`，A/B/C）；滚动与其他开关未报告 | 同左 |
| 2.0c.2 | 下载 B（约 1010 MB）期间离开设置页再回来 | 回来后 C/B 行显示当前进度并继续更新，页面不闪烁 | | |
| 2.0c.3 | 下载中点按该行 → 取消；再点按 → 继续下载 | “取消”随时可用；取消后显示“已取消”；继续下载从断点开始 | | |
| 2.0c.4 | 下载完成 | 该行变为“已安装 · 未启用 · 点按使用”，下方出现“启用模型 X 用于语音输入”开关（关）；当前服务**不变** | | |
| 2.0c.5 | 同时打开 A 与 C 的启用开关 | 两个都启用；“语音识别服务”列表中分别出现“本地 · A…”与“本地 · C…”；未安装或未启用的模型不出现 | 独立启用（摘要，`7ed0fa78`）；行文字与列表细节未报告 | 同左 |
| 2.0c.6 | 在列表中选“本地 · C…” | 当前服务 = 本地 · C；C 行显示“已安装 · 已启用 · 正用于语音输入”；A 行为“已启用” | 选择/切换当前服务（摘要，`7ed0fa78`）；行文字未报告 | 同左 |
| 2.0c.7 | 关闭 C 的启用开关 | 当前服务仍显示 C，并标“已停用”；不自动换成 A 或其他服务 | | |
| 2.0c.8 | 重新启用 C，然后删除 C | C 行变回“未安装”，开关消失；当前服务显示 C“此本地模型未安装”；不自动改选 | | |
| 2.0c.9 | 从 `6007c8ca` 升级（不清数据）：原先“启用本地语音识别”开、本地模型 = B、当前 = 本地 | 升级后：B 已启用，当前服务 = 本地 · B；A、C 未被启用 | | |

仪器测试（需本机构建；不清除 App 数据；运行时会在前台打开语音设置页）：

```powershell
.\gradlew.bat :app:assembleDebug :app:assembleDebugAndroidTest
# 安装 $apk 与 $tapk、找到 $inst 的命令同 §2.0b
adb -s $vivo shell am instrument -w -e class "org.fcitx.fcitx5.android.ui.main.settings.behavior.VoiceSettingsProgressTest,org.fcitx.fcitx5.android.ui.main.settings.behavior.ActionListDialogTest" $inst
```

预期 `OK (4 tests)`。VoiceSettingsProgressTest 用假任务（不下载）对 A/B/C 依次上报进度，断言：不整页重建、该行是同一 Preference、条目数与滚动位置不变、任务结束时恰好重建一次；并测试任务进行中关闭再打开页面。它**不能**代替 2.0c.1–2.0c.3 的肉眼观察。

### 2.1 显示与下载

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.1 | 查看 A/B/C 三行 | 每行：状态与下一步、约 N MB、一句简短说明（A：仅个人测试；B：约 1 GB、长语音可能无结果；C：实验性候选）；来源、许可与限制在“详情”中（`3116a7b8` 起） | 行可见（`7af0cc16`，旧布局）；新布局未测 | |
| 2.2 | 点 A →“下载（约 168 MB）” | 确认框写明：HF 转换仓库固定 revision ad658fa0、未声明许可、原始检查点在 HF 有访问门槛、仅限个人测试、不得分享文件 | | |
| 2.3 | 确认下载 A；离开设置页再回来 | 进度按百分比与 MB 更新；返回后仍显示进度；完成后“已安装” | 完成并显示已安装（`6007c8ca`）；下载期间页面闪烁 FAIL | |
| 2.4 | 同样下载 C（约 199 MB）与 B（约 1010 MB，建议 Wi-Fi） | 确认框分别显示来源与许可；完成后“已安装” | 完成并显示已安装（`6007c8ca`）；下载期间页面闪烁 FAIL | |

### 2.2 中断、取消、重试

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.5 | B 下载到约 30% 时打开飞行模式 | 自动重试后显示“失败：…”（网络错误）；**不**显示为已安装；行内显示“已部分下载（N MB）” | | |
| 2.6 | 关闭飞行模式，再点“下载” | 进度从已下载处继续（不是从 0% 开始） | | |
| 2.7 | C 下载中途执行 `adb -s $vivo shell am force-stop $pkg`，重新打开设置页 | 显示“已部分下载（N MB）”；再点下载 → 从断点继续并完成 | | |
| 2.8 | 任一模型下载中点“取消” | 显示“已取消”；部分文件被清除（不再显示已部分下载）；此前已安装的版本仍可用 | | |
| 2.9 | 对某个“已部分下载”的模型点“丢弃未完成的下载” | 部分文件被删除 | | |

### 2.3 校验失败与空间不足

2.10 用 PC 上的假文件制造校验失败（不需要任何真实模型文件）：

```powershell
$d = "$env:TEMP\fake-model"; New-Item -ItemType Directory -Force $d | Out-Null
Set-Content -Path "$d\encoder.int8.onnx" -Value "not a model"   # A 的第一个文件，大小与哈希都不对
python -m http.server 8000 --directory $d                          # 保持运行
# 另一个窗口：
adb -s $vivo reverse tcp:8000 tcp:8000
```

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.10 | 先删除 A（若已安装）；A →“从其他地址下载…”→ 填 `http://127.0.0.1:8000` → 确认 | 确认框提示这是你填写的地址、仍按固定 SHA-256 校验；结果“失败：encoder.int8.onnx 与预期校验值不符”；A 不显示为已安装 | | |
| 2.11 | 同一对话框填 `http://127.0.0.1:8000/?x=1` 或 `ftp://…` | 提示地址无效，不开始下载 | | |
| 2.12 | （可选，会让手机存储暂时几乎占满）空间不足：见下方命令 | 下载前即提示“存储空间不足（约需 N MB）”；不留下文件 | | |

```powershell
# 2.12：先看可用空间（KB），再占用到只剩约 100 MB，下载 A（约需 168 MB + 余量）
adb -s $vivo shell df -k /data
$freeKb = <上一行 Available 列>
adb -s $vivo shell run-as $pkg fallocate -l "$([int64](($freeKb - 100*1024)*1024))" no_backup/fill
#   ……在 App 中下载 A，观察提示……
adb -s $vivo shell run-as $pkg rm no_backup/fill                    # 测完必须删除
adb -s $vivo reverse --remove-all
```

### 2.4 切换、识别、删除

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.13 | A 行 →“启用并用于语音输入”（当前服务 = 本地 · A）；麦克风与长按空格各说一句约 10 s 中文 | 提交最终文本；logcat 有 `Local ASR result`（记录 RTF、stop→final、PSS） | 识别可用：中文好、英文差（摘要，`7ed0fa78`）；时长、入口、指标未报告 | 同左 |
| 2.14 | 在“语音识别服务”中切换到本地 · C（需已启用），说 10 s、30 s、接近 60 s 的中英混说 | 流式识别；长语音无空结果；记录上述指标；有无标点 | 中英文均可（摘要）；时长、长语音、指标、标点未报告 | 同左 |
| 2.15 | 切换到本地 · B，说约 10 s 与约 40 s | 10 s 正常；约 34–39 s 以上可能为空结果（已知问题，照实记录） | 中英文均可（摘要）；约 40 s 未报告 | 同左 |
| 2.16 | 用 A 识别时（说话中）删除 A | 当前会话结束且不崩溃；之后当前服务显示“模型文件缺失”，**不**自动改选 | | |
| 2.17 | 旧 adb 副本：若手机上仍有 4B.3b 时推送的模型（`adb -s $vivo shell ls /sdcard/Android/data/$pkg/files/local-asr/`），在 App 中对该模型点“删除” | 该目录被删除（再次 `ls` 不再列出）；该模型显示“未安装” | | |
| 2.18 | 设置 → 高级 → 导出用户数据；解压查看 | 不含 `local-asr`、已安装模型、`no_backup` 下的凭据或 `voice/last-error` | | |

### 2.5 下一步：B 与 C 对比（两台手机，`7ed0fa78` 或更新）

目的：为 B/C 取得可比较的证据；**不据此自动选定正式/默认模型**（需另行决定）。A 已知英文差，可作为对照但不是重点。

1. **离线确认**：关闭 Wi-Fi 与移动数据（或开飞行模式后确认 Wi-Fi 未被重新打开）；设置页不显示下载进度；记录“已断网”。
2. 录好三段固定音频并在两台手机上复用同一内容（同一人、同一环境；建议用 PC 播放同一录音，或照同一文本朗读）：
   - S1 短中文约 5–10 s；
   - S2 中英混说约 10–15 s（含英文单词与数字）；
   - S3 约 40 s 连续中文（覆盖 B 已知的 34–39 s 空结果区间）。
3. 对 B、C 各跑 S1/S2/S3（入口：麦克风按钮；可再用长按空格跑一次 S1）。
4. 每次记录：设备、APK SHA、模型、片段、实际时长、最终文本（与参考文本比对的错字/漏字）、是否空结果、有无标点、logcat `Local ASR result` 中的 stop→final、RTF、PSS。
5. 断网状态下若任一识别失败，照实记录错误提示；结束后恢复网络。

| 片段 | 模型 | vivo：文本 / 空结果 / stop→final / RTF / PSS | Redmi：文本 / 空结果 / stop→final / RTF / PSS |
|---|---|---|---|
| S1 短中文 | B | | |
| S1 短中文 | C | | |
| S2 中英混说 | B | | |
| S2 中英混说 | C | | |
| S3 约 40 s | B | | |
| S3 约 40 s | C | | |

### 2.6 语音设置页简化（`f651a082`；设备测试用 `ca294467` 的 CI APK；设备检查待执行）

变更只涉及设置页的呈现与交互；服务选择语义、凭据加密、模型校验与下载逻辑不变（详见工作日志）。**从 `7ed0fa78` 覆盖安装，不清数据**，两台手机各做一遍。CI 只能证明编译与单元测试，不能证明版面或手机上的交互。

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 2.6.1 | 升级后打开语音设置 | 原有当前服务、已启用的服务、豆包等已保存凭据、已安装的 A/B/C 均保留；凭据行显示“已设置（密钥已隐藏）”，不显示任何密钥 | | |
| 2.6.2 | “当前使用” | 该行标题直接是所选服务名；无“语音识别服务”标签；可用时没有“可用”二字，只有“点按更换” | | |
| 2.6.3 | 点“当前使用” | 列表只含已启用且可用的服务：未填凭据的云端、未安装或未启用的模型、不可用的系统识别都不出现 | | |
| 2.6.4 | 停用当前服务（或删除当前模型） | “当前使用”仍显示原服务，并标“已停用”或“模型未安装”；不自动改选 | | |
| 2.6.5 | “系统自带” | 只有一行“Android 系统语音识别”，第二行为“音频处理方式由设备的语音服务决定，可能联网。”；未启用、已启用未授权、已启用可选、当前使用 四种状态文字不同 | | |
| 2.6.6 | 系统识别未授权时点该行 →“启用” | 立即弹出系统识别说明；允许 → 已启用且可选；不允许 → 已启用 · 未授权；之后可再“授权使用…”或“撤销授权”、“停用” | | |
| 2.6.7 | “本地 ASR（不联网）” | 分组名已改；A/B/C 各一行，无单独的开关行；已安装的模型开关在本行右侧；未安装的没有开关 | | |
| 2.6.8 | 点开关 / 点行其他位置 | 开关只改启用（不改当前服务）；点行打开操作列表（下载、从其他地址下载、导入、继续、删除、详情等按状态出现）；详情仍含来源、许可、大小与限制 | | |
| 2.6.9 | 下载 C 时停留并上下滚动 | 只有 C 行显示“安装中 · N% · 点按取消”；页面不闪烁、滚动位置不变；完成后显示“已安装 · 未启用” | | |
| 2.6.10 | 豆包/Qwen/腾讯开关 | 未填凭据：“需在下方填写凭据才能使用”，且不出现在“当前使用”列表中；无凭据时打开开关会弹出凭据表单 | | |
| 2.6.11 | 打开任一凭据表单 | 表单上方说明密钥只加密保存在本机、语音的实际接收方（火山引擎 openspeech.bytedance.com / 阿里云百炼所选地域 maas.aliyuncs.com / 腾讯云 asr.cloud.tencent.com）、不经本应用中转；小屏上表单可滚动到“确定” | | |
| 2.6.12 | 窄屏 / 大字体 | 标题与摘要换行完整，无截断或横向溢出；开关不遮挡文字；TalkBack 读出开关为“启用模型 A/B/C” | | |

仪器测试（需本机构建；不修改设置、凭据或模型）：`VoiceSettingsLayoutTest`，与 `VoiceSettingsProgressTest`、`ActionListDialogTest` 一起运行，命令同 §2.0c（class 列表加上 `org.fcitx.fcitx5.android.ui.main.settings.behavior.VoiceSettingsLayoutTest`），预期 `OK (8 tests)`。

## 3. Managed Cloud（需所有者凭据；无凭据则整节不可测）

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 3.1 | 豆包：填 API Key（或 App Key + Access Token），启用，选为当前，Mic 与长按空格各说一句 | 最终文本提交；logcat 有 `Doubao ASR final` | PASS（所有者报告，2026-09-28，`7af0cc16`；细节未另行记录） | |
| 3.2 | 重新打开豆包凭据 | 密钥字段为空、提示“已保存，留空则保持不变”；不回显 | | |
| 3.3 | Qwen：API Key + Workspace ID + 地域 + 模型（默认 qwen-audio-3.1-asr-flash-streaming），同上 | 最终文本提交；logcat `Qwen … final` | | |
| 3.4 | Qwen 填错 Key | 失败提示（InvalidApiKey 类）；未建立会话即失败 | | |
| 3.5 | Tencent：AppID + SecretId + SecretKey，引擎 16k_zh_en，同上 | 最终文本提交；logcat `Tencent … final` | | |
| 3.6 | Tencent 填错 SecretKey | 失败提示（4002 类鉴权失败） | | |
| 3.7 | 飞行模式下使用任一云端服务 | 早期失败并报告；**不**切换到其他云服务或系统识别（当前无正式 Local 模型，所以不回落） | | |
| 3.8 | 设备备份/恢复到另一台设备后 | 云端凭据显示“未设置”或会话失败提示重新填写（Keystore 密钥不随备份迁移） | | |
| 3.9 | 覆盖安装新 APK（不清数据）后查看豆包凭据 | 已保存的 API Key 仍在，无需重新填写 | PASS（所有者报告，升级到 `7ed0fa78`） | PASS（同左） |

## 4. Self-hosted（需服务器）

服务器准备（示例，均来自上游固定版本）：

- sherpa-onnx：`pip install sherpa-onnx==1.13.8`；`python python-api-examples/streaming_server.py --encoder … --decoder … --joiner … --tokens … --port 6006`（v1.13.8 源码）。
- FunASR 2-pass：FunASR v1.4.16 `runtime/python/websocket/funasr_wss_server.py --port 10095 --ngpu 0 --device cpu`（首次下载默认模型约 1.8 GB）。
- Fun-ASR-Nano：`funasr-realtime-server`（需 NVIDIA GPU / vLLM）。
- OpenAI-compatible：FunASR v1.4.16 `funasr-server --model sensevoice --device cpu --port 9000`，端点 `http://…:9000/v1/audio/transcriptions`。
- release 构建需要 `wss://`：在服务器前放置带有效证书的反向代理（如 Caddy/nginx），可选 Bearer 鉴权。

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 4.1 | debug APK 添加 sherpa-onnx 实例 `ws://<局域网IP>:6006`，启用，选为当前，说话 | 最终文本提交；英文词间保留空格 | | |
| 4.2 | 同上，FunASR 2-pass `ws://…:10095` | 最终文本带标点（第二遍修正） | | |
| 4.3 | 同上，Fun-ASR-Nano | 最终文本提交（需 GPU 服务器） | | |
| 4.4 | release 构建中添加 `ws://` 实例 | 保存被拒绝：“不允许未加密的 ws:// 地址” | | |
| 4.5 | 反向代理要求 Bearer，实例不填令牌 | 早期失败（HTTP 401），不切换服务 | | |
| 4.6b | 同上，OpenAI-compatible 实例 `http://…:9000/v1/audio/transcriptions` | 说话期间无实时文字；停止后约数秒提交最终文本 | | |
| 4.6 | 删除当前实例 | 当前服务显示“已删除”，不改选 | | |

## 5. fallback（D035）

当前没有正式 Local 模型，A/B 为研究模型（D037），因此**预期不会发生任何自动回落**：

| # | 步骤 | 预期 | vivo | Redmi |
|---|---|---|---|---|
| 5.1 | 当前 = 云端/自建，安装并启用 B，制造早期失败（飞行模式/错误地址） | 报告失败；**不**回落到 B（研究模型不是 fallback 目标）；不回落到 System | | |

## 6. 记录要求

记录设备、APK 的实际 SHA、步骤、按钮可见性、实际结果与 logcat 关键行；不可测写明原因。
