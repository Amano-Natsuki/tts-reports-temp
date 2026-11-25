# Qwen3-TTS 独立实验状态（04A / 04B / 04C）

> 目录：`F:\AI_Project\Qwen3-TTS-Experiments`
> 不修改主项目 `F:\AI_Project_Development`、MOSS 环境或 llama.cpp 源码。
> 最近更新：2026-11-24（04C 人工试听回收；默认 `vivian` 被否）

---

## 1. 阶段总览

| 阶段 | 内容 | 状态 | 产物 |
| --- | --- | --- | --- |
| 04A | MOSS 实时播放断续诊断 | ✅ 完成（主项目） | `F:\AI_Project_Development\docs\phase04a-realtime-playback-diagnosis.md` |
| 04B | Qwen3-TTS 生态调研 + 4090 实测 | ✅ 完成 | `QWEN3_TTS_RESEARCH.md`、`QWEN3_TTS_BENCHMARK.md`、`DECISION.md` |
| **04C** | **中文音色/克隆、Base 情绪、CustomVoice/VoiceDesign、1.7B 并发与调度** | **✅ 技术验证完成；✅ 29 条人工试听完成；默认 vivian/情绪被否；中文参考/换音色复测 Pending** | 见 §2 |

---

## 2. 04C 交付物

| 文件 | 内容 |
| --- | --- |
| `docs/qwen3-tts-voice-quality.md` | 参考音频现状、x-vector vs ICL、Base/CustomVoice 10 组样例、说话人/VoiceDesign 客观音高、29 条试听结果 |
| `docs/qwen3-tts-emotion.md` | GGML Base 拒绝 instruct 的契约证据、Torch 1.7B Base 8 指令 ×3、CustomVoice 1.7B/0.6B instruct、VoiceDesign 描述、7 条情绪试听结果 |
| `docs/qwen3-tts-concurrency.md` | LLM/TTS 单独、0.6B/1.7B 并发、长重叠、串行、恢复、断连取消、显存相位、04E 建议 |
| `docs/qwen3-tts-decision-04c.md` | **CONDITIONAL GO（整体）/ 当前 vivian 与情绪 NO-GO**、§9 人工试听回填、04D 接入配置、风险、主项目状态更新建议 |
| `docs/recording-guide-zh.md` | 中文参考音频录制指南（5–15 s、24 kHz+、单声道、ref_text 人工校对、隐私授权） |
| `docs/listening-sheet-04c.md` | 29 条实际产物表 + 已填评分 + 模型级汇总（2026-11-24） |
| `docs/listening-key-04c.md` | 解码表（评分已完成，可复核） |
| `samples/04c-listening/` | A01–A29.wav 匿名样本 |
| `benchmarks/04c/` | 所有原始 probe JSON/WAV、index、VRAM、契约拒绝、质量评估 |
| `benchmarks/04c/listening-scores.json` / `listening-summary.md` | 试听评分解析与分组汇总（脚本 `tools/analyze_listening_04c.py`） |
| `tools/ref_audio_check.py` | 参考音频体检（纯标准库） |
| `tools/emotion_metrics.py` | 情绪客观代理指标（纯标准库；F0/语速/停顿/电平） |
| `harness/04c_run.py` | 批量 TTS 探测/voice 注册封装 |
| `harness/04c_serial.py` | LLM 完成后串行 TTS 的时延测量 |
| `research/04c-model-families.md` | 官方模型卡/Qwen3-TTS 三族/qwentts 源码契约核实 |

---

## 3. 环境与固定版本

| 项 | 值 |
| --- | --- |
| OS | Windows 10 19045 |
| GPU | RTX 4090 24,564 MiB（sm_89），驱动 621.14 |
| qwentts.cpp | **`2edb35f`**（2026-11-17），CUDA 12.9 / MSVC 2022，`sm_89-real` |
| 二进制 | `models/qwentts/src/build-cuda/Release/tts-server.exe` + 同目录 DLL |
| Talker（主） | `models/qwentts/qwen-talker-1.7b-customvoice-Q8_0.gguf`（2,042,834,304 B） |
| Talker（Base） | `models/qwentts/qwen-talker-1.7b-base-Q8_0.gguf`（2,079,448,256 B） |
| Talker（VD） | `models/qwentts/qwen-talker-1.7b-voicedesign-Q8_0.gguf`（2,042,833,824 B） |
| Talker（0.6B CV） | `models/qwentts/qwen-talker-0.6b-customvoice-Q8_0.gguf`（968,588,544 B） |
| Codec | `models/qwentts/qwen-tokenizer-12hz-Q8_0.gguf`（291,150,624 B） |
| 官方 checkpoint | `checkpoints/Qwen3-TTS-12Hz-{0.6B-CustomVoice,1.7B-CustomVoice,1.7B-VoiceDesign}`（ModelScope 直连下载） |
| Torch 环境 | `.venv`：torch 2.9.1+cu128 / faster-qwen3-tts 0.5.4 / transformers 5.15.1 |
| LLM | `Qwen3.5-9B-Uncensored-HauhauCS-Aggressive-Q4_K_M`，ctx 131072，KV q8_0，`llama-server 11435` |
| 质量评估 | GSV runtime（`G:\Audio_Processing\TTS\GPT-SoVITS\...\runtime\python.exe`）FunASR Paraformer + CAM++，CPU |

---

## 4. 关键结论（一句话版）

1. **技术主方案仍是 1.7B GGML Q8_0**：常驻增量 3,027 MiB（约 3.17 GB）、暖机 TTFA 15–42 ms、RTF 0.09–0.10、71.2 s 长音频零欠载、20/20 稳定、断连 313 ms 真停。
2. **默认 `vivian` 被 2026-11-24 人工试听否决**：12 条均值自然 2.58/机械 3.25；短句中性 A28 2/5、7 条情绪中 6 条差（`fast` 仅勉强）。候选改为 VoiceDesign `cute_girl`、CustomVoice `ryan`、VoiceDesign `young_female_gentle` / `mature_male_deep`、Base x-vector（需中文参考）。
3. **情绪不能算通过**：GGML Base 400 拒绝 instruct；CustomVoice `vivian` 6/7 主观不合格（`fast` 仅“勉强”）；全部 experimental。
4. **1.7B 与 9B 并发 LLM Decode −61.0%/−61.7%**；0.6B −59.2%；默认必须 `llm_priority`，overlap 仅实验，sentence_interleave unsupported。
5. **04B DECISION 摘要的 0.6B/1.7B 口径已更正**：详细表是 0.6B（−57.7%），本轮 1.7B 单独实测 −61.0%/−61.7%。

---

## 5. 待用户完成（阻塞最终验收）

- [x] 盲听 `samples/04c-listening/A01…A29.wav` 并填分（2026-11-24 完成；默认 `vivian`/情绪被否）。
- [ ] **从候选里选一个音色**：`cute_girl` / `ryan` / `young_female_gentle` / `mature_male_deep` / Base x-vector；选定后补 10 文本复测。
- [ ] 若选 Base 克隆：按 `docs/recording-guide-zh.md` 录 5–15 s 中文 WAV + 人工逐字 `ref_text`，跑 `tools/ref_audio_check.py`；ICL 暂不可用。
- [ ] **情绪指令默认禁用**；若必须启用，先在选定音色上按指令 ≥20 次抽检，`fast` 也只能 experimental。
- [ ] 确认 04D 是否把 Qwen 设为默认 TTS（未确认前保持 `none`，不静默改配置）。

---

## 6. 已知问题 / 坑

1. **Windows locale 坑（本轮已踩）**：`models/qwentts/src/convert.py` 用 `Path.read_text()` 默认按 GBK 解码 UTF-8 的 `vocab.json`/`merges.txt`，导致 BPE token 表错位、中文 tokenize 出 5 个 token、`[Prompt] FATAL: tokenized prompt too short`、HTTP 502。**必须 `python -X utf8` / `PYTHONUTF8=1` 运行转换**；重转后 token 0–255 与 Base GGUF 一致。04D 若重新转换/打包，务必带 UTF-8 模式并核对 token 190=`Ă`、198=`Ċ`。
2. **HF 直连 GGUF 下载不稳**：Clash 代理下 0.6B CV 在 21% 卡住；改用 ModelScope 官方 checkpoint + 本地 convert/quantize（9.07 GB 直连，约 4 分钟）更稳。下载总量 <5 GB 时才考虑 HF GGUF。
3. **qwentts 服务端 `max_batch=1`**：并发请求由服务端串行；Provider 必须显式排队/忙碌，不得假设可并发。
4. **WDDM 进程级显存不可用**：`nvidia-smi --query-compute-apps used_memory` 全 `[N/A]`；只能用整卡相位差。
5. **Base 不能 instruct**：HTTP 400；04D 的 Provider 必须拒绝/禁用该字段，不能静默透传。
6. **`stop_server.ps1` 在 Windows PowerShell 5.1 下用 `$pid` 会报只读变量错误**（本轮临时手工 kill）；04D/04E 若复用该脚本，应把循环变量改成 `$procId`。
7. **长重叠 TTS 块间隔 max 164 ms**：不是"TTS 完全不受影响"；300 ms 预缓冲可盖住，但 04D 要真实播放器复测。

---

## 7. 下一步（04D）

1. 读 `docs/qwen3-tts-decision-04c.md`，按 §6 冻结配置实现 `Qwen3TTSProvider`。
2. 用 `run/build_qwentts_windows.cmd` 固定构建；模型放项目外路径；Provider 只走 HTTP，不加载权重。
3. 先显式配置启用（默认仍 `providers.tts.type=none`），真实 `AudioPlayer` 复测 200–300 ms 预缓冲、取消、10 组样例。
4. 人工试听结果回来后再决定是否把 Qwen 设为默认、是否走 Base 克隆。

---

## 8. 快速入口

| 想知道什么 | 看这个 |
| --- | --- |
| 最终选型/冻结配置 | `docs/qwen3-tts-decision-04c.md` |
| 音色/克隆/参考音频 | `docs/qwen3-tts-voice-quality.md` |
| 情绪/CustomVoice/VoiceDesign | `docs/qwen3-tts-emotion.md` |
| 并发/串行/显存/取消 | `docs/qwen3-tts-concurrency.md` |
| 官方模型卡/源码契约 | `research/04c-model-families.md` |
| 04B 基线 | `QWEN3_TTS_BENCHMARK.md`、`DECISION.md` |
| 环境版本 | `environment/versions.json`、`environment/README.md` |
