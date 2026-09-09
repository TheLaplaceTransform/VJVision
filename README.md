# VJVision

> 实时音频识别 + 音频响应可视化，专为 DJ 现场设计。
> Real-time audio recognition + audio-reactive visualizer, designed for DJ live sets.

VJVision 监听 DJ 台的输出音频，自动识别当前播放的曲目，并在第二块屏幕（投影/LED 墙）上展示随音乐律动的频谱与专辑封面动画。
VJVision listens to the DJ booth output, auto-recognises the currently playing track, and renders a music-reactive spectrum + album cover animation on a second display (projector / LED wall).

---

## 功能特性 / Features

- **自动曲目识别**：基于 Dejavu 音频指纹算法，实时识别正在播放的歌曲（支持 12 秒采样窗口）
- **音频响应可视化**：pygame 渲染的频谱柱、波形、镜像三种样式，封面随节拍旋转
- **GPU 硬件加速**：SDL2 GPU 渲染 + 垂直同步，原生分辨率全屏不撕裂、高分辨率屏稳定 60fps（可在控制台开关，失败自动回退软件渲染）
- **混音过渡脉动**：检测到 DJ 切歌/混音时，视觉进入脉动过渡状态，平滑切换到下一首歌
- **中英双语界面**：右上角可随时切换中文/英文，PEAK 指示灯始终显示 PEAK
- **多显示器支持**：控制界面在主屏，可视化窗口投放到副屏
- **便携部署**：打包为单个 exe，带上 `data/` 文件夹即可在任意机器上运行（含已分析的指纹库）

- **Automatic track recognition** — Dejavu audio fingerprinting, real-time (12 s sampling window)
- **Audio-reactive visualizer** — pygame bar / wave / mirror spectrum styles, cover rotates with the beat
- **GPU hardware acceleration** — SDL2 GPU rendering with vsync: tear-free native-resolution fullscreen, steady 60fps on hi-DPI screens (toggle in the console; auto-falls back to software rendering)
- **Mix-transition pulse** — detects DJ cross-fades and pulses the display for a smooth switch
- **Bilingual UI (中文 / English)** — switch from the top-right corner; PEAK indicator always shows "PEAK"
- **Multi-display support** — control panel on primary, visualizer on secondary
- **Portable deployment** — single exe plus a `data/` folder runs anywhere (fingerprint DB included)

## 系统要求 / Requirements

| 项目 Item | 要求 Requirement |
|------|------|
| 操作系统 OS | Windows 10 / 11；macOS 11+（Intel 与 Apple Silicon 均提供 DMG / DMGs for both Intel & Apple Silicon） |
| Python（开发 dev） | 3.13+（Windows）；macOS 源码运行需先 `brew install portaudio` |
| 音频输入 Audio input | Windows：支持 WASAPI / DirectSound / MME 的声卡或虚拟音频线；macOS：USB 声卡 / 麦克风 / 线路输入直连可用，内录系统声音需 BlackHole（见下文） |
| 显示器 Displays | 至少 1 块；推荐 2 块（控制 + 投影） |

## 快速开始 / Quick Start（exe 用户 / for exe users）

1. 从 [Releases](https://github.com/ichiryu0021/VJVision/releases) 下载最新 `VJVision.exe`
   Download the latest `VJVision.exe` from [Releases](https://github.com/ichiryu0021/VJVision/releases)
2. 双击运行，首次启动会在 exe 同级目录生成 `data/` 文件夹
   Double-click to run; first launch creates a `data/` folder next to the exe

### macOS 用户 / macOS users

1. 从 [Releases](https://github.com/ichiryu0021/VJVision/releases) 下载与芯片对应的 DMG：Apple Silicon（M1/M2/M3/M4）→ `VJVision-macos-arm64.dmg`；Intel → `VJVision-macos-x86_64.dmg`
   Download the DMG matching your Mac from [Releases](https://github.com/ichiryu0021/VJVision/releases): Apple Silicon (M-series) → `VJVision-macos-arm64.dmg`; Intel → `VJVision-macos-x86_64.dmg`
2. 打开 DMG，把 **VJVision** 拖入「应用程序 / Applications」。首次启动请**右键点按 → 打开**（应用未做苹果签名，双击会被 Gatekeeper 拦截，右键打开一次后即可正常启动）
   Open the DMG and drag **VJVision** to Applications. On first launch **right-click → Open** (the app is not Apple-notarized, so double-click is blocked by Gatekeeper; after one right-click open it launches normally)
3. 首次运行会在 `VJVision.app` 同级目录生成 `data/` 文件夹（指纹库、封面缓存等便携数据，拷贝该文件夹即可迁移分析结果）
   First launch creates a `data/` folder next to `VJVision.app` (portable fingerprint DB / cover cache; copy the folder to migrate analysis results)
4. **录制电脑内部播放的声音**（识别 Mac 自己放的歌）：macOS 没有系统级内录 API，需安装一次免费开源的 [BlackHole](https://existential.audio/blackhole/) 虚拟声卡（`brew install --cask blackhole-2ch`），并在「音频 MIDI 设置」中创建多输出设备让声音同时送往耳机和 BlackHole，然后在 VJVision 设备下拉里选 BlackHole。**USB 声卡 / 麦克风 / 线路输入无需任何额外安装**，直接在下拉框选择即可
   To capture the Mac's own audio output: macOS has no system loopback API — install the free open-source [BlackHole](https://existential.audio/blackhole/) virtual driver once (`brew install --cask blackhole-2ch`), create a Multi-Output Device in Audio MIDI Setup so sound goes to both your headphones and BlackHole, then pick BlackHole in VJVision's device dropdown. **USB soundcards / mics / line-in need no extra setup** — just select them in the dropdown

### 使用流程 / Workflow

```
添加曲库 → 分析指纹 → 选择音频设备 → 开始捕获 → 看副屏可视化
Add library → Analyze fingerprints → Select audio device → Start capture → Watch the visualizer
```

1. **曲库准备 / Library Prep**
   - 点击「+ 添加文件夹 / + Add Folder」选择音乐目录，或「+ 添加文件 / + Add Files」逐个添加
     Click 「+ Add Folder」to choose a music directory, or 「+ Add Files」to add files one by one
   - 支持格式 Supported formats: `.flac` `.wav` `.mp3` `.aiff` `.aif` `.ogg`
   - 点击「分析队列 / Analyze Queue」开始生成音频指纹（首次分析较慢，之后增量）
     Click 「Analyze Queue」to generate audio fingerprints (first run is slow; subsequent runs are incremental)
   - 分析完成后状态显示「曲库：N 首 / Library: N tracks prepared」
     When done, the status shows 「Library: N tracks prepared」

2. **音频设备 / Audio Device**
   - 在「音频设备 / Audio Device」下拉框选择监听 DJ 输出的设备（如声卡输入或虚拟音频线）
     From the 「Audio Device」dropdown, select the device monitoring the DJ output (e.g. soundcard input or virtual audio cable)
   - 观察「输入电平 / Input Level」表，确认有信号输入
     Watch the 「Input Level」meter to confirm a signal is present

3. **可视化 / Visualizer**
   - 「可视化显示 / Visualizer Display」选择副屏（默认第 2 块显示器）
     In 「Visualizer Display」, select the secondary screen (default: 2nd display)
   - 可设频谱样式、旋转速度、是否随节拍旋转
     Adjust spectrum style, rotation speed, and beat-reactive rotation
   - 可上传待机 LOGO 图片（识别到第一首歌前显示）
     Upload a standby LOGO image (shown before the first track is recognised)
   - 「GPU 硬件加速」开关：显卡渲染 + 垂直同步，切换后自动重启可视化窗口
     The 「GPU hardware acceleration」toggle enables GPU rendering with vsync; the visualizer restarts automatically after switching
   - 「演示模式」复选框：无音频输入时旋转随机封面，用于测试渲染效果
     The 「Demo mode」checkbox rotates a random cover without audio input, for testing the render
   - 点击可视化窗口后按 **F / F11** 全屏（原生分辨率），**Esc** 退出全屏
     Click the visualizer window, then press **F / F11** for native-resolution fullscreen; **Esc** exits fullscreen

4. **开始 / Start**
   - 点击「开始采集 / Start Capture」，副屏即出现可视化窗口
     Click 「Start Capture」— the visualizer window appears on the secondary screen
   - 播放音乐，几秒后自动识别并显示歌曲封面
     Play music; after a few seconds the track is auto-recognised and its cover is shown

5. **语言切换 / Language**
   - 右上角下拉框选择「中文」或「English」，即时生效
     Use the top-right dropdown to choose 「中文」 or 「English」; changes apply immediately

## 从源码运行 / Run from Source

```bash
git clone https://github.com/ichiryu0021/VJVision.git
cd VJVision
pip install -r requirements.txt
python main.py
```

macOS 源码运行前需先安装 PortAudio（vanilla pyaudio 编译依赖）/ On macOS install PortAudio first:

```bash
brew install portaudio
```

## 打包 / Build

Windows（产物 / output: `dist/VJVision.exe`）：

```bash
python -m PyInstaller VJVision.spec --noconfirm --clean
```

macOS（产物 / output: `dist/VJVision.app`）：

```bash
pip install pyinstaller
pyinstaller VJVision.spec --noconfirm --clean
# 可选：打成 DMG / optional DMG packaging:
hdiutil create -volname VJVision -srcfolder dist/VJVision.app -ov -format UDZO VJVision-macos.dmg
```

> macOS 的 DMG 由 GitHub Actions 自动构建：推送 `v*` 标签后，云端 Mac（Intel + Apple Silicon）自动打包并上传到对应 Release，无需本地有 Mac。
> macOS DMGs are built automatically by GitHub Actions: pushing a `v*` tag builds on cloud Macs (Intel + Apple Silicon) and uploads both DMGs to the Release — no local Mac required.

或使用一键发布脚本（Windows：构建 + 提交 + 推送 + 创建 GitHub Release）：
Or use the one-click release script (Windows: build + commit + push + create GitHub Release):

```powershell
.\release.ps1 -Version 1.2.0-beta -Notes "GPU acceleration beta"
```

## 数据目录说明 / Data Directories

| 路径 Path | 说明 Description | 便携 Portable |
|------|------|--------|
| `data/fingerprints.db` | 音频指纹库（SQLite）/ Audio fingerprint DB (SQLite) | ✅ |
| `data/song_paths.sqlite` | 歌曲 ID → 文件路径索引 / Song ID → file path index | ✅ |
| `data/covers/` | 专辑封面缓存 / Album cover cache | ✅ |
| `data/vjvision.log` | 运行日志 / Runtime log | ✅ |
| `%APPDATA%/VJVision/prefs.json`（Windows）<br>`~/Library/Application Support/VJVision/prefs.json`（macOS） | 音频设备、显示器、语言等本机偏好 / Per-machine prefs (device, display, language) | ❌ 每台机器独立 / per-machine |

> 在 A 电脑分析完曲库后，把 `VJVision.exe` + `data/` 一起拷到 U 盘，插到 B 电脑即可直接使用，无需重新分析。
> After analysing the library on PC A, copy `VJVision.exe` + `data/` to a USB stick and run on PC B — no re-analysis needed.

## 识别参数 / Recognition Tunables

在 `vjvision/config.py` 的 `CaptureConfig` 中可调整：
Tunable in `vjvision/config.py` → `CaptureConfig`:

| 参数 Param | 默认 Default | 说明 Description |
|------|--------|------|
| `match_seconds` | 12 | 每次识别采样时长（秒）。过短会降低置信度 / Sampling duration per recognition (s). Too short lowers confidence |
| `match_interval` | 4 | 识别间隔（秒）。混音期间会自动减半 / Recognition interval (s). Auto-halved during mixes |
| `match_confirmations` | 2 | 连续命中几次才切歌，防止误判 / Consecutive hits required to switch (avoids false positives) |
| 首歌确认阈值 First-track threshold | 0.25 | 第一首歌需达到此置信度才确认 / Confidence needed to confirm the first track |
| 切歌确认阈值 Switch threshold | 0.30 | 后续歌曲需达到此置信度才切换 / Confidence needed to switch to a later track |

## 常见问题 / FAQ

**Q：识别不到歌曲？ / Recognition fails?**
- 检查输入电平表是否有信号 / Check the input level meter for signal
- 确认该歌曲已在曲库中并分析完成 / Confirm the song is in the library and analysed
- 查看 `data/vjvision.log` 中的置信度数值，低于 0.25 通常是音量过低或音频质量差
  Check `data/vjvision.log` for confidence; below 0.25 usually means too quiet or poor quality

**Q：不支持 m4a / aac？ / m4a / aac not supported?**
- 当前使用 libsndfile 解码，原生不支持 m4a/aac。建议转成 flac 或 mp3 后再分析。
  libsndfile is used for decoding; m4a/aac are not natively supported. Convert to flac or mp3 first.

**Q：切歌反应慢？ / Slow track switching?**
- 长混音（cross-fade）期间两首歌重叠，置信度爬升较慢，属正常现象
  During long cross-fades two songs overlap, so confidence climbs slowly — normal
- 可降低 `match_interval` 或切歌阈值提高响应速度，但会增加误判风险
  Lower `match_interval` or the switch threshold for faster response (more false positives)

**Q：关闭控制台后可视化窗口没关？ / Visualizer stays open after closing console?**
- v1.1.1+ 已修复：可视化子进程会检测父进程存活，主进程退出后自动关闭
  Fixed in v1.1.1+: the visualizer child detects parent liveness and exits automatically

**Q：macOS 上设备列表为空 / 提示无音频设备？ / Empty device list on macOS?**
- 首次采集时系统会弹出**麦克风权限**请求：请到「系统设置 → 隐私与安全性 → 麦克风」允许 VJVision（BlackHole 等虚拟设备也走同一权限）；未签名应用首次请用右键→打开启动
  macOS prompts for **Microphone** permission on first capture: allow VJVision in System Settings → Privacy & Security → Microphone (virtual devices like BlackHole use the same permission). Remember to launch the unsigned app via right-click → Open the first time

**Q：下载来的歌曲分析失败（文件损坏）？ / A downloaded song fails analysis (corrupt file)?**
- v1.2.1+ 已内置 ffmpeg 容错解码：soundfile 解码失败时自动用内置 ffmpeg 跳过坏帧继续分析，日志会标注 `Recovered via ffmpeg fallback`；无需自行安装 ffmpeg
  v1.2.1+ bundles ffmpeg tolerant decoding: when soundfile fails, the bundled ffmpeg skips broken frames and analysis continues (logged as `Recovered via ffmpeg fallback`); no manual ffmpeg install needed

## 技术架构 / Architecture

```
main.py
├── VisualizerManager  →  pygame 子进程（副屏可视化）/ pygame subprocess (secondary display)
├── MatcherThread      →  音频采集 + Dejavu 识别 + 元数据 / audio capture + Dejavu recognition + metadata
└── DebugUI            →  CustomTkinter 控制面板（主线程）/ CustomTkinter control panel (main thread)
```

- 进程间通信 IPC：`multiprocessing.Queue`
- 指纹算法 Fingerprinting：Dejavu（声学指纹 + 哈希匹配）
- 音频解码 Decoding：soundfile（libsndfile，支持 24-bit FLAC）
- 国际化 i18n：`vjvision/i18n.py`（中英双语，语言偏好持久化）

## 许可证 / License

本项目采用 **MIT License** —— 开源且**必须署名**。
This project uses the **MIT License** — open source with **attribution required**.

任何人使用、复制、修改、分发本软件时，必须保留原始版权声明和许可证全文。详见 [LICENSE](LICENSE)。
Anyone using, copying, modifying or distributing this software must retain the original copyright notice and full license text. See [LICENSE](LICENSE).

### 第三方依赖许可证 / Third-party Licenses

| 依赖 Dependency | 许可证 License |
|------|--------|
| sounddevice | MIT |
| soundfile | BSD-3-Clause |
| numpy | BSD-3-Clause |
| dejavu | MIT |
| pygame-ce | LGPL-2.1 |
| customtkinter | MIT |
| Pillow | HPND（MIT 兼容） |
| mysql-connector-python | GPLv2 + FOSS Exception |
| mutagen | GPLv2+ |

> **注意 Note**：`mutagen` 为 GPLv2+ 许可证。当前分发形式下，使用者应同时遵守 mutagen 的 GPLv2+ 条款。
> `mutagen` is GPLv2+. Under the current distribution, users must also comply with mutagen's GPLv2+ terms.

## 版本历史 / Changelog

见 [RELEASE_NOTES.md](RELEASE_NOTES.md)。
See [RELEASE_NOTES.md](RELEASE_NOTES.md).
