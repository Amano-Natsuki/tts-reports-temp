# 04C 决策：Qwen3-TTS 是否作为桌面助手常驻 TTS 后端

> 依据：`qwen3-tts-voice-quality.md`、`qwen3-tts-emotion.md`、`qwen3-tts-concurrency.md` 与本轮原始数据 `benchmarks/04c/`。
> 环境：Windows 10 19045 / RTX 4090 24 GB / i7-13700KF / 32 GB；qwentts.cpp **`2edb35f`**（2026-11-17），MSVC 2022 + CUDA 12.9，`sm_89-real`。
> 日期：2026-11-18
> **最终判定：`CONDITIONAL GO`**——技术门槛通过，但中文音色的人工试听与用户参考音频仍未完成，不能写成最终音质验收通过。

---

## 1. 判定

| 维度 | 结果 | 说明 |
| --- | --- | --- |
| 性能（TTFA/RTF/欠载） | ✅ 通过 | 1.7B CV/Base 暖机 TTFA 15–42 ms、RTF 0.09–0.10、71.2 s 长音频零欠载 |
| 显存（常驻/峰值） | ✅ 通过 | 1.7B CV/VD **+3,027 MiB（约 3.17 GB）** 常驻；1.7B Base 克隆 **+3,461/3,592 MiB（约 3.63/3.77 GB）**；均低于 ≤4/≤5 GB |
| 稳定性（20 连发/10 固定样本） | ✅ 通过 | 20/20 成功、零欠载、无崩溃；10 组固定样本全成功；speaker 交替无串音 |
| 取消（断连） | ✅ 通过 | 1.7B 断连后 313 ms 内 GPU 94%→1–5%，新请求 TTFA 61.5 ms |
| 并发（1.7B + 9B） | ✅ 通过（默认 llm_priority） | LLM Decode −61.0%（短）/−61.7%（长重叠）；TTS 不受损；串行首音频 2.7–3.0 s |
| 1.7B 并发数据 | ✅ 已单独留档 | 纠正 04B 摘要的 0.6B/1.7B 口径混用 |
| 情绪稳定性 | ⚠️ **未通过（未满足稳定控制）** | GGML Base 完全不支持 instruct；CustomVoice 有响应但不稳定（ryan angry/slow）；人工未试听 |
| 中文音色人工验收 | ⚠️ **Pending** | 29 条匿名样本已备；无用户中文参考；不得替用户签字 |
| 中文参考/克隆 | ⚠️ **Pending** | ICL 需要中文参考 + 人工校对 ref_text；自动英文转写已证明会泄漏/漏字 |

**判定理由**：

- 性能、显存、稳定性、取消、并发五个硬门槛都过；Qwen3-TTS GGML 明显优于 MOSS（MOSS RTF 1.24–1.95、12 GB、CER 36.6%）。
- **不能给 GO**：人工中文音色未试听，用户参考音频未提供；情绪稳定控制未满足。
- **不是 NO-GO**：技术链路可复现、指标达标，且有可执行的补齐路径（CustomVoice 预设 + 人耳确认，或用户录中文参考后走 Base 克隆）。

---

## 2. 推荐方案（04D 冻结配置）

### 2.1 后端与模型

| 项 | 冻结值 |
| --- | --- |
| 后端 | qwentts.cpp GGML CUDA，commit **`2edb35f`**（2026-11-17） |
| 构建 | MSVC 2022 + CUDA 12.9，`-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=89-real`；`tts-server.exe` 同目录至少需 `ggml-base.dll` / `ggml-cpu.dll` / `ggml-cuda.dll`（04B 实测）；`qwen.dll` / `yyjson` 的运行时依赖需在 04D 用 dumpbin 核对，不凭猜测打包 |
| Talker | **`qwen-talker-1.7b-customvoice-Q8_0.gguf`**（2,042,834,304 B，本地 convert + quantize 件；字节数与上游 HF GGUF 一致，实测可用） |
| Codec | **`qwen-tokenizer-12hz-Q8_0.gguf`**（291,150,624 B） |
| 模型模式 | **`custom_voice`**（`qwen3-tts.model_type` 由 GGUF 元数据决定，不靠文件名） |
| 默认说话人 | **`vivian`**（bright young female；中文母语）；备选 **`serena`**（warm gentle young female）、`uncle_fu`（低沉稳重男声） |
| 方言预设 | `dylan`（北京话）、`eric`（四川话）仅在 `--lang chinese\|auto` 时触发；默认不用 |
| 语言 | `Chinese` |
| 音频 | `response_format=pcm`，s16le / 24 kHz / mono，流式 chunk（约 80 ms/帧，1920 samples） |
| 预缓冲 | **初始 300 ms**（04D 用真实 `AudioPlayer` 复测；04B 的 2500 ms 不适用于 Qwen） |
| 服务模式 | `external` 或 `managed`；`tts-server` 独立进程，主进程不加载权重 |
| 并发 | `tts-server` 默认 `max_batch=1`；Provider 侧显式排队，不假设可并发合成多段 |
| 取消 | 客户端断连（已验证真停）；不需要显式 cancel 端点 |

### 2.2 声音策略

**默认路径（无需用户录音，立即可用）**：

```
voice = vivian（或 serena）
instructions = 仅透传；默认空
```

- 1.7B CustomVoice 9 个官方预设说话人；`vivian/serena/uncle_fu` 为中文母语，优先。
- 10 组固定文本 CER 1.64%，与 Base x-vector 的 1.42% 同级；简单句 CER 0%。
- 显存比 Base 少约 434 MiB（约 0.46 GB，无 speaker encoder），更利于 04E 的 ≤4 GB 预算。

**进阶路径（用户提供中文参考后）**：

```
mode = base
talker = qwen-talker-1.7b-base-Q8_0.gguf
voice = 用户注册的 ref_xvec 或 ref_icl
ref_wav = 5–15 s、24 kHz+、单声道、干净中文 WAV
ref_text = 人工逐字校对（ICL 必需）
```

- Base 1.7B 的 10 组固定文本 CER 1.42%、cosine 0.555（英文参考）。
- x-vector 内容干净、跨语种安全；ICL 音色更贴（cosine 0.74–0.80）但**必须**人工校对 `ref_text`，否则出现英文起头/漏字（本轮实测 CER 4.8–14.3%）。
- **在拿到中文参考 + 校对文本之前，不要把 Base 设成默认**。

**可选路径（造音色）**：

```
mode = voice_design
talker = qwen-talker-1.7b-voicedesign-Q8_0.gguf
instructions = "<中文/英文音色描述>"
```

- VoiceDesign 1.7B 对描述有明确响应（F0 69–389 Hz；中英文 instruct 都有效），CER 0%（简单句）；但音色一致性与主观质量未验收，只作为候选/实验，不作为默认。

### 2.3 调度

| 项 | 冻结值 |
| --- | --- |
| 默认策略 | **`llm_priority`**：LLM 生成期间只缓冲/切分文本，不发起 TTS 合成 |
| `overlap` | 默认关闭；仅用户显式开启的实验模式，UI 必须显示"LLM Decode 约 −61%、TTS 块间隔 max 约 164 ms" |
| `sentence_interleave` | **unsupported**：llama.cpp 当前接口不能暂停/续接同一生成请求；只能 cancel + 重新 prefill，04E 不做生产实现 |
| 串行代价 | 337 token 回复 + 12 s 语音：首音频 2.7–3.0 s，整回合 3.8–4.1 s |
| 并发代价 | 1.7B + 9B：LLM 133.9→52.2 tok/s（−61.0%）；0.6B + 9B：141.1→57.6（−59.2%）；0.6B 不保护 LLM |
| 取消 | 新请求到达时断连旧 TTS；已验证 313 ms 内 GPU 降下来，下一请求 61.5 ms 可用 |

---

## 3. 为什么不选其他方案

| 方案 | 不选的原因 |
| --- | --- |
| **MOSS-TTS-Realtime** | 12 GB、RTF 1.24–1.95、长句 17/50 次欠载、ASR CER 36.6%；04A 已定位根因在模型侧，播放器无独立缺陷 |
| **Torch faster-qwen3-tts 1.7B** | 常驻 5.06 GB > 5 GB 峰值预算；TTFA 302–327 ms；RTF 0.273–0.287；只适合做 Base instruct 对照 |
| **Torch 0.6B** | 3.58/3.74 GB，TTFA 257–286 ms、RTF 0.23；纯 pip 不编译是优点，但性能比 GGML 差一个量级，且需自建 cancel |
| **GGML 0.6B Base** | 2.55/2.61 GB，比 1.7B CV 只省约 0.5 GB；音质/指令跟随弱于 1.7B；仅低显存 fallback |
| **GGML 0.6B CustomVoice** | instruct 可用（实测），但只测 1 speaker/2 seed；音色/稳定性弱于 1.7B；低显存备选 |
| **Base + instruct（GGML）** | **不可用**：HTTP 400 `--instruct is not supported for base models`；Provider 必须拒绝或禁用该字段 |
| **Base ICL + 英文自动 ref_text** | 音色余弦提升到 0.74–0.80，但 CER 4.8–14.3%、出现"Really"英文起头；必须中文参考 + 人工校对 |
| **VoiceDesign 作为默认** | 描述→F0 有效，但音色一致性/主观质量未验收；作为可选造音色路径 |

---

## 4. 遗留风险

| 风险 | 证据 | 缓解 |
| --- | --- | --- |
| CustomVoice instruct 不稳定 | `ryan + angry` F0 95→233–268；`ryan + slow` 16.72 s（疑似重复/拖长） | 生产先固定 `vivian`/`serena`；每条指令做 ≥20 次抽检 + 人耳；UI 标 experimental |
| Base instruct 不可用于 GGML | HTTP 400 | Provider 层拒绝 `instructions`；不要用提示词包装 |
| 中文参考/ICL 未验收 | 无用户中文参考；自动英文 ref_text 已证明会污染 | 按 `docs/recording-guide-zh.md` 录制 + 人工校对；用 `tools/ref_audio_check.py` 体检 |
| 人工音色未试听 | 29 条样本待听 | 用户按 `docs/listening-sheet-04c.md` 盲听；完成前保持 CONDITIONAL |
| 0.6B CV 官方口径矛盾 | 官方总表留空 vs 卡片示例用 instruct | 实测可用，但只作 fallback；不引用官方"0.6B 有 instruct 评测" |
| Windows 转换 locale 坑 | `convert.py` 用 `Path.read_text()` 默认 GBK，UTF-8 vocab 被误读 → BPE 乱码、`[Prompt] FATAL: prompt too short`、502 | **必须 `python -X utf8` / `PYTHONUTF8=1` 运行 convert.py**；已重转并验证 token 0–255 与 Base GGUF 一致 |
| 并发时 TTS 块间隔抖动 | 长重叠 max 164 ms | 300 ms 预缓冲；04D 用真实播放器复测；不要宣称完全无抖动 |
| WDDM 进程级显存不可用 | `nvidia-smi --query-compute-apps used_memory` 全 `[N/A]` | 用整卡相位差；文档已注明，不把 `[N/A]` 当 0 |
| 服务串行 `max_batch=1` | `tts-server` 默认单请求 | Provider 显式排队/忙碌状态；不要并发多段 |

---

## 5. 需要用户本人完成的事项

1. **盲听 29 条样本**：`samples/04c-listening/A01…A29.wav`，按 `docs/listening-sheet-04c.md` §10 填 1–5 分；评分完成后再打开 `docs/listening-key-04c.md` 解码。
2. **选音色路线**：
   - A. 直接用官方预设：`vivian`（明亮年轻女声）或 `serena`（温柔年轻女声）；或 `uncle_fu`（低沉稳重男声）。
   - B. 录自己的中文参考（5–15 s、24 kHz+、单声道、干净 WAV），按 `docs/recording-guide-zh.md` 录 + 人工校对 `ref_text`；随后重跑 Base x-vector/ICL 音色验收。
3. **确认情绪需求**：若必须"稳定情绪控制"，需要接受 `CustomVoice + 固定 speaker + 少量指令`，并追加每条指令 ≥20 次的稳定性抽检；若只是"偶尔有点语气变化"，可把 instruct 标 experimental 透传。
4. **确认默认是否切换**：04D 合并后 Qwen 先显式配置启用；是否替换 `providers.tts.type=none` 的默认，由用户在 04D 收尾确认。

---

## 6. 04D 冻结配置（可直接抄）

```yaml
# 建议写入 04D 的 Provider 配置（具体 YAML 结构按主项目现有 config 风格）
providers:
  tts:
    type: qwen3_tts          # 04D 新 Provider 名称，按项目风格调整
    base_url: http://127.0.0.1:8095
    voice: vivian            # 或 serena / uncle_fu
    language: Chinese
    response_format: pcm
    # 初始实验值，04D 用真实 AudioPlayer 复测后再冻结
    start_prebuffer_ms: 300
  qwen3_tts:
    mode: external           # 或 managed
    commit: 2edb35f
    talker: <项目外路径>/qwen-talker-1.7b-customvoice-Q8_0.gguf
    codec:  <项目外路径>/qwen-tokenizer-12hz-Q8_0.gguf
    alias: qwen3-tts
    max_batch: 1
    instruct: experimental   # 默认空；不要宣传稳定情绪
speech:
  scheduling: llm_priority   # overlap 仅显式实验；sentence_interleave unsupported
```

启动命令（external 模式，模型/DLL 路径按实际部署调整）：

```powershell
tts-server.exe `
  --model  <models>\qwen-talker-1.7b-customvoice-Q8_0.gguf `
  --codec  <models>\qwen-tokenizer-12hz-Q8_0.gguf `
  --alias  qwen3-tts --host 127.0.0.1 --port 8095 `
  --lang   Chinese --max-batch 1
```

---

## 7. 对主项目状态文档的更新建议

> 以下内容供合并到 `F:\AI_Project_Development\docs\IMPLEMENTATION_STATUS.md`，本轮**没有修改主项目**。

```markdown
### Phase 04C（2026-11-18，独立实验，未合并代码）

- 判定：**CONDITIONAL GO**。qwentts.cpp GGML 1.7B Q8_0 通过性能/显存/稳定性/取消/并发门槛；
  中文音色人工试听、用户中文参考音频、情绪稳定控制仍 Pending。
- 推荐 04D 冻结：commit `2edb35f`；CustomVoice 1.7B Q8_0 + tokenizer Q8_0；
  默认 speaker `vivian`（备选 `serena`/`uncle_fu`）；`llm_priority` 默认，overlap 仅实验。
- 关键实测：常驻增量 3,027 MiB（CV/VD，约 3.17 GB）/3,461 MiB（Base 克隆，约 3.63 GB）；峰值 3,592 MiB（约 3.77 GB）；
  暖机 TTFA 15–42 ms；RTF 0.09–0.10；71.2 s 长音频零欠载；1.7B + 9B 并发 LLM Decode −61.0%/−61.7%。
- GGML Base 不支持 instruct（HTTP 400）；情绪控制只能走 CustomVoice/VoiceDesign，且未证明稳定。
- 未改主项目代码、Provider、SpeechCoordinator、AudioPlayer 或主配置；MOSS 仍为 experimental。
- 详情：`F:\AI_Project\Qwen3-TTS-Experiments\docs\qwen3-tts-decision-04c.md` 及同目录三份报告。
```

---

## 8. 证据索引

| 内容 | 路径 |
| --- | --- |
| 音色/参考/克隆 | `docs/qwen3-tts-voice-quality.md` |
| 情绪/CustomVoice/VoiceDesign | `docs/qwen3-tts-emotion.md` |
| 并发/串行/显存/取消 | `docs/qwen3-tts-concurrency.md` |
| 原始探测 | `benchmarks/04c/**`（每个 case 有 probe JSON + WAV） |
| 匿名试听 | `samples/04c-listening/A01…A29.wav`、`docs/listening-key-04c.md` |
| 契约拒绝 | `benchmarks/04c/contract-base.json`、`contract-cv.json`、`contract-vd.json` |
| 模型家族/源码依据 | `research/04c-model-families.md` |
