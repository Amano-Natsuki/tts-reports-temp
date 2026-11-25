# Qwen3-TTS 音色、参考音频与克隆模式验收（04C）

> 任务书：`04C_任务书_Qwen音质情绪与GPU调度验收.md`（§3.1、§5 音色/克隆部分）
> 执行环境：Windows 10 19045 / RTX 4090 24 GB / i7-13700KF / 32 GB；qwentts.cpp `2edb35f`（2026-11-17），MSVC 2022 + CUDA 12.9，`sm_89-real`；Codec `qwen-tokenizer-12hz-Q8_0.gguf`
> 证据目录：`benchmarks/04c/`、`samples/04c-listening/`、`samples/04c-*`
> 结论日期：2026-11-18（技术实测）；**2026-11-24 人工试听回填**
> **状态：技术验证完成；29 条人工试听已完成，但默认 `vivian` 被否；候选为 VoiceDesign `cute_girl`/`young_female_gentle`/`mature_male_deep`、CustomVoice `ryan`、Base x-vector。中文参考/克隆仍 Pending。最终判定见 `qwen3-tts-decision-04c.md` §9。**

---

## 1. 结论先行

1. **技术链路可用**：GGML 1.7B Base Q8_0 的 x-vector 克隆、CustomVoice 预设说话人、VoiceDesign 描述生成音色，三者都在当前 `tts-server` 上跑通；中文简单句 CER 0%，10 组固定文本 CER 均值 1.4–1.6%。
2. **本轮没有“最终中文参考音频”**：没有用户提供/授权的中文 5–15 s 干净录音；现有 `assets/ref_zh_prompt_audio1_*.wav` 实际是 **MOSS 的英文女声**（11.73 s），只做了技术对照，不代表目标中文音色。任何第三方中文 demo 声音（MOSS 仓库自带的 `reference_zh*.wav`）**没有**被拿来克隆。
3. **x-vector vs ICL 有明确取舍**：同一英文参考下，ICL 的音色余弦从 0.52–0.59 提高到 0.74–0.80，但因为 `ref_text` 是 FunASR 自动转写（未校对）且是英文，ICL 出现英文起头泄漏（“Really…”）与中文漏字，CER 从 0% 升到 4.8–14.3%。**ICL 只能配"中文参考 + 人工逐字校对 ref_text"使用**，不能拿自动转写当 ICL 基准。
4. **CustomVoice 是本轮唯一无需用户录音即可给出中文音色的路径**：1.7B `vivian` / `serena` 等 9 个官方预设说话人；10 组固定文本 CER 1.64%，与 Base 的 1.42% 同级；常驻显存比 Base 少约 0.46 GB（无 speaker encoder）。
5. **人工试听已完成（2026-11-24）**：29/29 条评分回收。结论：默认 `vivian` 被否；可用的主观候选是 VoiceDesign `cute_girl`（5/5/5/5/1）、CustomVoice `ryan`（4/4/5/5/1）、VoiceDesign `young_female_gentle` / `mature_male_deep`，以及需中文参考的 Base x-vector。详见 `qwen3-tts-decision-04c.md` §9。

---

## 2. 参考音频现状（这是本轮的硬缺口）

| 项 | 值 | 说明 |
| --- | --- | --- |
| 现有参考 | `assets/ref_zh_prompt_audio1_16k.wav` / `_24k.wav` | 11.728813 s，原为 **英文** MOSS 女声 `prompt_audio1.mp3`；16 kHz mono，峰值 -4.00 dBFS，RMS -22.19 dBFS，静音 14.8%，无削波 |
| ref_text | `assets/ref_text_auto.txt` | FunASR Paraformer 自动转写，**未人工校对**；原文含 `a thrive`、`they ththrive` 等错字 |
| 用户中文参考 | **未提供** | 04C 任务书允许在未提供时标 Pending，不得自行假定/下载/克隆第三方声音 |
| MOSS 自带中文 demo | 未使用 | `F:\AI_Project\MOSS-TTS\assets\audio\reference_zh*.wav` 是上游 demo 素材（第三方声音），按任务书 §3.1 不用于克隆 |
| 新录指南 | `docs/recording-guide-zh.md` | 5–15 s、24 kHz+、单声道、WAV、峰值 -6~-3 dBFS、无 BGM/混响；含 T1–T3 录制文本与 ref_text 校对要求 |
| 校验工具 | `tools/ref_audio_check.py` | 纯标准库；格式/时长/峰值/RMS/削波/静音/DC/重采样检查；示例 `docs/ref-audio-check-example.json` |

**为什么不能拿现有英文参考当最终中文音色**：跨语种 x-vector 克隆会保留参考的基频/共振峰习惯，中文听感容易"外国人读中文"；ICL 虽然音色更贴，但需要准确 ref_text，英文自动转写会直接污染输出。因此音色验收必须等用户提供中文参考，或改用官方 CustomVoice 预设。

---

## 3. 克隆模式对照：x-vector-only vs ICL（GGML 1.7B Base Q8_0）

- 同一参考 WAV、同一中文文本 T11（“今天的工作已经完成了，我们可以一起休息一会儿。”）、3 个 seed（42/123/2026）、同一采样参数。
- 注册方式：`POST /v1/audio/voices`；`ref_xvec` 不带 `ref_text`（x-vector），`ref_icl` 带自动转写 `ref_text`（ICL）。
- 原始数据：`benchmarks/04c/clone-ggml17b/index.json`、`quality-clone-ggml17b-eval.json`；WAV 同目录。

| 模式 | TTFA ms | RTF_project | 音频时长 s | CER | 音色余弦 vs 参考 | 备注 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| x-vector-only | 33.3 / 34.9（首条冷 161.1） | 0.093 / 0.094（首条 0.170） | 3.12 / 3.68 / 3.92 | **0%** | 0.518 / 0.592 / 0.518 | 内容干净；音色细节少 |
| ICL（EN 自动 ref_text） | 19.5 / 19.8 / 74.0 | 0.090 / 0.091 / 0.107 | 2.96 / 3.12 / 3.68 | **4.8% / 14.3% / 14.3%** | **0.736 / 0.801 / 0.761** | 音色更贴，但出现 “Really” 英文起头、漏“一会儿”等 |

ICL 的 ASR 复听原文（`quality-clone-ggml17b-eval.json`）：

```
icl_s42   hyp: Really今天的工作已经完成了，我们可以一起休息。
icl_s123  hyp: Really今天的工作已经完成了，我们可以一起休息一会。
icl_s2026 hyp: Really今天的工作已经完成了，我们可以一起学习一会。
```

**判定**：ICL 的音色收益真实（余弦 +0.2 量级），但当前参考文本不可信；在拿到中文参考与人工校对 `ref_text` 之前，**生产路径只能用 x-vector 或 CustomVoice**。这也是 04C 不能给"克隆验收通过"的直接原因。

**注册开销**：`POST /v1/audio/voices` 整次往返（含 750,780 B WAV 的 base64 上传、speaker encoder/codec 提取）实测 **x-vector 545.9 ms / ICL 391.2 ms**（暖机后；首次冷启动含编码器懒加载约 1.18 s，04B 记录）。注册只在启动/换音色时发生，不进入每请求热路径（`benchmarks/04c/voice-register.json`）。

---

## 4. 10 组固定中文样例（Base / CustomVoice）

文本与 04B 完全一致（`scenarios/ggml_1.7b_main.json` 的 s01–s10），1 次/条，`seed=42`，`prebuffer_ms=300`。原始数据：`benchmarks/04c/quality-base17b/`、`quality-cv17b/`、`quality-*-eval.json`。

| 配置 | TTFA ms（暖机后） | RTF_project | 欠载 | CER 均值 / 最大 | 音色余弦 |
| --- | ---: | ---: | ---: | ---: | ---: |
| **Base GGML 1.7B Q8_0 + x-vector（英文参考）** | 19–42（首条 232 冷） | 0.090–0.137（中位 0.093） | 0/10 | **1.42% / 5.13%** | 0.555 |
| **CustomVoice GGML 1.7B Q8_0 + `vivian`** | 17–36 | 0.091–0.095（中位 0.093） | 0/10 | **1.64% / 5.13%** | 0.079（对英文参考，无意义） |
| 对照：MOSS（04B 项目样例） | 417–1185 | 1.24–1.95 | 17/50 | 36.6% | 0.555 |

读法：

- 两条 1.7B 路径的**可懂度同级**（1.4–1.6%），都远好于 MOSS 的 36.6%；差异主要在**音色来源**（Base 跟参考走，CustomVoice 是官方预设）。
- Base 的首条 232 ms 是参考音色注册/编码器懒加载；暖机后 19–42 ms，符合 ≤800 ms 门槛。
- 10 条全部零欠载；RTF 中位 0.093（≤0.7 门槛，余量很大）。

### 4.1 说话人/音高（客观代理，非主观结论）

`tools/emotion_metrics.py` 对同一句 T11 的 F0 中位数（Hz），用于确认不同 speaker/描述确实产生了不同音色：

| 1.7B CustomVoice speaker | F0 中位 | 时长 s | 备注 |
| --- | ---: | ---: | --- |
| serena | 277.9 | 3.84 | 官方：warm gentle young female |
| vivian | 225.6 | 3.84 | 官方：bright slightly edgy young female |
| uncle_fu | 119.3 | 4.88 | 低沉稳重男声 |
| ryan | 95.4 | 4.08 | 英文母语男声，中文可用 |
| aiden | 157.6 | 3.52 | 英文母语男声 |
| ono_anna | 243.5 | 2.96 | 日语母语女声 |
| sohee | 241.0 | 4.64 | 韩语母语女声 |
| eric | 157.5 | 3.84 | 四川话预设 |
| dylan | 166.9 | 3.44 | 北京话预设 |

> 这些数字只说明"音色确实不同"，不说明"好听/像不像"。主观项见 §6。

### 4.2 VoiceDesign 描述生成音色（GGML 1.7B，18 次）

同一句 T11，中文描述 8 组 × 2 seed + 英文描述 2 组 × 1 seed。原始：`benchmarks/04c/voicedesign-17b/`、`emotion-metrics-vd17b.json`。

| 描述 | F0 中位（seed 42 / 2026） | 时长 s | 观察 |
| --- | ---: | ---: | --- |
| 年轻女性，温柔，偏低，语速适中 | 197.5 / 185.5 | 3.44 / 3.76 | 与 vivian 同量级 |
| 成年男性，沉稳，略带磁性 | 107.2 / 87.8 | 3.76 / 3.52 | 男声 |
| 活泼少女，明亮，稍快 | 351.6 / 379.7 | 3.28 / 3.12 | 高音、短 |
| 成熟女性，知性，新闻播报 | 183.1 / 225.5 | 4.48 / 3.36 | 中音 |
| 老年男性，沙哑、缓慢 | 80.9 / 88.9 | 5.84 / 6.64 | 低音、慢、RMS 更低 |
| 中性化，平静，低情绪 | 81.8 / 100.2 | 3.92 / 3.52 | 偏低（描述"中性"未指定性别） |
| 可爱小女孩，高音，轻快 | 388.7 / 386.8 | 3.36 / 3.92 | 高音 |
| 严肃男性，低音，命令式 | 69.0 / 73.3 | 4.16 / 4.08 | 最低音 |
| **英文描述** young female, gentle | 198.5 | 4.88 | 跨语种 instruct 有效 |
| **英文描述** mature male, deep | 82.0 | 3.12 | 跨语种 instruct 有效 |

**判定**：VoiceDesign 对中文/英文描述都有明显响应，F0 覆盖 69–389 Hz；CER 18/18 = 0%（T11 简单句）；但"像不像描述、是否自然"必须人耳。VoiceDesign 是"没有用户参考时造音色"的可选项，不是本轮默认推荐。

---

## 5. 与 04B 的对照

| 项 | 04B（参考） | 04C（本轮） |
| --- | --- | --- |
| 1.7B Base Q8_0 常驻/峰值 | 3.57 / 3.62 GB | Base 带 speaker encoder：增量 3,461 / 3,592 MiB（约 3.63 / 3.77 GB；整卡 5,367 / 5,498 MiB，桌面基线 1,906 MiB） |
| 1.7B Base TTFA | 20–37 ms | 19–42 ms（冷首条 232 ms） |
| 1.7B Base RTF | 0.091–0.115 | 0.090–0.137（中位 0.093；首条冷 0.137） |
| 1.7B Base CER | 1.44% | 1.42% |
| 克隆模式 | 只测 x-vector | **x-vector vs ICL 对照**；ICL 有音色收益但被自动 ref_text 污染 |
| 中文参考 | 英文 MOSS 音色（自动转写） | **仍无用户中文参考，Pending** |

> 04C 的 1.7B Base 数字与 04B 同级；没有出现"04B 报告 1.7B 数字其实是 0.6B"的迹象（本轮的 1.7B 单独测了 TTFA/RTF/CER/并发，见 `qwen3-tts-concurrency.md`）。

### 5.1 稳定性：20 连发 + speaker 交替

- 配置：GGML 1.7B CustomVoice Q8_0，`vivian` / `ryan` 交替，20 次连续请求（10 条不同文本循环，seed 42/123/2026 轮换），`prebuffer_ms=300`。
- 结果：**20/20 成功，0 欠载、0 崩溃、无服务重启**；TTFA 15.6–40.1 ms（中位 27.6 ms），RTF 0.090–0.096（中位 0.0925），墙钟 339–1460 ms。
- 串音检查（F0 代理）：`vivian` 10 次 F0 中位 **214.2–364.1 Hz**；`ryan` 10 次 **102.7–197.4 Hz**；无一次跨到对方区间，说明交替请求没有把上一轮 speaker/音频带进下一轮。
- 10 组固定样本（Base 与 CustomVoice 各 10 条）也全部成功、零欠载。
- 原始：`benchmarks/04c/stability-cv17b/index.json`、`emotion-metrics-stability-cv17b.json`；`quality-base17b/`、`quality-cv17b/`。

---

## 6. 人工试听（已完成，2026-11-24）

**29/29 条已由哥哥本人盲听并评分**（评分表 `docs/listening-sheet-04c.md` §10；解码表 `docs/listening-key-04c.md` 已可复核）。

- 目录：`samples/04c-listening/A01.wav … A29.wav`
- 覆盖：Base x-vector 5 条、CustomVoice vivian 5 条、其他 speaker 4 条、VoiceDesign 6 条、vivian 情绪 7 条、ICL 1 条、0.6B CustomVoice 1 条。
- 文本：T01/T03/T04/T07/T10 + T11。
- **结果**：
  - 默认 `vivian` 被否：12 条均值自然 2.58/机械 3.25；A28 中性 3/2/2/4/5，A06 sad 4/1/1/3/5，A13 gentle 4/1/3/4/5。
  - 可用候选：VoiceDesign `cute_girl` 5/5/5/5/1、CustomVoice `ryan` 4/4/5/5/1、VoiceDesign `young_female_gentle` 4/4/4/5/2、`mature_male_deep` 4/4/3/5/2。
  - Base x-vector 5 条均值 4.00/3.80/N/A/4.60/2.00；T07 绕口令弱；当前英文参考不是最终中文音色。
  - ICL A04 4/4/3/2/2：可懂度 2，不通过。
  - 0.6B CV A29 3/1/2/3/4：不通过。
- 详细汇总与最终判定：`qwen3-tts-decision-04c.md` §9。

---

## 7. 未完成项与风险

| 项 | 状态 | 影响 |
| --- | --- | --- |
| 用户中文参考音频（5–15 s，干净） | **Pending** | 无法做最终中文音色/ICL 验收 |
| 人工校对的中文 `ref_text` | **Pending** | ICL 不能用；自动转写已证明会漏字/英文起头 |
| 人工盲听 29 条 | **✅ 已完成（2026-11-24）** | 默认 `vivian` 被否；候选见 §6 / decision §9 |
| 0.6B Base 10 组 CER | 未做 | 不影响主推荐（0.6B 仅低显存档；CV 0.6B 简单句 CER 0） |
| 真实 Windows 音频设备播放 | 未在本轮重测 | 04D 需用 `AudioPlayer` 复测 200–300 ms 预缓冲与欠载 |

---

## 8. 原始数据与复现

| 内容 | 路径 |
| --- | --- |
| x-vector/ICL 原始探测 | `benchmarks/04c/clone-ggml17b/index.json` + 每 case JSON/WAV |
| 克隆 CER/余弦 | `benchmarks/04c/quality-clone-ggml17b-eval.json` |
| Base 10 组 | `benchmarks/04c/quality-base17b/`、`quality-base17b-eval.json` |
| CustomVoice 10 组 | `benchmarks/04c/quality-cv17b/`、`quality-cv17b-eval.json` |
| CustomVoice 说话人/情绪 | `benchmarks/04c/customvoice-17b/`、`emotion-metrics-cv17b.json` |
| VoiceDesign | `benchmarks/04c/voicedesign-17b/`、`emotion-metrics-vd17b.json` |
| 0.6B CustomVoice | `benchmarks/04c/customvoice-06b/`、`quality-cv06b-all-eval.json` |
| 匿名试听 | `samples/04c-listening/`、`docs/listening-key-04c.md` |

复现命令（节选）：

```powershell
# 注册 x-vector / ICL
python harness\04c_run.py register --url http://127.0.0.1:8095 --name ref_xvec --wav assets\ref_zh_prompt_audio1_24k.wav
python harness\04c_run.py register --url http://127.0.0.1:8095 --name ref_icl  --wav assets\ref_zh_prompt_audio1_24k.wav --ref-text-file assets\ref_text_auto.txt

# 10 组固定样例（Base / CustomVoice）
python harness\04c_run.py run --url http://127.0.0.1:8095 --cases scenarios\04c_quality_base17b.json --out-dir benchmarks\04c\quality-base17b
python harness\04c_run.py run --url http://127.0.0.1:8096 --cases scenarios\04c_quality_cv17b.json  --out-dir benchmarks\04c\quality-cv17b

# CER / 音色余弦（CPU，GSV runtime）
G:\Audio_Processing\TTS\GPT-SoVITS\GPT-SoVITS-v3lora-20250228-nvidia50\runtime\python.exe tools\eval_quality.py `
  --manifest benchmarks\04c\manifest-quality-cv17b.json --ref-wav assets\ref_zh_prompt_audio1_16k.wav `
  --out benchmarks\04c\quality-cv17b-eval.json
```
