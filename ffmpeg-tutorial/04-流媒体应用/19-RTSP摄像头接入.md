# 第 19 篇：RTSP 摄像头接入

> **模块**：04-流媒体应用 | **难度**：中级 | **预计阅读**：50 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-08-19

---

## 目录

1. [RTSP 协议概述](#1-rtsp-协议概述)
2. [RTSP 协议原理](#2-rtsp-协议原理)
3. [RTSP 与 ONVIF 标准](#3-rtsp-与-onvif-标准)
4. [GB/T 28181 国标详解](#4-gbt-28181-国标详解)
5. [摄像头接入实战](#5-摄像头接入实战)
6. [主码流与子码流策略](#6-主码流与子码流策略)
7. [RTSP 常见问题处理](#7-rtsp-常见问题处理)
8. [多摄像头批量管理](#8-多摄像头批量管理)
9. [监控上云场景](#9-监控上云场景)
10. [动手实践](#10-动手实践)
11. [本章总结](#11-本章总结)
12. [术语表](#12-术语表)

---

## 学习目标

完成本章后，你将能够：

- 理解 RTSP/RTP/RTCP 协议栈与传输模式
- 掌握 ONVIF 标准与 GB/T 28181 国标的区别与应用场景
- 接入常见 IPC 厂商摄像头（海康/大华/宇视）
- 处理 RTSP 常见问题（超时、音视频不同步、编码兼容）
- 实现多摄像头批量管理与健康监测
- 完成监控上云（RTSP → RTMP/HLS/WebRTC）

---

## 1. RTSP 协议概述

### 1.1 RTSP 的历史定位

RTSP（Real Time Streaming Protocol，RFC 2326）是 1998 年 IETF 制定的流媒体控制协议，最初为了解决 IP 摄像头的远程控制问题。

**时间线**：
- **1998**：IETF 发布 RTSP 1.0（RFC 2326），配合 RTP/RTCP 设计
- **2000-2010**：IP 摄像头（IPC）普及，RTSP 成为监控行业标准
- **2008**：ONVIF 联盟成立，推动摄像头互操作标准
- **2011**：GB/T 28181 国标发布，中国政府/公安项目强制使用
- **2016**：RTSP 2.0（RFC 7826）发布，改进会话管理与安全性

### 1.2 RTSP 的应用场景

**RTSP 的核心定位**：摄像头取流，而非直播推流（RTMP 的强项）。

```
┌────────────────────────────────────────────────────┐
│          RTMP vs RTSP 场景对比                      │
│                                                      │
│  RTMP：主播/编码器 ──推流──▶ 直播平台               │
│        (OBS/手机 App)                                │
│                                                      │
│  RTSP：摄像头 ──被拉流──▶ NVR/VMS/平台              │
│        (海康/大华 IPC)                               │
└────────────────────────────────────────────────────┘
```

**典型应用场景**：
- ✅ **监控录像**：摄像头 → NVR（网络录像机）→ 存储
- ✅ **安防系统**：摄像头 → VMS（视频管理系统）→ 监控中心
- ✅ **监控上云**：摄像头 → 云平台 → 手机 App/Web 观看
- ✅ **AI 分析**：摄像头 → AI 服务器 → 人脸识别/行为分析
- ❌ **直播推流**：不是 RTSP 的设计目标（用 RTMP/SRT）

### 1.3 RTSP 的技术特点

| 维度 | 特性 |
|------|------|
| 传输层 | TCP（控制信令）+ UDP/TCP（媒体数据）|
| 默认端口 | 554（RTSP 信令）|
| 端到端延迟 | 0.2-1 秒（局域网）|
| CDN 支持 | 差（无状态连接，不适合大规模分发）|
| 防火墙穿透 | 差（UDP 动态端口，TCP 模式好一些）|
| 丢包抗性 | 可配置（UDP 模式快但丢包，TCP 模式慢但可靠）|
| 主要场景 | 摄像头取流（IPC → NVR/平台）|
| FFmpeg 支持 | ✅ 原生支持 |

**RTSP vs RTMP**：
- 使用场景：RTSP 适合取流（被动拉流），RTMP 适合推流（主动推送）
- 延迟：RTSP 更低（0.2-1s vs 1-3s），但 CDN 分发能力弱
- 硬件支持：摄像头原生支持 RTSP，RTMP 需要额外开发
- 
---

## 2. RTSP 协议原理

### 2.1 RTSP 不传数据，只做控制

RTSP 常被误解为传输协议。实际上它是**控制协议**——像遥控器一样发指令，真正的音视频数据由 RTP 传输：

```
┌───────────────────────────────────────────────────────┐
│              RTSP / RTP / RTCP 三件套                  │
│                                                         │
│  RTSP (TCP 554)   ── 控制信令：播放/暂停/停止           │
│      OPTIONS / DESCRIBE / SETUP / PLAY / TEARDOWN      │
│                                                         │
│  RTP  (UDP 动态)  ── 音视频数据：带时间戳的媒体包       │
│                                                         │
│  RTCP (UDP 动态)  ── 质量反馈：丢包率/抖动/往返时延     │
└───────────────────────────────────────────────────────┘
```

**类比理解**：
- RTSP = 电视遥控器（发指令：播放、暂停、快进）
- RTP = 有线电视信号（实际传输画面和声音）
- RTCP = 信号质量监测器（反馈信号强度、干扰情况）

### 2.2 RTSP 交互流程

```
客户端（FFmpeg/VLC）                  服务端（摄像头）
   │                                     │
   │──── OPTIONS ──────────────────────▶ │  查询支持的方法
   │◀─── 200 OK (Public: DESCRIBE...) ── │
   │                                     │
   │──── DESCRIBE rtsp://... ──────────▶ │  请求媒体描述
   │◀─── 200 OK + SDP ────────────────── │  返回 SDP（编码/分辨率）
   │                                     │
   │──── SETUP (Transport: RTP/UDP) ───▶ │  协商传输方式和端口
   │◀─── 200 OK (Session: 12345) ────── │  分配 session + 端口
   │                                     │
   │──── PLAY (Session: 12345) ────────▶ │  开始播放
   │◀─── 200 OK (RTP-Info: seq...) ──── │
   │                                     │
   │◀════════ RTP 音视频数据流 ════════ │  UDP 传输媒体
   │════════▶ RTCP 接收报告 ═══════════ │  反馈质量
   │                                     │
   │──── TEARDOWN ─────────────────────▶ │  结束会话
```

**关键步骤解析**：

1. **OPTIONS**：客户端问摄像头"你支持哪些命令？"
2. **DESCRIBE**：客户端问"这路流是什么编码、分辨率、采样率？"，摄像头返回 SDP 描述
3. **SETUP**：客户端说"我要用 UDP 接收，我的端口是 5000"，摄像头回复"好的，session ID 是 12345，我的端口是 6000"
4. **PLAY**：客户端说"开始播放"，摄像头开始通过 RTP 发送数据
5. **TEARDOWN**：客户端说"结束播放"，摄像头释放资源

### 2.3 SDP（会话描述协议）

DESCRIBE 步骤返回的 SDP 示例：

```
v=0
o=- 1234567890 1234567890 IN IP4 192.168.1.100
s=RTSP Session
c=IN IP4 192.168.1.100
t=0 0
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 profile-level-id=42e01e; packetization-mode=1
a=control:track1
m=audio 0 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/16000/1
a=control:track2
```

**解读**：
- `m=video 0 RTP/AVP 96`：视频流，RTP payload 类型 96
- `a=rtpmap:96 H264/90000`：payload 96 是 H.264 编码，时钟频率 90kHz
- `a=fmtp:96 profile-level-id=42e01e`：H.264 profile 是 Baseline
- `m=audio 0 RTP/AVP 97`：音频流，payload 类型 97
- `a=rtpmap:97 MPEG4-GENERIC/16000/1`：AAC 编码，采样率 16kHz，单声道

### 2.4 RTSP 传输模式

| 模式 | 说明 | 适用场景 | FFmpeg 参数 |
|------|------|----------|-------------|
| UDP 单播 | RTP over UDP，默认模式 | 局域网低延迟 | `-rtsp_transport udp` |
| TCP 交错 | RTP over RTSP（TCP 隧道）| 公网/防火墙环境 | `-rtsp_transport tcp` |
| UDP 组播 | 多客户端共享一路流 | 局域网大量观看 | `-rtsp_transport udp_multicast` |
| HTTP 隧道 | RTSP over HTTP（端口 80）| 严格防火墙 | `-rtsp_transport http` |

```bash
# 指定 TCP 传输（推荐公网使用，避免 UDP 丢包）
ffmpeg -rtsp_transport tcp -i rtsp://admin:pass@192.168.1.100:554/stream1 ...

# 指定 UDP 传输（局域网，延迟最低）
ffmpeg -rtsp_transport udp -i rtsp://192.168.1.100:554/stream1 ...

# HTTP 隧道
ffmpeg -rtsp_transport http -i rtsp://192.168.1.100/stream1 ...
```

**UDP vs TCP 选择**：

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 局域网监控 | UDP | 延迟最低（0.2-0.5s），丢包可接受 |
| 公网远程访问 | TCP | 可靠传输，丢包会重传，穿防火墙容易 |
| 无线网络（WiFi） | TCP | WiFi 丢包率高，UDP 会花屏/卡顿 |
| 录像存储 | TCP | 不能丢帧，确保录像完整 |
| 实时预览（多路） | UDP | 降低服务器负载，丢几帧无所谓 |

---

## 3. RTSP 与 ONVIF 标准

### 3.1 ONVIF 是什么

**ONVIF**（Open Network Video Interface Forum）是由安讯士（Axis）、博世（Bosch）、索尼（Sony）在 2008 年成立的开放标准组织，目标是**让不同厂商的摄像头和 NVR 能互通**。

**核心问题**：
- 2008 年前：海康的摄像头只能配海康的 NVR，大华的摄像头只能配大华的 NVR
- ONVIF 后：任何符合 ONVIF 标准的摄像头可以接入任何符合标准的 NVR/VMS

### 3.2 ONVIF 与 RTSP 的关系

ONVIF 不是协议，是**规范集合**，包括：
- **设备发现**：摄像头如何被自动发现（基于 WS-Discovery）
- **设备管理**：如何配置摄像头参数（分辨率、码率、曝光）
- **流媒体传输**：如何取流（**强制使用 RTSP**）
- **事件订阅**：如何接收报警事件（移动侦测、遮挡报警）
- **PTZ 控制**：如何控制云台转动

```
┌─────────────────────────────────────────────┐
│            ONVIF 协议栈                      │
│                                              │
│  设备发现  ─────  WS-Discovery (SOAP)       │
│  设备管理  ─────  ONVIF Device Service      │
│  流媒体    ─────  RTSP/RTP/RTCP ←─ 核心      │
│  事件订阅  ─────  ONVIF Event Service       │
│  PTZ 控制  ─────  ONVIF PTZ Service         │
└─────────────────────────────────────────────┘
```

**ONVIF Profile**（子集规范）：
- **Profile S**：视频流 + PTZ 控制（2009，最常用）
- **Profile G**：录像存储与回放（2013）
- **Profile T**：高级视频流 + H.265 支持（2018）
- **Profile M**：元数据（2020，AI 分析结果）

### 3.3 如何判断摄像头是否支持 ONVIF

```bash
# 方法1：查看摄像头 Web 界面（通常在"网络"或"协议"设置）
http://192.168.1.100

# 方法2：用 ONVIF Device Manager 扫描
# Windows 工具：https://sourceforge.net/projects/onvifdm/

# 方法3：用 onvif-cli 命令行工具
pip install onvif-cli
onvif-cli --host 192.168.1.100 --port 80 --user admin --password 123456 devicemgmt GetDeviceInformation

# 方法4：FFmpeg 探测（ONVIF 摄像头通常支持 /onvif1 路径）
ffprobe -rtsp_transport tcp -i rtsp://admin:pass@192.168.1.100:554/onvif1
```

**ONVIF RTSP URL 格式**：
- 通用路径：`rtsp://ip:554/onvif1`（Profile S 主码流）
- 海康 ONVIF：`rtsp://ip:554/onvif1`（与私有协议共存）
- 大华 ONVIF：`rtsp://ip:554/onvif1`

---
## 4. GB/T 28181 国标详解

### 4.1 GB/T 28181 是什么

**GB/T 28181**（全称《安全防范视频监控联网系统信息传输、交换、控制技术要求》）是中国公安部在 2011 年发布的**国家强制标准**，规定了公共安全视频监控系统的联网规范。

**核心目标**：
- 打通不同厂商、不同地区的监控系统
- 政府/公安可以统一调阅全国的监控摄像头
- 避免信息孤岛（某市的监控系统无法接入省平台）

### 4.2 为什么有 ONVIF 还要 GB/T 28181

| 维度 | ONVIF | GB/T 28181 |
|------|-------|-----------|
| **制定方** | 国际厂商联盟（安讯士/博世/索尼） | 中国公安部 |
| **应用范围** | 全球商用监控市场 | **中国政府/公安项目强制** |
| **架构设计** | 设备级（摄像头 ↔ NVR）| **平台级**（省平台 ↔ 市平台 ↔ 摄像头） |
| **信令协议** | SOAP/HTTP | **SIP**（类似 VoIP 电话系统） |
| **媒体协议** | RTSP/RTP | **PS 封装（GB/T 1078.1）** |
| **设备编码** | IP 地址 | **20 位国标编码**（省市区设备类型序号） |
| **级联能力** | 弱（设备对设备）| **强**（省级 ↔ 市级 ↔ 区级多级联网） |
| **应用场景** | 企业监控、小区物业 | 雪亮工程、天网工程、政府项目 |

**典型案例**：
- **ONVIF**：某写字楼装了海康摄像头，用大华 NVR 管理（只要都支持 ONVIF）
- **GB/T 28181**：某市公安局要接入全市 10 万个摄像头（包括街道、小区、商场），必须用国标平台

### 4.3 GB/T 28181 架构

```
┌────────────────────────────────────────────────────┐
│              GB/T 28181 三层架构                    │
│                                                      │
│  省级平台 (SIP Server)                              │
│      │                                               │
│      ├── 市级平台 A (SIP Server)                    │
│      │      ├── 区级平台 A1                         │
│      │      │     ├── 摄像头 A1-001                 │
│      │      │     └── 摄像头 A1-002                 │
│      │      └── 区级平台 A2                         │
│      │                                               │
│      └── 市级平台 B (SIP Server)                    │
│           └── 摄像头 B-001                          │
└────────────────────────────────────────────────────┘
```

**20 位国标编码示例**：
```
3201020000001320000001
││││││││││││││││││││││
│││││├──┴─┴──┴─┴──┴─ 设备序号（10位）
││││└────────────── 设备类型（3位）：132=IPC摄像头
│││└─────────────── 行业编码（2位）：00=无
││└──────────────── 区县编码（2位）：02=玄武区
│└───────────────── 市编码（2位）：01=南京市
└────────────────── 省编码（2位）：32=江苏省
```

### 4.4 GB/T 28181 媒体传输：PS 封装

**关键区别**：国标不用标准 RTSP/RTP，而是用 **PS（Program Stream）封装 over RTP**。

| 协议 | 媒体封装 | 传输方式 |
|------|---------|---------|
| ONVIF/标准 RTSP | 裸流 H.264/H.265 | RTP 直接传输 NAL 单元 |
| GB/T 28181 | **PS 封装**（类似 MPEG-2 PS）| RTP 传输 PS 包 |

**PS 封装的麻烦**：
- FFmpeg 默认的 RTSP 解封装器无法直接处理 PS over RTP
- 需要特殊处理或转换（很多平台用 ZLMediaKit/EasyDarwin 做网关）

**为什么国标要用 PS 封装？**
- 兼容老设备（2011 年时很多模拟摄像头编码卡输出 MPEG-2 PS）
- 统一封装（音视频 + 时间戳 + 设备信息放在一起）
- 政治因素（避免完全依赖国外标准）

### 4.5 如何判断摄像头是否支持国标

```bash
# 方法1：查看摄像头 Web 界面
# 通常在"网络" → "协议" → "GB/T 28181"

# 方法2：检查是否有 SIP 注册端口（默认 5060）
nmap -p 5060 192.168.1.100

# 方法3：查看厂商文档
# 海康：需要开启"国标平台接入"，配置 SIP 服务器地址
# 大华：需要开启"国标 28181"，配置上级平台信息
```

### 4.6 GB/T 28181 的流获取方式

国标摄像头不能直接 FFmpeg 拉流，需要通过**国标平台**（SIP Server）中转：

```
┌──────────────────────────────────────────────┐
│         GB/T 28181 取流流程                   │
│                                                │
│  1. 客户端向平台发 SIP INVITE（请求某摄像头流）│
│  2. 平台向摄像头发 SIP INVITE（转发请求）      │
│  3. 摄像头回复 SDP（媒体描述 + RTP 端口）      │
│  4. 平台回复客户端 SDP                        │
│  5. 客户端收 RTP 媒体流（PS 封装）             │
└──────────────────────────────────────────────┘
```

**常见国标平台软件**：
- **商用**：海康 iVMS、大华 DSS、宇视 IMOS
- **开源**：
  - [ZLMediaKit](https://github.com/ZLMediaKit/ZLMediaKit)（C++，最流行）
  - [wvp-GB28181-pro](https://github.com/648540858/wvp-GB28181-pro)（Java，带 Web 界面）
  - [LiveGBS](https://github.com/liveqiu/livegbs)（Go，国标服务器）

### 4.8 流媒体服务器选型对比

在 RTSP 摄像头接入场景中，常用三类开源流媒体服务器：

#### ZLMediaKit
- **实现语言**：C++
- **支持协议**：RTSP、RTMP、HTTP-FLV、WebRTC、HLS、WebSocket、**GB28181**
- **主要用途**：
  - 监控摄像头流媒体转发（RTSP → 其他协议）
  - 直播推流和分发
  - 低延迟实时通信
  - 国标平台（兼容 GB28181）
- **特点**：性能高、协议全、文档相对完善、**国内最流行**
- **GitHub**：[ZLMediaKit/ZLMediaKit](https://github.com/ZLMediaKit/ZLMediaKit)

#### EasyDarwin
- **实现语言**：Go
- **支持协议**：RTSP、HLS
- **主要用途**：轻量级监控流媒体转发
- **特点**：部署简单、资源占用低、协议支持较少
- **适用场景**：只需 RTSP → HLS 的简单转发
- **GitHub**：[EasyDarwin/EasyDarwin](https://github.com/EasyDarwin/EasyDarwin)

#### 与 SRS 的区别
SRS（Simple Realtime Server）主要面向**互联网直播场景**：
- **核心协议**：RTMP、WebRTC、HLS（RTSP 支持较弱）
- **典型用途**：OBS 推流 → SRS → CDN/观众拉流
- **特点**：直播生态完善（鉴权、录制、转码、回源）

**对比总结**：
| 服务器 | 主场景 | 核心协议 | 国标支持 | 适用项目 |
|--------|--------|---------|---------|---------|
| **ZLMediaKit** | 监控/直播通用 | RTSP/RTMP/WebRTC/GB28181 | ✅ | 安防监控、多协议转换 |
| **EasyDarwin** | 监控流转发 | RTSP/HLS | ❌ | 轻量级 RTSP 转 HLS |
| **SRS** | 互联网直播 | RTMP/WebRTC/HLS | ❌ | 直播推流、CDN 源站 |

**选型建议**：
- **需要接入国标摄像头/NVR** → 必须用支持 GB28181 的平台（ZLMediaKit / wvp-GB28181-pro）
- **只转发 RTSP 流到 Web** → ZLMediaKit（协议全）或 EasyDarwin（轻量）
- **互联网直播推流** → SRS
- **需要 WebRTC 低延迟** → ZLMediaKit 或 SRS

### 4.9 实战架构：ZLMediaKit + SRS 配合

在生产环境中，常见做法是 **ZLMediaKit + SRS 双引擎架构**，按场景分工，而不是二选一。

#### 分工原则

| 组件 | 负责场景 | 原因 |
|------|---------|------|
| **ZLMediaKit + WVP** | GB28181 国标摄像头接入 | 只有 ZLM 原生支持 SIP 信令和 PS 码流解复用 |
| **SRS** | RTMP 推流/算法流/录像 | RTMP 生态成熟、DVR 录像配置简单、资源占用低 |

#### 典型数据流路径

**路径 1：国标摄像头接入**
```
GB28181 摄像头（海康/大华）
  │
  ├─ SIP REGISTER（设备注册）
  │   └─→ WVP（iot-gb28181:48088）          # Java 后端，管理 SIP 信令
  │        └─→ ZLMediaKit HTTP API           # WVP 通过 API 控制 ZLM
  │
  └─ RTP 媒体流（PS 封装）
      └─→ ZLMediaKit（媒体处理引擎）
           ├─ PS 解复用 → H.264/H.265
           ├─ 协议转换 → HTTP-FLV / WS-FLV / WebRTC
           └─→ Nginx（zlm-host:6080）→ 前端播放器
```

**路径 2：RTMP 推流（算法/直播）**
```
算法节点 / OBS 推流端
  │
  └─ RTMP 推流（1935）
      └─→ SRS
           ├─ on_publish 回调 → video-service:6000（鉴权）
           ├─ DVR 录像 → data/playbacks/live,ai/（落盘）
           └─ 转流输出 → HTTP-FLV / HLS → 前端播放
```

#### 关键端口与配置

| 组件 | 端口 | 作用 |
|------|------|------|
| **ZLMediaKit** | 6080 | HTTP-FLV/WS-FLV 播流端口 |
| **WVP** | 48088 | GB28181 SIP 信令管理（Java 后端） |
| **SRS** | 1935 | RTMP 推流接收端口 |
| **SRS** | 1985 | HTTP API（流管理、统计） |

**SRS 回调配置**：
```nginx
# SRS 配置片段
http_hooks {
    enabled on;
    on_publish http://video-service:6000/api/srs/on_publish;  # 推流鉴权
    on_unpublish http://video-service:6000/api/srs/on_unpublish;
}

# DVR 录像配置
vhost __defaultVhost__ {
    dvr {
        enabled on;
        dvr_path /data/playbacks/[app]/[stream].[timestamp].flv;
        dvr_plan session;  # 按推流会话录制
    }
}
```

#### DVR（Digital Video Recorder）是什么

DVR 这个词有两种含义，容易混淆：

**含义 1：硬件 DVR（传统安防设备）**
- 早期模拟监控时代的录像机硬盒子，接模拟摄像头（BNC 同轴线），内置硬盘录像
- 现在基本被 NVR 取代，IP 摄像头时代已很少见

**含义 2：软件 DVR 功能（流媒体服务器）**
- SRS/ZLM 等流媒体服务器内置的**录像模块**，不是硬件
- 接收 RTMP/RTSP 推流 → 自动保存成文件（FLV/MP4）
- 本文档提到的 DVR 均指此含义

**DVR vs NVR 对比**：

| 设备 | 接入协议 | 摄像头类型 | 时代 |
|------|---------|-----------|------|
| **DVR（硬件）** | 模拟信号（BNC） | 模拟摄像头 | 2010 年前 |
| **NVR** | 网络（RTSP/ONVIF/GB28181） | IP 摄像头 | 现在主流 |
| **软件 DVR** | RTMP/RTSP | 任意推流 | 云原生场景 |

**摄像头自身不录像**，录像由三种方案负责：

| 方案 | 录像主体 | 适用场景 |
|------|---------|---------|
| **NVR 录像** | 海康/大华硬盘录像机 | 传统监控（7×24 长期存储） |
| **VMS 平台** | 海康 iVMS、大华 DSS | 大型联网监控（集中存储） |
| **流媒体服务器 DVR** | SRS / ZLM 软件录像 | 临时录像、算法流录像（短期） |

**典型配置（大华摄像头 + NVR 录像）**：
```
大华 IPC 摄像头
  ├─ PoE 网线供电
  ├─ 输出 RTSP 流（H.264）
  └─ 接入大华 NVR（DHI-NVR5432-EI）
       └─ 内置 4TB 硬盘，24 小时循环录像
```

**为什么软件 DVR 只用 SRS 而不用 ZLM？**

| 维度 | SRS | ZLMediaKit |
|------|-----|------------|
| **DVR 配置** | 配置文件一键开启，支持 FLV/MP4/HLS 切片 | 需要通过 HTTP API 手动触发录制 |
| **录像策略** | 按推流会话自动录制（`dvr_plan session`） | 需要应用层控制录制生命周期 |
| **文件管理** | 支持自动分段、过期清理、时间戳命名 | 需要自己实现文件管理逻辑 |
| **回放接口** | 原生支持 HTTP-FLV 点播回放 | 需要额外配置 Web 服务器提供文件访问 |
| **场景适配** | 算法流、直播流短期录像（小时级） | 监控流长期存储（天级）通常外接 NVR/VMS |

**典型录像场景**：
- ✅ **AI 算法结果流**：算法节点推 RTMP 到 SRS，自动录像保存分析结果（如车辆抓拍视频），业务系统可回查
- ✅ **临时直播录制**：活动直播、会议录制，推流期间自动落盘
- ❌ **7×24 监控录像**：摄像头长期录像用专业 NVR 或 VMS，不用流媒体服务器

**录像文件路径示例**（SRS DVR）：
```
data/playbacks/
├── live/                         # 直播流录像
│   └── room01.1723456789.flv
├── ai/                           # AI 算法流录像
│   ├── vehicle_detect.1723456790.flv
│   └── face_detect.1723456791.flv
```

#### WVP 是什么

**WVP（wvp-GB28181-pro）= Web Video Platform**，一个开源的 GB28181 国标信令管理平台（Java 实现，带 Web 界面）。

GB28181 协议本身分两层：
- **信令层**：SIP 协议（设备注册、心跳、目录查询、PTZ 控制）
- **媒体层**：RTP 传输 PS 封装的视频流

ZLMediaKit 擅长媒体处理，但信令管理比较弱，所以需要 WVP 配合：

```
GB28181 摄像头
  │
  ├─ SIP 信令（注册、心跳、目录查询、云台控制）
  │   └─→ WVP（Java，管理 SIP 协议 / 设备目录 / Web 界面）
  │        └─→ 调用 ZLM HTTP API（控制拉流）
  │
  └─ RTP 媒体流（PS 封装）
      └─→ ZLMediaKit（C++，媒体处理引擎）
           ├─ PS 解复用 → H.264/H.265
           └─ 协议转换 → HTTP-FLV / WS-FLV / WebRTC → 前端播放
```

**WVP 负责**：
- 接收摄像头 SIP 注册（设备上线/离线/心跳）
- 管理设备目录（通道列表、云台能力）
- 提供 Web 管理界面（预览、录像回放、设备配置）
- 通过 ZLM HTTP API 控制媒体流的拉取和停止

**ZLM 负责**：
- 接收 RTP 媒体流（PS 封装）
- PS 解复用，提取 H.264/H.265 裸流
- 输出 RTSP/RTMP/HTTP-FLV/WebRTC 供前端播放

**为什么分两个组件？**
- Java 生态更适合做信令管理和 Web 后端（Spring Boot）
- C++ 更适合做高性能媒体处理（CPU/内存效率高）
- 职责分离：WVP 管控制面，ZLM 管数据面

**大华摄像头接入 WVP 配置示例**：
```
摄像头 Web 界面：
  网络 → 协议 → GB/T 28181
    SIP 服务器 ID：34020000002000000001
    SIP 服务器域：3402000000
    SIP 服务器地址：192.168.1.100（WVP 服务器 IP）
    SIP 服务器端口：5060
    设备 ID（用户名）：34020000001320000001
```

配置完成后，摄像头自动注册到 WVP，在 WVP Web 界面点"实时预览"即可触发 ZLM 拉流播放。

#### 为什么不能只用一个？

| 方案 | 问题 |
|------|------|
| ❌ **只用 SRS** | 不支持 GB28181 协议，无法接入国标摄像头（政府项目必须） |
| ❌ **只用 ZLM** | RTMP 推流生态不如 SRS 成熟，DVR 配置复杂，需要额外开发 |
| ✅ **ZLM + SRS 异构** | 场景互补：国标走 ZLM，普通 RTMP/录像走 SRS |

#### 大规模扩展架构

```
                      ┌──────────────────────────┐
                      │   流媒体调度中心          │
                      │  (video-service:6000)    │
                      └────────┬─────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
      ┌───────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
      │  ZLM 节点池   │  │  SRS 节点池  │  │  Nginx 集群 │
      │ (GB28181)    │  │  (RTMP/DVR) │  │  (分发层)   │
      ├──────────────┤  ├─────────────┤  └────────────┘
      │ zlm-01:6080  │  │ srs-01:1935 │
      │ zlm-02:6080  │  │ srs-02:1935 │
      │ zlm-03:6080  │  │ srs-03:1935 │
      └──────────────┘  └─────────────┘
          (500路/节点)      (1000路/节点)
```

**扩展能力**：
- **ZLM 单节点**：~500 路并发（GB28181 PS 解复用 + 协议转换，CPU 密集）
- **SRS 单节点**：~1000 路并发（RTMP 透传 + FLV 封装，I/O 密集）
- **万级并发**：10 台 ZLM + 5 台 SRS + 统一调度可支撑 10,000 路摄像头

### 4.7 FFmpeg 与国标设备的配合

**FFmpeg 不能直接拉国标流**，但可以通过国标平台转换后的 RTSP/RTMP 流：

```bash
# 方案1：ZLMediaKit 转换
# 1. 摄像头注册到 ZLMediaKit（国标协议）
# 2. ZLMediaKit 提供 RTSP/RTMP 输出
ffmpeg -i rtsp://zlmedia-server/34020000001320000001 -c copy output.mp4

# 方案2：wvp-GB28181-pro 转换
# 1. 摄像头注册到 wvp
# 2. wvp 提供 HTTP-FLV/HLS 输出
ffmpeg -i http://wvp-server/live/34020000001320000001.flv -c copy output.mp4
```

**你是否需要关注 GB/T 28181？**
- ✅ **政府/公安项目**：必须（雪亮工程、天网工程强制要求）
- ✅ **多级联网场景**：必须（省市区三级平台对接）
- ❌ **企业内部监控**：不需要（用 ONVIF 即可）
- ❌ **小规模项目**：不需要（RTSP 直接取流更简单）

---

## 5. 摄像头接入实战

### 5.1 常见 IPC 厂商 RTSP 地址

| 厂商 | 主码流 RTSP URL | 子码流 RTSP URL |
|------|----------------|----------------|
| **海康威视** | `rtsp://user:pass@ip:554/Streaming/Channels/101` | `rtsp://user:pass@ip:554/Streaming/Channels/102` |
| **大华** | `rtsp://user:pass@ip:554/cam/realmonitor?channel=1&subtype=0` | `rtsp://user:pass@ip:554/cam/realmonitor?channel=1&subtype=1` |
| **宇视** | `rtsp://user:pass@ip:554/media/video1` | `rtsp://user:pass@ip:554/media/video2` |
| **雄迈** | `rtsp://user:pass@ip:554/user=admin&password=&channel=1&stream=0.sdp` | `rtsp://user:pass@ip:554/user=admin&password=&channel=1&stream=1.sdp` |
| **ONVIF 通用** | `rtsp://user:pass@ip:554/onvif1` | `rtsp://user:pass@ip:554/onvif2` |

**通道与码流说明**：
- 海康：`/101` = 通道1主码流，`/102` = 通道1子码流，`/201` = 通道2主码流
- 大华：`channel=1` = 通道1，`subtype=0` = 主码流，`subtype=1` = 子码流
- 多通道 IPC：一个摄像头可能有多个镜头（全景 + 特写）

### 5.2 探测摄像头流信息

```bash
# 查看流的编码、分辨率、帧率、码率
ffprobe -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101

# 示例输出解读：
# Stream #0:0: Video: h264 (High), yuv420p, 1920x1080, 25 fps
# Stream #0:1: Audio: aac (LC), 16000 Hz, mono, 64 kb/s
```

### 5.3 拉取摄像头流保存

```bash
# 拉流保存（流复制，无编码开销）
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101 \
  -c copy \
  -f mp4 \
  -movflags +faststart \
  camera_record.mp4

# 注意：直播流保存 MP4 需要正常结束（Ctrl+C）才能写入 moov box
# 更安全的做法：先存 TS/FLV，再转 MP4
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101 \
  -c copy output.flv

# 循环录制（每小时一个文件）
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101 \
  -c copy \
  -f segment \
  -segment_time 3600 \
  -segment_format mp4 \
  -reset_timestamps 1 \
  -strftime 1 \
  "camera_%Y%m%d_%H%M%S.mp4"
```

### 5.4 RTSP 转 RTMP 推流（监控上云）

```bash
# RTSP 摄像头 → RTMP 推流到直播平台
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 1.5M -g 30 -sc_threshold 0 \
  -c:a aac -b:a 64k \
  -f flv \
  rtmp://live.example.com/live/camera01
```

### 5.5 RTSP 转 HLS（Web 可访问）

```bash
# RTSP → HLS（局域网摄像头 → Web 可访问）
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/stream1 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 1.5M -g 30 -sc_threshold 0 \
  -c:a aac -b:a 64k \
  -f hls \
  -hls_time 2 \
  -hls_list_size 10 \
  -hls_flags delete_segments+append_list \
  -hls_segment_filename "/var/hls/cam1/seg_%05d.ts" \
  /var/hls/cam1/index.m3u8
```

---

## 6. 主码流与子码流策略

### 6.1 为什么需要双码流

摄像头通常提供两路码流：
- **主码流**：高分辨率（1080p/4K），高码率（4-8 Mbps），用于**录像存储**
- **子码流**：低分辨率（D1/720p），低码率（512-1024 Kbps），用于**实时预览**

**资源消耗对比**：

| 场景 | 使用码流 | 单路带宽 | 16 路总带宽 | 16 路解码 CPU |
|------|---------|---------|------------|--------------|
| 录像 | 主码流 | 4 Mbps | 64 Mbps | 不解码（直接存） |
| 预览（全部主码流）| 主码流 | 4 Mbps | 64 Mbps | **极高**（16 路 1080p） |
| 预览（全部子码流）| 子码流 | 512 Kbps | 8 Mbps | 低（16 路 D1） |

**典型应用**：
- 监控中心大屏：同时显示 16/32 路，用子码流（否则解码和带宽都扛不住）
- 单路放大查看：切换到主码流（看清车牌号、人脸细节）
- 录像回放：播放主码流录像（保证证据清晰度）

### 6.2 动态切换码流示例

```bash
# 场景：监控中心 16 宫格显示，点击某路放大

# 默认：所有摄像头用子码流
ffplay -rtsp_transport tcp rtsp://192.168.1.101:554/Streaming/Channels/102
ffplay -rtsp_transport tcp rtsp://192.168.1.102:554/Streaming/Channels/102
...

# 用户点击摄像头 #1，切换到主码流
ffplay -rtsp_transport tcp rtsp://192.168.1.101:554/Streaming/Channels/101
```

### 6.3 多路预览优化策略

```bash
# 方案1：用 FFmpeg 将多路子码流合并成一个画面（减少解码压力）
ffmpeg \
  -rtsp_transport tcp -i rtsp://cam1/sub \
  -rtsp_transport tcp -i rtsp://cam2/sub \
  -rtsp_transport tcp -i rtsp://cam3/sub \
  -rtsp_transport tcp -i rtsp://cam4/sub \
  -filter_complex "\
    [0:v]scale=640:360[v0];\
    [1:v]scale=640:360[v1];\
    [2:v]scale=640:360[v2];\
    [3:v]scale=640:360[v3];\
    [v0][v1][v2][v3]xstack=inputs=4:layout=0_0|w0_0|0_h0|w0_h0[out]" \
  -map "[out]" \
  -c:v libx264 -preset veryfast -b:v 2M \
  -f flv rtmp://localhost/live/grid4
```

---
## 7. RTSP 常见问题处理

### 7.1 问题一：连接超时

**现象**：
```
[rtsp @ 0x...] method DESCRIBE failed: 504 Gateway Time-out
rtsp://192.168.1.100:554/stream1: Connection timed out
```

**原因**：
- 摄像头未开机或网络不通
- RTSP 端口被防火墙拦截
- 摄像头响应慢（老设备、高负载）

**解决方案**：

```bash
# 1. 检查网络连通性
ping 192.168.1.100

# 2. 检查 RTSP 端口是否开放
telnet 192.168.1.100 554
# 或
nmap -p 554 192.168.1.100

# 3. 增加超时时间（单位：微秒）
ffmpeg -rtsp_transport tcp \
  -timeout 10000000 \
  -i rtsp://192.168.1.100:554/stream1 ...

# 4. 使用 stimeout（连接超时）和 timeout（读取超时）
ffmpeg -rtsp_transport tcp \
  -stimeout 5000000 \
  -timeout 10000000 \
  -i rtsp://192.168.1.100:554/stream1 ...
```

### 7.2 问题二：音视频不同步

**现象**：
- 画面和声音延迟 1-2 秒
- 声音正常但画面卡顿
- 播放一段时间后逐渐不同步

**原因**：
- 摄像头时间戳不准确（廉价摄像头常见问题）
- RTP 包乱序到达（UDP 模式）
- FFmpeg 未正确解析时间戳

**解决方案**：

```bash
# 方法1：增加分析时长，让 FFmpeg 充分解析流
ffmpeg -rtsp_transport tcp \
  -analyzeduration 10M \
  -probesize 10M \
  -i rtsp://192.168.1.100:554/stream1 ...

# 方法2：重置 PTS（Presentation Time Stamp）
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 \
  -vf "setpts=PTS-STARTPTS" \
  -af "asetpts=PTS-STARTPTS" \
  -c:v libx264 -c:a aac \
  output.mp4

# 方法3：丢弃 B 帧，减少乱序
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 \
  -bsf:v "filter_units=remove_types=6" \
  -c copy output.mp4

# 方法4：强制音视频同步（-async）
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 \
  -async 1 \
  -c:v libx264 -c:a aac \
  output.mp4
```

### 7.3 问题三：流编码格式不兼容

**现象**：
```
[rtsp @ 0x...] Estimating duration from bitrate, this may be inaccurate
Stream #0:0: Video: hevc (Main), yuv420p, 1920x1080
```
部分播放器不支持 H.265（HEVC）。

**解决方案**：

```bash
# H.265 → H.264 转码（兼容性更好）
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 \
  -c:v libx264 -preset veryfast -crf 23 \
  -c:a aac \
  -f flv rtmp://localhost/live/cam1

# 保留 H.265（如果平台支持）
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 \
  -c:v libx265 -preset veryfast -crf 23 \
  -c:a aac \
  output.mp4
```

### 7.4 问题四：UDP 丢包导致花屏/卡顿

**现象**：
- 画面出现马赛克、绿屏、撕裂
- 播放几秒后卡住，然后跳到最新画面
- 无线网络环境下尤其明显

**原因**：
- UDP 传输不可靠，丢包后不重传
- RTP 序列号断层，解码器无法处理

**解决方案**：

```bash
# 方案1：强制 TCP 传输（推荐）
ffmpeg -rtsp_transport tcp \
  -i rtsp://192.168.1.100:554/stream1 ...

# 方案2：增加接收缓冲区
ffmpeg -rtsp_transport udp \
  -buffer_size 4194304 \
  -i rtsp://192.168.1.100:554/stream1 ...

# 方案3：设置最大延迟（丢弃过时的包）
ffmpeg -rtsp_transport udp \
  -max_delay 500000 \
  -i rtsp://192.168.1.100:554/stream1 ...
```

### 7.5 问题五：摄像头拒绝连接（401/403）

**现象**：
```
[rtsp @ 0x...] method DESCRIBE failed: 401 Unauthorized
[rtsp @ 0x...] method DESCRIBE failed: 403 Forbidden
```

**原因**：
- 用户名密码错误
- 摄像头开启了 IP 白名单
- 摄像头同时连接数达到上限

**解决方案**：

```bash
# 1. 确认用户名密码（URL 编码特殊字符）
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:Admin%40123@192.168.1.100:554/stream1 ...
# 密码 Admin@123 需编码为 Admin%40123

# 2. 检查摄像头 Web 界面
# - 安全设置 → IP 访问控制 → 添加客户端 IP
# - 连接数限制 → 增加最大连接数

# 3. 使用摄像头提供的低权限账号（如果有）
ffmpeg -rtsp_transport tcp \
  -i rtsp://viewer:viewer123@192.168.1.100:554/stream1 ...
```

### 7.6 问题六：长时间运行后断流

**现象**：
- FFmpeg 拉流几小时后自动退出
- 日志显示 `Connection reset by peer`

**原因**：
- 摄像头重启或网络抖动
- 摄像头有会话超时限制
- RTSP Keep-Alive 未正确发送

**解决方案**：

```bash
# 方案1：启用自动重连
ffmpeg -rtsp_transport tcp \
  -reconnect 1 \
  -reconnect_streamed 1 \
  -reconnect_delay_max 5 \
  -i rtsp://192.168.1.100:554/stream1 ...

# 方案2：外层 shell 循环重启
while true; do
  ffmpeg -rtsp_transport tcp \
    -i rtsp://192.168.1.100:554/stream1 \
    -c copy output.flv
  echo "连接断开，3秒后重连..."
  sleep 3
done

# 方案3：使用 systemd/supervisor 守护进程自动重启
```

### 7.7 问题七：摄像头分辨率/帧率不符合预期

**现象**：
- 配置了 4K 摄像头，拉到的是 1080p
- 配置了 30fps，实际只有 15fps

**原因**：
- 摄像头 Web 界面配置的是子码流参数
- RTSP URL 拉的是子码流
- 摄像头性能不足（夜间模式、编码能力限制）

**解决方案**：

```bash
# 1. 确认拉的是主码流（不是子码流）
# 海康：/101 是主码流，/102 是子码流
ffprobe -rtsp_transport tcp -i rtsp://admin:pass@192.168.1.100:554/Streaming/Channels/101

# 2. 检查摄像头 Web 界面配置
# 视频编码 → 主码流 → 分辨率/帧率/码率

# 3. 强制指定期望的分辨率（如果摄像头支持协商）
ffmpeg -rtsp_transport tcp \
  -video_size 1920x1080 \
  -framerate 30 \
  -i rtsp://192.168.1.100:554/stream1 ...
```

---

## 8. 多摄像头批量管理

### 8.1 批量探测摄像头

```bash
#!/bin/bash
# probe_cameras.sh - 批量探测摄像头流信息

CAMERAS=(
  "rtsp://admin:pass@192.168.1.101:554/Streaming/Channels/101"
  "rtsp://admin:pass@192.168.1.102:554/Streaming/Channels/101"
  "rtsp://admin:pass@192.168.1.103:554/Streaming/Channels/101"
)

for url in "${CAMERAS[@]}"; do
  echo "==== 检测: $url ===="
  ffprobe -rtsp_transport tcp \
    -timeout 5000000 \
    -i "$url" \
    -show_entries stream=codec_name,width,height,r_frame_rate \
    -of json 2>&1 | grep -E '(codec_name|width|height|r_frame_rate)'
  echo ""
done
```

### 8.2 批量录制

```bash
#!/bin/bash
# record_all.sh - 批量录制所有摄像头

CAMERAS=(
  "cam1:rtsp://admin:pass@192.168.1.101:554/stream1"
  "cam2:rtsp://admin:pass@192.168.1.102:554/stream1"
  "cam3:rtsp://admin:pass@192.168.1.103:554/stream1"
)

OUTPUT_DIR="/var/recording"
mkdir -p "$OUTPUT_DIR"

for entry in "${CAMERAS[@]}"; do
  IFS=":" read -r name url <<< "$entry"
  
  # 后台启动录制进程
  nohup ffmpeg -rtsp_transport tcp \
    -reconnect 1 -reconnect_streamed 1 \
    -i "$url" \
    -c copy \
    -f segment \
    -segment_time 3600 \
    -segment_format mp4 \
    -reset_timestamps 1 \
    -strftime 1 \
    "$OUTPUT_DIR/${name}_%Y%m%d_%H%M%S.mp4" \
    > "/var/log/${name}.log" 2>&1 &
  
  echo "[$name] 录制已启动，PID=$!"
done
```

### 8.3 健康监测脚本

```bash
#!/bin/bash
# health_check.sh - 摄像头健康检测

CAMERAS=(
  "cam1:rtsp://admin:pass@192.168.1.101:554/stream1"
  "cam2:rtsp://admin:pass@192.168.1.102:554/stream1"
)

for entry in "${CAMERAS[@]}"; do
  IFS=":" read -r name url <<< "$entry"
  
  # 尝试连接并获取流信息
  if ffprobe -rtsp_transport tcp \
       -timeout 5000000 \
       -i "$url" \
       -show_entries stream=codec_name \
       2>/dev/null | grep -q "codec_name"; then
    echo "[$(date)] [$name] 正常"
  else
    echo "[$(date)] [$name] 异常 - 发送告警"
    # 这里接入告警系统：钉钉/企业微信/邮件
    # curl -X POST https://oapi.dingtalk.com/robot/send \
    #   -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"摄像头 $name 离线\"}}"
  fi
done
```

### 8.4 定时健康检测（cron）

```bash
# 添加到 crontab（每 5 分钟检测一次）
crontab -e

# 添加以下行
*/5 * * * * /path/to/health_check.sh >> /var/log/camera_health.log 2>&1
```

---

## 9. 监控上云场景

### 9.1 场景一：RTSP 转 RTMP 推流

```bash
# 摄像头 → RTMP 服务器 → CDN 分发
ffmpeg -rtsp_transport tcp \
  -reconnect 1 -reconnect_streamed 1 \
  -i rtsp://admin:pass@192.168.1.100:554/stream1 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2M -g 30 -sc_threshold 0 \
  -c:a aac -b:a 64k \
  -f flv \
  rtmp://live.example.com/live/camera01
```

### 9.2 场景二：RTSP 转 HLS（Web 观看）

```bash
# 摄像头 → HLS → Nginx 静态服务 → 浏览器
ffmpeg -rtsp_transport tcp \
  -reconnect 1 -reconnect_streamed 1 \
  -i rtsp://admin:pass@192.168.1.100:554/stream1 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 1.5M -g 30 -sc_threshold 0 \
  -c:a aac -b:a 64k \
  -f hls \
  -hls_time 4 \
  -hls_list_size 10 \
  -hls_flags delete_segments+append_list \
  -hls_segment_filename "/var/www/hls/cam1/seg_%05d.ts" \
  /var/www/hls/cam1/index.m3u8
```

### 9.3 场景三：多摄像头合并推流

```bash
# 4 个摄像头 → 4 宫格画面 → RTMP 推流
ffmpeg \
  -rtsp_transport tcp -i rtsp://cam1/stream1 \
  -rtsp_transport tcp -i rtsp://cam2/stream1 \
  -rtsp_transport tcp -i rtsp://cam3/stream1 \
  -rtsp_transport tcp -i rtsp://cam4/stream1 \
  -filter_complex "\
    [0:v]scale=640:360[v0];\
    [1:v]scale=640:360[v1];\
    [2:v]scale=640:360[v2];\
    [3:v]scale=640:360[v3];\
    [v0][v1][v2][v3]xstack=inputs=4:layout=0_0|w0_0|0_h0|w0_h0[out]" \
  -map "[out]" \
  -c:v libx264 -preset veryfast -b:v 3M \
  -f flv rtmp://live.example.com/live/grid4
```

### 9.4 场景四：移动侦测录制

```bash
# 仅在检测到画面变化时录制（节省存储）
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100:554/stream1 \
  -vf "select='gt(scene,0.03)',setpts=N/(25*TB)" \
  -c:v libx264 -preset veryfast -crf 23 \
  -f segment \
  -segment_time 60 \
  -reset_timestamps 1 \
  -strftime 1 \
  "motion_%Y%m%d_%H%M%S.mp4"
```

---

## 10. 动手实践

### 实验一：模拟 RTSP 摄像头

```bash
# 用 FFmpeg 创建一个 RTSP 服务器（测试用）
# 需要先安装 FFmpeg 的 rtsp_server 支持

# 方案1：用 mediamtx（原 rtsp-simple-server）
docker run -d --name mediamtx \
  -p 8554:8554 \
  -p 1935:1935 \
  -p 8888:8888 \
  bluenviron/mediamtx

# 推流到 mediamtx（模拟摄像头）
ffmpeg -re -i test.mp4 \
  -c copy \
  -f rtsp \
  rtsp://localhost:8554/mystream

# 拉流测试
ffplay -rtsp_transport tcp rtsp://localhost:8554/mystream

# 方案2：用 FFmpeg 直接推流到 mediamtx
ffmpeg -re -f lavfi -i testsrc2=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=44100 \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 2M \
  -c:a aac -b:a 128k \
  -f rtsp rtsp://localhost:8554/testcam
```

### 实验二：多摄像头健康检测

```bash
# 创建测试脚本
cat > check_cameras.sh << 'SCRIPT'
#!/bin/bash
CAMERAS=(
  "testcam1:rtsp://localhost:8554/cam1"
  "testcam2:rtsp://localhost:8554/cam2"
)

for entry in "${CAMERAS[@]}"; do
  IFS=":" read -r name url <<< "$entry"
  if timeout 5 ffprobe -rtsp_transport tcp -i "$url" 2>/dev/null | grep -q "Stream #0"; then
    echo "[$name] OK"
  else
    echo "[$name] FAIL"
  fi
done
SCRIPT

chmod +x check_cameras.sh
./check_cameras.sh
```

### 实验三：RTSP 转 RTMP 推流

```bash
# 1. 启动 RTMP 服务器
docker run -d --name srs -p 1935:1935 -p 8080:8080 ossrs/srs:5

# 2. 启动 RTSP 模拟服务器
docker run -d --name mediamtx -p 8554:8554 bluenviron/mediamtx

# 3. 推流到 RTSP（模拟摄像头）
ffmpeg -re -i test.mp4 -c copy -f rtsp rtsp://localhost:8554/camera &

# 4. RTSP → RTMP 转推
ffmpeg -rtsp_transport tcp \
  -i rtsp://localhost:8554/camera \
  -c copy \
  -f flv rtmp://localhost:1935/live/cam1 &

# 5. 播放验证
ffplay rtmp://localhost:1935/live/cam1
```

---
## 11. 本章总结

本章覆盖了 RTSP 摄像头接入的完整链路：

| 场景 | 关键技术 | 核心命令/参数 |
|------|----------|---------------|
| RTSP 协议栈 | RTSP/RTP/RTCP 三件套 | RTSP 控制 + RTP 传输数据 |
| 传输模式选择 | TCP vs UDP | `-rtsp_transport tcp`（公网/录像） |
| ONVIF 标准 | 设备互操作 | `rtsp://ip:554/onvif1` |
| GB/T 28181 国标 | 政府/公安强制 | SIP 信令 + PS 封装 |
| 主码流 vs 子码流 | 双码流策略 | 主码流录像，子码流预览 |
| 连接超时 | 增加超时参数 | `-timeout 10000000` |
| 音视频不同步 | 重置时间戳 | `-vf "setpts=PTS-STARTPTS"` |
| UDP 丢包 | 强制 TCP 传输 | `-rtsp_transport tcp` |
| 断流重连 | 自动重连 + shell 循环 | `-reconnect 1 -reconnect_streamed 1` |
| 监控上云 | RTSP → RTMP/HLS | 转码 + 推流 |

**关键原则**：

1. **公网/录像必用 TCP**：`-rtsp_transport tcp` 避免 UDP 丢包，确保录像完整
2. **双码流合理使用**：多路预览用子码流（降低带宽和解码压力），单路放大用主码流（保证细节）
3. **超时参数是必需的**：摄像头响应慢时，默认超时会导致连接失败
4. **断流重连是生产必备**：摄像头可能重启、网络抖动，必须有重连机制
5. **音视频不同步很常见**：廉价摄像头时间戳不准，需要重置 PTS 或增加分析时长

**ONVIF vs GB/T 28181 选型**：

| 场景 | 推荐标准 | 原因 |
|------|---------|------|
| 企业内部监控 | ONVIF | 设备互通，国际标准，简单易用 |
| 小区物业监控 | ONVIF | 无需多级联网，成本低 |
| 政府/公安项目 | GB/T 28181 | **强制要求**，雪亮工程/天网工程 |
| 多级联网（省市区）| GB/T 28181 | 平台级架构，支持级联 |
| 跨厂商设备对接 | ONVIF | 海康摄像头配大华 NVR |

**常见厂商 RTSP 地址速查**：
- 海康主码流：`rtsp://user:pass@ip:554/Streaming/Channels/101`
- 海康子码流：`rtsp://user:pass@ip:554/Streaming/Channels/102`
- 大华主码流：`rtsp://user:pass@ip:554/cam/realmonitor?channel=1&subtype=0`
- 大华子码流：`rtsp://user:pass@ip:554/cam/realmonitor?channel=1&subtype=1`
- ONVIF 通用：`rtsp://user:pass@ip:554/onvif1`

**与其他协议的配合**：
- RTSP 摄像头 → RTMP 推流（监控上云，见第 18 篇）
- RTSP 摄像头 → HLS 输出（Web 观看）
- RTSP 摄像头 → WebRTC 输出（超低延迟互动，需 SRS 等平台支持）

**下一章**：[第 20 篇：直播录制与切片](20-直播录制与切片.md)

深入直播录制策略（按时长/大小/时钟切片）、断点续录、存储管理与自动化归档。

---

## 12. 术语表

| 术语 | 英文全称 | 说明 |
|------|----------|------|
| RTSP | Real Time Streaming Protocol | 实时流控制协议（RFC 2326/7826）|
| RTP | Real-time Transport Protocol | 实时传输协议，承载音视频数据 |
| RTCP | RTP Control Protocol | RTP 控制协议，传输质量反馈 |
| SDP | Session Description Protocol | 会话描述协议（RTSP DESCRIBE 返回）|
| ONVIF | Open Network Video Interface Forum | 网络视频设备互操作标准 |
| GB/T 28181 | 国家标准 28181 | 安全防范视频监控联网系统信息传输标准 |
| IPC | IP Camera | 网络摄像机 |
| NVR | Network Video Recorder | 网络录像机 |
| VMS | Video Management System | 视频管理系统 |
| PTZ | Pan-Tilt-Zoom | 云台摄像机（可转动、变焦）|
| SIP | Session Initiation Protocol | 会话初始化协议（GB/T 28181 使用）|
| PS | Program Stream | 节目流封装（MPEG-2 PS，国标使用）|
| NAL | Network Abstraction Layer | H.264/H.265 的网络抽象层单元 |
| GOP | Group of Pictures | 图像组（关键帧间隔）|
| CBR | Constant Bit Rate | 恒定码率 |
| VBR | Variable Bit Rate | 可变码率 |
| PTS | Presentation Time Stamp | 显示时间戳 |
| DTS | Decoding Time Stamp | 解码时间戳 |

---

## 附录：常见摄像头品牌 RTSP 地址汇总

### A.1 海康威视（Hikvision）

```bash
# 主码流（通道1）
rtsp://admin:password@192.168.1.100:554/Streaming/Channels/101

# 子码流（通道1）
rtsp://admin:password@192.168.1.100:554/Streaming/Channels/102

# 第三码流（通道1，部分型号支持）
rtsp://admin:password@192.168.1.100:554/Streaming/Channels/103

# 通道2主码流（多通道设备）
rtsp://admin:password@192.168.1.100:554/Streaming/Channels/201

# ONVIF 地址
rtsp://admin:password@192.168.1.100:554/onvif1
```

### A.2 大华（Dahua）

```bash
# 主码流（通道1）
rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=1&subtype=0

# 子码流（通道1）
rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=1&subtype=1

# 第三码流（通道1）
rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=1&subtype=2

# 通道2主码流
rtsp://admin:password@192.168.1.100:554/cam/realmonitor?channel=2&subtype=0

# ONVIF 地址
rtsp://admin:password@192.168.1.100:554/onvif1
```

### A.3 宇视（Uniview）

```bash
# 主码流
rtsp://admin:password@192.168.1.100:554/media/video1

# 子码流
rtsp://admin:password@192.168.1.100:554/media/video2

# 第三码流
rtsp://admin:password@192.168.1.100:554/media/video3

# ONVIF 地址
rtsp://admin:password@192.168.1.100:554/onvif1
```

### A.4 雄迈（XMEye）

```bash
# 主码流
rtsp://admin:password@192.168.1.100:554/user=admin&password=&channel=1&stream=0.sdp

# 子码流
rtsp://admin:password@192.168.1.100:554/user=admin&password=&channel=1&stream=1.sdp

# 注意：雄迈的 URL 中 user= 和 password= 参数通常留空
```

### A.5 TP-Link（普联）

```bash
# 主码流
rtsp://admin:password@192.168.1.100:554/stream1

# 子码流
rtsp://admin:password@192.168.1.100:554/stream2
```

### A.6 小米（Xiaomi）

```bash
# 小米摄像头需要先在米家 App 中开启 RTSP
# 开启后在 App 中可以看到 RTSP 地址，格式如下：
rtsp://192.168.1.100:8554/live

# 部分型号
rtsp://192.168.1.100:8554/unicast
```

### A.7 Axis（安讯士）

```bash
# 主码流
rtsp://root:password@192.168.1.100:554/axis-media/media.amp

# 子码流
rtsp://root:password@192.168.1.100:554/axis-media/media.amp?resolution=640x480

# ONVIF 地址
rtsp://root:password@192.168.1.100:554/onvif-media/media.amp
```

### A.8 Foscam

```bash
# 主码流
rtsp://admin:password@192.168.1.100:554/videoMain

# 子码流
rtsp://admin:password@192.168.1.100:554/videoSub

# H.264
rtsp://admin:password@192.168.1.100:554/videoMain?codec=h264

# MJPEG
rtsp://admin:password@192.168.1.100:554/videoMain?codec=mjpeg
```

### A.9 通用探测方法

如果上述地址都不对，可以尝试以下方法：

```bash
# 方法1：ONVIF 标准地址（大部分摄像头支持）
rtsp://admin:password@192.168.1.100:554/onvif1
rtsp://admin:password@192.168.1.100:554/onvif2

# 方法2：常见通用路径
rtsp://admin:password@192.168.1.100:554/live
rtsp://admin:password@192.168.1.100:554/stream1
rtsp://admin:password@192.168.1.100:554/stream2
rtsp://admin:password@192.168.1.100:554/h264
rtsp://admin:password@192.168.1.100:554/mpeg4

# 方法3：用 ONVIF Device Manager 扫描（Windows）
# 下载地址：https://sourceforge.net/projects/onvifdm/

# 方法4：用 nmap 扫描 RTSP 端口
nmap -p 554 192.168.1.100

# 方法5：查看摄像头 Web 界面
# 通常在"网络设置" → "RTSP" 中可以看到完整 URL
```

---

*本文属于 [FFmpeg 完全指南](../README.md) 系列第 19 篇。*
