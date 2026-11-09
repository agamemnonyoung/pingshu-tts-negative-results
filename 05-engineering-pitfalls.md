# 05 · 十五条工程坑

每条都写清**症状 → 原因 → 修法**，按"会不会浪费你半天"排序。

## 1. TTS 输出的 int16 被再乘一次 32767 → 白噪声

**症状**：生成的音频"全是机械噪音、震耳朵"，峰值贴着 0 dB，但生成过程无报错、时长也对。

**原因**：`GPT-SoVITS/TTS_infer_pack/TTS.py` 的 `audio_postprocess` 返回的已经是 int16 量级（内部做过 `* 32768`），我们按"浮点 −1..1"又做了一次 `clip(*1)*32767`，整数回绕成方波。

**修法**：先看 dtype，整数就直接当 PCM 用：

```python
pcm = audio.astype("<i2") if np.issubdtype(audio.dtype, np.integer) \
      else (np.clip(audio, -1, 1) * 32767).astype("<i2")
```

**通用规矩**：交付音频前必须过两道客观检查——`ffmpeg -af volumedetect`（峰值是否贴 0 dB）与 ASR 回读（内容是否可懂）。"文件生成成功、时长正确"不是证据。

## 2. 用 PowerShell 改中文源码 → 整份文件变乱码

**症状**：`Get-Content -Raw | ... | Set-Content -Encoding utf8` 之后，源码里所有中文变成 `鍒囩墖` 之类的乱码，引号被破坏，脚本语法错误。

**原因**：PowerShell 5/7 读文件时默认按 ANSI(GBK) 解码，写出时按 UTF-8 编码，一进一出就损毁。

**修法**：**改代码只用编辑器/写入工具**，永远不要用 shell 管道读写源文件。这个坑我们踩了两次。

## 3. 思考型模型把 max_tokens 全烧在思维链上

**症状**：纠错脚本"零改动"，耗时 188 秒，`content` 返回空字符串，但 `finish_reason: length`。

**原因**：模型默认输出 `reasoning_content`，token 预算被思维链吃光。

**修法**：请求体加 `"chat_template_kwargs": {"enable_thinking": false}`。单块耗时从 12.1 秒降到 1.0 秒。

**附带一条**：prompt 里必须给 few-shot 例句，否则模型只加标点、不改错字。

## 4. 参考音频被硬编码在 3–10 秒

**症状**：换 15 秒或 25 秒参考直接 `OSError: 参考音频在3~10秒范围外，请更换！`

**原因**：`TTS.py:918` 写死的检查，不是配置项。

**后果**：所有"用一段长录音学腔调"的方案在 GPT-SoVITS 上不成立。IndexTTS 上限 15 秒（内部还会再截前 15 秒），Breeze 建议 10 秒。

## 5. 用标点切句会切出"假切点"

**症状**：生成的音频在该断的地方不断、不该断的地方喘一口，听感"碎"。

**原因**：按标点/VAD 切，会把"停顿 0.00 秒"的地方当切点。我们对 47 个切点做过审计：**13 个（28%）落在实测停顿 0.00 秒处**——纯标点切出来的。

**修法**：切点只认声学停顿 ≥ 0.6 秒，且不要切在虚词之后（`的地得了着过和跟把被在是将就而却`）。改用"最少 60 字 + 长停顿"后，切点从 47 个降到 6 个，听感明显变好。

## 6. 用 atempo 降速，听起来像快进

**症状**：把 3.6 字/秒压到 3.0 字/秒后，人耳反馈"像音频快播"，不是"人说话变慢"。

**原因**：人加速靠**吃掉停顿、把短句连成长句**（实测：慢版平均浊段 0.494 s、24 段；快版 0.591 s、16 段）；而 atempo 是把所有音素等比例压短（0.417 s、22 段），语速分布完全变了。

**修法**：降速只能靠"文本层加停顿 + 指令层改语气"，不能靠后期变速。atempo 只适合最后的 ±5% 微调。

## 7. 硬链接目录拖进回收站，空间一点都不释放

**症状**：删了 6.2 GB 的硬链接分组目录，磁盘可用空间没变。

**原因**：回收站是"移动到同卷的 `$Recycle.Bin`"，硬链接仍然占用同一份数据块。

**修法**：硬链接目录用真删除；只有真文件才走回收站。

## 8. FunASR 的时间戳拿法（三个连环坑）

```python
res = model.generate(input=f, sentence_timestamp=True, punc_model="ct-punc")
```

1. `sentence_timestamp=True` **必须同时给 `punc_model`**，否则没有时间戳；
2. 给了 punc 之后 `res["sentence_info"]` 是 `None`，时间戳在 `res["timestamp"]`；
3. `timestamp` 是 `[[start_ms, end_ms], ...]` **纯区间、不带字**，要和"去掉标点后的字序列"一一 zip。

## 9. 快速合成出来的音频不能用来校验文本

**症状**：把 3.3 字/秒的合成音频送 ASR 回读，出现了原文里没有的错字，误判为"文本有错"。

**原因**：语速偏快时 ASR 自己会掉字（实测同一段在 3.3 字/秒下回读出现"这钱危刀兵马乱"）。

**修法**：用 ASR 校验 TTS 输出时，先把语速降到 ≤2.8 字/秒；否则校验结果里混着校验器自身的错误。

## 10. Qwen3-ASR 默认按 20 分钟一块喂，直接 OOM

**症状**：`torch.OutOfMemoryError: Tried to allocate 8.80 GiB`。

**原因**：库默认 `MAX_ASR_INPUT_SECONDS = 1200`。

**修法**：

```python
import qwen_asr.inference.qwen3_asr as q
q.MAX_ASR_INPUT_SECONDS = 60   # 按低能量点切块
```

改后 20.4 分钟音频 75 秒跑完（16× 实时），峰值 4–5 GB。

## 11. faster-whisper 在本机的两个启动错误

- `TypeError: open() got an unexpected keyword argument 'metadata_errors'`（PyAV 版本不匹配）→ 绕过解码，直接把 `float32` numpy 波形传给 `transcribe()`；
- `RuntimeError: Library cublas64_12.dll is not found or cannot be loaded` → 需要 `os.add_dll_directory(r"<venv>\Lib\site-packages\torch\lib")`，并把它加进 `PATH`。

另外：`vad_filter=True` 在这批语料上会丢掉 20% 的内容。

## 12. 降噪门的阈值定太高，会把"气口"抹平

**症状**：换了"净化过"的参考音频后，生成的音色反而变"光滑、平均、不像"。

**原因**：我们设了 −42 dB 门限，把参考里所有低电平帧压到 2%。原始参考的 p10 是 −71 dB（那是换气、唇齿声、房间余音），压完变成 −180 dB（数字静音）。模型从"有呼吸的录音"里学不到气口，只能退化成平均音色。

**修法**：只在**长静音（≥150 ms）**上做门限，字内一律不动。

## 13. 从 GitHub Release 下的 exe 被 Defender 判为木马

**症状**：解压后几秒，`breeze-cli.exe` 消失，安全中心显示 `Trojan:Win32/Malgent`（ThreatID 2147794830）。

**判断**：高概率误报——同包的 `breeze-server.exe`、`libbreeze.dll`、`breeze-convert.exe` 都完好，且与 CLI 链接同一个库；该文件从未运行过就被静态扫描拦下；`Malgent` 是启发式泛称（llama.cpp 的 ggml DLL 常报 `Win32/Wacatac.H!ml`）。

**处理**：不恢复、不加白名单，改用同包的 server（HTTP 接口功能更全）。若确实需要 CLI，重下 zip、比对 SHA256、丢 VirusTotal。

## 14. 大文件下载：`Invoke-WebRequest` 会静默卡死

**症状**：192 MB 的文件停在 87.9 MB 不动；2.7 GB 的 torch wheel 停在 1198 MB；之后 `Expand-Archive` 报"文件正由另一进程使用"。

**修法**：改用 `curl.exe -L --retry 5 -C -`（断点续传），或者用支持流式写盘的下载器。

**顺带一条测速教训**：同一份 torch wheel，阿里云镜像 0.23 MB/s，PyTorch 官方源 35.4 MB/s。**批量下载前先测三个源再决定**，不要凭"国内镜像更快"的直觉。

## 15. 跨盘符的工作目录会让某些库直接抛异常

**症状**：`ValueError: path is on mount 'C:', start on mount 'F:'`

**原因**：`tools/i18n/i18n.py` 里用 `os.path.relpath(__file__)` 算相对路径，源码在 C 盘而进程 cwd 在 F 盘时无解。

**修法**：把进程 cwd 设到代码所在盘（我们设成 `C:\AI\apps\GPT-SoVITS`）。这只是单条命令的 `workdir`，不是"工作区"。

## 附：Excel 富文本的读法

如果让人在 xlsx 里做校对（删除线=删、红色=新增），必须用：

```python
wb = openpyxl.load_workbook(path, rich_text=True)
```

否则 `cell.value` 只返回拼接后的纯文本，看不到每个 run 的 `font.strike` 与 `font.color`。判断颜色用 `str(run.font.color.rgb).endswith("FF0000")`。
