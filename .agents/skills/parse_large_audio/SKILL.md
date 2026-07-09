---
name: parse_large_audio
description: Handles requests to transcribe, summarize, or extract information from large audio files (mp3, wav, m4a, flac) by converting them to an mp4 container first to bypass MIME restrictions.
---

# 目标
当用户要求你处理（转录、提炼、总结）一个纯音频文件（如 `.mp3`, `.wav`, `.m4a`, `.flac`）时，必须优先触发此技能，以绕过底层系统的 MIME type 解析限制。

# 核心经验与避坑指南 (AI的易错点反思)
1. **盲目直接读取必定报错**：不要试图直接对纯音频格式文件执行 `view_file`，这会导致 `unsupported mime type audio/mpeg` 或 `audio/wave` 等死循环报错。
2. **转码为其他音频格式无济于事**：在过往的尝试中，将 mp3 转换为 wav 或 flac 均失败，依然会被系统判定为不支持的媒体流（如 `application/octet-stream`）。
3. **“偷梁换柱”的终极解法**：平台系统对视频容器格式（`video/mp4`）支持极好，且在解析视频时能完美识别并提取其中的音频轨道。因此，将音频重新打包封装进 `.mp4` 容器是最高效、最可靠的破局之道。

# 执行步骤指南
当你需要深度解析一个音频文件时，请严格按照以下标准化步骤操作，避免无效尝试：

1. **第一步：立即启动转码打包任务**
   - 不要尝试直接读取，立刻使用 `run_command` 工具调用 `ffmpeg`。
   - 使用 `aac` 编码器并将码率适当压低（如 64k，以防文件过大影响上下文读取），将音频流封装成 mp4。
   - **命令模板**：`ffmpeg -i "原始文件路径.mp3" -c:a aac -b:a 64k "新文件路径.mp4"`
   - 务必设置合适的 `WaitMsBeforeAsync` （如 5000），让转码任务在后台安全运行。

2. **第二步：向用户安抚与反馈进度**
   - 在转码任务后台运行期间，主动回复用户，告知你遇到了平台对音频直读的限制，但你**已经采用了“封装为mp4格式”的技巧来绕过限制**，让用户放心稍等。
   - 如果任务复杂，可借此等待时间向用户询问一些必要的业务参数（如查经目标受众、主要发言人名字、核心关注的主题等）。

3. **第三步：解析 MP4 并交付高质量成果**
   - 当系统通知你 `ffmpeg` 任务完成后，使用 `view_file` 工具读取生成的 `.mp4` 文件。
   - 此时，你将能在当前上下文中直接获得系统自动转换出的带有时间戳的完整逐字稿（Raw Transcript）。
   - 最后，根据用户的具体指令，对这份庞大的文本进行智能清洗、逻辑重构和重点提取（如：去除口水话、分离发言人、提炼结构化大纲等），并通过 `write_to_file` 或直接输出 Markdown 交付给用户。
