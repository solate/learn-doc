# WebRTC 协议栈详解

> **适合读者**：后端开发者、对实时通信感兴趣的工程师  
> **前置知识**：[第 05 篇：传输层协议 TCP/UDP/QUIC](05-传输层协议TCP-UDP-QUIC.md)、[第 06 篇：RTP/RTCP](06-RTP-RTCP实时传输协议详解.md)

---

## 一、WebRTC 是什么

**WebRTC（Web Real-Time Communication）** 是 Google 2011 年开源并推动标准化的实时通信框架，现已成为 W3C + IETF 双重标准。

**核心能力**：浏览器之间无需服务器中转，直接传输音视频和数据，延迟 100ms 以内。

```
传统直播：编码器 → RTMP服务器 → CDN → 播放器（延迟 5-30s）
WebRTC：  浏览器A ←→ 浏览器B（延迟 50-150ms，点对点）
```

### 1.1 WebRTC 的应用场景

| 场景 | 代表产品 |
|------|---------|
| 视频会议 | Google Meet、Zoom（部分）、Teams |
| 直播互动 | 连麦、PK、实时连线 |
| 实时监控 | 摄像头实时预览（延迟 <200ms） |
| 在线教育 | 双师课堂、互动白板 |
| 游戏串流 | Stadia（部分） |

### 1.2 WebRTC 协议栈全景

```
应用层
  ├── getUserMedia（摄像头/麦克风采集）
  ├── RTCPeerConnection（点对点连接）
  └── RTCDataChannel（数据传输）

信令层（非 WebRTC 标准，由应用自定义）
  └── WebSocket / HTTP（交换 SDP 和 ICE 候选）

传输层
  ├── ICE（Interactive Connectivity Establishment）穿透
  ├── DTLS（加密握手）
  ├── SRTP（加密 RTP，传输媒体）
  └── SCTP（传输 DataChannel 数据）

网络层
  └── UDP（主要）/ TCP（备选）
```

---

## 二、ICE：NAT 穿透的核心

WebRTC 最复杂的部分是 **ICE（交互式连接建立）**——解决两台设备在 NAT 后面如何互相连接。

### 2.1 为什么需要 ICE

```
家庭网络：
  你的电脑：192.168.1.100（内网 IP）
  路由器：  公网 IP 1.2.3.4
  
对方电脑：192.168.2.100（内网 IP）
对方路由器：公网 IP 5.6.7.8

问题：192.168.1.100 无法直接连接 192.168.2.100
```

### 2.2 三种连接候选（ICE Candidate）

**ICE Candidate 类型**：

```
1. Host Candidate（本地地址）
   192.168.1.100:9000（内网直连，LAN 内有效）

2. Server Reflexive Candidate（通过 STUN 获取公网映射地址）
   1.2.3.4:45678（路由器 NAT 后的公网地址）

3. Relay Candidate（通过 TURN 中继，100% 成功）
   turn-server:3478（中继服务器地址）
```

**优先级**：Host > Server Reflexive > Relay  
（优先直连，避免中继浪费带宽）

### 2.3 STUN 工作原理

**STUN（Session Traversal Utilities for NAT）** 帮助设备发现自己的公网地址：

```
你的设备                STUN 服务器
    |                        |
    |-- STUN Binding Request -->|
    |   （发送 UDP 包）        |
    |                        |
    |<-- STUN Binding Response--|
    |   返回：1.2.3.4:45678   |
    |   （你在公网看到的地址） |
```

STUN 是免费的：`stun:stun.l.google.com:19302`（Google 公共 STUN）

**STUN 的局限**：只能穿透"锥形 NAT"，无法穿透"对称 NAT"。

### 2.4 TURN 中继

**TURN（Traversal Using Relays around NAT）** 在 STUN 失败时提供中继：

```
客户端 A  ──→  TURN 服务器  ←──  客户端 B
              中继转发所有流量
```

**TURN 的代价**：所有流量都经过服务器，带宽成本高，延迟增加约 20-50ms。

**穿透成功率统计**（业界数据）：
```
Host（局域网直连）：~20%
Server Reflexive（STUN）：~60%
Relay（TURN）：~20%（最后保障）
总体成功率：~99.5%
```

### 2.5 ICE 连接建立流程

```
1. 收集候选地址
   ├── Host：本机网卡地址
   ├── Server Reflexive：STUN 查询公网地址
   └── Relay：TURN 分配中继地址

2. 通过信令服务器交换候选地址

3. 连接性检查（ICE Connectivity Checks）
   对所有候选地址对进行 STUN 连通性测试
   选择最优路径

4. 确定最终传输路径
```

---

## 三、信令（Signaling）

WebRTC 自身不规定信令协议，由应用层实现（通常用 WebSocket）。

### 3.1 SDP：描述媒体能力

**SDP（Session Description Protocol）** 是一种文本格式，描述音视频能力：

```sdp
v=0
o=- 4611731400430051336 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1

m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp-mux
a=rtpmap:96 VP8/90000        ← 支持 VP8 视频编码
a=rtpmap:97 H264/90000       ← 支持 H264 视频编码
a=fmtp:97 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=ssrc:3735928559 cname:stream1
a=ice-ufrag:xyz1
a=ice-pwd:secretpassword123456789012

m=audio 9 UDP/TLS/RTP/SAVPF 111 103
a=rtpmap:111 opus/48000/2    ← Opus 音频编码
a=rtpmap:103 ISAC/16000
```

### 3.2 Offer/Answer 协商流程

```
调用方（Caller）                    被调方（Callee）
     |                                    |
     |-- createOffer() ──────────────────>|  (通过信令服务器)
     |   SDP Offer：我支持 VP8/H264       |
     |                                    |
     |<── createAnswer() ─────────────────|
     |   SDP Answer：我选择 H264          |
     |                                    |
     |-- ICE Candidates ─────────────────>|
     |<── ICE Candidates ─────────────────|
     |                                    |
     |===== DTLS 握手 ===================|
     |===== SRTP 流传输 =================|
```

### 3.3 信令服务器实现（Go）

```go
// 简单的 WebSocket 信令服务器
type SignalingServer struct {
    rooms sync.Map // roomID -> []*websocket.Conn
}

func (s *SignalingServer) HandleWS(c *gin.Context) {
    roomID := c.Query("room")
    
    ws, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return
    }
    defer ws.Close()
    
    s.joinRoom(roomID, ws)
    defer s.leaveRoom(roomID, ws)
    
    for {
        _, msg, err := ws.ReadMessage()
        if err != nil {
            break
        }
        // 转发信令消息给同房间的其他人
        s.broadcast(roomID, ws, msg)
    }
}

func (s *SignalingServer) broadcast(roomID string, sender *websocket.Conn, msg []byte) {
    if conns, ok := s.rooms.Load(roomID); ok {
        for _, conn := range conns.([]*websocket.Conn) {
            if conn != sender {
                conn.WriteMessage(websocket.TextMessage, msg)
            }
        }
    }
}
```

---

## 四、DTLS 与 SRTP

### 4.1 DTLS 握手

**DTLS（Datagram TLS）** 是在 UDP 上实现的 TLS，用于 WebRTC 的加密握手：

```
WebRTC 强制加密，所有媒体必须通过 SRTP 传输
DTLS 的作用：协商加密密钥（类似 TLS 握手）

DTLS 握手完成后：
- 双方获得 SRTP 密钥
- 后续媒体通过 SRTP 加密传输
```

**指纹验证**：SDP 中包含 DTLS 证书指纹，防止中间人攻击：
```
a=fingerprint:sha-256 AA:BB:CC:...（证书 SHA-256 指纹）
a=setup:actpass（谁发起 DTLS 握手）
```

### 4.2 SRTP（安全 RTP）

WebRTC 使用 SRTP 而非普通 RTP，数据包在 UDP 中传输但已加密：

```
SRTP 包结构：
[RTP Header（明文）][Payload（AES-128-CM 加密）][HMAC-SHA1（认证）]
```

---

## 五、媒体传输与 Jitter Buffer

### 5.1 抖动缓冲（Jitter Buffer）

网络延迟是不均匀的，到达时间有抖动：

```
发送：  Pkt1 → Pkt2 → Pkt3 → Pkt4
接收：  Pkt1(10ms) Pkt3(15ms) Pkt2(25ms) Pkt4(11ms)
                  ↑（乱序到达）

Jitter Buffer 重新排序：
  缓存等待 → 按序号排列 → 定时输出给解码器
  代价：引入约 20-80ms 额外延迟
```

### 5.2 FEC（前向纠错）

WebRTC 在高丢包场景用 FEC 减少重传：

```
发送 5 个包时额外发送 1 个冗余包：
  原始：Pkt1 Pkt2 Pkt3 Pkt4 Pkt5
  冗余：FEC = Pkt1 XOR Pkt2（如果 Pkt1 丢失，可从 Pkt2+FEC 恢复）

ULP-FEC（WebRTC 标准方式）：
  丢包恢复无需重传，延迟不增加
  代价：带宽增加约 5-20%
```

### 5.3 自适应码率（REMB/TWCC）

WebRTC 通过 RTCP 反馈实时调整码率：

- **REMB（Receiver Estimated Maximum Bitrate）**：接收方告知发送方最大可用带宽
- **TWCC（Transport Wide Congestion Control）**：更精准的拥塞控制

```
接收方：
  测量包间延迟，估算带宽
  发送 RTCP REMB/TWCC 反馈

发送方：
  根据反馈，通过 GCC 算法调整：
  带宽充裕 → 升码率
  出现拥塞 → 降码率（GoogCC 算法）
```

---

## 六、WebRTC SFU 架构

### 6.1 为什么需要 SFU

纯 P2P 的 WebRTC 对多人场景不适用：

```
4 人会议（P2P）：
  每个人需要发 3 路上行 + 接收 3 路下行
  总共 12 路视频流，CPU 和带宽压力极大

4 人会议（SFU）：
  每个人只发 1 路上行到 SFU
  SFU 分发给其他 3 人
  总共 4 路上行 + 12 路下行（下行可优化）
```

### 6.2 SFU vs MCU

| 特性 | SFU（选择性转发单元） | MCU（多点控制单元） |
|------|--------------------|--------------------|
| 服务器处理 | 只转发，不混流 | 解码 + 混流 + 编码 |
| 延迟 | 低（无转码） | 高（需转码） |
| CPU 消耗 | 低 | 极高 |
| 带宽（客户端下行）| 多路流（每路独立）| 单路混合流 |
| 布局灵活性 | 客户端自己排布 | 服务端决定布局 |
| 代表产品 | mediasoup、Janus | OpenMCU |

### 6.3 SFU 关键功能

**Simulcast（联播）**：发送方同时发 3 种分辨率：
```
发送：高清(1080p) + 中清(360p) + 低清(180p)
SFU 根据接收方带宽，选择转发哪路
```

**SVC（可伸缩视频编码）**：
```
单流包含多个空间层和时间层
SFU 按需丢弃高层，无需客户端多路发送
```

---

## 七、浏览器 WebRTC API 实战

### 7.1 RTCPeerConnection 核心 API

**RTCPeerConnection** 是 WebRTC 的核心 API，封装了整个点对点连接的建立、媒体传输、数据通道等功能。

**基本使用流程**：

```javascript
// 1. 创建 PeerConnection 实例
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { 
      urls: 'turn:turn.example.com:3478',
      username: 'user',
      credential: 'pass'
    }
  ]
});

// 2. 添加本地媒体轨道（发送端）
const stream = await navigator.mediaDevices.getUserMedia({
  video: true,
  audio: true
});
stream.getTracks().forEach(track => {
  pc.addTrack(track, stream);
});

// 3. 接收远端媒体轨道
pc.ontrack = (event) => {
  const remoteVideo = document.getElementById('remoteVideo');
  remoteVideo.srcObject = event.streams[0];
};

// 4. ICE 候选收集
pc.onicecandidate = (event) => {
  if (event.candidate) {
    // 通过信令服务器发送给对方
    signalingChannel.send({
      type: 'ice-candidate',
      candidate: event.candidate
    });
  }
};

// 5. 连接状态监控
pc.oniceconnectionstatechange = () => {
  console.log('ICE 连接状态:', pc.iceConnectionState);
};

pc.onconnectionstatechange = () => {
  console.log('连接状态:', pc.connectionState);
};
```

### 7.2 getUserMedia 采集约束详解

**getUserMedia** 用于获取摄像头和麦克风权限，通过约束参数控制采集质量。

**完整约束示例**：

```javascript
const constraints = {
  video: {
    width: { min: 640, ideal: 1280, max: 1920 },
    height: { min: 480, ideal: 720, max: 1080 },
    frameRate: { min: 15, ideal: 30, max: 60 },
    facingMode: 'user',  // 'user' = 前置, 'environment' = 后置
    aspectRatio: 16/9,
  },
  audio: {
    echoCancellation: true,       // 回声消除
    noiseSuppression: true,       // 降噪
    autoGainControl: true,        // 自动增益
    sampleRate: 48000,            // 采样率
    channelCount: 2,              // 立体声
  }
};

try {
  const stream = await navigator.mediaDevices.getUserMedia(constraints);
  console.log('获取到的媒体轨道:', stream.getTracks());
  
  // 将流绑定到 video 元素
  document.getElementById('localVideo').srcObject = stream;
} catch (err) {
  console.error('无法获取媒体设备:', err);
  
  // 常见错误处理
  if (err.name === 'NotAllowedError') {
    alert('用户拒绝了摄像头/麦克风权限');
  } else if (err.name === 'NotFoundError') {
    alert('未找到摄像头或麦克风设备');
  } else if (err.name === 'OverconstrainedError') {
    alert('约束条件过于严格，设备不支持');
  }
}
```

**设备选择（切换摄像头）**：

```javascript
// 1. 枚举所有设备
const devices = await navigator.mediaDevices.enumerateDevices();
const videoDevices = devices.filter(d => d.kind === 'videoinput');

console.log('可用摄像头:', videoDevices);
// 输出示例：
// [
//   { deviceId: 'abc123', label: 'FaceTime HD Camera', kind: 'videoinput' },
//   { deviceId: 'def456', label: 'USB Camera', kind: 'videoinput' }
// ]

// 2. 选择特定设备
const stream = await navigator.mediaDevices.getUserMedia({
  video: { deviceId: { exact: videoDevices[1].deviceId } },
  audio: true
});
```

### 7.3 完整的 1 对 1 通话实现

**发起方（Caller）完整代码**：

```javascript
class WebRTCCaller {
  constructor(signalingChannel) {
    this.signalingChannel = signalingChannel;
    this.pc = null;
  }

  async start() {
    // 1. 创建 PeerConnection
    this.pc = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
      ]
    });

    // 2. 获取本地媒体流
    const localStream = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: true
    });

    // 3. 添加本地轨道到 PeerConnection
    localStream.getTracks().forEach(track => {
      this.pc.addTrack(track, localStream);
    });

    // 4. 显示本地视频
    document.getElementById('localVideo').srcObject = localStream;

    // 5. 接收远端视频
    this.pc.ontrack = (event) => {
      document.getElementById('remoteVideo').srcObject = event.streams[0];
    };

    // 6. 发送 ICE 候选
    this.pc.onicecandidate = (event) => {
      if (event.candidate) {
        this.signalingChannel.send({
          type: 'ice-candidate',
          candidate: event.candidate
        });
      }
    };

    // 7. 创建 Offer
    const offer = await this.pc.createOffer();
    await this.pc.setLocalDescription(offer);

    // 8. 发送 Offer 给对方
    this.signalingChannel.send({
      type: 'offer',
      sdp: offer.sdp
    });

    // 9. 监听信令消息
    this.signalingChannel.onmessage = async (msg) => {
      if (msg.type === 'answer') {
        // 收到 Answer，设置远端描述
        await this.pc.setRemoteDescription({
          type: 'answer',
          sdp: msg.sdp
        });
      } else if (msg.type === 'ice-candidate') {
        // 收到对方的 ICE 候选
        await this.pc.addIceCandidate(msg.candidate);
      }
    };
  }

  close() {
    if (this.pc) {
      this.pc.close();
      this.pc = null;
    }
  }
}
```

**接收方（Callee）完整代码**：

```javascript
class WebRTCCallee {
  constructor(signalingChannel) {
    this.signalingChannel = signalingChannel;
    this.pc = null;
  }

  async start() {
    // 1. 创建 PeerConnection
    this.pc = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
      ]
    });

    // 2. 获取本地媒体流
    const localStream = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: true
    });

    // 3. 添加本地轨道
    localStream.getTracks().forEach(track => {
      this.pc.addTrack(track, localStream);
    });

    // 4. 显示本地视频
    document.getElementById('localVideo').srcObject = localStream;

    // 5. 接收远端视频
    this.pc.ontrack = (event) => {
      document.getElementById('remoteVideo').srcObject = event.streams[0];
    };

    // 6. 发送 ICE 候选
    this.pc.onicecandidate = (event) => {
      if (event.candidate) {
        this.signalingChannel.send({
          type: 'ice-candidate',
          candidate: event.candidate
        });
      }
    };

    // 7. 监听信令消息
    this.signalingChannel.onmessage = async (msg) => {
      if (msg.type === 'offer') {
        // 收到 Offer，设置远端描述
        await this.pc.setRemoteDescription({
          type: 'offer',
          sdp: msg.sdp
        });

        // 创建 Answer
        const answer = await this.pc.createAnswer();
        await this.pc.setLocalDescription(answer);

        // 发送 Answer 给对方
        this.signalingChannel.send({
          type: 'answer',
          sdp: answer.sdp
        });
      } else if (msg.type === 'ice-candidate') {
        // 收到对方的 ICE 候选
        await this.pc.addIceCandidate(msg.candidate);
      }
    };
  }

  close() {
    if (this.pc) {
      this.pc.close();
      this.pc = null;
    }
  }
}
```

### 7.4 ICE 连接状态监控与错误处理

**连接状态完整生命周期**：

```javascript
const pc = new RTCPeerConnection(config);

// ICE 连接状态（关键）
pc.oniceconnectionstatechange = () => {
  console.log('ICE 状态:', pc.iceConnectionState);
  
  switch (pc.iceConnectionState) {
    case 'new':
      // 初始状态，尚未开始 ICE 协商
      console.log('ICE 初始化');
      break;
    
    case 'checking':
      // 正在检查 ICE 候选，尝试连接
      console.log('正在尝试连接...');
      showLoadingIndicator();
      break;
    
    case 'connected':
      // 至少找到一对可用的候选，连接成功
      console.log('连接成功！');
      hideLoadingIndicator();
      break;
    
    case 'completed':
      // ICE 协商完成，找到最优候选对
      console.log('ICE 协商完成');
      break;
    
    case 'failed':
      // ICE 协商失败，所有候选都不可达
      console.error('连接失败');
      showError('无法建立连接，请检查网络或 TURN 服务器');
      break;
    
    case 'disconnected':
      // 临时断开，可能恢复
      console.warn('连接断开，尝试重连...');
      showReconnectingIndicator();
      break;
    
    case 'closed':
      // 连接已关闭
      console.log('连接已关闭');
      break;
  }
};

// 整体连接状态（更高层）
pc.onconnectionstatechange = () => {
  console.log('连接状态:', pc.connectionState);
  // 可能值：new, connecting, connected, disconnected, failed, closed
};

// ICE 候选收集状态
pc.onicegatheringstatechange = () => {
  console.log('ICE 收集状态:', pc.iceGatheringState);
  // 可能值：new, gathering, complete
};

// 信令状态（SDP 交换状态）
pc.onsignalingstatechange = () => {
  console.log('信令状态:', pc.signalingState);
  // 可能值：stable, have-local-offer, have-remote-offer, 
  //        have-local-pranswer, have-remote-pranswer, closed
};
```

**常见错误处理**：

```javascript
// 错误 1：ICE 连接失败（NAT 穿透失败）
pc.oniceconnectionstatechange = () => {
  if (pc.iceConnectionState === 'failed') {
    console.error('ICE 连接失败，可能原因：');
    console.error('1. 双方都在严格 NAT 后面，需要 TURN 服务器');
    console.error('2. TURN 服务器配置错误或不可达');
    console.error('3. 防火墙阻止了 UDP 流量');
    
    // 尝试重启 ICE
    pc.restartIce();
  }
};

// 错误 2：远端轨道添加失败
pc.ontrack = (event) => {
  if (!event.streams || event.streams.length === 0) {
    console.error('远端轨道没有关联流');
    return;
  }
  
  const remoteVideo = document.getElementById('remoteVideo');
  remoteVideo.srcObject = event.streams[0];
  
  // 确保视频自动播放
  remoteVideo.play().catch(err => {
    console.error('自动播放失败:', err);
    // 浏览器自动播放策略阻止，需要用户交互
    showPlayButton();
  });
};

// 错误 3：协商失败（SDP 不兼容）
pc.onnegotiationneeded = async () => {
  try {
    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    // 发送 offer...
  } catch (err) {
    console.error('创建 Offer 失败:', err);
    // 可能原因：没有添加任何轨道
  }
};
```

### 7.5 WHIP/WHEP 集成示例（与 SRS/MediaMTX 对接）

**WHIP（WebRTC-HTTP Ingestion Protocol）**：浏览器推流到服务器。

**推流端（WHIP）代码**：

```javascript
async function publishToWHIP(whipUrl) {
  // 1. 创建 PeerConnection
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  // 2. 获取本地媒体流
  const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: 1280, height: 720 },
    audio: true
  });

  // 3. 添加轨道
  stream.getTracks().forEach(track => {
    pc.addTrack(track, stream);
  });

  // 4. 创建 Offer
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  // 5. 发送 Offer 到 WHIP 端点
  const response = await fetch(whipUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/sdp',
    },
    body: offer.sdp,
  });

  if (!response.ok) {
    throw new Error(`WHIP 失败: ${response.status}`);
  }

  // 6. 获取 Answer
  const answerSdp = await response.text();
  await pc.setRemoteDescription({
    type: 'answer',
    sdp: answerSdp
  });

  // 7. 从 Link header 获取 ICE 服务器（可选）
  const linkHeader = response.headers.get('Link');
  if (linkHeader) {
    // 解析 Link header 中的 ICE 服务器
    console.log('服务器提供的 ICE 服务器:', linkHeader);
  }

  console.log('推流成功！');
  return pc;
}

// 使用示例
const whipUrl = 'http://localhost:8889/stream1/whip';
const pc = await publishToWHIP(whipUrl);
```

**WHEP（WebRTC-HTTP Egress Protocol）**：浏览器从服务器拉流。

**拉流端（WHEP）代码**：

```javascript
async function playFromWHEP(whepUrl) {
  // 1. 创建 PeerConnection
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  // 2. 添加接收器（告诉服务器我们要接收音视频）
  pc.addTransceiver('video', { direction: 'recvonly' });
  pc.addTransceiver('audio', { direction: 'recvonly' });

  // 3. 创建 Offer
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  // 4. 发送 Offer 到 WHEP 端点
  const response = await fetch(whepUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/sdp',
    },
    body: offer.sdp,
  });

  if (!response.ok) {
    throw new Error(`WHEP 失败: ${response.status}`);
  }

  // 5. 获取 Answer
  const answerSdp = await response.text();
  await pc.setRemoteDescription({
    type: 'answer',
    sdp: answerSdp
  });

  // 6. 接收远端媒体流
  return new Promise((resolve) => {
    pc.ontrack = (event) => {
      const remoteVideo = document.getElementById('remoteVideo');
      remoteVideo.srcObject = event.streams[0];
      resolve(pc);
    };
  });
}

// 使用示例
const whepUrl = 'http://localhost:8889/stream1/whep';
const pc = await playFromWHEP(whepUrl);
```

**MediaMTX 配置（支持 WHIP/WHEP）**：

```yaml
# mediamtx.yml
paths:
  stream1:
    # 允许 WebRTC 推流和拉流
    webrtcAddress: :8889
    
  camera1:
    # RTSP 摄像头作为源
    source: rtsp://admin:pass@192.168.1.100/stream
    webrtcAddress: :8889
```

**SRS 配置（支持 WebRTC）**：

```nginx
# srs.conf
vhost __defaultVhost__ {
    # 启用 WebRTC
    rtc {
        enabled     on;
        listen      8000;  # WebRTC 信令端口
        candidate   $CANDIDATE;  # 公网 IP 或域名
    }
}

# 浏览器访问 WebRTC 播放页面
# http://localhost:8080/players/rtc_player.html?stream=stream1
```

### 7.6 实战案例：监控摄像头 WHEP 播放

**完整的监控播放器**：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>WebRTC 监控播放器</title>
</head>
<body>
  <h1>实时监控</h1>
  <video id="video" autoplay muted style="width: 100%; max-width: 800px;"></video>
  <div id="status">状态：未连接</div>
  <button id="playBtn">开始播放</button>
  <button id="stopBtn">停止</button>

  <script>
    let pc = null;
    const whepUrl = 'http://localhost:8889/camera1/whep';

    document.getElementById('playBtn').onclick = async () => {
      try {
        updateStatus('正在连接...');
        
        // 创建 PeerConnection
        pc = new RTCPeerConnection({
          iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
        });

        // 监听 ICE 状态
        pc.oniceconnectionstatechange = () => {
          updateStatus(`ICE 状态: ${pc.iceConnectionState}`);
          if (pc.iceConnectionState === 'connected') {
            updateStatus('连接成功！延迟 < 500ms');
          } else if (pc.iceConnectionState === 'failed') {
            updateStatus('连接失败，请检查网络');
          }
        };

        // 添加接收器
        pc.addTransceiver('video', { direction: 'recvonly' });
        pc.addTransceiver('audio', { direction: 'recvonly' });

        // 创建 Offer
        const offer = await pc.createOffer();
        await pc.setLocalDescription(offer);

        // 发送到 WHEP 端点
        const response = await fetch(whepUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/sdp' },
          body: offer.sdp,
        });

        if (!response.ok) {
          throw new Error(`WHEP 失败: ${response.status}`);
        }

        // 设置 Answer
        const answerSdp = await response.text();
        await pc.setRemoteDescription({ type: 'answer', sdp: answerSdp });

        // 接收视频
        pc.ontrack = (event) => {
          document.getElementById('video').srcObject = event.streams[0];
        };

      } catch (err) {
        updateStatus(`错误: ${err.message}`);
      }
    };

    document.getElementById('stopBtn').onclick = () => {
      if (pc) {
        pc.close();
        pc = null;
        document.getElementById('video').srcObject = null;
        updateStatus('已停止');
      }
    };

    function updateStatus(msg) {
      document.getElementById('status').textContent = `状态：${msg}`;
    }
  </script>
</body>
</html>
```

---

## 八、Go 项目集成 WebRTC

### 7.1 使用 pion/webrtc

**pion/webrtc** 是 Go 最主流的 WebRTC 实现：

```go
package main

import (
    "github.com/pion/webrtc/v3"
)

func handleOffer(offerSDP string) (string, error) {
    // 创建 PeerConnection
    config := webrtc.Configuration{
        ICEServers: []webrtc.ICEServer{
            {URLs: []string{"stun:stun.l.google.com:19302"}},
            {
                URLs:       []string{"turn:turn.example.com:3478"},
                Username:   "user",
                Credential: "password",
            },
        },
    }
    pc, err := webrtc.NewPeerConnection(config)
    if err != nil {
        return "", err
    }

    // 处理接收到的视频轨道
    pc.OnTrack(func(track *webrtc.TrackRemote, _ *webrtc.RTPReceiver) {
        for {
            pkt, _, err := track.ReadRTP()
            if err != nil {
                return
            }
            // 处理 RTP 包：转码、存储、转发...
            _ = pkt
        }
    })

    // 设置 Offer
    offer := webrtc.SessionDescription{
        Type: webrtc.SDPTypeOffer,
        SDP:  offerSDP,
    }
    if err = pc.SetRemoteDescription(offer); err != nil {
        return "", err
    }

    // 创建 Answer
    answer, err := pc.CreateAnswer(nil)
    if err != nil {
        return "", err
    }

    // ICE 收集完成后再返回
    gatherComplete := webrtc.GatheringCompletePromise(pc)
    if err = pc.SetLocalDescription(answer); err != nil {
        return "", err
    }
    <-gatherComplete

    return pc.LocalDescription().SDP, nil
}
```

### 7.2 WebRTC → RTMP 转推

监控场景常见需求：浏览器 WebRTC 推流 → 服务器转成 RTMP 推到直播平台：

```go
// WebRTC 接收，通过 FFmpeg 管道推 RTMP
func transcodeToRTMP(track *webrtc.TrackRemote, rtmpURL string) {
    // 启动 FFmpeg，从 stdin 读取 RTP，推到 RTMP
    cmd := exec.Command("ffmpeg",
        "-f", "rtp", "-i", "pipe:0",
        "-c:v", "libx264", "-preset", "veryfast",
        "-c:a", "aac",
        "-f", "flv", rtmpURL,
    )
    
    stdin, _ := cmd.StdinPipe()
    cmd.Start()
    
    for {
        pkt, _, err := track.ReadRTP()
        if err != nil {
            return
        }
        b, _ := pkt.Marshal()
        stdin.Write(b)
    }
}
```

---

## 八、WebRTC 延迟分析

### 8.1 延迟组成

```
采集延迟：摄像头帧缓存 ~ 10-30ms
编码延迟：H264 real-time ~ 20-50ms
ICE 建立：首次连接 ~ 200-500ms（之后复用）
网络传输：取决于距离 ~ 5-200ms
Jitter Buffer：~ 20-80ms
解码延迟：~ 10-30ms

总典型延迟：~ 100-400ms
```

### 8.2 与其他协议延迟对比

```
WebRTC：100-400ms（P2P）
SRT：   300-2000ms（可配置）
RTMP：  1000-3000ms
HLS：   6000-30000ms
LL-HLS：2000-4000ms
```

---

## 九、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **ICE** | 自动发现最优路径（Host > STUN > TURN），保证穿透 |
| **信令** | SDP offer/answer 协商媒体能力，由应用自定义（WebSocket） |
| **DTLS+SRTP** | 强制加密，没有明文 WebRTC 流 |
| **Jitter Buffer** | 重新排序抖动包，代价是约 20-80ms 延迟 |
| **SFU** | 多人场景服务端转发，比 P2P 节省客户端带宽和 CPU |
| **pion** | Go 的 WebRTC 实现，支持 SFU 构建 |

---

## 下一篇

下一篇讲解 **WHIP/WHEP**：WebRTC 推流（WHIP）和拉流（WHEP）的标准化协议，告别信令自定义的混乱，以及如何用 Go 实现 WHIP/WHEP 服务器。

---

## 参考资料

1. [WebRTC 标准 - W3C](https://www.w3.org/TR/webrtc/)
2. [ICE 标准 - RFC 8445](https://datatracker.ietf.org/doc/html/rfc8445)
3. [pion/webrtc - GitHub](https://github.com/pion/webrtc)
4. [mediasoup - SFU 框架](https://mediasoup.org/)
5. [WebRTC for the Curious（免费书）](https://webrtcforthecurious.com/)

---

## 动手实践

### 练习1：部署 coturn STUN/TURN 服务器
```bash
docker run -d --name coturn \
  --network host \
  coturn/coturn \
  -n --log-file=stdout \
  --min-port=49152 --max-port=65535 \
  --lt-cred-mech \
  --fingerprint \
  --realm=example.com \
  --user=test:password123
```
**预期输出**：`turnutils_uclient -T -u test -w password123 <server>` 返回 RELAY 地址，证明 TURN 中继可用。

### 练习2：用 pion/webrtc 做回环连接测试
```go
// 创建两个 PeerConnection，模拟本地 Offer/Answer 交换
pc1, _ := webrtc.NewPeerConnection(webrtc.Configuration{})
pc2, _ := webrtc.NewPeerConnection(webrtc.Configuration{})
// 收集并交换 ICE candidate ...
// 最终两端 ICE state 变为 connected
```
**预期输出**：两端 ICE 连接状态变为 `connected`，数据通道可互传消息。

### 练习3：抓取 WebRTC 连接统计
```javascript
// 浏览器控制台获取实时统计
const pc = ...; // 已建立的 PeerConnection
const stats = await pc.getStats();
stats.forEach(report => {
  if (report.type === 'inbound-rtp' && report.kind === 'video') {
    console.log('jitter:', report.jitter, 'packetsLost:', report.packetsLost);
  }
});
```
**预期输出**：看到 `jitter < 50ms`，`framesDecoded` 持续增长，`packetsLost` 保持低值。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| ICE 卡在 checking 状态 | 双方无法直连，缺少 TURN 服务器 | 配置 TURN 服务器并验证端口可达 |
| DTLS 握手失败 | 证书指纹不匹配或 UDP 被防火墙拦截 | 检查防火墙 UDP 端口范围，确认 DTLS 证书 |
| 建立连接后黑屏 | 视频编码器参数不匹配（H.264 profile）| 检查 SDP 中 `profile-level-id`，对齐双端配置 |
| 音频回声 | 本端扬声器声音被麦克风拾取 | 开启 `echoCancellation: true` |
| 连接延迟突然增大 | TURN 中继带宽不足，负载过高 | 监控 TURN 服务器带宽，增加节点 |
| 移动端断网重连失败 | ICE restart 未实现 | 检测连接中断后触发 `restartIce()` |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| ICE | Interactive Connectivity Establishment | 穿透 NAT 的连接建立框架（RFC 8445）|
| STUN | Session Traversal Utilities for NAT | 获取公网 IP/Port 的协议 |
| TURN | Traversal Using Relays around NAT | 无法直连时通过中继服务器转发数据 |
| DTLS | Datagram TLS | 基于 UDP 的 TLS，WebRTC 使用它加密媒体通道 |
| SRTP | Secure RTP | 加密版 RTP，WebRTC 媒体数据的传输格式 |
| SDP | Session Description Protocol | 协商媒体参数（编解码/ICE 候选）的文本格式 |
| Offer/Answer | Offer/Answer | WebRTC 信令模型：一端发 Offer，另一端回 Answer |
| Signaling | Signaling | 信令通道，负责传递 SDP 和 ICE candidate（WebRTC 不内置）|
| Trickle ICE | Trickle ICE | 边收集边发送 ICE candidate，加快连接速度 |

---

> **全文完** | 上一篇：[第 11 篇 — DASH 与 CMAF 统一分片格式](11-DASH与CMAF统一分片格式.md) | 下一篇：[第 13 篇 — WHIP/WHEP 标准化推拉流](13-WHIP-WHEP标准化推拉流.md)
