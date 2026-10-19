# FFmpeg 完全指南：从命令行到生产集群

> 一套面向后端工程师的 FFmpeg 系统教程。FFmpeg 是音视频领域的"瑞士军刀"，几乎所有流媒体项目都离不开它——转码、推流、切片、录像、抽帧、水印，全靠它完成。这套教程从架构原理讲到生产集群，让你从"抄命令"进阶到"设计方案"。

---

## 为什么 FFmpeg 需要单独一个系列

流媒体项目里，FFmpeg 出现的频率高得不像一个"工具"：

| 场景 | FFmpeg 的角色 |
|------|--------------|
| 摄像头接入 | RTSP 拉流 → 转封装 → 推给流媒体服务器 |
| 直播转码 | RTMP 收流 → 转多码率 → 输出 HLS/DASH |
| 点播上传 | 用户上传 MP4 → 转码标准化 → 切片存 S3 |
| AI 视频检测 | 抽帧（每秒 1 帧）→ 送模型推理 |
| 监控录像 | 分段录像（segment）→ 按时间命名 → 存储 |
| 多分屏预览 | 多路流合成一路（九宫格）→ 减少客户端压力 |
| 视频剪辑 | 裁剪、拼接、加水印、加字幕 |
| 格式兼容 | 用户上传的奇怪格式 → 统一转 H.264 + AAC |

**问题是**：大部分人用 FFmpeg 靠搜索复制命令，遇到报错就懵。参数为什么这样写、为什么这个顺序、为什么换个环境就不行——不理解原理就永远在踩坑。

这套教程解决的就是这个问题：**讲清楚 FFmpeg 的工作模型，让你能自己写出正确的命令，并知道为什么。**

---

## 这套教程能帮你做什么

读完整个系列，你将：

### 1. 理解 FFmpeg 的工作原理
- 掌握处理管道：Demux → Decode → Filter → Encode → Mux
- 理解流（Stream）、编码器（Codec）、容器（Container）的关系
- 看懂 `-map`、`-c:v`、`-filter_complex` 等参数的作用位置和顺序规则
- 知道什么时候需要重新编码，什么时候可以 `-c copy` 秒级完成

### 2. 熟练完成各类音视频任务
- 转码：H.264/H.265/AV1 参数调优，码率控制策略选择
- 滤镜：缩放、裁剪、水印、字幕、拼接、多路合成
- 流媒体：RTMP/RTSP/SRT 推拉流，HLS/DASH 切片生成
- 分析：ffprobe 查看 GOP、码率分布、关键帧位置

### 3. 具备工程化能力
- 硬件加速：NVENC/QSV/VideoToolbox/VAAPI/Vulkan 选型与配置
- Go 集成：进程管理、进度解析、超时控制、异常恢复
- 生产部署：Docker 化、资源限制、并发控制、任务队列
- 性能调优：定位瓶颈，成本优化，转码集群架构设计

---

## 教程结构（共 28 篇，6 个模块）

### 📗 模块 1：基础入门（第 1-5 篇）

搞懂 FFmpeg 的工作模型，这是后面所有内容的基础。不要跳过。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 1 篇](01-基础入门/01-ffmpeg全景与架构解析.md) ⭐ | FFmpeg 全景与架构解析 | libav* 库分工、处理管道五阶段、三大工具定位 |
| [第 2 篇](01-基础入门/02-安装验证与自定义编译.md) | 安装、验证与自定义编译 | 各平台安装、Docker 镜像、编译选项、能力检查 |
| [第 3 篇](01-基础入门/03-ffprobe流信息深度分析.md) ⭐ | ffprobe 流信息深度分析 | 查看编码/GOP/码率/帧类型，JSON 输出解析 |
| [第 4 篇](01-基础入门/04-命令行语法完全解析.md) ⭐ | 命令行语法完全解析 | 参数顺序规则、-map 流选择、全局 vs 局部选项 |
| [第 5 篇](01-基础入门/05-转封装与流复制.md) | 转封装与流复制（不重编码） | -c copy 原理、MP4/FLV/TS 互转、faststart |

### 📘 模块 2：编解码核心（第 6-11 篇）

转码是 FFmpeg 最常用也最容易踩坑的部分。参数怎么调、质量和体积怎么权衡，这里讲透。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 6 篇](02-编解码核心/06-h264编码完全指南.md) ⭐ | H.264 编码完全指南 | CRF/ABR/CBR 模式、preset/tune/profile/level 完整解析 |
| [第 7 篇](02-编解码核心/07-h265-hevc编码与对比.md) | H.265/HEVC 编码与对比 | x265 参数体系、与 H.264 的码率/耗时对比、专利问题 |
| [第 8 篇](02-编解码核心/08-av1编码与对比.md) ⭐ | AV1 编码与对比 | SVT-AV1/libaom 选型、preset 调优、三代编码器横向评测 |
| [第 9 篇](02-编解码核心/09-音频处理完全指南.md) | 音频处理完全指南 | 转码、重采样、混音、声道映射、音量归一化 |
| [第 10 篇](02-编解码核心/10-硬件加速完全指南.md) ⭐ | 硬件加速完全指南 | NVENC/QSV/VideoToolbox/VAAPI/Vulkan 配置与性能对比 |
| [第 11 篇](02-编解码核心/11-视频质量评估与码率梯队设计.md) | 视频质量评估与码率梯队设计 | PSNR/SSIM/VMAF 测量，Netflix ABR Ladder，Per-Title Encoding |

### 📙 模块 3：滤镜系统（第 12-15 篇）

滤镜是 FFmpeg 最强大也最难懂的部分。filter_complex 语法搞懂了，什么画面处理都能做。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 12 篇](03-滤镜系统/12-滤镜系统架构与基础.md) ⭐ | 滤镜系统架构与基础 | -vf vs -filter_complex、标签、链式、分支合并 |
| [第 13 篇](03-滤镜系统/13-视频滤镜实战.md) | 视频滤镜实战 | 缩放、裁剪、旋转、水印、字幕、淡入淡出、色彩分级 |
| [第 14 篇](03-滤镜系统/14-音频滤镜实战.md) ⭐ | 音频滤镜实战 | 音量归一化、混音、变速、降噪、EQ、动态压缩 |
| [第 15 篇](03-滤镜系统/15-复杂滤镜图.md) | 复杂滤镜图 | 九宫格多分屏、画中画、绿幕抠图、多路合成优化 |

### 📕 模块 4：流媒体应用（第 16-20 篇）

FFmpeg 在流媒体链路中的实战用法，直接对接生产需求。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 16 篇](04-流媒体应用/16-HLS直播推流.md) ⭐ | HLS 直播推流 | m3u8+TS 分片、AES-128 加密、LL-HLS、fMP4 切片 |
| [第 17 篇](04-流媒体应用/17-DASH流媒体.md) ⭐ | DASH 流媒体 | MPD 清单、多码率 ABR、CMAF 与 HLS 共用切片 |
| [第 18 篇](04-流媒体应用/18-RTMP-RTSP实时流.md) | RTMP/RTSP 实时流 | 推拉流实战、重连策略、超时参数、低延迟调优 |
| [第 19 篇](04-流媒体应用/19-直播录制与切片.md) | 直播录制与切片 | segment 滚动录像、按时间命名、7x24 录制策略 |
| [第 20 篇](04-流媒体应用/20-流媒体CDN集成.md) | 流媒体 CDN 集成 | URL 签名鉴权、缓存策略、多 CDN 调度与容灾 |

### 📓 模块 5：高级主题（第 21-23 篇）

深入音视频处理的进阶领域，覆盖 2026 年主流技术。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 21 篇](05-高级主题/21-字幕与水印处理.md) | 字幕与水印处理 | SRT/ASS/VTT 格式、硬字幕 vs 软字幕、drawtext、overlay 水印 |
| [第 22 篇](05-高级主题/22-视频拼接与剪辑.md) | 视频拼接与剪辑 | concat 三种方式、xfade 转场、trim 精确裁剪、变速 |
| [第 23 篇](05-高级主题/23-截图与缩略图.md) | 截图与缩略图 | 关键帧截图、雪碧图（tile）、GIF/WebP 动图生成 |

### 📔 模块 6：工程化与生产部署（第 24-28 篇）

从"能跑通命令"到"能上生产"，这部分是后端工程师最需要的。

| 篇章 | 主题 | 学习重点 |
|------|------|---------|
| [第 24 篇](06-工程化与生产部署/24-ffmpeg-8.0新特性详解.md) | FFmpeg 8.0 新特性详解 | 并行转码管道、Vulkan 编解码、Enhanced RTMP、Whisper 字幕滤镜 |
| [第 25 篇](06-工程化与生产部署/25-go集成ffmpeg完整方案.md) ⭐ | Go 集成 FFmpeg 完整方案 | exec 调用、进度解析、超时控制、异步任务队列、Prometheus 指标 |
| [第 26 篇](06-工程化与生产部署/26-生产环境部署.md) ⭐ | 生产环境部署 | Docker 多阶段构建、CPU/内存限制、健康检查、优雅关闭 |
| [第 27 篇](06-工程化与生产部署/27-性能调优与故障排查.md) ⭐ | 性能调优与故障排查 | 三层诊断框架、VMAF 画质评估、音视频同步问题、pprof 分析 |
| [第 28 篇](06-工程化与生产部署/28-转码集群架构设计.md) | 转码集群架构设计 | Master-Worker 架构、Redis Stream、分片并行转码、Leader 选举 |

⭐ 标记的是重点篇章，时间有限可以优先读。

---

## 推荐学习路径

### 路径 A：快速上手（3-4 小时）
只想尽快能用起来：

```
第 1 篇（架构）→ 第 4 篇（命令行语法）→ 第 3 篇（ffprobe）
→ 第 5 篇（转封装）→ 第 6 篇（H.264 转码）→ 第 18 篇（推拉流）
```

### 路径 B：系统学习（推荐，2-3 周）
按顺序读完 28 篇，每篇跟着做动手实践。

### 路径 C：按需查阅
遇到具体问题直接跳到对应篇章：

| 我要做什么 | 看哪篇 |
|-----------|-------|
| 视频转码质量差/体积大 | 第 6、8（AV1）、11 篇 |
| 转码太慢想加速 | 第 10 篇（硬件加速） |
| 加水印/字幕/裁剪 | 第 13、21 篇 |
| 音频处理（混音/归一化/变速）| 第 14 篇 |
| 做多分屏合成 | 第 15 篇 |
| 摄像头 RTSP/RTMP 接入 | 第 18 篇 |
| 生成 HLS 给前端播 | 第 16 篇 |
| 生成 DASH 自适应流 | 第 17 篇 |
| 做监控录像 | 第 19 篇 |
| 流媒体接 CDN | 第 20 篇 |
| 视频拼接、裁剪、变速 | 第 22 篇 |
| AI 检测要截图/缩略图 | 第 23 篇 |
| Go 项目里调 FFmpeg | 第 25 篇 |
| 转码服务部署上线 | 第 26 篇 |
| 报错看不懂/转码慢 | 第 27 篇 |
| 多机器转码集群 | 第 28 篇 |

---

## 环境准备

### 安装 FFmpeg 8.0

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian（官方源版本可能较旧，建议用静态编译版）
sudo apt update && sudo apt install ffmpeg

# 静态编译版（推荐，包含全部编解码器）
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar xf ffmpeg-release-amd64-static.tar.xz

# Docker（生产环境推荐，环境一致）
docker pull linuxserver/ffmpeg:latest
```

### 验证安装与能力检查

```bash
# 查看版本和编译选项
ffmpeg -version

# 查看支持的编码器（确认有 libx264/libx265/libsvtav1）
ffmpeg -encoders | grep -E "x264|x265|av1"

# 查看硬件加速支持
ffmpeg -hwaccels

# 查看支持的协议（确认有 rtsp/rtmp/srt）
ffmpeg -protocols | grep -E "rtsp|rtmp|srt"
```

### 准备测试素材

```bash
# 生成测试视频（10 秒彩条 + 正弦音）
ffmpeg -f lavfi -i testsrc=size=1920x1080:rate=30 \
       -f lavfi -i sine=frequency=1000 \
       -t 10 -c:v libx264 -c:a aac test.mp4

# 下载公开测试流（Big Buck Bunny）
curl -O https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/1080/Big_Buck_Bunny_1080_10s_5MB.mp4

# 公开 RTSP 测试流
# rtsp://rtspstream.com/movie
```

---

## FFmpeg 处理管道（核心心智模型）

理解这张图，FFmpeg 的所有参数位置就都说得通了：

```
输入文件/流                                                    输出文件/流
    │                                                              ▲
    ▼                                                              │
┌─────────┐   ┌─────────┐   ┌────────┐   ┌─────────┐   ┌─────────┐
│ Demuxer │──▶│ Decoder │──▶│ Filter │──▶│ Encoder │──▶│  Muxer  │
│ 解封装  │   │  解码   │   │  滤镜  │   │  编码   │   │  封装   │
└─────────┘   └─────────┘   └────────┘   └─────────┘   └─────────┘
     │             │             │             │             │
  拆出音视频    压缩数据      处理原始帧     原始帧      音视频流
  各条流       →原始帧      （缩放/水印）  →压缩数据    →目标容器
     │             │             │             │             │
  -f 指定     -c:v 指定    -vf/-filter_    -c:v 指定    -f 指定
  输入格式     解码器       complex        编码器      输出格式

     └──────── -c copy 直接跳过解码/滤镜/编码（秒级完成）────────┘
```

**关键结论**：
- `-c copy` 跳过中间三步，速度极快（IO 限制），但不能改分辨率/码率
- 加了滤镜就必须重新编码（滤镜处理的是原始帧）
- 参数写在 `-i` 前面作用于输入，写在后面作用于输出（第 4 篇详解）

---

## 常用命令速查

先放这里应急，原理在对应篇章。

```bash
# 查看视频信息
ffprobe -v quiet -print_format json -show_format -show_streams input.mp4

# 转封装（不重编码，秒级）
ffmpeg -i input.flv -c copy output.mp4

# 转码为 H.264（CRF 23 是视觉无损的常用值）
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -preset medium -c:a aac -b:a 128k output.mp4

# 缩放到 720p
ffmpeg -i input.mp4 -vf scale=-2:720 -c:v libx264 -crf 23 output.mp4

# RTSP 拉流转推 RTMP
ffmpeg -rtsp_transport tcp -i rtsp://cam/stream -c copy -f flv rtmp://server/live/key

# 生成 HLS
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -c:a aac \
  -f hls -hls_time 4 -hls_playlist_type vod -hls_segment_type fmp4 output.m3u8

# 每秒抽 1 帧（AI 检测用）
ffmpeg -i input.mp4 -vf fps=1 -q:v 2 frame_%04d.jpg

# 分段录像（每 10 分钟一个文件）
ffmpeg -i rtsp://cam/stream -c copy -f segment -segment_time 600 \
  -strftime 1 "rec_%Y%m%d_%H%M%S.mp4"

# 硬件加速转码（NVIDIA）
ffmpeg -hwaccel cuda -i input.mp4 -c:v h264_nvenc -cq 23 output.mp4
```

---

## 与其他系列的关系

| 系列 | 关注点 | 关系 |
|------|-------|------|
| [流媒体技术完整体系](../streaming-fundamentals-tutorial/README.md) | 协议原理、服务器选型、行业应用 | **先看那个**理解协议，再看这个学工具 |
| 本系列（FFmpeg 完全指南） | FFmpeg 工具本身的深度使用 | 流媒体系列的工具层展开 |

流媒体系列的第 17-19 篇是 FFmpeg 的概览，本系列是它的完整展开。两者可以并行阅读：遇到 FFmpeg 相关内容时跳到本系列查详解。

---

## 学完之后

你应该能够：

1. **看懂任何 FFmpeg 命令**：知道每个参数在管道哪个阶段起作用
2. **自己写出正确命令**：不用搜索复制，根据需求组合参数
3. **诊断报错**：看日志定位问题在解封装、解码、滤镜还是编码阶段
4. **做技术选型**：软编还是硬编、CRF 还是 CBR、x264 还是 SVT-AV1
5. **设计生产方案**：转码服务怎么部署、怎么控并发、怎么做集群

---

---

## 参考资料

本系列教程基于以下官方文档和权威资料编写，所有技术细节均可在对应来源中核实。

### 官方文档

| 资料 | 链接 | 说明 |
|------|------|------|
| FFmpeg 官网 | https://ffmpeg.org | 项目主页，下载与发布说明 |
| FFmpeg 文档中心 | https://ffmpeg.org/documentation.html | 命令行工具、编解码器、滤镜完整文档 |
| FFmpeg Wiki | https://trac.ffmpeg.org/wiki | 社区维护的教程、FAQ 和最佳实践 |
| FFmpeg Changelog | https://github.com/FFmpeg/FFmpeg/blob/master/Changelog | 历史版本变更记录（8.0 新特性来源） |
| FFmpeg Doxygen API | https://ffmpeg.org/doxygen/trunk/ | libav* 各库的 C API 文档 |

### 版本信息来源

| 版本 | 发布时间 | 代号 | 来源 |
|------|---------|------|------|
| FFmpeg 6.0 | 2023-02 | Von Neumann | https://ffmpeg.org/olddownload.html |
| FFmpeg 7.0 | 2024-04 | Dijkstra | https://ffmpeg.org/olddownload.html |
| FFmpeg 7.1 | 2024-09 | Péter | https://ffmpeg.org/olddownload.html |
| FFmpeg 8.0 | 2025-08-22 | Huffman | https://ffmpeg.org/download.html |
| FFmpeg 8.1.2 | 2026-06-17 | Huffman | https://ffmpeg.org/download.html（当前最新稳定版） |

### 编码标准

| 标准 | 来源 |
|------|------|
| H.264/AVC | ITU-T H.264, ISO/IEC 14496-10 |
| H.265/HEVC | ITU-T H.265, ISO/IEC 23008-2 |
| AV1 | https://aomedia.org/av1-features/ |
| VVC/H.266 | ITU-T H.266, ISO/IEC 23090-3 |
| HLS | https://datatracker.ietf.org/doc/html/rfc8216 |
| DASH | https://dashif.org/docs/DASH-IF-IOP-v4.3.pdf |
| Enhanced RTMP | https://github.com/veovera/enhanced-rtmp |

### 硬件加速文档

| 平台 | 文档来源 |
|------|---------|
| NVENC（NVIDIA） | https://developer.nvidia.com/video-codec-sdk |
| QSV（Intel） | https://www.intel.com/content/www/us/en/developer/articles/technical/quick-sync-video.html |
| VideoToolbox（Apple） | https://developer.apple.com/documentation/videotoolbox |
| VAAPI（Linux） | https://trac.ffmpeg.org/wiki/Hardware/VAAPI |
| Vulkan | https://trac.ffmpeg.org/wiki/Hardware/Vulkan |

### 质量评估标准

| 指标 | 来源 |
|------|------|
| PSNR/SSIM | ITU-T J.247 主观视频质量评估 |
| VMAF | https://github.com/Netflix/vmaf（Netflix 开源） |
| Netflix ABR Ladder | https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2 |

### 其他工具文档

| 工具 | 来源 |
|------|------|
| Whisper（语音识别）| https://github.com/openai/whisper |
| SVT-AV1 | https://github.com/AOMediaCodec/SVT-AV1 |
| libx264 | https://www.videolan.org/developers/x264.html |
| libx265 | https://x265.readthedocs.io |

---

**最后更新**：2026-07-29
**篇数**：28 篇（6 个模块）
**生成提示词**：[00-生成提示词.md](00-生成提示词.md)