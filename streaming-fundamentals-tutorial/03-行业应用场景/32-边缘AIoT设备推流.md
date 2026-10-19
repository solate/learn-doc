# 边缘 AIoT 设备推流

> **适合读者**：需要把边缘设备（AI 摄像头 / 无人机 / 机器人 / 边缘盒子）的视频流接入云端的后端 / 嵌入式开发者
> **核心问题**：固定摄像头有稳定网线和充足带宽；边缘设备"移动、弱网、省电"——视频该怎么可靠地推到云端？

---

## 一、边缘 AIoT 与固定摄像头：本质区别

先把场景立起来。同样是"推流"这个动作，固定摄像头和边缘设备要解决的问题完全不同：

| 维度 | 固定摄像头（监控）| 边缘 AIoT 设备 |
|------|-----------------|---------------|
| 网络 | 有线 / 局域网，稳定 | 4G/5G/Wi-Fi，弱网、抖动、随时断 |
| 算力 | 弱（只采集）| 中（要本地跑 AI 模型）|
| 功耗 | 市电，无压力 | 电池供电，编码功耗必须省 |
| 位置 | 固定，一次部署终身不动 | 移动，飞行 / 巡检中持续推流 |
| 上行带宽 | 充足（百兆内网）| 拮据（4G 上行实测常只有 1-5Mbps）|

**一句话认知**：固定摄像头拼"画质和稳定"；边缘设备拼"**弱网下的存活**"——宁可掉帧掉画质，不能断流、不能烧完电池。本章所有技术选择都围绕这一条展开。

读完本章你能理解：为什么边缘 IoT 的推流方案不能照搬固定监控方案。

---

## 二、四层链路：两条通道各管一件事

边缘设备接入云端，从来不是"一条视频流"的事，而是**两条独立通道**并行：

```
设备端                                  云端
────────                                  ──────
① 视频通道   采集 → 硬件编码 → SRT/RTSP ────→ 流媒体服务器(MediaMTX) → WHEP/HLS → 浏览器
② 消息通道   AI 结果/心跳/状态 ── MQTT ──────→ MQTT Broker → 业务后端(Go)
             控制命令 ←─── MQTT/WebSocket ───── 业务后端 → 前端操作台
```

**为什么必须分两条通道？**

- 视频流是**大而连续、可容忍丢帧**的数据；控制 / 事件是**小而关键、必须可靠**的数据，诉求南辕北辙（协议分工详见[第 33 篇](33-IoT消息协议与流媒体的分工.md)）。
- 故障域隔离：视频断了不影响控制，反过来也一样。
- 弱网时可以把视频降级，但**心跳和控制永远要保持**——否则设备失联、无法遥控。

读完本章你能理解：边缘接入 = 视频通道 + 消息通道，缺一不可，且选型各自独立。

---

## 三、设备端采集编码：为什么首选硬件编码 + GStreamer

边缘盒子（Jetson / RK3568 等）CPU 本就不强，还要实时跑 AI 模型，编码不能再抢 CPU。硬件编码器（NVENC / VPU）把 H.264 编码的 CPU 占用从 **20-50% 压到接近 0**，这是电池供电设备的刚需；顺带编码延迟更低，画面更跟手。

在 Jetson 上推流的惯用武器是 **GStreamer** 而非 FFmpeg：GStreamer 流水线全程在 NVMM 显存 / 内存中搬帧、**零内存拷贝**，而 FFmpeg 每级管道都要把数据过一遍系统内存，更费电费 CPU。

```bash
# NVIDIA Jetson：CSI 摄像头 → 硬件编码 → RTSP 推流
# 流水线语义：采集 → 颜色空间转换 → H.264 硬编 → 封装 → 推流
gst-launch-1.0 \
  nvarguscamerasrc ! \                              # 步骤1 摄像头采集（硬件）
  'video/x-raw(memory:NVMM), width=1280, height=720, framerate=30/1' ! \
  nvvidconv ! \                                     # 步骤2 颜色空间转换（GPU）
  'video/x-raw(memory:NVMM), format=I420' ! \
  nvv4l2h264enc bitrate=1500000 ! \                 # 步骤3 硬件 H.264 编码
  h264parse ! \
  rtspclientsink location=rtsp://server:8554/edge/jetson01 protocols=tcp
```

**FFmpeg 等效命令**（通用设备，无 Jetson 的 NVMM 硬件内存）：

```bash
ffmpeg -f v4l2 -input_format mjpeg -framerate 30 -i /dev/video0 \
  -c:v h264_nvenc -b:v 1.5M -f rtsp rtsp://server:8554/edge/jetson01
```

读完本章你能理解：边缘编码选"硬件编码器 + 少内存拷贝的框架"，是省电、并把算力留给 AI 的前提。

---

## 四、AI 与推流并行：AI 慢，但永远不挡住视频

边缘设备比固定摄像头多一条"本地 AI"。设计的铁律（与[第 35 篇](35-AI实时检测集成.md)服务端同理，这里是设备端）：

**采集 / 推流是实时主链路，任何情况下不能等 AI。**

```
摄像头帧 ──→ 推流编码器 ──→ 云端（实时，零阻塞）
     └──→ 帧队列(maxsize=10) ──→ AI 推理(慢) ──→ 事件(MQTT) ──→ 云端
                                          ↑
                         队列满就丢帧（丢的是给 AI 的帧，不是推流的帧）
```

```python
# Jetson 端：两个线程，各管各的
def capture_and_stream():
    """线程1：采集 + 硬编推流，绝不等待 AI"""
    while True:
        frame = cap.read()
        writer.write(frame)                 # 直接推流
        if not frame_queue.full():          # 非阻塞送 AI，满了丢
            frame_queue.put_nowait(frame)

def ai_inference():
    """线程2：AI 推理，结果走 MQTT 上报"""
    while True:
        frame = frame_queue.get()
        for det in model.infer(frame):
            mqtt_client.publish(
                "edge/jetson01/events", json.dumps(det), qos=1)
```

读完本章你能理解：AI 是"旁路消费者"，靠有界帧队列 + 非阻塞入队，永远拖不垮推流。

---

## 五、弱网传输：边缘推流的主战场

这是本章真正的核心。4G 上行带宽窄且波动大，边缘设备选传输协议只看一件事：**一丢包，画面是"卡死等重传"还是"抖一下继续播"？**

| 协议 | 传输层 | 弱网行为 | 边缘场景取舍 |
|------|--------|---------|-------------|
| RTMP | TCP | 丢包 → 队头阻塞 → 整画面卡住像死机 | 只适合网络尚可的固定位 |
| RTSP/UDP | UDP | 丢包 → 花块，但不断流 | 局域网就近接入 |
| **SRT** | UDP + ARQ/FEC | 丢包 → 应用层选择性重传，画面轻抖继续 | **弱网首选** |
| WebRTC | UDP + FEC/RTX | 自动降码率 + NAT 穿透 | 交互式双向场景 |

TCP 的问题在[第 5 篇](../01-底层技术基础/05-传输层协议TCP-UDP-QUIC.md)讲过：一丢包整条流堵住等重传，4G 场景下 2% 丢包率就能让 RTMP 画面卡到不能看。SRT（原理详见[第 9 篇](../01-底层技术基础/09-SRT协议与弱网推流.md)）在 UDP 上加"应用层 ARQ 重传 + FEC 纠错"，丢包只影响个别包，画面持续往前。

**SRT 弱网调参**——边缘场景最常动的两个旋钮：

```bash
ffmpeg -re -i /dev/video0 \
  -c:v h264_nvenc -b:v 1M \
  -f mpegts \
  "srt://server:9000?streamid=edge/jetson01&mode=caller&latency=300&maxbw=250000"
```

| 参数 | 作用 | 边缘建议 |
|------|------|---------|
| `latency`(ms) | 接收端缓冲窗口，垫平网络抖动 | **RTT×4** 左右：RTT 80ms → 300ms。越大越稳但越延迟 |
| `maxbw`(B/s) | 上行带宽上限（**字节/秒**，非 bit）| 0=自动；常被别家抢病时手动限，如 2Mbps 填 `250000` |
| `payloadsize` | 包大小 | 保持默认 1316，不动 |

**断线自愈**：4G 下被运营商周期性踢断是常态，设备端必须自动重连：

```bash
while true; do
  ffmpeg -rtsp_transport tcp -timeout 5000000 -i "$RTSP" \
    -c copy -f mpegts "srt://server:9000?streamid=...&mode=caller&latency=300" 2>/dev/null
  echo "[$(date)] 推流失联，5秒后重连"
  sleep 5
done
```

读完本章你能理解：边缘弱网选 SRT，用 `latency`/`maxbw` 两个旋钮在"抗抖动"与"低延迟"间取平衡，再配合重连循环保命。

---

## 六、节省上行带宽：平时低码率，事件升码率，极端只发关键帧

4G 上行带宽是稀缺资源。边缘设备推流有一套"**平时省、有事抬**"的三档策略：

1. **平时（无事件）**：低码率 512Kbps，够远程看一眼即可。
2. **有事件（AI 告警）**：MQTT 通知云端"有情况"，同时把推流临时升到高清（1.5Mbps+），事后再降回来。
3. **极端省带宽**（如无人机长时间巡检）：云端**不拉全量流**，设备端每 2 秒只发一个 **I 帧 / JPEG 快照**（几 KB ~ 百 KB），配 MQTT 上报元数据（定位 / 电池 / 检测结果）。要"看实时画面"时，云端再按需下命令切全量推流（见第七节）。

```bash
# 档位2：告警触发，低码率切高码率（服务端下发命令，设备端切换进程）
# 平时
ffmpeg -i rtsp://device/stream -c:v h264_nvenc -b:v 512K \
  -f mpegts "srt://server:9000?streamid=edge/edge01&mode=caller&latency=300" &
PID=$!
# 收到 MQTT 告警命令 → 停低码率，起高清
mosquitto_sub -h broker -t "cmd/edge01" -C 1 | while read _; do
  kill $PID
  ffmpeg -i rtsp://device/stream -c:v h264_nvenc -b:v 1500K \
    -f mpegts "srt://server:9000?streamid=edge/edge01&mode=caller&latency=300"
done
```

```bash
# 档位3：只抽关键帧 JPEG（巡检模式，带宽主力全省在这）
ffmpeg -rtsp_transport tcp -i rtsp://edge/stream -vf "select=eq(pict_type\,I)" \
  -vsync 0 -q:v 3 -f image2 -strftime 1 "snap/edge01_%Y%m%d_%H%M%S.jpg"
# 每抽一张，就把元数据发 MQTT
mosquitto_pub -t edge/edge01/kf \
  -m '{"ts":1789400000,"lat":31.23,"lng":121.47,"size":132}'
```

读完本章你能理解：边缘上行带宽靠"三档策略"来省——低码率兜底、事件升码率、极致场景只发关键帧 + 元数据。

---

## 七、双向控制：视频下行 + 指令上行

边缘设备不是单向直播，是"云能反控设备"的闭环：

```
操作台 ──WebSocket──→ Go 后端                         边缘设备
                          │ MQTT cmd/edge01/ptz（转向/变焦/重启）
                          └─────────────────────────→ 设备执行
设备状态/心跳/信号  ←── MQTT edge01/status ────────── 设备
```

- **控制下行**：MQTT QoS1 保证送达（详见[第 33 篇](33-IoT消息协议与流媒体的分工.md)）。需要走视频同路的即时低延迟交互（如实时遥控机器人）时，用 WebRTC DataChannel 顺路带过去，见[第 12 篇](../01-底层技术基础/12-WebRTC协议栈详解.md)。
- **状态上行**：心跳 / 电量 / 信号强度，MQTT 定时上报。
- **按需拉流**：配合第六节——平时只收关键帧，**有人要看实时画面时，云端再发命令让设备切全量推流**。这一条往往被忽略，却是省流量的关键开关。

读完本章你能理解：边缘接入是"能看 + 能控"闭环；控制走消息通道，视频按需启停来省钱。

---

## 八、云端接入：设备注册与推流鉴权

云端必须对"谁在推流"做主，核心三件事：

1. **验身份**：每台设备预置 SN 码 + 签名密钥，云端正向验证——防伪造设备混入。
2. **发推流密钥**：设备上线注册 → 云端签发一次性 `stream_key` → 设备用带 key 的地址推流，流媒体服务器（MediaMTX）校验 key 才收流。
3. **状态联动**：流上 / 下线回调通知业务后端，异常掉线自动告警。

```go
// Go 云端的设备注册服务（MediaMTX 接入详见第 16 篇）
type EdgeDeviceService struct {
    db       *gorm.DB
    mediaMTX *mediamtx.Client   // 管理流媒体服务器路径
    mqtt     mqtt.Client        // 接收设备事件 + 下发命令
}

func (s *EdgeDeviceService) RegisterDevice(ctx context.Context, req *dto.DeviceRegisterRequest) error {
    if !s.validateDeviceSN(req.SerialNumber, req.Signature) {   // 1. 验身份
        return xerr.New(xerr.ErrForbidden.Code, "设备认证失败")
    }
    key := generateStreamKey(req.SerialNumber)                   // 2. 生成推流密钥
    if err := s.mediaMTX.AddPath("edge/"+req.SerialNumber, key); err != nil {
        return err
    }
    // 3. 返回给设备，设备拿 stream_key 去推流
    return s.replyRegister(ctx, &dto.DeviceRegisterResponse{
        StreamURL: "srt://server:9000?latency=300",
        StreamKey: key,
    })
}
```

读完本章你能理解：设备接入云端的核心是"注册发密钥 → 校验推流 → 状态联动"，把设备管理收进业务层，而不是裸收流。

---

## 本篇小结

1. 边缘设备三大约束——**弱网、小算力、移动**，一切选型服从"弱网下不断流"。
2. 接入是**两条通道**：视频走 SRT/RTSP，控制与事件走 MQTT，故障域隔离。
3. 弱网传输首选 **SRT**：`latency ≈ RTT×4` + `maxbw` 限速 + 自动重连循环。
4. 上行带宽靠**三档策略**省：平时低码率 / 事件升码率 / 极端只发关键帧 + 元数据。
5. 云端要做**设备注册 + 推流鉴权 + 按需拉流控制**，别裸收流。

---

## 动手实践

### 练习1：对比 RTMP vs SRT 在弱网下的表现（Linux）
```bash
# 模拟 2% 丢包（需要管理员权限；macOS 可用 Network Link Conditioner 等效模拟）
sudo tc qdisc add dev eth0 root netem loss 2%

# 分别推同一视频，观察播放端卡顿差异：
ffmpeg -re -stream_loop -1 -i test.mp4 -c copy        -f flv     rtmp://server/live/a
ffmpeg -re -stream_loop -1 -i test.mp4 -c copy        -f mpegts "srt://server:9000?mode=caller&streamid=a&latency=300"

# 结束实验，清除丢包模拟
sudo tc qdisc del dev eth0 root
```
**预期输出**：RTMP 端画面明显停顿（TCP 队头阻塞）；SRT 端画面轻抖、断开视觉几乎察觉。再把 `latency` 从 300 调到 60 重试，体会"更灵敏"与"更稳"的此消彼长。

### 练习2：USB 摄像头模拟边缘设备，SRT 推流 + 自动重连
```bash
#!/bin/sh
# /opt/stream/start.sh — 设备开机自启推流
while true; do
  ffmpeg -f v4l2 -framerate 30 -i /dev/video0 \
    -c:v libx264 -preset veryfast -b:v 1M \
    -f mpegts "srt://cloud:9000?streamid=edge/cam01&mode=caller&latency=300" \
    2>>/var/log/push.log
  echo "[$(date)] 推流失联，5s 后重连" >>/var/log/push.log
  sleep 5
done
```
**预期输出**：`tail -f /var/log/push.log` 呈现"启动 → 推流 → 断开 → 5s 重连"的周期日志；云端 MediaMTX 上 `srt://.../edge/cam01` 路径稳定，掉线后画面在 latency 时间窗内自动恢复。

### 练习3：关键帧 + 元数据上报（巡检省带宽模式）
```bash
# 设备端：只抽 I 帧存 JPEG（每 2s 关键帧一张）
ffmpeg -rtsp_transport tcp -i rtsp://edge/stream -vf "select=eq(pict_type\,I)" \
  -vsync 0 -q:v 3 -f image2 -strftime 1 "snap/%Y%m%d_%H%M%S.jpg"
# 云端的收流进程：监听 MQTT 关键帧元数据，拼出低带宽监控视图
mosquitto_sub -t "edge/edge01/kf" -v
```
**预期输出**：持续收到类似 `edge/edge01/kf {"ts":...,"size":132}` 的元数据流，配合上一步的 JPEG 图片即可低成本跟踪设备状态；此时上行流量远低于全量推流。试着在事件发生时通过 MQTT 命令把设备切到全量推流，对比两种模式的上行带宽。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| 推流画面卡顿、像死机 | 4G 弱网下用了 RTMP/TCP，队头阻塞 | 换 SRT，`latency=RTT×4`，`maxbw` 限制上行 |
| SRT 延迟偏高 | `latency` 缓冲调得过大 | 减到 RTT×4 左右，平衡稳定与延迟 |
| 推流周期性断开重连 | 运营商限时断连 / NAT 会话老化 | 重连循环保活；必要时用 SRT Rendezvous 模式双向穿透 |
| 电池掉得飞快 | 软件编码吃满 CPU | 换硬件编码（NVENC / VPU），降帧率降码率 |
| 打开 AI 后画面卡顿 | 推流和 AI 抢了同一线程 / 队列无界 | 双线程 + 有界帧队列，AI 非阻塞消费（见第四节） |
| 事件时间戳与云端画面对不上 | 设备时钟漂移 | 设备端配置 NTP / chrony，时间戳统一 Unix 毫秒 |
| 云端收到伪造推流 | 没有设备鉴权 | 设备注册 + 一次性推流密钥，MediaMTX 校验 key |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 边缘计算 | Edge Computing | 设备本地做数据处理 / AI 推理，只把结果与必要数据传云端 |
| 硬件编码 | NVENC / VPU | 设备内置专用编码芯片，编码几乎不占 CPU |
| GStreamer | GStreamer | 多媒体流水线框架，Jetson 上零拷贝硬件推流首选 |
| SRT | Secure Reliable Transport | UDP 上的应用层可靠传输，弱网推流首选（第 9 篇）|
| latency（SRT）| SRT Latency | SRT 接收端缓冲窗口（ms），"抗抖动"与"低延迟"的旋钮 |
| maxbw（SRT）| Max Bandwidth | SRT 上行带宽上限（字节/秒），限速防拥塞 |
| 关键帧 | I-Frame / Keyframe | GOP 起始帧，不依赖参考帧、可独立解码 |
| 帧队列 | Frame Queue | 有界队列，AI 慢时丢帧保推流不阻塞 |
| QoS 1 | MQTT QoS 1 | 至少一次送达，断线重连后补发（第 33 篇）|

---

> **全文完** | 上一篇：[第 31 篇 — 云游戏流媒体技术](31-云游戏流媒体技术.md) | 下一篇：[第 33 篇 — IoT 消息协议与流媒体的分工](33-IoT消息协议与流媒体的分工.md)