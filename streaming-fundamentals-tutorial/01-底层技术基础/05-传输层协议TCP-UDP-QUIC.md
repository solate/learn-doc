# 第 5 篇：传输层协议：TCP vs UDP vs QUIC

> **模块**：底层技术基础 | **难度**：中级 | **预计阅读时间**：20 分钟

## 目录

- [读完本篇你能做什么](#读完本篇你能做什么)
- [一、为什么传输层协议对流媒体如此重要](#一为什么传输层协议对流媒体如此重要)
- [二、TCP：可靠但有代价](#二tcp可靠但有代价)
- [三、UDP：速度优先，应用层自己处理](#三udp速度优先应用层自己处理)
- [四、QUIC：HTTP/3 的传输层革命](#四quichttp3-的传输层革命)
- [五、流媒体场景的协议选择](#五流媒体场景的协议选择)
- [六、拥塞控制算法对延迟的影响](#六拥塞控制算法对延迟的影响)
- [七、动手实践：观察协议行为](#七动手实践观察协议行为)
- [本篇小结](#本篇小结)

---

## 读完本篇你能做什么

- 解释 TCP 的三次握手、拥塞控制和队头阻塞对流媒体延迟的影响
- 理解为什么 WebRTC/RTP 使用 UDP 而非 TCP
- 解释 QUIC 如何解决 TCP 的队头阻塞问题
- 根据场景（实时通信/直播/CDN 分发）选择合适的传输层协议
- 理解 BBR 拥塞控制算法对直播低延迟的意义

---

## 一、为什么传输层协议对流媒体如此重要

视频流对网络的要求非常特殊：

```
普通文件下载：
  只关心"最终完整性"，慢一点也没关系
  TCP 的重传机制 → 完美适用

视频直播（1080p, 4Mbps）：
  每秒必须传输 4Mbps × 1/8 = 500KB 数据
  任何超过 100ms 的阻塞 → 播放卡顿
  丢包率 > 5% → 画面花屏/绿屏

实时视频通话（720p, 1.5Mbps）：
  总延迟要求 < 150ms（MOS 分数 > 4.0）
  TCP 重传可能引入 200-500ms 额外延迟
  → 必须用 UDP，丢帧比延迟更可接受
```

---

## 二、TCP：可靠但有代价

### TCP 的可靠性机制

```
TCP 三次握手（每次连接至少 1.5 RTT 延迟）：
  Client ──SYN──────────────→ Server
  Client ←──SYN-ACK───────── Server
  Client ──ACK──────────────→ Server
  Client ──HTTP GET──────────→ Server  ← 这时才开始传数据

TLS 握手额外增加 1-2 RTT：
  总启动延迟：3-4 RTT（北京→上海 ~120ms RTT = 360-480ms 启动）
```

### TCP 的问题：队头阻塞（Head-of-Line Blocking）

```
TCP 字节流顺序保证：
  [Packet 1 OK] → [Packet 2 丢失] → [Packet 3 OK] → ...
                         ↓
  Packet 3 到了，必须等 Packet 2 重传完才能交给应用层
  等待时间 = 至少 1 RTT（重传请求来回）

视频播放器影响：
  假设 RTT = 50ms，丢了 1 个包：
  → 播放器必须等 50ms 才能继续解码
  → 如果在 I 帧位置丢包，整个 GOP 被阻塞
```

### TCP 的拥塞控制（慢启动）

```
TCP 慢启动过程（新连接）：
  初始窗口 10 MSS ≈ 14.5 KB
  每收到 1 ACK → 窗口 +1 MSS
  窗口指数增长，直到 ssthresh

对直播的影响：
  断线重连后 TCP 需要重新慢启动
  重新达到所需带宽可能需要 2-5 秒
  → 这就是为什么直播断线后要"缓冲"
```

---

## 三、UDP：速度优先，应用层自己处理

UDP 是无连接、无序、无可靠性保证的传输协议，只提供最基本的数据传输。

```
UDP 数据报文：
  [源端口 2B][目标端口 2B][长度 2B][校验和 2B][数据]
  
特点：
  - 无连接：不需要握手，直接发送
  - 无序：包可能乱序到达
  - 无重传：丢了就丢了，应用层决定是否重传
  - 低延迟：无队头阻塞，无慢启动
```

### 为什么实时通信用 UDP？

```
实时视频通话延迟容忍分析：
  
  用户说话 → 编码 → 发送 → 网络 → 接收 → 解码 → 播放
  总预算：≤ 150ms（人眼/耳感知阈值）

  编码：10-20ms
  解码：5-15ms
  缓冲区：10-30ms
  
  网络可用时间：≤ 150 - 20 - 15 - 30 = 85ms
  
  如果用 TCP：
    1次重传 = 1 RTT（北京→上海 = 40ms）
    → 1次重传就超预算！
  
  如果用 UDP：
    丢帧 → 用上一帧替代（画面短暂静止）
    → 视觉上不好，但延迟不受影响
    → 比等待重传好得多
```

---

## 四、QUIC：HTTP/3 的传输层革命

**QUIC（Quick UDP Internet Connections）** 是 Google 2012 年开始开发、2021 年成为 RFC 9000 标准的传输层协议。它基于 UDP，但实现了 TCP 的可靠性，同时解决了 TCP 的关键缺陷。

### QUIC 解决了哪些 TCP 问题？

| TCP 问题 | QUIC 解决方案 |
|---------|-------------|
| **队头阻塞** | 多路复用：每个流独立，一个流丢包不影响其他流 |
| **握手延迟** | 0-RTT 或 1-RTT 建连（TLS 1.3 集成） |
| **连接迁移** | 基于 Connection ID 而非 IP+端口，Wi-Fi→4G 无缝切换 |
| **头部压缩** | QPACK（针对 UDP 优化的 HPACK 变体） |
| **明文暴露** | 加密整个传输层（只有 Connection ID 可见） |

### QUIC 的多流独立性

```
TCP 上的 HTTP/2（仍有队头阻塞）：
  TCP 字节流: [Stream1-P1][Stream2-P1][Stream1-P2丢失][Stream2-P2]
                                          ↓
                              Stream2-P2 被阻塞，等待 Stream1-P2 重传

QUIC 上的 HTTP/3（真正独立）：
  UDP 数据报:  [Stream1-P1][Stream2-P1][Stream1-P2丢失][Stream2-P2]
                                          ↓
                              Stream2-P2 正常交付！Stream1 独立重传
```

### QUIC 在流媒体中的应用

| 应用 | 使用情况 |
|------|---------|
| **HTTP/3 + HLS** | CDN 分发 HLS 切片，减少首屏时间 |
| **WebTransport** | 浏览器实时双向通信，WebRTC 的 HTTP 替代 |
| **DASH over QUIC** | Netflix/YouTube 实验性部署 |
| **媒体服务器间传输** | SRS 5.x 支持 QUIC 级联 |

---

## 五、流媒体场景的协议选择

```
场景 → 传输层协议 → 理由
──────────────────────────────────────────────────────────────
WebRTC 实时通话         → UDP    → 延迟优先，丢帧可接受
RTMP 直播推流           → TCP    → 历史包袱，推流端容忍延迟
SRT 弱网推流            → UDP    → 内置 ARQ，可靠 + 低延迟
RTSP 摄像头拉流         → TCP/UDP → 信令 TCP，数据 UDP（RTP）
HLS/DASH 点播          → TCP    → 基于 HTTP，可靠性优先
HTTP-FLV 直播播放       → TCP    → 基于 HTTP，1-3s 延迟可接受
WebTransport 实验性     → QUIC   → 未来方向，低延迟 + 可靠
```

### 为什么 RTMP 使用 TCP？

RTMP 是 2003 年设计的协议，当时的考虑：
1. Flash Player 没有 UDP 访问权限
2. TCP 的可靠性降低了客户端实现复杂度
3. 当时网络质量好，TCP 延迟问题不突出

2023 年后，RTMP 的 TCP 局限性逐渐显现，Enhanced RTMP 规范开始讨论支持 QUIC 传输。

---

## 六、拥塞控制算法对延迟的影响

**拥塞控制**决定了 TCP/QUIC 如何探测可用带宽，对直播延迟影响很大。

| 算法 | 年份 | 原理 | 流媒体适用性 |
|------|------|------|------------|
| **Cubic**（Linux 默认） | 2006 | 丢包触发降速 | 一般（丢包后降速影响直播质量） |
| **BBR v1**（Google） | 2016 | 带宽+延迟估计 | 好（低延迟，带宽利用率高） |
| **BBR v3**（2024） | 2024 | BBR v2 改进版 | 最佳（减少队列积累） |
| **QUIC CC** | 集成 | 各实现不同 | 视实现而定 |

```bash
# 查看当前服务器拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

# 启用 BBR（Linux 4.9+）
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# 验证
sysctl net.ipv4.tcp_congestion_control  # 应显示 bbr
```

**BBR 对直播的价值**：BBR 不等丢包再降速，而是主动探测带宽瓶颈，在同等网络条件下吞吐量更高、延迟更低，对直播推流有 20-30% 的延迟改善。

---

## 七、动手实践：观察协议行为

```bash
# 1. 用 tcpdump 观察 RTMP（TCP）连接建立
sudo tcpdump -i any -n port 1935 -c 100

# 2. 用 netstat 查看 TCP 连接状态（推流中）
watch -n 1 "netstat -ant | grep 1935"

# 3. 用 ss 查看 TCP 拥塞窗口
ss -tin dst :1935 | grep cwnd

# 4. 测试带宽延迟（模拟弱网）
# macOS Network Link Conditioner 或 Linux tc：
sudo tc qdisc add dev eth0 root netem delay 100ms loss 3%  # 100ms 延迟 + 3% 丢包
sudo tc qdisc del dev eth0 root  # 恢复

# 5. 测试 QUIC/HTTP3 支持
curl -I --http3 https://www.cloudflare.com  # 需要 curl 7.88+ with QUIC

# 6. 用 Wireshark 观察 RTP（UDP）包
# 过滤器：rtp || rtcp
# 观察 RTP 时间戳、序列号、RTCP SR/RR 报告
```

---

## 本篇小结

1. **TCP 的代价**：可靠但引入队头阻塞和慢启动延迟，不适合实时通信
2. **UDP 的优势**：无队头阻塞，发完即走，应用层灵活处理丢包（丢帧优于延迟）
3. **QUIC 的创新**：基于 UDP 实现 TCP 可靠性，多流独立、0-RTT 建连、连接迁移，是 HTTP/3 的传输层
4. **直播用 TCP 是历史包袱**：RTMP/HLS/HTTP-FLV 都基于 TCP，新系统（SRT/WebRTC/WHIP）转向 UDP
5. **BBR 拥塞控制**：服务器启用 BBR 可减少直播延迟 20-30%，生产环境推荐

---

## 下一篇

下一篇将深入讲解 **RTP/RTCP 协议**：实时传输的骨干，WebRTC 和 RTSP 都依赖它。RTP 包如何携带时间戳实现音视频同步？RTCP 如何反馈网络质量？

👉 [第 6 篇：RTP/RTCP 实时传输协议详解](./06-RTP-RTCP实时传输协议详解.md)

---

**参考资料**：

- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000) — QUIC 标准
- [RFC 5681 — TCP Congestion Control](https://www.rfc-editor.org/rfc/rfc5681) — TCP 拥塞控制
- [Google BBR 论文](https://research.google/pubs/pub45646/) — BBR 拥塞控制
- [HTTP/3 explained（中文版）](https://http3-explained.haxx.se/zh) — QUIC/HTTP3 入门

---

**本篇完** | [返回目录](../../README.md)

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| UDP 推流频繁丢帧 | 网络抖动大、缓冲区不足 | 改用 SRT（自带 ARQ 重传），或增大接收缓冲 |
| QUIC 连接失败 | 防火墙拦截 UDP 443 | 检查防火墙规则，放行 UDP 443 |
| TCP 推流延迟高 | TCP 队头阻塞，重传等待 | 若延迟敏感场景改用 SRT/WebRTC |
| 连接时断时续 | MTU 不匹配导致分片 | `ping -s 1400 <server>` 测试，调整 MTU |
| 弱网下视频卡顿 | 带宽波动超过 TCP 拥塞窗口 | 使用 BBR 拥塞算法，或换 QUIC/SRT |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| RTT | Round-Trip Time | 数据从发送到收到确认的往返时间 |
| MTU | Maximum Transmission Unit | 网络层最大传输单元，以太网默认 1500 字节 |
| 拥塞控制 | Congestion Control | TCP 动态调整发送速率避免网络拥塞的机制 |
| QUIC | Quick UDP Internet Connections | 基于 UDP 的传输层协议，HTTP/3 底层 |
| TLS | Transport Layer Security | 传输层加密协议，HTTPS 和 QUIC 内置 |
| 队头阻塞 | Head-of-Line Blocking | TCP 中前一个包丢失导致后续包等待的问题 |
| BBR | Bottleneck Bandwidth and Round-trip propagation time | Google 开发的基于带宽估计的拥塞算法 |
