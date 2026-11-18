# 实现状态：Phase 04（MOSS-TTS Provider 与流式音频播放）/ Phase 04D（Qwen3-TTS GGML Provider）

> 更新时间：2026-11-18
> 环境：Windows 10 专业版 10.0.19045 / Python 3.12.12 / PySide6 6.12.0 / httpx 0.28.1 /
> numpy 2.5.5 / sounddevice 0.5.6 / uv 0.11.7 / RTX 4090 24 GB
> 任务书：`00_项目总纲.md` + `04_任务书_MOSS_TTS与音频播放.md` + `04D_任务书_Qwen3-TTS_GGML_Provider正式接入.md`
> （Phase 01–03 已合并，作为基线）
> MOSS 服务：本机 `F:\AI_Project\MOSS-TTS`（上游 `moss_tts_realtime/fast_api.py`，未改上游代码）
> Qwen 服务：本机 `F:\AI_Project\Qwen3-TTS-Experiments`（qwentts.cpp `2edb35f`，`build-cuda/Release/tts-server.exe`）
>
> 结论：**任务书 04 §3 功能要求 10/10 落地、§6 验收 6/7 达标 + 1 项部分达标**；
> **04D 已接入 Qwen3-TTS 主候选后端**（Provider/配置/工厂/装配/测试/真实服务验证/文档），
> 判定沿用 04C 的 **CONDITIONAL GO**：性能/显存/取消/稳定性达标，**中文音色人工试听、用户中文参考音频、
> 情绪稳定控制仍 Pending**。本节 §11 记录 04D；**§12 记录 04E**（GPU 调度 + Qwen 运行管理，
> 默认 `llm_priority`、`overlap` 实验、`sentence_interleave` 判定 unsupported，真实 4090 验收完成）。
> **Phase 04A（2026-11-18）追加诊断**：实时播放断续的主因是 MOSS 生产速度（RTF 1.10–1.22）
> 与突发节奏（块间隔 P99 0.58–0.69 s），不是播放器；并发 LLM decode 时 RTF≈3.1。
> 详见 `docs/phase04a-realtime-playback-diagnosis.md`；结论与修复见本文 §10。
> 部分达标的是 §6.6 "10 组固定中文样例的手工试听记录"：10 组 WAV 与性能数据已生成
> （`docs/tts-samples/`），但**主观试听必须由哥哥本人完成**（试听记录表已留空列，见 §7.1）。
> §6.7 的 4090 显存测量已完成（含 LLM+TTS 并发峰值），并给出"27B 与 TTS 不能共存"的实测结论。

## 1. 完成项（对照任务书 04 §3 / §4 / §6）

### 1.1 §3 功能要求

| 要求 | 状态 | 实现/证据 |
| --- | --- | --- |
| 独立 `MossTTSProvider`，主进程不加载模型权重 | ✅ | `providers/moss_tts.py` 只走 HTTP（`/health`、`/tts/session/*`），无 torch/transformers 依赖；`import` 里没有权重加载代码 |
| `SpeechCoordinator`：按标点/最小长度/最大等待/句子完整性切分，不按 token 调 TTS | ✅ | `speech/segmenter.py`（强/弱边界、首段更短、`max_chars` 硬切、`flush`）+ `speech/coordinator.py`（`max_wait_ms` 定时 flush，`feed_text` 同步非阻塞）；测试 22 项 + 30 项 |
| 顺序、序号与 request/session 关联；上一轮迟到音频不得插入新一轮 | ✅ | 一轮一个 utterance/服务端会话；`_is_current(turn)` + `epoch` 丢弃迟到块并计数；`test_barge_in_*`、provider 用新 uuid 会话 |
| 缓冲、播放队列、静音、停止、清空待播、取消当前生成 | ✅ | `speech/player.py`（线程安全队列、背压、硬上限、`stop(clear=True)`、静音仍消费）；`cancel()` 停 pump+reader、关会话、清队列 |
| 音频格式显式（采样率/声道/格式/块时长），不隐式猜 | ✅ | `providers/tts_base.py::AudioFormat`（校验帧对齐、`as_metadata()`）；MOSS 响应头逐一核对，不一致 → `tts_invalid_format`；`/audio` 实测 `pcm_s16le / 24000 / mono` |
| 低延迟输出：`sounddevice` callback/stream，不为每块建线程/播放器 | ✅ | 单条 `OutputStream` + 回调（`SpeechPlayer` 只在首块时打开、播完即关）；块大小 480 帧（20 ms），回调只取队列；**设备打开在工作线程 + 5 s 超时**（见 §7.4） |
| 欠载/超载、断连、无效帧、设备切换、服务重启 | ✅（有边界） | 欠载计数、硬上限丢最旧块、`end_utterance` 空轮兜底、Sink `on_broken` → 重开默认设备一次 → null 播放；服务端断连 → `tts_stream_interrupted` / 会话失效 → `tts_session_lost`；**设备插拔只在代码路径与假 Sink 上验证**（§7.3） |
| TTS 失败不影响文字；UI 显示静音/服务错误 | ✅ | `speech.error(degraded=True)` 只更新语音行；`speech_smoke.py` 与 `test_orchestrator` 覆盖；UI 头部「语音：未启用/合成中…/播放中…/静音/不可用」 |
| 可配置音色与风格/情绪映射，只透传真实支持的字段 | ✅（如实：无风格字段） | `providers.tts.voice` → 服务端 `prompt_audio`；MOSS-TTS-Realtime **没有文本风格/情绪参数**，文档与设置页都写明"换音色=换参考音频" |

### 1.2 §4 并发与显存

| 要求 | 状态 | 证据 |
| --- | --- | --- |
| 不假设生成与播放同阶段；分别记录 | ✅ | `speech.synthesis.started` 与 `speech.playback.started/completed` 分开；`SpeechMetrics` 分开 `synthesize_ms` / `audio_ms` |
| 测量 TTS 单独 / LLM 单独 / LLM+TTS 并发峰值 | ✅ | `scripts/gpu_mem_probe.py` + `docs/gpu-mem-probe.json`；手工分阶段数据见 `docs/moss-tts-probe.md` §5 |
| TTFA/TTFB、RTF、缓冲、队列等待统一口径 | ✅ | `docs/ARCHITECTURE.md` §14；`docs/moss-tts-probe.md` §3/§6；RTF=合成/音频（>1 慢于实时） |
| 不自动卸载 LLM / 清 CUDA cache，先给配置与观测 | ✅ | 无任何卸载/清缓存代码；`providers.tts.type`、`speech.*`、`audio.*` 全部可配；显存只测量与记录 |

### 1.3 §6 验收

| # | 验收标准 | 状态 | 证据 |
| --- | --- | --- | --- |
| 1 | 真实 MOSS 健康检查并生成语音 | ✅ | `scripts/tts_probe.py`：health=True（`MOSS-TTS · sr=24000 · cuda:0 · sdpa`）、4 次合成全部 OK、WAV 落盘；`docs/moss-tts-probe.json` |
| 2 | 文字先于语音完整/增量显示；TTS 不阻塞 UI/LLM | ✅ | `scripts/speech_smoke.py`（真实 MOSS + 完整 UI 链）：首个文本 delta 243.6 ms，首个音频播放 4471.6 ms（差 4.2 s），状态 `idle→synthesizing→playing→idle`；文本完成事件先于播报（Orchestrator 测试断言） |
| 3 | 多语义片段连续播放，无错序/明显重叠/串轮 | ✅（预缓冲后） | 10 组样例各切 3–6 段；2.5 s 预缓冲后 8/10 句 underrun=0，长句仍有欠载（§7.2）；`test_speech_coordinator` 断言段序、monotonic segment_index |
| 4 | Stop/取消终止生成与待播 | ✅ | `test_cancel_stops_speech_before_cancelling_turn`、`test_barge_in_during_playback_*`、`test_speech_cancelled_exactly_once` |
| 5 | 服务不可用时文字仍可用 | ✅ | `test_speech_degraded_does_not_switch_turn_to_error`、`test_tts_unhealthy_is_reported_without_blocking_startup`；降级后状态回 idle、无 `system.error` |
| 6 | ≥10 组固定中文样例的**手工试听记录** + 性能数据 | ⚠️ 部分 | 10 组 WAV + `samples.json` + `record.md`（性能列已填、试听列留空）；主观试听需哥哥本人（§7.1） |
| 7 | 4090 实际显存（LLM/TTS/并发），不得用其他 GPU | ✅ | 并发峰值 21.4–22.1 GB；27B+TTS 实测掉到 12.5 tok/s（不可用）；`docs/moss-tts-probe.md` §5 |

## 2. 新增/修改文件

### 新增（Phase 04）

```text
src/desktop_companion/providers/tts_base.py     语音 Provider 协议/AudioFormat/AudioChunk/SpeechMetrics
src/desktop_companion/providers/moss_tts.py     MOSS-TTS-Realtime HTTP 适配（会话/推文本/收 PCM/错误翻译）
src/desktop_companion/providers/mock_tts.py     零网络 Mock TTS（正弦/静音，测试与演示）
src/desktop_companion/providers/tts_factory.py  type → Provider（none/mock/moss_tts）
src/desktop_companion/speech/__init__.py        语音层说明
src/desktop_companion/speech/segmenter.py       流式文本切分（标点/长度/等待/清洗）
src/desktop_companion/speech/player.py          单流回调播放器（队列/背压/静音/录音/降级）
src/desktop_companion/speech/coordinator.py     一轮播报的编排（段队列/pump/事件/取消/降级）
tests/fake_moss_tts_server.py                   真 asyncio TCP 假 MOSS 服务（测试基础设施）
tests/test_moss_tts_provider.py                 Provider 契约测试（56 项）
tests/test_mock_tts.py                          Mock/工厂测试（23 项）
tests/test_tts_segmenter.py                     切分器测试（22 项）
tests/test_audio_player.py                      播放器测试（19 项，含 2 项预缓冲）
tests/test_speech_coordinator.py                协调器测试（30 项）
scripts/tts_probe.py                            真实 MOSS 诊断/延迟/RTF/取消
scripts/tts_samples.py                          10 组中文样例 + WAV + 试听记录表
scripts/speech_smoke.py                         完整语音链路冒烟（offscreen Qt）
scripts/gpu_mem_probe.py                        TTS/LLM/并发显存与吞吐测量
docs/moss-tts-probe.md / docs/moss-tts-probe.json    真实服务验证与显存结论
docs/gpu-mem-probe.json / docs/gpu-mem-probe.md      显存测量脚本产物
docs/tts-samples/                               10 个 WAV + samples.json + record.md
.dsh/notes/phase04-contract.md                  本阶段实现/测试契约（冻结版）
```

### 修改

| 文件 | 改动 |
| --- | --- |
| `core/events.py` | 新增 `speech.state.changed / synthesis.started / audio.chunk / playback.started / playback.completed / cancelled / error` 与 `SpeechState`；payload 只带元数据 |
| `core/config.py` | 新增 `providers.tts`、`providers.moss_tts`、`speech`、`audio` 配置（全部 `extra="forbid"`）；`AudioConfig.start_prebuffer_ms`、设备索引字符串归一 |
| `app/orchestrator.py` | 语音阶段：文本 delta 同步喂语音层；`completed` 事件后 `ready → 播报 → idle`；取消/barge-in 先停播报；TTS 健康与 LLM 并发检查；`set_muted` |
| `app/bootstrap.py` | 装配 TTS Provider + AudioPlayer + SpeechCoordinator；关闭顺序加入 `speech.aclose`；启动预算按两者较大 health 超时 |
| `ui/ui_state.py` | 语音状态/静音/tts 健康（与模型健康分开）；`speech_error` 不占用文字错误横幅 |
| `ui/main_window.py` | 头部「语音：…」+ 静音按钮；语音 tooltip；静音状态由事件回传 |
| `ui/settings_dialog.py` | 静音改为真实生效；显示音色来源；语音不可用时禁用并说明 |
| `ui/app_shell.py` | 静音偏好的挂载/持久化/与面板/设置页同步 |
| `ui/qt_event_bridge.py` | `set_muted()` → asyncio 线程 |
| `ui/window_state.py` | 持久化 `speech_muted` |
| `config/default.yaml` | TTS/语音/音频默认值与注释（默认 `tts.type=none`） |
| `pyproject.toml` / `uv.lock` | 新增 `numpy`、`sounddevice` |
| `main.py` / `__init__.py` | 版本 0.4.0、CLI/描述更新 |
| `tests/*` | `helpers.make_config` 支持 TTS/语音；config/events/orchestrator/ui_state/app_shell/architecture 增量断言 |
| `README.md` / `docs/ARCHITECTURE.md` / `AGENTS.local.md` / `.dsh/notes/*` | Phase 04 同步 |

## 3. 依赖版本

| 包 | 版本 | 用途 |
| --- | --- | --- |
| Python | 3.12.12 | 运行环境 |
| PySide6 | 6.12.0 | UI（不变） |
| pydantic | 2.14.3 | 配置/事件/音频类型 |
| httpx | 0.28.1 | MOSS HTTP（`trust_env=False`） |
| **numpy** | **2.5.5** | PCM 缓冲/静音填充（播放回调） |
| **sounddevice** | **0.5.6** | PortAudio 输出流（自带 DLL） |
| pytest / pytest-asyncio / ruff | 8.4.2 / 1.5.0 / 0.17.5 | 测试与静态检查 |

外部服务：MOSS-TTS-Realtime（torch 2.9.1+cu128 / torchcodec 0.8.1 / transformers 5.0.0，sdpa），
由用户启动，不属于本项目依赖。

## 4. 运行过的命令与结果

| 命令 | 结果 |
| --- | --- |
| `uv run pytest` | **598 passed in 22.65s**（Phase 03 基线 418 + Phase 04 新增 175 + Phase 04A 新增 5） |
| `uv run ruff check` | All checks passed |
| `uv run ruff format --check` | 87 files already formatted |
| `uv run desktop-companion --check` | 退出码 0；`providers.tts/moss_tts/speech/audio` 全量默认值与来源链正确 |
| `uv run python scripts/tts_probe.py --runs 3 --json docs/moss-tts-probe.json` | 退出码 0；health + 3 次单段 + 三段增量全部 OK；TTFB 2.73–3.27 s / RTF 1.21–1.29（三段 TTFB 1.37 s） |
| `uv run python scripts/tts_samples.py --play --prebuffer-ms 2500` | 退出码 0；10/10 成功；8/10 句 underrun=0，长句 17/50 次；WAV 落 `docs/tts-samples/` |
| `uv run python scripts/speech_smoke.py --tts-type moss_tts --expect-speech` | 退出码 0；`idle→synthesizing→playing→idle`；首文本 243.6 ms / 首音频 4471.6 ms；`leftover_tasks=()` |
| `uv run python scripts/gpu_mem_probe.py` | 退出码 0；TTS 单独 14.0 GB 峰值、LLM 单独 22.1 GB（TTS 常驻）、并发峰值 22.1 GB；`docs/gpu-mem-probe.json` |
| 手工分阶段显存（停/起服务） | TTS 单独 Δ≈12.0 GB；9B 单独 Δ≈7.3 GB；27B(8K ctx)+TTS 峰值 23.9 GB 且 LLM 12.5 tok/s → 不可用 |

真实服务关键输出（摘自 `docs/moss-tts-probe.json`）：

```text
health: healthy=True detail="MOSS-TTS · sr=24000 · cuda:0 · sdpa" (28.9 ms)
single_1: audio 12.96s / 152 chunks / ttfb 3272.6ms / synth 16650.9ms / rtf 1.28
single_2: audio 14.64s / 172 chunks / ttfb 2740.2ms / synth 17742.0ms / rtf 1.21
single_3: audio 9.76s  / 115 chunks / ttfb 2733.3ms / synth 12449.8ms / rtf 1.28
multi_segment: audio 10.32s / 121 chunks / ttfb 1368.4ms / synth 13340.9ms / rtf 1.29
format: 24000 Hz / mono / s16le（与 expected 一致）
```

## 5. 测试结果明细

- 通过：**598 / 598**；失败 0；跳过 0（Phase 04A 新增 hold/probe/config 5 项）。
- Phase 04 新增 175 项：

| 组 | 数量 | 文件 |
| --- | --- | --- |
| TTS Provider / Mock / 工厂 / 假服务器 | 79 | test_moss_tts_provider 56 / test_mock_tts 23 |
| 文本切分 | 22 | test_tts_segmenter |
| 音频播放器 | 19 | test_audio_player（含 2 项预缓冲） |
| 语音协调器 | 30 | test_speech_coordinator |
| 事件 / 配置 / UI / Orchestrator / 架构增量 | 25 | test_events +2 / test_config +7 / test_ui_state +3 / test_orchestrator +6 / test_app_shell +2 / test_architecture +1 / test_ui_widgets 改写 |

覆盖要点：真实 TCP 假 MOSS 服务（块大小/首字节延迟/错误码/错格式/断连/不回应/文本顺序记录）、
帧对齐与半帧拼接、格式头校验、TTL 健康缓存、取消隔离、切分边界（英文标点/连续标点/超长/代码块/
清洗）、背压/硬上限/欠载/静音/停止/设备降级/录音、段序与尾段不丢、barge-in 迟到块丢弃、
TTS 降级不影响文字、静音状态事件、`emit_chunk_events` 开关、配置校验、UI 语音行与静音持久化。

未由自动化覆盖、但已人工执行的验证：

- 真实 MOSS 服务的健康/合成/三段增量/取消复用（`scripts/tts_probe.py`，见 §4）；
- 真实 MOSS + 完整 UI 链路（`scripts/speech_smoke.py`，offscreen Qt，退出无残留）；
- 10 组中文样例的真实播放（sounddevice 到系统默认设备）与 underrun 统计。

## 6. 未完成项（刻意范围控制 / 后续阶段）

- **人工主观试听**：10 组 WAV 已生成，试听结论列留给哥哥填（§7.1）；
- STT/VAD、麦克风（任务书 05）；
- 工具执行/权限/审批（任务书 07）、线上强模型（任务书 08）、DSH/Codex Adapter（任务书 09）；
- VLM/截图/长期记忆（任务书 10/11）、任务中心 `task.*`（任务书 12）；
- llama-server / MOSS 的启动停止与显存预算（任务书 13）——本阶段只连接已运行的服务；
- 多会话并发播报、全双工打断（任务书 14/后续）。

## 7. 已知问题与风险

1. **手工试听未完成（验收 §6.6 部分达标）**：AI 无法判断音色/断句/吞字的主观质量。
   `docs/tts-samples/record.md` 的"试听结论"列留空，请哥哥逐条听并填写；
   WAV 是**实际播放**的 PCM（24 kHz mono），与设备听到的一致。
2. **RTF 1.10–1.22（本机 sdpa+eager，无 triton）**：合成比实时慢 10–22%，且音频块以
   ~0.5 s 一批、批间隔 P99 0.58–0.69 s 的方式到达；0 ms 预缓冲时每句 1–200 次 underrun。
   `audio.start_prebuffer_ms=2500` 能让 ≤12 s 的句子欠载归零，更长句子仍会欠载——
   需要的预缓冲 ≈ (RTF−1)×音频时长 + 一个批间隔（04A 实测口径）。
   **并发 LLM decode 时 RTF 恶化到 3.1**（块间隔 max 2.9 s），任何有限缓冲都盖不住；
   根治是给 TTS 提速（`triton-windows` + `MOSS_ENABLE_COMPILE=1`）、TTS/LLM 分卡，
   或长回复改用"整轮合成后播放"（延迟 ~13–16 s / 11 s 音频）。
3. **设备路径只做了代码级验证**：真实声卡播放（默认设备 WH-1000XM4）跑通且能落 WAV；
   但"播放中拔掉设备/切换默认设备/蓝牙休眠"只在假 Sink 与超时降级路径上验证过（§7.4 的 5 s 超时
   即为此加）。真实插拔仍建议实机确认。
4. **设备打开可能阻塞**：本机实测，默认输出是睡眠中的蓝牙设备时 `sd.OutputStream.start()`
   可能长时间阻塞且不释放 GIL（会冻住整个事件循环）。现已改为**工作线程 + 5 s 超时**，
   超时降级为 null 播放（时序照常、无声），并在日志里 warning；但这意味着"极端情况下首播无声"，
   需要用户在设置里选设备或等设备唤醒。
5. **两次 `app.quit()` 不退出**（离屏脚本）：PySide6 下在 QTimer 槽里调用 `app.quit()` 偶发不退出
   事件循环（实测约 1/5）。`scripts/speech_smoke.py` 改为 `processEvents()` 泵循环，稳定；
   应用主窗口走正常关闭路径，不受影响。
6. **MOSS 冷启动首请求慢**：空闲后首个 TTFB 5.7 s，取消后重开曾达 21.9 s；稳态 1.4–3.3 s。
   UI 用"合成中…"状态吸收，但首播延迟可能明显。
7. **服务端 batch=1 / 采样参数硬编码**：两轮并发会在服务端排队；风格/情绪无法按请求调整，
   只能换参考音频。这是上游实现能力，不是本项目可修的参数。
8. **`services.tts_base_url` 仍是 Phase 01 占位**：真实地址在 `providers.tts.base_url`
   （与 Phase 03 的 `services.llm_base_url` 情况相同，README 已注明）。
9. **TTS 事件里的 `first_audio_latency_ms` 口径**：目前是"首段切出 → 首块音频入队"，
   不是"设备输出第一帧"（设备在首块入队时才打开）。播放器已暴露 `played_ms/underruns`，
   更精确的设备首帧延迟需要回调接线（Phase 04 收口时未改，避免重复事件）。
10. **`speech.audio.chunk` 默认不发事件**（`emit_chunk_events=false`）：10 块/秒会淹没总线；
    需要逐块诊断时打开该开关。

## 8. 对下一阶段的接口建议

- **任务书 05（STT）**：转写文本回填输入框即可走同一条链；播报与录音的互斥（说话时暂停 TTS）
  建议放在 Orchestrator，而不是语音层；`UserMessage.source="voice"` 已预留。
- **任务书 07（Tool Broker）**：语音层只消费文本，不感知工具；工具活动摘要可以复用
  `speech.state.changed` 之外的独立事件，不要把工具日志塞进 `speech.*`。
- **任务书 12（任务中心）**：`UiState.task_active` 已经能驱动桌宠；语音播报与后台任务并行时，
  建议"任务完成摘要"走一次普通播报（同样的 `SpeechCoordinator` 入口），不要另开口播通道。
- **任务书 13（生命周期）**：`MossTTSProvider.health_check()` 可直接做健康监控探针；
  启动/停止 MOSS 与显存预算由 Runtime Manager 负责，Provider 不碰进程；
  `docs/moss-tts-probe.md` §5 可作为显存预算的实测基线（9B+TTS ≈ 22 GB，27B+TTS 不可行）。
- **通用**：新增 `speech.*` 事件属于协议变更，先改 `docs/ARCHITECTURE.md` 与
  `.dsh/notes/phase04-contract.md`；音频 PCM 永远不进事件总线；每个阶段保持
  `pytest` + `ruff check` + `ruff format --check` 全绿。

## 9. 审查与收口记录（2026-11-18）

- 实现分工：Lead 冻结契约（`providers/tts_base.py`、`speech.*` 事件、配置骨架）并做
  Orchestrator/UI/文档/真实服务验证；三个并行实现者分别负责 Provider 层、播放器/切分器、
  SpeechCoordinator（各自只写自己的文件，不做 git 提交）。共享任务板见会话记录。
- Lead 在集成时修复了三类问题：① coordinated 契约缺口（`provider_name`/`health_check` 由实现者补齐）；
  ② 协调器不拥有播放器生命周期 → `start()`/`aclose()` 现在成对启停播放器；
  ③ `AudioPlayer` 缺 `speech_config` → 背压水位不生效（bootstrap 已传入）。
- 收口时新增：`audio.start_prebuffer_ms`（+2 测试）、设备打开工作线程 + 5 s 超时、
  `speech_smoke.py` 的 processEvents 泵循环（绕开 PySide6 偶发不退出）。
- 审查未覆盖：真实设备插拔/切换、真实双屏与 DPI（承袭 Phase 02 限制）、MOSS 开编译后的性能。


## 10. Phase 04A：实时播放断续诊断与修复（2026-11-18）

> 任务书：`04A_任务书_MOSS实时播放断续诊断与修复.md`；完整报告：
> `docs/phase04a-realtime-playback-diagnosis.md`；原始数据：`docs/diagnostics/`。

### 10.1 结论

- **主因**：MOSS 到达侧 RTF 1.10–1.22 + 突发块节奏（~0.5 s/批、批间隔 P99 0.58–0.69 s、
  max 1.1 s）。WAV 连贯只说明"内容"无边界异常，不证明"到达及时"。
- **播放器不是根因**：Mock TTS 以 0.22–0.94× 实时推送时 **0 underrun**；1.5×→103 次、
  2.3×→248 次，欠载随生产缺口单调增加；回调帧恒 480、无 null 降级、设备暖启动 25–34 ms。
- **并发是最坏场景**：9B LLM decode 与 TTS 同时用 GPU 时到达 RTF **3.13–3.16**、
  块间隔 max 2.84–2.88 s、静音 21.7–27.8 s/12–15 s 音频 → 流式播报必然大面积断续。
- **客户端修复是否足够**：不够。本轮加入 `audio.underrun_hold_ms`（欠载后等回填再续播），
  实测把 44–59 次碎响合并成 **1 次停顿**（总静音相近）；但 RTF>1 时无法保证长句连续，
  只能靠"整轮合成后播放"（实测 0 underrun、首帧 13.4–16.1 s）或 TTS 提速/分卡。

### 10.2 本轮代码与配置变更

| 文件 | 变更 |
| --- | --- |
| `core/config.py` | 新增 `audio.underrun_hold_ms`（默认 0=旧行为；建议 1000–1500） |
| `speech/player.py` | 欠载后 hold（不消费、输出静音、回填到阈值再续播）；`PlaybackReport` 增加 `pause_episodes`/`hold_silence_ms`；`diagnostics()` 暴露 `holding`；新增可选 `probe=` 诊断钩子（默认零开销） |
| `tests/test_audio_player.py` | +3 项（hold 生效/hold=0 回退/probe 钩子） |
| `tests/test_config.py` | +2 项（播放策略默认值与覆盖） |
| `scripts/playback_diagnosis.py` | 新增诊断脚本（矩阵/对照/utterance/并发，JSONL+summary） |
| `docs/diagnostics/`、`docs/phase04a-...md` | 原始证据与报告 |
| `config/default.yaml` / `README.md` / `docs/ARCHITECTURE.md` | 新增字段与口径说明 |

**未做**：不提高默认预缓冲、不插静音掩盖、不降采样/丢音频、不改 MOSS 上游、不改事件协议、
不动 Coordinator 的取消/降级语义。

### 10.3 当前实时播放是否达标

| 场景 | 结论 |
| --- | --- |
| 短句（≤6 s 音频）、预缓冲 ≥ 1 s | 基本可用（欠载 0–45 次，长尾仍可能碎响） |
| 中等/长句（12–15 s）、预缓冲 2.5 s | 可连续（实测 0 underrun），但首播多等 2.5 s |
| 长句（>15 s）或并发 LLM decode | **不达标**：必然出现 1 次到多次停顿；客户端无法根治 |

建议把 MOSS 当**临时/实验 TTS 后端**；正式实时对话前先做 TTS 提速或分卡（见报告 §8）。

### 10.4 收口决定（2026-11-18）

- MOSS Provider **保留在仓库**，代码与配置注释标记 **experimental**；创建实例时记一条 warning。
- `providers.tts.type` 默认保持 `none`，**不再把 MOSS 作为默认服务启动**；`mock` 可用于验证播放链路。
- 诊断脚本 `scripts/playback_diagnosis.py` 与可选 `audio.underrun_hold_ms` 保留；
  不再继续优化 MOSS 播放器（根因在模型侧，客户端已到边界）。

## 11. Phase 04D：Qwen3-TTS GGML Provider 正式接入（2026-11-18）

> 任务书：`04D_任务书_Qwen3-TTS_GGML_Provider正式接入.md`；前置 04C 决策：**CONDITIONAL GO**。
> 完整说明：`docs/qwen3-tts-provider.md`；证据：`docs/qwen3-tts-probe.json`、
> `docs/tts-samples-qwen/`、`docs/qwen3-tts-vram.json`；契约：`.dsh/notes/phase04d-contract.md`。

### 11.1 完成项（对照任务书 04D §2/§3）

| 要求 | 状态 | 实现/证据 |
| --- | --- | --- |
| 新增独立 `Qwen3TTSProvider`，走本地 HTTP | ✅ | `providers/qwen3_tts.py`；只依赖 httpx + core，主进程不加载权重 |
| 注册到 factory/registry，MOSS 保留 experimental | ✅ | `tts_factory.py` 新增 `qwen3_tts` 分支；MOSS 代码/标记/warning 不变 |
| 复用 `SpeechCoordinator`/`SpeechPlayer`/`AudioFormat`/取消/错误协议 | ✅ | 未改这些模块与 `speech.*` 事件；Qwen 只是新的 `SpeechProvider` 实现 |
| 每段一次完整请求，不传半句/token | ✅ | `open_stream(initial_text)` 预发首段，`push_text` 每段一次 `POST /v1/audio/speech`；测试断言"3 段 = 3 请求"且顺序一致 |
| 异步读真实流式音频，按到达顺序产生 chunk，跨段不串轮 | ✅ | `audio()` 逐段串行 + `AudioChunk.index` 跨段单调；多段样例与 10 组样例全部顺序正确 |
| 采样率/声道/格式显式解析与验证 | ✅ | 服务端**没有**格式响应头 → `expected_*` + 健康检查 WAV 探针（24 kHz/mono/16 bit）；不一致判不健康 |
| 末块/空流/半帧/异常结束/服务错误/超时/断连 | ✅ | 半帧拼接、末尾残帧丢弃、空段 warning 继续、400/502/503 映射、三段超时、断连 → `tts_stream_interrupted` |
| 首字节/首个完整帧/首帧入播放器分开记录 | ✅ | `stream.stats.first_byte_ms` / `first_chunk_ms`；播放器侧用 `speech.playback.started`（Provider 不猜） |
| 声音/克隆参数只在真实支持时发送；Base instruct 拒绝 | ✅ | `voice`/`language`/`instructions` 仅在配置时出现；`model_type=base` 开流前拒绝 instructions；400 兜底映射 |
| 声音素材只引用授权路径 + 路径规范化 | ✅ | `scripts/qwen_voice_register.py`（`Path.resolve`、存在性、`.wav` 限制、时长/采样率/声道检查、dry-run） |
| 取消 = 客户端断连，确认服务端真停 | ✅ | 读到 2–8 块后断开：服务端日志 `on_chunk callback aborted the synthesis`；GPU 95%→3% 约 0.5 s；下一请求正常 |
| 迟到块丢弃、request_id 绑定 | ✅ | 沿用 coordinator 的 utterance/epoch 过滤；Provider 每段请求独立，`audio()` 结束即停 |
| TTS 故障不影响文字 / degraded | ✅ | ProviderFailure → coordinator `speech.error(degraded=True)`；Provider 测试与既有集成测试覆盖 |
| 服务端 batch=1 → 显式排队/忙碌状态 | ✅ | Provider 级 `asyncio.Lock`；等锁超时 → `tts_busy`（retryable），并发两路测试断言 |
| 队列上限/背压 | ✅ | 沿用 `SpeechCoordinator`+`AudioPlayer` 的背压/硬上限（未改） |

### 11.2 验收对照（任务书 04D §6）

| # | 验收标准 | 状态 | 证据 |
| --- | --- | --- | --- |
| 1 | 1.7B Q8_0 通过真实 Provider 生成、播放和取消 | ✅ | `tts_probe --tts qwen` 单段×3/多段×3/取消+复测全 OK；10 组样例真实播放（null sink 实时消费） |
| 2 | 暖机 TTFA ≤800 ms、RTF_project ≤0.7 | ✅ | TTFA **14.3–16.1 ms**（健康检查含 WAV 探针 27.3 ms）、RTF **0.09–0.10** |
| 3 | 常驻 ≤4 GB、峰值 ≤5 GB | ✅ | 常驻 **+3,105 MiB（≈3.03 GB）**、峰值 **+3,211 MiB（≈3.14 GB）**（基线 1,848 MiB） |
| 4 | 15 s 以上、10–20 句连读无欠载/爆音/错序/串轮 | ✅ | 15 句 ≈109.4 s 音频：`underruns=0`、`dropped=0`；10 组样例（4.3–13.3 s）全部 0 欠载 |
| 5 | 200–300 ms 初始预缓冲有实测依据、按 Provider 独立 | ✅ | `providers.tts.start_prebuffer_ms: 300`（Qwen）与全局 `audio.start_prebuffer_ms` 分离，MOSS 仍可 2500；用真实 `AudioPlayer` 验证（0 欠载） |
| 6 | 取消是真停生成，下一请求可执行 | ✅ | §11.1 取消行；`cancel_ms=34.3`（客户端关闭耗时）、GPU 回落约 0.5 s、after_cancel TTFA 97.5 ms |
| 7 | MOSS 仍可在 experimental 配置下使用，旧测试不回归 | ✅ | MOSS 代码未改；全量 683 项测试通过（含 MOSS/Mock/播放器/协调器） |
| 8 | 默认是否启用 Qwen 由用户确认；未确认前保持安全默认 | ⏳ | `providers.tts.type` 仍是 `none`；04D 合并后由用户显式改成 `qwen3_tts`（**未静默改默认**） |

### 11.3 新增/修改文件

```text
新增  src/desktop_companion/providers/qwen3_tts.py     Qwen3TTSProvider + QwenStreamStats/QwenRequestStats
新增  tests/fake_qwen_tts_server.py                    真 asyncio TCP 假 qwen3-tts（pcm chunked / wav RIFF / 错误注入）
新增  tests/test_qwen3_tts_provider.py                 Provider 契约测试 72 项
新增  scripts/qwen_voice_register.py                   Base 模型克隆音色注册（路径/时长/采样率检查 + dry-run）
新增  docs/qwen3-tts-provider.md                        固定 commit/启动/DLL/API/PCM/voice/取消/实测/限制
新增  docs/qwen3-tts-probe.json                        真实服务探针（health/单段×3/多段/取消/复测）
新增  docs/qwen3-tts-vram.json                         显存基线/常驻/峰值增量
新增  docs/tts-samples-qwen/                           10 组中文样例 WAV + samples.json + record.md
新增  .dsh/notes/phase04d-contract.md                  04D 契约（实现/测试共同基准）
改动  core/config.py                                   + Qwen3TTSConfig；tts.type/json/language/start_prebuffer_ms；校验
改动  providers/tts_factory.py                         + qwen3_tts 分支
改动  app/bootstrap.py                                 startup_timeout_for 计入 qwen 健康超时；audio_config_for 按后端覆盖预缓冲
改动  config/default.yaml                              TTS 注释、language/start_prebuffer_ms、providers.qwen3_tts 默认值
改动  scripts/{tts_probe,tts_samples,speech_smoke,gpu_mem_probe}.py   增加 --tts qwen 支持（默认行为不变）
改动  tests/{test_config,test_mock_tts,test_runtime}.py               配置/工厂/启动预算/预缓冲覆盖 断言
改动  docs/{ARCHITECTURE,IMPLEMENTATION_STATUS}.md、README.md、AGENTS.local.md
```

### 11.4 运行过的命令与结果

| 命令 | 结果 |
| --- | --- |
| `uv run pytest` | **683 passed**（Phase 04A 收口 598 + 04D 新增：Provider 72 + 配置/工厂/运行时/预缓冲 13 等） |
| `uv run pytest tests/test_qwen3_tts_provider.py -q` | **72 passed in 5.94s**（真 asyncio TCP 假服务器，不联网） |
| `uv run ruff check` / `ruff format --check` | 全绿（src/tests/scripts） |
| `uv run desktop-companion --check` | 退出码 0（含 `providers.qwen3_tts` 新默认值） |
| `uv run python scripts/tts_probe.py --tts qwen --runs 3 --json docs/qwen3-tts-probe.json` | RESULT OK：health `format=verified`；单段 TTFA 14.3–16.1 ms / RTF 0.09；多段 3 段=3 请求；取消 2 块 + 复测 97.5 ms |
| `uv run python scripts/tts_samples.py --tts qwen --sink player --prebuffer-ms 300 --out-dir docs/tts-samples-qwen` | 10/10 成功；音频 4.3–13.3 s；**underruns=0** |
| `uv run python scripts/speech_smoke.py --tts-type qwen3_tts --expect-speech` | `idle→synthesizing→playing→idle`；首文本 492 ms / synthesizing 597 ms / playing 1238 ms；干净退出 |
| `gpu_mem_probe --tts qwen --baseline-only` → 起服务 → `--skip-llm` | 基线 1,848 MiB；常驻 4,953（+3,105）；峰值 5,059（+3,211）；RTF 0.10 |
| 取消 GPU 回落（临时脚本，nvidia-smi 50 ms 采样） | 95% → 3% 约 0.5 s；after_cancel 正常（47 块 / 3.76 s） |
| 15 句长语音（真实 Coordinator+Player，静音消费） | 109.4 s 音频 / 墙钟 110.6 s / `underruns=0` / `dropped=0` |

### 11.5 未完成项与风险（04D 收口时）

1. **人工试听未完成**：`docs/tts-samples-qwen/record.md` 的试听列留空；04C 的 29 条匿名样本同样待听。
   在用户签字前不能写"音质验收通过"。
2. **中文参考/克隆未验收**：`qwen_voice_register.py` 已可用，但真实 Base 注册与克隆效果未在本轮验证。
3. **情绪控制不稳定**：CustomVoice `instructions` 只透传、标 experimental；`ryan + angry/slow` 有漂移证据。
4. **服务端无格式响应头**：靠 WAV 探针（首查多一次短合成，服务忙时可能延后为 `format=unverified`）。
5. **`max_batch=1`**：并发合成未验证；Provider 串行排队。与本地 LLM 的 GPU 竞争属于 04E。
6. **默认未切换**：`providers.tts.type` 仍 `none`；是否把 Qwen 设为默认由用户确认。
7. **MOSS 静音冒烟的既有问题**：`speech_smoke.py --mute` 在 Mock/MOSS 路径同样超时，与 04D 无关。
8. **未提交模型权重/可执行文件/参考音频**：都在仓库外，只记录路径与 commit。

### 11.6 对下一阶段（04E）的接口

- `providers.qwen3_tts.mode=managed` 已预留（现在明确报 `ConfigError`）：04E 实现进程托管时替换构造期分支即可，
  请求/音频路径不动。
- `llm_priority` 判定应在 04E 的 Scheduler：Provider 只提供健康检查、`stream.stats`（TTFA/RTF/每段耗时）
  与 `busy_timeout_s`/`max_batch` 语义，不自己决定调度。
- 04E 可复用 `QwenStreamStats` 记录每段首字节/首块/合成耗时；显存基线见 `docs/qwen3-tts-vram.json`。

## 12. Phase 04E：Qwen TTS 运行管理与 LLM 优先 GPU 调度（2026-11-18）

> 任务书：`04E_任务书_TTS运行管理与GPU调度.md`；契约：`.dsh/notes/phase04e-contract.md`；
> 完整说明与实测：`docs/qwen3-tts-runtime-and-scheduling.md`；
> 证据：`docs/gpu-schedule-probe.json`（应用级）、`docs/tts-concurrency-probe.json`（直连并发）、
> `docs/qwen-runtime-probe.json`（managed 启停）。

### 12.1 结论

- 默认 `gpu.policy=llm_priority`：本地 LLM 生成期间只缓冲语音文本，LLM 本轮结束后才向 TTS
  发合成请求；应用级实测 **LLM decode 130.76 tok/s = 基线的 100.3%**、合成与 LLM **0/3 轮重叠**；
  直连串行对照 **101.2%**（四条轮次 99.8%–102.9%）。
- `overlap` 仅显式开启：应用级实测 **59.93 tok/s（−54%）**；直连并发窗口 **−60.6%**、整轮 −50.8%，
  单条长请求（复现 04C 形态）窗口 −61.2%。设置页在 `overlap` 时显示这段实测代价（−61% / −51% / RTF 3.2×）。
- `providers.qwen3_tts.mode` 三态落地：`external`（默认，用户自管）/ `managed`（本应用启停
  `tts-server.exe`，无 shell）/ `disabled`（等价 `type=none`）；**只终止自己启动的进程**。
- `sentence_interleave` 判定 **unsupported**（llama.cpp 无暂停/续接同一生成的 API），
  `gpu.policy` 不接受该取值。

### 12.2 对照任务书 §7 验收

| # | 验收标准 | 状态 | 证据 |
| --- | --- | --- | --- |
| 1 | `llm_priority` 下 LLM 生成期间没有本地 TTS 推理 | ✅ | 应用级 3/3 轮 `speech_overlapped_llm=false`、3/3 轮出现 `waiting`；协调器测试断言"waiting 期间 `open_stream` 调用数=0" |
| 2 | 默认调度下 LLM decode tok/s ≥ LLM-only 的 95% | ✅ | 130.76 / 130.34 = **100.3%**（直连串行 99.8%）；20 轮连测 96.6% |
| 3 | LLM 完成后 TTS 暖机 TTFA ≤800 ms、RTF ≤0.7、常驻 ≤4 GB、峰值 ≤5 GB | ✅ | TTFA **18.5 ms**（应用级首个整帧 36–65 ms）、RTF **0.092**；TTS 常驻增量 **+3021 MiB**、并发整卡峰值 13.24 GB（TTS 侧无额外暴涨） |
| 4 | 语音生成期不与 LLM decode 重叠；合成结束后即使仍在播放也允许 LLM 用 GPU | ✅ | 槽位在合成结束即 `release_tts()`（`_pump` 的 finally）；252–260 s 长音频播放期间 GPU 已交还（显存/利用率采样见 JSON） |
| 5 | 新请求/停止/切会话时旧轮生成与待播取消、无串轮 | ✅ | 取消场景：旧 utterance `speech.cancelled`、新轮新 utterance 起播，旧音频不串入；协调器/编排器既有 barge-in 测试全绿 |
| 6 | `overlap` 默认关闭；开启前能提示实测代价 | ✅ | 默认 `llm_priority`；设置页 `gpu_policy=overlap` 时显示"下降约 61%（整轮 −51%）、RTF 0.09→0.29" |
| 7 | external/managed/disabled 行为符合规定，不误杀外部进程 | ✅ | `docs/qwen-runtime-probe.json`：managed 真实启停（2.1 s / 155 ms，exit=1，端口释放）；external 不启动不终止（8095 手工实例全程存活）；disabled 工厂返回 `None`；缺模型/端口占用有可读诊断（263 ms 失败，不拉子进程） |
| 8 | 15 s+ 语音、20 轮连续会话：无明显欠载、队列无限增长、资源泄漏、服务僵死 | ✅（含一条已知服务端崩溃） | 259.84 s 音频 underruns=0、dropped=0；20/20 轮每轮 underruns=0、0 语音错误、显存无增长；**第一轮 20 轮测试第 10 轮 `tts-server.exe` 进程内崩溃（0xc0000409），重跑未复现**，按设计降级为纯文字 |
| 9 | 全量 pytest / ruff check / ruff format --check 通过 | ✅ | 见 §12.4 |

### 12.3 新增/修改文件

```text
新增  src/desktop_companion/speech/scheduler.py        GpuScheduler + VramProbe（单槽/预算/快照）
新增  src/desktop_companion/services/base.py           ServiceState/ServiceStatus/ManagedService（Phase 13 复用点）
新增  src/desktop_companion/services/qwen_tts_runtime.py  Qwen 进程托管（检查/启动/日志/重启/停止）
新增  tests/test_gpu_scheduler.py                      调度器单测 28 项
新增  tests/test_qwen_tts_runtime.py                   runtime 真子进程 39 项
新增  tests/fake_tts_server_process.py                 可被 spawn 的假 tts-server（含失败模式）
新增  tests/test_speech_scheduling.py                  调度门控/拒绝/取消/服务装配/Orchestrator 通知 16 项
新增  scripts/gpu_schedule_probe.py                    应用级 GPU 调度验收探针（6 场景）
新增  scripts/qwen_runtime_probe.py                    managed 真实启停验收探针（4 场景）
新增  scripts/tts_concurrency_probe.py                 1.7B×9B 并发/串行/取消直连测量
新增  docs/qwen3-tts-runtime-and-scheduling.md         本阶段主文档
新增  docs/gpu-schedule-probe.json                     应用级调度证据
新增  docs/tts-concurrency-probe.json / .md            直连并发证据
新增  docs/qwen-runtime-probe.json                     managed 运行管理证据
新增  .dsh/notes/phase04e-contract.md                  04E 契约（含 §10 偏差）
改动  core/config.py                                   + GpuConfig、Qwen3TTSConfig.managed/mode=disabled、
                                                       回环校验、ProvidersConfig 交叉校验
改动  core/events.py                                   + SpeechState.WAITING（无新事件类型）
改动  speech/coordinator.py                            + scheduler/service 注入、acquire/release、
                                                       WAITING 状态、降级码、diagnostics()
改动  app/orchestrator.py                              + note_llm_generating（stream_text 前后 True/False）
改动  app/bootstrap.py                                 + build_scheduler/build_tts_service/llm_is_local_gpu；
                                                       diagnostics() 透出 speech
改动  providers/qwen3_tts.py                           managed 不再报错（进程由 runtime 管）；disabled 防御性报错
改动  providers/tts_factory.py                         mode=disabled → None
改动  providers/llama_cpp.py / mock_llm.py             + resource 元数据（local_gpu / none）
改动  ui/main_window.py / settings_dialog.py / app_shell.py  「语音：等待 LLM…」+ GPU 策略行与 overlap 代价
改动  config/default.yaml                              + gpu 段、qwen3_tts.managed 段、mode 注释
改动  tests/{test_config,test_events,test_mock_tts,test_qwen3_tts_provider,test_runtime}.py  04E 语义同步
改动  docs/{ARCHITECTURE,IMPLEMENTATION_STATUS}.md、README.md、AGENTS.local.md
改动  services/qwen_tts_runtime.py                     端口占用检测加 bind 兜底（收口时实测补强）
```

### 12.4 运行过的命令与结果

| 命令 | 结果 |
| --- | --- |
| `uv run pytest` | **789 passed**（04D 收口 683 + 04E 新增 106：调度 28 + runtime 40 + 集成 19 + UI 3 + 配置/事件/工厂/运行时 16） |
| `uv run ruff check` / `ruff format --check` | 全绿 |
| `uv run python scripts/gpu_schedule_probe.py --scenario baseline` | 130.34 tok/s 中位（122.37/130.64/130.34） |
| `--scenario priority --baseline-tps 130.34` | 130.76（100.3%）、3/3 轮 waiting、0 重叠、0 错误 |
| `--scenario overlap --baseline-tps 130.34` | 59.93（46.0%）、2/2 轮重叠（首个合成在 LLM 第 357 ms） |
| `--scenario long --max-tokens 250` | 音频 259.84 s、3248 块、underruns=0、dropped=0 |
| `--scenario turns20` | 20/20 成功、每轮 underruns=0、0 语音错误、LLM 125.92（96.6%） |
| `--scenario cancel --cancel-after-s 6.0` | 播放中取消 → 旧 utterance cancelled、新轮独立起播、无串轮 |
| `scripts/qwen_runtime_probe.py --port 8096` | 启动 2.18 s、健康（format=verified）、停止 155 ms、端口释放 |
| 同上 `--scenario missing-model / port-busy / external` | 缺模型/端口占用 FAILED+可读诊断；external 不启停进程 |

### 12.5 未完成项与风险（04E 收口时）

1. **TTS 服务端一次性崩溃**：第一轮 20 轮连测第 10 轮 `tts-server.exe` 进程内 fail-fast
   （`0xc0000409` / `ucrtbase.dll` / BEX64），重跑 20 轮未复现；客户端按设计降级为纯文字。
   `managed.restart_on_crash` 可兜底自动拉起。**不判 04E 失败，但作为已知风险保留。**
2. **TTS 侧并发代价与 04C 不一致**：04C 记录 RTF +6.6%，本测（两轮全量）0.087/0.092→0.281/0.291（3.2×），
   服务端 `[Perf]` 证明是服务端变慢（Talker 2.65→8.9 ms/帧、CodePredictor 4.21→13.2 ms/帧，
   非客户端读饥饿）；差异原因未定论（Base vs CustomVoice、llama.cpp 构建、Windows 调度状态）。
   文档如实写两方数字；另记录**绝对值随桌面 GPU 活动漂移**（124.7–131.8），跨轮比较只用比值。
3. **应用级仲裁的边界**：不做 CUDA 优先级/显存隔离；显存预算只做"开始前检查"（默认关闭）；
   WDDM 进程级显存不可得；多卡未验证。
4. **`note_llm_generating` 依赖上游**：绕过 Orchestrator 直接调 Provider 的代码（未来的工具短任务）
   不会通知调度器，需要显式补通知。
5. **`managed` 日志不滚动**：单文件追加（>1 MiB 启动时轮转一次），运行中不滚动；
   Phase 10 的日志清理再统一。
6. **健康巡检只到"进程+探针"级别**：不落 GPU 利用率曲线；曲线用探针脚本采样。
7. **`sentence_interleave` 未实现**（审查结论 unsupported）；`overlap` 仍标实验。
8. **人工试听/克隆/情绪验收仍 Pending**（承袭 04C/04D）。

### 12.6 对下一阶段的接口

- **Phase 13（通用 ServiceManager）**：复用 `services/base.py` 的
  `ServiceState/ServiceStatus/ManagedService`；`QwenTtsRuntime` 是参考实现。
  不要在本阶段之外再写一套服务生命周期。
- **Phase 05（STT）**：`SpeechState.WAITING` 只表示"等本地 LLM"，不要复用来表示"等麦克风"。
- **Phase 06（Tool Broker）/12（任务中心）**：工具/任务播报必须走同一个 `SpeechCoordinator` 入口，
  才能自动受 `llm_priority` 约束；不要另开不受控 TTS 通道。
- **Phase 07（线上模型）**：给线上 Provider 声明 `resource="remote"`（或让用户设
  `gpu.llm_local_gpu=no`），调度器就不会因为它门控本地 TTS。
