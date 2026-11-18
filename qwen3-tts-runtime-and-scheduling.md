# Qwen TTS 运行管理与 GPU 调度（Phase 04E）

> 状态：**已实现并验收**（2026-11-18）。默认 GPU 策略 `llm_priority`；Qwen TTS 支持
> `external` / `managed` / `disabled` 三种运行模式；`overlap` 保留为实验模式。
> 上游依据：`04E_任务书_TTS运行管理与GPU调度.md`、`00_项目总纲.md` §4.4/§10、
> 04D 契约（`.dsh/notes/phase04d-contract.md`）、04C 决策（CONDITIONAL GO）。
> 契约：`.dsh/notes/phase04e-contract.md`（实现/测试共同基准）。

## 1. 一句话结论

- **默认不再让本地 LLM decode 与 Qwen TTS 推理抢 GPU**：`gpu.policy=llm_priority`，
  LLM 生成期间只缓冲语音文本，本轮 LLM 结束后再向 TTS 发合成请求（`SpeechState.WAITING`）。
- **TTS 服务可以由本应用托管**（`providers.qwen3_tts.mode=managed`）：只启动
  `tts-server.exe`，只杀自己启动的进程，失败给可读诊断且不阻塞 GUI。
- `overlap` 只能显式开启，设置页与本文都给出**实测代价**（见 §6）。
- `sentence_interleave`（LLM 生成中途暂停、把已生成文本送 TTS、再续接同一生成请求）
  **判定 unsupported**：llama.cpp 没有暂停/续接同一 SSE 生成的 API（见 §7）。

## 2. GPU 策略

### 2.1 三种策略

| `gpu.policy` | 行为 | 默认 | 适用 |
| --- | --- | --- | --- |
| `llm_priority` | 本地 LLM 生成期间 `acquire_tts()` 挂起；LLM 本轮结束后放行 | ✅ | 日常使用（保护 LLM 体验） |
| `overlap` | 立即放行（仍做显存预算检查） | ❌ 仅显式开启 | 实验：接受 LLM 掉速换首音频更早 |
| `sentence_interleave` | **不支持**（配置不接受该值） | — | 见 §7 |

判定"本地 LLM"的依据是 **Provider 资源元数据**：`LlamaCppProvider.resource="local_gpu"`、
`MockLLMProvider.resource="none"`；未声明的一律按 `local_gpu` 处理（宁可多等，也不把 LLM 挤慢）。
`gpu.llm_local_gpu=yes|no` 可强制覆盖 —— 将来接入"线上强模型"时，TTS 不应因为它被门控。

### 2.2 为什么默认 `llm_priority`

- 04A/04C 实测：0.6B GGML TTS 与 9B LLM decode 并发时 LLM 从 **133.2 tok/s 掉到 56.4 tok/s
  （-57.7%）**，TTS 侧 RTF 仍约 0.075 —— 掉的是 LLM 体验，不是 TTS 质量。
- 04E 补测 1.7B（见 §6.2）：并发时 LLM 掉速同样显著，而串行（`llm_priority`）几乎不掉。
- 一轮回复里 TTS 天然只发生在文本产出之后；把合成推迟到 `completed` 之后，几乎不影响
  用户感知的首音频时间（TTFA 仍在十几毫秒量级 + 300 ms 预缓冲），却把 LLM 的 decode 完全保住。

### 2.3 边界（必须如实说）

- 这是**应用级任务仲裁**：只决定"本应用何时向本地 TTS 发合成请求"。
- 它**不是** CUDA stream 优先级、不做 SM/显存带宽隔离、不能阻止其它进程（浏览器、游戏、
  其它推理服务）抢 GPU；也不能在 TTS 生成中途被操作系统抢占式让路。
- 显存预算只用于"开始合成前"的检查；已经在生成的 TTS 请求不会被"半路掐掉"（只有
  barge-in/停止会取消它）。

## 3. 调度实现（`src/desktop_companion/speech/scheduler.py`）

```text
Orchestrator ──note_llm_generating(True/False)──▶ GpuScheduler ◀──acquire_tts()/release_tts()── SpeechCoordinator
                                                   │
                                                   └── VramProbe（nvidia-smi，只读、TTL 缓存）
```

- **单槽互斥**：同一时刻只允许一路 TTS 合成（服务端 `max_batch=1`，Provider 也串行）；
  第二个 `acquire_tts()` 挂起直到 `release_tts()`。
- **等待模型**：每个等待者一个 `asyncio.Event`，注册后复查状态（避免丢唤醒），
  没有忙等轮询；`acquire_tts()` 被取消时 `CancelledError` 原样上抛且不占槽。
- **LLM 门控**：`llm_busy` 期间挂起，`note_llm_generating(False)` 唤醒；超过
  `gpu.llm_wait_timeout_s`（默认 180 s）→ `TIMEOUT` → 本轮语音降级（文字不受影响）。
  Orchestrator 在 `stream_text` 的 `finally` 里一定发 `False`，所以取消/失败不会把门永久关死。
- **显存预算**：`gpu.whole_card_budget_mb`（默认 `null` = 不限制）。设置后每次 `acquire`
  前读整卡已用显存；超预算按 `probe_interval_s` 重试，超过 `budget_defer_timeout_s`
  → `BUDGET_DENIED` → 本轮语音降级。探测不可用（没有 nvidia-smi/超时/输出异常）→ **放行**
  并在快照里记 `vram.state="unavailable"`（宁可抢，也不因探针坏掉不说话）。
- **释放时机**：`SpeechCoordinator._pump` 的 `finally` —— 即"合成结束/失败/取消"时立刻
  `release_tts()`；随后的**播放阶段只占 CPU/声卡**，GPU 可以继续服务 LLM（验收 §4 条）。
- **观测**：`GpuScheduler.snapshot()` →
  `policy/llm_busy/tts_active/deferred/budget_deferred/timeout/granted/overlap/last_wait_ms/max_wait_ms/vram{budget_mb,used_mb,state}`；
  `SpeechCoordinator.diagnostics()` 把调度器与服务状态一起暴露给探针脚本。

### 3.1 一轮播报的状态机（新增 `SpeechState.WAITING`）

```text
feed_text（LLM 增量） ──▶ 切段 ──▶ 段队列
                                     │  LLM 忙 + llm_priority
                                     ▼
                              WAITING（"语音：等待 LLM…"，detail 说明原因）
                                     │  LLM 完成 → acquire 放行
                                     ▼
                            SYNTHESIZING ──▶ PLAYING ──▶ IDLE
```

- 只有"真的等待了"才出现 `WAITING`（立即放行不闪状态）；没有配置调度器时状态序列与 04D 完全一致。
- 调度拒绝（超时/超预算）走既有降级路径：`speech.error(degraded=True)`，错误码
  `tts_scheduler_deferred` / `tts_vram_budget`，`user_message` 仍是"语音暂时不可用，本轮只显示文字。"，
  `assistant.text.completed` 保持 `completed`。

## 4. Qwen TTS 运行管理（`src/desktop_companion/services/`）

### 4.1 三种模式

| `providers.qwen3_tts.mode` | 行为 |
| --- | --- |
| `external`（默认） | 连接用户自己启动的 `tts-server.exe`；本应用只握手，**永不**终止它 |
| `managed` | 本应用用 executable + argv + cwd + env 启动/停止它（无 shell），退出按 `stop_on_exit` |
| `disabled` | 语音不启用（与 `providers.tts.type=none` 同效） |

`managed` 要求 `providers.tts.base_url` 是**回环地址**（不能托管远端进程），并且
`providers.qwen3_tts.managed.executable` 必填 —— 这两条是配置期错误；其它（文件缺失、DLL 缺失、
端口占用、启动超时）都是**运行期可读诊断**，不会拦死 GUI 启动。

### 4.2 managed 启动参数（与 04D 冻结的 CLI 一致）

```text
tts-server.exe --model <talker.gguf> --codec <tokenizer-codec.gguf>
               --alias qwen3-tts --host 127.0.0.1 --port 8095
               --lang Chinese --max-batch 1 [*managed.args]
cwd  = managed.cwd（默认 exe 所在目录：Windows DLL 搜索默认含它）
env  = 父进程环境 + managed.env
```

- `host`/`port` 默认从 `providers.tts.base_url` 推导（单一事实源）；`alias`/`lang`/`max_batch`
  在 `managed` 段可覆盖。
- **不用 `shell=True`**：参数逐项传递，`managed.args` 原样追加（例如 `["--device","0"]`）。

### 4.3 启动前检查（失败 → `FAILED` + 可读 `detail`，不抛异常）

1. `executable` 存在；`model`/`codec` 存在；
2. 同目录 `ggml.dll` / `ggml-base.dll` / `ggml-cpu.dll` / `ggml-cuda.dll` 存在
   （这四个是 `tts-server.exe` 的真实依赖，缺一不可；`managed.required_files` 可追加）；
3. `host:port` 未被占用 —— 已占用时只报告"端口被占用（可能已有一个 tts-server 在跑）"，
   **绝不 kill** 任何进程；
4. 轮询注入的健康探针（复用 `Qwen3TTSProvider.health_check()`，含 WAV 格式探针）直到
   `startup_timeout_s`；进程提前退出 → 用退出码 + 最近日志行生成诊断
   （`0xC0000135`/`3221225781` = `STATUS_DLL_NOT_FOUND` → 提示缺 DLL/VC 运行库）。

### 4.4 退出、崩溃与日志

- **只终止自己启动的进程**：`stop_on_exit=false` 时只 detach 并保留进程；`external` 实例永不碰。
- `terminate()` → `shutdown_timeout_s` → `kill()`；`stop()` 幂等。
- `restart_on_crash`（默认关）+ `max_restarts` + `restart_backoff_s`：崩溃后有限次重启，
  超过上限转 `FAILED` 并停止拉起；单实例锁，不会并发拉起两个。
- `health_monitor_interval_s`（默认 10 s，仅 managed）：进程在 `RUNNING` 后消失 → 按重启策略处理；
  探针失败但进程还活 → 只记 `detail`（可能是模型还在加载/服务忙），不重启。
- stdout/stderr 落盘 `<log_dir>/qwen-tts-server.log`（`log_dir` 默认用户状态目录 `logs/`），
  每次 `start()` 截断；超过 1 MiB 先轮转成 `.log.1`（只留一份，运行中不滚动）。
  内存保留最近 200 行（单行截 2000 字符）供诊断；`status().artifacts` 记录
  `commit/model/codec/executable/dlls{found,missing}/host/port/log_path/tail`。

### 4.5 与 Phase 13 的关系

- 本阶段只交付 **Qwen TTS 特定的 lifecycle adapter**（`services/base.py` 的
  `ServiceState` / `ServiceStatus` / `ManagedService` 协议 + `QwenTtsRuntime` 实现）。
- Phase 13 的通用 `ServiceManager` 应复用这三个类型，把 llama-server / STT / 其它服务
  接进同一套生命周期与诊断口径；**不要在 04E 里提前实现全局服务管理**。

## 5. 播放与队列策略（沿用 + 按后端区分）

- **预缓冲按后端区分**：Qwen `providers.tts.start_prebuffer_ms=300`（04D 实测 RTF≈0.09，
  0 欠载）；MOSS 的 2500 ms 经验值**不**复制给 Qwen；全局 `audio.start_prebuffer_ms` 不动。
- **队列有上限**：`speech.buffer_high_watermark_s`（背压）与 `speech.buffer_max_s`（硬上限）。
  Qwen 合成快于播放，长回答时背压会让合成侧等待，队列不会无限增长（§6.4 的 15 s+ 样例实测）。
- **取消/新轮隔离**：一轮 = 一个 utterance = 一个 epoch；取消会停 pump/读任务、清队列、
  关闭当前合成响应，迟到音频按 `(utterance_id, epoch)` 丢弃并计入 `late_chunks_dropped`。
- **barge-in**：新用户消息到达时 Orchestrator 先 `speech.cancel()` 再取消 LLM；
  TTS 合成中被取消 = 客户端断连 = 服务端停止生成（04C/04D 实测 GPU 约 0.5 s 回落）。
- **分段长度**：`speech.min_chars/first_min_chars/max_chars/max_wait_ms` 仍是唯一旋钮；
  Qwen 每段一次请求（`chunk_target_ms=80` 聚合音频块），不按 token 切。

## 6. 实测数据（本机 RTX 4090 24 GB）

> 口径：LLM = `DeepSeek-V4-Pro-Qwen3.5-9B-UNCENSORED-heretic.Q4_K_M`，131072 ctx、q8_0 KV、
> `--reasoning off`（`/props` 确认 `n_ctx=131072`、`total_slots=1`）；TTS = `qwentts.cpp 2edb35f`
> 1.7B CustomVoice Q8_0 + tokenizer Q8_0，`--max-batch 1`、`vivian`、`Chinese`。
> 两组证据：
> - **应用级**（真实 `Orchestrator` + `SpeechCoordinator` + `AudioPlayer`，null sink 按实时速度消费）：
>   `docs/gpu-schedule-probe.json`，脚本 `scripts/gpu_schedule_probe.py`；
> - **直连 HTTP**（不经过应用层，测服务端竞争上限）：`docs/tts-concurrency-probe.json` +
>   `docs/tts-concurrency-probe.md`，脚本 `scripts/tts_concurrency_probe.py`。
> 所有数字都是本机实测，不是跨机器保证。

### 6.1 LLM-only 基线（decode，用同一 provider 路径）

| 来源 | decode tok/s（每次） | 中位 | TTFT |
| --- | --- | --- | --- |
| 应用级 baseline（TTS 已加载、空闲） | 122.37 / 130.64 / 130.34 | **130.34** | 96.8 ms（首轮冷启动偏高） |
| 直连 HTTP（3 次，干净全量） | 124.45 / 124.69 / 124.97 | **124.69**（后置复测 125.50） | 54.9 ms |

两口径的**绝对值会随桌面 GPU 活动漂移**（本轮直连 124.69 vs 应用级 130.34；前几轮 128.4–131.8，
桌面基线 util 12%→20%），所以跨层比较只用**比值/百分比**；各自内部的串行/并发对比不受影响。

### 6.2 `llm_priority` vs `overlap`（应用级，同一 prompt）

| 场景 | LLM decode tok/s（中位） | 相对基线 | TTS 合成与 LLM 是否重叠 | 语音错误 |
| --- | --- | --- | --- | --- |
| `llm_priority`（3 轮） | **130.76**（132.04/130.76/130.66） | **100.3%** | **0/3 轮**（`waiting` 状态 3/3） | 0 |
| `overlap`（2 轮，短 prompt） | **59.93**（61.20/58.67） | **46.0%**（−54.0%） | 2/2 轮（首个合成在 LLM 第 357 ms 开始） | 0 |

直连 HTTP（T3 干净全量）同口径：并发窗口内 LLM **124.60 → 49.13 tok/s（−60.6%）**、整轮 **61.34（−50.8%）**；
单条长请求（复现 04C 请求形态，`overlap-long`）窗口 **48.77（−61.2%）**、整轮 59.55（−52.2%）；
串行时序 **126.22 = 101.2%**（对后置基线 100.6%）。四条独立轮次里比值稳定：
**并发窗口 −60.6%～−61.3%、串行 99.8%～102.9%**，与 04C 的 1.7B 并发数据（LLM 侧 −61.0%/−61.7%）一致。

**给设置页 `overlap` 提示的硬数字**（用比值，避免绝对 tok/s 随桌面 GPU 活动漂移）：
语音合成期间 LLM decode 下降约 **61%**（应用级整轮约 **−54%**，直连整轮 −50.8%）；
语音 RTF 从 0.09 升到 **0.29（3.2×）**（TTS 侧与 04C 的 +6.6% 不一致，见 §9.6）。

### 6.3 TTS 单独（暖机，直连 HTTP，干净全量）

| 指标 | 实测 | 验收目标 | 结论 |
| --- | --- | --- | --- |
| TTFA（首字节） | **18.5 ms**（三次同值） | ≤800 ms | ✅ |
| RTF（合成/音频） | **0.092**（0.092–0.092） | ≤0.7 | ✅ |
| 长文本音频 | 25.4–26.4 s（127 字） | — | — |
| 并发时 TTFA（应用级首个整帧） | 36–65 ms；直连并发 51.4 ms | ≤800 ms | ✅ |

**显存**（整卡 used，基线=服务未启动 1901 MiB）：

| 状态 | 整卡 MiB | 相对基线 |
| --- | --- | --- |
| 桌面基线 | 1901 | — |
| LLM 加载后 | 10008 | +8107 |
| ＋TTS 加载后 | 13029 | **TTS 自身 +3021**（04D 为 +3105，与本轮应用级测量一致） |
| 并发峰值 | 13242 | +11341（24.56 GB 卡的 54%） |
| 全部停止后 | 1861 | −40（无泄漏迹象） |

TTS 常驻增量 ≈3.0 GB ≤4 GB ✅；并发峰值 13.24 GB 含 LLM，TTS 侧没有额外暴涨（≤5 GB 的 TTS 目标按增量算）。
**WDDM 下拿不到进程级显存**：只记整卡 used 与"TTS 自身增量"，不伪造进程级数字。

### 6.4 长语音 / 连续 20 轮 / 取消（应用级）

| 场景 | 结果 |
| --- | --- |
| 长回复（700-token prompt，1 轮） | 音频 **259.84 s**、3248 块、**underruns=0**、dropped=0；合成在 LLM 结束后开始（等待 1/1），播放期间 GPU 已释放给 LLM |
| 连续 20 轮（短问句） | **20/20 成功**、20/20 走 `waiting` 门控、**每轮 underruns=0**、0 语音错误；总耗时 372.7 s（每轮音频 9.8–26.6 s）；LLM decode 中位 125.92（相对基线 96.6%）；整卡显存 12.95–13.51 GB（无增长趋势） |
| 取消（LLM 完成、播放中 6.0 s 处取消） | 旧 utterance 收到 `speech.cancelled` 且状态回 idle；新轮用**新的 utterance** 从头合成（`speech.playback.started` 只属于新 id），**旧轮音频没有串入**；播放器 `dropped_chunks=0` |

**已知一次性异常（如实记录）**：第一轮 20 轮测试在第 10 轮时 `tts-server.exe`（pid 23772）
**在进程内崩溃**（Windows 事件日志 `0xc0000409` / `ucrtbase.dll` / BEX64），客户端读到断连后
按设计**降级为纯文字**（`tts_unreachable`，文字不受影响）；同一台机器重跑 20 轮 **20/20 成功、0 崩溃**。
判定为**服务端一次性崩溃**（非调度/配置导致）；`managed.restart_on_crash` 可兜底自动拉起。
证据：`docs/gpu-schedule-probe.json` 的 `turns20`、WER 报错（`Report.wer`，异常码 `c0000409`）。

### 6.5 `managed` 真实启停（`docs/qwen-runtime-probe.json`）

| 项 | 实测 |
| --- | --- |
| 启动（含模型加载 + `/health` 就绪） | **2.14–2.18 s**（8096，1.7B CV Q8_0） |
| 就绪后健康 | `Qwen3-TTS · model=qwen3-tts · sr=24000 · voices=9 · format=verified`（含 WAV 探针，1.55 s） |
| 合成冒烟 | 首块 **36–39 ms**、RTF **0.106–0.121**、音频 2.4–3.7 s |
| `status().artifacts` | `pid` / `started_at` / `model` / `codec` / `dlls{found: ggml.dll, ggml-base.dll, ggml-cpu.dll, ggml-cuda.dll, missing: []}` / host/port / log_path / tail；`commit` 来自 `providers.qwen3_tts.commit`（探针已填 `2edb35f`；配置留空时为 `null`） |
| 停止 | terminate→wait **153–155 ms**，`exit_code=1`（Windows TerminateProcess 的正常表现），端口释放 |
| 外部实例 | `mode=external` 时 `state=external`、`pid=None`、健康检查可用（`healthy=True`），**stop 后端口仍被占用 = 没有终止用户进程**（探针自己不会去杀它） |
| 失败诊断 | 缺模型 → `FAILED`+「找不到 Talker 模型文件：…」；端口占用 → `FAILED`+「端口 127.0.0.1:8096 已被占用（可能已有一个 tts-server 在跑…）」（**263 ms 内失败，不拉子进程**） |

### 6.6 取消后的 GPU 回落（直连 HTTP）

合成中客户端断开 → 服务端日志 `[Pipeline] on_chunk callback aborted the synthesis` 1 条/次；
GPU util 从 93–94% 回落到 2–4% 的中位耗时 **478.8 ms**（三次 338/479/585 ms；50 ms 请求采样实际 126 ms，
含 ±1 周期误差），断开后立即可用的 TTFA 24.5 ms。与 04D 的 ~0.5 s 同量级。

## 7. `sentence_interleave` 可行性审查（结论：unsupported）

问题：能否在 llama.cpp 生成中途**暂停**，把已生成的文本送去 TTS，然后再**续接同一个**
生成请求（不重复 prefill、不丢上下文）？

审查结论（按 `llama-server` 的 HTTP API 与本机实测字段）：

1. `POST /v1/chat/completions`（或 `/completion`）的流式响应是**单向输出**：服务端生成过程中
   没有"暂停/恢复"控制端点；客户端能做的只有"继续读"或"断开"。
2. 断开（取消）会结束该请求；没有"续接 token / 服务端会话句柄"这种东西。
   llama.cpp 的 slot/KV cache（`/slots`、`--slot-save-path`）是**请求之间**复用前缀，
   不是暂停正在生成的请求（`total_slots=1`、`-np 1` 时更不可能并行保留两个生成态）。
3. 因此"切句后暂停"只能退化成 **cancel + 重新请求**：
   - 重复 prefill：需要把"原始 prompt + 已生成文本"重新 prefill。本机实测 prefill 约
     **1000–1600 tok/s**（短 prompt；见 `docs/gpu-schedule-probe.json` 的 `prefill_tps_timings`），
     即已生成 200 token 时重算约 **+150–200 ms**，且每切一次句就多一次；
   - 上下文一致性：重新请求的采样状态（RNG/温度/重复惩罚）与原请求不同，输出会漂移
     （重复/遗漏/风格跳变）；
   - 额外延迟 + 过度占用：每次重算都要重新 prefill，LLM 侧的 decode 会被 TTS 打断多次，
     正好抵消 `llm_priority` 要保护的体验。
4. **结论**：本阶段不实现 `sentence_interleave`；配置层不接受该取值（`gpu.policy` 只允许
   `llm_priority|overlap`）。若未来 llama.cpp 提供"暂停/续接同一生成"的能力，再重新评估；
   在那之前不允许把"切句"描述成"可以暂停并续接 LLM 生成"。

## 8. 复现命令

```powershell
# 1) 起 llama-server（9B Q4_K_M，131072 ctx，端口 11435）
#    参考 F:\AI_Project\llama\start-heretic-9b-q4.bat
# 2) 起 TTS（external 模式）
& "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\src\build-cuda\Release\tts-server.exe" `
  --model "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\qwen-talker-1.7b-customvoice-Q8_0.gguf" `
  --codec "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\qwen-tokenizer-12hz-Q8_0.gguf" `
  --alias qwen3-tts --host 127.0.0.1 --port 8095 --lang Chinese --max-batch 1

# 3) 应用级调度测量（本仓库；每个场景写同一份 JSON，--merge 按 scenario 合并）
uv run python scripts/gpu_schedule_probe.py --scenario baseline --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario priority --baseline-tps <基线> --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario overlap  --baseline-tps <基线> --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario long     --max-tokens 700 --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario turns20  --merge --json docs/gpu-schedule-probe.json
uv run python scripts/gpu_schedule_probe.py --scenario cancel   --cancel-after-s 6.0 --merge --json docs/gpu-schedule-probe.json

# 4) 直连并发对照（1.7B TTS × 9B decode，不经过应用层）
uv run python scripts/tts_concurrency_probe.py

# 5) managed 真实启停（会真实启动/停止 tts-server.exe，默认 8096 端口避免与 external 8095 串台）
uv run python scripts/qwen_runtime_probe.py --port 8096 --json docs/qwen-runtime-probe.json --merge
uv run python scripts/qwen_runtime_probe.py --scenario missing-model --port 8096 --merge --json docs/qwen-runtime-probe.json
uv run python scripts/qwen_runtime_probe.py --scenario port-busy    --port 8096 --merge --json docs/qwen-runtime-probe.json
uv run python scripts/qwen_runtime_probe.py --scenario external     --port 8095 --merge --json docs/qwen-runtime-probe.json
```

**证据索引**

| 内容 | 路径 |
| --- | --- |
| 应用级调度证据（baseline/priority/overlap/long/turns20/cancel） | `docs/gpu-schedule-probe.json` |
| 直连 HTTP 并发/串行/取消证据 | `docs/tts-concurrency-probe.json` + `docs/tts-concurrency-probe.md` |
| managed 运行管理证据（含负路径与 external） | `docs/qwen-runtime-probe.json` |
| Provider 与显存基线（04D） | `docs/qwen3-tts-probe.json` / `docs/qwen3-tts-vram.json` |

## 9. 已知限制

- 显存预算只做"开始前检查"，不能阻止已开始的 TTS 生成；也不能强制让出 GPU。
- `WDDM` 下**进程级**显存不可用（`nvidia-smi` 只给整卡与 PID 列表，拿不到每进程 MB）：
  本文只记录"整卡 used"与"TTS 独立测量增量"，不伪造进程级数字。
- 监控长驻进程（managed 的巡检）只到"健康/进程存在"级别，不做 GPU 利用率曲线落盘；
  需要曲线时用探针脚本的 `nvidia-smi` 采样（见 `scripts/gpu_schedule_probe.py`）。
- 多卡未验证：`nvidia-smi` 采样与预算检查取第一块卡。
- 线上模型（不占本地 GPU）不受本调度门控；按 Provider 资源元数据区分，当前只有
  llama_cpp/mock 两种元数据。
- **TTS 服务端一次性崩溃（2026-11-18 实测）**：第一轮 20 轮连测在第 10 轮触发
  `tts-server.exe` 进程内 fail-fast（`0xc0000409` / `ucrtbase.dll`），客户端按设计降级为纯文字；
  重跑 20 轮未复现。属服务端风险，不是调度/配置导致；`managed.restart_on_crash` 可自动兜底。
- **detach（`stop_on_exit=false`）后子进程的 stdout 无人读取**：停止时 pump 任务被取消，
  保留实例后续日志不再落盘；若应用继续运行且子进程写满管道（约 64 KB）会阻塞在写日志。
  需要长期保留实例时，建议把 `managed.log_dir` 与外部日志重定向配合使用，或改为
  "spawn 时 stdout 直接接文件"（Phase 13 统一 ServiceManager 时一并处理）。
- **`mode=disabled` 的口径**：设置页与健康行按"语音未启用"处理（静音禁用、detail 写原因），
  与 `type=none` 同效，但配置仍保留 `type=qwen3_tts`，方便只切 `mode` 恢复。
- **TTS 侧并发代价与 04C 不一致**：04C 记录重叠时 TTS RTF 0.091→0.097（+6.6%），本测
  （1.7B CustomVoice + 9B Q4）两轮全量都是 0.087/0.092 → 0.281/0.291（3.2×），且服务端 `[Perf]` 显示
  Talker/CodePredictor 真的变慢（不是客户端读饥饿）。差异候选原因：Base vs CustomVoice 模型、
  llama.cpp 构建差异、Windows 调度状态；**尚无定论**。本文对 TTS 侧取"本测 0.28、04C 曾测几乎不变"。

## 10. 与后续阶段的接口

- **Phase 05（STT）**：录音/播报互斥仍在 Orchestrator；`SpeechState.WAITING` 只表示"等 LLM"，
  不要复用来表示"等麦克风"。
- **Phase 06（Tool Broker）**：工具活动不经过语音层；工具播报走同一条 `SpeechCoordinator` 入口，
  因此自动受 `llm_priority` 约束。
- **Phase 13（通用 ServiceManager）**：复用 `ServiceState/ServiceStatus/ManagedService`；
  `QwenTtsRuntime` 可作为参考实现，把 llama-server/STT 的生命周期接进同一套诊断口径。
