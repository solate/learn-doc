# RTP/RTCP 实时传输协议详解

## 目录

- [读完本篇你能做什么](#读完本篇你能做什么)
- [一、RTP 协议核心结构](#一rtp-协议核心结构)
- [二、RTCP 反馈机制](#二rtcp-反馈机制)
- [三、时序同步与 RTT 计算](#三时序同步与-rtt-计算)
- [四、Jitter Buffer 原理](#四jitter-buffer-原理)
- [五、RTP 扩展头与多路复用](#五rtp-扩展头与多路复用)
- [六、WebRTC 中的 RTP 实践](#六webrtc-中的-rtp-实践)
- [七、动手实践](#七动手实践)
- [八、常见问题排查](#八常见问题排查)
- [九、本篇小结](#九本篇小结)
- [下一篇](#下一篇)
- [参考资料](#参考资料)

---

## 读完本篇你能做什么

1. **理解 RTP 数据包结构**：掌握序列号、时间戳、SSRC 等字段含义
2. **解析 RTCP 反馈报文**：读懂 SR/RR/NACK/PLI 等反馈包
3. **实现 RTT 计算**：通过 RTCP SR/RR 的 NTP 时间戳计算往返时延
4. **设计 Jitter Buffer**：理解延迟抖动缓冲区的工作原理
5. **调试 WebRTC 丢包**：通过 chrome://webrtc-internals 分析 RTP 统计信息
6. **优化传输策略**：根据 RTCP 反馈调整码率或触发关键帧

---

## 一、RTP 协议核心结构

### 1.1 为什么需要 RTP？

UDP 提供了低延迟传输，但缺少：
- **时序信息**：音视频帧需要按时间戳播放
- **同步机制**：音视频流需要对齐
- **丢包检测**：需要序列号判断是否丢包
- **流标识**：多路复用时需要区分不同流

RTP（Real-time Transport Protocol）在 UDP 之上增加了这些能力，是音视频实时传输的事实标准。

### 1.2 RTP Header 结构（12 字节固定头）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           SSRC (Synchronization Source)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            CSRC[0..15] (Contributing Source, 可选)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段解析**：

| 字段 | 长度 | 说明 |
|------|------|------|
| **V (Version)** | 2 bits | 固定为 2 |
| **P (Padding)** | 1 bit | 是否填充（最后一个字节标识填充长度） |
| **X (Extension)** | 1 bit | 是否有扩展头 |
| **CC (CSRC Count)** | 4 bits | CSRC 数量（0-15） |
| **M (Marker)** | 1 bit | 标记位（视频帧结束、音频静音开始等） |
| **PT (Payload Type)** | 7 bits | 负载类型（96-127 为动态编码） |
| **Sequence Number** | 16 bits | 序列号（每包 +1，用于检测丢包和乱序） |
| **Timestamp** | 32 bits | 时间戳（采样时钟，非真实时间） |
| **SSRC** | 32 bits | 流标识（同一流的所有 RTP 包 SSRC 相同） |
| **CSRC** | 可变 | 混音场景下的贡献源列表 |

### 1.3 Payload Type 常见值

| PT | 编码 | 采样率 | 说明 |
|----|------|--------|------|
| 0 | PCMU | 8 kHz | u-law 音频 |
| 8 | PCMA | 8 kHz | A-law 音频 |
| 96-127 | 动态 | 动态 | H.264/Opus/VP8 等需通过 SDP 协商 |

**示例：WebRTC SDP 中的 PT 映射**
```sdp
m=video 9 UDP/TLS/RTP/SAVPF 96 97
a=rtpmap:96 VP8/90000
a=rtpmap:97 H264/90000
a=fmtp:97 profile-level-id=42e01f
```

这里 PT=96 对应 VP8，PT=97 对应 H.264，具体对应关系由 SDP 协商决定。

### 1.4 Timestamp 的采样时钟

**不同编码的时间戳单位**：
- **音频（Opus）**：48 kHz 时钟，每秒 +48000
- **视频（H.264/VP8）**：90 kHz 时钟，每秒 +90000

**示例：30fps 视频的时间戳增量**
```
帧间隔 = 1/30 秒 = 0.0333 秒
时间戳增量 = 90000 × 0.0333 = 3000
```

### 1.5 序列号回绕问题

Sequence Number 是 16 位无符号整数，范围 0-65535，达到 65535 后会回绕到 0。

**丢包检测逻辑**：
```go
func isLost(lastSeq, currentSeq uint16) bool {
    diff := int32(currentSeq) - int32(lastSeq)
    if diff < 0 {
        diff += 65536 // 处理回绕
    }
    return diff != 1 // 正常应该是连续 +1
}
```

---

## 二、RTCP 反馈机制

RTCP（RTP Control Protocol）与 RTP 配合使用，提供传输质量反馈和同步信息。

### 2.1 RTCP 报文类型

| 类型 | 缩写 | 说明 |
|------|------|------|
| **Sender Report** | SR | 发送者报告（发送量、时间戳映射） |
| **Receiver Report** | RR | 接收者报告（丢包率、抖动） |
| **Source Description** | SDES | 源描述（CNAME、NAME 等） |
| **Goodbye** | BYE | 流结束通知 |
| **Application Defined** | APP | 应用自定义 |

### 2.2 Receiver Report 中的关键统计

```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 SSRC_1 (被报告流的 SSRC)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| fraction lost |       累计丢包数量                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           扩展的最高序列号                                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      抖动值 (jitter)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      LSR (Last SR Timestamp)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      DLSR (Delay Since Last SR)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**字段含义**：
- **fraction lost**：本统计周期丢包率（8 位，表示 0-255/256）
- **累计丢包数**：从流开始到现在的总丢包数
- **jitter**：延迟抖动（单位是 RTP 时间戳单位）
- **LSR/DLSR**：用于 RTT 计算

### 2.3 RTCP Feedback 扩展

WebRTC 使用 RTCP Feedback 机制进行拥塞控制和丢包恢复：

| 类型 | 全称 | 触发条件 | 作用 |
|------|------|----------|------|
| **NACK** | Negative ACK | 检测到丢包 | 请求重传特定序列号的包 |
| **PLI** | Picture Loss Indication | 视频解码器无法恢复 | 请求发送 I 帧 |
| **FIR** | Full Intra Request | 需要完整关键帧 | 请求 I 帧（带序列号） |
| **REMB** | Receiver Estimated Maximum Bitrate | 接收端带宽变化 | 通知发送端调整码率 |
| **TWCC** | Transport-wide CC | 每个 RTP 包 | 反馈每个包的到达时间（用于 GCC 算法） |

**NACK 示例**：
```
接收方收到包：seq=100, 101, 104, 105
缺失：102, 103
发送 RTCP NACK，要求重传 102 和 103
```

**PLI 示例**：
```
解码器因累计丢包无法解析 P 帧
发送 RTCP PLI
编码器立即编码 I 帧并发送
```

---

## 三、时序同步与 RTT 计算

### 3.1 音视频同步原理

**问题**：音频和视频的 RTP 时间戳单位不同（48 kHz vs 90 kHz），如何对齐？

**解决方案**：通过 RTCP SR 的 **NTP Timestamp** 和 **RTP Timestamp** 映射。

**RTCP SR 结构**：
```
NTP Timestamp (64 bits): 真实时间（秒 + 小数秒）
RTP Timestamp (32 bits): 对应的 RTP 时间戳
```

**音视频同步算法**：
```python
# 音频流 SR
audio_ntp = 3900000000.500  # NTP 时间
audio_rtp = 48000           # RTP 时间戳

# 视频流 SR
video_ntp = 3900000000.500  # 同一时刻的 NTP 时间
video_rtp = 90000           # RTP 时间戳

# 播放时，将两个流的 RTP 时间戳转换为 NTP 时间对齐
def rtp_to_ntp(rtp, base_rtp, base_ntp, clock_rate):
    return base_ntp + (rtp - base_rtp) / clock_rate
```

### 3.2 RTT 计算（通过 RTCP SR/RR）

**步骤**：
1. 发送方发送 RTCP SR，携带 NTP Timestamp（记为 **LSR**）
2. 接收方收到 SR，等待一段时间（记为 **DLSR**）
3. 接收方发送 RTCP RR，携带 LSR 和 DLSR
4. 发送方收到 RR，计算 RTT

**RTT 公式**：
```
RTT = 当前时间 - LSR - DLSR
```

**代码示例**：
```go
func calculateRTT(receivedAt time.Time, lsr uint32, dlsr uint32) time.Duration {
    // lsr 和 dlsr 单位是 1/65536 秒
    lsrTime := ntpToTime(lsr)
    dlsrDuration := time.Duration(dlsr) * time.Second / 65536
    return receivedAt.Sub(lsrTime) - dlsrDuration
}
```

---

## 四、Jitter Buffer 原理

### 4.1 什么是 Jitter？

**延迟抖动**：由于网络拥塞、路由变化，数据包到达时间间隔不稳定。

**示例**：
```
理想情况：包每 20ms 到达
实际情况：第 1 个包 0ms，第 2 个包 18ms，第 3 个包 42ms
抖动 = |(42-18) - 20| = 4ms
```

### 4.2 Jitter Buffer 工作流程

```
[网络] --> [Jitter Buffer] --> [解码器]
               |
          延迟 100-200ms
          吸收抖动，稳定输出
```

**策略**：
1. **固定延迟**：始终缓存 150ms，简单但可能卡顿
2. **自适应延迟**：根据网络状况动态调整 100-300ms
3. **快速播放/慢速播放**：通过轻微变速（0.95x-1.05x）平滑调整

**自适应算法伪代码**：
```python
target_delay = 150  # ms
current_delay = len(buffer) * packet_duration

if current_delay < target_delay - 50:
    # 缓冲区太小，增加目标延迟
    target_delay += 10
elif current_delay > target_delay + 50:
    # 缓冲区太大，减少目标延迟
    target_delay -= 10

# 通过时间拉伸平滑调整
playback_rate = current_delay / target_delay  # 1.05x 或 0.95x
```

### 4.3 Jitter 的 RTCP 计算

RTP RFC 3550 定义的抖动计算公式：
```
J(i) = J(i-1) + (|D(i-1,i)| - J(i-1)) / 16
```
其中：
- **D(i-1,i)**：相邻两包的到达时间间隔差
- **J(i)**：平滑后的抖动值

---

## 五、RTP 扩展头与多路复用

### 5.1 RTP Header Extension

当 RTP Header 中的 X bit = 1 时，固定头后跟扩展头。

**扩展头格式**：
```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      defined by profile       |           length              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        extension data                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**WebRTC 常用扩展**：
- **abs-send-time**：绝对发送时间（24 位 NTP 时间戳）
- **video-orientation**：视频旋转角度
- **audio-level**：音频音量（用于显示说话人指示器）
- **transport-cc**：传输层 CC 序列号（配合 TWCC 使用）

### 5.2 SSRC 多路复用

WebRTC 中，同一 UDP 端口可以传输多路音视频流，通过 **SSRC** 区分。

**示例：屏幕共享 + 摄像头**
```
摄像头视频：SSRC = 0x12345678
屏幕共享视频：SSRC = 0x87654321
音频：SSRC = 0xABCDEF00
```

接收方根据 SSRC 分流：
```go
func demuxRTP(packet *rtp.Packet) {
    switch packet.SSRC {
    case 0x12345678:
        videoPipeline1.Process(packet)
    case 0x87654321:
        videoPipeline2.Process(packet)
    case 0xABCDEF00:
        audioPipeline.Process(packet)
    }
}
```

---

## 六、WebRTC 中的 RTP 实践

### 6.1 SRTP 加密

WebRTC 使用 **SRTP**（Secure RTP），通过 **DTLS** 协商密钥。

**加密流程**：
```
RTP Packet --> AES-GCM 加密 Payload --> 添加 Auth Tag --> SRTP Packet
```

**SRTP Header 变化**：
```
[RTP Header] [Payload]  -->  [RTP Header] [Encrypted Payload] [Auth Tag]
```

### 6.2 GCC 拥塞控制

Google Congestion Control（GCC）基于 **TWCC** 反馈调整发送码率。

**工作流程**：
1. 发送方给每个 RTP 包打上 Transport-CC Sequence Number（扩展头）
2. 接收方收到包后，记录到达时间
3. 接收方定期发送 TWCC Feedback（包含每个包的到达时间）
4. 发送方计算单向延迟梯度，判断是否拥塞
5. 拥塞 -> 降低码率；空闲 -> 增加码率

**延迟梯度计算**：
```python
for i in range(1, len(packets)):
    send_delta = packets[i].send_time - packets[i-1].send_time
    recv_delta = packets[i].recv_time - packets[i-1].recv_time
    delay_gradient = recv_delta - send_delta
    if delay_gradient > threshold:
        # 网络拥塞，减少码率
        pass
```

---

## 七、动手实践

### 实践 1：解析 RTP 包头

```go
package main

import (
    "encoding/binary"
    "fmt"
)

func parseRTPHeader(data []byte) {
    if len(data) < 12 {
        fmt.Println("包太短")
        return
    }

    version := (data[0] >> 6) & 0x03
    padding := (data[0] >> 5) & 0x01
    extension := (data[0] >> 4) & 0x01
    csrcCount := data[0] & 0x0F
    marker := (data[1] >> 7) & 0x01
    payloadType := data[1] & 0x7F
    seqNum := binary.BigEndian.Uint16(data[2:4])
    timestamp := binary.BigEndian.Uint32(data[4:8])
    ssrc := binary.BigEndian.Uint32(data[8:12])

    fmt.Printf("V=%d P=%d X=%d CC=%d M=%d PT=%d\n", version, padding, extension, csrcCount, marker, payloadType)
    fmt.Printf("SeqNum=%d Timestamp=%d SSRC=0x%X\n", seqNum, timestamp, ssrc)
}

func main() {
    // 示例 RTP 包（12 字节头 + 负载）
    rtpPacket := []byte{
        0x80, 0x60, 0x12, 0x34, // V=2, PT=96, SeqNum=4660
        0x00, 0x00, 0x00, 0x00, // Timestamp=0
        0xAB, 0xCD, 0xEF, 0x12, // SSRC=0xABCDEF12
    }
    parseRTPHeader(rtpPacket)
}
```

**输出**：
```
V=2 P=0 X=0 CC=0 M=0 PT=96
SeqNum=4660 Timestamp=0 SSRC=0xABCDEF12
```

### 实践 2：检测丢包和计算丢包率

```go
type RTPStats struct {
    lastSeq       uint16
    receivedCount uint32
    lostCount     uint32
}

func (s *RTPStats) ProcessPacket(seq uint16) {
    if s.receivedCount > 0 {
        expected := s.lastSeq + 1
        if seq != expected {
            diff := int32(seq) - int32(expected)
            if diff < 0 {
                diff += 65536
            }
            s.lostCount += uint32(diff)
        }
    }
    s.lastSeq = seq
    s.receivedCount++
}

func (s *RTPStats) LossRate() float64 {
    total := s.receivedCount + s.lostCount
    if total == 0 {
        return 0
    }
    return float64(s.lostCount) / float64(total)
}
```

### 实践 3：WebRTC Stats API 获取 RTP 统计

```javascript
// 获取 WebRTC 连接的 RTP 统计信息
async function getRTPStats(pc) {
    const stats = await pc.getStats();
    stats.forEach(report => {
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            const total = report.packetsReceived + report.packetsLost;
            const lossRate = total > 0 ? (report.packetsLost / total * 100).toFixed(2) : '0.00';
            console.log('视频接收统计：');
            console.log('  总接收包数:', report.packetsReceived);
            console.log('  总丢包数:', report.packetsLost);
            console.log('  丢包率:', lossRate + '%');
            console.log('  抖动值:', (report.jitter * 1000).toFixed(1), 'ms');
            console.log('  总接收字节:', report.bytesReceived);
        }
    });
}

// 每 5 秒采集一次
const pc = new RTCPeerConnection();
setInterval(() => getRTPStats(pc), 5000);
```

---

## 八、常见问题排查

### 问题 1：WebRTC 视频卡顿，但网络带宽充足

**排查步骤**：
1. 打开 `chrome://webrtc-internals`
2. 查看 `packetsLost` 和 `jitterBufferDelay`
3. 如果 `jitter` 很大（> 100ms），说明网络抖动严重
4. 排查链路：优先走有线或 5GHz WiFi，减少中间路由跳数

### 问题 2：音视频不同步

**原因**：音频和视频的 RTP 时间戳未对齐，通常是发送端未正确发送 RTCP SR。

**解决方案**：
- 确保发送方在同一时刻采集音视频时，使用相同的 NTP 时间生成 RTP 时间戳
- 检查 RTCP SR 发送频率（建议每秒至少 1 次）
- 接收方根据 RTCP SR 的 NTP-RTP 映射对齐播放

### 问题 3：NACK 重传无效，视频仍然花屏

**原因**：
- 重传的包到达时已超过 Jitter Buffer 播放时间
- P 帧依赖的 I 帧丢失，重传 P 帧无法解码

**解决方案**：
- 增加 Jitter Buffer 延迟（允许更多时间等待重传）
- 改用 PLI 而非 NACK，请求 I 帧从头重建画面

### 问题 4：TWCC Feedback 包占用带宽过大

**现象**：RTCP TWCC 包大小达到 1 KB，高频发送时占用带宽。

**优化方案**：
- 减少 TWCC 反馈频率（从 100ms 改为 200ms）
- 使用 Delta Encoding（只发送时间差，而非绝对时间）

---

## 九、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **RTP Header** | SeqNum 检测丢包，Timestamp 播放对齐，SSRC 流标识 |
| **RTCP 反馈** | SR/RR 质量统计，NACK 重传，PLI 请求 I 帧 |
| **RTT 计算** | 通过 RTCP SR/RR 的 LSR 和 DLSR 字段 |
| **Jitter Buffer** | 吸收网络抖动，自适应调整延迟 |
| **WebRTC 实践** | SRTP 加密，GCC 拥塞控制，TWCC 反馈 |

**关键数字速查**：
- RTP Header 固定长度：**12 字节**
- Sequence Number 回绕点：**65535 → 0**
- 视频时间戳时钟：**90 kHz**
- 音频时间戳时钟（Opus）：**48 kHz**
- Jitter Buffer 典型延迟：**100-200ms**

---

## 下一篇

下一篇将深入讲解 **SRT 协议**：传输层的重传机制、延迟控制、加密方案，以及与 RTMP 的对比，看 SRT 如何在不稳定网络下保障直播质量。

---

## 参考资料

1. [RFC 3550 - RTP: A Transport Protocol for Real-Time Applications](https://datatracker.ietf.org/doc/html/rfc3550)
2. [RFC 4585 - Extended RTP Profile for RTCP-based Feedback](https://datatracker.ietf.org/doc/html/rfc4585)
3. [WebRTC Stats API - W3C](https://www.w3.org/TR/webrtc-stats/)
4. [GCC Algorithm - draft-ietf-rmcat-gcc](https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-gcc)
5. [Pion WebRTC - Go RTP/RTCP 实现](https://github.com/pion/webrtc)

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| RTP | Real-time Transport Protocol | 实时传输协议，承载音视频媒体数据 |
| RTCP | RTP Control Protocol | RTP 控制协议，传递统计和控制信息 |
| SSRC | Synchronization Source | 同步源标识符，唯一标识一路媒体流 |
| 序列号 | Sequence Number | RTP 包的递增编号，用于检测丢包和重排序 |
| 时间戳 | Timestamp | 媒体帧的采样时间，用于音视频同步 |
| NACK | Negative Acknowledgment | 否定确认，通知发送方某包丢失需重传 |
| PLI | Picture Loss Indication | 图像丢失指示，请求对端发送关键帧 |
| FIR | Full Intra Request | 完整帧请求，要求立即发送 I 帧 |
| 抖动 | Jitter | 包到达时间的变化量，过大会导致播放卡顿 |
| SR | Sender Report | RTCP 发送方报告，含发包统计和 NTP 时间戳 |
