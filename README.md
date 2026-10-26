# 声幕 · Sonic Script

本地中文歌曲分析工作室。上传 MP3，自动分离人声、识别歌词、分析节拍和主副歌，再用可编辑的时间轴连接代码 MV。

## 现在可以做什么

- 上传 MP3 / WAV / FLAC / M4A / OGG / Opus / AAC，最大 100MB，3 秒至 11 分钟。
- Demucs HTDemucs 分离四轨；Whisper 在真实人声轨上自动听写中文歌词和词级时间。
- All-In-One 单折 ONNX 模型输出歌曲段落、主副歌、节拍与小节重拍。
- pYIN 从人声轨估计旋律音高曲线，提供 Hz、MIDI 音高和置信度。
- 原曲 / 人声试听、点击波形跳转、单句试听、歌词和起止时间编辑、段落拆分与标签校正。
- 可选标准歌词：匹配 ASR 拼音时间锚点，未匹配部分插值并标明需要复核。
- 保存本地项目，导出 MV JSON / JSON / LRC / SRT。
- Canvas 代码动画随歌词、拍点和主副歌同步预览。

第一版是**音乐分析与时间轴编辑器**。Canvas 提供同步视觉预览，当前不导出完整 MV 视频。

## 当前电脑直接启动

双击 `start.cmd`，浏览器打开 http://127.0.0.1:8765 。关闭启动终端可停止服务。已有服务运行时，再次启动只会打开页面。

## 搬到另一台 Windows 电脑

1. 安装 64 位 Python 3.12：https://www.python.org/downloads/windows/ 。勾选添加到 PATH 或安装 Python Launcher。
2. 复制本文件夹，**不要复制 `.venv`**（虚拟环境有绝对路径，不能直接搬）。保留 `models` 可以避免重复下载已有模型；保留 `data` 可以带走分析项目。
3. NVIDIA GPU 电脑双击 `install-nvidia.cmd`；没有 NVIDIA CUDA 的电脑双击 `install-cpu.cmd`。
4. 安装完成后双击 `start.cmd`。

建议目标配置：32GB 内存、16GB 显存的 NVIDIA 显卡。GPU 安装脚本安装 PyTorch 2.8 CUDA 12.8，并预下载 Whisper Large v3；需要支持该 CUDA 运行时的 NVIDIA 驱动。分轨与歌词模型顺序运行，尽量释放上一阶段的显存。All-In-One ONNX 和 pYIN 目前仍在 CPU 上运行。

建议预留至少 12GB 磁盘空间用于 Python、GPU 依赖、模型与分析缓存。CPU 安装使用本机已验证的 PyTorch 2.14.1 / torchaudio 2.11.0，完整依赖快照在 `requirements-tested-cpu.lock.txt`。

安装脚本包含 CUDA 可用性与一次真实解码冒烟检查。**当前开发电脑只有 CPU，目标电脑的 CUDA 分支尚未实机验证**；若安装检查失败，先用 CPU 模式，查看输出错误。AMD / Intel 的 16GB 显卡不能使用这个 NVIDIA 安装入口。

CPU 安装默认下载 Whisper Turbo；Small 保留为快速调试选项。Large v3 和 Turbo 需要 128 维音频特征配置，安装准备会一并下载对应的 `preprocessor_config.json`。

也可以在 PowerShell 中运行：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\setup.ps1 -Device cpu
# 或 NVIDIA GPU
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\setup.ps1 -Device cuda
```

首次安装会联网下载开源依赖和模型。日常分析只在本机运行，不调用外部音频识别 API。模型缺失时切换到新模型会下载对应权重。

## 结果与精度边界

- 自动听写可能出现错字、漏句、伴唱混合或间奏幻听，尤其是老录音、混响、Rap。置信度不是经过校准的准确率。
- 识别前按分离人声的能量切段，保留时间偏移和两侧余量，避开长静音前奏 / 尾奏的幻听；不使用容易误删歌唱的通用语音 VAD。极弱人声仍可能需要人工校正。
- 主副歌是模型预测，未人工标注的真实歌曲不应宣称 100% 正确。当前使用 harmonix-fold0 单折，不是八折集成。
- 节拍改用 Beat This! 三个模型串行核验、主拍层级选择和局部序列解码。多模型仍可能共同出错，不能保证无漏拍或重拍错位。
- 工作台按已有单字候选时间逐字点亮，使用音频时钟同步；未知字级边界以虚线标出，多字单元不按时长均分。
- 标准歌词校正使用 ASR 锚点与拼音匹配，**不是 WhisperX 的 CTC 强制对齐**。不匹配的歌词会拒绝对齐，较低匹配片段需人工核对。
- pYIN 是单音基频估计。人声分离残留、和声和清辅音会造成错误或无音高点；不能当作完整编曲转谱。
- 小于 76 秒或大于 660 秒超出 All-In-One 原训练时长范围；第一版允许较短文件做测试，但结果可靠性会降低。

## 数据格式

`data/jobs/<项目ID>/result.json` 是完整时间轴。单位统一为秒，以解码后完整歌曲开头为零点：

```json
{
  "schema_version": "1.0",
  "duration": 210.0,
  "bpm": 120,
  "beats": [0.5, 1.0],
  "downbeats": [0.5, 2.5],
  "sections": [{"start": 0, "end": 20, "label": "intro", "review": true}],
  "lyrics": [{"start": 20, "end": 24, "text": "示例歌词", "words": [], "review": true}],
  "melody": [{"time": 20.1, "hz": 440, "midi": 69, "confidence": 0.8}]
}
```

每种结果保留算法来源；修改后的歌词需保存再导出。`result.json` 有递增 revision，防止多个页面覆盖彼此修改。

本地项目还保留原音频、试听 MP3、人声轨、四轨缓存、原始 ASR 和结构激活，方便检查与重跑。四轨缓存较大，一首歌可能占用数百 MB。第一版不自动删除用户项目。

## 测试

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

真实中文歌曲来源在 [samples/SOURCES.md](samples/SOURCES.md)。实际运行结果在 [VERIFICATION.md](VERIFICATION.md)。

另外加入了五首中文歌曲与网上 LRC 的独立核验，比较 Turbo 与完整 Large v3 的字错误率、漏句和句首偏差。方法与复跑方式见 [BENCHMARK.md](BENCHMARK.md)，实测结果见 [ACCURACY_REPORT.md](ACCURACY_REPORT.md)。模型更大不保证每首歌曲都更准确，验证方案不会因模型更大就自动覆盖工作室原结果。

针对准确性增加了可选实验模式 **Qwen 纠词 + Turbo 时间**：先运行 `install-accuracy.cmd`，然后在模型菜单选择该模式。两个识别器逐个加载，只处理当前一首；Qwen 提供文字，Turbo 提供拼音时间锚点。当前一首实测字错误率从 6.6% 降为 4.0%，时间偏差仍约半秒，不能推断每首歌都改善。GPU 路径仍需在目标电脑实测。

第一轮使用《恭喜恭喜》的历史录音，另加入现代中文流行歌测试，均只输入 MP3、不注入标准歌词。模拟 / 单元测试用于检查数据校验和接口，真实歌曲测试另行记录，二者不混为准确率评测。

## 项目结构

```text
app/          本地 API、分轨、结构推理、歌词与旋律分析
web/          无构建步骤的中文工作室界面
scripts/      安装准备、启动、真实歌曲验证
tests/        接口、时间轴校验与算法边界测试
models/       本机模型权重（不入源码）
data/jobs/    本机歌曲与分析项目（不入源码）
```

时间轴优先的专项实验见 [PRECISION_REPORT.md](PRECISION_REPORT.md)：当前仍有漏定位和大偏差，尚无方案达到整曲逐字精准。工作台等字数改词会保留已有词级时间，增删字不会伪造新字时间。

JSON 另外提供 `character_timeline`、`beat_timeline` 和 `timing_quality`。未知字级起止为 `null`，自动候选全部标为未验证；模型置信度、界面“已复核”和节拍内部一致性都不当作零误差证据。`/export/json?require_verified=true` 等严格导出目前返回未通过验收，常规导出明确为草稿。歌词按实际发声、拍点按音乐节拍分别定义，不把每个字强行吸附到拍点。

开源组件、许可证和模型转换来源见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

## 给其他模型制作 MV

导出面板新增 **MV JSON**，接口为 `/api/jobs/<id>/export/mv`。包含原音频 SHA-256、解码采样率和实际采样数、独立拍点、逐字候选与整首歌逐秒事件索引。旧项目未知的采样数和文件哈希保持为空。下游应使用同版本音频和同一零点，逐个读取事件时间，不按全局 BPM 补拍，不均分缺失字的时间。采样坐标只是候选时间的表示，不能据此声称检测精确到采样点。

五首中文歌的专用节拍核验见 [BEAT_REPORT.md](BEAT_REPORT.md)。现有安装可运行 `install-beats.cmd` 补装节拍模型。

可选歌唱对齐实验见 [SINGING_REPORT.md](SINGING_REPORT.md)，安装入口 `install-singing-experiment.cmd`。TIFA 预训练模型为 CC-BY-NC-SA-4.0，本项目不打包其模型或上游源码，也未把它设为默认对齐器。

新增《夜空中最亮的星》《晴天》两首中文 MP3 的逐字起止对比，分别测试自动歌词和校正文字。HubertFA、TIFA 与 SOFA 的改善和失败都记录在 [WORD_TIMING_REPORT.md](WORD_TIMING_REPORT.md)，尚未达到 95% 整体时间准确率，不能把社区参考下的误差分档当作正式验收。

已安装可选 TIFA、HubertFA 实验模型后，可运行 `python -X utf8 -m scripts.refine_job_timing 项目ID`。该流程只读取现有项目的歌词与音频，两个模型依次运行，创建一个独立的“歌唱逐字复核（实验）”项目，工作台原有导出可查看模型候选和冲突原因。它不下载模型、不读取网上歌词/时间戳，也不修改原项目；当前精校后端使用 CPU，GPU 尚未实测。安装与复跑命令见逐字报告。

面向其他 AI 的交付入口新增 **AI 复核包 ZIP**：包含原曲、人声、伴奏、固定时钟混音、完整独立拍点、全部逐字候选、音频时钟与逐秒检查、可疑清单、备选时间、段落、旋律及复核说明。17 个文件通过清单与 SHA-256 绑定，供其他 AI 以代码编排 MV 并逐项听审。95% 为逐项验收目标，当前没有独立声学真值，不会写成已通过。格式、指标和复核方法见 [AI_REVIEW_EXPORT.md](AI_REVIEW_EXPORT.md)。

