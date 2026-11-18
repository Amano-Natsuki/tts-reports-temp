# Qwen3-TTS 实测报告（Windows 10 + RTX 4090，2026-11-18）

> 全部数字为**本机实测**，不是社区 benchmark 的转述。原始数据：`benchmarks/*.json`；
> 声音样本：`samples/`；环境版本：`environment/versions.json`；调研与仓库对比：`QWEN3_TTS_RESEARCH.md`。
> 口径：`RTF_project = 合成墙钟 / 音频时长`（>1 慢于实时）。社区 README 的 "RTF 4.78x" 是倒数口径，
> 换算后 `RTF_project = 1/4.78 = 0.209`；本文若引用会同时写两个方向。

---

## 0. 一页结论

| 后端 / 模型 | 常驻显存增量 | 生成峰值增量 | 暖机 TTFA | `RTF_project` | 零欠载 | 可取消 | CER | 音色余弦 |
| --- | ---: | ---: | ---: | ---: | --- | --- | ---: | ---: |
| **GGML qwentts.cpp / 0.6B Q8_0** | **2.55 GB** | **2.61 GB** | **11–36 ms**（中位 ~18） | **0.075–0.080** | ✅ 全部 | ✅ 断连即停 | 1.44% | 0.530 |
| **GGML qwentts.cpp / 1.7B Q8_0** | **3.57 GB** | **3.62 GB** | **20–37 ms** | **0.091–0.115** | ✅ 全部 | ✅ 断连即停 | 1.44% | 0.543 |
| Torch CUDA Graph / 0.6B bf16 | 3.58 GB | 3.74 GB | 257–286 ms | 0.228–0.248 | ✅ 全部 | ✅ 服务端协作取消 195 ms | 2.27% | 0.541 |
| Torch CUDA Graph / 1.7B bf16 | **5.06 GB** ⚠️ | 未单独采样 | 302–327 ms | 0.273–0.287 | ✅ 全部 | ✅ 同上 | 1.87% | 0.556 |
| 对照：MOSS-TTS-Realtime（项目基线） | ≈12 GB | ≈12.3 GB | 417–1185 ms | **1.24–1.95** | ❌ 长句 17/50 次 | ❌ 无请求级 abort | **36.6%** ⚠️ | 0.555 |

**结论**：GGML（qwentts.cpp）1.7B Q8_0 **同时满足**四条硬指标——显存 3.57 GB（≤4 GB）、
`RTF_project` 0.09（≤0.7）、TTFA ~30 ms（≤800 ms）、零欠载；且是唯一"断连即真停"的后端。
Torch 路线 0.6B 也达标，但 TTFA/RTF 差 10 倍量级，且 1.7B 会越过 5 GB 峰值预算。

> ⚠️ 关于 MOSS 的 36.6% CER：见 §5 的解释——那是 **ASR 复听 MOSS 已经保存的 10 个 WAV** 得到的，
> 其中 5 条明显不可懂（s07 绕口令、s10 多句连读转录出来是拉丁字母乱码），与项目记录的
> RTF>1 + 长句欠载一致。声速/时长不同会让 CER 偏高，但 36% 不是"口径问题"能解释的量级。

---

## 1. 测试环境与时间线

| 项 | 值 |
| --- | --- |
| GPU | RTX 4090 24,564 MiB（sm_89），驱动 621.14，CUDA UMD 13.5 |
| 桌面空闲整卡显存 | **1108 MiB**（无 TTS、无 LLM） |
| 测量窗口 | 2026-11-18 09:04–10:45（+08:00） |
| 资源协调 | 04A 会话在 08:28–08:55 占用 MOSS + LLM（显存 22.4 GB / GPU 97%），小希**等其释放后**才做独占测量；09:20 之后的数字均为干净基线 |
| 参考音色 | `prompt_audio1.mp3`（桌面助手当前 MOSS 音色；**英文**、11.73 s），转 24 kHz 单声道 WAV；ref_text 用 FunASR Paraformer 本地自动转写（**标注为自动生成**，未人工校对） |
| 克隆模式 | 主测 **x-vector-only**（跨语种推荐路径）；ICL 模式单独验证见 §7 |
| 文本集 | 项目 `docs/tts-samples/` 的 10 组固定中文样例 + 20 句长文本（457 字） |
| 每项重复 | 主矩阵 3 次、1.7B 2 次、SSE 逻辑块 2 次；报告取中位数 |

**坐标**：`RTF_project` = 流式请求从发出到流结束的墙钟 ÷ 收到的 PCM 折算出的音频时长。
`TTFA` = 请求提交 → **第一个音频字节**到达（不是响应头）。块间隔 = 相邻两次读到的音频字节时间差。

---

## 2. 冷启动 / 暖机 / 首包

| 阶段 | Torch 0.6B | Torch 1.7B | GGML 0.6B Q8_0 | GGML 1.7B Q8_0 |
| --- | ---: | ---: | ---: | ---: |
| 模型加载 | 5.43 s | 5.99 s | 进程内（日志：权重 1.16 GB） | 进程内（权重 2.2 GB） |
| CUDA Graph 捕获 | 0.75 s | 0.76 s | 不适用（无 graph） | 不适用 |
| 首次参考音色暖机 | 1.33 s | 1.56 s | 1.18 s（voice 注册，含编码器懒加载 132 ms） | 1.16 s |
| 暖机后 TTFA | 257–286 ms | 302–327 ms | **11–36 ms** | **20–37 ms** |
| 服务端自报 TTFA（首帧 codes） | — | — | 9–12 ms | 20–38 ms |
| 首块音频时长 | 960 ms（chunk=12） | 960 ms | **80 ms**（1 帧） | **80 ms** |

GGML 服务端日志（`[Perf]`）逐项：prefill 3–6 ms、TalkerDecode 1.35–1.45 ms/帧、
CodePredictor 4.1–4.2 ms/帧、CodecDecode 0.4 ms/帧、合计 **5.56 ms/帧**（≈179 帧/s）。

---

## 3. 流式能力：只有 A，没有 B

| | 事实 |
| --- | --- |
| **A. 音频输出流式** | ✅ 两个后端都是真流式。GGML 逐帧 yield（80 ms/帧），Torch 每 12 帧 yield（960 ms/块）。首块远早于整段完成（例：GGML 0.6B 整段 5.52 s，首帧 11 ms 到达；Torch 0.6B 整段 5.12 s，首块 286 ms 到达）。 |
| **B. 文本增量输入流式** | ❌ **两者都不支持**。`faster_qwen3_tts.model._prepare_generation` 一次性 `_tokenize_texts(input_texts)`；qwentts.cpp 的 `[Prompt] Built: N ids` 也是每次请求整段构建。没有任何"追加文本"的端点/参数。 |
| 结论 | 桌面助手必须沿用 `SpeechCoordinator` 的"语义切分 → 每片一次请求 → 音频队列顺序播放"。 |

块节奏（GGML，逻辑块 = 1 帧，SSE 之外的 PCM 探针按 4096 字节读块统计，间隔中位为 0 是因为
生成远快于消费、读缓冲被一次填满；下表用 **P99/max** 看抖动）：

| 配置 | 块数/整段 | 块音频中位 | 块间隔 P50 | P95 | P99 | max |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| GGML 0.6B（s03 11.9 s） | 149 | 80 ms | ~48.1 ms | 48.4 | 48.3 | 49.0 ms |
| GGML 1.7B（s03 11.1 s） | 139 | 80 ms | ~57.6 ms | 57.7 | 57.6 | 58.1 ms |
| Torch 0.6B（SSE 逻辑块，s10 10.3 s） | 10 | 960 ms | 228.7 ms | 229.5 | 230.2 | **230.3 ms** |

Torch 的 SSE 逻辑块（每块 960 ms）间隔极其稳定：**P99 与 max 只差 0.1 ms**。

---

## 4. 性能总表（10 组样例，中位数）

### 4.1 GGML 0.6B Q8_0（推荐）

| # | 场景 | TTFA ms | 墙钟 ms | 音频 ms | RTF_project | RTF 倒数 | 欠载 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 01 | 问候 | 34.6 | 492.7 | 5440 | 0.080 | 12.50 | 0 |
| 02 | 情绪 | 15.4 | 426.9 | 5680 | 0.077 | 13.00 | 0 |
| 03 | 长句 | 24.7 | 903.9 | 11920 | 0.076 | 13.10 | 0 |
| 04 | 数字与单位 | 17.7 | 805.1 | 10640 | 0.076 | 13.19 | 0 |
| 05 | 中英混排 | 11.9 | 659.4 | 8560 | 0.075 | 13.31 | 0 |
| 06 | 疑问与感叹 | 12.1 | 317.2 | 3920 | 0.077 | 12.99 | 0 |
| 07 | 绕口令 | 31.5 | 1013.4 | 13520 | 0.076 | 13.11 | 0 |
| 08 | 技术术语 | 11.5 | 742.6 | 9840 | 0.075 | 13.25 | 0 |
| 09 | 标点与停顿 | 11.9 | 818.3 | 10880 | 0.076 | 13.23 | 0 |
| 10 | 多句连读 | 17.9 | 837.8 | 11120 | 0.075 | 13.36 | 0 |

### 4.2 Torch CUDA Graph 0.6B bf16

| # | TTFA ms | 墙钟 ms | 音频 ms | RTF_project | 欠载 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 01 | 286.3 | 1271.2 | 5120 | 0.248 | 0 |
| 02 | 269.4 | 1444.8 | 5920 | 0.244 | 0 |
| 03 | 275.6 | 2886.8 | 12560 | 0.230 | 0 |
| 04 | 279.1 | 2174.3 | 9520 | 0.230 | 0 |
| 05 | 260.6 | 1999.2 | 8720 | 0.230 | 0 |
| 06 | 257.7 | 882.1 | 3680 | 0.237 | 0 |
| 07 | 276.2 | 2170.0 | 9440 | 0.230 | 0 |
| 08 | 274.9 | 2279.1 | 9920 | 0.231 | 0 |
| 09 | 278.8 | 2352.1 | 10240 | 0.230 | 0 |
| 10 | 256.7 | 2248.1 | 9840 | 0.228 | 0 |

### 4.3 1.7B

| 配置 | TTFA ms（范围） | RTF_project（范围） | 备注 |
| --- | --- | --- | --- |
| GGML 1.7B Q8_0 | 20.2–37.2（首条 131.8） | 0.091–0.115 | 首条偏慢是 KV 冷分配；稳态 ~0.093 |
| Torch 1.7B bf16 | 302–327 | 0.273–0.287 | 比 0.6B Torch 慢约 20% |

### 4.4 长文本（20 句 / 457 字）

| 配置 | 音频 | 墙钟 | RTF_project | TTFA | 欠载 |
| --- | ---: | ---: | ---: | ---: | ---: |
| Torch 0.6B | 77.96 s | 17.98 s | 0.231 | 285.5 ms | 0 |
| GGML 0.6B | **75.68 s** | **5.83 s** | **0.077** | 18–287 ms（首条冷 287 / 稳态 18） | **0** |

> 20 句 / 457 字一次请求跑完：Torch 17.98 s、GGML 5.83 s，均**零欠载**。
> 457 字 < `max_seq_len=2048`（约 170 s 音频上限），说明 15 s 以上长句与 10–20 句连读
> 都无需切句即可跑通；超过 ~170 s 才必须按句切开。

---

## 5. 显存（整卡 `nvidia-smi`，桌面空闲 1108 MiB）

| 阶段 | 整卡 MiB | 相对桌面空闲 |
| --- | ---: | ---: |
| 桌面空闲（无 TTS/LLM） | 1108 | — |
| Torch 0.6B 常驻 | 4691 | **+3583** |
| Torch 0.6B 生成峰值 | 4852 | +3744 |
| Torch 1.7B 常驻 | 6169 | **+5061** ⚠️ 超 5 GB 峰值预算 |
| GGML 0.6B 常驻 | 3657 | **+2549** |
| GGML 0.6B 加载峰值 | 3716 | +2608 |
| GGML 1.7B 常驻 | 4681 | **+3573** |
| GGML 1.7B 生成峰值 | **4723** | **+3615** |
| LLM 单独（Qwen3.5-9B Q4_K_M, ctx131072, KV q8_0） | 9253 | +8145 |

> WDDM 下 `nvidia-smi --query-compute-apps` 的进程级显存全是 `[N/A]`（本机实测），
> 因此用"整卡 used 的相位差"而不是进程级读数（`harness/vram_sampler.py` 已把
> `apps.memory_available=false` 写进每份 JSON，避免下游误读为 0）。

### 5.1 LLM + TTS 并发（9B 解码 + TTS 生成）

| 指标 | 单独 | 并发 | 变化 |
| --- | ---: | ---: | ---: |
| **GGML 0.6B** TTS TTFA | 11.28 ms | 12.99 ms | +1.7 ms（+15%） |
| GGML TTS RTF_project | 0.075 | 0.075 | 0 |
| GGML TTS 欠载 | 0 | 0 | — |
| LLM 首 token | 55.96 ms | 55.83 ms | ≈0 |
| **LLM 解码速度** | **133.2 tok/s** | **56.4 tok/s** | **−57.7%** |
| 整卡峰值 | 11,792 MiB | 11,792 MiB | 0（两者同时在显存里） |
| **Torch 0.6B** TTS TTFA | 306.6 ms | 295.8 ms | −3.5% |
| Torch TTS RTF_project | 0.249 | 0.244 | −2% |
| LLM 解码速度 | 134.7 tok/s | 49.7 tok/s | **−63.1%** |

**读法**：TTS 几乎不受影响（它是一串短促的 CUDA kernel，插空就能跑），
但 **LLM 解码掉 58–63%**——这是"边播边聊"时用户真正能感到的代价。
显存上两者共存没问题（11.8 GB / 24.5 GB），真正的约束是**带宽/SM 抢占**，不是容量。

---

## 6. 声音质量（客观代理 + 试听清单）

方法：FunASR Paraformer-large（CPU）复听生成的 WAV 算 **CER**（仅比较汉字/数字，忽略标点与拉丁转写），
CAM++（`iic/speech_campplus_sv_zh-cn_16k-common`）算与参考音色的 **余弦相似度**。
两条都是**代理指标**，最终仍需人耳；WAV 全部保留在 `samples/`。

| 配置 | CER 均值 | CER 最大 | 音色余弦均值 | 余弦范围 | 备注 |
| --- | ---: | ---: | ---: | --- | --- |
| GGML 0.6B Q8_0 | **1.44%** | 5.1% | 0.530 | 0.429–0.592 | 混排/术语最高（拉丁缩写） |
| GGML 1.7B Q8_0 | **1.44%** | 5.1% | 0.543 | — | |
| Torch 0.6B bf16 | 2.27% | 9.1% | 0.541 | 0.416–0.611 | |
| Torch 1.7B bf16 | 1.87% | 5.1% | **0.556** | 0.475–0.653 | 1.7B 略优 |
| 对照：MOSS 项目样例 | **36.6%** | 100% | 0.555 | 0.260–0.714 | s07/s10 不可懂 |

**普通话发音**：0.6B/1.7B 的 CER 都在 1.5–2.3%，绕口令（四是四…）Qwen3 全部 0%；
MOSS 同一句转录失败（CER=100%）。数字与单位（十二点五秒/两万四千赫兹）CER 2.4%，
主要错在"二十四千赫兹"读法差异，不是漏读。

**音色相似度**：约 0.53–0.56。参考音色是**英文**女声，目标是中文——跨语种克隆本身会掉分；
MOSS 的 0.555 与本机 Qwen3 的 0.53–0.556 基本同级，说明"像不像"两者没有代差。

### 6.1 情绪/语气指令（Base + xvec，同句 5 个变体）

| 指令 | 音频时长 | 相对基线 | RTF_project |
| --- | ---: | ---: | ---: |
| 无 | 6.32 s | — | 0.273 |
| 用非常缓慢、低沉的语速朗读 | 6.24 s | −1.3% | 0.236 |
| 用非常快、兴奋的语速朗读 | **4.08 s** | **−35.4%** | 0.240 |
| 用温柔的气声、耳语般的方式朗读 | 5.92 s | −6.3% | 0.236 |
| 用愤怒、严厉的语气朗读 | 5.68 s | −10.1% | 0.235 |

**结论**：Base 的 `instruct` 是"有反应但不稳定"——"快"生效明显（时长掉 35%），
"慢"几乎无效（−1.3%）。这与上游文档一致（Base 的 instruct 是实验性；稳定指令控制要用
CustomVoice/VoiceDesign 的 1.7B）。**不能把"模型有情绪"当成"能稳定服从情绪指令"。**

---

## 7. ICL（上下文克隆）与 x-vector 的差别

- **x-vector-only**（主测）：不要求 ref_text，跨语种不串音，但音色细节少；
- **ICL**（`non_streaming_mode=False` + ref_text）：把参考音频的 codec 也放进上下文，
  音色更贴，但**必须有 ref_text**；本任务参考音频是英文且无官方文本，自动转写有错字，
  因此 ICL 质量受限（🅐 见 `samples/` 与 `research/` 里的说明）。
- 实测：Torch 的 `_voice_prompt_cache` 会把 (路径, ref_text, xvec_only) 缓存住，
  第一次 1.33–1.56 s，之后 0 开销；qwentts.cpp 的两级缓存（`.spk/.rvq` 文件 +
  `CODEC_SNAP_SLOTS` LRU）同理，voice 注册 1.18 s、后续请求无感。

---

## 8. 取消 / 打断（barge-in）

| 后端 | 机制 | 实测 |
| --- | --- | --- |
| **GGML qwentts.cpp** | 客户端断开 → chunked sink 返回 false → 生成停止 + releaser join | 读到 1.02 s 音频后断开：GPU util 310 ms 时 93%（在收尾当前帧）→ **612 ms 起 4–5%**；随后新请求 TTFA 81.6 ms（无排队/僵尸） |
| **Torch + 本实验服务器** | 显式 `POST /tts/cancel/{rid}` → 生成循环检查 stop_event | 2.07 s 时取消 → 请求 200 → **195 ms 后流结束**，取消后只多收 64 ms 音频（1 块在途） |
| Torch 上游示例 server | 无取消，断连后继续跑完整段 | 读代码确认（`examples/openai_server.py` 无界 queue + daemon 线程） |

---

## 9. 复现与原始数据

```powershell
# 0. 环境
environment\install.ps1            # torch cu128 + faster-qwen3-tts + transformers 5.15.1
tools\fetch_models_ms.ps1 -Model 0.6B
tools\fetch_models_ms.ps1 -Model 1.7B
# 1. Torch 服务
run\start_torch_server.ps1 -Model ...\models\Qwen3-TTS-12Hz-0.6B-Base -Port 8091
# 2. GGML 服务（需先 cmd /c run\build_qwentts_windows.cmd）
models\qwentts\src\build-cuda\Release\tts-server.exe --model ...\qwen-talker-0.6b-base-Q8_0.gguf --codec ...\qwen-tokenizer-12hz-Q8_0.gguf --port 8094
# 3. 测量
python harness\run_matrix.py --scenarios scenarios\ggml_0.6b_main.json
python harness\run_matrix.py --scenarios scenarios\torch_0.6b_main.json
python harness\concurrent_run.py --tts-url ... --llm-base-url http://127.0.0.1:11435 ...
python tools\summarize_bench.py --root benchmarks --out benchmarks\SUMMARY.json
python tools\eval_quality.py --manifest samples\manifest-0.6b-ggml.json --ref-wav assets\ref_zh_prompt_audio1_16k.wav --out benchmarks\quality-0.6b-ggml.json
```

| 数据文件 | 内容 |
| --- | --- |
| `benchmarks/phase1-torch-0.6b/**` | Torch 0.6B 主矩阵 / 预缓冲 / 长文本 / 质量样本（每 case 一份 `probe.json`） |
| `benchmarks/phase2-ggml-0.6b/**` | GGML 0.6B 同上 |
| `benchmarks/phase5-torch-1.7b/**`、`phase6-ggml-1.7b/**` | 1.7B 两组 |
| `benchmarks/phase3-concurrent/both.json`、`phase4-concurrent-ggml/both.json` | LLM+TTS 并发（含 summary.md） |
| `benchmarks/vram-*.json` | 分阶段显存时间序列 |
| `benchmarks/cancel-0.6b-torch.json`、`abort-ggml-0.6b.json` | 取消 / 断连终止 |
| `benchmarks/quality-*.json` | CER + 说话人余弦（逐条 hyp/ref） |
| `benchmarks/SUMMARY.json` | 所有 probe 的汇总索引（`tools/summarize_bench.py` 生成） |

### 试听清单（人工）

| 目录 | 内容 | 建议比较 |
| --- | --- | --- |
| `samples/0.6b-ggml-q8/` | 10 组中文（推荐方案） | 与 `samples/0.6b-torch/` 同文本对比：音色、语速、数字读法 |
| `samples/1.7b-ggml-q8/`、`samples/1.7b-torch/` | 1.7B 两组 | 是否值得多 1 GB |
| `samples/emotion/emotion-*.wav` | 5 个情绪变体 | 听"快/慢"是否真的区分 |
| `F:\AI_Project_Development\docs\tts-samples\sample-*.wav` | MOSS 基线 | 同一文本下的可懂度、流畅度、欠载痕迹 |
