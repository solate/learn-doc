# IoT 消息协议与流媒体的分工

> **适合读者**：同时处理 IoT 传感器数据和视频流的后端开发者
> **核心问题**：MQTT/HTTP 和 RTSP/RTMP 各管什么？为什么不能混用？

---

## 一、两类数据：为什么必须分两条通道

先立起场景。一个 IoT 视频系统里同时躺着两类数据，诉求完全相反：

| 维度 | 消息数据（状态/告警/命令）| 视频流 |
|------|--------------------------|--------|
| 单条大小 | 几十 B ~ 几 KB | 单帧 50–200KB（压缩后），持续每秒 1.5–6MB |
| 产生方式 | 事件驱动，突发（告警风暴）| 恒速连续（30fps 不停）|
| 丢失成本 | 丢一条 = 丢一次告警/命令，后果严重 | 丢几帧 = 画面闪一下，可接受 |
| 延迟诉求 | 命令要快、要可靠 | 要流畅，允许少量丢包 |
| 本质 | 有语义的事件："门开了" | 无语义的二进制码流 |

**混用会怎样？** 这才是"不能混用"的底层原因：

- **用 MQTT 传视频**：一帧几百 KB 的 payload 会把 Broker 的消息处理栈堵死——传一帧的时间与带宽，够传几百条告警。MQTT 生来为"小消息、高并发"设计。
- **用 RTSP/RTMP 传控制命令**：流媒体协议根本没有"命令/应答"和"重试"语义，一条指令淹没在码流里，设备既收不到也不知道。

```
设备端
 ├─ 传感器模块 ── MQTT(几十B/条) ──→ MQTT Broker ──→ 业务后端(Go) ─→ 告警/通知/联动
 └─ 摄像头模块 ── RTSP/RTMP(大码流) ─→ 流媒体服务器 ──→ 播放器

两边互不阻塞：视频断了不影响控制，Broker 挂了也不影响录像。
```

**结论**：消息走消息协议（有语义、可靠可重试），视频走流媒体协议（大而连续、丢帧可容错）。两条链路完全独立，在业务层再汇聚。

读完本章你能理解：分通道不是工程习惯，而是两类数据"大小量级"和"丢失成本"完全不同的必然结果。

---

## 二、协议地图：每类数据选哪个

| 协议 | 管什么 | 关键特性 | 本文之外详见 |
|------|--------|---------|-------------|
| **MQTT** | 设备↔云端消息总线：状态/告警/命令/事件 | 发布订阅、QoS 0/1/2、离线补发、遗嘱 | 本篇主角 |
| **HTTP/REST** | 低频请求-应答：配置查询、注册、上传 | 语义丰富、无状态、超时即败（结果可知）| — |
| **WebSocket** | 服务端→浏览器的实时推送；也是浏览器连 MQTT 的通道 | 全双工长连接、浏览器原生支持 | — |
| **RTSP/RTMP** | 视频推拉流 | 大码流、低延迟；RTSP 靠 UDP、RTMP 靠 TCP | [第 7 篇](../01-底层技术基础/07-RTSP协议详解.md) / [第 8 篇](../01-底层技术基础/08-RTMP协议详解.md) |
| **HLS** | CDN 分发播放（切分片走 HTTP）| 兼容最好、延迟 6-30s | [第 10 篇](../01-底层技术基础/10-HLS协议详解.md) |
| **WebRTC / SRT** | 低延迟交互 / 弱网推流 | 浏览器端低延迟 / UDP 上抗丢包 | [第 12 篇](../01-底层技术基础/12-WebRTC协议栈详解.md) / [第 9 篇](../01-底层技术基础/09-SRT协议与弱网推流.md) |
| **Kafka**（升级位）| 设备量几十万级、单 Broker 扛不住时做消息总线 | 高吞吐、消息可重放 | 量小阶段用 MQTT 即可，不必提前上 |

**补充**：RabbitMQ（AMQP）、Kafka 与 MQTT 同属消息中间件，同样不适合传视频码流——这是本质决定的，不是配置问题。IoT 边缘优先 MQTT（轻量，几百 KB 起步），量大到单 Broker 扛不住再升级 Kafka。

**一句话选型**（对应四种典型操作）：

| 操作 | 选型 | 理由 |
|------|------|------|
| 单向上报（温度/心跳/事件）| MQTT QoS0/1 | 轻量，丢了可等下一次 |
| 必须送达的命令（重启/云台/截图）| MQTT QoS1 | 断线自动补发 |
| 请求并要拿结果（查配置/调状态）| HTTP | 语义匹配"我问你答" |
| 实时双向（遥控/弹幕）| WebSocket / WebRTC | 全双工低延迟 |
| 视频本体 | RTSP/RTMP/SRT/WebRTC | 绝不进消息总线 |

读完本章你能理解：选协议的第一步不是看名字，而是看数据是"一条条有语义的事件"还是"连续码流"。

---

## 三、MQTT 必懂的四个机制

只记一嘴"MQTT 是发布订阅"在工业现场不够用。这四个机制才是 MQTT 可靠性的全部来源：

### 1. QoS——送达次数与代价的切换

| QoS | 语义 | 会重复吗 | 会丢吗 | 代价 |
|-----|------|---------|--------|------|
| 0 | 最多一次：发出即忘 | 否 | **可能** | 最低，无确认 |
| 1 | 至少一次：等 PUBACK，超时重发 | **可能** | 不丢 | 中，确认+重传 |
| 2 | 恰好一次：四次握手 | 否 | 不丢 | 最高，慢且占内存 |

**现场默认值**：传感器上报 QoS0（丢了下次再报）；控制命令、告警 QoS1；QoS2 几乎不用——需要"不重不丢"时，用 QoS1 + 消费端按 `(设备ID, 消息序号)` 去重，代价远低于 QoS2。

```go
// QoS1 可能重复投递，消费端必须幂等：处理前先去重
var seen sync.Map // key = deviceID + msgID
if _, loaded := seen.LoadOrStore(alert.DeviceID+":"+alert.MsgID, struct{}{}); loaded {
    return // 这条已处理过，丢弃
}
```

### 2. Retain——新订阅者立即拿到"当前值"

Broker 保存**最后一条** retain 消息，任何新订阅者一上线立刻收到。经典用途：设备状态、当前固件版本、配置快照——新设备注册 / 订阅者重连，马上知道"现在是什么"，而不是只等将来的新消息。

```bash
# 发布时带 -r（retain）：设备上线就把自己的状态写进 Broker
mosquitto_pub -h broker -t devices/cam01/state -r -m '{"online":true}' -q 1
# 新订阅者一订阅立即打印这条，无需等待
mosquitto_sub -h broker -t devices/cam01/state -C 1
```

### 3. Will 遗嘱——设备掉线被立刻知道

注册连接时声明一条遗嘱：当客户端**异常断开**（KeepAlive 超时未心跳、网络中断），Broker 代发这条消息。业务端订阅遗嘱主题，几秒内就知道"哪台设备死了"，不用干等 30 秒心跳超时才发现。

```go
opts := mqtt.NewClientOptions().
    SetKeepAlive(30 * time.Second). // 心跳；超 1.5 倍未收到才判离线
    SetWill("devices/+/rst", `{"device_id":"cam01","reason":"offline"}`, 1, false)
// 注意：正常 ctrl+c 发送 DISCONNECT 不触发遗嘱，只有异常断开才触发
```

### 4. Persistent Session——离线也不漏消息

`CleanSession=false`（MQTT5 叫 Clean Start=false + Session Expiry）时，Broker 保留客户端的订阅关系，断线期间发来的 QoS1/QoS2 消息**缓存在 Broker**，重连后一次补发。保证"设备重启 / 4G 掉线的几十秒里，云端下的命令一条不丢"。

```go
opts.SetCleanSession(false).SetClientID("gateway-01") // clientID 要固定，会话才找得到
```

代价提醒：保留的会话和离线消息占 Broker 内存，只给"必须可靠"的角色（控制网关 / 命令接收端）开，不要给海量传感器开。

读完本章你能理解：QoS 管"送达几遍"，Retain 管"新来的也能看到现在"，Will 管"掉线被即刻知道"，Session 管"离线也不漏"。四个机制配齐，MQTT 才是"可靠消息总线"，否则只是"广播喇叭"。

---

## 四、Topic 设计与命令通道：HTTP 还是 MQTT

### Topic 规范

```
设备 → 云端（上行）：
  devices/{device_id}/status          # 心跳/在线状态（retain）
  devices/{device_id}/alert           # 告警事件（QoS1）
  devices/{device_id}/sensor/temp     # 温度数据（QoS0）
  devices/{device_id}/ai/detection    # AI 检测结果

云端 → 设备（下行）：
  cmd/{device_id}/config              # 配置下发（QoS1）
  cmd/{device_id}/ptz                 # 云台控制
  cmd/{device_id}/reboot              # 远程重启
  cmd/{device_id}/snapshot            # 触发截图
```

三条纪律：上行下行分前缀（`devices/` vs `cmd/`）、按设备隔离（`{device_id}` 可做服务端 ACL，防设备互订阅）、事件名用名词（`alert` / `status`），不要动词当主题。

### 命令下发：HTTP 还是 MQTT？

| 场景 | 选型 | 原因 |
|------|------|------|
| 请求并要结果（询配置）| HTTP | "我问你答"，语义匹配 |
| 只要送达、不要即时应答（重启/云台/截图）| MQTT QoS1 | 送达保证 + 离线补发 |
| 广播（固件升级/批量唤醒）| MQTT | 订阅关系天然一对多，HTTP 做不了 |
| 正向推送（状态变更主动通知）| MQTT/WebSocket | HTTP 需要轮询，不自然 |

**IoT 系统的结论**：命令也走 MQTT（QoS1），HTTP 只留给管理后台 / 浏览器——因为"送达保证、离线补发、广播"恰好是 MQTT 的强项。

读完本章你能理解：HTTP 是"我问你答"，MQTT 是"我说你收"。要送达保证、离线补发、一对多广播就用 MQTT；要拿到请求结果才用 HTTP。

---

## 五、Go 实战：订阅事件 → 联动视频快照

后端把 MQTT 当**输入事件源**：订阅主题 → 回调里做业务动作。视频操作（拉流/截图）由后端主动发起，MQTT 侧只读事件。

```go
// 依赖：github.com/eclipse/paho.mqtt.golang
// 订阅设备告警，触发视频截图存证
func (s *AlertService) SubscribeDeviceAlerts() {
    s.mqtt.Subscribe("devices/+/alert", 1, func(client mqtt.Client, msg mqtt.Message) {
        var alert DeviceAlert
        if err := json.Unmarshal(msg.Payload(), &alert); err != nil {
            log.Warn().Err(err).Msg("解析告警失败")
            return
        }
        log.Info().Str("device", alert.DeviceID).Str("type", alert.Type).Msg("收到设备告警")

        // 告警发生时，截取对应摄像头当前帧存证（异步，不阻塞订阅回调）
        go s.captureSnapshot(context.Background(), alert.CameraID, alert.Timestamp)
        // 推送通知给相关用户
        go s.notifyUsers(context.Background(), &alert)
    })
}

// 截图：调 FFmpeg 从 RTSP 抓一帧
func (s *AlertService) captureSnapshot(ctx context.Context, cameraID string, ts int64) {
    cmd := exec.CommandContext(ctx, "ffmpeg",
        "-rtsp_transport", "tcp",
        "-i", s.getCameraRTSP(ctx, cameraID),
        "-vframes", "1", "-q:v", "2", // 只抓 1 帧，JPEG 质量 2
        fmt.Sprintf("snapshots/%s_%d.jpg", cameraID, ts))
    if err := cmd.Run(); err != nil {
        log.Error().Err(err).Str("camera", cameraID).Msg("截图失败")
    }
}
```

读完本章你能理解：消息通道负责"知道发生了什么"，视频通道负责"记录发生了什么"——后端用一个订阅回调，把两类通道联动起来。

---

## 六、事件联动：从告警到处置的闭环

一个完整的联动例子（AI 检测到异常 → 录像 + 截图 + 告警 + 通知）：

```go
func (s *EventService) HandleAIDetection(ctx context.Context, e *AIDetectionEvent) {
    if e.Confidence < 0.8 {
        return // 置信度低，忽略
    }
    log.Info().Str("type", e.Type).Str("camera", e.CameraID).Float64("conf", e.Confidence).Msg("AI 检测到事件")

    // 1. 触发录像：流媒体服务器开始保存 30s 片段
    s.recordingService.StartEventRecording(ctx, e.CameraID, 30*time.Second)
    // 2. 截图存证（走视频通道）
    go s.captureSnapshot(ctx, e.CameraID, e.Timestamp)
    // 3. 落库告警记录
    s.alertRepo.Create(ctx, &model.Alert{
        AlertID:    idgen.GenerateUUID(),
        CameraID:   e.CameraID,
        Type:       e.Type,
        Confidence: e.Confidence,
        CreatedAt:  e.Timestamp,
    })
    // 4. 推送通知（短信/App 走各自通道）
    s.notificationService.Send(ctx, e.TenantID, &Notification{
        Title: fmt.Sprintf("[告警] %s 检测到 %s", e.CameraName, e.Type),
    })
}
```

时序：`边缘 AI 事件 —MQTT→ 后端判置信度 —命令→ 录像/截图 —写库→ 通知`。视频流自始至终没进消息总线，消息通道只传"事件"这个信号。

读完本章你能理解：两条通道靠"事件"握手——消息通道传信号，视频通道传画面，业务层负责编排。

---

## 本篇小结

1. 两类数据**大小量级与丢失成本不同**，决定必须分通道：消息走 MQTT、视频走流媒体，互不阻塞。
2. 选协议看数据形状：**有语义的事件 → 消息协议；连续码流 → 流媒体协议**（RabbitMQ/Kafka 同理不适合传视频）。
3. MQTT 四个机制：**QoS** 管送达遍数（默认 0/1，放弃 2）、**Retain** 让新订阅者见"当前值"、**Will** 让掉线被即刻知道、**CleanSession=false** 让离线不丢消息。
4. 命令下发 **MQTT QoS1 优于 HTTP**：送达保证 + 离线补发 + 一对多广播；HTTP 只留给"请求拿结果"。
5. Topic 三纪律：上下行分前缀、按设备隔离（服务端 ACL）、名词命名。
6. 消费端必须**幂等 + 去重**（QoS1 重复投递），这是最容易被忽略的坑。

---

## 动手实践

### 练习1：Go 服务订阅 MQTT（QoS1 + 幂等去重）
```go
// 依赖：github.com/eclipse/paho.mqtt.golang
var seen sync.Map // key = deviceID + msgID

func (s *IoTService) SubscribeEvents(brokerURL string) {
    opts := mqtt.NewClientOptions().
        AddBroker(brokerURL).
        SetClientID("backend-subscriber").
        SetAutoReconnect(true).
        SetConnectRetryInterval(3 * time.Second)

    client := mqtt.NewClient(opts)
    if token := client.Connect(); token.Wait() && token.Error() != nil {
        log.Error().Err(token.Error()).Msg("MQTT 连接失败")
        return
    }

    client.Subscribe("devices/+/alert", 1, func(c mqtt.Client, msg mqtt.Message) {
        var alert DeviceAlert
        if json.Unmarshal(msg.Payload(), &alert) != nil {
            log.Warn().Msg("解析失败")
            return
        }
        if _, ok := seen.LoadOrStore(alert.DeviceID+":"+alert.MsgID, struct{}{}); ok {
            return // QoS1 重传导致的重复，丢弃
        }
        s.handleAlert(&alert)
    })
    log.Info().Str("broker", brokerURL).Msg("MQTT 订阅启动")
}
```
**预期输出**：向 `devices/cam01/alert` 连发 100 条（含人为重发的重复消息），订阅端日志"收到告警"且无重复处理；发布中途断开 Broker 再重连，对比 QoS0（消息立丢）与 QoS1（重连补发）的差异。

### 练习2：用 mosquitto 命令行验证 Retain 与 Will（无 Go 依赖）
```bash
# ① Retain：新订阅者立即拿到"当前值"
mosquitto_pub -h broker -t devices/cam01/state -r -m '{"online":true}' -q 1
mosquitto_sub -h broker -t devices/cam01/state -C 1   # 一订阅立刻打印，不用等

# ② Will：模拟设备异常掉线
#    终端A：订阅遗嘱主题
mosquitto_sub -h broker -t 'devices/+/rst' -v
#    终端B：带遗嘱发布一条消息（模拟设备连接）
mosquitto_pub -h broker -t devices/cam01/up -m on \
  --will-topic devices/cam01/rst --will-payload offline -q 1
#    在终端B上 kill -9 强杀该进程（模拟断网），观察终端A
```
**预期输出**：① 订阅命令立即打印 `devices/cam01/state {"online":true}`；② 强杀后 1.5×KeepAlive 内，终端A打印 `devices/cam01/rst offline`。注意：正常 ctrl+c 不触发遗嘱。

### 练习3：MQTT 事件触发 FFmpeg 视频快照
```go
// 收到"火焰检测"事件 → 从 RTSP 抓一帧存证
func (s *IoTService) processEvent(deviceID string, e *DetectionEvent) {
    if e.EventType != "fire" || e.Confidence < 0.8 {
        return
    }
    go func() {
        out := fmt.Sprintf("/tmp/alert_%s_%d.jpg", deviceID, e.Timestamp)
        cmd := exec.Command("ffmpeg", "-rtsp_transport", "tcp",
            "-i", fmt.Sprintf("rtsp://mediamtx/%s", deviceID),
            "-vframes", "1", "-q:v", "2", out)
        if err := cmd.Run(); err != nil {
            log.Error().Err(err).Str("device", deviceID).Msg("截图失败")
            return
        }
        log.Info().Str("path", out).Msg("告警截图已保存")
    }()
}
```
**预期输出**：向 `devices/cam01/alert` 发布火焰事件后，几秒内 `/tmp/alert_cam01_<ts>.jpg` 生成并出现在日志；试着把 `-q:v` 从 2 改到 10，观察截图体积变化（画质与体积的权衡）。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| QoS1 消息重复收到 | Broker 未收到 PUBACK 触发重传 | 消费端按 `(deviceID, msgID)` 幂等去重 |
| 告警风暴（1s 内上千条）| 边缘设备未节流 | 边缘侧同类型事件 30s 冷却，只上报一条 |
| MQTT 断线后命令丢失 | QoS0 + CleanSession=true | 命令改 QoS1，`SetCleanSession(false)` 离线补发 |
| 设备掉线很久才发现 | 只依赖"上报不来了"做判断 | 设备注册 Will 遗嘱 + 订阅遗嘱主题，秒级感知掉线 |
| 遗嘱没触发 | 客户端正常退出（发了 DISCONNECT）| 只有异常断开才触发；测试用 kill -9/断网模拟 |
| 事件时间戳与视频对不上 | 设备时钟漂移 | 统一 Unix 毫秒，设备端配 NTP 同步 |
| QoS2 导致消息积压、延迟高 | 握手开销大 | 换 QoS1 + 消费端去重 |
| Broker 连接数告罄 | 单节点 EMQX/Mosquitto | 部署 EMQX 集群（Erlang 节点）横向扩展 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| MQTT | MQTT | 轻量 IoT 消息协议，基于发布订阅模型 |
| Broker | MQTT Broker | 消息代理服务器，如 EMQX、Mosquitto |
| QoS 0/1/2 | Quality of Service | 最多一次 / 至少一次（会重复）/ 恰好一次 |
| Retain | Retained Message | 保留消息，新订阅者立即可见"当前值" |
| Will | Will Message | 遗嘱，设备异常掉线时由 Broker 代发 |
| KeepAlive | Keep Alive | 心跳间隔，超 1.5 倍未收到判定离线 |
| CleanSession | Clean Session | false 时保留订阅并缓存离线期间的 QoS1/2 消息 |
| 事件驱动 | Event-Driven | 以事件为触发源驱动业务逻辑的架构模式 |

---

> **全文完** | 上一篇：[第 32 篇 — 边缘 AIoT 设备推流](32-边缘AIoT设备推流.md) | 下一篇：[第 34 篇 — 电力行业视频协议与国网标准](34-电力行业视频协议与国网标准.md)