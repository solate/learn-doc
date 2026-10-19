# 第 1 篇：FFmpeg 全景与架构解析

> **读完本篇你将理解**：FFmpeg 是什么、由哪些组件构成、视频处理的完整流程是怎样的、为什么 `-c copy` 能秒级完成转换。

---

## 目录

- [FFmpeg 是什么？](#ffmpeg-是什么)
- [三大工具定位](#三大工具定位)
- [六大核心库分工](#六大核心库分工)
- [处理管道五阶段详解](#处理管道五阶段详解)
- [为什么 `-c copy` 这么快？](#为什么--c-copy-这么快)
- [Stream、Codec、Container 的关系](#streamcodeccontainer-的关系)
- [FFmpeg 生态与历史](#ffmpeg-生态与历史)
- [动手实践](#动手实践)
- [本篇小结](#本篇小结)
- [关键术语速查表](#关键术语速查表)

---

## FFmpeg 是什么？

FFmpeg（Fast Forward MPEG）是一个**开源的音视频处理框架**，几乎覆盖了所有音视频处理需求：

- 转码（格式转换、分辨率调整、码率压缩）
- 流媒体（推流、拉流、直播、点播）
- 编辑（裁剪、拼接、加水印、加字幕）
- 分析（抽帧、截图、质量评估、元数据提取）
- 录制（屏幕录制、摄像头采集、音频录制）

**为什么 FFmpeg 这么重要？**

| 场景 | FFmpeg 的作用 |
|------|-------------|
| 视频网站（YouTube/Bilibili） | 用户上传视频 → FFmpeg 转码为标准格式 → 切片存储 |
| 直播平台（Twitch/抖音） | 主播推流 → FFmpeg 转多码率 → 分发给观众 |
| 监控系统 | 摄像头 RTSP 流 → FFmpeg 录像 → 按时间段存储 |
| AI 视频分析 | FFmpeg 抽帧 → 图片送模型推理 → 返回结果 |
| 短视频编辑 | FFmpeg 加滤镜/水印/字幕 → 生成成品 |

FFmpeg 不是一个软件，而是一整套工具链和库的集合，包括**三大工具**和**六大核心库**。

---

## 三大工具定位

FFmpeg 项目提供了三个命令行工具，各有分工：

### 1. `ffmpeg` — 转码引擎（最常用）

**用途**：转码、转封装、推拉流、录制、滤镜处理

```bash
# 典型用法：格式转换
ffmpeg -i input.mp4 -c:v libx264 -crf 23 output.mp4
```

**工作流程**：输入 → 解封装 → 解码 → 滤镜 → 编码 → 封装 → 输出

**定位**：这是你 90% 时间在用的工具。

---

### 2. `ffprobe` — 分析工具（诊断必备）

**用途**：查看视频信息（编码格式、分辨率、码率、GOP、帧类型）

```bash
# 查看完整信息（JSON 格式）
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4
```

**典型输出**：
```json
{
  "streams": [
    {
      "codec_name": "h264",
      "codec_type": "video",
      "width": 1920,
      "height": 1080,
      "r_frame_rate": "30/1",
      "bit_rate": "5000000"
    }
  ]
}
```

**定位**：遇到问题先 `ffprobe` 查看文件信息，这是诊断的起点（第 3 篇详解）。

---

### 3. `ffplay` — 播放器（测试用）

**用途**：快速播放视频/音频/流，验证处理结果

```bash
# 播放本地文件
ffplay video.mp4

# 播放 RTSP 流
ffplay rtsp://192.168.1.100/stream
```

**定位**：开发测试时快速预览，不是生产环境播放器。

---

## 六大核心库分工

FFmpeg 的底层是一系列 C 库（`libav*`），命令行工具只是这些库的封装。理解库的分工，才能理解 FFmpeg 的工作原理。

```
┌────────────────────────────────────────────────────────┐
│                    ffmpeg / ffprobe / ffplay            │  ← 命令行工具层
└────────────────────────────────────────────────────────┘
                            ↓
┌──────────────┬──────────────┬──────────────┬───────────┐
│ libavformat  │ libavcodec   │ libavfilter  │ libavutil │  ← 核心库层
│ 封装/解封装   │ 编码/解码     │ 滤镜处理      │ 公共工具  │
└──────────────┴──────────────┴──────────────┴───────────┘
            ↓                 ↓                 ↓
┌──────────────┐     ┌──────────────┐
│ libswscale   │     │ libswresample │                     ← 格式转换层
│ 像素格式转换  │     │ 音频重采样     │
└──────────────┘     └──────────────┘
```

### 1. libavformat — 封装/解封装（Mux/Demux）

**职责**：处理各种**容器格式**（Container），理解文件结构

**支持的格式**：
- 视频容器：MP4、MKV、FLV、AVI、MOV、TS、WebM
- 流媒体：HLS（m3u8）、DASH（mpd）、RTMP、RTSP、HTTP-FLV
- 音频容器：MP3、AAC、FLAC、WAV、OGG

**核心概念**：
- **解封装（Demux）**：从容器文件中拆出各条流（Stream）
  - 一个 MP4 文件包含：1 条视频流 + 1 条音频流 + 字幕流
- **封装（Mux）**：将多条流打包进容器
  - 把 H.264 视频流 + AAC 音频流 → 封装成 MP4 文件

**关键点**：容器格式 ≠ 编码格式
- MP4 是容器，里面可以放 H.264、H.265、AV1 等不同编码的视频流
- 同样的 H.264 视频流，可以封装成 MP4、FLV、MKV 等不同容器

---

### 2. libavcodec — 编码/解码（Encode/Decode）

**职责**：处理各种**编解码器**（Codec），压缩和解压音视频数据

**支持的编解码器**：
- 视频编码：H.264（AVC）、H.265（HEVC）、AV1、VP9、VVC、MPEG-2、ProRes
- 音频编码：AAC、Opus、MP3、FLAC、Vorbis、AC3

**核心概念**：
- **解码（Decode）**：压缩数据 → 原始帧
  - H.264 压缩数据 → YUV 像素数组（视频）
  - AAC 压缩数据 → PCM 音频样本（音频）
- **编码（Encode）**：原始帧 → 压缩数据
  - YUV 像素 → H.264 压缩数据
  - PCM 音频 → AAC 压缩数据

**编解码器类型**：
- 软件编解码器：libx264、libx265、libsvtav1（CPU 计算）
- 硬件编解码器：h264_nvenc、hevc_qsv、h264_videotoolbox（GPU/芯片加速）

---

### 3. libavfilter — 滤镜处理（Filter）

**职责**：处理原始帧（像素/音频样本），实现各种效果

**常用滤镜**：
- 视频滤镜：scale（缩放）、crop（裁剪）、overlay（水印）、drawtext（文字）
- 音频滤镜：volume（音量）、atempo（变速）、amix（混音）、loudnorm（响度归一化）

**工作原理**：
- 滤镜只能处理**原始帧**（解码后的数据），不能直接处理压缩数据
- 所以加了滤镜就必须重新解码和编码（不能 `-c copy`）

---

### 4. libavutil — 公共工具库

**职责**：提供各种底层工具函数

**包含内容**：
- 数学运算（MD5、CRC、随机数）
- 数据结构（字典、链表、队列）
- 时间戳处理（PTS/DTS 计算）
- 日志系统（FFmpeg 的日志输出就靠它）

---

### 5. libswscale — 像素格式转换

**职责**：在不同像素格式间转换

**典型转换**：
- YUV420P → RGB24（视频解码后送显示器）
- YUV422P → YUV420P（降低色度采样）
- 分辨率缩放（内部也调用这个库）

**为什么需要转换？**
- 编码器要求输入 YUV420P，但源视频是 YUV422P → 需要转换
- 硬件解码器输出 NV12，但软件编码器要 YUV420P → 需要转换

---

### 6. libswresample — 音频重采样

**职责**：在不同音频格式间转换

**典型转换**：
- 采样率：48000 Hz → 44100 Hz
- 声道数：5.1 声道 → 立体声（Stereo）
- 采样格式：float32 → int16

**为什么需要重采样？**
- 编码器要求 48kHz 双声道，但源音频是 44.1kHz 单声道 → 需要重采样

---

## 处理管道五阶段详解

理解 FFmpeg 的核心工作流程，这是所有参数逻辑的基础。

```
输入文件/流                                                    输出文件/流
    │                                                              ▲
    ▼                                                              │
┌─────────┐   ┌─────────┐   ┌────────┐   ┌─────────┐   ┌─────────┐
│ Demuxer │──▶│ Decoder │──▶│ Filter │──▶│ Encoder │──▶│  Muxer  │
│ 解封装  │   │  解码   │   │  滤镜  │   │  编码   │   │  封装   │
└─────────┘   └─────────┘   └────────┘   └─────────┘   └─────────┘
  (libavformat) (libavcodec) (libavfilter) (libavcodec) (libavformat)
     │             │             │             │             │
  MP4容器      H.264压缩     YUV像素       YUV像素      H.264压缩
  拆成各流      →YUV像素     处理后像素    →H.264压缩    +AAC音频
                                                        →MP4容器
```

### 阶段 1：Demux（解封装）

**职责**：从容器文件中拆出各条流

**输入**：MP4 文件（容器）  
**输出**：视频流（H.264 压缩数据）+ 音频流（AAC 压缩数据）+ 字幕流

**举例**：
```bash
input.mp4 (容器)
  ├─ Stream #0:0 (视频)：H.264，1920x1080，30fps
  ├─ Stream #0:1 (音频)：AAC，48000Hz，立体声
  └─ Stream #0:2 (字幕)：SRT，简体中文
```

**关键点**：此时数据还是压缩的，没有解码。

---

### 阶段 2：Decode（解码）

**职责**：将压缩数据解压为原始帧

**输入**：H.264 压缩数据（几 KB/帧）  
**输出**：YUV 像素数组（几 MB/帧）

**数据膨胀**：
- 1080p 一帧 YUV420P：1920 × 1080 × 1.5 = 3,110,400 字节 ≈ 3MB
- H.264 压缩后：约 50KB（压缩比 60:1）

**音频解码**：
- AAC 压缩数据 → PCM 音频样本（16-bit 立体声 48kHz）

**硬件加速在这里**：
- 软件解码：CPU 计算（libx264）
- 硬件解码：GPU/专用芯片（h264_nvenc/h264_qsv）

---

### 阶段 3：Filter（滤镜）

**职责**：处理原始帧，实现各种效果

**输入**：YUV 原始像素（1920x1080）  
**输出**：YUV 原始像素（可能改变分辨率/内容）

**典型操作**：
- 缩放：1920x1080 → 1280x720
- 裁剪：去掉黑边
- 叠加：加水印、加字幕
- 调色：亮度、对比度、饱和度

**关键点**：
- 滤镜处理的是原始像素，不是压缩数据
- 所以加了滤镜必须重新编码（不能 `-c copy`）

---

### 阶段 4：Encode（编码）

**职责**：将原始帧重新压缩

**输入**：YUV 原始像素（3MB/帧）  
**输出**：H.264 压缩数据（50KB/帧）

**编码器选择**：
- libx264（H.264 软件编码，最常用）
- libx265（H.265 软件编码，体积减半）
- h264_nvenc（NVIDIA GPU 硬件编码，速度快 10 倍）

**质量控制**：
- CRF 模式：`-crf 23`（恒定质量，码率自动）
- CBR 模式：`-b:v 5M`（恒定码率，直播必须用）

---

### 阶段 5：Mux（封装）

**职责**：将多条流打包进容器

**输入**：H.264 视频流 + AAC 音频流  
**输出**：MP4 文件（容器）

**容器选择**：
- MP4：通用性最好，但不支持实时流
- FLV：RTMP 直播推流必须用
- MKV：支持多音轨/多字幕/章节
- TS：HLS 切片（现在更多用 fMP4）

---

## 为什么 `-c copy` 这么快？

理解了上面的五阶段，`-c copy` 的原理就清楚了：

```
┌─────────┐                                           ┌─────────┐
│ Demuxer │──────────────────────────────────────────▶│  Muxer  │
│ 解封装  │     跳过解码/滤镜/编码三个阶段                │  封装   │
└─────────┘                                           └─────────┘
     │                                                      ▲
  MP4容器                                               FLV容器
  拆出H.264流  ────────────(数据包直接复制)──────────────▶  封装成FLV
```

**工作原理**：
- 数据包（压缩数据）直接从解封装传到封装
- 不经过解码、滤镜、编码三个阶段
- 只是容器格式变了，编码数据没变

**速度对比**：
- `-c copy`：IO 限制，几秒到几十秒（取决于文件大小）
- 重新编码：CPU/GPU 限制，可能需要几分钟到几小时

**限制**：
- 不能改分辨率（需要滤镜 scale）
- 不能改码率（需要重新编码）
- 不能加水印/字幕（需要滤镜）
- 容器必须兼容该编码格式（如 RTMP 不支持 H.265）

**典型用法**：
```bash
# FLV → MP4（秒级完成）
ffmpeg -i input.flv -c copy output.mp4

# 提取音频（不重编码）
ffmpeg -i video.mp4 -vn -c:a copy audio.aac

# RTSP 转 RTMP（实时转推）
ffmpeg -i rtsp://cam/stream -c copy -f flv rtmp://server/live/key
```

---

## Stream、Codec、Container 的关系

这是新手最容易混淆的概念，理清楚才能避免"为什么 H.265 放不进 MP4"这类问题。

### 关系图

```
┌─────────────────────────────────────────┐
│           Container（容器）              │ ← MP4 / MKV / FLV / AVI
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Stream #0 (Video)               │   │
│  │ Codec: H.264 / H.265 / AV1      │   │ ← 视频流（编码格式）
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Stream #1 (Audio)               │   │
│  │ Codec: AAC / Opus / MP3         │   │ ← 音频流（编码格式）
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Stream #2 (Subtitle)            │   │
│  │ Codec: SRT / ASS / PGS          │   │ ← 字幕流（编码格式）
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 核心概念

- **Container（容器）**：文件的"包装盒"，定义文件结构和索引
  - MP4、MKV、FLV、AVI、MOV、WebM
  - 容器决定：能放几条流、支持哪些编码、是否支持流式传输

- **Stream（流）**：容器内的独立数据轨道
  - 视频流、音频流、字幕流、数据流
  - 一个文件可以有多条流（如多音轨电影）

- **Codec（编解码器）**：流的压缩格式
  - 视频编码：H.264、H.265、AV1、VP9
  - 音频编码：AAC、Opus、MP3、FLAC

### 容器与编码的兼容性

| 容器 | 常见视频编码 | 常见音频编码 | 备注 |
|------|------------|------------|------|
| MP4 | H.264、H.265、AV1 | AAC、MP3 | 通用性最好 |
| MKV | 几乎全支持 | 几乎全支持 | 万能容器 |
| FLV | H.264、H.263 | AAC、MP3 | RTMP 推流专用 |
| WebM | VP8、VP9、AV1 | Opus、Vorbis | Web 开放标准 |
| TS | H.264、H.265 | AAC、MP3 | 传统 HLS 切片 |

**常见坑**：
- ❌ H.265 放进传统 RTMP/FLV（不支持，需要 Enhanced RTMP）
- ❌ Opus 音频放进 MP4（MP4 标准不支持，虽然某些播放器能播）
- ✅ H.265 放进 MP4（支持，但老设备可能播不了）
- ✅ H.265 放进 MKV（完美支持）

---

## FFmpeg 生态与历史

### FFmpeg 的历史

- **2000**：Fabrice Bellard 创建 FFmpeg 项目
- **2011**：项目分叉，部分开发者创建 Libav 项目
- **2015**：Libav 开发停滞，FFmpeg 重新成为主流
- **2025**：FFmpeg 8.0 发布，引入并行转码管道和 Vulkan 支持

**FFmpeg vs Libav**：
- 两者 API 类似，但不完全兼容
- 目前 FFmpeg 更活跃，主流项目都用 FFmpeg
- 你可能看到代码里有 `libavcodec`/`libavformat` 等库，这些名字是历史遗留

### FFmpeg 在开源生态中的位置

几乎所有音视频项目都依赖 FFmpeg：

| 项目 | 用途 |
|------|------|
| VLC | 播放器，底层用 FFmpeg 解码 |
| OBS Studio | 直播推流，用 FFmpeg 编码和推流 |
| HandBrake | 视频转码工具，封装了 FFmpeg |
| Plex / Jellyfin | 媒体服务器，转码用 FFmpeg |
| Chrome / Firefox | 浏览器视频解码部分用 FFmpeg 代码 |

**FFmpeg 的强大之处**：
- 支持几乎所有格式（500+ 编解码器，300+ 容器格式）
- 跨平台（Linux、Windows、macOS、Android、iOS）
- 性能优化极致（SIMD 指令、硬件加速、多线程）
- 持续更新（每年 2-3 个大版本）

---

## 动手实践

### 实践 1：安装 FFmpeg 并验证

```bash
# macOS
brew install ffmpeg

# Ubuntu
sudo apt update && sudo apt install ffmpeg

# 验证安装
ffmpeg -version

# 确认支持的编解码器
ffmpeg -encoders | grep -E "264|265|av1"

# 确认硬件加速
ffmpeg -hwaccels
```

### 实践 2：生成测试视频

```bash
# 生成 10 秒测试视频（彩条 + 音频）
ffmpeg -f lavfi -i testsrc=duration=10:size=1920x1080:rate=30 \
       -f lavfi -i sine=frequency=1000:duration=10 \
       -c:v libx264 -c:a aac test.mp4
```

### 实践 3：用 ffprobe 分析视频

```bash
# 查看视频信息（JSON 格式）
ffprobe -v quiet -print_format json -show_format -show_streams test.mp4

# 只看视频流信息
ffprobe -v quiet -select_streams v -show_streams test.mp4

# 查看编码器类型
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name -of default=noprint_wrappers=1:nokey=1 test.mp4
```

### 实践 4：对比 `-c copy` 和重新编码的速度

```bash
# 方案 1：流复制（秒级）
time ffmpeg -i test.mp4 -c copy output_copy.mp4

# 方案 2：重新编码（慢很多）
time ffmpeg -i test.mp4 -c:v libx264 -crf 23 -c:a aac output_encode.mp4

# 对比文件大小
ls -lh output_copy.mp4 output_encode.mp4
```

### 实践 5：查看处理管道的日志

```bash
# 运行转码并查看详细日志
ffmpeg -i test.mp4 -c:v libx264 -crf 23 output.mp4

# 观察日志中的关键信息：
# - Input #0：解封装阶段，识别容器和流
# - Stream mapping：确定哪些流要处理
# - Codec: libx264：选择编码器
# - frame= ... fps= ... speed=：实时进度
```

---

## 本篇小结

1. **FFmpeg 三大工具**：`ffmpeg`（转码引擎）、`ffprobe`（分析工具）、`ffplay`（播放器）

2. **六大核心库**：
   - libavformat（封装/解封装）
   - libavcodec（编码/解码）
   - libavfilter（滤镜处理）
   - libavutil（公共工具）
   - libswscale（像素格式转换）
   - libswresample（音频重采样）

3. **处理管道五阶段**：Demux → Decode → Filter → Encode → Mux

4. **`-c copy` 快的原因**：跳过解码/滤镜/编码三个阶段，只改容器格式

5. **容器 ≠ 编码**：MP4 是容器，H.264 是编码格式，容器决定能放哪些编码

---

## 关键术语速查表

| 术语 | 英文 | 解释 |
|------|------|------|
| 容器 | Container | 文件的包装格式，如 MP4、MKV、FLV |
| 流 | Stream | 容器内的独立轨道，如视频流、音频流 |
| 编解码器 | Codec | 压缩/解压算法，如 H.264、AAC |
| 解封装 | Demux | 从容器中拆出各条流 |
| 封装 | Mux | 将多条流打包进容器 |
| 解码 | Decode | 压缩数据 → 原始帧 |
| 编码 | Encode | 原始帧 → 压缩数据 |
| 滤镜 | Filter | 处理原始帧，如缩放、水印 |
| GOP | Group of Pictures | 一组帧，通常从一个 I 帧到下一个 I 帧 |
| 流复制 | Stream Copy | 不重新编码，直接复制数据包（`-c copy`） |

---

## 参考资料

### 官方文档
- [FFmpeg 官方网站](https://ffmpeg.org) — FFmpeg 项目主页
- [FFmpeg 文档中心](https://ffmpeg.org/documentation.html) — 完整的命令行工具、库和 API 文档
- [FFmpeg Wiki](https://trac.ffmpeg.org/wiki) — 社区维护的教程和最佳实践

### 架构与设计
- [FFmpeg 架构概览](https://ffmpeg.org/developer.html) — 开发者文档中的架构说明
- [libavformat 文档](https://ffmpeg.org/doxygen/trunk/group__libavf.html) — 封装/解封装库的 API 文档
- [libavcodec 文档](https://ffmpeg.org/doxygen/trunk/group__libavc.html) — 编解码库的 API 文档
- [libavfilter 文档](https://ffmpeg.org/doxygen/trunk/group__lavfi.html) — 滤镜库的 API 文档

### 历史与生态
- [FFmpeg vs Libav 分叉历史](https://en.wikipedia.org/wiki/FFmpeg#History) — 项目分叉与重新合并的历史
- [容器格式规范](https://en.wikipedia.org/wiki/Comparison_of_video_container_formats) — 主流容器格式对比

### 版本信息
本文基于 FFmpeg 8.0 "Huffman"（2025年8月发布）编写，核心架构概念适用于所有 FFmpeg 版本。

---

**下一步**：[第 2 篇：安装、验证与自定义编译](02-安装验证与自定义编译.md)

---

**最后更新**：2026-07-29  
**字数**：约 6800 字
