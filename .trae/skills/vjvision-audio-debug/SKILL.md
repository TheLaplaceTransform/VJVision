---
name: "vjvision-audio-debug"
description: "VJVision 音频链路（pyaudiowpatch 采集 + dejavu/soundfile 索引识别 + ffmpeg 容错兜底）排查指南。当出现无音频硬件闪退/无声卡崩溃、采集电平为零、设备切换失败、索引歌曲失败（LibsndfileError/decoder lost sync/文件损坏）、识别不工作等问题，或需要修改 audio_capture.py/matcher.py/fingerprint.py 时调用。"
---

# VJVision 音频链路诊断指南

VJVision 的音频链路分两段：**实时采集**（pyaudiowpatch/PortAudio → MatcherThread → 电平/频谱/dejavu 识别）和**曲库索引**（mp worker 进程 → soundfile/libsndfile 解码 → dejavu 指纹 → SQLite）。两段的进程模型、日志位置、失败语义完全不同，排查前先定位是哪一段。

## 1. 进程与日志（先看日志再动手）

- 运行时 3 类 python 进程：主程序（控制台 UI + MatcherThread 在进程内）、可视化子进程（spawn）、**索引 worker（mp.Pool spawn，最多 12 个，仅索引期间存在）**。
- **主程序日志**：`cache/vjvision.log`（RotatingFileHandler）。采集、设备、索引调度的日志都在这里。
- **关键坑：worker 进程里的 logging 到不了主日志**（spawn 进程无主进程的 handler）。所以 worker 的错误/备注通过**返回元组**带回主进程打印：
  - `_mp_fingerprint_worker` 返回 `(path, song_name, hashes, file_hash, err_or_note)`；成功时第 5 项是 ffmpeg 兜底备注（None=正常路径），失败时前三项为 None、第 5 项是错误字符串。
  - 主进程在 `index_files`/`index_library` 的结果循环里打 `Failed to fingerprint <file>: <err>`（ERROR）或 `Recovered via ffmpeg fallback: <file> (<note>)`（WARNING）。
- 杀进程重启：`Get-Process python | Stop-Process -Force; Start-Sleep 2`（改代码后必须全杀，旧进程退出会回写 prefs）。
- 全局崩溃留痕：main.py 装了 `sys.excepthook`/`threading.excepthook`，未捕获异常会进 `cache/vjvision.log`；主线程致命崩溃弹 tkinter 错误框。用户说"闪退没日志"时先查这个文件。

## 2. 实时采集段（audio_capture.py / matcher.py）

架构事实：
- `AudioCapture`（audio_capture.py）：构造时创建**唯一** `pyaudio.PyAudio()` 实例（WASAPI 下 COM+音频会话初始化很贵），回调喂环形缓冲；两种模式：MONITOR（仅电平表，启动即开）和 CAPTURE（频谱+识别，按开始采集）。
- `MatcherThread`（matcher.py）：命令队列 `cmd`（device/start/stop/quit/index...）驱动；`_reconfigure_capture(idx)` 重建 AudioCapture（换设备/启动）；`_open_stream_monitor()` 开监听流；`_start_capture()` 进识别模式。

无音频硬件/设备消失的防护（2026-09 已加固，改动这些文件时必须保持）：
- `pyaudio.PyAudio()` 失败（无声卡/驱动/Windows Audio 服务禁用）抛 **`AudioUnavailableError`**（audio_capture.py 顶部定义），不要让裸 OSError 逃逸。
- `AudioCapture.start()` 开流时 **device=None 不传 `input_device_index`**（让 PortAudio 选系统默认；`int(None)` 会 TypeError）。
- `current_level()` 里 `stream.is_active()` 必须 try/except → 设备运行中拔出/流死亡时返回 `active=False`，不抛。
- matcher 的命令队列排空循环**逐条 try/except**：单条命令异常绝不能杀死 matcher 线程（症状：窗口还在但电平/识别全死，用户感知"软件废了"）。
- `_reconfigure_capture()` 永不抛异常：失败 → `self._capture=None`、`_capture_running=False`、经 `_log(t('cap.no_audio')...)` 给 UI 明确中文提示，返回 False。
- 可视化子进程在 `pygame.init()` 前设 `SDL_AUDIODRIVER=dummy`（visualizer.py）——可视化不碰音频硬件，无声卡时窗口也必须能出。
- 换设备重建 capture 时旧对象要 `close()`（terminate PyAudio 实例），否则 PortAudio 会话泄漏。

模拟"无音频硬件"的测试方法（不用真禁用声卡）：monkeypatch `vjvision.audio_capture.pyaudio` 为假模块——`type("M",(),{"PyAudio": <__init__ 抛 OSError 的类>, "paInt16":8, "paContinue":0})()`，然后验证：构造抛 AudioUnavailableError、`list_input_devices()` 返回 `[]`、`_reconfigure_capture()` 返回 False 且不抛、`_handle_cmd({"type":"device"/"start"...})` 不抛。注意假模块必须带 `PyAudio/paInt16/paContinue` 属性，否则报错信息会变成 AttributeError 而非模拟的 OSError。

## 3. 曲库索引段（fingerprint.py / dejavu）

解码链路事实：
- dejavu 自带 pydub 解码路径**已被旁路**：`FingerprintDB._install_soundfile_fallback()` 把 `dejavu.logic.decoder.read` 替换为 `_patched_read`——soundfile(libsndfile) 原生解码 FLAC/WAV/MP3/OGG/AIFF（pydub 解不了 24-bit FLAC）。
- `_patched_read`：`sf.info` 拿采样率 → `sf.read(dtype="int16")` → 分声道 → `_resample_channels()` 用 `scipy.signal.resample_poly` 重采样到 **44100Hz**（指纹频率 bin 依赖采样率，不重采样则 48k 源永远匹配不上 44.1k 采集）→ 对**原始文件**算 SHA1 作 file_hash。
- Python 3.13+ 删了 stdlib `audioop`，pydub import 即崩；任何脚本/进程在 `import dejavu` 之前必须先调 `FingerprintDB._install_audioop_shim()`（numpy 垫片）。独立测试脚本里顺序：`_install_audioop_shim()` → `_install_soundfile_fallback()` → 再 `import dejavu...`。
- 指纹计算 CPU 密集在 worker，DB 写只在主进程（SQLite 单写）；worker 返回 hashes，主进程 `_insert_fingerprints`。

**ffmpeg 容错兜底**（2026-09 新增，处理下载来的损坏音频）：
- 背景：libsndfile 对同步错误**零容忍**——文件末尾一帧坏（"flac decoder lost sync"/"invalid residual"）就拒绝整首；ffmpeg 会跳过坏帧解出其余 99.9%。
- 机制：`_patched_read` 中 soundfile 失败 → `_decode_via_ffmpeg(file_name, limit, sf_error)`：`_find_ffmpeg()`（先 PATH 再可执行文件同目录 ffmpeg.exe，打包分发时把 ffmpeg.exe 放程序目录即可）→ ffmpeg 转临时 16bit WAV（`-vn` 丢内嵌封面视频流、`-t` 支持 limit、Windows 加 CREATE_NO_WINDOW=0x08000000、临时文件 finally 删除）→ sf.read 读回 → 走同一套重采样管道。file_hash 仍按原文件算，身份稳定。
- 成功兜底：worker 写 `_fallback_notes[abspath]`（模块级 dict，per-process），`_fingerprint_one` 成功时把 note pop 出来随元组返回，主进程打 WARNING `Recovered via ffmpeg fallback`。
- 彻底失败（无 ffmpeg / ffmpeg 也解不了 / 输出为空 / 超时 900s）：抛带**中英双语可操作提示**的 RuntimeError（"文件可能已损坏…安装 ffmpeg 或转码后重试"），经 worker 错误通道进主日志。

## 4. 损坏音频文件诊断手册（用户回报"分析失败"时）

1. 先看 `cache/vjvision.log` 的 `Failed to fingerprint <name>: <err>`：
   - `LibsndfileError ... lost sync / System error` → 文件损坏，soundfile 拒解；
   - `RuntimeError: 音频文件可能已损坏…ffmpeg…` → 损坏且该机无 ffmpeg；
   - 其他（路径不存在、权限、dejavu 未连接）按字面处理。
2. 对可疑文件跑一次性诊断脚本（tmp_*.py，用完即删）：
   - `os.path.getsize` + 读头 16 字节：FLAC 魔数 `66 4c 61 43`("fLaC")；`ID3` 开头说明带 ID3v2 头（按 synchsafe 算 tag 大小跳过再看）。
   - `sf.info(path)`：能读元数据（sr/ch/frames/duration）不代表能解码——**必须 `sf.read` 整文件**才暴露 lost sync。
   - 分块定位坏点：`sf.SoundFile` 每 10s 一块 `read()`，记录首个失败帧 → 换算时间/字节偏移（本案坏点在 99.92% 处，最后 0.15s）。
   - ffmpeg 验全文件：`ffmpeg -v error -i <file> -vn -f null -`（**必须 -vn**，FLAC 内嵌 mjpeg 封面会让不带 -vn 的 null 解码报 "Error selecting an encoder" 干扰判断）；returncode 0 + audio 解出完整时长 = ffmpeg 可救。
3. 手工修复单文件：`ffmpeg -i <坏.flac> -vn -c:a flac -compression_level 8 <修好.flac>`（保留原规格，截掉坏尾段），修完用 `sf.read(dtype="int16")` 验证。
4. 注意：带 ffmpeg 兜底的版本对可救文件会自动成功（日志有 Recovered 警告）；无需手工修。只有 ffmpeg 不可用或坏得太彻底才需要人工介入。

## 5. 识别/采集运行期问题速查

- 电平表为零：先确认 UI 设备下拉选的是输入/回采设备（WASAPI loopback 名带"（回采）"后缀）；matcher 日志有 `Opening input stream device=N sr=...`；`current_level()["active"]` False = 流没开或设备死了（监控流失败日志 `cap.monitor_unavailable`）。
- 识别不到歌：确认曲库已索引（日志 `dejavu connected. N songs indexed`）；采集模式已开（UI 状态"采集：运行中"）；源文件采样率与识别无关（两端都重采样到 44.1k）。
- 设备切换无效/切换后死：`device` 命令走 `_reconfigure_capture`；流死亡时 `current_level()` 抛异常已被 device 分支 try/except 兜住（按 not-live 处理→重建）。
- 索引卡住/worker 死锁：worker 里调 dejavu 必须 `print_output=False`（stdout 管道无人抽会撑满死锁）；黑框闪现检查 CREATE_NO_WINDOW monkeypatch（main.py 和 fingerprint.py 各有一份）。

## 6. 边界与约定

- 改 audio_capture/matcher/fingerprint 后：`python -m py_compile` 全部改动文件 → 全杀进程 → `python main.py` 后台启动 → 查 `cache/vjvision.log` 尾部干净（正常启动序列：Starting visualizer → Matcher thread started → Opening input stream）→ 交用户手工测。
- 临时诊断/测试脚本一律 `tmp_*.py`，用完删除，勿留项目根。
- i18n 新增文案 zh/en 两个 dict 都要加（键数一致）；**注意 IDE 旧标签页自动保存会回退 Edit 写入**——改动后用 Grep 复核关键标记在磁盘上，若被回退用磁盘补丁脚本（`io.open(..., newline="")` 保 CRLF）重写，并提醒用户关闭旧标签页。
- 不主动打包 EXE、不主动 git commit；工作分支 `feat/gpu-acceleration`。
- 渲染/可视化窗口问题见同目录 `../vjvision-viz-debug/SKILL.md`。
