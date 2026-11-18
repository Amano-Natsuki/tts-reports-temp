# Qwen3-TTS GGML Provider（Phase 04D）

> 状态：**已接入为主候选 TTS 后端**（Phase 04D，2026-11-18）。默认配置仍是 `providers.tts.type=none`，
> 由用户显式启用；MOSS-TTS-Realtime 保留为 `experimental` 后端，代码与诊断资产未删除。
> 04C 判定：**CONDITIONAL GO** —— 性能/显存/稳定性/取消/并发门槛通过；**中文音色人工试听、
> 用户中文参考音频、情绪稳定控制仍未完成**（见 §9）。
> 上游依据：`04D_任务书_Qwen3-TTS_GGML_Provider正式接入.md`；04C 决策
> `F:\AI_Project\Qwen3-TTS-Experiments\docs\qwen3-tts-decision-04c.md` 及同目录三份报告。

## 1. 固定资产（冻结，不允许"升级到最新 commit"）

| 项 | 值 |
| --- | --- |
| 后端 | `qwentts.cpp` GGML CUDA，commit **`2edb35f`**（2026-11-17 10:26:48 +0100，*codec: run the causal conv1d as an implicit GEMM*） |
| 构建 | MSVC 2022（14.44）+ CUDA 12.9，`-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=89-real`，输出目录 `build-cuda/Release` |
| 可执行 | `F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\src\build-cuda\Release\tts-server.exe`（823,808 B） |
| Talker | `…\models\qwentts\qwen-talker-1.7b-customvoice-Q8_0.gguf`（2,042,834,304 B，`model_type=custom_voice`） |
| Codec | `…\models\qwentts\qwen-tokenizer-12hz-Q8_0.gguf`（291,150,624 B） |
| Low-VRAM 备用 | `qwen-talker-0.6b-customvoice-Q8_0.gguf`（968,588,544 B）/ `qwen-talker-0.6b-base-Q8_0.gguf` |
| Base / VoiceDesign（进阶） | `qwen-talker-1.7b-base-Q8_0.gguf` / `qwen-talker-1.7b-voicedesign-Q8_0.gguf` |

`tts-server.exe` 的运行时依赖（`dumpbin /dependents` 实测，与 04C 的猜测不同，以这里为准）：

```text
tts-server.exe
├─ ggml.dll              （直接依赖）
├─ ggml-base.dll         （直接依赖）
├─ ggml-cpu.dll          （ggml 运行时加载；同目录必须有）
├─ ggml-cuda.dll         （ggml 运行时加载；同目录必须有，34.9 MB）
├─ MSVCP140.dll / VCRUNTIME140.dll / VCRUNTIME140_1.dll（MSVC 2022 x64 运行库）
└─ Windows 系统 DLL（WS2_32/KERNEL32/api-ms-win-crt-*）
```

`qwen.dll` **不是** `tts-server.exe` 的依赖（`qwen-core` 是静态链接）；`qwen.dll` 是 `qwen-tts.exe`
等其它工具的动态库。只部署 `tts-server.exe` 时不要漏掉 `ggml-cpu.dll` / `ggml-cuda.dll`。

启动（external 模式，端口 8095）：

```powershell
& "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\src\build-cuda\Release\tts-server.exe" `
  --model "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\qwen-talker-1.7b-customvoice-Q8_0.gguf" `
  --codec "F:\AI_Project\Qwen3-TTS-Experiments\models\qwentts\qwen-tokenizer-12hz-Q8_0.gguf" `
  --alias qwen3-tts --host 127.0.0.1 --port 8095 --lang Chinese --max-batch 1
```

服务端 CLI（`tools/tts-server.cpp`）：`--model` / `--codec`（必填）、`--device`、`--alias`、
`--host`（默认 127.0.0.1）、`--port`（默认 8080）、`--lang`（默认 `auto`）、`--max-batch`（默认 1）、
`--no-fa`、`--clamp-fp16`、`--codec-chunk-dur`。**进程启动/停止不由本应用管理**（04E 的
ServiceManager 才做这件事），Provider 只连接已经在跑的 `base_url`。

## 2. HTTP 契约（`src/tts-server.h`）

```text
POST /v1/audio/speech
  body: {"input": "<非空文本>", "voice"?: "<speaker|registered>", "language"?: "Chinese|English|auto|…",
         "instructions"?: "<str>", "response_format": "pcm"|"wav", "speed"?: num, ...可选采样覆盖}
  200: Content-Type: audio/pcm；Transfer-Encoding: chunked；头只有 Cache-Control / X-Accel-Buffering
  400: {"error":{"message":"…","type":"invalid_request_error"}}   （JSON 非法/字段越界/未知 voice/Base+instruct）
  502: {"error":{"message":"…","type":"server_error"}}           （合成失败）

GET  /health              -> {"status":"ok"}
GET  /v1/models           -> {"object":"list","data":[{"id":"<alias>","object":"model","owned_by":"local"}]}
GET  /v1/audio/voices     -> {"voices":[{"name":"serena","kind":"speaker"}, …, {"name":"<cloned>","kind":"registered"}]}
POST /v1/audio/voices     -> {name, ref_text?, wav_b64? | (spk_b64 + rvq_b64)}（**仅 base 模型**，否则 400）
DELETE /v1/audio/voices/{name} -> {"status":"deleted"} / 404
```

必须记住的六件事：

1. **没有采样率/声道响应头**。格式来自约定（s16le / 24 kHz / mono）与 `expected_*` 配置；
   Provider 用健康检查里的 **WAV 探针**（`response_format=wav` 合成"你好"、解析 RIFF `fmt ` 头）
   显式核对 24 kHz / 1 声道 / 16 bit。不一致 → `healthy=False`，宁可不播也不播错速率。
2. **一次请求只能合成一段完整文本**（文本不支持增量追加）。多句话 = 多次请求，音频按顺序拼接。
3. **`max_batch=1`**：同一时刻只合成一路；客户端必须串行排队。
4. **取消 = 客户端断开连接**（没有 cancel 端点）。cpp-httplib 在写失败时置 `client_gone`，
   下一次 chunk 回调返回 false → 中止生成；服务端日志出现
   `[Pipeline] on_chunk callback aborted the synthesis (lane 0)`。
5. **`voice` 是名字**：CustomVoice 有 9 个内置 speaker；Base 模型没有 speaker 表，只认
   `POST /v1/audio/voices` 注册过的克隆音色。传未知名字 → 400 `unknown voice '<name>'`。
6. **`instructions` 只在 CustomVoice/VoiceDesign 上可用**；GGML Base 返回 400
   （`--instruct is not supported for base models`）。04C 判定情绪控制**不稳定**，Provider 只透传。

### 2.1 内置 speaker（CustomVoice，`GET /v1/audio/voices` 实测）

`serena` / `vivian` / `uncle_fu` / `ryan` / `aiden` / `ono_anna` / `sohee` / `eric` / `dylan`。
中文母语优先：**`vivian`（明亮年轻女声，04D 默认）**、`serena`（温柔年轻女声）、`uncle_fu`（低沉稳重男声）；
`dylan`（北京话）/ `eric`（四川话）是方言预设，只在 `--lang chinese|auto` 时触发。

## 3. Provider 设计与映射

`src/desktop_companion/providers/qwen3_tts.py`：`Qwen3TTSProvider`（`name="qwen3_tts"`），
只依赖 `httpx` + `core`，不加载权重、不管理进程。

| `SpeechStream` 动作 | 实际 HTTP |
| --- | --- |
| `provider.open_stream(req)`（`initial_text` 非空） | 立刻 `POST /v1/audio/speech`，**等到响应头**才返回（400/坏 Content-Type/502 在这里就抛） |
| `push_text(seg)` | 不做 IO：段文本进 asyncio 队列（服务端不支持增量文本，所以每段各自一次请求） |
| `audio()` | 逐段取锁 → 发请求 → 读 `audio/pcm` 字节流 → 聚合成整帧块（`chunk_target_ms=80` → 3840 B）→ 顺序 yield |
| `push_text("", final=True)` | 入队结束哨兵；最后一段读完后 `audio()` 正常结束 |
| `stream.aclose()` / `provider.aclose()` | 关闭当前响应（= 服务端取消）+ 唤醒等待中的队列；幂等、不抛 |
| `health_check()` | `GET /health` + `/v1/models` + `/v1/audio/voices` + WAV 格式探针；TTL 缓存，不抛 |

- **串行锁**：Provider 级 `asyncio.Lock`，等锁超过 `busy_timeout_s` → `tts_busy`（可重试）。
- **三段超时**：等响应头 `request_timeout_s`；块间空闲 `stream_idle_timeout_s`；
  单段音频总预算 `audio_total_timeout_s`（都按"段"计，不是整轮）。
- **帧对齐**：跨 wire 块半帧拼接；流末尾不足一帧丢弃并 warning（与 Phase 04 一致）。
- **稳定错误码**：`tts_unreachable / tts_timeout / tts_http_error / tts_unavailable /
  tts_invalid_response / tts_invalid_format / tts_closed / tts_stream_interrupted / tts_text_empty /
  tts_unknown_voice / tts_instruct_unsupported / tts_invalid_request / tts_synthesis_failed / tts_busy`。
- **统计**（`stream.stats`，非协议字段，日志/probe 用）：`requests / text_chars / chunks / bytes /
  audio_ms / synth_ms / rtf / first_byte_ms / first_chunk_ms / per_request[]`。
  三个时间戳口径分开：`first_byte_ms`（请求发出 → 首个原始字节）、`first_chunk_ms`（→ 首个整帧块
  yield 给协调器）、首帧送入播放器由协调器的 `speech.playback.started` 记录（Provider 不猜）。

## 4. 配置

```yaml
providers:
  tts:
    type: qwen3_tts            # 显式启用（默认仍是 none）
    base_url: http://127.0.0.1:8095
    voice: vivian              # speaker 名或已注册的克隆音色名（**不是**参考音频路径）
    language: Chinese          # 透传；null = 不发该字段（服务端 --lang 默认 auto）
    start_prebuffer_ms: 300    # 按后端覆盖 audio.start_prebuffer_ms（MOSS 的 2500 不适用于 Qwen）
  qwen3_tts:
    mode: external             # managed（进程托管）属于 04E，本阶段报 ConfigError
    commit: 2edb35f            # 记录冻结 commit（诊断/文档用）
    expected_model: qwen3-tts  # 与 /v1/models 的 id 核对，不一致只记 note
    model_type: custom_voice   # base 时拒绝 instructions；unknown 时交给服务端 400 映射
    connect_timeout_s: 5.0
    request_timeout_s: 30.0
    stream_idle_timeout_s: 30.0
    audio_total_timeout_s: 120.0
    health_timeout_s: 3.0
    probe_ttl_s: 30.0
    busy_timeout_s: 30.0
    expected_sample_rate: 24000
    expected_channels: 1
    chunk_target_ms: 80.0
    format_probe: true         # 健康检查用 WAV 头核对格式（服务端没有格式响应头）
    validate_voice: true       # 开流前核对 voice 是否在 /v1/audio/voices 里
    instructions: null         # 仅 CustomVoice；experimental，默认空
    max_batch: 1               # 记录用；Provider 仍串行
```

配置纪律（`extra="forbid"`，错误在加载期就报）：

- `type=qwen3_tts` 必须给 `base_url`（代码里没有默认端口）；
- `voice` 不能是 `*.mp3/*.wav` 或带路径——那是 MOSS 的"参考音频"语义，Qwen 会回 400 `unknown voice`；
- `audio.start_prebuffer_ms` 全局默认不动；Qwen 用 `providers.tts.start_prebuffer_ms` 单独覆盖，
  这样仍保留的 MOSS 实验后端还能用 2500 ms。

## 5. 声音：预设、克隆、情绪

**默认（无需用户录音）**：`voice: vivian`（或 `serena` / `uncle_fu`）。

**克隆音色（Base 模型，需要用户授权的中文参考音频）**：

```powershell
# 5–15 s、≥24 kHz、单声道、干净的 WAV；ref_text 必须人工逐字校对（ICL 必需）
uv run python scripts/qwen_voice_register.py --ref-wav D:\voice\my_ref.wav --name my_voice --ref-text "参考音频的逐字文本"
# 成功后会提示 providers.tts.voice: my_voice
```

脚本做路径规范化、存在性、`.wav` 限制、时长/采样率/声道检查与 `--dry-run`；注册走
`POST /v1/audio/voices`（Base 专属，CustomVoice 会 400）。04C 明确：**没有中文参考 + 校对文本之前，
不要把 Base 设成默认**（自动英文转写会泄漏/漏字，CER 4.8–14.3%）。

**VoiceDesign（造音色）**：`qwen-talker-1.7b-voicedesign-Q8_0.gguf` + `instructions` 描述；
04C 只验证了"描述对 F0 有响应"，音色一致性/主观质量未验收，**不作为默认**。

**情绪**：CustomVoice 的 `instructions` 有真实响应（`fast` 时长 −25%、`slow` +102%、`angry` F0 +48%），
但 `ryan + angry/slow` 出现音色漂移/超长（16.72 s），人工未试听 → 04D 只保留**透传**能力：
`instructions` 默认空、文档/配置注明 experimental，不宣传"稳定情绪控制"。Base + instruct 被 Provider
直接拒绝（`tts_instruct_unsupported`），不要用提示词包装。

## 6. 04D 实测（Windows 10 19045 / RTX 4090 24 GB / i7-13700KF，真实 8095 服务）

### 6.1 健康与探针（`docs/qwen3-tts-probe.json`）

```text
health: healthy=True detail="Qwen3-TTS · model=qwen3-tts · sr=24000 · voices=9 · format=verified" (27.3 ms)
cancel: 读 2 块后关闭连接，cancel_ms=34.3；服务端日志 "[Pipeline] on_chunk callback aborted the synthesis"
after_cancel: ok=True ttfb=97.5 ms（下一请求立即可用，4.64 s 音频 / 58 块）
single_1: audio 13.68 s / 171 块 / ttfb 15.2 ms / synth 1214.9 ms / rtf 0.09
single_2: audio 13.84 s / 173 块 / ttfb 16.1 ms / synth 1229.7 ms / rtf 0.09
single_3: audio 14.08 s / 176 块 / ttfb 14.3 ms / synth 1249.8 ms / rtf 0.09
multi_segment: 3 段 → 3 次请求 / 201 块 / 16.08 s / rtf 0.09
块大小：3840 B（= chunk_target_ms 80 × 24 kHz × 2 B），无半帧（min=max=avg=3840）
```

验收对照：**暖机 TTFA ≤800 ms** → 实测 14.3–16.1 ms（健康检查含 WAV 探针 27.3 ms）；
**RTF_project ≤0.7** → 实测 0.09–0.10。显著优于 04C 的 15–42 ms / 0.09–0.10（同量级）。

### 6.2 10 组中文样例（`docs/tts-samples-qwen/`，`--sink player --prebuffer-ms 300`）

| # | 场景 | 切段 | 音频 | TTFB | 欠载 | 播放 |
| --- | --- | --- | --- | --- | --- | --- |
| 01 | 问候 | 3 | 5.6 s | 14.6 ms | 0 | 5.6 s |
| 02 | 情绪 | 3 | 6.6 s | 29.8 ms | 0 | 6.6 s |
| 03 | 长句 | 4 | 13.1 s | 30.5 ms | 0 | 13.1 s |
| 04 | 数字与单位 | 3 | 10.5 s | 29.3 ms | 0 | 10.5 s |
| 05 | 中英混排 | 3 | 10.6 s | 110.9 ms | 0 | 10.6 s |
| 06 | 疑问与感叹 | 3 | 4.3 s | 31.3 ms | 0 | 4.3 s |
| 07 | 绕口令 | 6 | 11.0 s | 30.3 ms | 0 | 11.0 s |
| 08 | 技术术语 | 4 | 10.4 s | 30.0 ms | 0 | 10.4 s |
| 09 | 标点与停顿 | 4 | 13.3 s | 110.0 ms | 0 | 13.3 s |
| 10 | 多句连读 | 5 | 13.3 s | 101.0 ms | 0 | 13.3 s |

> 表里的 `rtf≈1.0` 是**整轮含实时播放**的比率（`--sink player` 有背压，合成被播放拖住）；
> 真正的合成 RTF 看 §6.1 的 provider 统计（0.09–0.10）。10 组全部 `underruns=0`、`dropped=0`。
> WAV 是 `--sink player` 实际消费的 PCM；试听结论列留空（见 §9）。

### 6.3 长语音连续播放（15 s 以上）

同一套 `SpeechCoordinator + AudioPlayer`（`start_prebuffer_ms=300`，静音消费 → 只测时序）喂 15 句 ≈ 109 s
音频：`played_ms=109360`、`underruns=0`、`dropped_chunks=0`、墙钟 110.6 s（≈ 实时 + 1.2 s）。
即 **`chunk_target_ms=80` + 300 ms 预缓冲在单后端场景足够连续**，没有爆音/错序/串轮。

### 6.4 显存（`docs/qwen3-tts-vram.json`，nvidia-smi 采样）

| 状态 | 整卡已用 | 相对基线增量 |
| --- | --- | --- |
| 服务未启动（基线） | 1,848 MiB | — |
| 1.7B CustomVoice 加载后（常驻） | 4,953 MiB | **+3,105 MiB ≈ 3.03 GB** |
| 合成期间峰值 | 5,059 MiB | **+3,211 MiB ≈ 3.14 GB** |

验收：常驻 ≤4 GB、峰值 ≤5 GB → **通过**；与 04C 的 3,027 MiB 常驻增量一致（±80 MiB 采样噪声）。

### 6.5 取消 = 真停

客户端读到 8 块（约 0.2 s 音频）后断开：GPU 利用率 `95% → 54% → 3%`，
**断开后约 0.5 s 内回落到 ≤10%**（采样间隔 50 ms、nvidia-smi 每次约 100 ms，实际比 04C 的 313 ms 慢一档但同量级）；
服务端日志立刻出现 `on_chunk callback aborted the synthesis`；随后新请求正常（3.76 s 音频 / 47 块）。

### 6.6 完整 UI 链路（`scripts/speech_smoke.py --tts-type qwen3_tts`）

```text
状态序列：idle → synthesizing → playing → idle
timeline_ms：首文本 delta 492 ms / synthesizing 597 ms / playing 1238 ms
tts_health：qwen3_tts · model=qwen3-tts · sr=24000 · voices=9 · format=verified · type=custom_voice
leftover_tasks=()；shutdown errors=()（干净退出）
```

## 7. 复现命令

```powershell
# 1) 启动服务（见 §1），确认健康
curl.exe -s http://127.0.0.1:8095/health

# 2) Provider 契约测试（72 项，不联网）
uv run pytest tests/test_qwen3_tts_provider.py -q

# 3) 真实服务探针（健康/单段×3/多段×3/取消+复测）
uv run python scripts/tts_probe.py --tts qwen --runs 3 --json docs/qwen3-tts-probe.json

# 4) 10 组中文样例（无 --play 时用 null sink 按实时消费，可测 underrun）
uv run python scripts/tts_samples.py --tts qwen --sink player --prebuffer-ms 300
#   真机试听：加上 --play --out-dir docs/tts-samples-qwen

# 5) 显存：先停服务记基线，再起服务测增量
uv run python scripts/gpu_mem_probe.py --tts qwen --baseline-only --json docs/qwen3-tts-vram.json
uv run python scripts/gpu_mem_probe.py --tts qwen --skip-llm --json docs/qwen3-tts-vram.json

# 6) 完整 UI 链（offscreen；--mute 无声音但仍走 Player）
uv run python scripts/speech_smoke.py --tts-type qwen3_tts --mute --expect-speech

# 7) 克隆音色注册（Base 模型 + 用户授权 WAV）
uv run python scripts/qwen_voice_register.py --ref-wav <wav> --name <name> --dry-run
```

## 8. 与其它模块的边界

- **主进程不加载权重、不启动/停止服务**：Provider 只走 HTTP；进程管理/GPU 调度是 04E。
- **`SpeechCoordinator` / `AudioPlayer` / `AudioFormat` / 事件协议未改**：Qwen 只是新的
  `SpeechProvider` 实现；MOSS 与 Mock 仍可用。
- **预缓冲按后端区分**：`providers.tts.start_prebuffer_ms` 覆盖全局 `audio.start_prebuffer_ms`，
  不改 MOSS 的历史配置。
- **MOSS 的实验标记不变**：`providers.tts.type=moss_tts` 仍会打 experimental warning。

## 9. 已知限制与未验证事项（如实记录）

1. **中文音色人工试听未完成**：`docs/tts-samples-qwen/record.md` 的试听列留空，必须由用户本人听。
   04C 的 29 条匿名样本同样待听——**在此之前"音质验收通过"不成立**（判定仍是 CONDITIONAL GO）。
2. **情绪控制不稳定**：`instructions` 只在 CustomVoice 上有响应，`ryan + angry/slow` 有漂移/超长证据；
   Provider 只透传、标 experimental，不承诺稳定情绪。
3. **中文参考音频/ICL 未验收**：Base 路线需要用户录 5–15 s 中文参考 + 人工校对 `ref_text`；
   `scripts/qwen_voice_register.py` 已可用，但真实注册/克隆效果未在本轮验证（需要 Base 模型与授权素材）。
4. **服务端不提供音频格式头**：格式靠约定 + WAV 探针；探针本身要合成一句"你好"（首次健康检查多花
   约 0.3–0.5 s），且与合成共享串行锁——服务忙时探针会延后（记 `format=unverified`，不判不健康）。
5. **`max_batch=1`**：并发合成未验证；Provider 串行排队，等待超时报 `tts_busy`。
   与本地 LLM 的 GPU 竞争（04C：LLM Decode −61%）属于 04E 的调度决策，本阶段不在 Provider 里处理。
6. **MOSS 的静音冒烟是既有问题**：`speech_smoke.py --mute` 在 Phase 04 的 MOSS/Mock 路径上同样
   会超时（本次回归对照：`--tts-type mock --mute` 也 FAILED），**与本 Provider 无关**；
   真机验收用非静音模式通过（§6.6）。
7. **未提交大文件**：模型 GGUF、`tts-server.exe`、DLL 都在仓库外（`F:\AI_Project\Qwen3-TTS-Experiments`）；
   仓库只记录路径、commit 与启动命令。

## 10. 给 04E 的接口

- `providers.qwen3_tts.mode: external|managed` 已预留；`managed` 现在会明确报 `ConfigError`
  （"04E 才实现"），04E 实现进程托管时只需替换 Provider 构造期分支，不动请求/音频路径。
- 健康检查可直接做服务探针（TTL 缓存、带 WAV 格式核对）；`QwenStreamStats` 提供每个请求的
  首字节/首块/合成耗时，可用于调度记录。
- `busy_timeout_s` / `max_batch` 是调度接入点：`llm_priority` 需要在发起 TTS 前检查 LLM 状态，
  该逻辑应放在 04E 的 Scheduler，而不是 Provider（Provider 只保证"同一时刻一路合成"）。

## 11. 证据索引

| 内容 | 路径 |
| --- | --- |
| 真实服务探针 JSON | `docs/qwen3-tts-probe.json` |
| 10 组样例（WAV + `samples.json` + `record.md`） | `docs/tts-samples-qwen/` |
| 显存测量 | `docs/qwen3-tts-vram.json` |
| 契约（实现/测试基准） | `.dsh/notes/phase04d-contract.md` |
| 04C 决策与三份报告 | `F:\AI_Project\Qwen3-TTS-Experiments\docs\qwen3-tts-decision-04c.md`、`qwen3-tts-voice-quality.md`、`qwen3-tts-emotion.md`、`qwen3-tts-concurrency.md` |
| 假服务器与 72 项契约测试 | `tests/fake_qwen_tts_server.py`、`tests/test_qwen3_tts_provider.py` |
