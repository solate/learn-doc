# 第 18 篇：RTMP 直播推流

> **模块**：04-流媒体应用 | **难度**：中级 | **预计阅读**：50 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-08-19

---

## 目录

1. [RTMP 协议概述](#1-rtmp-协议概述)
2. [RTMP 协议原理](#2-rtmp-协议原理)
3. [RTMP 推流实战](#3-rtmp-推流实战)
4. [RTMP 拉流与转封装](#4-rtmp-拉流与转封装)
5. [流媒体服务器搭建](#5-流媒体服务器搭建)
6. [断流重连与健康监测](#6-断流重连与健康监测)
7. [动手实践](#7-动手实践)
8. [本章总结](#8-本章总结)
9. [术语表](#9-术语表)

---

## 学习目标

完成本章后，你将能够：

- 理解 RTMP 协议的设计原理与应用场景
- 用 FFmpeg 完成 RTMP 推流、拉流、转封装的全链路操作
- 搭建 SRS/Nginx-RTMP 流媒体服务器，打通推流到播放
- 实现带断流重连、健康监测的推流服务

---

## 1. RTMP 协议概述

### 1.1 RTMP 的历史定位

RTMP（Real-Time Messaging Protocol）是 Adobe 在 2002 年为 Flash Player 设计的流媒体传输协议。在移动互联网和 HTML5 普及之前，RTMP + Flash 是互联网直播的唯一解决方案。

**时间线**：
- **2002**：Adobe 发布 RTMP，配合 Flash Player 实现浏览器直播
- **2005-2015**：RTMP 成为直播行业事实标准（斗鱼、YY、虎牙等平台推流协议）
- **2017**：Adobe 宣布 2020 年停止更新 Flash Player
- **2020**：Flash Player 停止支持，RTMP 播放端死亡
- **2020 至今**：RTMP 作为推流协议继续存活（OBS/硬件编码器生态成熟）

### 1.2 RTMP 过时了还在用？

Flash 死后，RTMP 的角色发生了分裂：

**播放端**（已死 ☠️）：
- 浏览器无法直接播放 RTMP（Flash Player 停止支持）
- 观众看直播现在用 HLS 或 HTTP-FLV

**推流端**（仍活跃 ✅）：
- OBS、FFmpeg、硬件编码器生态成熟，仍用 RTMP 推流
- 直播平台服务器端兼容性好（收流后转码分发）
- 替代方案（SRT/WebRTC 推流）普及较慢

**典型架构**：
```
主播 OBS ──RTMP推流──> 直播服务器 ──转码──> CDN ──HLS/FLV──> 观众浏览器
         ↑ 还在用                                    ↓ 不用 RTMP 了
```

**你是否需要关注 RTMP？**
- ✅ **做直播推流**：必须（FFmpeg 推流、OBS 插件开发）
- ✅ **搭建直播平台**：必须（收流端要支持，因为大量设备在用）
- ❌ **做播放器**：不需要（直接用 HLS/FLV）
- ❌ **纯观众端应用**：不需要（平台已转码好）

### 1.3 RTMP 的技术特点

| 维度 | 特性 |
|------|------|
| 传输层 | TCP（可靠传输）|
| 默认端口 | 1935 |
| 端到端延迟 | 1-3 秒 |
| CDN 支持 | 好（成熟的 RTMP 边缘服务器）|
| 防火墙穿透 | 好（TCP 端口）|
| 丢包抗性 | 差（TCP 重传导致卡顿）|
| 主要场景 | 推流到服务器（OBS/编码器 → 直播平台）|
| FFmpeg 支持 | ✅ librtmp 内置 |

**RTMP vs 其他协议**：
- vs HLS/DASH：延迟低（1-3s vs 10-30s），但 CDN 分发能力弱
- vs RTSP：更适合推流（RTSP 主要用于摄像头取流）
- vs SRT：网络容错差（SRT 能抗 20% 丢包，RTMP 在 5% 丢包时就卡顿）
- vs WebRTC：延迟高（1-3s vs 0.1-0.5s），但服务器端实现简单

---

## 2. RTMP 协议原理

### 2.1 RTMP 握手与连接建立

RTMP 基于 TCP，连接建立需要握手 + 命令交互：

```
┌─────────────────────────────────────────────────────┐
│               RTMP 连接建立过程                       │
│                                                       │
│  客户端          服务端                               │
│    │                │                                │
│    │──── C0 ────────▶  版本号（1字节，通常=3）        │
│    │◀─── S0+S1 ─────   版本+随机32字节               │
│    │──── C1+C2 ────▶   随机数+S1的echo               │
│    │◀─── S2 ─────────  C1的echo                     │
│    │  握手完成（≈3 RTT）                              │
│    │──── connect ──▶   发送连接命令（app名）          │
│    │◀─── _result ───   服务端确认                     │
│    │──── createStream▶  创建流通道                   │
│    │◀─── streamID ───   分配 stream ID               │
│    │──── publish ──▶   声明推流 or play              │
│    │◀─── onStatus ───   流状态回调                    │
│    │                │                                │
│    │══════════ 音视频数据流 ═══════════════           │
└─────────────────────────────────────────────────────┘
```

### 2.2 RTMP URL 结构

```
rtmp://host:port/app/stream_key
  │       │      │    │
  │       │      │    └── 流密钥（推流鉴权）
  │       │      └─────── 应用名（app name）
  │       └────────────── 端口（默认 1935）
  └────────────────────── 协议

示例：
rtmp://live.example.com:1935/live/abc123token
rtmps://live.example.com/live/abc123token   # TLS 加密版本
rtmpt://live.example.com/live/abc123token   # HTTP 隧道版本
```

**RTMP URL 变种**：
- `rtmp://`：标准 TCP 传输
- `rtmps://`：TLS 加密（端口 443），防 CDN 劫持
- `rtmpt://`：HTTP 隧道（穿防火墙，但增加延迟）
- `rtmpe://`：Adobe 私有加密（已过时）

### 2.3 RTMP Chunk 分块传输

RTMP 不直接传输完整视频帧，而是**把每一帧切成多个小块（Chunk）**交错传输，避免大帧阻塞其他流。

**为什么需要切块？**

假设同时传输视频和音频：
- 视频 I 帧：100KB（关键帧很大）
- 音频帧：1KB（音频帧很小）

如果不切块，传输顺序是串行的：
```
发送完整 I 帧（100KB）→ 发送音频帧（1KB）→ 发送 P 帧（10KB）→ ...
```
**问题**：大 I 帧传输期间，音频必须等待，导致音视频不同步、播放卡顿。

**Chunk 机制的解决方案**：

RTMP 把每一帧都切成固定大小的小块（默认 128 字节），多路复用交错传输：

```
视频 I 帧（100KB）切成多个 128B 的 Chunk：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│V_C1 │V_C2 │V_C3 │ ... │V_C799│V_C800│  (约 800 个 chunk)
└─────┴─────┴─────┴─────┴─────┴─────┘

音频帧（1KB）切成多个 128B 的 Chunk：
┌─────┬─────┬─────┐
│A_C1 │A_C2 │ ... │  (约 8 个 chunk)
└─────┴─────┴─────┘

实际传输顺序（交错多路复用）：
V_C1 → A_C1 → V_C2 → A_C2 → V_C3 → ...
```

**核心优势**：
1. **大帧不阻塞**：100KB 的 I 帧被切成 800 个小块，每发完一个就可以插入音频 chunk
2. **多路复用**：视频、音频、控制消息在同一 TCP 连接上交错传输
3. **降低延迟**：接收端不用等完整帧到达才能处理

**接收端处理**：
1. 收到带 `chunk stream ID` 的 chunk
2. 按 ID 归类、按序号重组成完整的 Message
3. 再把 Message 解析成完整的视频/音频帧

**对开发者透明**：使用 FFmpeg 或其他 RTMP 库时，不需要手动处理 chunk —— 协议层会自动切分和重组。

**Chunk Size 影响**：
- 小 Chunk（128B）→ 音视频交错快，延迟低，但包头开销大
- 大 Chunk（4KB）→ 效率高，但大帧可能阻塞音频
- FFmpeg 默认：4096 字节

---
## 3. RTMP 推流实战

> **⚠️ 前置准备**：执行下面的推流命令前，需要先启动一个**流媒体服务器**。
> 
> FFmpeg 只是客户端工具（类比 curl），不能监听端口接收推流。你需要一个服务器（类比 nginx）来接收流。
> 
> **快速启动**（1 分钟）：跳到 [第 5 节](#5-流媒体服务器搭建) 查看详细说明，或直接运行：
> ```bash
> docker run -d -p 1935:1935 -p 8080:8080 ossrs/srs:5
> ```
> 
> 启动后即可执行下面的推流命令。

### 3.1 本地文件推流

```bash
# 最简单的推流（本地视频 → RTMP 服务器）
ffmpeg -re -i input.mp4 \
  -c copy \
  -f flv \
  rtmp://localhost:1935/live/stream

# -re：按实时速率读取（模拟直播）
# -c copy：流复制，不重新编码
# -f flv：RTMP 使用 FLV 容器
```

### 3.2 重新编码推流

```bash
# 转码为 RTMP 适配格式（H.264 + AAC）
# 推流到本地测试服务器
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -maxrate 2M -bufsize 4M \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -c:a aac -b:a 128k -ar 44100 \
  -f flv \
  rtmp://localhost:1935/live/mystream

# 推流到远程直播平台（实际使用时替换为平台提供的地址）
# ffmpeg -re -i input.mp4 \
#   -c:v libx264 -preset veryfast -tune zerolatency \
#   -b:v 2M -maxrate 2M -bufsize 4M \
#   -g 60 -keyint_min 60 -sc_threshold 0 \
#   -c:a aac -b:a 128k -ar 44100 \
#   -f flv \
#   rtmp://live.bilibili.com/live/your-stream-key
```

**关键参数**：
- `-tune zerolatency`：最小化编码延迟
- `-g 60 -keyint_min 60`：固定 GOP 为 2 秒（30fps × 2）
- `-sc_threshold 0`：禁用场景切换检测（避免意外插入关键帧）
- `-maxrate` + `-bufsize`：CBR 恒定码率，防止突发流量
- `-ar 44100`：音频采样率（RTMP 标准）

**播放验证**（推流开始后，打开新终端）：

```bash
# 方式1：RTMP 直接播放（延迟最低 1-3s）
ffplay rtmp://localhost:1935/live/mystream

# 方式2：HTTP-FLV 播放（延迟 1-3s，浏览器兼容性好）
ffplay http://localhost:8080/live/mystream.flv

# 方式3：HLS 播放（延迟 6-20s，CDN 友好）
# 等待 6-12 秒生成 m3u8 文件后播放
ffplay http://localhost:8080/live/mystream.m3u8

# 方式4：浏览器播放 HLS（需支持 HLS 的播放器插件）
# macOS Safari 原生支持：
open http://localhost:8080/live/mystream.m3u8
```

**推流状态检查**：

```bash
# 查看 SRS 服务器收到的流列表
curl http://localhost:1985/api/v1/streams/ | python3 -m json.tool

# 查看服务器统计信息
curl http://localhost:1985/api/v1/summaries/ | python3 -m json.tool
```

### 3.3 摄像头实时推流

```bash
# macOS：摄像头 + 麦克风推流
ffmpeg -f avfoundation -framerate 30 -video_size 1280x720 -i "0:0" \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 1.5M \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/webcam

# Linux（V4L2）：
ffmpeg -f v4l2 -framerate 30 -video_size 1280x720 -i /dev/video0 \
  -f alsa -i hw:0 \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 1.5M \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/webcam

# Windows（DirectShow）：
ffmpeg -f dshow -i video="Integrated Camera":audio="Microphone" \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 1.5M \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/webcam
```

### 3.4 屏幕录制推流

**基础命令**（适合快速测试）：

```bash
# macOS（屏幕共享推流）
ffmpeg -f avfoundation -framerate 30 -i "1:0" \
  -c:v libx264 -preset medium \
  -crf 20 \
  -pix_fmt yuv420p \
  -g 60 -keyint_min 60 \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/screen

# Linux（X11）
ffmpeg -f x11grab -framerate 30 -video_size 1920x1080 -i :0.0+0,0 \
  -c:v libx264 -preset medium \
  -crf 20 \
  -pix_fmt yuv420p \
  -g 60 -keyint_min 60 \
  -f flv \
  rtmp://localhost:1935/live/screen
```

**播放验证**（推流开始后，打开新终端）：

```bash
# 方式1：RTMP 播放（延迟最低）
ffplay rtmp://localhost:1935/live/screen

# 方式2：HTTP-FLV 播放
ffplay http://localhost:8080/live/screen.flv

# 方式3：HLS 播放（等待 6-12 秒）
ffplay http://localhost:8080/live/screen.m3u8
```

**画质优化**（如果发现画面模糊）：

屏幕录制的特点是**细节多**（文字、代码、UI），需要更高的编码质量。常见问题和解决方案：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 画面模糊 | CRF 值太高或码率太低 | 降低 CRF 到 18（更清晰）或 15（极致） |
| 文字不清晰 | 编码 preset 太快 | 改用 `-preset medium` 或 `slow` |
| 推流卡顿 | CPU 编码太慢 | 改用硬件编码（VideoToolbox/NVENC） |
| 文件太大 | 质量过高 | 适当提高 CRF 到 23-25 |

**高清晰度配置**（文字/代码清晰可读）：

```bash
# macOS - 高质量屏幕录制（CRF 18，文字清晰）
ffmpeg -f avfoundation -framerate 30 -i "1:0" \
  -c:v libx264 -preset medium \
  -crf 18 \
  -pix_fmt yuv420p \
  -g 60 -keyint_min 60 \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/screen

# Linux - 高质量屏幕录制
ffmpeg -f x11grab -framerate 30 -video_size 1920x1080 -i :0.0+0,0 \
  -c:v libx264 -preset medium \
  -crf 18 \
  -pix_fmt yuv420p \
  -g 60 -keyint_min 60 \
  -f flv \
  rtmp://localhost:1935/live/screen
```

**硬件加速配置**（降低 CPU 占用）：

```bash
# macOS VideoToolbox 硬件编码
ffmpeg -f avfoundation -framerate 30 -i "1:0" \
  -c:v h264_videotoolbox \
  -b:v 10M -maxrate 10M \
  -pix_fmt yuv420p \
  -c:a aac -b:a 128k \
  -f flv \
  rtmp://localhost:1935/live/screen

# Linux NVENC（NVIDIA 显卡）
ffmpeg -f x11grab -framerate 30 -video_size 1920x1080 -i :0.0+0,0 \
  -c:v h264_nvenc \
  -preset p4 -tune hq \
  -b:v 10M -maxrate 10M \
  -pix_fmt yuv420p \
  -f flv \
  rtmp://localhost:1935/live/screen
```

**CRF 质量参考**：

| CRF 值 | 质量 | 适用场景 | 码率参考（1080p） |
|--------|------|---------|------------------|
| 15-18 | 极高（视觉无损） | 代码演示、教学视频 | 8-15 Mbps |
| 20-23 | 高（推荐） | 日常屏幕分享、会议 | 4-8 Mbps |
| 25-28 | 中等 | 带宽受限场景 | 2-4 Mbps |
| 30+ | 低 | 不推荐 | < 2 Mbps |

### 3.5 推流参数调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-preset` | `veryfast` / `ultrafast` | 直播优先速度，不追求压缩率 |
| `-tune` | `zerolatency` | 关闭 B 帧和前瞻，降低编码延迟 |
| `-g` | fps × 2 | GOP 2 秒，兼顾首屏和码率 |
| `-b:v` | 按分辨率定 | 720p→2M，1080p→4M |
| `-maxrate` | = `-b:v` | 恒定码率，防突发 |
| `-bufsize` | 2 × `-b:v` | VBV 缓冲区 |
| `-profile:v` | `main` / `high` | 兼容性 vs 压缩率 |
| `-pix_fmt` | `yuv420p` | 播放器通用格式 |

```bash
# 完整生产级推流命令
ffmpeg -re -i input.mp4 \
  -c:v libx264 \
  -preset veryfast -tune zerolatency \
  -profile:v main -level 4.0 -pix_fmt yuv420p \
  -b:v 2500k -maxrate 2500k -bufsize 5000k \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -c:a aac -b:a 128k -ar 44100 -ac 2 \
  -f flv \
  -rtmp_live live \
  rtmp://localhost:1935/live/mystream
  # 生产环境替换为实际平台地址：
  # rtmp://live.bilibili.com/live/your-stream-key
```

---

## 4. RTMP 拉流与转封装

### 4.1 RTMP 拉流保存

**本地演示完整流程**：

```bash
# 步骤 1：启动 RTMP 服务器（如果还未启动）
docker run -d -p 1935:1935 -p 8080:8080 --name srs ossrs/srs:5

# 步骤 2：推流到服务器（终端窗口 1 - 保持运行）
# 用测试视频循环推流，模拟主播直播
ffmpeg -re -stream_loop -1 -i test.mp4 \
  -c copy \
  -f flv \
  rtmp://localhost:1935/live/stream

# 步骤 3：拉流保存（终端窗口 2）
# 方案 A：直接存 MP4（需要 Ctrl+C 正常结束才能写入 moov box）
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c copy \
  -f mp4 \
  -movflags +faststart \
  output.mp4

# 方案 B：先存 FLV（推荐，更安全，随时可停）
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c copy \
  output.flv

# 停止后再转 MP4
ffmpeg -i output.flv -c copy -movflags +faststart output.mp4

# 步骤 4：验证文件
ffplay output.mp4  # 或用 VLC 播放
```

**关键参数说明**：

| 参数 | 作用 |
|------|------|
| `-re` | 按实时速率读取（推流必须，否则瞬间推完整个文件）|
| `-stream_loop -1` | 无限循环播放文件（模拟连续直播）|
| `-movflags +faststart` | 把 moov box 移到文件开头（方便 HTTP 渐进式播放）|

**为什么推荐先存 FLV 再转 MP4？**

MP4 格式的 moov box（元数据）在文件末尾写入，直播录制中按 Ctrl+C 中断会导致：
- ❌ 文件损坏无法播放
- ❌ 已录制的几小时数据全部丢失

而 FLV 格式：
- ✅ 每一帧独立写入，随时可停
- ✅ 即使异常中断，已写入的部分仍可播放
- ✅ 生产环境优先选择（录制直播回放、故障恢复）

**拉取远程直播流并保存**：

```bash
# 拉取远程平台的直播流（实际使用时替换为真实地址）
# ffmpeg -i rtmp://pull.example.com/live/stream -c copy output.flv
```

### 4.2 RTMP 转 HLS（转协议）

```bash
# RTMP 拉流 → HLS 输出（流复制，零编码开销）
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c copy \
  -f hls \
  -hls_time 6 \
  -hls_list_size 5 \
  -hls_flags delete_segments+append_list \
  -hls_segment_filename "/var/hls/seg_%05d.ts" \
  /var/hls/index.m3u8
```

### 4.3 RTMP 转 DASH

```bash
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c:v libx264 -preset veryfast -b:v 2M \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 4 \
  -window_size 5 \
  -remove_at_exit 1 \
  /var/dash/manifest.mpd
```

### 4.4 一路输入多路输出（tee）

```bash
# 一次拉流，同时输出 HLS + DASH + 录制文件
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c:v libx264 -preset veryfast -b:v 2M -c:a aac -b:a 128k \
  -f tee -map 0:v -map 0:a \
  "[f=hls:hls_time=6:hls_list_size=5:hls_flags=delete_segments]/var/hls/index.m3u8|\
[f=dash:seg_duration=4:window_size=5]/var/dash/manifest.mpd|\
[f=flv]/var/rec/record.flv"
```

**tee 复用器优势**：只解码编码一次，多路输出共享编码结果，CPU 开销降低 60% 以上。

### 4.5 RTMP 转推（中继）

```bash
# 从本地 SRS 拉流，转推到多个平台（多平台同步直播）
ffmpeg -i rtmp://localhost:1935/live/stream \
  -c copy -f flv rtmp://live.douyu.com/live/your-key-a \
  -c copy -f flv rtmp://live.bilibili.com/live/your-key-b \
  -c copy -f flv rtmp://live.huya.com/live/your-key-c

# 从远程源拉流并转推
# ffmpeg -i rtmp://source.example.com/live/stream \
#   -c copy -f flv rtmp://platform-a.com/live/key-a \
#   -c copy -f flv rtmp://platform-b.com/live/key-b
```

---
## 5. 流媒体服务器搭建

### 5.1 为什么需要流媒体服务器

**FFmpeg 只是客户端工具**，它可以：
- ✅ 推流（把视频推到某个地址）
- ✅ 拉流（从某个地址拉视频）

但它**不能**：
- ❌ 接收推流（监听 1935 端口等待客户端连接）
- ❌ 分发流（一路推流，多人观看）
- ❌ 协议转换（RTMP 推流 → HLS/HTTP-FLV 播放）

**流媒体服务器的作用**：

```
┌─────────────────────────────────────────────────────┐
│          没有流媒体服务器的困境                       │
│                                                       │
│  主播 OBS ──RTMP推流──▶ ？谁监听 1935 端口接收？      │
│                                                       │
│  观众浏览器 ──想看直播──▶ ？从哪拉流？                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│           有流媒体服务器的完整链路                    │
│                                                       │
│  主播 OBS ──RTMP推流──▶ SRS/Nginx-RTMP (监听1935)   │
│                           │                          │
│                           ├─ 转封装 HLS              │
│                           ├─ 转封装 HTTP-FLV         │
│                           ├─ 录制存储                │
│                           └─ 分发给多个观众          │
│                                                       │
│  观众A ◀──HLS──────────────┘                         │
│  观众B ◀──HTTP-FLV─────────┘                         │
│  观众C ◀──WebRTC───────────┘                         │
└─────────────────────────────────────────────────────┘
```

**类比理解**：
- FFmpeg = curl（发请求、下载文件）
- 流媒体服务器 = nginx（接收请求、处理响应、分发内容）

### 5.2 常见流媒体服务器对比

| 服务器 | 开发语言 | 协议支持 | 特点 | 适用场景 |
|--------|---------|---------|------|---------|
| **SRS** | C++ | RTMP/HLS/HTTP-FLV/WebRTC/SRT | 功能全、文档好、社区活跃 | 国内最流行，推荐首选 |
| **Nginx-RTMP** | C (Nginx 模块) | RTMP/HLS/HTTP-FLV | 轻量稳定、成熟可靠 | 简单场景、老项目兼容 |
| **ZLMediaKit** | C++ | 全协议 | 性能极致、商用级 | 大规模、高并发 |
| **Monibuca** | Go | 全协议 | 纯 Go 生态、插件化 | Go 项目集成、二次开发 |
| **LiveGo** | Go | RTMP/HLS/HTTP-FLV | 轻量 Go 实现 | 学习参考、小型项目 |

**SRS vs Nginx-RTMP 选型**：

| 维度 | SRS | Nginx-RTMP |
|------|-----|------------|
| **上手难度** | ⭐⭐⭐⭐⭐ Docker 一行启动 | ⭐⭐⭐ 需编译 Nginx 或装特殊版本 |
| **协议支持** | RTMP/HLS/HTTP-FLV/WebRTC/SRT | RTMP/HLS/HTTP-FLV |
| **WebRTC** | ✅ 原生支持 | ❌ 需配合其他服务 |
| **HTTP API** | ✅ 内置（查流、踢人、统计）| ❌ 需自己开发 |
| **集群部署** | ✅ Origin-Edge 方案 | ⚠️ 需第三方方案 |
| **文档** | ✅ 中文文档完善（ossrs.io）| ⚠️ 主要英文，社区分散 |
| **维护状态** | ✅ 活跃开发（2024 v6.0）| ⚠️ 上游停更（2017），社区分支维护 |
| **性能** | 单机可承载 1000+ 并发流 | 单机可承载 800+ 并发流 |
| **适用场景** | 新项目、需 WebRTC、需完整功能 | 老项目、只用 RTMP/HLS、追求稳定 |

**推荐选择**：
- 🎯 **新项目首选 SRS**：功能全、社区好、Docker 部署简单
- 🔧 **已有 Nginx 环境用 Nginx-RTMP**：不引入新组件，统一运维
- 🚀 **高性能场景用 ZLMediaKit**：商用级稳定性，单机性能天花板
- 💻 **Go 项目集成用 Monibuca/LiveGo**：原生 Go 调用，不依赖外部进程

---

### 5.3 SRS（Simple Realtime Server）快速上手

**SRS** 是国内开源流媒体服务器，由杨成立（winlinvip）创建，已成为国内直播行业事实标准：

**核心特性**：
- 支持 RTMP/HLS/HTTP-FLV/WebRTC/SRT 全协议
- 内置 HTTP API（查流状态、踢人、统计）
- Docker 镜像开箱即用，无需编译配置
- Origin-Edge 集群方案（源站 + 边缘节点）
- 社区活跃，中文文档完善（https://ossrs.io）

```bash
# Docker 快速启动 SRS
docker run -d --name srs \
  -p 1935:1935 \     # RTMP
  -p 8080:8080 \     # HTTP（HLS + HTTP-FLV + API）
  -p 8000:8000/udp \ # WebRTC
  ossrs/srs:5

# 推流测试
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://localhost:1935/live/stream

# 播放地址：
# RTMP：  rtmp://localhost:1935/live/stream
# HLS：   http://localhost:8080/live/stream.m3u8
# HTTP-FLV：http://localhost:8080/live/stream.flv
```

**SRS 配置文件（`conf/srs.conf`）**：
```nginx
listen              1935;
max_connections     1000;
http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
}
vhost __defaultVhost__ {
    hls {
        enabled     on;
        hls_path    ./objs/nginx/html;
        hls_fragment 6;
        hls_window  60;
    }
}
```

**SRS HTTP API 示例**：
```bash
# 查询所有流
curl http://localhost:1985/api/v1/streams/

# 查询服务器状态
curl http://localhost:1985/api/v1/summaries/

# 踢掉某个客户端
curl -X DELETE http://localhost:1985/api/v1/clients/{client_id}
```

---

### 5.4 Nginx-RTMP 快速上手

**Nginx-RTMP** 是 Nginx 的流媒体扩展模块，由 Roman Arutyunyan 开发（2012 年），成熟稳定：

**核心特性**：
- 基于 Nginx 事件驱动模型，性能稳定
- 支持 RTMP/HLS/HTTP-FLV（不支持 WebRTC）
- 配置简洁，与 Nginx 统一管理
- 社区分支活跃维护（上游 2017 年停更）

**注意**：上游仓库（arut/nginx-rtmp-module）已停止维护，建议用社区分支：
- [winshining/nginx-http-flv-module](https://github.com/winshining/nginx-http-flv-module)（推荐，支持 HTTP-FLV + RTMP）
- [sergey-dryabzhinsky/nginx-rtmp-module](https://github.com/sergey-dryabzhinsky/nginx-rtmp-module)（社区维护版）

**Docker 快速启动**：
```bash
# 使用包含 RTMP 模块的 Nginx 镜像
docker run -d --name nginx-rtmp \
  -p 1935:1935 \
  -p 8080:80 \
  alfg/nginx-rtmp

# 推流测试
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://localhost:1935/stream/test

# 播放地址：
# RTMP：rtmp://localhost:1935/stream/test
# HLS： http://localhost:8080/hls/test.m3u8
```

**手动安装（编译版）**：
```bash
# Ubuntu/Debian
sudo apt install build-essential libpcre3 libpcre3-dev libssl-dev zlib1g-dev

# 下载 Nginx 和 RTMP 模块
wget http://nginx.org/download/nginx-1.24.0.tar.gz
git clone https://github.com/winshining/nginx-http-flv-module.git

# 编译安装
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0
./configure --add-module=../nginx-http-flv-module
make && sudo make install
```

**配置文件（/usr/local/nginx/conf/nginx.conf）**：
```nginx
rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        
        application live {
            live on;
            
            # HLS 输出
            hls on;
            hls_path /var/hls;
            hls_fragment 6s;
            hls_playlist_length 60s;
            hls_continuous on;
            
            # 录制（可选）
            record all;
            record_path /var/rec;
            record_suffix _%Y%m%d_%H%M%S.flv;
            
            # 转推到其他平台（可选）
            # push rtmp://platform-a.com/live/key-a;
            # push rtmp://platform-b.com/live/key-b;
        }
    }
}

http {
    server {
        listen 8080;
        
        # HTTP-FLV 播放
        location /live {
            flv_live on;
            chunked_transfer_encoding on;
        }
        
        # HLS 播放
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /var;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }
        
        # 统计页面
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            root /path/to/nginx-http-flv-module;
        }
    }
}
```

**启动服务**：
```bash
# 启动 Nginx
sudo /usr/local/nginx/sbin/nginx

# 重新加载配置
sudo /usr/local/nginx/sbin/nginx -s reload

# 停止服务
sudo /usr/local/nginx/sbin/nginx -s stop
```

**Nginx-RTMP 优势**：
- ✅ 稳定可靠，生产环境验证多年
- ✅ 配置简洁，与 Nginx 统一管理
- ✅ 资源占用低，适合简单场景
- ✅ 已有 Nginx 环境可直接复用

**Nginx-RTMP 限制**：
- ❌ 不支持 WebRTC（需配合 Janus/Jitsi）
- ❌ 不支持 SRT
- ❌ 无内置 HTTP API（需自己开发或用第三方工具）
- ⚠️ 上游停更，依赖社区分支

---

## 6. 断流重连与健康监测

### 6.1 FFmpeg 内置断流恢复

```bash
# 推流断流自动重连（无限次，每次等 2 秒）
while true; do
  ffmpeg -re -i input.mp4 \
    -c:v libx264 -preset veryfast -tune zerolatency -b:v 2M \
    -c:a aac -b:a 128k \
    -f flv \
    -reconnect 1 \
    -reconnect_at_eof 1 \
    -reconnect_streamed 1 \
    -reconnect_delay_max 10 \
    rtmp://live.example.com/live/stream
  echo "推流断开，2秒后重连..."
  sleep 2
done
```

**重连参数说明**：
- `-reconnect 1`：连接失败时自动重连
- `-reconnect_at_eof 1`：文件播放完后重连（循环播放文件流）
- `-reconnect_streamed 1`：流中断时重连
- `-reconnect_delay_max 10`：最大重连间隔 10 秒

### 6.2 流健康检测

```bash
# 检测 RTMP 流是否存活（超时 5 秒）
ffprobe -timeout 5000000 \
  -i rtmp://live.example.com/live/stream \
  -show_entries stream=codec_name,codec_type,width,height,bit_rate \
  -of json 2>/dev/null

# 返回码：0 = 存活，非 0 = 异常
if [ $? -eq 0 ]; then
  echo "流正常"
else
  echo "流异常，触发告警"
fi
```

### 6.3 生产级推流脚本示例

```bash
#!/bin/bash
# rtmp_push.sh - 生产级 RTMP 推流脚本

INPUT="$1"
RTMP_URL="$2"
MAX_RETRIES=10
RETRY_COUNT=0

if [ -z "$INPUT" ] || [ -z "$RTMP_URL" ]; then
  echo "用法: $0 <输入文件> <RTMP地址>"
  exit 1
fi

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  echo "[$(date)] 开始推流 (尝试 $((RETRY_COUNT + 1))/$MAX_RETRIES)"
  
  ffmpeg -re -i "$INPUT" \
    -c:v libx264 -preset veryfast -tune zerolatency \
    -b:v 2M -maxrate 2M -bufsize 4M \
    -g 60 -keyint_min 60 -sc_threshold 0 \
    -c:a aac -b:a 128k -ar 44100 \
    -f flv \
    -reconnect 1 -reconnect_streamed 1 \
    "$RTMP_URL"
  
  EXIT_CODE=$?
  
  if [ $EXIT_CODE -eq 0 ]; then
    echo "[$(date)] 推流正常结束"
    exit 0
  fi
  
  RETRY_COUNT=$((RETRY_COUNT + 1))
  echo "[$(date)] 推流异常退出（exit code: $EXIT_CODE），3秒后重试..."
  sleep 3
done

echo "[$(date)] 达到最大重试次数，推流失败"
exit 1
```

---
## 7. 动手实践

### 实验一：本地搭建 RTMP 服务器并推流

```bash
# 1. 启动 SRS 服务器
docker run -d --name srs -p 1935:1935 -p 8080:8080 ossrs/srs:5

# 2. 推流
ffmpeg -re -i test.mp4 -c copy -f flv rtmp://localhost:1935/live/test

# 3. 播放（三种方式）
ffplay rtmp://localhost:1935/live/test              # RTMP 直播
ffplay http://localhost:8080/live/test.flv          # HTTP-FLV
ffplay http://localhost:8080/live/test.m3u8         # HLS

# 4. 查看 SRS 状态
curl http://localhost:1985/api/v1/streams/ | python3 -m json.tool
```

### 实验二：生成测试流（无需真实视频源）

```bash
# 用 FFmpeg 内置测试源推流（彩条 + 正弦音）
ffmpeg -re \
  -f lavfi -i "testsrc2=size=1280x720:rate=30" \
  -f lavfi -i "sine=frequency=1000:sample_rate=44100" \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 2M \
  -c:a aac -b:a 128k \
  -f flv rtmp://localhost:1935/live/testsrc

# 带时间戳的测试流（便于观察延迟）
ffmpeg -re \
  -f lavfi -i "testsrc2=size=1280x720:rate=30" \
  -vf "drawtext=text='%{localtime}':fontsize=48:fontcolor=white:x=50:y=50" \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 2M \
  -f flv rtmp://localhost:1935/live/clock
```

### 实验三：多平台同步直播

```bash
# 从本地 SRS 拉流，转推到多个平台（流复制，零编码开销）
ffmpeg -i rtmp://localhost:1935/live/source \
  -c copy -f flv rtmp://platform-a.com/live/key-a \
  -c copy -f flv rtmp://platform-b.com/live/key-b \
  -c copy -f flv rtmp://platform-c.com/live/key-c
```

### 实验四：RTMP 转 HLS + HTTP-FLV 双输出

```bash
# 一次编码，多协议输出（tee 复用器）
ffmpeg -i rtmp://localhost:1935/live/source \
  -c:v libx264 -preset veryfast -b:v 2M -c:a aac -b:a 128k \
  -f tee -map 0:v -map 0:a \
  "[f=hls:hls_time=6:hls_list_size=5]/var/hls/index.m3u8|\
[f=flv]rtmp://localhost:1935/live/output"
```

---

## 8. 本章总结

本章覆盖了 RTMP 直播推流的完整链路：

| 场景 | 关键技术 | 核心命令/参数 |
|------|----------|---------------|
| RTMP 推流 | libx264 + FLV | `-tune zerolatency -g 60 -f flv` |
| 摄像头推流 | avfoundation/v4l2 | `-f avfoundation -i "0:0"` |
| RTMP 转 HLS | `-c copy` 流复制 | `-f hls -hls_time 6` |
| 多路转推 | tee 复用器 | `-f tee "[f=flv]url1|[f=flv]url2"` |
| 断流重连 | `-reconnect 1` + shell 循环 | `-reconnect_delay_max 10` |
| 流健康检测 | ffprobe + JSON 解析 | `-timeout 5000000` |
| 服务器搭建 | SRS / Nginx-RTMP | Docker 秒级启动 |

**关键原则**：

1. **生产推流必须固定 GOP**：`-g fps×2 -keyint_min fps×2 -sc_threshold 0`，避免变长 GOP 导致播放卡顿
2. **多路转推用 `-c copy`**：流复制零编码开销，CPU 几乎不增加
3. **tee 复用器优于多次 ffmpeg**：一次解码多路输出，比起多个 FFmpeg 进程节省 50% 以上资源
4. **断流重连是生产必备**：所有长连接推流都要有 while 循环或重试逻辑
5. **SRS 是新项目首选**：Docker 部署简单，协议支持全，HTTP API 完善

**延迟对比**：
- 无交互直播（录播/普通直播）→ HLS/DASH（2-30s，CDN 友好）
- 低延迟直播（游戏/秀场）→ RTMP/LL-HLS（1-3s）
- 超低延迟互动（连麦/电商直播）→ WebRTC（< 1s）

**与其他协议的配合**：
- RTMP 推流 → HLS/HTTP-FLV 播放（本章重点）
- RTMP 推流 → WebRTC 播放（需 SRS 等支持转换）
- RTSP 摄像头 → RTMP 推流（见第 19 篇：RTSP 摄像头接入）

**下一章**：[第 19 篇：RTSP 摄像头接入](19-RTSP摄像头接入.md)

深入 RTSP 协议原理、摄像头接入、GB/T 28181 国标、多摄像头批量管理与监控上云场景。

---

## 9. 术语表

| 术语 | 英文全称 | 说明 |
|------|----------|------|
| RTMP | Real-Time Messaging Protocol | Adobe 实时消息传输协议 |
| FLV | Flash Video | RTMP 使用的容器格式 |
| CBR | Constant Bit Rate | 恒定码率 |
| VBV | Video Buffering Verifier | 视频缓冲验证，控制码率平稳性 |
| GOP | Group of Pictures | 图像组（关键帧间隔）|
| SRS | Simple Realtime Server | 国内开源流媒体服务器，支持全协议 |
| NMP | Nginx-RTMP-Module | Nginx 的 RTMP 扩展模块 |
| tee | tee muxer | FFmpeg 复用器，一路输入多路输出 |
| HTTP-FLV | HTTP Flash Video | 基于 HTTP 长连接传输 FLV，替代 RTMP 播放 |
| OBS | Open Broadcaster Software | 开源直播推流软件 |
| CDN | Content Delivery Network | 内容分发网络 |
| HLS | HTTP Live Streaming | Apple 开发的 HTTP 分片协议 |
| DASH | Dynamic Adaptive Streaming over HTTP | MPEG 开发的自适应码率协议 |
| WebRTC | Web Real-Time Communication | 浏览器内 P2P 实时通信协议 |

---

*本文属于 [FFmpeg 完全指南](../README.md) 系列第 18 篇。*
