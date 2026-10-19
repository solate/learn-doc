# WHIP/WHEP 标准化推拉流

> **适合读者**：后端开发者、流媒体工程师  
> **前置知识**：[第 12 篇：WebRTC 协议栈详解](12-WebRTC协议栈详解.md)

---

## 一、为什么需要 WHIP/WHEP

### 1.1 WebRTC 信令的混乱现状

WebRTC 标准只定义了媒体传输，信令（如何交换 SDP）完全由应用自定义：

```
OBS 想推流到 Cloudflare → 需要学习 Cloudflare 的私有信令 API
OBS 想推流到 Mux       → 又要学一套不同的信令 API
OBS 想推流到 SRS       → 再学一套...

每个平台信令格式不同，互不兼容
```

### 1.2 WHIP/WHEP 的目标

**WHIP（WebRTC-HTTP Ingestion Protocol）**：标准化推流信令  
**WHEP（WebRTC-HTTP Egress Protocol）**：标准化拉流信令

```
推流：OBS/编码器 ──WHIP──> 任意支持 WHIP 的服务器
拉流：任意播放器 <──WHEP── 任意支持 WHEP 的服务器

一套 API，到处互通
```

---

## 二、WHIP 协议详解

### 2.1 WHIP 核心：HTTP POST 交换 SDP

WHIP 极其简单——用一个 HTTP POST 完成 WebRTC 推流建立：

```
推流端                              WHIP 服务器
  |                                      |
  | POST /whip/stream-key               |
  | Content-Type: application/sdp       |
  | Body: SDP Offer                     |
  |------------------------------------->|
  |                                      |
  |          HTTP 201 Created            |
  |          Content-Type: application/sdp|
  |          Location: /whip/session/xxx |
  |          Body: SDP Answer            |
  |<-------------------------------------|
  |                                      |
  |====== ICE + DTLS + SRTP 建立 ======|
  |====== 开始发送媒体流 ================|
```

### 2.2 WHIP 请求格式

```http
POST /whip/live/my-stream HTTP/1.1
Host: streaming.example.com
Authorization: Bearer <stream-key>
Content-Type: application/sdp

v=0
o=- 8297126837888455 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
m=video 9 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 H264/90000
a=ice-ufrag:abc1
a=ice-pwd:secretpassword
...
```

**成功响应**：
```http
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: /whip/session/xyz789
ETag: "abc123"

v=0
o=- 3921375 2 IN IP4 0.0.0.0
s=-
...（SDP Answer）
```

**关键字段**：
- `Location`：会话资源路径，用于后续操作（PATCH/DELETE）
- `ETag`：用于 ICE restart 的版本控制

### 2.3 ICE Trickle（ICE 候选逐步发送）

WHIP 支持 ICE Trickle——不等候选收集完就开始连接：

```http
PATCH /whip/session/xyz789 HTTP/1.1
Content-Type: application/trickle-ice-sdpfrag

a=ice-ufrag:abc1
a=ice-pwd:secretpassword
a=candidate:1 1 UDP 2130706431 192.168.1.100 9000 typ host
```

### 2.4 断开推流

```http
DELETE /whip/session/xyz789 HTTP/1.1
Authorization: Bearer <stream-key>
```

---

## 三、WHEP 协议详解

WHEP 是 WHIP 的拉流版本，同样基于 HTTP：

```http
POST /whep/live/my-stream HTTP/1.1
Host: streaming.example.com
Content-Type: application/sdp

v=0
...（播放端的 SDP Offer）
```

**响应**：
```http
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: /whep/session/abc456
Link: <stun:stun.l.google.com>; rel="ice-server"

v=0
...（SDP Answer，包含媒体轨道信息）
```

**`Link` 头**：服务器可以通过 Link 头告知客户端 STUN/TURN 服务器，避免客户端硬编码。

---

## 四、WHIP/WHEP 与 RTMP 的对比

| 特性 | RTMP | WHIP/WHEP |
|------|------|-----------|
| 延迟 | 1-3s | 100-500ms |
| 协议基础 | TCP | UDP（WebRTC） |
| 信令 | 自定义（RTMP 握手） | HTTP REST（标准化） |
| 加密 | RTMPS（可选） | 强制 DTLS+SRTP |
| 防火墙穿透 | 需要开 1935 | 走 443（HTTPS） |
| 浏览器直接推流 | 需要 Flash（已死）| 原生支持 |
| CDN 支持 | 成熟 | 逐步完善 |

---

## 五、用 Go 实现 WHIP 服务器

### 5.1 基础框架

```go
package main

import (
    "io"
    "net/http"
    "sync"
    
    "github.com/gin-gonic/gin"
    "github.com/pion/webrtc/v3"
)

type WHIPServer struct {
    mu       sync.RWMutex
    sessions map[string]*webrtc.PeerConnection
}

func NewWHIPServer() *WHIPServer {
    return &WHIPServer{
        sessions: make(map[string]*webrtc.PeerConnection),
    }
}

// POST /whip/:streamKey - 接收推流
func (s *WHIPServer) HandlePublish(c *gin.Context) {
    streamKey := c.Param("streamKey")

    // 读取 SDP Offer
    body, err := io.ReadAll(c.Request.Body)
    if err != nil {
        c.Status(http.StatusBadRequest)
        return
    }

    // 创建 PeerConnection
    pc, err := s.createPeerConnection(string(body), streamKey)
    if err != nil {
        c.Status(http.StatusInternalServerError)
        return
    }

    // 生成 session ID
    sessionID := generateSessionID()
    s.mu.Lock()
    s.sessions[sessionID] = pc
    s.mu.Unlock()

    // 返回 SDP Answer
    c.Header("Location", "/whip/session/"+sessionID)
    c.Header("Content-Type", "application/sdp")
    c.String(http.StatusCreated, pc.LocalDescription().SDP)
}

// DELETE /whip/session/:sessionID - 断开推流
func (s *WHIPServer) HandleDelete(c *gin.Context) {
    sessionID := c.Param("sessionID")

    s.mu.Lock()
    pc, ok := s.sessions[sessionID]
    if ok {
        pc.Close()
        delete(s.sessions, sessionID)
    }
    s.mu.Unlock()

    if !ok {
        c.Status(http.StatusNotFound)
        return
    }
    c.Status(http.StatusNoContent)
}

func (s *WHIPServer) createPeerConnection(offerSDP, streamKey string) (*webrtc.PeerConnection, error) {
    config := webrtc.Configuration{
        ICEServers: []webrtc.ICEServer{
            {URLs: []string{"stun:stun.l.google.com:19302"}},
        },
    }

    pc, err := webrtc.NewPeerConnection(config)
    if err != nil {
        return nil, err
    }

    // 处理接收到的媒体轨道
    pc.OnTrack(func(track *webrtc.TrackRemote, _ *webrtc.RTPReceiver) {
        log.Info().Str("streamKey", streamKey).Str("kind", track.Kind().String()).Msg("收到媒体轨道")
        go handleTrack(track, streamKey)
    })

    // 设置 Offer
    offer := webrtc.SessionDescription{
        Type: webrtc.SDPTypeOffer,
        SDP:  offerSDP,
    }
    if err = pc.SetRemoteDescription(offer); err != nil {
        return nil, err
    }

    // 创建 Answer
    answer, err := pc.CreateAnswer(nil)
    if err != nil {
        return nil, err
    }

    // 等待 ICE 收集完成
    gatherComplete := webrtc.GatheringCompletePromise(pc)
    if err = pc.SetLocalDescription(answer); err != nil {
        return nil, err
    }
    <-gatherComplete

    return pc, nil
}

func handleTrack(track *webrtc.TrackRemote, streamKey string) {
    // 将 RTP 包写入 FFmpeg 管道，转成 HLS 或其他格式
    for {
        pkt, _, err := track.ReadRTP()
        if err != nil {
            return
        }
        // 处理 RTP 包...
        _ = pkt
    }
}
```

### 5.2 注册路由

```go
func main() {
    r := gin.Default()
    srv := NewWHIPServer()

    // WHIP 端点
    r.POST("/whip/:streamKey", srv.HandlePublish)
    r.DELETE("/whip/session/:sessionID", srv.HandleDelete)
    r.PATCH("/whip/session/:sessionID", srv.HandleICETrickle)

    r.Run(":8080")
}
```

---

## 六、OBS 配置 WHIP 推流

OBS 30+ 原生支持 WHIP 推流：

```
设置 → 直播
  服务：WHIP
  服务器：https://streaming.example.com/whip/
  串流密钥：my-stream-key

高级设置：
  自定义 HTTP 头：Authorization: Bearer <token>
```

**测试用 MediaMTX（推荐）**：
```yaml
# mediamtx.yml
paths:
  all:
    source: publisher

# MediaMTX 自动支持 WHIP/WHEP，无需额外配置
# WHIP 地址：http://localhost:8889/stream/whip
# WHEP 地址：http://localhost:8889/stream/whep
```

---

## 七、当前生态支持

| 产品 | WHIP 支持 | WHEP 支持 |
|------|----------|----------|
| **OBS** 30+ | ✅ | - |
| **MediaMTX** | ✅ | ✅ |
| **SRS 6.0+** | ✅ | ✅ |
| **Cloudflare Stream** | ✅ | ✅ |
| **Mux** | ✅ | ✅ |
| **Twitch** | 实验性 | - |
| **FFmpeg** | 开发中 | 开发中 |

---

## 八、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **WHIP** | HTTP POST 交换 SDP，标准化 WebRTC 推流信令 |
| **WHEP** | WHIP 的拉流版本，同样基于 HTTP |
| **优势** | 替代私有信令；走 HTTPS，防火墙友好；强制加密 |
| **实现** | pion/webrtc + Gin 可快速搭建 WHIP 服务器 |
| **生态** | OBS 30+、MediaMTX、SRS 已支持 |

WHIP/WHEP 代表 WebRTC 流媒体的标准化趋势，预计 2026-2028 年会成为低延迟直播的主流推流方式。

---

## 模块 1 总结

完成了流媒体协议基础的 13 篇：

| 协议 | 延迟 | 主要用途 |
|------|------|---------|
| RTP/RTCP | 50-200ms | 传输层基础 |
| RTSP | 100-500ms | 摄像头拉流 |
| RTMP | 1-3s | 直播推流 |
| SRT | 0.1-2s | 弱网推流 |
| HLS | 6-30s | CDN 分发 |
| DASH/CMAF | 6-30s | 全球分发 |
| WebRTC | 50-400ms | 实时互动 |
| WHIP/WHEP | 100-500ms | 标准化 WebRTC 推拉流 |

---

## 下一篇

下一篇进入**模块 2：工程实践与优化**，讲解主流**流媒体服务器选型对比**：SRS、MediaMTX、Nginx-RTMP、Ant Media 的架构特点与适用场景。

---

## 参考资料

1. [WHIP RFC Draft - IETF](https://datatracker.ietf.org/doc/html/draft-ietf-wish-whip)
2. [WHEP RFC Draft - IETF](https://datatracker.ietf.org/doc/html/draft-ietf-wish-whep)
3. [OBS WHIP 文档](https://obsproject.com/kb/whip-guide)
4. [MediaMTX WHIP/WHEP](https://github.com/bluenviron/mediamtx#whip)
5. [SRS WHIP 文档](https://ossrs.net/lts/zh-cn/docs/v6/doc/whip)

---

## 动手实践

### 练习1：用 OBS WHIP 推流到 MediaMTX
```bash
# 1. 启动 MediaMTX
docker run -d -p 8889:8889 -p 8554:8554 bluenviron/mediamtx

# 2. OBS 设置：推流 → 服务：WHIP → 服务器：http://localhost:8889/live/stream/whip
# 或使用 FFmpeg 推 WHIP（需 FFmpeg 7.0+）
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -c:a opus -b:a 64k \
  -f whip http://localhost:8889/live/stream/whip
```
**预期输出**：MediaMTX 日志显示 "WHIP session created"，WebRTC 连接建立成功。

### 练习2：用 WHEP 在浏览器拉流
```javascript
// WHEP 客户端（原生 fetch + RTCPeerConnection）
const pc = new RTCPeerConnection({ iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] });
pc.addTransceiver('video', { direction: 'recvonly' });
pc.addTransceiver('audio', { direction: 'recvonly' });

const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

const resp = await fetch('http://localhost:8889/live/stream/whep', {
  method: 'POST',
  headers: { 'Content-Type': 'application/sdp' },
  body: offer.sdp,
});
const answer = await resp.text();
await pc.setRemoteDescription({ type: 'answer', sdp: answer });
```
**预期输出**：浏览器 `<video>` 元素开始播放直播流，延迟 < 500ms。

### 练习3：测试 Bearer Token 鉴权
```bash
# 带 Token 推流
ffmpeg ... -f whip -auth_token "mytoken123" http://localhost:8889/live/secret/whip

# MediaMTX 配置鉴权
# paths:
#   secret:
#     publishPermissions: [{action: publish, query: token=mytoken123}]
```
**预期输出**：正确 token 推流成功，错误 token 返回 401 Unauthorized。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| POST /whip 返回 404 | 服务器未开启 WHIP 支持 | 检查 MediaMTX/SRS 的 `webrtc: yes` 配置 |
| ICE 连接失败 | 服务器无公网 IP，需要 TURN | 配置 `iceServers` 包含 TURN，或确保 UDP 端口可达 |
| 401 Unauthorized | Bearer Token 不匹配 | 检查推流端和服务端 token 配置是否一致 |
| 浏览器拉流无视频 | WHEP 没有正确添加 transceiver | 确保 `addTransceiver('video', {direction:'recvonly'})` 在 createOffer 前执行 |
| OBS 断线重连卡住 | WHIP session 未正确清理 | 服务端配置合理的 session 超时时间 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| WHIP | WebRTC-HTTP Ingestion Protocol | 基于 HTTP POST 的 WebRTC 推流标准（RFC 草案）|
| WHEP | WebRTC-HTTP Egress Protocol | 基于 HTTP POST 的 WebRTC 拉流标准 |
| Bearer Token | Bearer Token | HTTP Authorization 头中的认证令牌 |
| ICE Candidate | ICE Candidate | ICE 收集到的连接候选地址（host/srflx/relay）|
| Transceiver | RTCRtpTransceiver | WebRTC 收发器，控制媒体流方向（send/recv/both）|
| SDP Body | SDP Body | WHIP/WHEP HTTP 请求/响应体中的 SDP 内容 |
| 201 Created | 201 Created | WHIP 成功响应状态码，Location 头含会话 URL |

---

> **全文完** | 上一篇：[第 12 篇 — WebRTC 协议栈详解](12-WebRTC协议栈详解.md) | 下一篇：[第 14 篇 — 流媒体服务器选型对比](../02-工程实践优化/14-流媒体服务器选型对比.md)
