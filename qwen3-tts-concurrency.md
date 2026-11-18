# Qwen3-TTS 与本地 LLM 的并发、串行与 GPU 调度实测（04C）

> 任务书：`04C_任务书_Qwen音质情绪与GPU调度验收.md`（§3.4、§3.5、§5 并发部分）
> 环境：Windows 10 19045 / RTX 4090 24 GB / i7-13700KF / 32 GB；qwentts.cpp `2edb35f`（GGML CUDA，sm_89）；llama.cpp `llama-server`，模型 `Qwen3.5-9B-Uncensored-HauhauCS-Aggressive-Q4_K_M`，ctx 131072，KV q8_0，端口 11435
> 原始数据：`benchmarks/04c/concurrent-17b-short.json` / `.md`、`concurrent-17b-longoverlap.json` / `.md`、`concurrent-06b-short.json` / `.md`、`serial-17b-longllm-s03.json`、`abort-17b.json`、`vram-*.json`、`llm-alone-*.json`、`llm-post-concurrency.json`、`tts-post-concurrency.json`
> 结论日期：2026-11-18
> **口径**：TTFA = 请求提交 → 首个可播放音频字节；`RTF_project = 合成墙钟 / 音频时长`（<1 快于实时）；LLM `tok_s` = 生成段 token 速率。显存均为**整卡 nvidia-smi**（WDDM 下进程级 `used_memory` 全部 `[N/A]`，不能当 0）。

---

## 1. 结论先行

1. **1.7B + 9B LLM 并发代价是真实的、且比 0.6B 略大**：短请求下 LLM Decode 从 **133.9 → 52.2 tok/s（−61.0%）**；长文本持续重叠下 **127.4 → 48.8 tok/s（−61.7%）**。0.6B 复现为 **141.1 → 57.6 tok/s（−59.2%）**。
2. **TTS 几乎不受影响**：1.7B 并发时 RTF 0.091（短）/ 0.091→0.097（长重叠，+6.6%），零欠载；但长重叠时块间隔 max 从 60 ms 升到 **164 ms**，说明"TTS 不掉速"不等于"完全无抖动"。
3. **串行（llm_priority）代价可接受**：337 token 的 LLM 回复约 2.7 s，LLM 完成后再合成 12 s 语音，用户首音频 **2.71–3.02 s**，整回合 **3.77–4.08 s**；TTS 只额外加 14–35 ms TTFA。
4. **TTS 完成后 LLM 完全恢复**：并发后 LLM 单独 **141.3 tok/s**、TTS RTF **0.090**、零欠载。没有发现显存泄漏或长期降速。
5. **断连取消是真实的**：1.7B 读到 2.05 s 音频后断开，GPU 利用率 **94% → 1–5%（约 313 ms 内）**，紧随其后的新请求 TTFA **61.5 ms**；无僵尸/排队。
6. **显存**：1.7B Base 带 speaker encoder 常驻增量 **3,461 MiB（约 3.63 GB）**、生成峰值增量 **3,592 MiB（约 3.77 GB）**；CustomVoice/VoiceDesign 无 speaker encoder，常驻增量 **3,027 MiB（约 3.17 GB）**；LLM 单独增量 **8,147 MiB（约 8.54 GB）**；1.7B + LLM 并发整卡峰值 **13,184 MiB（约 13.82 GB）**（24 GB 卡内安全）。
7. **调度结论**：`llm_priority` 必须是默认；`overlap` 只能作为显式实验模式，代价约 **−61% LLM Decode**。`sentence_interleave` 在 llama.cpp 当前接口下**没有暂停/续接同一生成请求的能力**（只能 cancel + 重新 prefill），04C 不实现，04E 只能写 unsupported 或做重复 prefill 的代价评估。

---

## 2. 测试设置与基线

| 项 | 值 |
| --- | --- |
| GPU 桌面空闲 | 1,906 MiB（本轮；04B 时为 1,108 MiB，桌面应用不同，只用于本轮的相位差） |
| TTS 服务 | `tts-server.exe --model qwen-talker-1.7b-base-Q8_0.gguf --codec qwen-tokenizer-12hz-Q8_0.gguf --port 8095`，`max_batch=1` |
| TTS 参考 | x-vector 注册的英文 MOSS 音色（与 04B 相同，仅用于技术对照） |
| LLM | `run/start_llm_9b.ps1`，9B Q4_K_M，ctx 131072，KV q8_0，`-np 1`，含 mmproj（不 offload） |
| 短 TTS 文本 | s03 长句（11.76 s 音频） |
| 长 TTS 文本 | `texts/long_20sentences.txt`（71.2 s 音频） |
| LLM 短 prompt | `llm_bench.py` 默认 prompt，`max_tokens=128`，实际 29 tokens |
| LLM 长 prompt | `texts/llm_long_prompt_zh.txt`，`max_tokens=512`，实际 337 tokens |
| 重复 | 并发各 3 次；串行 3 次；0.6B 3 次；长重叠 3 次 |

> 测量前确认 GPU 没有其他 Agent/服务占用（桌面空闲 1.8–1.9 GB）；04A 的 MOSS 诊断已在 08:55 前结束。队列时长：`stream_probe` 在每 case JSON 里保留 `playback_start_ms` / `buffer_end_ms`，本轮没有单独汇总"队列深度"——`tts-server` 是 `max_batch=1` 串行，客户端也是顺序探测；04D 需用真实 `AudioPlayer` 统计队列深度/背压。

### 2.1 单独基线

| 基线 | 结果 |
| --- | --- |
| LLM 短（29 tok） | TTFT 52.2 ms；wall 276.0 ms；**133.9 tok/s**；显存 10,053 MiB（idle）/10,067（gen） |
| LLM 长（337 tok） | TTFT 71.1 ms；wall 2,647.8 ms；**131.6 tok/s** |
| TTS 短（s03，11.76 s） | TTFA 19.2 ms；RTF **0.091**；0 欠载；147 chunks；块间隔 p95 57.8 / max 58.4 ms |
| TTS 长（71.2 s） | TTFA 19.7 ms；RTF **0.091**；0 欠载；890 chunks；块间隔 p95 58.2 / max 60.0 ms |
| TTS 1.7B 显存 | idle 5,367 MiB；gen peak 5,498 MiB → **增量 3,461 / 3,592 MiB** |
| CustomVoice/VoiceDesign 1.7B | idle 4,909 MiB → **增量 3,027 MiB**（无 speaker encoder） |

---

## 3. 短请求并发（复现 04B 口径）

方法：`harness/concurrent_run.py` 同时启动 TTS 与 LLM 子进程，各自 `repeat=3`，200 ms 采样显存；两进程启动差 <7 ms，重叠约 **1.92 s**（TTS 1.07 s / LLM 0.63 s 的并发段）。

| 指标 | 单独 | 并发 | 变化 |
| --- | ---: | ---: | ---: |
| **1.7B TTS TTFA** | 19.2 ms | 12.4 ms | −6.8 ms（噪声量级；TTS 未受损） |
| TTS RTF | 0.091 | 0.091 | 0 |
| TTS 欠载 | 0 | 0 | — |
| TTS 块间隔 p95 / max | 57.8 / 58.4 ms | 57.7 / 58.3 ms | ≈0 |
| **LLM 首 token** | 52.2 ms | 55.7 ms | +3.5 ms（+6.7%） |
| **LLM Decode** | **133.9 tok/s** | **52.2 tok/s** | **−81.7 tok/s（−61.0%）** |
| LLM wall（29 tok） | 276.0 ms | 633.9 ms | +129.7% |
| 整卡显存峰值 | 13,233 MiB（TTS 单独阶段，LLM 已驻留） | 13,184 MiB | 并发没有额外涨（两模型都已驻留） |

**0.6B 复现**（同一 LLM、同一 TTS 文本）：

| 指标 | 单独 | 并发 | 变化 |
| --- | ---: | ---: | ---: |
| TTS RTF | 0.072 | 0.073 | +1.4% |
| TTS 欠载 | 0 | 0 | — |
| LLM Decode | **141.1 tok/s** | **57.6 tok/s** | **−59.2%** |
| 整卡峰值 | 12,688 MiB（TTS 单独阶段，LLM 已驻留） | 12,638 MiB | — |

**读法**：

- 短请求的 LLM 只有 29 token（约 0.55 s），并发重叠 ~1.9 s；这个口径能复现 04B 的"掉 58–61%"，但**不能说明长回复全程的代价**——所以补了下一节长重叠。
- 1.7B 的 −61.0% 与 0.6B 的 −59.2% 同级；**0.6B 并不能"保护 LLM"**，只省约 0.4–0.6 GB 显存。
- TTS 的 TTFA/RTF 在并发下没有变差，说明它是一串短 kernel，能插空执行；瓶颈是 LLM Decode 被抢 SM/带宽。

---

## 4. 长文本持续重叠（1.7B，本轮重点）

方法：TTS 用 71.2 s 音频的长文本（合成墙钟 6.48 s），LLM 用 337 token 长回复（单独 2.70 s），两边各 3 次；TTS 比 LLM 长，保证 LLM 整个 Decode 都落在 TTS 生成窗口内。重叠 20.9 s（3 轮累计）。

| 指标 | 单独 | 并发 | 变化 |
| --- | ---: | ---: | ---: |
| TTS TTFA | 19.7 ms | 25.7 ms | +6.1 ms（+30.8%，绝对值仍 <30 ms） |
| TTS RTF | 0.091 | **0.097** | **+6.6%** |
| TTS 欠载 | 0 | 0 | — |
| TTS 块间隔 p95 / max | 58.2 / **60.0 ms** | 58.5 / **164.1 ms** | max +104 ms（抖动变大） |
| LLM 首 token | 43.6 ms | 64.6 ms | +21.0 ms（+48.3%） |
| LLM Decode | **127.4 tok/s** | **48.8 tok/s** | **−78.6 tok/s（−61.7%）** |
| LLM wall（337 tok） | 2,696.9 ms | 6,984.6 ms | +159.0% |
| 整卡峰值 | 13,200 MiB | 13,287 MiB | +87 MiB |

**读法**：

- 在"LLM 边想、TTS 边合成"的真实重叠形态下，**LLM Decode 仍然掉 61.7%**，与短请求一致；不是"只有开头被抢一下"。
- TTS 只慢 6.6%，但块间隔 max 164 ms 说明 GPU 争抢会造成偶发抖动；对 24 kHz 播放来说 164 ms 的块间隔仍远小于 80 ms 帧的预缓冲需求（300 ms 预缓冲可盖住），但**不能宣称"TTS 完全不受影响"**。
- 这组数据是 04E `overlap` 风险提示的直接依据：**默认不要 overlap**。

---

## 5. 串行方案：LLM 完成后再 TTS（`llm_priority`）

方法：`harness/04c_serial.py` 在同一进程内先跑 LLM（337 token 长 prompt），等它完全结束后立即发 TTS 请求（s03 12 s 文本）；3 次。没有 GPU 重叠。

| 运行 | LLM wall | LLM tok/s | TTS TTFA | 用户首音频 | 整回合 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 1 | 2,988.3 ms | 122.3 | 34.8 ms | **3,023.1 ms** | 4,076.7 ms |
| 2 | 2,693.5 ms | 127.2 | 34.9 ms | **2,728.4 ms** | 3,782.8 ms |
| 3 | 2,699.4 ms | 126.9 | 13.8 ms | **2,713.2 ms** | 3,767.8 ms |

**读法**：

- 用户首音频 = LLM 整段生成完成 + TTS 暖机 TTFA（14–35 ms）；对 337 token 回复约 **2.7–3.0 s**，比"LLM 首 token 43–71 ms"慢，但换来了 **LLM 全程不降速**。
- 整回合（LLM + 12 s 音频合成）约 **3.8–4.1 s**；播放本身可在合成开始后异步进行（04D 用流式 PCM），不需要等整段合成完。
- 这是 04E 默认策略的量化依据：**用 2.7–3.0 s 的首音频延迟，换 LLM 133→127 tok/s（不重叠）而不是 52 tok/s（重叠）**。

---

## 6. 恢复与取消

### 6.1 并发后恢复

| 项 | 并发前 | 并发后 | 判定 |
| --- | ---: | ---: | --- |
| LLM Decode（短） | 133.9 tok/s | **141.3 tok/s** | 完全恢复（甚至略高，属 run-to-run 波动） |
| TTS RTF（s03） | 0.091 | **0.090** | 完全恢复 |
| TTS 欠载 | 0 | 0 | — |

### 6.2 断连取消（1.7B）

| 项 | 值 |
| --- | --- |
| 断开前已读 | 98,304 B = **2.048 s** 音频 |
| 读取耗时 | 209.1 ms |
| GPU 利用率 | 断开前 **94%**；断开后 313 ms 采样 **1–5%** |
| 新请求 TTFA | **61.5 ms**（无排队/僵尸） |
| 后续音频 | 2,080 ms 正常返回 |

**判定**：GGML 的"客户端断连 → chunked sink 返回 false → 生成停止"在 1.7B 上同样成立；04D 可以直接用断连取消，不必依赖显式 cancel 端点。

---

## 7. 显存与预算

| 阶段 | 整卡 MiB | 相对桌面空闲（1,906 MiB） |
| --- | ---: | ---: |
| 桌面空闲 | 1,906（1,900–1,937；CV/VD 相位用同会话 1,882 基线） | — |
| 1.7B Base idle | 5,367 | **+3,461 MiB（3.38 GiB）** |
| 1.7B Base 生成峰值 | 5,498 | **+3,592 MiB（3.51 GiB）** |
| 1.7B CustomVoice / VoiceDesign idle | 4,909 | **+3,027 MiB（2.96 GiB；同会话 1,882 基线）** |
| LLM idle | 10,053 | **+8,147 MiB（7.96 GiB）** |
| LLM 生成峰值 | 10,067 | +8,161 MiB |
| 1.7B + LLM 并发峰值 | 13,184（短）/13,287（长） | 相对桌面 +11,278 / +11,381 MiB |

说明：

- **WDDM 下 `nvidia-smi --query-compute-apps` 的 `used_memory` 全是 `[N/A]`**（本机实测）；上表是整卡 `memory.used` 的相位差，不是进程级读数。`harness/vram_sampler.py` 会把 `apps.memory_available=false` 写进 JSON，避免误读为 0。
- CustomVoice/VoiceDesign 比 Base 少约 434 MiB（约 0.46 GB），是因为它们没有 speaker encoder；**1.7B CV 的 3,027 MiB（约 3.17 GB）比 1.7B Base 的 3,461 MiB（约 3.63 GB）更省**，同时满足 ≤4 GB 常驻 / ≤5 GB 峰值门槛。
- 0.6B Base 约 2.5 GB 量级（04B 2.55/2.61 GB），只比 1.7B CV 少约 0.5 GB；**为了这 0.5 GB 牺牲 1.7B 的音质/指令稳定性不划算**。

---

## 8. 调试/环境备注

- **04B 的 DECISION 摘要有口径错位**：详细表 §5.1 的并发数据是 **0.6B**（LLM 133.2→56.4 tok/s，−57.7%），但 DECISION 摘要把近似掉速写到了 1.7B 行。本轮 1.7B 实测 **−61.0%/−61.7%**，量级接近但**不能用 0.6B 冒充**；`qwen3-tts-decision-04c.md` 已按本轮 1.7B 数据更正。
- **LLM 基线会随 run 波动**：单独短请求 133.9–141.3 tok/s，长请求 127.4–131.6 tok/s；报告统一用同一次 `concurrent_run` 里的 baseline 做 delta，不跨 run 硬比。
- **TTS 的 RTF 包含 TTFA**（项目口径）；短句冷启动首条会偏高（如 0.137），报告用中位数并单列首条。
- **长重叠的 TTS 音频 71.2 s 是单请求**；`max_seq_len=4096`（约 328 s 音频）内，未触发切句。04D 的长回复仍建议语义切分，避免单请求过长。
- **并发时不要让两个 TTS 服务同时驻留**：本轮的并发只加载一个 TTS 模型；多 TTS 实例会额外占 3 GB/实例，且 `--max-batch=1` 的服务端串行限制没有改变。

---

## 9. 对 04E 的直接建议

1. **默认 `llm_priority`**：LLM 生成期间只做文本缓冲/切分，不发起 TTS 合成；LLM 完成后按段请求 TTS。实测首音频 2.7–3.0 s，LLM 不掉速。
2. **`overlap` 只做显式实验模式**：必须显示"LLM Decode 约 −61%、TTS 块间隔 max 约 164 ms"的实测代价；默认关闭。
3. **`sentence_interleave` 标 unsupported**：llama.cpp 当前 server/client 没有"暂停并续接同一生成请求"的接口；只能 cancel 后重新 prefill，会重复计算并可能造成上下文/输出重复。04C 不做实现。
4. **取消走断连**：1.7B 已验证断连 313 ms 内 GPU 降下来；下一请求 61.5 ms 可用。不需要等显式 cancel API。
5. **服务串行**：`tts-server` 默认 `max_batch=1`；04D/04E 必须显式排队，不要假设可并发合成多段。
6. **资源阈值**：1.7B CV 常驻 +3,027 MiB（约 3.17 GB）、峰值预计约 3.2 GB；1.7B Base 克隆 +3,461/3,592 MiB（约 3.63/3.77 GB）；均低于 04E 的 ≤4/≤5 GB 门槛。LLM 单独 +8,147 MiB（约 8.54 GB），24 GB 卡可共存。
