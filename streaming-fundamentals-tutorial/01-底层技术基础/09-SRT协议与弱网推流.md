# SRT 协议与弱网推流

## 目录

- [读完本篇你能做什么](#读完本篇你能做什么)
- [一、SRT 协议概述](#一srt-协议概述)
- [二、SRT 可靠传输机制](#二srt-可靠传输机制)
- [三、SRT 延迟控制](#三srt-延迟控制)
- [四、SRT 加密与安全](#四srt-加密与安全)
- [五、SRT 连接模式](#五srt-连接模式)
- [六、SRT vs RTMP 性能对比](#六srt-vs-rtmp-性能对比)
- [七、动手实践](#七动手实践)
- [八、常见问题排查](#八常见问题排查)
- [九、本篇小结](#九本篇小结)
- [下一篇](#下一篇)
- [参考资料](#参考资料)

---

## 读完本篇你能做什么

1. **理解 SRT 优势**：在弱网环境下如何保持稳定推流
2. **配置 OBS SRT 推流**：使用 SRT 协议代替 RTMP
3. **部署 SRT 服务器**：使用 srt-live-server 搭建 SRT 网关
4. **调整延迟参数**：根据网络条件设置 latency 参数
5. **对比 SRT/RTMP**：在丢包 10-30% 场景下的性能差异
6. **排查 SRT 连接问题**：处理防火墙和 NAT 穿透

---

## 一、SRT 协议概述

### 1.1 什么是 SRT？

**SRT**（Secure Reliable Transport）是 Haivision 开发的开源传输协议，专为**不稳定网络**设计。

**核心目标**：
- 在 **10-30% 丢包率**下保持稳定传输（RTMP 在 5% 丢包时就严重卡顿）
- 提供 **AES 加密**（RTMP 明文传输）
- **低延迟**（200ms-8s 可配置）
- 穿透防火墙和 NAT

**典型应用场景**：
- 4G/5G 移动网络推流
- 跨国推流（高延迟高丢包网络）
- 无人机直播
- 卫星通信
- 新闻现场直播

### 1.2 SRT vs RTMP

| 维度 | SRT | RTMP |
|------|-----|------|
| **传输层** | UDP + ARQ | TCP |
| **丢包恢复** | 选择性重传（NACK） | TCP 全序重传 |
| **延迟范围** | 200ms-8s 可配 | 1-3s 固定 |
| **抗丢包能力** | 30% 丢包仍可用 | 5% 丢包严重卡顿 |
| **加密** | 内置 AES-128/256 | 需 RTMPS（TLS） |
| **端口** | 自定义（常用 9000） | 1935 |
| **浏览器支持** | 无（需服务器转码） | 无（Flash 已死） |

**为什么 SRT 更抗丢包？**
- RTMP 基于 TCP，一个包丢失会阻塞整个流（TCP Head-of-Line Blocking）
- SRT 基于 UDP，只重传丢失的包，不阻塞后续数据

---

## 二、SRT 可靠传输机制

### 2.1 ARQ 选择性重传

SRT 使用 **NACK-based ARQ**（自动重传请求）：

```
发送方              接收方
  |                   |
  |  Pkt 1 ---------> |
  |  Pkt 2 ---------> |
  |  Pkt 3 -- 丢失 -- X
  |  Pkt 4 ---------> |
  |                   |
  |   <--- NACK(3)    |  (请求重传第 3 包)
  |                   |
  |  Pkt 3 (重传) --> |
  |                   |
  |  继续发 Pkt 5 --> |  (不阻塞后续包)
```

**与 TCP 的关键区别**：
- TCP：丢包后等待重传，后续所有包在接收缓冲区中积压
- SRT：丢包后只重传该包，其他包继续播放（只要在延迟窗口内）

### 2.2 数据包格式

**SRT 数据包头（16 字节）**：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                     Sequence Number                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|P P|O|K K|R|              Message Number                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Timestamp                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Socket ID                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段**：
- **Sequence Number**：31 位序列号（第 1 位 0 表示数据包）
- **PP**：位置标记（10=first, 00=middle, 01=last, 11=single）
- **K K**：加密状态（00=clear, 01=even key, 10=odd key）

### 2.3 NACK 报文

接收方检测到丢包后，立即发送 NACK：

```
Control Packet (bit 1 = control)
Type = 0x0003 (NAK)
Lost Sequence List:
  - 单包：直接写序列号
  - 范围：写 [start, end]（第 1 位 = 1 表示范围起点）
```

**NACK 发送时机**：
1. 检测到序列号不连续时立即发送
2. 之后每 RTT/2 时间重复发送，直到收到重传包

---

## 三、SRT 延迟控制

### 3.1 延迟参数（Latency）

SRT 的核心参数是 **latency**，也叫 TL（Transmission Latency）：

```
latency ≥ RTT + 重传时间
```

**推荐设置**：
```
本地局域网：120ms
国内城市间：300ms
跨国（如中美）：1000-2000ms
卫星链路：4000-8000ms
```

### 3.2 延迟窗口原理

```
时间轴：
[发送] -------latency------> [播放]
         重传机会窗口
```

- 发送方缓存最近 `latency` 毫秒的数据
- 接收方等待 `latency` 毫秒再播放
- 在 latency 窗口内的丢包可以被重传恢复
- 超出窗口的包直接丢弃（不影响后续播放）

**latency 与丢包恢复的权衡**：

| Latency | 丢包恢复 | 延迟 |
|---------|---------|------|
| 120ms | 仅能恢复 RTT < 30ms 的丢包 | 低 |
| 500ms | 能恢复多次重传（RTT 100ms 场景） | 中 |
| 2000ms | 能恢复卫星/跨国丢包 | 高 |

### 3.3 带宽估算

SRT 持续测量实际带宽，自动调整发送速率：

```
实际带宽 = 接收到的字节数 / 时间窗口
冗余带宽 = 原始数据 + 重传数据

if 拥塞（丢包率 > 阈值）:
    降低发送速率
else:
    维持或小幅增加
```

---

## 四、SRT 加密与安全

### 4.1 内置 AES 加密

SRT 原生支持 AES-128 和 AES-256 加密，无需 TLS。

**工作原理**：
1. 连接建立时，双方通过 KMX（Key Material Exchange）协商密钥
2. 使用 **Passphrase** 生成 **Salt + SEK**（Stream Encryption Key）
3. 每隔一段时间自动轮换密钥（Even/Odd Key 交替）

**密钥轮换**：
```
Even Key (0x01) --> 加密前 10 秒
                    双方协商 Odd Key
Odd Key (0x02)  --> 继续加密
                    双方协商新 Even Key
无缝切换，不中断传输
```

### 4.2 配置加密

```bash
# FFmpeg 推流时启用加密
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -f mpegts \
  "srt://server:9000?passphrase=my-secret-key&pbkeylen=16"

# pbkeylen=16：AES-128
# pbkeylen=32：AES-256
```

**安全注意事项**：
- Passphrase 至少 10 个字符（SRT 规范要求）
- 不要在 URL 中明文传输密码（用环境变量或配置文件）

---

## 五、SRT 连接模式

### 5.1 三种连接模式

| 模式 | 说明 | 应用场景 |
|------|------|----------|
| **Caller** | 主动连接到 Listener | 编码器连接服务器 |
| **Listener** | 监听等待连接 | 服务器/接收端 |
| **Rendezvous** | 双方同时发起连接 | P2P 穿透 NAT |

### 5.2 Caller-Listener 模式（最常见）

```
[OBS/编码器 - Caller]  ----SRT----> [srt-live-server - Listener]
    srt://server:9000                    listen :9000
```

**FFmpeg Caller 模式**：
```bash
ffmpeg -i input.mp4 -f mpegts "srt://server:9000?mode=caller"
```

**FFmpeg Listener 模式**（等待推流）：
```bash
ffmpeg -i "srt://0.0.0.0:9000?mode=listener" -c copy output.ts
```

### 5.3 Rendezvous 模式（P2P 穿透）

双方同时向对方发起连接，利用 UDP 打洞穿透 NAT：

```
[端 A] <---SRT Rendezvous---> [端 B]
   |                              |
   发起连接                     同时发起连接
   到 B 的公网地址               到 A 的公网地址
```

```bash
# 端 A
ffmpeg -i input.mp4 -f mpegts "srt://B_IP:9000?mode=rendezvous&localport=9000"

# 端 B
ffmpeg -i "srt://A_IP:9000?mode=rendezvous&localport=9000" -c copy output.ts
```

---

## 六、SRT vs RTMP 性能对比

### 6.1 弱网测试结果

| 丢包率 | SRT（latency=500ms） | RTMP |
|--------|---------------------|------|
| 0% | 正常，延迟 500ms | 正常，延迟 1-2s |
| 5% | 轻微卡顿，可恢复 | 明显卡顿 |
| 10% | 基本正常 | 严重卡顿，频繁断流 |
| 20% | 轻微马赛克 | 基本不可用 |
| 30% | 仍可播放 | 完全断流 |

### 6.2 延迟对比

```
RTMP（正常网络）：约 1-3 秒
SRT（latency=120ms）：约 120-200 毫秒
SRT（latency=500ms）：约 500-600 毫秒
```

**SRT 在良好网络下延迟比 RTMP 更低**，因为 UDP 无需 TCP 的拥塞控制慢启动。

---

## 七、动手实践

### 实践 1：模拟弱网测试

```bash
# macOS 用 Network Link Conditioner
# Linux 用 tc 命令模拟丢包

# 在服务器端模拟 10% 丢包
tc qdisc add dev eth0 root netem loss 10%

# 测试 RTMP 推流（会严重卡顿）
ffmpeg -re -i test.mp4 -c copy -f flv rtmp://server/live/test

# 测试 SRT 推流（基本正常）
ffmpeg -re -i test.mp4 -c copy -f mpegts "srt://server:9000?latency=500000"

# 恢复网络
tc qdisc del dev eth0 root
```

### 实践 2：部署 srt-live-server

```bash
# 从源码编译 SRT
git clone https://github.com/Haivision/srt.git
cd srt && mkdir build && cd build
cmake ..
make -j4 && sudo make install

# 使用 srt-live-transmit（SRT 官方工具）
# 接收 SRT 流并输出到 UDP（供 FFmpeg 消费）
srt-live-transmit "srt://:9000?mode=listener" "udp://127.0.0.1:5000"

# FFmpeg 消费 UDP 流并转成 HLS
ffmpeg -i udp://127.0.0.1:5000 \
  -c:v copy -c:a aac \
  -f hls -hls_time 2 /var/www/html/hls/stream.m3u8
```

### 实践 3：OBS 配置 SRT 推流

OBS 28+ 原生支持 SRT 推流：
```
设置 → 直播
  服务：自定义...
  服务器：srt://server:9000?latency=500000
  串流密钥：（留空）

高级设置 → 网络
  绑定 IP：（默认）
```

**latency 参数单位是微秒**：
```
500ms = 500000 微秒
1s    = 1000000 微秒
```

### 实践 4：使用 SRT 统计信息监控链路质量

```bash
# srt-live-transmit 的统计输出示例
# 可以通过 API 获取详细统计

# Go 代码读取 SRT 统计
package main

import (
    "fmt"
    "time"
    gosrt "github.com/datarhei/gosrt"
)

func main() {
    config := gosrt.DefaultConfig()
    config.Latency = 500 * time.Millisecond

    conn, err := gosrt.Dial("srt", "server:9000", config)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    // 定期打印统计信息
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        stats := conn.Stats()
        fmt.Printf("RTT: %vms, 丢包率: %.2f%%, 带宽: %.2f Mbps\n",
            stats.RTT.Milliseconds(),
            stats.PacketLossRate*100,
            stats.Bandwidth/1024/1024)
    }
}
```

---

## 八、常见问题排查

### 问题 1：SRT 连接建立失败

**排查步骤**：
1. 确认 UDP 端口开放（SRT 使用 UDP）：
   ```bash
   nc -u -zv server 9000
   ```
2. 检查防火墙是否放行 UDP：
   ```bash
   iptables -A INPUT -p udp --dport 9000 -j ACCEPT
   ```

### 问题 2：延迟过高

**现象**：设置了 500ms latency 但实际延迟超过 2 秒。

**原因**：latency 设置不够，丢包重传次数超出预期。

**解决**：增大 latency，计算公式：
```
建议 latency = RTT × 4 + 100ms
```

### 问题 3：间歇性马赛克（丢包率正常）

**原因**：latency 太小，部分重传包来不及到达就已超时丢弃。

**解决**：增大 latency 参数。

### 问题 4：高延迟网络下内存增长

**原因**：SRT 缓存了 latency 时间内的所有包，高延迟 + 高码率 = 大内存。

**计算公式**：
```
缓存大小 = 码率 (Mbps) × latency (s)
例：4 Mbps × 2s = 1 MB 缓存
```

---

## 九、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **SRT 优势** | 弱网抗丢包（30%），内置 AES 加密 |
| **可靠传输** | NACK 选择性重传，不阻塞后续包 |
| **延迟参数** | latency ≥ RTT × 4；单位微秒（OBS）或毫秒（FFmpeg） |
| **连接模式** | Caller/Listener（常用）、Rendezvous（P2P） |
| **与 RTMP 对比** | 丢包恢复更强；延迟可更低；无浏览器直播支持 |

**选择指南**：
- 局域网/稳定网络 → RTMP（够用，生态成熟）
- 4G/5G 移动推流 → SRT（抗弱网）
- 跨国/卫星推流 → SRT（必选）

---

## 下一篇

下一篇将深入讲解 **HLS 协议**：苹果的 HTTP 直播标准，M3U8 格式解析，分片策略对延迟的影响，以及 HLS 如何配合 CDN 实现全球分发。

---

## 参考资料

1. [SRT Protocol Technical Overview - Haivision](https://www.haivision.com/resources/white-paper/srt-protocol-technical-overview/)
2. [SRT RFC Draft - IETF](https://datatracker.ietf.org/doc/html/draft-sharabayko-srt)
3. [libsrt - GitHub](https://github.com/Haivision/srt)
4. [FFmpeg SRT 文档](https://ffmpeg.org/ffmpeg-protocols.html#srt)
5. [OBS SRT 推流指南](https://obsproject.com/wiki/Streaming-With-SRT-Protocol)

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| SRT | Secure Reliable Transport | 安全可靠传输协议，基于 UDP，专为弱网优化 |
| ARQ | Automatic Repeat Request | 自动重传请求，丢包后请求发送方重传 |
| FEC | Forward Error Correction | 前向纠错，通过冗余包在接收端恢复丢失数据 |
| latency | Latency Buffer | SRT 延迟缓冲窗口，决定重传等待时间（默认120ms）|
| Caller | Caller | SRT 主动发起连接的一端 |
| Listener | Listener | SRT 被动等待连接的一端 |
| Rendezvous | Rendezvous | SRT 点对点穿透模式，双方同时尝试连接 |
| passphrase | Passphrase | SRT 加密密钥，基于 AES-128/256 |
| streamid | Stream ID | SRT 流标识符，用于服务端区分不同的流 |
| RTT | Round-Trip Time | 往返时延，SRT 动态调整重传策略的依据 |
