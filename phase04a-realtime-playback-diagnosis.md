# Phase 04A：MOSS 实时播放断续诊断与修复

> 任务书：`04A_任务书_MOSS实时播放断续诊断与修复.md`
> 环境：Windows 10 19045 / RTX 4090 24 GB / Python 3.12.12 / sounddevice 0.5.6 / MOSS-TTS-Realtime（sdpa+eager）
> 基线：Phase 04 实现（提交 `3b51ad1`）与 `docs/moss-tts-probe.md`
> 原始数据：`docs/diagnostics/`（每组的 `.jsonl.gz` 事件流 + `-summary.json` 汇总，索引见该目录 `README.md`）
> 诊断脚本：`scripts/playback_diagnosis.py`（不改生产逻辑，从外部包住真实链路）
> 结论日期：2026-11-18

## 1. 结论

**主要根因是 MOSS 端"生产速度 + 输出节奏"：到达侧 RTF 1.10–1.22（合成慢于播放），且音频块以
~0.5 s 一批、批间隔 P99 0.58–0.69 s / max 1.1 s 的方式到达；当 LLM 同时 decode 时 RTF 恶化到
3.1、批间隔 max 2.9 s。**

播放器（`AudioPlayer` 队列/回调）**不是根因**：用 Mock TTS 以 0.22–0.94× 实时速度推送时
**0 underrun**；只有生产慢于实时才出现 underrun，且次数随缺口单调增加（1.5×→103 次、
2.3×→248 次）。"落盘 WAV 连贯"只说明**内容**没有边界异常——拼接时把块间等待时间全部去掉了，
而实时播放必须等这些块。

**客户端修复是否足够？不够。** 本轮做了两件**有证据支持的客户端修复**（见 §6），能把
"每 20 ms 碎一下"变成"一次较长停顿"，但**不能**在 RTF>1 时保证长句连续播放：
数学上需要预缓冲 ≈ (RTF−1)×音频时长，输出越长需求越大，且并发 LLM 时需求随 RTF 一起爆炸。
要根治只有：TTS 提速（`torch.compile`/更强后端）、TTS 与 LLM 分卡、或在长回复上改用
"整句/整轮合成完再播"的模式（用延迟换连续，见 §3.4 的实测延迟代价）。

## 2. 测量方法（同一声钟域、不改生产逻辑）

```text
MossTTSProvider ──ProbeProvider/ProbeStream──► SpeechCoordinator ──► AudioPlayer(probe=…) ──► 设备
     记录 open/push/chunk.t                   （真实 pump/段队列）      记录 enqueue/pull/sink_open/finalize
```

- 所有事件时间用 `time.perf_counter()`；JSONL 每条带 `run`/`event`/`t` 与字段；
  `summary.json` 由事件换算（定义见 `docs/ARCHITECTURE.md` §14）。
- **入队时间 ≠ 设备消费时间**：`enqueue.buffered_ms` 与 `pull.filled_frames` 分开记录；
  `first_data_pull_ms` 才是"设备真正开始放声音"。
- 文本按 6 字符 / 30 ms 的节奏喂给 `SpeechCoordinator.feed_text`（模拟 LLM 流式），
  切分配置用默认 `speech.*`（首段 4 字、常规 6 字、`max_wait_ms=350`）。
- 对照组：预缓冲 0/250/500/1000/2500 ms × 3 文本（短 29 字 / 中 55 字 / 长 64 字）× 2 次；
  Mock TTS 快/略快/实时偏慢/慢四档；设备参数（blocksize 960 + latency high）；
  整轮合成后播放；LLM 并发。

## 3. 结果

### 3.1 预缓冲矩阵（真实 MOSS，默认设备，每格 2 次运行）

| 文本 | 预缓冲 | 音频 | TTFB | 到达RTF | gap P99 | gap max | 末累计缺口 | underrun | 静音总时长 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| short(29) | 0 ms | 6.36 s | 1103 ms | 1.10 | 587 ms | 623 ms | −634 ms | 9–58 | 620 ms |
| short(29) | 250 ms | 7.24 s | 946 ms | 1.12 | 580 ms | 581 ms | −887 ms | 43–83 | 1160 ms |
| short(29) | 500 ms | 5.52 s | 985 ms | 1.16 | 621 ms | 633 ms | −859 ms | 38–58 | 920 ms |
| short(29) | 1000 ms | 6.20 s | 968 ms | 1.18 | 626 ms | 633 ms | −1120 ms | 19–45 | 610 ms |
| short(29) | 2500 ms | 5.68 s | 948 ms | 1.10 | 579 ms | 580 ms | −575 ms | **0** | 20 ms |
| medium(55) | 0 ms | 13.60 s | 513 ms | 1.21 | 667 ms | 1118 ms | −2856 ms | 149–200 | 3290 ms |
| medium(55) | 250 ms | 13.88 s | 482 ms | 1.20 | 664 ms | 1080 ms | −2753 ms | 106–146 | 2350 ms |
| medium(55) | 500 ms | 14.16 s | 505 ms | 1.20 | 660 ms | 1098 ms | −2870 ms | 111–145 | 2380 ms |
| medium(55) | 1000 ms | 16.96 s | 498 ms | 1.19 | 614 ms | 1074 ms | −3225 ms | 77–110 | 1710 ms |
| medium(55) | 2500 ms | 12.56 s | 466 ms | 1.20 | 683 ms | 1093 ms | −2530 ms | **0** | 20 ms |
| long(64) | 0 ms | 12.16 s | 917 ms | 1.15 | 641 ms | 1089 ms | −1885 ms | 116–124 | 2250 ms |
| long(64) | 250 ms | 12.96 s | 921 ms | 1.15 | 617 ms | 1058 ms | −1925 ms | 115–119 | 2190 ms |
| long(64) | 500 ms | 12.80 s | 903 ms | 1.15 | 626 ms | 1083 ms | −1943 ms | 41–89 | 1190 ms |
| long(64) | 1000 ms | 12.64 s | 918 ms | 1.16 | 626 ms | 1075 ms | −2051 ms | 64–71 | 1250 ms |
| long(64) | 2500 ms | 12.28 s | 948 ms | 1.16 | 629 ms | 1083 ms | −1938 ms | **0** | 20 ms |

读法（三个关键量）：

- **到达 RTF 1.10–1.22**：收到全部音频所花的墙钟比音频本身长 10–22% → 末累计缺口 −0.6～−3.2 s；
  缺口随时长线性增长，**2500 ms 预缓冲只够覆盖 ≤12 s 左右的句子**（实测三轮 0 underrun，
  因为缺口 −0.6～−2.5 s < 2.5 s）。
- **块间隔 P99 0.58–0.69 s、max 1.1 s**：即使 RTF 恰好 =1，0 ms 预缓冲也会因"批到达"立刻碎响；
  这就是为什么"预缓冲 250 ms 并不比 0 ms 好"——它小于一个批间隔。
- **静音总时长** ≈ 缺口（RTF−1）×音频；`underrun` 次数与静音时长同向，说明不是偶发调度毛刺。

### 3.2 播放器能力对照（Mock TTS，固定 24 kHz / mono / s16le）

| 生产者 | 到达 RTF | gap P99 | underrun | 结论 |
| --- | --- | --- | --- | --- |
| `--mock-delay-ms 0`（远快于实时） | 0.22–0.45 | 40–61 ms | **0** | 播放器稳定 |
| `--mock-delay-ms 10`（略快） | 0.77–0.94 | 17–23 ms | **0** | 播放器稳定 |
| `--mock-delay-ms 20`（≈1.5× 实时） | 1.54–1.70 | 32–34 ms | 103 | 生产不足 → 欠载 |
| `--mock-delay-ms 33`（≈2.3× 实时） | 2.30–2.48 | 48 ms | 248–253 | 生产不足 → 欠载 |

**这是"播放器无独立缺陷"的直接证据**：同一播放器、同一设备，只要生产者跟得上就 0 欠载；
欠载次数随生产缺口单调上升。

### 3.3 设备与回调

- 每次回调请求的帧数恒为 **480**（24 kHz 下 20 ms），没有设备侧异常帧/抖动；
- 设备打开：进程内首次 **620–740 ms**，之后 **25–34 ms**（PortAudio 句柄复用）；
  首次打开只增加"首播延迟"，不造成持续碎响；
- 设备参数对照（blocksize 960 + latency `high`，medium/预缓冲 250）：underrun 59–78、
  静音 1.9–2.6 s —— 与 blocksize 480/low 的 106–146 / 2.35 s 同量级，**没有改变根因**；
- 所有运行 `sink=sounddevice`，**没有**发生 null 播放降级（即不是"设备打不开假装在放"）。

### 3.4 整轮合成后播放（utterance 模式对照）

| 文本 | 音频 | underrun | 首帧开始播放 | TTFB | 播放/音频 |
| --- | --- | --- | --- | --- | --- |
| short | 10.24 s | **0** | 13.39 s | 1.02 s | 10.240/10.240 s |
| medium | 11.36 s | **0** | 15.05 s | 1.19 s | 11.360/11.360 s |
| long | 11.36 s | **0** | 16.11 s | 0.71 s | 11.360/11.360 s |

结论：**完整音频到齐再播，播放必然连续**（这是上界）；代价是首帧要等 ~13–16 s
（≈ 合成时间 + 设备打开），比流式首帧（1–3 s）慢一个数量级。这是"低延迟 vs 连续"的
真实取舍，不能两者兼得（在当前 RTF 下）。

### 3.5 并发 LLM decode 时（9B 正在生成，长文本，预缓冲 1000，hold 0）

| 运行 | 音频 | 到达RTF | gap P99 | gap max | underrun | 静音总时长 | LLM 吞吐 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 12.48 s | **3.16** | 1733 ms | 2881 ms | 1093 | 21.7 s | 69.5 tok/s |
| 2 | 15.28 s | **3.13** | 1690 ms | 2837 ms | 1403 | 27.8 s | 69.8 tok/s |

这是本项目**真实使用形态**（LLM 边生成、TTS 边说话）的对照：GPU 被 LLM decode 抢占后，
MOSS 的到达 RTF 从 1.1–1.2 恶化到 **3.1**，块间隔最长 2.9 s。此时任何有限预缓冲都盖不住，
流式播放必然大面积断续。

### 3.6 客户端"欠载后等回填"（`audio.underrun_hold_ms`）对照

| 配置 | underrun | 停顿次数（pause_episodes） | 静音总时长 |
| --- | --- | --- | --- |
| medium / pb500 / hold 0（旧行为） | 44 / 59 | 0 / 0 | 820 / 1100 ms |
| medium / pb500 / **hold 1200** | **1 / 1** | **1 / 1** | 1260 / 1280 ms |
| long / pb1000 / hold 1200 | 0 / 2 | 0 / 2 | 20 / 2380 ms |

机制：第一次取不到数据时不立刻"来一块放一块"，而是**先静音等待，等缓冲回填到阈值再续播**。
总静音时长接近（甚至略增），但听感从"每 0.6 s 碎一下"变成"一次停顿后连续放一段"。
它不改变根因（RTF>1），也不减少总缺口，只把碎响合并成可接受的停顿——**这是有意的取舍**。

## 4. 六类可能性的逐项判定

| 可能性 | 关键指标 | 判定 | 证据 |
| --- | --- | --- | --- |
| 1. TTS 生产速度不足 | 到达 RTF 1.10–1.22；末累计缺口 −0.6～−3.2 s；并发时 RTF 3.1 | **主因** | §3.1 / §3.5 |
| 2. 输出节奏不均匀（burst） | 块间隔 P99 0.58–0.69 s、max 0.58–1.14 s；Mock 固定节奏时 gap P99 17–48 ms | **次因**（放大碎响：0/250 ms 预缓冲小于一个批间隔） | §3.1 / §3.2 |
| 3. HTTP/async reader 调度 | `open_stream` 14–15 ms；到达间隔与生成节奏一致；未出现成批积压后突涌 | **无异常** | `open_stream_ms`、`chunk` 间隔分布 |
| 4. Coordinator 段间等待 | 段数 3–6；相邻 `push_text` 间隔 P50 55–85 ms、max 117–125 ms（文本侧供给） | **无异常**（无段间长空档） | `push_gap_ms` |
| 5. Player 队列/回调 | 回调帧恒 480；Mock 快时 0 欠载；`queue_ms.max` 从 825 ms（pb0）到 2540 ms（pb2500）；入队/消费分开记账 | **无独立缺陷** | §3.2 / `pull` 事件 |
| 6. 设备 / 声卡 | 首次打开 620–740 ms、暖 25–34 ms；无 null 降级；blocksize/latency 变化同量级 | **不是根因**（只影响首播延迟） | §3.3 |

## 5. 为什么"WAV 连贯、实时断续"

WAV 是把 `AudioChunk.data` **按顺序拼接**：块之间的真实等待（0.6–1.1 s）被完全抹掉，
所以听起来连贯只证明**音频内容**没有边界错乱/静音/咔哒。实时播放必须等这些块到达，
而播放器按 20 ms 的节奏消费；块没到就填静音 → 听感碎响。
两者不矛盾：一个测"内容"，一个测"到达节奏"。

## 6. 本轮代码修改（最小、可回退、有测试）

| 文件 | 修改 | 原因 |
| --- | --- | --- |
| `src/desktop_companion/core/config.py` | 新增 `audio.underrun_hold_ms`（默认 **0** = 旧行为；建议 1000–1500） | 用配置表达"欠载后等回填再续播"，不写死策略 |
| `src/desktop_companion/speech/player.py` | 欠载后进入 hold：不消费队列、输出静音，缓冲 ≥ 阈值再续播；`PlaybackReport` 增加 `pause_episodes` / `hold_silence_ms`；`diagnostics()` 暴露 holding；新增可选 `probe=` 诊断钩子（默认 None，零开销） | §3.6 的实测改善；probe 供本任务采集，不改变生产行为 |
| `tests/test_audio_player.py` | +2 项：hold 生效（100 ms 缓冲不解冻、200 ms 才续播）、hold=0 立即可续播 | 钉住行为与回退语义 |
| `scripts/playback_diagnosis.py` | 新增诊断脚本（ProbeProvider/ProbeStream/AudioPlayer.probe、矩阵/对照/utterance/并发模式、JSONL+summary） | 可复现证据 |
| `docs/diagnostics/` | 原始 JSONL（gzip）+ 汇总 JSON + 索引 | 交付物 |

没有做（刻意）：不提高默认预缓冲、不插静音"掩盖"、不降采样、不丢音频、不改 MOSS 上游、
不改事件协议、不动 Coordinator 的取消/降级语义。

## 7. 仍然存在的限制（未解决）

1. **RTF 1.10–1.22 是模型/后端能力**：客户端无法让"生产慢于实时"变得连续；
   有限预缓冲只买到时间（≈预缓冲长度/(RTF−1) 秒的连续播放）。
2. **并发 LLM decode 时 RTF≈3.1**：这是本项目最常见的使用形态；流式播报在该窗口内必然严重断续，
   只能通过"整轮合成后播放"（延迟 ~13–16 s/11 s 音频）、TTS/LLM 分卡、或 TTS 提速来规避。
3. **首播延迟组成**：TTFB 0.5–1.2 s（冷启动可到 3–6 s）+ 设备打开 0.62–0.74 s（进程内首次）
   + 预缓冲；三者要分开看，不能混成一个"卡顿"。
4. **设备侧只验证到"打开/回调正常"**：真实蓝牙插拔/切换、其他声卡独占模式未系统验证。
5. **diagnosis 的 utterance 模式只是对照测量**，尚未做成生产配置项（若需要，建议作为
   `speech.playback_mode: streaming|utterance` 的后续任务，含取消/背压/内存上限设计）。

## 8. 建议

1. **近期（不改模型）**：MOSS 后端仅作**临时/实验后端**保留。实时对话把
   `audio.start_prebuffer_ms` 设为 1000–1500、`audio.underrun_hold_ms` 设为 1200；
   长回复（>12 s 音频）预期仍会有一次到数次停顿——如实告知用户/在 UI 上以"语音较慢"提示。
2. **中期**：给 MOSS 开 `torch.compile`（`triton-windows` + `MOSS_ENABLE_COMPILE=1`），
   或换低 RTF 的 TTS 后端；目标 RTF < 0.8，届时有限预缓冲即可稳定。
3. **若必须"长回复连续"**：实现 `speech.playback_mode: utterance`（整轮合成后播放），
   代价是最长 ~RTF×音频 的首播等待；或让 TTS 与 LLM 分卡。
4. **并发策略**：LLM decode 与 TTS 合成同时抢卡时，优先保证 LLM；TTS 在 LLM 完成后
   再补播，或把 TTS 放到第二张卡/CPU 后端（需单独评估）。

## 9. 复现命令与证据路径

```powershell
# 预缓冲矩阵（3 文本 × 5 预缓冲 × 2 次）
uv run python scripts/playback_diagnosis.py --matrix --repeats 2

# 播放器能力对照
uv run python scripts/playback_diagnosis.py --tts mock --mock-delay-ms 0  --text medium --prebuffer-ms 0 --repeats 2
uv run python scripts/playback_diagnosis.py --tts mock --mock-delay-ms 33 --text medium --prebuffer-ms 0 --repeats 2

# 整轮合成后播放（连续性上界 / 延迟代价）
uv run python scripts/playback_diagnosis.py --tts moss --text long --playback-mode utterance --repeats 1

# 并发 LLM decode（需 llama-server 已运行）
uv run python scripts/playback_diagnosis.py --tts moss --text long --prebuffer-ms 1000 --llm-url http://127.0.0.1:11435 --repeats 2

# hold 对照
uv run python scripts/playback_diagnosis.py --tts moss --text medium --prebuffer-ms 500 --underrun-hold-ms 1200 --repeats 2
```

证据：

- `docs/diagnostics/playback-20261118-083158-{summary.json,jsonl.gz}` —— 预缓冲矩阵（30 次）
- `docs/diagnostics/playback-20261118-083942/083952/084005/084112-*` —— Mock 四档
- `docs/diagnostics/playback-20261118-084122-*` —— 设备参数对照
- `docs/diagnostics/playback-20261118-084156/084220/084247-*` —— utterance 模式
- `docs/diagnostics/playback-20261118-084315-*` —— 并发 LLM
- `docs/diagnostics/playback-20261118-084527/084602/084635-*` —— hold 对照
- 索引与字段说明：`docs/diagnostics/README.md`
