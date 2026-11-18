# 决策表：Qwen3-TTS 是否作为桌面助手常驻 TTS 后端

> 依据：本机 4090 实测（`QWEN3_TTS_BENCHMARK.md`）+ 生态调研（`QWEN3_TTS_RESEARCH.md`）。
> 门槛来自任务书 §9：常驻 ≤4 GB、峰值 ≤5 GB、暖机 TTFA ≤800 ms、`RTF_project` ≤0.7（<1 才稳定）、
> 15 s+ 长句无明显欠载、LLM 并发无明显换页/掉速、情绪控制可验证可复现。

## 1. 方案对比

| 方案 | 常驻/峰值显存 | TTFA | RTF_project | 长句欠载 | LLM 并发 | 取消 | 克隆 | 指令 | 判定 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **GGML qwentts.cpp 1.7B Q8_0** | 3.57 / 3.62 GB | 20–37 ms | 0.091–0.115 | 0 | LLM −58%（TTS 不掉速） | ✅ 断连即停 | ✅ | ⚠️ Base experimental | **推荐主方案** |
| **GGML qwentts.cpp 0.6B Q8_0** | 2.55 / 2.61 GB | 11–36 ms | 0.075–0.080 | 0 | LLM −58% | ✅ | ✅ | ⚠️ 同上 | **推荐低显存档** |
| Torch faster-qwen3-tts 0.6B bf16 | 3.58 / 3.74 GB | 257–286 ms | 0.228–0.248 | 0 | LLM −63% | ⚠️ 需自建 cancel | ✅ | ⚠️ | **备选（纯 pip，无需编译）** |
| Torch faster-qwen3-tts 1.7B bf16 | **5.06** / — GB | 302–327 ms | 0.273–0.287 | 0 | LLM −63% | ⚠️ | ✅ | ⚠️ | **不满足显存预算** |
| MOSS-TTS-Realtime（现状） | ≈12 / 12.3 GB | 417–1185 ms | **1.24–1.95** | **17/50 次** | TTS TTFB → 12 s | ❌ | ✅ | ❌（无字段） | **应替换** |

## 2. 最终建议

1. **主方案：qwentts.cpp（GGML）+ 1.7B Base Q8_0**，常驻 `tts-server`，按段请求 + 流式收 PCM。
   - 显存比 MOSS 少约 8.4 GB（12 → 3.6 GB），`RTF_project` 从 1.2–2.0 降到 0.09–0.12；
   - 唯一需要额外工程的是 **Windows 源码构建**（已跑通，脚本在 `run/build_qwentts_windows.cmd`）。
   - 若显存再紧一档，降到 0.6B Q8_0（2.55 GB），音质代价：音色余弦 0.530 vs 0.543、CER 相同。
2. **备选：faster-qwen3-tts（Torch CUDA Graph）+ 0.6B Base bf16**，只走 pip、不编译。
   - 代价：TTFA 约 0.27 s（仍远低于 800 ms）、RTF 0.23（仍远快于实时）、显存 3.58 GB；
   - **必须**自己补取消端点（本实验的 `servers/qwen3_tts_server.py` 可直接借用）。
3. **不选**：Torch 1.7B（5.06 GB 常驻就超过 5 GB 峰值预算）、官方 qwen-tts（无流式）、
   speech.cpp（无克隆）、HaujetZhao（无 HTTP server）、vllm-omni（Windows 只有 WSL）。
4. **情绪/风格**：Base 的 `instruct` 不稳定（"快"有效、"慢"无效）。若项目确实需要稳定情绪，
   要另测 1.7B **CustomVoice/VoiceDesign**（各约 4.2 GB 权重，本轮未下载）——这是**下一个待验证点**，
   不是"Base 已经能做"。

## 3. 替换 MOSS 前必须做的三件事

1. 在桌面助手侧实现 `qwen3_tts.py` 适配器（方案见 `docs/INTEGRATION_PROPOSAL.md`），
   **预缓冲从 2500 ms 改 200–300 ms**，取消走"断连 + 可选显式 cancel"。
2. 用 `samples/` 里的 10 组样本做一次人工 A/B（尤其 s07 绕口令、s08 技术术语、s10 多句连读）——
   这是本轮唯一没有自动化替代的验收项。
3. 打包/启动方式确认：`tts-server.exe` 需要同目录的 `ggml-cuda.dll` / `ggml-cpu.dll` / `ggml-base.dll`，
   模型 GGUF 放在项目外或随包分发（Q8_0 0.6B 946 MB + tokenizer 278 MB / 1.7B 1.98 GB）。
