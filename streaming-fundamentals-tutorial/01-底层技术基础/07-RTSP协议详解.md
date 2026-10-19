# RTSP 协议详解（摄像头拉流标准）

## 目录

- [读完本篇你能做什么](#读完本篇你能做什么)
- [一、RTSP 协议概述](#一rtsp-协议概述)
- [二、RTSP 交互流程](#二rtsp-交互流程)
- [三、SDP 会话描述协议](#三sdp-会话描述协议)
- [四、RTSP 传输模式](#四rtsp-传输模式)
- [五、RTSP 认证机制](#五rtsp-认证机制)
- [六、RTSP vs RTMP vs HTTP](#六rtsp-vs-rtmp-vs-http)
- [七、动手实践](#七动手实践)
- [八、RTSP 状态码与降级策略](#八rtsp-状态码与降级策略)
- [九、常见问题排查](#九常见问题排查)
- [十、本篇小结](#十本篇小结)
- [下一篇](#下一篇)
- [参考资料](#参考资料)

---

## 读完本篇你能做什么

1. **理解 RTSP 工作原理**：掌握 RTSP 请求-响应模型和状态机
2. **解析 SDP 协议**：读懂摄像头返回的 SDP 描述信息
3. **使用 FFmpeg 拉取 RTSP 流**：从 IP 摄像头拉流并保存录像
4. **选择传输模式**：理解 RTP over UDP vs TCP 的区别和适用场景
5. **调试鉴权问题**：处理 RTSP Digest 认证失败
6. **对比 RTSP/RTMP/HLS**：为不同场景选择合适的协议

---

## 一、RTSP 协议概述

### 1.1 什么是 RTSP？

**RTSP**（Real Time Streaming Protocol）是一个应用层协议，用于控制实时流媒体的传输。

**关键特点**：
- **控制协议**：RTSP 本身不传输媒体数据，只负责建立会话、控制播放（播放/暂停/快进）
- **媒体传输**：实际音视频数据通过 **RTP/RTCP** 传输
- **类 HTTP 协议**：使用文本协议，请求-响应模式，端口号 **554**

**典型应用场景**：
- IP 摄像头 (IPC) 拉流
- 视频监控平台 (NVR/VMS)
- 流媒体点播服务器（Darwin Streaming Server）

### 1.2 RTSP URL 格式

```
rtsp://[username:password@]host[:port]/path
```

**示例**：
```
rtsp://192.168.1.100:554/Streaming/Channels/101
rtsp://admin:admin123@192.168.1.100/h264/ch1/main/av_stream
rtsp://example.com/media/video.264
```

### 1.3 RTSP 与 HTTP 的相似之处

| 特性 | HTTP | RTSP |
|------|------|------|
| **协议类型** | 文本协议 | 文本协议 |
| **传输方式** | 请求-响应 | 请求-响应 |
| **状态码** | 200 OK, 404 Not Found | 200 OK, 404 Stream Not Found |
| **头部字段** | Content-Type, User-Agent | CSeq, Session |
| **方法** | GET, POST | DESCRIBE, SETUP, PLAY |

但核心区别是：HTTP 传输数据，RTSP 只控制流媒体会话。

---

## 二、RTSP 交互流程

### 2.1 完整交互流程

```
客户端                                    服务器
  |                                         |
  |  (1) OPTIONS 查询支持的方法              |
  |---------------------------------------->|
  |<----------------------------------------|
  |                200 OK                   |
  |                                         |
  |  (2) DESCRIBE 获取 SDP 描述             |
  |---------------------------------------->|
  |<----------------------------------------|
  |        200 OK + SDP                     |
  |                                         |
  |  (3) SETUP 建立传输通道                 |
  |---------------------------------------->|
  |<----------------------------------------|
  |    200 OK + Session ID                  |
  |                                         |
  |  (4) PLAY 开始播放                      |
  |---------------------------------------->|
  |<----------------------------------------|
  |            200 OK                       |
  |                                         |
  |<======= RTP/RTCP 媒体数据 =============>|
  |                                         |
  |  (5) TEARDOWN 结束会话                  |
  |---------------------------------------->|
  |<----------------------------------------|
  |            200 OK                       |
```

### 2.2 RTSP 方法详解

| 方法 | 说明 | 必需 |
|------|------|------|
| **OPTIONS** | 查询服务器支持的方法 | 可选 |
| **DESCRIBE** | 获取媒体描述（SDP） | 必需 |
| **SETUP** | 建立传输会话，协商端口 | 必需 |
| **PLAY** | 开始发送媒体数据 | 必需 |
| **PAUSE** | 暂停发送（实时流通常不支持） | 可选 |
| **TEARDOWN** | 结束会话，释放资源 | 必需 |
| **GET_PARAMETER** | 保持会话活跃（心跳） | 可选 |
| **SET_PARAMETER** | 设置参数 | 可选 |

### 2.3 DESCRIBE 请求示例

**客户端发送**：
```
DESCRIBE rtsp://192.168.1.100:554/Streaming/Channels/101 RTSP/1.0
CSeq: 2
User-Agent: LibVLC/3.0.0
Accept: application/sdp

```

**服务器响应**：
```
RTSP/1.0 200 OK
CSeq: 2
Content-Type: application/sdp
Content-Length: 452

v=0
o=- 0 0 IN IP4 192.168.1.100
s=RTSP/RTP stream
c=IN IP4 0.0.0.0
t=0 0
a=control:*
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 profile-level-id=420029; packetization-mode=1
a=control:track1
m=audio 0 RTP/AVP 8
a=rtpmap:8 PCMA/8000
a=control:track2
```

### 2.4 SETUP 请求示例

**客户端发送**：
```
SETUP rtsp://192.168.1.100:554/Streaming/Channels/101/track1 RTSP/1.0
CSeq: 3
Transport: RTP/AVP;unicast;client_port=50000-50001

```

**服务器响应**：
```
RTSP/1.0 200 OK
CSeq: 3
Session: 12345678
Transport: RTP/AVP;unicast;client_port=50000-50001;server_port=60000-60001

```

**关键字段**：
- **Session**：会话 ID，后续请求必须携带
- **client_port**：客户端接收 RTP(50000)/RTCP(50001) 的端口
- **server_port**：服务器发送 RTP(60000)/RTCP(60001) 的端口

### 2.5 PLAY 请求示例

**客户端发送**：
```
PLAY rtsp://192.168.1.100:554/Streaming/Channels/101 RTSP/1.0
CSeq: 4
Session: 12345678
Range: npt=0.000-

```

**服务器响应**：
```
RTSP/1.0 200 OK
CSeq: 4
Session: 12345678
Range: npt=0.000-
RTP-Info: url=track1;seq=12345;rtptime=3914950

```

此后服务器开始通过 RTP 发送媒体数据。

### 2.6 RTSP 核心字段深度解析

#### 2.6.1 CSeq 字段

**作用**：匹配请求与响应，类似 TCP 的序列号，确保客户端能正确关联每个响应。

**递增逻辑**：
- 客户端每发送一个新请求，CSeq 值递增 1
- 服务器响应必须携带与请求相同的 CSeq 值
- 即使请求失败（4xx/5xx 错误），CSeq 仍然递增

**错误后果**：
- 响应中 CSeq 不匹配：客户端可能丢弃响应或进入错误状态
- CSeq 重复：可能导致客户端混淆请求-响应关系
- 不递增：某些严格的服务器实现可能拒绝请求

**示例**：
```
C→S: DESCRIBE rtsp://example.com/stream RTSP/1.0
     CSeq: 1

S→C: RTSP/1.0 200 OK
     CSeq: 1

C→S: SETUP rtsp://example.com/stream/track1 RTSP/1.0
     CSeq: 2

S→C: RTSP/1.0 200 OK
     CSeq: 2
```

#### 2.6.2 Range 字段

**作用**：指定播放的时间范围，实现点播视频的跳转和片段播放。

**NPT（Normal Play Time）**：
- 正常播放时间，以秒为单位的媒体时间轴
- `npt=0.000-` 表示从头播放到结束
- `npt=30-60` 表示播放第 30 秒到第 60 秒的片段
- `npt=now-` 表示从当前时刻开始（直播流常用）

**直播流 vs 点播流的差异**：
- **点播流**：支持任意时间跳转，Range 字段有效
  ```
  PLAY rtsp://example.com/video.mp4 RTSP/1.0
  Range: npt=120.5-
  ```
  响应会从视频的第 120.5 秒开始播放。

- **直播流**：通常不支持跳转，Range 字段可能被忽略或返回 `455 Method Not Valid in This State`
  ```
  PLAY rtsp://example.com/live RTSP/1.0
  Range: npt=now-
  ```
  直播流只能从当前时刻播放。

**其他时间格式**：
- `smpte=10:12:33:20-10:13:00:00`（SMPTE 时间码，用于专业视频制作）
- `clock=20260831T120000Z-20260831T130000Z`（绝对 UTC 时间，用于预约录制）

#### 2.6.3 RTP-Info 字段

**作用**：告知客户端首个 RTP 包的序列号（seq）和时间戳（rtptime），用于同步和丢包检测。

**格式**：
```
RTP-Info: url=track1;seq=12345;rtptime=3914950,url=track2;seq=54321;rtptime=1234567890
```

**字段说明**：
- **url**：对应哪个媒体轨道（通常是 SETUP 时的 track ID）
- **seq**：首个 RTP 包的序列号（16 位，从随机值开始，每包递增 1）
- **rtptime**：首个 RTP 包的 RTP 时间戳（32 位，单位由编码时钟频率决定）

**同步用法**：
1. **丢包检测**：客户端收到 RTP 包后，检查 seq 是否连续。如果 seq 跳跃（如 12345→12347），说明中间丢了一个包。
2. **音视频同步**：
   - 视频时钟频率通常是 90kHz（H.264/H.265）
   - 音频时钟频率通常是采样率（如 PCMA 8kHz，AAC 48kHz）
   - 通过 rtptime 和时钟频率换算成实际时间，对齐音视频的播放时刻

**示例**：
```
RTP-Info: url=track1;seq=12345;rtptime=3914950
```
- 视频第一个 RTP 包的序列号是 12345
- RTP 时间戳是 3914950（假设视频是 90kHz 时钟，实际时间 = 3914950 / 90000 ≈ 43.5 秒）

**多轨道场景**：
```
RTP-Info: url=track1;seq=10000;rtptime=0,url=track2;seq=20000;rtptime=0
```
- track1（视频）和 track2（音频）分别有独立的 seq 和 rtptime
- 客户端需要根据各自的时钟频率换算后对齐播放

### 2.7 RTCP 协议与 QoS 反馈

**RTCP**（RTP Control Protocol）是 RTP 的控制协议，专门用于传输质量监控和会话管理。

**为什么端口号是 RTP + 1**：
- RTP 数据包用偶数端口（如 50000）
- RTCP 控制包用下一个奇数端口（如 50001）
- 这是 RFC 3550 的约定，让防火墙和 NAT 设备能同时放行两个端口

**RTCP 包的四种类型**：

| 类型 | 全称 | 作用 |
|------|------|------|
| **SR** | Sender Report | 发送端报告：已发送的包数、字节数、时间戳 |
| **RR** | Receiver Report | 接收端报告：丢包率、抖动、延迟 |
| **SDES** | Source Description | 源描述：发送端的 CNAME（用于多流同步） |
| **BYE** | Goodbye | 会话结束通知 |

**实际应用场景**：

**场景 1：根据丢包率调整码率**
```
接收端通过 RTCP RR 上报丢包率 15%
     ↓
发送端（摄像头）检测到网络拥塞
     ↓
动态降低视频码率：2Mbps → 1Mbps
     ↓
下一个 RTCP RR 显示丢包率降至 2%
```

**场景 2：音视频同步**
- RTCP SR 包含 NTP 时间戳和 RTP 时间戳的映射关系
- 客户端用此映射将音视频的 RTP 时间戳对齐到同一时间轴
- 例如：视频 RTP timestamp=3600000（90kHz），音频 RTP timestamp=48000（48kHz），通过 NTP 对齐后可判断两者是否同步

**RTCP 发送频率**：
- RTCP 不能占用过多带宽，RFC 规定 RTCP 流量不超过总流量的 5%
- 典型间隔：5-10 秒发送一次 RTCP 包

### 2.8 直播流与点播流的差异

摄像头直播流和录像文件点播流在协议层面有本质区别，理解这些差异能帮助你正确处理不同场景。

**特性对比表**：

| 特性 | 直播流（Live） | 点播流（VOD） |
|------|----------------|---------------|
| **PAUSE 支持** | 不支持（返回 455 错误） | 支持 |
| **Range 跳转** | 不支持或只支持 `npt=now-` | 支持任意时间点 |
| **PLAY 起点** | 只能从当前时刻开始 | 可指定任意起点 |
| **SDP 标识** | `a=range:npt=now-` 或无 range | `a=range:npt=0-123.45` |
| **典型用途** | 监控摄像头、会议直播 | 录像回放、视频点播 |

**如何从 SDP 判断流类型**：

```sdp
# 直播流 SDP
a=range:npt=now-   # 只能从当前时刻播放，或完全没有 range 字段

# 点播流 SDP
a=range:npt=0-3600.0   # 有明确的时间范围（0 到 3600 秒）
```

**实际例子**：

**例子 1：摄像头直播**
```bash
# 尝试暂停直播流
ffmpeg -i "rtsp://192.168.1.100/live" -c copy -t 10 out.mp4
# 按 Ctrl+C 暂停
# ❌ 摄像头不会暂停，继续推流，再次拉流时已经跳过了中间的内容
```

**例子 2：录像回放**
```bash
# 从录像的第 5 分钟开始播放
PLAY rtsp://192.168.1.100/playback/record1.mp4 RTSP/1.0
Range: npt=300-
# ✅ 服务器从 300 秒位置开始发送数据
```

---

## 三、SDP 会话描述协议

### 3.1 SDP 是什么？

**SDP**（Session Description Protocol）用于描述媒体会话的参数，DESCRIBE 响应中携带。

**SDP 行格式**：每行都是 `type=value`。

### 3.2 SDP 结构解析

```sdp
v=0                                   # 版本号
o=- 0 0 IN IP4 192.168.1.100          # 会话发起方信息
s=RTSP/RTP stream                     # 会话名称
c=IN IP4 0.0.0.0                      # 连接信息
t=0 0                                 # 会话时间（0 0 = 无时间限制）
a=control:*                           # 控制 URL（* 表示整个 URL）

# 视频媒体描述
m=video 0 RTP/AVP 96                  # 媒体类型、端口、协议、PT
a=rtpmap:96 H264/90000                # PT=96 对应 H.264，时钟90kHz
a=fmtp:96 profile-level-id=42e01f; sprop-parameter-sets=Z0IAHqtAUB7bA==,aM48gA==
a=control:track1                      # 此轨道的控制 URL

# 音频媒体描述
m=audio 0 RTP/AVP 8                   # PT=8 对应 PCMA
a=rtpmap:8 PCMA/8000                  # A-law 编码，8kHz
a=control:track2
```

### 3.3 SDP 关键字段说明

| 字段 | 示例 | 说明 |
|------|------|------|
| `v=` | `v=0` | SDP 版本，固定为 0 |
| `m=` | `m=video 0 RTP/AVP 96` | 媒体类型：video/audio，端口，PT |
| `a=rtpmap:` | `a=rtpmap:96 H264/90000` | PT 到编码格式的映射 |
| `a=fmtp:` | `a=fmtp:96 profile-level-id=42e01f` | 编码格式参数 |
| `a=control:` | `a=control:track1` | 该媒体流的控制 URL（SETUP 时使用） |

### 3.4 RTP 时钟频率与时间戳换算

**时钟频率的含义**：

在 SDP 中看到的 `a=rtpmap:96 H264/90000`，其中 90000 是 **RTP 时钟频率**（单位：Hz），表示 RTP 时间戳每秒递增 90000。

**为什么视频常用 90kHz**：
- 这是历史遗留约定，来自 MPEG-2 标准（90kHz = 27MHz / 300）
- H.264、H.265 沿用了这个频率
- 90kHz 能被常见帧率整除（如 30fps: 90000/30=3000，每帧增 3000）

**音频 vs 视频的时钟差异**：

| 媒体类型 | 时钟频率 | 示例 |
|----------|----------|------|
| **视频** | 固定 90kHz | H.264/90000, H.265/90000 |
| **音频** | 等于采样率 | PCMA/8000, AAC/48000 |

**RTP 时间戳到真实时间的换算**：

```
真实时间（秒） = RTP timestamp / clock_rate
```

**实际例子**：

```sdp
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
```

假设收到一个 RTP 包，其 timestamp = 5400000：
```
真实时间 = 5400000 / 90000 = 60 秒
```
这个包对应视频的第 60 秒位置。

**音视频时间戳对齐示例**：

```
视频：RTP timestamp=2700000, clock=90kHz → 30 秒
音频：RTP timestamp=240000, clock=8kHz  → 30 秒
```

通过换算后发现两者都在 30 秒位置，说明音视频同步。

**为什么需要 RTCP SR 中的 NTP 时间戳**：

- RTP timestamp 是相对值（从随机起点开始）
- RTCP SR 提供 NTP 绝对时间 ↔ RTP timestamp 的映射
- 有了这个映射，客户端才能将不同轨道（不同时钟频率）对齐到同一时间轴

### 3.5 sprop-parameter-sets 解析

H.264 SDP 中的 `sprop-parameter-sets` 包含两个 Base64 编码的 NALU：
- **SPS**（Sequence Parameter Set）：分辨率、帧率、Profile 等
- **PPS**（Picture Parameter Set）：量化参数等

**解码示例**：
```python
import base64
sps = base64.b64decode("Z0IAHqtAUB7bA==")
pps = base64.b64decode("aM48gA==")
# 在 NALU 前加 0x00000001 起始码即可送入解码器
```

---

## 四、RTSP 传输模式

### 4.1 UDP 传输（默认）

```
SETUP .../track1 RTSP/1.0
Transport: RTP/AVP;unicast;client_port=50000-50001
```

**特点**：
- RTP 数据包通过 UDP 发送
- 延迟低（无需等待重传）
- 容易被防火墙/NAT 阻拦

**适用场景**：局域网内监控（摄像头与服务器在同一网段）

### 4.2 TCP 传输（交织模式）

```
SETUP .../track1 RTSP/1.0
Transport: RTP/AVP/TCP;unicast;interleaved=0-1
```

**RTP over TCP 封装格式**（交织帧）：
```
+------+--------+-------+----------+
| $ | channel | length | RTP data |
+------+--------+-------+----------+
  1B      1B      2B     length B
```
- `$`：固定标识符
- `channel`：0=RTP, 1=RTCP（视频）；2=RTP, 3=RTCP（音频）
- `length`：RTP 数据长度

**特点**：
- RTP 数据通过 RTSP 的 TCP 连接传输
- 穿透防火墙能力强
- 延迟略高（TCP 重传机制）

**适用场景**：公网访问、有防火墙/NAT 的环境

### 4.3 多播传输

```
SETUP .../track1 RTSP/1.0
Transport: RTP/AVP;multicast;destination=239.255.1.1;port=5000-5001
```

**适用场景**：大规模单向广播（如 IPTV）

### 4.4 传输模式选择策略

```
                     局域网直连
                         |
                    防火墙/NAT？
                    /         \
                  否             是
                  |              |
              UDP（低延迟）   TCP 交织（穿透性强）
```

### 4.5 多轨道音视频处理

一个 RTSP 流通常同时包含视频和音频，需要分别为每个轨道建立传输通道。

**双轨 SETUP 流程**：

```
1. DESCRIBE 获取 SDP
   → 解析出 track1（视频）和 track2（音频）

2. 第一次 SETUP（视频轨）
   C→S: SETUP rtsp://.../track1 RTSP/1.0
        Transport: RTP/AVP;unicast;client_port=50000-50001
   
   S→C: RTSP/1.0 200 OK
        Session: ABC123
        Transport: RTP/AVP;unicast;client_port=50000-50001;server_port=60000-60001

3. 第二次 SETUP（音频轨）
   C→S: SETUP rtsp://.../track2 RTSP/1.0
        Session: ABC123   # ← 必须携带第一次获得的 Session ID
        Transport: RTP/AVP;unicast;client_port=50002-50003
   
   S→C: RTSP/1.0 200 OK
        Session: ABC123
        Transport: RTP/AVP;unicast;client_port=50002-50003;server_port=60002-60003

4. PLAY 开始播放（使用同一个 Session ID）
   C→S: PLAY rtsp://.../ RTSP/1.0
        Session: ABC123
   
   S→C: RTSP/1.0 200 OK
        RTP-Info: url=track1;seq=10000;rtptime=0,url=track2;seq=20000;rtptime=0
```

**关键点**：
- **Session ID 共享**：第一次 SETUP 获得 Session ID，后续 SETUP 和 PLAY 必须携带
- **独立端口**：每个轨道占用不同的端口对（视频 50000-50001，音频 50002-50003）
- **独立 seq/rtptime**：RTP-Info 中每个轨道有各自的序列号和时间戳起点

**音视频同步方法**：

1. 从 SDP 获取各自的时钟频率（视频 90kHz，音频 8kHz）
2. 从 RTP-Info 获取各自的起始 rtptime
3. 收到 RTP 包后，用 `(current_rtptime - start_rtptime) / clock_rate` 换算成秒
4. 对齐两个轨道的播放时间

**伪代码示例**：

```go
// 解析 SDP
videoTrack := parseSDP(sdp, "video")  // track1, clock=90000
audioTrack := parseSDP(sdp, "audio")  // track2, clock=8000

// 分别 SETUP
sessionID := setupTrack(videoTrack, "50000-50001")
setupTrack(audioTrack, "50002-50003", sessionID)  // 传入已有的 sessionID

// PLAY 并解析 RTP-Info
rtpInfo := play(sessionID)
videoStartTime := rtpInfo["track1"].rtptime  // 0
audioStartTime := rtpInfo["track2"].rtptime  // 0

// 音视频同步逻辑
for {
    videoPkt := receiveRTP(videoTrack)
    audioPkt := receiveRTP(audioTrack)
    
    videoTime := (videoPkt.timestamp - videoStartTime) / 90000.0  // 秒
    audioTime := (audioPkt.timestamp - audioStartTime) / 8000.0   // 秒
    
    // 如果时间差 > 阈值，丢弃快的包或等待慢的包
    if abs(videoTime - audioTime) > 0.1 {
        // 同步逻辑
    }
}
```

---

## 五、RTSP 认证机制

### 5.1 Basic 认证

**请求失败时服务器返回**：
```
RTSP/1.0 401 Unauthorized
WWW-Authenticate: Basic realm="RTSP Server"
```

**客户端重新发送（Base64 编码 username:password）**：
```
DESCRIBE rtsp://... RTSP/1.0
CSeq: 2
Authorization: Basic YWRtaW46YWRtaW4xMjM=
```

**安全问题**：明文传输，容易被中间人截获，不建议在公网使用。

### 5.2 Digest 认证（更安全）

**服务器返回 nonce**：
```
RTSP/1.0 401 Unauthorized
WWW-Authenticate: Digest realm="RTSP Server", nonce="abc123xyz"
```

**客户端计算响应**：
```python
import hashlib

def digest_auth(username, password, method, uri, realm, nonce):
    ha1 = hashlib.md5(f"{username}:{realm}:{password}".encode()).hexdigest()
    ha2 = hashlib.md5(f"{method}:{uri}".encode()).hexdigest()
    response = hashlib.md5(f"{ha1}:{nonce}:{ha2}".encode()).hexdigest()
    return response
```

**客户端发送**：
```
DESCRIBE rtsp://... RTSP/1.0
CSeq: 2
Authorization: Digest username="admin", realm="RTSP Server",
               nonce="abc123xyz", uri="/Streaming/Channels/101",
               response="d41d8cd98f00b204e9800998ecf8427e"
```

---

## 六、RTSP vs RTMP vs HTTP

### 6.1 协议特性对比

| 维度 | RTSP | RTMP | HTTP（HLS/DASH） |
|------|------|------|-----------------|
| **端口** | 554 | 1935 | 80/443 |
| **传输协议** | RTP over UDP/TCP | TCP | HTTP/HTTPS |
| **延迟** | 1-3 秒 | 1-3 秒 | 10-30 秒（HLS）|
| **防火墙穿透** | 差（UDP）/好（TCP） | 好 | 极好 |
| **点播支持** | 支持（较少用） | 不支持 | 原生支持 |
| **CDN 支持** | 差 | 差 | 极好 |
| **浏览器直播** | 不支持 | Flash 已废弃 | 原生支持 |
| **主要用途** | 摄像头拉流 | 直播推流 | CDN 分发 |

### 6.2 使用场景选择

```
监控摄像头
    |
    v
  RTSP 拉流（行业标准）
    |
    v
  流媒体服务器
  /           \
RTMP 推流    HLS/DASH 封装
（内部传输）  （CDN 分发）
    |
    v
  CDN
    |
    v
  浏览器/App 播放
```

---

## 七、动手实践

### 实践 1：FFmpeg 拉取 RTSP 流

```bash
# 1. 测试 RTSP 流是否可访问
ffprobe -v quiet -print_format json -show_streams \
  "rtsp://admin:admin123@192.168.1.100/Streaming/Channels/101"

# 2. 实时预览（UDP 传输）
ffplay -rtsp_transport udp "rtsp://192.168.1.100:554/stream"

# 3. 实时预览（TCP 传输，推荐用于公网）
ffplay -rtsp_transport tcp "rtsp://192.168.1.100:554/stream"

# 4. 拉流并转换为 HLS
ffmpeg -rtsp_transport tcp \
  -i "rtsp://admin:admin123@192.168.1.100/stream" \
  -c:v copy -c:a aac \
  -f hls -hls_time 2 -hls_list_size 5 \
  /var/www/live/stream.m3u8

# 5. 拉流并录制为 MP4（1 分钟）
ffmpeg -rtsp_transport tcp \
  -i "rtsp://192.168.1.100/stream" \
  -c copy -t 60 output.mp4
```

### 实践 2：Go 解析 RTSP 流（使用 gortsplib）

```go
package main

import (
    "log"
    "github.com/bluenviron/gortsplib/v4"
    "github.com/bluenviron/gortsplib/v4/pkg/base"
    "github.com/bluenviron/gortsplib/v4/pkg/format"
    "github.com/pion/rtp"
)

func main() {
    c := gortsplib.Client{}

    // 解析 URL
    u, _ := base.ParseURL("rtsp://192.168.1.100:554/stream")

    // 连接服务器
    err := c.Start(u.Scheme, u.Host)
    if err != nil {
        log.Fatal(err)
    }
    defer c.Close()

    // 获取 SDP 描述
    desc, _, err := c.Describe(u)
    if err != nil {
        log.Fatal(err)
    }

    // 找到 H264 轨道
    var h264Format *format.H264
    medi := desc.FindFormat(&h264Format)
    if medi == nil {
        log.Fatal("未找到 H264 轨道")
    }

    // 处理 RTP 包
    c.OnPacketRTP(medi, h264Format, func(pkt *rtp.Packet) {
        log.Printf("收到 RTP 包: seq=%d, ts=%d, len=%d", pkt.SequenceNumber, pkt.Timestamp, len(pkt.Payload))
    })

    // 开始播放
    _, err = c.Play(nil)
    if err != nil {
        log.Fatal(err)
    }

    // 等待
    panic(c.Wait())
}
```

### 实践 3：检测摄像头 RTSP URL

很多国产摄像头使用固定 URL 格式：
```bash
# 海康威视
rtsp://[ip]:554/Streaming/Channels/[ch]01  # 主码流
rtsp://[ip]:554/Streaming/Channels/[ch]02  # 子码流

# 大华
rtsp://[ip]:554/cam/realmonitor?channel=[ch]&subtype=0  # 主码流
rtsp://[ip]:554/cam/realmonitor?channel=[ch]&subtype=1  # 子码流

# 宇视
rtsp://[ip]:554/video1   # 主码流
rtsp://[ip]:554/video2   # 子码流

# ONVIF 标准（厂商无关）
rtsp://[ip]:554/onvif/device_service
```

### 实践 4：从 RTP 包提取 H.264 NALU

RTP Payload 中携带的 H.264 数据有两种模式：单 NALU 模式和 FU-A 分片模式。

**RTP Payload 结构**：

```
单 NALU 模式（小帧）：
+----------+-------------+
| RTP 头   | 完整 NALU   |
| 12 字节  | ...         |
+----------+-------------+

FU-A 分片模式（大帧）：
+----------+----------+----------+--------------+
| RTP 头   | FU 指示符 | FU 头    | NALU 片段    |
| 12 字节  | 1 字节    | 1 字节   | ...          |
+----------+----------+----------+--------------+
```

**NALU 类型判断**（取第一个字节的低 5 位）：

| type & 0x1F | 类型 | 说明 |
|-------------|------|------|
| 7 | SPS | Sequence Parameter Set（分辨率等） |
| 8 | PPS | Picture Parameter Set（量化参数） |
| 5 | IDR | 关键帧 |
| 1 | P 帧 | 非关键帧 |
| 28 | FU-A | 分片包（需组装） |

**Go 代码示例**：

```go
package main

import (
    "encoding/binary"
    "fmt"
)

func parseRTPPacket(rtpData []byte) {
    if len(rtpData) < 12 {
        return
    }

    // 解析 RTP 头
    seq := binary.BigEndian.Uint16(rtpData[2:4])
    timestamp := binary.BigEndian.Uint32(rtpData[4:8])
    payload := rtpData[12:]

    if len(payload) == 0 {
        return
    }

    naluType := payload[0] & 0x1F

    if naluType == 28 { // FU-A 分片
        if len(payload) < 2 {
            return
        }
        fuHeader := payload[1]
        start := (fuHeader & 0x80) != 0  // S bit
        end := (fuHeader & 0x40) != 0    // E bit
        fragmentType := fuHeader & 0x1F

        if start {
            fmt.Printf("分片起始: seq=%d, ts=%d, type=%d\n", seq, timestamp, fragmentType)
        }
        if end {
            fmt.Printf("分片结束: seq=%d\n", seq)
        }
    } else { // 单 NALU
        fmt.Printf("完整NALU: seq=%d, ts=%d, type=%d, len=%d\n", seq, timestamp, naluType, len(payload))
        
        switch naluType {
        case 7:
            fmt.Println("  → SPS")
        case 8:
            fmt.Println("  → PPS")
        case 5:
            fmt.Println("  → IDR 关键帧")
        case 1:
            fmt.Println("  → P 帧")
        }
    }
}
```

**输出 NALU 到文件（可用 ffplay 播放）**：

```go
// 加 H.264 起始码 0x00000001
func writeNALU(file *os.File, nalu []byte) {
    startCode := []byte{0x00, 0x00, 0x00, 0x01}
    file.Write(startCode)
    file.Write(nalu)
}

// 使用示例
f, _ := os.Create("output.h264")
defer f.Close()

// SPS/PPS 必须先写入（从 SDP 的 sprop-parameter-sets 获取）
writeNALU(f, spsData)
writeNALU(f, ppsData)

// 写入视频帧 NALU
for _, rtpPkt := range rtpPackets {
    nalu := extractNALU(rtpPkt)  // 处理单 NALU 或组装 FU-A
    writeNALU(f, nalu)
}

// 播放：ffplay -f h264 output.h264
```

---

## 八、RTSP 状态码与降级策略

### 8.1 常见 RTSP 状态码

| 状态码 | 含义 | 常见原因 |
|--------|------|----------|
| **200 OK** | 请求成功 | - |
| **401 Unauthorized** | 需要认证 | 用户名密码错误或未提供 |
| **404 Stream Not Found** | 流不存在 | URL 路径错误 |
| **455 Method Not Valid in This State** | 方法在当前状态无效 | 直播流不支持 PAUSE、未 SETUP 就 PLAY |
| **461 Unsupported Transport** | 不支持的传输方式 | 服务器不支持 TCP 交织模式或多播 |
| **500 Internal Server Error** | 服务器内部错误 | 摄像头固件 bug、资源不足 |

### 8.2 客户端降级策略

**策略 1：传输模式降级（UDP → TCP）**

```go
func connectRTSP(url string) error {
    // 第一次尝试：UDP 传输（低延迟）
    err := setupTransport(url, "RTP/AVP;unicast;client_port=50000-50001")
    if err != nil && isTransportError(err) {
        // 降级：改用 TCP 交织模式
        return setupTransport(url, "RTP/AVP/TCP;unicast;interleaved=0-1")
    }
    return err
}
```

**策略 2：认证方式降级（Digest → Basic）**

```go
func authenticate(url string) error {
    // 第一次尝试：Digest 认证
    resp, err := describeWithDigest(url, username, password)
    if err != nil && isAuthError(err) {
        // 降级：Basic 认证（部分老摄像头只支持 Basic）
        return describeWithBasic(url, username, password)
    }
    return err
}
```

**策略 3：多路径尝试**

```go
// 海康摄像头可能有多个 URL 格式
urls := []string{
    "rtsp://ip/Streaming/Channels/101",
    "rtsp://ip/h264/ch1/main/av_stream",
    "rtsp://ip/onvif/device_service",
}

for _, url := range urls {
    if err := tryConnect(url); err == nil {
        return url, nil
    }
}
```

**伪代码：完整的健壮连接逻辑**

```
1. 尝试 UDP + Digest
     ↓ 失败（461 Unsupported Transport）
2. 尝试 TCP + Digest
     ↓ 失败（401 认证错误）
3. 尝试 TCP + Basic
     ↓ 失败（404 Stream Not Found）
4. 尝试备用 URL
     ↓ 成功
5. 连接成功
```

### 8.3 错误处理最佳实践

**实践 1：记录失败的方法，避免重复尝试**

```go
type RTSPClient struct {
    failedTransports map[string]bool  // "udp", "tcp"
}

func (c *RTSPClient) Setup(url string) error {
    if !c.failedTransports["udp"] {
        err := c.setupUDP(url)
        if err == nil {
            return nil
        }
        c.failedTransports["udp"] = true
    }
    
    // 直接尝试 TCP
    return c.setupTCP(url)
}
```

**实践 2：超时重试（处理网络抖动）**

```go
func retryConnect(url string, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := connect(url)
        if err == nil {
            return nil
        }
        
        if isTemporaryError(err) {  // 网络超时、连接重置
            time.Sleep(time.Second * time.Duration(i+1))  // 指数退避
            continue
        }
        
        return err  // 非临时错误（401、404）立即返回
    }
    return fmt.Errorf("max retries reached")
}
```

---

## 九、常见问题排查

### 问题 1：RTSP 连接超时

**排查步骤**：
1. 确认摄像头 IP 可达：`ping 192.168.1.100`
2. 确认 554 端口开放：`nc -zv 192.168.1.100 554`
3. 检查防火墙：`iptables -L -n | grep 554`
4. 尝试 TCP 模式：`ffplay -rtsp_transport tcp rtsp://...`

### 问题 2：视频卡顿/丢帧（公网场景）

**原因**：UDP 包在公网路由中丢失。

**解决方案**：
```bash
# 强制使用 TCP 传输
ffplay -rtsp_transport tcp rtsp://...
```

### 问题 3：401 Digest 认证失败

**排查**：
1. 确认用户名密码正确
2. 检查摄像头认证方式：部分摄像头只支持 Basic
3. 强制指定认证方式：
```bash
ffplay -rtsp_transport tcp -allowed_media_types video \
  "rtsp://admin:admin123@192.168.1.100/stream"
```

### 问题 4：NAT 穿透失败（UDP 模式）

**原因**：服务器发送 RTP 到客户端端口，但 NAT 设备没有对应映射。

**解决方案**：
- 在服务器/摄像头端映射端口
- 改用 TCP 交织模式（推荐）
- 部署在同一局域网

---

## 十、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **RTSP 角色** | 控制协议，本身不传数据；数据由 RTP 传输 |
| **交互流程** | OPTIONS → DESCRIBE → SETUP → PLAY → TEARDOWN |
| **SDP** | 描述媒体格式、编码参数、控制 URL |
| **传输模式** | UDP（局域网，低延迟）vs TCP（公网，穿透性强） |
| **认证** | Basic（明文）/ Digest（MD5 摘要，更安全） |
| **定位** | 摄像头拉流标准；不适合 CDN 分发和浏览器直播 |

**快速记忆口诀**：
- RTSP = 遥控器（控制），RTP = 信号线（传数据）
- 局域网用 UDP，公网用 TCP
- 摄像头用 RTSP，直播推流用 RTMP，CDN 用 HLS

---

## 下一篇

下一篇将深入讲解 **RTMP 协议**：直播推流的事实标准，Chunk 流设计原理，为什么 OBS 默认用 RTMP，以及 RTMP 在 Flash 消亡后的新生（RTMPS/rtmp-sharp）。

---

## 参考资料

1. [RFC 2326 - Real Time Streaming Protocol (RTSP)](https://datatracker.ietf.org/doc/html/rfc2326)
2. [RFC 4566 - SDP: Session Description Protocol](https://datatracker.ietf.org/doc/html/rfc4566)
3. [gortsplib - Go RTSP 库](https://github.com/bluenviron/gortsplib)
4. [FFmpeg RTSP 文档](https://ffmpeg.org/ffmpeg-protocols.html#rtsp)
5. [海康威视 RTSP URL 规范](https://www.hikvision.com/content/dam/hikvision/en/support/FAQ/3-RTSP.pdf)

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| RTSP | Real Time Streaming Protocol | 实时流媒体协议，RFC 2326/7826，控制音视频流 |
| SDP | Session Description Protocol | 会话描述协议，描述媒体流参数（编码/地址/端口）|
| CSeq | Command Sequence | RTSP 请求序列号，用于匹配请求和响应 |
| DESCRIBE | DESCRIBE | RTSP 方法：获取媒体资源的 SDP 描述 |
| SETUP | SETUP | RTSP 方法：建立传输通道，协商 RTP 端口 |
| PLAY | PLAY | RTSP 方法：开始或恢复媒体流播放 |
| TEARDOWN | TEARDOWN | RTSP 方法：释放会话资源，关闭连接 |
| Interleaved | Interleaved Mode | 交织模式：RTP 数据通过 TCP 连接传输（穿透 NAT）|
| Session | RTSP Session | RTSP 会话 ID，维持连接状态 |
| 拉流 | Pull | 客户端主动向服务器请求媒体流 |
