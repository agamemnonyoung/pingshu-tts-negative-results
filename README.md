# 用 238 回现场评书录音做风格化 TTS：一次失败的技术复盘

**作者：agamemnon young**

2026 年 11 月，我用一位评书演员（王玥波）的 238 回现场录音（263.5 小时），试图做一套"本地、零 API 成本、能模仿他的腔调念任意文本"的 TTS 流水线。结论是**没有做成**。这套仓库记录的是为什么没做成——具体是哪些数字、哪些办法试过、哪些路是死路。

如果你正打算用"老录音 + 开源 ASR + 开源 TTS"复刻某个说话人，这里的负结果应该能帮你省掉一些时间和 token 成本。

## 一句话结论

**在长文本语音模拟里，文本正确性压倒音色；而开源 ASR 在这类语料上的错误率，让"自动化生产"从一开始就不成立。**

我们把 15 分钟的一段书做到"听感够用"，代价是：人工校对 3 小时 + 机器时间约 1.5 小时（时间均为从范例反推得出）。照这个比例，238 回需要 4000 小时以上的人工，项目在经济学上就结束了。

## 三个最硬的数字

| 环节 | 数字 | 含义 |
|---|---|---|
| ASR 转写 | 最佳 17.2% 字错率（paraformer-zh），人工改后仍有 36% 的句子有错 | 一段 60 分钟的书，平均每 3 句就有 1 句要动 |
| 降噪 | 先降噪再识别，字错率 17.2% → 30.2% | "先修复音频再识别"是被证伪的直觉 |
| TTS 速度 | IndexTTS-2.5 RTF 10–12×；Breeze-TTS-2 RTF 1.6–2.0 | 一小时的成片，前者要 11 小时算力，后者只要 2 小时 |

## 目录

| 文件 | 内容 |
|---|---|
| [01-corpus-and-quality.md](01-corpus-and-quality.md) | 238 集语料的质量分档：用无参考 MOS 模型给每集打分，与人工判断零逆序 |
| [02-asr-benchmark.md](02-asr-benchmark.md) | 同一金标准下的 ASR 横评：paraformer / Qwen3-ASR / Whisper large-v3 / 多模型投票 |
| [03-denoise-results.md](03-denoise-results.md) | 两种降噪路线的结局：轻抑制有用但有限，生成式增强 PESQ 飙升却在改内容 |
| [04-tts-selection.md](04-tts-selection.md) | 三个开源 TTS 的实测：IndexTTS-2.5 / GPT-SoVITS v2Pro / Breeze-TTS-2 |
| [05-engineering-pitfalls.md](05-engineering-pitfalls.md) | 十五条踩过的工程坑，附复现条件 |
| [numbers.csv](numbers.csv) | 所有实测数字，一条一行 |

## 给后来者的三条建议

1. **先量人工成本，再谈模型。** 决定项目生死的是"每 15 分钟成片要人改多少字"，不是 MOS 分数。我们在做完 TTS 之后才回头量这个数，顺序错了。
2. **不要用降噪做识别的前置。** 现代 ASR 对平稳噪声相当鲁棒，而降噪会引入失真；我们三种降噪路线全部使字错率上升。
3. **"像不像"可以被度量，但要用对指标。** 无参考 MOS（如 SQUIM）与人工听感在这批语料上零逆序；而我们自己造的各种手工指标（底噪、共振峰、谱平坦度）全部预测失败——因为它们测的是"录音干不干净"，不是"人声好不好听"。

## 边界说明

- 本仓库**不包含任何音频**，也不包含任何用该演员声音合成的产物。
- 文中保留演员姓名，仅用于说明语料来源与风格对象。
- 所有结论来自自用、非商业的个人实验。

## English summary

We tried to build a local, zero-API-cost TTS pipeline that reads arbitrary text in the storytelling style of a Chinese *pingshu* performer, using 238 live recordings (263.5 h) as both text source and style reference. It did not work, and this repository documents why: the bottleneck was text accuracy, not voice quality. Best-case ASR (17.2% CER) still left 36% of sentences needing manual edits; denoising before ASR made things worse (17.2% → 30.2%); and the best-sounding open TTS ran at 10–12× real time on an 8 GB GPU. Producing 15 minutes of acceptable audio cost 3 hours of human proofreading plus ~1.5 hours of compute. Scaling that to 238 episodes is roughly 4000+ human hours — the project ends there, not in the model.

## 许可

本仓库文字内容采用 [CC BY 4.0](LICENSE) 许可，转载或改编请署名 **agamemnon young**。
