# 第 20 篇：SRT 弱网推流实战

> **模块**：04-流媒体应用 | **难度**：中级 | **预计阅读**：40 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-08-19

---

## 目录

1. [SRT 与 FFmpeg](#1-srt-与-ffmpeg)
2. [FFmpeg SRT 编译与验证](#2-ffmpeg-srt-编译与验证)
3. [FFmpeg SRT 推流](#3-ffmpeg-srt-推流)
4. [FFmpeg SRT 拉流](#4-ffmpeg-srt-拉流)
5. [SRT 参数调优](#5-srt-参数调优)
6. [SRT 服务器搭建](#6-srt-服务器搭建)
7. [弱网场景实战](#7-弱网场景实战)
8. [SRT 常见问题处理](#8-srt-常见问题处理)
9. [动手实践](#9-动手实践)
10. [本章总结](#10-本章总结)

---

## 学习目标

完成本章后，你将能够：

- 编译支持 SRT 的 FFmpeg
- 使用 FFmpeg 进行 SRT 推流和拉流
- 根据网络条件调优 SRT 参数（latency、maxbw、overhead）
- 搭建 SRT 服务器（srt-live-server/SRS）
- 处理弱网场景（4G/5G、跨国、高丢包）
- 解决 SRT 常见问题（连接失败、防火墙、NAT 穿透）

---

## 前置阅读

**SRT 协议原理**：本章专注于 FFmpeg 实战，协议细节（ARQ 机制、加密原理、延迟控制）请参考：
- [streaming-fundamentals-tutorial/01-底层技术基础/09-SRT协议与弱网推流.md](../../streaming-fundamentals-tutorial/01-底层技术基础/09-SRT协议与弱网推流.md)

**本章定位**：FFmpeg 命令行使用、参数调优、生产环境部署。

---

## 1. SRT 与 FFmpeg

### 1.1 为什么选择 SRT

**SRT 的核心优势**（相比 RTMP）：

```
┌────────────────────────────────────────────────┐
│         RTMP vs SRT 弱网对比                    │
│                                                  │
│  丢包率 5%：                                     │
│    RTMP：严重卡顿（TCP 阻塞）                    │
│    SRT：流畅（UDP + 选择性重传）                 │
│                                                  │
│  丢包率 15%：                                    │
│    RTMP：几乎不可用                              │
│    SRT：轻微卡顿（latency 需增加）               │
│                                                  │
│  丢包率 30%：                                    │
│    RTMP：完全断流                                │
│    SRT：仍可用（latency 需设置 2s+）             │
└────────────────────────────────────────────────┘
```

**典型应用场景**：
- ✅ **移动推流**：4G/5G 网络不稳定，丢包率 5-15%
- ✅ **跨国推流**：中国 → 美国，高延迟（200ms+）
- ✅ **无人机直播**：无线链路，丢包率 10-20%
- ✅ **新闻现场**：户外环境，网络质量差
- ✅ **卫星通信**：高延迟（500ms+）、高丢包
- ❌ **稳定局域网**：不如 RTMP 简单（RTMP 足够）
- ❌ **浏览器直接播放**：SRT 无浏览器支持（需服务器转 HLS/HTTP-FLV）

### 1.2 FFmpeg SRT 支持情况

| 功能 | FFmpeg 支持 | 说明 |
|------|-------------|------|
| SRT 推流 | ✅ 完全支持 | `srt://host:port` |
| SRT 拉流 | ✅ 完全支持 | `-i srt://host:port` |
| Caller 模式 | ✅ 支持 | 主动连接（推流默认）|
| Listener 模式 | ✅ 支持 | 监听端口（拉流默认）|
| Rendezvous 模式 | ✅ 支持 | 双向连接（穿 NAT）|
| AES 加密 | ✅ 支持 | `passphrase=xxx` |
| 参数调优 | ✅ 支持 | latency/maxbw/overhead 等 |

**限制**：
- FFmpeg 本身不是 SRT 服务器（不能同时接收多路推流），需配合 srt-live-server/SRS
- FFmpeg 的 SRT 支持需要编译时启用 `libsrt`

---

## 2. FFmpeg SRT 编译与验证

### 2.1 检查当前 FFmpeg 是否支持 SRT

```bash
# 方法1：查看 configuration
ffmpeg -version | grep srt

# 方法2：查看 protocols
ffmpeg -protocols 2>&1 | grep srt

# 期望输出：
#  Output: srt
#  Input: srt
```

### 2.2 macOS 安装支持 SRT 的 FFmpeg

```bash
# Homebrew 默认启用 SRT
brew install ffmpeg

# 如果已安装但未启用 SRT，重新编译
brew reinstall ffmpeg --with-srt

# 验证
ffmpeg -protocols 2>&1 | grep srt
```

### 2.3 Ubuntu/Debian 编译 FFmpeg with SRT

```bash
# 1. 安装依赖
sudo apt update
sudo apt install -y build-essential yasm cmake git

# 2. 编译安装 libsrt
git clone https://github.com/Haivision/srt.git
cd srt
./configure
make -j$(nproc)
sudo make install
sudo ldconfig

# 3. 编译 FFmpeg
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure \
  --enable-libsrt \
  --enable-gpl \
  --enable-nonfree \
  --enable-libx264 \
  --enable-libx265
make -j$(nproc)
sudo make install

# 4. 验证
ffmpeg -protocols 2>&1 | grep srt
```

### 2.4 Docker 快速启动（推荐）

```bash
# 使用预编译好的镜像
docker run --rm -it \
  -v $(pwd):/workspace \
  linuxserver/ffmpeg:latest \
  ffmpeg -protocols 2>&1 | grep srt

# 或使用 jrottenberg/ffmpeg（支持 SRT）
docker run --rm -it jrottenberg/ffmpeg:4.4-ubuntu \
  -protocols 2>&1 | grep srt
```

---

## 3. FFmpeg SRT 推流

### 3.1 基础推流（Caller 模式）

```bash
# 本地文件 → SRT 推流（默认 Caller 模式，主动连接服务器）
ffmpeg -re -i input.mp4 \
  -c copy \
  -f mpegts \
  srt://192.168.1.100:9000

# 参数说明：
# -re          按实时速率读取（模拟直播）
# -c copy      流复制（不重新编码）
# -f mpegts    封装格式必须是 mpegts（SRT 不支持 FLV/MP4）
```

**关键点**：
- SRT 推流必须用 **mpegts 封装**（`-f mpegts`），不能用 FLV/MP4
- `-re` 是必需的（否则会尽可能快地发送，导致接收端缓冲溢出）

### 3.2 推流到 SRT 服务器（指定 Stream ID）

```bash
# 推流到 srt-live-server（需要指定 stream ID）
ffmpeg -re -i input.mp4 \
  -c copy \
  -f mpegts \
  "srt://192.168.1.100:9000?streamid=publish/live/stream1"

# stream ID 用于服务器区分不同的流
# 格式：publish/appname/streamname（类似 RTMP 的 app/stream）
```

### 3.3 推流时设置 latency（延迟）

```bash
# 设置 latency 为 1000ms（适合跨国推流）
ffmpeg -re -i input.mp4 \
  -c copy \
  -f mpegts \
  "srt://192.168.1.100:9000?latency=1000000"
# 注意：latency 单位是微秒（1000ms = 1000000us）

# 推荐 latency 设置：
# 本地局域网：  latency=120000   (120ms)
# 国内城市间：  latency=300000   (300ms)
# 跨国（中美）：latency=2000000  (2000ms)
# 卫星链路：    latency=5000000  (5000ms)
```

### 3.4 推流时启用加密

```bash
# AES-128 加密推流
ffmpeg -re -i input.mp4 \
  -c copy \
  -f mpegts \
  "srt://192.168.1.100:9000?passphrase=mySecretKey123&pbkeylen=16"

# AES-256 加密推流
ffmpeg -re -i input.mp4 \
  -c copy \
  -f mpegts \
  "srt://192.168.1.100:9000?passphrase=mySecretKey123&pbkeylen=32"

# 参数说明：
# passphrase  - 密码短语（10-79 字符）
# pbkeylen    - 密钥长度（16=AES-128, 24=AES-192, 32=AES-256）
```

### 3.5 摄像头推流（实时场景）

```bash
# macOS 摄像头 → SRT 推流
ffmpeg -f avfoundation -i "0:0" \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -g 30 \
  -c:a aac -b:a 128k \
  -f mpegts \
  "srt://192.168.1.100:9000?latency=500000"

# Linux 摄像头 → SRT 推流
ffmpeg -f v4l2 -i /dev/video0 \
  -f alsa -i hw:0 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -g 30 \
  -c:a aac -b:a 128k \
  -f mpegts \
  "srt://192.168.1.100:9000?latency=500000"

# 屏幕录制 → SRT 推流（macOS）
ffmpeg -f avfoundation -capture_cursor 1 -i "1:0" \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 3M -g 30 \
  -c:a aac -b:a 128k \
  -f mpegts \
  "srt://192.168.1.100:9000?latency=500000"
```

---

## 4. FFmpeg SRT 拉流

### 4.1 基础拉流（Listener 模式）

```bash
# SRT 拉流 → 本地保存
ffmpeg -i "srt://192.168.1.100:9000?mode=listener" \
  -c copy \
  output.mp4

# 参数说明：
# mode=listener  - 监听模式（等待推流端连接，默认）
# 不指定 mode 时，FFmpeg 自动判断（推流用 caller，拉流用 listener）
```

### 4.2 拉流并播放

```bash
# SRT 拉流 → ffplay 播放
ffplay "srt://192.168.1.100:9000?mode=listener"

# 低延迟播放（减少缓冲）
ffplay -fflags nobuffer -flags low_delay \
  "srt://192.168.1.100:9000?mode=listener"
```

### 4.3 拉流转推到 RTMP

```bash
# SRT 拉流 → RTMP 推流（SRT → CDN）
ffmpeg -i "srt://192.168.1.100:9000?mode=listener" \
  -c copy \
  -f flv \
  rtmp://live.example.com/live/stream1

# 场景：现场用 SRT 推流到中转服务器，服务器转 RTMP 到 CDN
```

### 4.4 拉流转 HLS（浏览器观看）

```bash
# SRT 拉流 → HLS 输出
ffmpeg -i "srt://192.168.1.100:9000?mode=listener" \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -g 30 \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 2 \
  -hls_list_size 10 \
  -hls_flags delete_segments+append_list \
  /var/www/hls/stream1/index.m3u8
```

---

## 5. SRT 参数调优

### 5.1 核心参数表

| 参数 | 默认值 | 说明 | 推荐场景 |
|------|--------|------|---------|
| `latency` | 120ms | 延迟窗口（越大越抗丢包，但延迟越高） | 弱网增加到 1-2s |
| `maxbw` | -1（无限）| 最大带宽（字节/秒）| 限制带宽占用 |
| `overhead` | 25% | 冗余带宽（用于重传）| 高丢包增加到 50% |
| `rcvbuf` | 8192 KB | 接收缓冲区 | 高带宽流增加 |
| `sndbuf` | 8192 KB | 发送缓冲区 | 高带宽流增加 |
| `conntimeo` | 3000ms | 连接超时 | 弱网增加到 10s |
| `peerlatency` | 0 | 对端延迟（协商用）| 通常不设置 |

### 5.2 弱网场景参数推荐

```bash
# 场景1：国内 4G/5G 推流（丢包 5-10%）
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server:9000?latency=500000&overhead=30"

# 场景2：跨国推流（中国 → 美国，丢包 10-15%）
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server:9000?latency=2000000&overhead=40&conntimeo=10000"

# 场景3：卫星链路（丢包 15-30%）
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server:9000?latency=5000000&overhead=50&rcvbuf=16384&sndbuf=16384"

# 场景4：稳定局域网（低延迟优先）
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server:9000?latency=120000&overhead=20"
```

### 5.3 参数调优策略

**latency 调优**：
```bash
# 步骤1：从默认 120ms 开始
# 步骤2：如果出现花屏/卡顿，翻倍（240ms → 500ms → 1000ms）
# 步骤3：找到稳定点后，尝试减半测试
# 最终目标：在保证流畅的前提下，尽可能低
```

**overhead 调优**：
```bash
# 公式：overhead ≈ 丢包率 × 2.5
# 丢包 5%  → overhead=15%
# 丢包 10% → overhead=25%（默认）
# 丢包 20% → overhead=50%
# 丢包 30% → overhead=70-80%
```

**maxbw 限制**：
```bash
# 限制最大带宽为 5 Mbps（避免占满网络）
"srt://server:9000?maxbw=625000"
# 注意：maxbw 单位是字节/秒（5 Mbps = 625000 B/s）

# 公式：maxbw(B/s) = 码率(Mbps) × 125000
```

---
## 6. SRT 服务器搭建

### 6.1 服务器选型

| 服务器 | 语言 | 功能 | 推荐场景 |
|--------|------|------|---------|
| **srt-live-server** | C++ | 纯 SRT 网关（接收推流 → 转发） | 轻量级中转 |
| **SRS 5.0+** | C++ | 完整流媒体服务器（SRT/RTMP/HLS/HTTP-FLV） | 生产环境首选 |
| **Nginx-RTMP + srt-nginx** | C | Nginx + SRT 模块（需编译）| 已有 Nginx 环境 |
| **MediaMTX** | Go | 多协议支持（RTSP/RTMP/SRT/WebRTC） | 轻量级全能 |

### 6.2 方案一：srt-live-server（最简单）

```bash
# 1. 安装（Docker 方式）
docker run -d --name srt-server \
  -p 9000:9000/udp \
  ravenium/srt-live-server

# 2. 推流测试
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  "srt://localhost:9000?streamid=publish/live/stream1"

# 3. 拉流测试
ffplay "srt://localhost:9000?streamid=live/stream1/stream1"

# 注意：streamid 格式是 srt-live-server 特定的
# 推流：publish/<app>/<stream>
# 拉流：<domain>/<app>/<stream>
```

**srt-live-server 配置文件**（`sls.conf`）：
```conf
server {
    listen 9000;
    
    # 推流域名
    domain_publisher live.push;
    # 拉流域名
    domain_player live.pull;
    
    # 默认延迟
    default_sid publish/live/feed;
    
    # 最大连接数
    backlog 100;
}

# 推流 URL：srt://ip:9000?streamid=publish/live/stream1
# 拉流 URL：srt://ip:9000?streamid=live.pull/live/stream1
```

### 6.3 方案二：SRS 5.0+（推荐生产环境）

```bash
# 1. 安装 SRS（Docker）
docker run -d --name srs \
  -p 1935:1935 \
  -p 1985:1985 \
  -p 8080:8080 \
  -p 10080:10080/udp \
  ossrs/srs:5

# 2. 配置 SRS（/etc/srs/srs.conf）
cat > srs.conf << 'SRS'
listen              1935;
max_connections     1000;
daemon              off;
srs_log_tank        console;

# SRT 配置
srt_server {
    enabled on;
    listen 10080;
    maxbw 1000000000;
    connect_timeout 4000;
    peerlatency 0;
    recvlatency 0;
    latency 0;
    tsbpdmode off;
    tlpktdrop on;
    sendbuf 2000000;
    recvbuf 2000000;
}

# VHOST 配置
vhost __defaultVhost__ {
    # SRT 转 RTMP
    srt {
        enabled on;
        srt_to_rtmp on;
    }
    
    # HLS 输出
    hls {
        enabled on;
        hls_path /data/hls;
        hls_fragment 2;
        hls_window 10;
    }
    
    # HTTP-FLV 输出
    http_remux {
        enabled on;
        mount [vhost]/[app]/[stream].flv;
    }
}

# HTTP API
http_api {
    enabled on;
    listen 1985;
}

# HTTP 服务器
http_server {
    enabled on;
    listen 8080;
    dir /data/hls;
}
SRS

# 3. 启动 SRS
docker run -d --name srs \
  -p 1935:1935 -p 1985:1985 -p 8080:8080 -p 10080:10080/udp \
  -v $(pwd)/srs.conf:/etc/srs/srs.conf \
  ossrs/srs:5

# 4. SRT 推流
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/stream1,m=publish"

# 5. 多种拉流方式
# SRT 拉流
ffplay "srt://localhost:10080?streamid=#!::r=live/stream1,m=request"

# RTMP 拉流（SRS 自动转换）
ffplay rtmp://localhost/live/stream1

# HTTP-FLV 拉流
ffplay http://localhost:8080/live/stream1.flv

# HLS 拉流
ffplay http://localhost:8080/live/stream1/index.m3u8
```

**SRS SRT streamid 格式**：
```
# 推流
streamid=#!::r=live/stream1,m=publish

# 拉流
streamid=#!::r=live/stream1,m=request

# 格式说明：
# #!::r=<app>/<stream>  - 应用名和流名
# ,m=publish            - 模式：推流
# ,m=request            - 模式：拉流
```

### 6.4 方案三：MediaMTX（Go 实现，轻量级）

```bash
# 1. 安装 MediaMTX
docker run -d --name mediamtx \
  -p 8554:8554 \
  -p 1935:1935 \
  -p 8888:8888 \
  -p 8889:8889/udp \
  bluenviron/mediamtx

# 2. SRT 推流
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  srt://localhost:8889?streamid=publish:mystream

# 3. 拉流
# SRT 拉流
ffplay srt://localhost:8889?streamid=read:mystream

# RTSP 拉流（MediaMTX 自动转换）
ffplay rtsp://localhost:8554/mystream

# RTMP 拉流
ffplay rtmp://localhost:1935/mystream

# HLS 拉流
ffplay http://localhost:8888/mystream/index.m3u8
```

### 6.5 服务器性能对比

| 服务器 | 单路延迟 | CPU 占用 | 内存占用 | 并发能力 |
|--------|---------|---------|---------|---------|
| srt-live-server | 100-200ms | 低 | 50MB | 100-200 路 |
| SRS 5.0 | 200-500ms | 中 | 100MB | 500-1000 路 |
| MediaMTX | 100-300ms | 低 | 30MB | 200-500 路 |

---

## 7. 弱网场景实战

### 7.1 场景一：4G/5G 移动推流

**网络特点**：
- 丢包率：5-15%
- 延迟：50-200ms
- 带宽波动：2-10 Mbps

**推流配置**：
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -maxrate 2.5M -bufsize 4M \
  -g 30 -sc_threshold 0 \
  -c:a aac -b:a 64k \
  -f mpegts \
  "srt://server:9000?latency=500000&overhead=30&maxbw=312500"
```

**关键参数**：
- `latency=500000`（500ms）：足够应对 4G 网络抖动
- `overhead=30`：30% 冗余应对 10% 丢包
- `maxbw=312500`（2.5 Mbps）：限制带宽防止拥塞
- `-maxrate 2.5M -bufsize 4M`：视频码率限制

### 7.2 场景二：跨国推流（中国 → 美国）

**网络特点**：
- 丢包率：10-20%
- 延迟：200-300ms
- 带宽稳定但延迟高

**推流配置**：
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 3M -maxrate 3.5M -bufsize 6M \
  -g 60 -sc_threshold 0 \
  -c:a aac -b:a 128k \
  -f mpegts \
  "srt://us-server:9000?latency=2000000&overhead=40&conntimeo=10000"
```

**关键参数**：
- `latency=2000000`（2000ms）：应对高延迟和丢包
- `overhead=40`：40% 冗余应对 15% 丢包
- `conntimeo=10000`（10s）：连接超时增加
- `-g 60`：GOP 增加（减少关键帧，降低丢包影响）

### 7.3 场景三：卫星链路（极高延迟）

**网络特点**：
- 丢包率：15-30%
- 延迟：500-1000ms
- 带宽有限（1-2 Mbps）

**推流配置**：
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 1M -maxrate 1.2M -bufsize 2M \
  -g 90 -sc_threshold 0 \
  -c:a aac -b:a 32k \
  -f mpegts \
  "srt://satellite:9000?latency=5000000&overhead=60&rcvbuf=16384&sndbuf=16384"
```

**关键参数**：
- `latency=5000000`（5000ms）：极高延迟窗口
- `overhead=60`：60% 冗余应对 30% 丢包
- `rcvbuf/sndbuf=16384`（16MB）：增加缓冲区
- `-b:v 1M`：降低码率（卫星带宽有限）
- `-g 90`：3 秒 GOP（减少关键帧）

### 7.4 场景四：无人机直播

**网络特点**：
- 丢包率：10-25%（无线链路不稳定）
- 延迟：100-300ms
- 带宽波动大（1-5 Mbps）

**推流配置**：
```bash
ffmpeg -f v4l2 -i /dev/video0 \
  -c:v libx264 -preset ultrafast -tune zerolatency \
  -b:v 1.5M -maxrate 2M -bufsize 3M \
  -g 30 -sc_threshold 0 \
  -c:a aac -b:a 32k \
  -f mpegts \
  "srt://ground-station:9000?latency=1000000&overhead=50&maxbw=250000"
```

**关键参数**：
- `latency=1000000`（1000ms）：应对无线波动
- `overhead=50`：50% 冗余应对高丢包
- `maxbw=250000`（2 Mbps）：限制带宽
- `-preset ultrafast`：降低编码延迟（无人机算力有限）

---

## 8. SRT 常见问题处理

### 8.1 问题一：连接超时

**现象**：
```
[srt] Connection timed out
srt://server:9000: Connection timed out
```

**原因**：
- 服务器未启动或端口未开放
- 防火墙拦截 UDP 9000 端口
- 网络不通

**解决方案**：

```bash
# 1. 检查服务器端口是否监听
netstat -uln | grep 9000

# 2. 检查防火墙（Linux）
sudo ufw allow 9000/udp

# 3. 增加连接超时时间
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server:9000?conntimeo=10000"

# 4. 检查 NAT/路由器端口转发（公网场景）
# 确保 UDP 9000 端口映射到内网服务器
```

### 8.2 问题二：推流成功但无法拉流

**现象**：
- FFmpeg 推流显示成功
- 拉流时报错 `No such file or directory`

**原因**：
- streamid 格式错误
- 服务器未正确识别流

**解决方案**：

```bash
# srt-live-server 的正确 streamid 格式
# 推流
"srt://server:9000?streamid=publish/live/stream1"

# 拉流
"srt://server:9000?streamid=live/stream1/stream1"
# 注意：拉流时需要重复流名

# SRS 的正确 streamid 格式
# 推流
"srt://server:10080?streamid=#!::r=live/stream1,m=publish"

# 拉流
"srt://server:10080?streamid=#!::r=live/stream1,m=request"
```

### 8.3 问题三：延迟过高

**现象**：
- 推流到播放延迟 5-10 秒
- 实时互动不可用

**原因**：
- latency 参数设置过大
- 服务器转码增加延迟
- 播放器缓冲过大

**解决方案**：

```bash
# 1. 降低 latency（在保证稳定的前提下）
"srt://server:9000?latency=200000"  # 200ms

# 2. 推流端使用流复制（不转码）
ffmpeg -re -i input.mp4 -c copy -f mpegts ...

# 3. 播放端减少缓冲
ffplay -fflags nobuffer -flags low_delay \
  -framedrop \
  "srt://server:9000?mode=listener"

# 4. 使用 SRT 直接播放（不经过服务器转 RTMP/HLS）
```

### 8.4 问题四：花屏/卡顿

**现象**：
- 播放时出现绿色/灰色花屏
- 间歇性卡顿

**原因**：
- latency 设置过小，丢包无法恢复
- overhead 不足，重传不及时
- 网络丢包率超出预期

**解决方案**：

```bash
# 1. 增加 latency（每次翻倍测试）
"srt://server:9000?latency=500000"   # 500ms
"srt://server:9000?latency=1000000"  # 1000ms
"srt://server:9000?latency=2000000"  # 2000ms

# 2. 增加 overhead
"srt://server:9000?latency=1000000&overhead=50"

# 3. 检查实际丢包率
# 推流端日志会显示：
# [srt] Packet loss: 12.5%
# 根据丢包率调整 overhead = 丢包率 × 2.5

# 4. 测试网络质量
ping -c 100 server-ip
# 查看丢包率和延迟波动
```

### 8.5 问题五：NAT 穿透失败

**现象**：
- 公网推流时连接失败
- 同局域网可以，跨网络不行

**原因**：
- NAT/路由器未映射 UDP 端口
- 运营商拦截 UDP

**解决方案**：

```bash
# 方案1：服务器开启 Listener 模式（推荐）
# 推流端主动连接（Caller），服务器被动监听（Listener）
# 推流端（公网）
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://server-public-ip:9000?mode=caller"

# 服务器端（监听）
ffmpeg -i "srt://0.0.0.0:9000?mode=listener" -c copy output.mp4

# 方案2：使用 Rendezvous 模式（双向连接）
# 适用于双方都在 NAT 后
# A 端
ffmpeg -re -i input.mp4 -c copy -f mpegts \
  "srt://peer-b-ip:9000?mode=rendezvous"

# B 端
ffmpeg -i "srt://peer-a-ip:9000?mode=rendezvous" -c copy output.mp4

# 方案3：使用中继服务器（最可靠）
# 两端都推流到公网服务器，服务器中转
```

### 8.6 问题六：CPU 占用过高

**现象**：
- FFmpeg 推流时 CPU 100%
- 服务器接收多路流时卡顿

**原因**：
- 推流端重复编码（未使用 `-c copy`）
- 服务器转码（SRT → RTMP 转码）

**解决方案**：

```bash
# 1. 推流端使用流复制
ffmpeg -re -i input.mp4 -c copy -f mpegts ...

# 2. 如果必须编码，降低编码复杂度
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset ultrafast \  # veryfast → ultrafast
  -f mpegts ...

# 3. 服务器端避免转码
# SRS 配置：srt_to_rtmp off;（不转 RTMP）
# 直接用 SRT 拉流

# 4. 使用硬件编码（如果支持）
ffmpeg -re -i input.mp4 \
  -c:v h264_videotoolbox \  # macOS 硬件编码
  -b:v 2M \
  -f mpegts ...
```

---
## 9. 动手实践

### 实验一：本地 SRT 推拉流测试

```bash
# 1. 启动 SRT 服务器（SRS）
docker run -d --name srs \
  -p 10080:10080/udp \
  -p 8080:8080 \
  ossrs/srs:5

# 2. 推流（Caller 模式）
ffmpeg -re -stream_loop -1 -i test.mp4 \
  -c copy \
  -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/test,m=publish" &

# 3. 拉流播放（多种方式）
# 方式1：SRT 直接拉流
ffplay "srt://localhost:10080?streamid=#!::r=live/test,m=request"

# 方式2：HTTP-FLV 拉流（SRS 自动转换）
ffplay http://localhost:8080/live/test.flv

# 4. 查看流信息
curl http://localhost:1985/api/v1/streams/
```

### 实验二：模拟弱网环境测试

```bash
# macOS 安装网络模拟工具
brew install network-link-conditioner

# Linux 使用 tc 模拟丢包
# 1. 添加 10% 丢包
sudo tc qdisc add dev eth0 root netem loss 10%

# 2. 推流测试（低 latency）
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/test1,m=publish&latency=120000"

# 3. 推流测试（高 latency）
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/test2,m=publish&latency=1000000"

# 4. 对比播放质量
ffplay "srt://localhost:10080?streamid=#!::r=live/test1,m=request"
ffplay "srt://localhost:10080?streamid=#!::r=live/test2,m=request"

# 5. 清除丢包模拟
sudo tc qdisc del dev eth0 root
```

### 实验三：SRT vs RTMP 弱网对比

```bash
# 1. 启动 SRS（同时支持 SRT 和 RTMP）
docker run -d --name srs \
  -p 1935:1935 \
  -p 10080:10080/udp \
  -p 8080:8080 \
  ossrs/srs:5

# 2. 模拟 15% 丢包
sudo tc qdisc add dev eth0 root netem loss 15%

# 3. 同时推流（SRT 和 RTMP）
# SRT 推流
ffmpeg -re -i test.mp4 -c copy -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/srt_test,m=publish&latency=1000000" &

# RTMP 推流
ffmpeg -re -i test.mp4 -c copy -f flv \
  rtmp://localhost/live/rtmp_test &

# 4. 对比播放（分屏观看）
# SRT 播放
ffplay -x 640 -y 360 -left 0 -top 0 \
  "srt://localhost:10080?streamid=#!::r=live/srt_test,m=request" &

# RTMP 播放
ffplay -x 640 -y 360 -left 650 -top 0 \
  rtmp://localhost/live/rtmp_test &

# 观察：RTMP 会严重卡顿，SRT 相对流畅

# 5. 清理
sudo tc qdisc del dev eth0 root
killall ffmpeg ffplay
```

### 实验四：SRT 加密传输

```bash
# 1. 启动 Listener（服务器端）
ffmpeg -i "srt://0.0.0.0:9000?mode=listener&passphrase=MySecret123&pbkeylen=16" \
  -c copy \
  -f flv rtmp://localhost/live/encrypted &

# 2. 推流（Caller，客户端）
ffmpeg -re -i test.mp4 \
  -c copy \
  -f mpegts \
  "srt://localhost:9000?mode=caller&passphrase=MySecret123&pbkeylen=16"

# 3. 验证加密（错误的密码会连接失败）
ffmpeg -re -i test.mp4 \
  -c copy \
  -f mpegts \
  "srt://localhost:9000?mode=caller&passphrase=WrongPassword&pbkeylen=16"
# 期望输出：Connection refused

# 4. 播放验证
ffplay rtmp://localhost/live/encrypted
```

### 实验五：多路 SRT 推流性能测试

```bash
# 1. 启动 SRS
docker run -d --name srs \
  -p 10080:10080/udp \
  ossrs/srs:5

# 2. 批量推流脚本
cat > multi_push.sh << 'SCRIPT'
#!/bin/bash
for i in {1..10}; do
  ffmpeg -re -stream_loop -1 -i test.mp4 \
    -c copy \
    -f mpegts \
    "srt://localhost:10080?streamid=#!::r=live/stream${i},m=publish" \
    > /dev/null 2>&1 &
  echo "推流 stream${i} 已启动"
  sleep 1
done
SCRIPT

chmod +x multi_push.sh
./multi_push.sh

# 3. 监控服务器资源
docker stats srs

# 4. 查看所有流
curl http://localhost:1985/api/v1/streams/ | jq

# 5. 清理
killall ffmpeg
```

### 实验六：SRT → HLS 转码（Web 播放）

```bash
# 1. 启动 SRS
docker run -d --name srs \
  -p 10080:10080/udp \
  -p 8080:8080 \
  -v $(pwd)/hls:/usr/local/srs/objs/nginx/html/live \
  ossrs/srs:5

# 2. SRT 推流
ffmpeg -re -i test.mp4 \
  -c copy \
  -f mpegts \
  "srt://localhost:10080?streamid=#!::r=live/web,m=publish" &

# 3. 生成 HLS（SRS 自动转换）
# 访问：http://localhost:8080/live/web/index.m3u8

# 4. 创建 HTML 播放器
cat > player.html << 'HTML'
<!DOCTYPE html>
<html>
<head>
  <title>SRT → HLS Player</title>
  <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
</head>
<body>
  <video id="video" controls width="800"></video>
  <script>
    const video = document.getElementById('video');
    const hls = new Hls();
    hls.loadSource('http://localhost:8080/live/web/index.m3u8');
    hls.attachMedia(video);
    hls.on(Hls.Events.MANIFEST_PARSED, () => video.play());
  </script>
</body>
</html>
HTML

# 5. 浏览器打开 player.html 观看
```

---

## 10. 本章总结

本章覆盖了 FFmpeg SRT 的完整使用链路：

| 场景 | 关键命令/参数 | 核心要点 |
|------|--------------|---------|
| **SRT 推流** | `ffmpeg -re -i input.mp4 -c copy -f mpegts srt://server:9000` | 必须用 mpegts 封装 |
| **SRT 拉流** | `ffmpeg -i "srt://server:9000?mode=listener" -c copy output.mp4` | 默认 listener 模式 |
| **弱网优化** | `latency=1000000&overhead=40` | latency 越大越抗丢包 |
| **连接模式** | `mode=caller/listener/rendezvous` | caller 主动连接，listener 被动 |
| **加密传输** | `passphrase=xxx&pbkeylen=16` | 内置 AES 加密 |
| **服务器选型** | SRS 5.0+ / srt-live-server / MediaMTX | 生产环境推荐 SRS |
| **4G/5G 推流** | `latency=500000&overhead=30` | 500ms 延迟 + 30% 冗余 |
| **跨国推流** | `latency=2000000&overhead=40` | 2s 延迟 + 40% 冗余 |
| **卫星链路** | `latency=5000000&overhead=60` | 5s 延迟 + 60% 冗余 |

**关键原则**：

1. **latency 是灵魂参数**：太小会花屏，太大延迟高，需根据网络实测调整
2. **overhead = 丢包率 × 2.5**：丢包 10% 则 overhead 设置 25%
3. **推流必须 mpegts**：SRT 不支持 FLV/MP4 封装
4. **生产环境用 SRS**：功能完整，支持 SRT/RTMP/HLS 互转
5. **加密很简单**：一个 passphrase 参数搞定，无需配置 TLS

**SRT vs RTMP 选型**：

| 场景 | 推荐协议 | 原因 |
|------|---------|------|
| 稳定局域网/机房 | RTMP | 简单，生态成熟 |
| 4G/5G 移动推流 | SRT | 抗丢包能力强 |
| 跨国推流 | SRT | 高延迟环境下更稳定 |
| 无人机/户外 | SRT | 无线链路丢包率高 |
| 卫星通信 | SRT | 唯一可用选择 |
| 浏览器直接播放 | 都不行 | 需转 HLS/HTTP-FLV/WebRTC |

**与其他协议的配合**：
- SRT 推流 → SRS 转 RTMP → CDN 分发（监控上云，见第 19 篇）
- SRT 推流 → SRS 转 HLS → 浏览器播放
- SRT 推流 → SRS 转 WebRTC → 超低延迟互动（需 WebRTC 网关）

**下一章**：[第 21 篇：直播录制与切片](21-直播录制与切片.md)

深入直播录制策略（按时长/大小/时钟切片）、断点续录、存储管理与自动化归档。

---

## 附录：SRT 参数速查表

### A.1 连接参数

| 参数 | 类型 | 默认值 | 说明 | 示例 |
|------|------|--------|------|------|
| `mode` | string | auto | 连接模式：caller/listener/rendezvous | `mode=caller` |
| `streamid` | string | - | 流标识符（服务器路由用）| `streamid=#!::r=live/stream1,m=publish` |
| `passphrase` | string | - | AES 加密密码（10-79 字符）| `passphrase=MySecret123` |
| `pbkeylen` | int | 0 | 密钥长度：0=不加密, 16=AES-128, 32=AES-256 | `pbkeylen=16` |
| `conntimeo` | int | 3000 | 连接超时（毫秒）| `conntimeo=10000` |

### A.2 传输参数

| 参数 | 类型 | 默认值 | 说明 | 示例 |
|------|------|--------|------|------|
| `latency` | int | 120000 | 延迟窗口（微秒）| `latency=1000000` (1s) |
| `overhead` | int | 25 | 冗余带宽百分比（用于重传）| `overhead=40` (40%) |
| `maxbw` | int | -1 | 最大带宽（字节/秒），-1=无限 | `maxbw=625000` (5 Mbps) |
| `rcvbuf` | int | 8192000 | 接收缓冲区（字节）| `rcvbuf=16384000` (16MB) |
| `sndbuf` | int | 8192000 | 发送缓冲区（字节）| `sndbuf=16384000` (16MB) |
| `peerlatency` | int | 0 | 对端延迟（微秒）| `peerlatency=500000` |

### A.3 高级参数

| 参数 | 类型 | 默认值 | 说明 | 示例 |
|------|------|--------|------|------|
| `transtype` | string | live | 传输类型：live/file | `transtype=live` |
| `tlpktdrop` | bool | true | 是否丢弃过期包 | `tlpktdrop=1` |
| `nakreport` | bool | true | 是否发送 NACK 报告 | `nakreport=1` |
| `payloadsize` | int | 1316 | UDP 包大小（字节）| `payloadsize=1400` |
| `fc` | int | 25600 | 流控窗口大小（包数）| `fc=32000` |

### A.4 常用参数组合

```bash
# 局域网低延迟
latency=120000&overhead=20

# 4G/5G 移动网络
latency=500000&overhead=30&maxbw=312500

# 跨国推流
latency=2000000&overhead=40&conntimeo=10000

# 卫星链路
latency=5000000&overhead=60&rcvbuf=16384000&sndbuf=16384000

# 加密传输
passphrase=MySecret123&pbkeylen=16&latency=500000

# NAT 穿透
mode=rendezvous&latency=1000000

# 限制带宽（2 Mbps）
maxbw=250000&latency=500000
```

### A.5 单位换算

**时间单位**（微秒 → 毫秒 → 秒）：
```
120ms  = 120000 微秒
500ms  = 500000 微秒
1s     = 1000000 微秒
2s     = 2000000 微秒
5s     = 5000000 微秒
```

**带宽单位**（字节/秒 → Mbps）：
```
1 Mbps   = 125000 字节/秒
2 Mbps   = 250000 字节/秒
5 Mbps   = 625000 字节/秒
10 Mbps  = 1250000 字节/秒
```

**缓冲区单位**（字节 → KB → MB）：
```
8192 KB  = 8192000 字节
16 MB    = 16384000 字节
32 MB    = 33554432 字节
```

---

*本文属于 [FFmpeg 完全指南](../README.md) 系列第 20 篇。*
