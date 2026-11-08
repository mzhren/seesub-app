# 识幕 SeeSub

> 把「烧进画面里的硬字幕」重新变成可编辑、可翻译、可导出的字幕文件。

识幕（SeeSub）是一款**本地运行**的桌面字幕工作台。它读取视频画面中的**内嵌硬字幕**（硬字幕压制进画面像素，无法像软字幕那样直接提取），生成带时间轴的 `SRT / ASS / VTT / TXT`，并提供语音转写、整篇翻译、批量编辑与字幕压制等完整工作流。所有重活默认在本机完成，在线模型按需选配。

![version](https://img.shields.io/badge/version-0.4.0-blue)
![platform](https://img.shields.io/badge/platform-macOS%20%7C%20Windows-lightgrey)
![stack](https://img.shields.io/badge/stack-Tauri%202%20%2B%20React%2019-orange)
![build](https://img.shields.io/badge/build-Rust%202024%20edition-red)

---

## 目录

- [它解决什么问题](#它解决什么问题)
- [四种工作模式](#四种工作模式)
- [核心能力](#核心能力)
- [快速开始](#快速开始)
- [从源码构建](#从源码构建)
- [配置说明](#配置说明)
- [导入与导出](#导入与导出)
- [快捷键](#快捷键)
- [技术架构](#技术架构)
- [项目结构](#项目结构)
- [数据与隐私](#数据与隐私)
- [路线图](#路线图)
- [常见问题](#常见问题)

---

## 它解决什么问题

网上大量视频的字幕是**压制进画面的像素**：没有字幕轨、没有外挂文件，播放器里既不能复制也不能翻译。手工处理意味着一边暂停一边打字，一小时视频要花好几个小时。

识幕把这件事变成一条流水线：

```
视频 → 抽帧 → 帧去重 → OCR 识别 → 相似合并 → 语音校对 → 带时间轴的字幕 → 编辑 / 翻译 / 压制 / 导出
```

关键点在于**不是逐帧暴力 OCR**：

- **采样抽帧**：按采样间隔（默认 500 ms）抽帧，而不是每一帧
- **感知哈希去重**：相邻帧哈希距离低于阈值（默认 5）即视为同一画面，跳过重复识别
- **相似合并**：连续帧识别结果相似度高于阈值（默认 0.85）时合并为一条字幕，自动推断起止时间
- **语音校对**（可选）：把 OCR 结果与同期语音比对，剔除「画面里有、但没人说过」的误识别（台标、UI 浮层、压缩噪点）
- **并行 OCR + 流式输出**：`rayon` 多线程识别，每 16 帧推送一次中间结果，字幕边识别边出现，不用等到最后

---

## 四种工作模式

首页选择模式，四个模式共用同一套主体布局（视频区 + 字幕列表 + 音频波形），只有操作行的配置项不同。

| 模式 | 需要框选字幕区域 | 操作行配置项 | 主按钮 |
| --- | --- | --- | --- |
| **硬字幕提取** | ✅ 需要 | OCR 引擎 + 提取语言 | 开始识别 |
| **语音识别** | ❌ 不需要 | 语音引擎 + 语言 | 开始语音识别 |
| **字幕翻译** | ❌ 不需要 | 翻译引擎 + 源/目标语言 | 开始翻译 |
| **字幕编辑** | ❌ 不需要 | 无（编辑模式无需运行任务） | — |

---

## 核心能力

### 硬字幕提取

- **自动定位字幕带**：采样 5 帧，分析画面底部 30% 的逐行边缘密度并投票；置信度低时提示手动框选
- **本地 PaddleOCR**：进程内 ONNX 推理，完全离线。简体中文的 PP-OCRv4 轻量模型（det + rec + 字典）随应用内置，**开箱即用**；繁体中文 / 英文 / 日文 / 韩文模型在设置里按需下载，也可把模型目录指向自有模型
- **在线 OCR 引擎**：可添加多个 OpenAI 兼容的视觉模型接口（如 GLM-OCR、DeepSeek-OCR），在引擎间自由切换
- **语音校对**：用语音识别结果校准 OCR 字幕，相似度低于阈值（默认 0.45）的条目判定为误识别并剔除；可调阈值、可整体关闭
- **可取消**：任何阶段都能中止任务；每一步（准备 → 抽帧 → 去重 → 识别 → 校对）都有进度与状态文案

### 语音识别

不框选任何区域，直接转写整段人声。四种引擎按场景选择：

| 引擎 | 形态 | 模型 |
| --- | --- | --- |
| **Paraformer**（阿里 FunASR） | 本地，sherpa-onnx 推理 | 模型随应用打包，无需下载 |
| **Whisper**（whisper.cpp） | 本地 | tiny / base / small，在线下载或导入本地 `.bin` |
| **Vosk** | 本地，运行时加载 libvosk | cn / en-us / ja / ko 等，按需下载 |
| **在线 API** | OpenAI `/audio/transcriptions` 协议 | 可添加多套接口配置 |

- 语言可指定（中文 / 英文 / 日语 / 韩语 / 粤语等）；明确指定比自动检测更快更准
- 语音识别同时服务于**语音校对**：校对引擎与识别模式共用同一套引擎配置

### 字幕翻译

- **三种引擎**：
  - `LLM API`：OpenAI 兼容聊天接口，可保存多套配置（默认对接 DeepSeek 等）
  - `OPUS-MT`：完全离线的中 ↔ 英翻译（INT8 ONNX，复用内置推理运行时）
  - `千问本地`：llama.cpp 运行 GGUF 模型（qwen3-0.6b / qwen2.5-0.5b），按需下载或导入本地文件
- **整篇批量提交**，一次请求处理几十条而不是逐条调用；条数对不上直接报错，不静默截断
- **时间轴完全不动**，只替换文本
- **双语支持**：译文可保留原文，支持「原文在前 / 原文在后」两种排列并随导出生效

### 字幕编辑

字幕列表上方一排批处理工具，每个工具都有独立的设置弹窗：

| 工具 | 说明 |
| --- | --- |
| **查找** | 关键词实时过滤并显示命中数；上一个 / 下一个会跳到对应时间点 |
| **替换** | 「查找内容 / 替换为」（留空即删除），支持替换单条与全部替换 |
| **划重点** | 把关键词包上标记样式（预设可在设置里维护），导出同样保留，可一键清除 |
| **去水词** | 清理「那个 / 就是 / 然后 / 嗯…」与 `you know / um` 等填充词；支持自定义词表和「词 → 替换」规则表，只剩水词的空字幕直接删除 |
| **翻译** | 选择引擎与语言后整篇翻译 |
| **撤销** | 还原上一次批量修改 |

单行编辑同样顺手：点任意一条即跳帧，文本可直接改写，悬停出现删除按钮，可在播放位置插入新字幕；列表密度可在舒适 / 紧凑间切换。

### 字幕压制

导出 ASS 后可一键把字幕**烧进视频**（ffmpeg `ass` 滤镜 / libass），产出自带硬字幕的成片，压制过程有实时进度。

---

## 快速开始

1. 打开应用，或把视频文件直接拖进窗口（会自动加载同目录的同名字幕文件，如 `a.mp4 → a.srt`）
2. 选择模式（默认是硬字幕提取）
3. **硬字幕提取**：框选画面中的字幕区域 → 点「开始识别」
4. 字幕流式出现在右侧列表，可随时编辑
5. 导出 `SRT / ASS / VTT / TXT`，或压制进视频，或一键复制全部

### 首次使用需要准备的

| 能力 | 准备 |
| --- | --- |
| 硬字幕提取（简体中文） | **开箱即用**，模型已内置 |
| 硬字幕提取（繁中/英/日/韩） | 在设置里按需下载对应识别模型 |
| 硬字幕提取（在线） | 添加 OpenAI 兼容视觉接口的地址、模型与密钥 |
| 语音识别 / 语音校对 | 默认引擎 Paraformer 无需下载；Whisper / Vosk 模型按需下载或导入 |
| 字幕翻译 | 在线引擎填接口地址与密钥；离线引擎按需下载模型 |

---

## 从源码构建

### 环境要求（所有平台）

- **Node.js** + **pnpm**
- **Rust**（2024 edition，`rust-version = 1.85`，建议最新 stable）
- 多个本地推理引擎经 cmake 编译内置 C++ 源码，**需要 cmake**

### macOS

- 仓库已包含 `aarch64-apple-darwin` 的 ffmpeg / ffprobe sidecar，开箱即用
- 自行压制字幕依赖 libass——Homebrew 的 ffmpeg 8.x 不含，需用带 libass 的构建（如 osxexperts.net 静态版）

```bash
pnpm install        # 首次：安装前端依赖
pnpm tauri dev      # 启动调试（vite 热更新 + Rust 自动重编）
pnpm build:mac      # 打包：tauri build → scripts/fix-macos-app.sh → scripts/make-dmg.sh
```

- 调试产物直接运行，前端改动即时热更新，Rust 改动自动重编并重启
- `pnpm build:mac` 产物在 `src-tauri/target/release/bundle/dmg/`

### Windows

> **不要直接运行 `pnpm tauri dev`**——Windows 上必须经由 `scripts/dev-windows.ps1`
> 注入构建环境（CRT 统一、ORT 动态链接等），否则链接失败；且 debug 配置会触发
> 调试堆断言，脚本内部已改用 release 配置编译（前端热更新不受影响）。

首次准备（只需一次）：

1. **工具链**：Rust（`x86_64-pc-windows-msvc`）+ VS 2022（含「使用 C++ 的桌面开发」，MSVC 工具集建议 ≥ 14.44）；CMake 用 VS 自带的即可
2. **LLVM**：`winget install LLVM.LLVM` —— bindgen（whisper / sherpa 绑定）需要 libclang
3. **ffmpeg sidecar**：把本机 ffmpeg / ffprobe 复制为
   `src-tauri/binaries/ffmpeg-x86_64-pc-windows-msvc.exe` 与 `ffprobe-x86_64-pc-windows-msvc.exe`
   （已被 `.gitignore` 忽略；需带 `ass` 滤镜，如 gyan.dev full 构建）
4. **ONNX Runtime 1.24.1**：下载官方 `onnxruntime-win-x64-1.24.1.zip` 解压到
   `%LOCALAPPDATA%\ort-manual\ort-1.24.1\`。ort 走 `ORT_LIB_PATH` + 导入库动态链接
   （pyke 预编译的静态 ORT 与 Tauri Windows 默认静态 CRT 冲突，LNK2038；脚本同时会把
   sherpa 自带的 1.17 版 onnxruntime.dll 换回官方 1.24.1）

日常命令：

```powershell
pnpm install                                                        # 首次：安装前端依赖
powershell -ExecutionPolicy Bypass -File scripts\dev-windows.ps1         # 启动调试（vite 热更新）
powershell -ExecutionPolicy Bypass -File scripts\dev-windows.ps1 build    # 打包（MSI + NSIS）
```

- 安装包产物：`src-tauri/target/release/bundle/msi/识幕_<版本>_x64_zh-CN.msi` 与 `bundle/nsis/识幕_<版本>_x64-setup.exe`
- **便携版（免安装）**：构建完成后运行
  `powershell -ExecutionPolicy Bypass -File scripts\portable.ps1`，
  产物在 `portable\识幕_<版本>_x64_portable\`（目录）与同名 `.zip`，内含主程序、运行时 DLL、
  ffmpeg/ffprobe 与内置模型，解压即用
- **`cargo test` 在 Windows 暂不可用**（测试二进制受混编 CRT 影响），单测请在 macOS 上运行

### 可选 features

```toml
default = ["paddle", "whisper", "local-translate", "paraformer", "qwen"]
```

| feature | 作用 | 关闭后 |
| --- | --- | --- |
| `paddle` | 进程内 ONNX 的 PaddleOCR | 无法编译 `ort` 的环境可关闭，PaddleOCR 会给出「模型未加载」的明确提示 |
| `whisper` | whisper.cpp 语音识别与校对 | 语音相关功能退化，其余不受影响 |
| `paraformer` | 内置 Paraformer 语音识别 | 同上（另有 Vosk / 在线引擎可用） |
| `local-translate` | 离线 OPUS-MT 翻译 | 离线翻译不可用，在线翻译不受影响 |
| `qwen` | 千问本地翻译（llama.cpp） | 千问翻译不可用 |

在无法编译本地推理依赖的机器上：

```bash
cargo build --no-default-features
```

### 测试（macOS）

```bash
cd src-tauri && cargo test --no-default-features   # 全部单测
```

---

## 配置说明

设置保存在应用数据目录的 `settings.json`（前后端同一份数据源），界面里均可改。

### 提取参数（硬字幕提取）

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| 采样间隔 | 500 ms | 每隔多久取一帧；越小越准、越慢 |
| 哈希阈值 | 5 | 相邻帧哈希距离低于该值视为同一画面，跳过识别 |
| 相似度阈值 | 0.85 | 连续帧文本相似度高于该值合并为一条字幕 |
| 尾部间隔 | 50 ms | 每条字幕结尾收掉的毫秒数 |
| 清理策略 | 识别后立即清理 | 保留 / 立即清理 / 退出时清理 中间帧目录 |

### 语音校对

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| 启用语音校对 | 开启 | 关掉则退化为纯 OCR 行为 |
| 语音引擎 / 模型 | Paraformer | paraformer / whisper（tiny / base / small）/ vosk / 在线 API |
| 语音语言 | 中文 | 明确语言比自动检测更快更准 |
| 匹配阈值 | 0.45 | 低于该相似度判定为误识别并删除；调低更严格，调高更保守 |

### OCR / ASR / 翻译接口

- OCR 与语音识别的在线引擎均以「API 配置列表」维护，可添加多套并随时切换
- 翻译支持在线 LLM、离线 OPUS-MT、本地千问三类引擎，目标语言可自由输入
- 所有密钥只保存在本机 `settings.json`

---

## 导入与导出

- **导入**：支持 `.srt` / `.vtt`（时间码解析兼容 `00:00:01,000`、`.mmm`、VTT 短格式与 cue settings，并去除 `<v Speaker>` 类标签）
- **导出**：`SRT`（默认）/ `ASS` / `VTT` / `TXT`；双语导出保留原文
- **压制**：ASS 字幕一键烧进视频（libass 渲染）
- **复制**：一键把全部字幕复制到剪贴板
- **文件名后缀**：可按业务类型、翻译语言对、应用名自动附加，避免覆盖原始文件

---

## 快捷键

| 按键 | 功能 |
| --- | --- |
| `空格` | 播放 / 暂停（在输入框内不触发） |
| `Esc` | 关闭当前弹窗 |
| `Enter` | 查找弹窗内跳到下一个匹配 |

播放器上另有后退 5 秒 / 前进 5 秒按钮；音频波形轨点任意位置跳转，字幕区间高亮。

---

## 技术架构

**前端**：React 19 + TypeScript + Zustand + Vite + lucide-react，样式为 CSS 变量驱动的主题系统（4 套深色 + 对应 4 套亮色，共 8 种）。

**后端**：Rust / Tauri 2，模块化拆分：

| 模块 | 职责 |
| --- | --- |
| `pipeline` | 编排一次完整运行：抽帧 → 去重 → 并行 OCR → 合并 → 语音校对 |
| `video` | ffmpeg sidecar 抽帧、波形峰值、时长探测、字幕带自动定位、感知哈希去重、字幕压制 |
| `ocr` | PaddleOCR（ONNX）与在线引擎、多语言模型下载、结果清洗 |
| `asr` | Paraformer / Whisper / Vosk / 在线引擎的统一目录、模型下载、分窗转写 |
| `subtitle` | 相似合并、时间轴整理、SRT/ASS/VTT/TXT 导出、SRT/VTT 导入 |
| `translate` | 在线 LLM 批量翻译；`translate_local`（OPUS-MT）；`translate_qwen`（llama.cpp） |
| `settings` | JSON 持久化的用户设置与旧版配置迁移 |

**事件流**（Rust → 前端）：

| 事件 | 内容 |
| --- | --- |
| `extraction-progress` | 抽帧进度 |
| `ocr-progress` | 识别进度与状态 |
| `cue-update` | 流式推送新识别出的字幕 |
| `asr-progress` | 语音识别进度与状态文案 |
| `asr-model-progress` | 模型下载进度 |
| `burn-progress` | 字幕压制进度 |

---

## 项目结构

```
seesub-app/
├── src/                        # 前端（React 19 + TypeScript）
│   ├── App.tsx                 # 事件订阅、拖拽打开、全局快捷键
│   ├── components/
│   │   ├── DropZone.tsx        # 首页 + 四模式选择
│   │   ├── ModeTabs.tsx        # 模式切换标签
│   │   ├── VideoPane.tsx       # 视频区 + 模式专属操作行
│   │   ├── CuePane.tsx         # 字幕列表 + 批处理工具
│   │   ├── PlayerBar.tsx       # 播放控制
│   │   ├── AudioTrack.tsx      # 音频波形轨
│   │   ├── SelectionOverlay / SelectionToolbar.tsx   # 框选字幕区域
│   │   ├── ExportDialog.tsx    # 导出与压制
│   │   ├── BurnPreview.tsx     # 压制预览
│   │   ├── SettingsDrawer.tsx  # 设置抽屉（分区标签页）
│   │   ├── LogPanel.tsx        # 运行日志
│   │   └── TopBar.tsx          # 顶部菜单
│   ├── lib/                    # api 封装、时间格式、主题、字幕文本处理
│   ├── store/useAppStore.ts    # 全局状态与所有动作
│   └── types.ts                # 类型与常量（模式、语言、设置）
├── src-tauri/
│   ├── src/                    # Rust 核心（见「技术架构」）
│   │   ├── pipeline.rs / video/ / ocr/ / asr/ / subtitle/
│   │   ├── translate.rs / translate_local.rs / translate_qwen.rs
│   │   └── settings.rs / commands.rs / state.rs / model.rs
│   ├── binaries/               # ffmpeg / ffprobe sidecar（不入库）
│   └── tauri.conf.json
├── scripts/                    # dev-windows.ps1 / portable.ps1 / fix-macos-app.sh / make-dmg.sh
├── portable/                   # Windows 便携版产物（不入库）
└── dist/                       # 前端构建产物
```

---

## 数据与隐私

- 视频、音频、抽帧中间产物全部保留在本机，默认不做任何上传
- 只有主动选择在线 OCR / 语音 / 翻译引擎并填写接口时，才会向该服务发送画面 / 音频 / 文本
- 设置文件：`<应用数据目录>/settings.json`
- 模型目录：`<应用数据目录>/models/`（paddleocr / whisper / vosk / opus-mt / qwen 等）
- 中间帧目录可设置保留、识别后立即清理或退出时清理

---

## 路线图

- 批量视频队列
- 字幕时间轴批量平移与吸附
- Linux 支持与对应平台 sidecar 打包
- 拼写与标点规范化（中英文混排场景）

---

## 常见问题

**识别结果里混进了播放器 UI / 台标？**
把框选区域收紧到字幕文字本身，并开启语音校对；仍有问题可调低「匹配阈值」。

**为什么中文语音识别出了英文？**
自动语言检测在小模型上容易误判。请在设置里把语音语言明确选为「中文」。

**PaddleOCR 报「模型未加载」？**
模型文件缺失或路径不对。检查设置里的模型目录是否包含对应语言的 det / rec 模型与字典；若在无法编译 `ort` 的环境（已关闭 `paddle` feature），请改用在线 OCR 引擎。

**翻译按钮点了没反应？**
需要先在设置里选好翻译引擎：在线引擎要填接口地址与模型，离线引擎要确认模型已下载。

**批量改错了能恢复吗？**
可以。查找替换、划重点、去水词都会保留一份快照，点「撤销」即可还原上一次批量修改。

**Windows 下能跑 `pnpm tauri dev` 吗？**
不能直接跑，请使用 `scripts/dev-windows.ps1`；它负责 CRT 统一、ORT 动态链接等环境注入，并以 release 配置编译（避开调试堆断言问题）。

---

## 许可证

本项目为私有仓库，未附开源许可证，默认保留所有权利。
