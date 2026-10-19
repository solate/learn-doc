# RTMP 协议详解（直播推流标准）

## 目录

- [读完本篇你能做什么](#读完本篇你能做什么)
- [一、RTMP 协议概述](#一rtmp-协议概述)
- [二、RTMP 握手流程](#二rtmp-握手流程)
- [三、RTMP 消息与 Chunk](#三rtmp-消息与-chunk)
- [四、RTMP 推流与拉流](#四rtmp-推流与拉流)
- [五、RTMP 变种协议](#五rtmp-变种协议)
- [六、RTMP vs RTSP vs HLS](#六rtmp-vs-rtsp-vs-hls)
- [七、动手实践](#七动手实践)
- [八、常见问题排查](#八常见问题排查)
- [九、本篇小结](#九本篇小结)
- [下一篇](#下一篇)
- [参考资料](#参考资料)

---

## 读完本篇你能做什么

1. **理解 RTMP 架构**：掌握 Chunk Stream、Message Stream 分层设计
2. **解析 RTMP 握手**：了解 C0/S0/C1/S1/C2/S2 握手过程及应用层命令序列
3. **手动实现 RTMP 客户端**：理解 connect/createStream/publish/play 完整流程
4. **解析 AMF 编码**：掌握 RTMP 命令消息的二进制格式
5. **使用 FFmpeg/OBS 推流**：将视频推送到 RTMP 服务器
6. **部署 RTMP 服务器**：使用 nginx-rtmp-module 搭建流媒体服务器
7. **Wireshark 抓包分析**：定位 RTMP 协议层问题
8. **对比 RTMP/RTSP/HLS**：为不同场景选择合适的协议

---

## 一、RTMP 协议概述

### 1.1 什么是 RTMP？

**RTMP**（Real-Time Messaging Protocol）是 Adobe 开发的流媒体传输协议，最初用于 Flash Player 播放器。

**关键特点**：
- **基于 TCP**：可靠传输，避免丢包导致的画面撕裂
- **低延迟**：1-3 秒延迟，适合互动直播
- **Chunk 流设计**：将大消息切分成小块，支持多路复用
- **双向通信**：支持推流和拉流

**典型应用场景**：
- 直播推流（OBS → 服务器）
- 直播平台内部传输（服务器间转发）
- CDN 源站协议（部分厂商）

### 1.2 RTMP URL 格式

```
rtmp://[host][:port]/app/streamKey
```

**示例**：
```
rtmp://live.example.com/live/my-stream-key
rtmp://localhost:1935/hls/test
rtmp://a.rtmp.youtube.com/live2/xxxx-xxxx-xxxx
```

**组成部分**：
- **host:port**：RTMP 服务器地址，默认端口 **1935**
- **app**：应用名（类似虚拟目录），如 `live`、`hls`
- **streamKey**：流密钥，唯一标识一个直播流

### 1.3 RTMP 历史与现状

| 时期 | 状态 |
|------|------|
| 2002 | Macromedia 开发，配合 Flash Player 使用 |
| 2009 | Adobe 公开部分规范（RTMP Specification） |
| 2011 | HLS 推出，逐步替代 RTMP 用于 CDN 分发 |
| 2020 | Flash Player 正式停止支持 |
| 至今 | **推流标准仍是 RTMP**（OBS/编码器默认支持） |

Flash 虽已消亡，但 RTMP 在**推流端**（编码器到服务器）依然是行业标准，原因：
- 低延迟（比 HTTP 上传更实时）
- 流量控制（TCP 拥塞控制自动限速）
- 所有直播平台均支持

---

## 二、RTMP 握手流程

### 2.1 握手阶段概述

RTMP 在 TCP 连接建立后进行三路握手，总共交换 **1537 + 1537 = 3074 字节**：

```
客户端                          服务器
  |                               |
  |  C0 (1B: version=3)           |
  |  C1 (1536B: time + random)    |
  |------------------------------>|
  |                               |
  |        S0 (1B: version=3)     |
  |        S1 (1536B: time+rand)  |
  |        S2 (1536B: echo C1)    |
  |<------------------------------|
  |                               |
  |  C2 (1536B: echo S1)          |
  |------------------------------>|
  |                               |
  |     握手完成，开始传输         |
```

### 2.2 握手包格式

**C0/S0**（1 字节）：版本号，固定为 `3`（RTMP v3）

**C1/S1**（1536 字节）：
```
+------------+-----------+----------+
|  time(4B)  | zeros(4B) | random(1528B) |
+------------+-----------+----------+
```
- `time`：客户端/服务器时间戳
- `zeros`：全零（旧规范）
- `random`：1528 字节随机数据

**C2/S2**（1536 字节）：
```
+------------+-----------+--------------+
|  time(4B)  |  time2(4B)| echo(1528B)  |
+------------+-----------+--------------+
```
- `echo`：复制 C1/S1 的 random 字段（用于验证）

---

## 三、RTMP 消息与 Chunk

### 3.1 消息类型

| 消息类型 ID | 类型 | 说明 |
|------------|------|------|
| 1 | Set Chunk Size | 设置 Chunk 大小（默认 128B） |
| 2 | Abort Message | 中止消息 |
| 3 | Acknowledgement | 收到数据确认 |
| 5 | Window Acknowledgement Size | 窗口大小 |
| 8 | **Audio** | 音频数据 |
| 9 | **Video** | 视频数据 |
| 15 | AMF3 Data | AMF3 格式数据消息 |
| 18 | **AMF0 Data** | AMF0 格式数据（元数据） |
| 20 | **AMF0 Command** | 命令消息（connect/publish/play） |

### 3.2 Chunk 格式

RTMP 将**消息**切分成多个 **Chunk**（块）传输，默认每块 128 字节。

**Chunk 结构**：
```
+-------------+-------------------+----------+
| Chunk Header| Extended Timestamp| Chunk Data |
+-------------+-------------------+----------+
```

**Chunk Header** 根据类型（fmt=0/1/2/3）有不同长度：

| fmt | 头部大小 | 说明 |
|-----|---------|------|
| 0 | 11 字节 | 完整头部（新流或时间倒退） |
| 1 | 7 字节 | 省略 Stream ID |
| 2 | 3 字节 | 省略 Stream ID 和消息长度 |
| 3 | 0 字节 | 无头部（与前一块相同） |

**fmt=0 完整头部**：
```
+--------+-------------------+------------+----------+
| cs_id  |  timestamp(3B)    | msg_len(3B)| msg_type |
| (基础头)|                   |            |  (1B)    |
+--------+-------------------+------------+----------+
|               msg_stream_id (4B, 小端)              |
+-----------------------------------------------------+
```

### 3.3 AMF 编码

RTMP 的命令消息使用 **AMF**（Action Message Format）序列化，类似 JSON。

**AMF0 类型标识**：
```
0x00  Number (8B double)
0x02  String (2B length + data)
0x03  Object
0x05  Null
0x08  ECMA Array
```

**connect 命令示例**：
```
Command Name: "connect"
Transaction ID: 1
Command Object: {
    "app": "live",
    "flashVer": "FMLE/3.0",
    "tcUrl": "rtmp://localhost/live"
}
```

---

## 四、RTMP 与 FLV 容器格式

### 4.1 为什么 RTMP 离不开 FLV

**关键事实**：RTMP 协议传输的视频数据本质上是 **FLV 格式的数据流**。

```
RTMP 推流端（OBS）
  ↓
编码 → 封装成 FLV Tag → 通过 RTMP Chunk 传输
  ↓
RTMP 服务器
  ↓
转发 RTMP Chunk → 客户端解析 FLV Tag → 解码播放
```

**为什么选择 FLV**：
- FLV 是 Adobe 设计的流式容器格式，与 Flash Player 原生配合
- Tag 结构简单，解析开销极低
- 支持 H.264/AAC 编码（AVC/AAC Sequence Header 机制）
- RTMP 消息（Message Type 8/9）可以直接映射到 FLV Tag

### 4.2 FLV 文件结构

**FLV 文件组成**：
```
+-------------+
| FLV Header  |  (13 字节)
+-------------+
| Previous Tag Size 0 (4B, 固定为0) |
+-------------+
| FLV Tag 1   |
+-------------+
| Previous Tag Size 1 (4B) |
+-------------+
| FLV Tag 2   |
+-------------+
| Previous Tag Size 2 (4B) |
+-------------+
|     ...     |
+-------------+
```

**FLV Header**（9 字节 + 可选 4 字节）：
```
+--------+--------+--------+--------+--------+
| 'F'(1B)| 'L'(1B)| 'V'(1B)|Version | Flags  |
+--------+--------+--------+--------+--------+
|       Data Offset (4B, 通常为 9)           |
+--------------------------------------------+
```

- **Version**：固定为 `1`（FLV v1）
- **Flags**：
  - Bit 0 (LSB)：视频标志（1 = 有视频）
  - Bit 2：音频标志（1 = 有音频）
  - 其他位保留为 0

**Previous Tag Size**：每个 Tag 后跟 4 字节，记录**前一个 Tag 的大小**（包含 Tag Header），用于反向解析。

### 4.3 FLV Tag 结构

**Tag Header**（11 字节）：
```
+----------+--------------+----------------+----------+
| Tag Type | Data Size(3B)| Timestamp(3B)  | TS Ext(1B)|
|  (1B)    |              |                |          |
+----------+--------------+----------------+----------+
|       Stream ID (3B, 固定为 0)                      |
+-----------------------------------------------------+
```

**Tag Type 值**：
- **8**：音频 Tag（Audio Tag）
- **9**：视频 Tag（Video Tag）
- **18**：脚本数据 Tag（Script Data Tag，通常是元数据 onMetaData）

**Timestamp**：4 字节时间戳（毫秒），拆分为：
- 低 3 字节：Timestamp
- 高 1 字节：Timestamp Extended（时间戳扩展，高 8 位）

**完整时间戳计算**：
```
timestamp_ms = (Timestamp Extended << 24) | Timestamp
```

### 4.4 视频 Tag（Video Tag）详解

**Video Tag Data 结构**：
```
+------------+---------+-------------------+
| Frame Type | Codec ID| Video Data        |
|  (4 bits)  |(4 bits) | (AVCPacket/...)   |
+------------+---------+-------------------+
```

**Frame Type（高 4 位）**：
- `1`：关键帧（I 帧，Keyframe）
- `2`：非关键帧（P 帧/B 帧，Inter frame）
- `3`：可丢弃非关键帧（H.263 only）
- `4`：生成关键帧（服务器端命令）
- `5`：视频信息/命令帧（**AVC Sequence Header**）

**Codec ID（低 4 位）**：
- `2`：Sorenson H.263
- `4`：VP6
- `5`：VP6 with alpha channel
- `7`：**H.264（AVC）**（主流）
- `12`：**H.265（HEVC）**（Enhanced RTMP 扩展）

**H.264 AVC Packet 格式**：
```
+-------------+-------------------+--------------+
| AVC Packet  | Composition Time  | NALU Data    |
|   Type(1B)  |   Offset (3B)     |              |
+-------------+-------------------+--------------+
```

**AVC Packet Type 值**：
- `0`：**AVC Sequence Header**（SPS/PPS）
- `1`：**AVC NALU**（实际视频帧数据）
- `2`：AVC End of Sequence（流结束标志）

**AVC Sequence Header**（关键）：
```
配置记录格式（AVCDecoderConfigurationRecord）：
  - Configuration Version: 1
  - AVCProfileIndication: Profile（如 100 = High）
  - profile_compatibility: 0
  - AVCLevelIndication: Level（如 51 = Level 5.1）
  - lengthSizeMinusOne: 3（NALU 长度字段 = 4 字节）
  - numOfSequenceParameterSets: SPS 个数
  - sequenceParameterSetLength: SPS 长度
  - sequenceParameterSetNALUnit: SPS 数据
  - numOfPictureParameterSets: PPS 个数
  - pictureParameterSetLength: PPS 长度
  - pictureParameterSetNALUnit: PPS 数据
```

**SPS/PPS 的作用**：
- **SPS**（Sequence Parameter Set）：序列参数集，包含视频宽高、Profile、Level 等全局信息
- **PPS**（Picture Parameter Set）：图像参数集，包含编码模式、量化参数等

**播放器必须先收到 AVC Sequence Header（SPS/PPS），才能解码后续的 NALU 帧。**

### 4.5 音频 Tag（Audio Tag）详解

**Audio Tag Data 结构**：
```
+-------------+-------------+----------------+
| Sound Format| Sound Rate  | Sound Size     |
|  (4 bits)   | (2 bits)    | (1 bit)        |
+-------------+-------------+----------------+
| Sound Type  | Audio Data                   |
|  (1 bit)    |                              |
+-------------+------------------------------+
```

**Sound Format（高 4 位）**：
- `0`：Linear PCM（未压缩）
- `2`：MP3
- `10`：**AAC**（主流）
- `14`：MP3 8kHz

**Sound Rate（2 位）**：
- `0`：5.5 kHz
- `1`：11 kHz
- `2`：22 kHz
- `3`：**44 kHz**（AAC 忽略此字段，使用 AAC 配置）

**Sound Size（1 位）**：
- `0`：8-bit
- `1`：16-bit

**Sound Type（1 位）**：
- `0`：Mono（单声道）
- `1`：Stereo（立体声）

**AAC Packet 格式**（当 Sound Format = 10）：
```
+-------------+-------------------+
| AAC Packet  | AAC Data          |
|   Type(1B)  |                   |
+-------------+-------------------+
```

**AAC Packet Type 值**：
- `0`：**AAC Sequence Header**（AudioSpecificConfig）
- `1`：**AAC Raw**（实际音频帧数据）

**AAC Sequence Header**：
```
AudioSpecificConfig（2 字节）：
  - audioObjectType (5 bits): 音频对象类型（2 = AAC-LC）
  - samplingFrequencyIndex (4 bits): 采样率索引（如 4 = 44100 Hz）
  - channelConfiguration (4 bits): 声道配置（2 = 立体声）
  - frameLengthFlag (1 bit): 0
  - dependsOnCoreCoder (1 bit): 0
  - extensionFlag (1 bit): 0
```

### 4.6 脚本数据 Tag（Script Data Tag）

**用途**：传输流的元数据（onMetaData），包含视频宽高、码率、帧率等信息。

**Script Data 结构**：
```
+------------------+-------------------+
| AMF0 Object Name | AMF0 Object Data  |
|  (AMF0 String)   |  (AMF0 Object)    |
+------------------+-------------------+
```

**典型 onMetaData 内容**：
```javascript
{
  "duration": 0,           // 直播流为 0
  "width": 1920,
  "height": 1080,
  "videodatarate": 2500,   // 视频码率（kbps）
  "framerate": 30,
  "videocodecid": 7,       // H.264
  "audiodatarate": 128,    // 音频码率（kbps）
  "audiosamplerate": 44100,
  "audiosamplesize": 16,
  "stereo": true,
  "audiocodecid": 10,      // AAC
  "encoder": "obs-output module (libobs version 29.0.2)"
}
```

### 4.7 RTMP 消息与 FLV Tag 的映射关系

| RTMP 消息类型 | Message Type ID | 对应 FLV Tag Type | 说明 |
|--------------|----------------|------------------|------|
| Audio | 8 | 8 | 音频数据，直接映射 |
| Video | 9 | 9 | 视频数据，直接映射 |
| AMF0 Data | 18 | 18 | 元数据（onMetaData），直接映射 |
| AMF0 Command | 20 | - | 控制命令（connect/publish），不进入 FLV |

**关键结论**：
- RTMP 的 Audio/Video 消息（Type 8/9）与 FLV 的 Audio/Video Tag **完全一致**
- RTMP → FLV 几乎是零成本转换：**只需去掉 RTMP Chunk Header，保留 Tag 数据**
- FLV → RTMP 也同样简单：**给 FLV Tag 数据加上 RTMP Chunk Header**

### 4.8 为什么 HTTP-FLV 延迟低

**HTTP-FLV 原理**：
```
RTMP 推流端 → RTMP 服务器 → HTTP-FLV 服务端点
                            ↓
                       HTTP 长连接 + FLV Tag 流式推送
                            ↓
                       浏览器前端播放器解析 FLV Tag
                       （flv.js / mpegts.js / jessibuca）
```

**浏览器播放器选择**：
- **flv.js**：B站开源，仅支持 H.264，已停止维护（2018）
- **mpegts.js**：轻量稳定，仅支持 H.264，适合通用场景
- **jessibuca**：国内监控场景首选，**唯一免费支持 H.265** 的开源播放器（详见第 21.6 篇专篇介绍）

**延迟低的原因**：
1. **HTTP 长连接**：客户端发起 HTTP GET 请求后，服务器持续推送 FLV Tag，不关闭连接
2. **流式传输**：不需要等待完整文件，收到 Tag 立刻解析播放
3. **无分片开销**：与 HLS 不同，不需要等待 TS 分片生成（HLS 至少 6-10 秒延迟）
4. **TCP 可靠传输**：避免 UDP 丢包，画面不会撕裂

**HTTP-FLV vs WebSocket-FLV**：
| 维度 | HTTP-FLV | WebSocket-FLV |
|------|---------|---------------|
| 协议 | HTTP/1.1 长连接 | WebSocket |
| 延迟 | 1-3 秒 | 1-3 秒（略低） |
| 穿透性 | 极好（80/443） | 好（80/443） |
| 浏览器兼容 | 全支持 | 需 WebSocket API |
| 服务端开销 | 低 | 略高（WebSocket 握手） |

**实际选择**：
- **直播平台**：优先 HTTP-FLV（兼容性好，CDN 支持完善）
- **内网监控**：WebSocket-FLV（延迟略低 100-200ms）

### 4.9 FLV 解析示例（Go 语言）

```go
package main

import (
	"encoding/binary"
	"fmt"
	"io"
	"os"
)

// FLV Header 结构
type FLVHeader struct {
	Signature [3]byte // 'F', 'L', 'V'
	Version   byte    // 1
	Flags     byte    // 0x05 = audio+video
	DataOffset uint32 // 9
}

// FLV Tag Header 结构
type FLVTagHeader struct {
	TagType   byte   // 8=audio, 9=video, 18=script
	DataSize  uint32 // Tag 数据大小（24 bits）
	Timestamp uint32 // 时间戳（32 bits，含扩展）
	StreamID  uint32 // 固定为 0（24 bits）
}

func parseFLV(filename string) error {
	file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	// 读取 FLV Header
	var header FLVHeader
	if err := binary.Read(file, binary.BigEndian, &header.Signature); err != nil {
		return err
	}
	if string(header.Signature[:]) != "FLV" {
		return fmt.Errorf("invalid FLV signature")
	}

	binary.Read(file, binary.BigEndian, &header.Version)
	binary.Read(file, binary.BigEndian, &header.Flags)
	binary.Read(file, binary.BigEndian, &header.DataOffset)

	fmt.Printf("FLV Header: Version=%d, Flags=0x%02X (Audio=%v, Video=%v), DataOffset=%d\n",
		header.Version, header.Flags,
		header.Flags&0x04 != 0, // Audio flag
		header.Flags&0x01 != 0, // Video flag
		header.DataOffset)

	// 跳过 Previous Tag Size 0
	file.Seek(int64(header.DataOffset), io.SeekStart)
	var prevTagSize uint32
	binary.Read(file, binary.BigEndian, &prevTagSize)

	// 逐个解析 Tag
	tagIndex := 0
	for {
		var tagHeader FLVTagHeader

		// 读取 Tag Type
		if err := binary.Read(file, binary.BigEndian, &tagHeader.TagType); err != nil {
			if err == io.EOF {
				break
			}
			return err
		}

		// 读取 Data Size（3 字节）
		var dataSizeBytes [3]byte
		binary.Read(file, binary.BigEndian, &dataSizeBytes)
		tagHeader.DataSize = uint32(dataSizeBytes[0])<<16 | uint32(dataSizeBytes[1])<<8 | uint32(dataSizeBytes[2])

		// 读取 Timestamp（3 字节 + 1 字节扩展）
		var timestampBytes [3]byte
		binary.Read(file, binary.BigEndian, &timestampBytes)
		var timestampExt byte
		binary.Read(file, binary.BigEndian, &timestampExt)
		tagHeader.Timestamp = uint32(timestampExt)<<24 | uint32(timestampBytes[0])<<16 | uint32(timestampBytes[1])<<8 | uint32(timestampBytes[2])

		// 读取 Stream ID（3 字节）
		var streamIDBytes [3]byte
		binary.Read(file, binary.BigEndian, &streamIDBytes)
		tagHeader.StreamID = uint32(streamIDBytes[0])<<16 | uint32(streamIDBytes[1])<<8 | uint32(streamIDBytes[2])

		// 解析 Tag 类型
		tagTypeName := "Unknown"
		switch tagHeader.TagType {
		case 8:
			tagTypeName = "Audio"
		case 9:
			tagTypeName = "Video"
		case 18:
			tagTypeName = "Script Data"
		}

		fmt.Printf("Tag #%d: Type=%s(%d), DataSize=%d, Timestamp=%dms, StreamID=%d\n",
			tagIndex, tagTypeName, tagHeader.TagType, tagHeader.DataSize, tagHeader.Timestamp, tagHeader.StreamID)

		// 跳过 Tag Data
		file.Seek(int64(tagHeader.DataSize), io.SeekCurrent)

		// 读取 Previous Tag Size
		binary.Read(file, binary.BigEndian, &prevTagSize)

		tagIndex++
		if tagIndex > 20 { // 只解析前 20 个 Tag
			break
		}
	}

	return nil
}

func main() {
	if err := parseFLV("test.flv"); err != nil {
		fmt.Println("Error:", err)
	}
}
```

**运行示例**：
```bash
go run flv_parser.go

# 输出：
# FLV Header: Version=1, Flags=0x05 (Audio=true, Video=true), DataOffset=9
# Tag #0: Type=Script Data(18), DataSize=315, Timestamp=0ms, StreamID=0
# Tag #1: Type=Video(9), DataSize=45, Timestamp=0ms, StreamID=0
# Tag #2: Type=Audio(8), DataSize=4, Timestamp=0ms, StreamID=0
# Tag #3: Type=Video(9), DataSize=12543, Timestamp=33ms, StreamID=0
# Tag #4: Type=Audio(8), DataSize=256, Timestamp=23ms, StreamID=0
# ...
```

---

## 五、RTMP 应用层协议详解

### 5.1 推流完整流程（关键）

RTMP 握手后，需要通过一系列**应用层命令**建立推流会话。这是 RTMP 最核心的部分。

```
OBS/编码器                                    RTMP 服务器
  |                                              |
  | ① TCP 三次握手 + RTMP 握手（C0/S0/C1/S1/C2/S2） |
  |--------------------------------------------->|
  |                                              |
  | ② connect 命令                                |
  |    Transaction ID: 1                         |
  |    Command Object: {                         |
  |      app: "live",                            |
  |      tcUrl: "rtmp://localhost/live"          |
  |    }                                         |
  |--------------------------------------------->|
  |                                              |
  | ③ _result 响应（connect 成功）                 |
  |    Transaction ID: 1                         |
  |    Properties: {                             |
  |      fmsVer: "FMS/3,0,1,123",                |
  |      capabilities: 31                        |
  |    }                                         |
  |<---------------------------------------------|
  |                                              |
  | ④ releaseStream 命令（可选，释放旧流）           |
  |    Transaction ID: 2                         |
  |    Stream Name: "stream-key"                 |
  |--------------------------------------------->|
  |                                              |
  | ⑤ FCPublish 命令（可选，Flash 兼容）            |
  |    Transaction ID: 3                         |
  |    Stream Name: "stream-key"                 |
  |--------------------------------------------->|
  |                                              |
  | ⑥ createStream 命令                          |
  |    Transaction ID: 4                         |
  |--------------------------------------------->|
  |                                              |
  | ⑦ _result 响应（返回 Stream ID）               |
  |    Transaction ID: 4                         |
  |    Stream ID: 1                              |
  |<---------------------------------------------|
  |                                              |
  | ⑧ publish 命令                               |
  |    Stream ID: 1（上一步返回的）                |
  |    Publishing Name: "stream-key"             |
  |    Publishing Type: "live"                   |
  |--------------------------------------------->|
  |                                              |
  | ⑨ onStatus 响应（NetStream.Publish.Start）    |
  |    Stream ID: 1                              |
  |    Info Object: {                            |
  |      level: "status",                        |
  |      code: "NetStream.Publish.Start",        |
  |      description: "Start publishing"         |
  |    }                                         |
  |<---------------------------------------------|
  |                                              |
  | ⑩ @setDataFrame 命令（发送 onMetaData）        |
  |    Stream ID: 1                              |
  |    Data: {                                   |
  |      width: 1920, height: 1080,              |
  |      videodatarate: 2500, framerate: 30,     |
  |      videocodecid: 7 (H.264)                 |
  |    }                                         |
  |--------------------------------------------->|
  |                                              |
  | ⑪ Video 消息（AVC Sequence Header）           |
  |    Frame Type: 5（配置帧）                     |
  |    Codec ID: 7（H.264）                       |
  |    AVC Packet Type: 0（SPS/PPS）              |
  |--------------------------------------------->|
  |                                              |
  | ⑫ Audio 消息（AAC Sequence Header）           |
  |    Sound Format: 10（AAC）                    |
  |    AAC Packet Type: 0（AudioSpecificConfig）  |
  |--------------------------------------------->|
  |                                              |
  | ⑬ Video 消息（I 帧）                          |
  |    Frame Type: 1（关键帧）                     |
  |    AVC Packet Type: 1（NALU 数据）            |
  |--------------------------------------------->|
  |                                              |
  | ⑭ Audio 消息（AAC 音频帧）                     |
  | ⑮ Video 消息（P 帧）                          |
  | ⑯ Audio 消息（AAC 音频帧）                     |
  |--------------------------------------------->| (持续推流)
```

**关键要点**：
- **Transaction ID**：用于匹配命令与响应，从 1 开始递增
- **Stream ID**：`createStream` 返回的流标识符，后续 `publish` 等命令必须使用
- **Sequence Header 必须先发**：播放器需要 SPS/PPS 和 AAC Config 才能解码后续帧
- **releaseStream/FCPublish**：OBS 等工具会发送，但多数服务器忽略（用于兼容 Flash）

### 5.2 拉流完整流程

```
播放器                                        RTMP 服务器
  |                                              |
  | ① TCP + RTMP 握手                            |
  |--------------------------------------------->|
  |                                              |
  | ② connect 命令                                |
  |    Command Object: { app: "live" }           |
  |--------------------------------------------->|
  |                                              |
  | ③ _result 响应                               |
  |<---------------------------------------------|
  |                                              |
  | ④ createStream 命令                          |
  |--------------------------------------------->|
  |                                              |
  | ⑤ _result 响应（Stream ID: 1）                |
  |<---------------------------------------------|
  |                                              |
  | ⑥ play 命令                                  |
  |    Stream ID: 1                              |
  |    Stream Name: "stream-key"                 |
  |    Start: -2（表示播放直播流，非 VOD）          |
  |--------------------------------------------->|
  |                                              |
  | ⑦ onStatus 响应（NetStream.Play.Start）       |
  |<---------------------------------------------|
  |                                              |
  | ⑧ onMetaData（流元数据）                      |
  |<---------------------------------------------|
  |                                              |
  | ⑨ Video（AVC Sequence Header - SPS/PPS）     |
  |<---------------------------------------------|
  |                                              |
  | ⑩ Audio（AAC Sequence Header）               |
  |<---------------------------------------------|
  |                                              |
  | ⑪ Video（最近的 I 帧，来自 GOP 缓存）           |
  |<---------------------------------------------|
  |                                              |
  | ⑫ Video/Audio 持续推送                        |
  |<---------------------------------------------| (持续拉流)
```

**play 命令的 start 参数**：
- `-2`：播放直播流（Live Stream），从当前位置开始
- `-1`：播放 VOD，从最近的关键帧开始
- `0`：从头开始播放
- `> 0`：从指定秒数开始播放

### 5.3 AMF0 编码详解（核心）

RTMP 的命令消息使用 **AMF0**（Action Message Format 0）序列化，类似 JSON 的二进制格式。

**AMF0 类型标识**（1 字节）：
```
0x00  Number      (8 字节 double)
0x01  Boolean     (1 字节)
0x02  String      (2 字节长度 + UTF-8 数据)
0x03  Object      (键值对，以 0x00 0x00 0x09 结束)
0x04  MovieClip   (已废弃)
0x05  Null        (无数据)
0x06  Undefined   (无数据)
0x08  ECMA Array  (4 字节长度 + 键值对)
0x0A  Strict Array (4 字节长度 + 元素列表)
0x0B  Date        (8 字节 double + 2 字节时区)
```

**connect 命令的完整二进制结构**：

假设发送 `connect("live")` 到 `rtmp://localhost/live`：

```
RTMP Chunk Header (12 字节，fmt=0):
  01           // Basic Header: fmt=0, cs_id=3（控制消息）
  00 00 00     // Timestamp = 0
  00 00 2F     // Message Length = 47 字节
  14           // Message Type ID = 20 (AMF0 Command)
  00 00 00 00  // Message Stream ID = 0

AMF0 Command Message (47 字节):
  // Command Name: "connect"
  02                // AMF0 String
  00 07             // 长度 = 7
  63 6F 6E 6E 65 63 74  // "connect" UTF-8

  // Transaction ID: 1
  00                // AMF0 Number
  3F F0 00 00 00 00 00 00  // double(1.0)

  // Command Object
  03                // AMF0 Object
    // 键值对 1: "app" = "live"
    00 03             // 键长度 = 3
    61 70 70          // "app" UTF-8
    02                // AMF0 String（值类型）
    00 04             // 值长度 = 4
    6C 69 76 65       // "live" UTF-8

    // 键值对 2: "tcUrl" = "rtmp://localhost/live"
    00 05             // 键长度 = 5
    74 63 55 72 6C    // "tcUrl" UTF-8
    02                // AMF0 String
    00 16             // 值长度 = 22
    72 74 6D 70 3A 2F 2F 6C 6F 63 61 6C 68 6F 73 74 2F 6C 69 76 65  
    // "rtmp://localhost/live" UTF-8

    // 对象结束标记
    00 00 09          // Object End Marker
```

**Go 语言解析 AMF0 示例**：

```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"math"
)

// AMF0 类型常量
const (
	AMF0Number = 0x00
	AMF0Bool   = 0x01
	AMF0String = 0x02
	AMF0Object = 0x03
	AMF0Null   = 0x05
)

// 读取 AMF0 String
func readAMF0String(buf *bytes.Buffer) (string, error) {
	marker, _ := buf.ReadByte()
	if marker != AMF0String {
		return "", fmt.Errorf("expected AMF0 String, got 0x%02X", marker)
	}
	var length uint16
	binary.Read(buf, binary.BigEndian, &length)
	data := make([]byte, length)
	buf.Read(data)
	return string(data), nil
}

// 读取 AMF0 Number
func readAMF0Number(buf *bytes.Buffer) (float64, error) {
	marker, _ := buf.ReadByte()
	if marker != AMF0Number {
		return 0, fmt.Errorf("expected AMF0 Number, got 0x%02X", marker)
	}
	var bits uint64
	binary.Read(buf, binary.BigEndian, &bits)
	return math.Float64frombits(bits), nil
}

// 读取 AMF0 Object
func readAMF0Object(buf *bytes.Buffer) (map[string]interface{}, error) {
	marker, _ := buf.ReadByte()
	if marker != AMF0Object {
		return nil, fmt.Errorf("expected AMF0 Object, got 0x%02X", marker)
	}

	obj := make(map[string]interface{})
	for {
		// 读取键长度
		var keyLen uint16
		binary.Read(buf, binary.BigEndian, &keyLen)
		if keyLen == 0 {
			// 对象结束标记：0x00 0x00 0x09
			endMarker, _ := buf.ReadByte()
			if endMarker == 0x09 {
				break
			}
		}

		// 读取键名
		keyBytes := make([]byte, keyLen)
		buf.Read(keyBytes)
		key := string(keyBytes)

		// 读取值类型
		valueType, _ := buf.ReadByte()
		buf.UnreadByte()

		// 根据类型解析值
		switch valueType {
		case AMF0String:
			value, _ := readAMF0String(buf)
			obj[key] = value
		case AMF0Number:
			value, _ := readAMF0Number(buf)
			obj[key] = value
		default:
			return nil, fmt.Errorf("unsupported value type: 0x%02X", valueType)
		}
	}
	return obj, nil
}

func main() {
	// 模拟 connect 命令的 AMF0 数据（省略 Chunk Header）
	data := []byte{
		// Command Name: "connect"
		0x02, 0x00, 0x07, 0x63, 0x6F, 0x6E, 0x6E, 0x65, 0x63, 0x74,
		// Transaction ID: 1.0
		0x00, 0x3F, 0xF0, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
		// Command Object
		0x03,
		0x00, 0x03, 0x61, 0x70, 0x70, // "app"
		0x02, 0x00, 0x04, 0x6C, 0x69, 0x76, 0x65, // "live"
		0x00, 0x00, 0x09, // Object End
	}

	buf := bytes.NewBuffer(data)

	commandName, _ := readAMF0String(buf)
	transactionID, _ := readAMF0Number(buf)
	commandObj, _ := readAMF0Object(buf)

	fmt.Printf("Command: %s\n", commandName)
	fmt.Printf("Transaction ID: %.0f\n", transactionID)
	fmt.Printf("Object: %+v\n", commandObj)
}
```

**运行输出**：
```
Command: connect
Transaction ID: 1
Object: map[app:live]
```

### 5.4 GOP 缓存机制（首屏优化）

**什么是 GOP 缓存**：
服务器缓存最近一个 **GOP**（Group of Pictures），包括：
- 最近的 **I 帧**（关键帧）
- I 帧后的 **P/B 帧**（直到下一个 I 帧）
- **Sequence Header**（SPS/PPS + AAC Config）
- **onMetaData**

**为什么需要 GOP 缓存**：
- 新观众加入时，如果等待下一个 I 帧才能播放，可能需要等待 2-10 秒（取决于关键帧间隔）
- 有了 GOP 缓存，服务器立即推送缓存的 I 帧，观众秒开

**流程对比**：

```
❌ 无 GOP 缓存：
新观众 play → 等待下一个 I 帧 → 开始播放（延迟 2-10 秒）
  P P P P P P [等待中...] I P P P [播放开始]

✅ 有 GOP 缓存：
新观众 play → 立即收到缓存的 I 帧 → 播放（延迟 < 1 秒）
  [立即推送 I P P P] P P P I P P P
```

**nginx-rtmp 配置**：
```nginx
application live {
    live on;
    wait_key on;      # 等待关键帧再发送给拉流端（避免首帧花屏）
    wait_video on;    # 等待视频帧再开始（音频可能先到）
    sync 10ms;        # 音视频同步容忍时间
    
    # GOP 缓存（默认开启）
    # 新观众会立即收到最近的 I 帧 + onMetaData
}
```

**编码器设置**（OBS / FFmpeg）：
```bash
# 关键帧间隔设置为 2 秒（GOP = 2s）
# 这意味着新观众最多等待 2 秒就能看到 I 帧
ffmpeg -i input.mp4 \
  -c:v libx264 -g 60 -keyint_min 60 \  # 30fps × 2s = 60 帧
  -f flv rtmp://localhost/live/test
```

**OBS Studio 设置**：
```
设置 → 输出 → 串流
  视频编码器: x264
  关键帧间隔: 2（秒）  # GOP = 2s，推荐值
```

**GOP 缓存的权衡**：
| 关键帧间隔 | 优点 | 缺点 |
|-----------|------|------|
| 1 秒 | 首屏极快，seek 精度高 | 码率高 30-50% |
| 2 秒 | **推荐**，平衡延迟与码率 | - |
| 5 秒 | 码率低 | 新观众等待久，seek 不精确 |
| 10 秒 | 极致节省带宽 | 首屏慢，不适合直播 |

### 5.5 Chunk Size 与性能调优

**默认 Chunk Size 太小的问题**：
- RTMP 握手后，默认 Chunk Size 是 **128 字节**
- 对于高清视频，一个 I 帧可能 100KB+，会被切分成 **800+ 个 Chunk**
- 每个 Chunk 都有 Chunk Header（1-12 字节），**开销巨大**

**计算示例**：
```
1920×1080 30fps 视频，I 帧 150KB：
- 默认 128B Chunk：150000 / 128 = 1172 个 Chunk
- 假设平均 Chunk Header 3 字节：1172 × 3 = 3516 字节开销（2.3%）

改为 4096B Chunk：150000 / 4096 = 37 个 Chunk
- Chunk Header 开销：37 × 3 = 111 字节（0.07%）
```

**Set Chunk Size 命令**：
```
客户端/服务器发送 Message Type 1（Set Chunk Size）：
  +--------+--------+--------+--------+
  | Chunk Size (4 字节，最高位必须为 0)  |
  +--------+--------+--------+--------+

示例：设置 Chunk Size 为 4096
  00 00 10 00  →  0x00001000 = 4096
```

**nginx-rtmp 配置**：
```nginx
rtmp {
    server {
        listen 1935;
        chunk_size 4096;  # 默认 4096（已优化）
        
        application live {
            live on;
        }
    }
}
```

**FFmpeg 推流设置**（客户端也要设置）：
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -c:a aac \
  -f flv -rtmp_buffer 100 -rtmp_live 1 \
  rtmp://localhost/live/test
```

**权衡**：
| Chunk Size | 优点 | 缺点 |
|-----------|------|------|
| 128B（默认） | 低延迟（更细粒度切分） | Header 开销大 |
| 4096B | **推荐**，开销低，延迟仍可接受 | - |
| 60000B | 极致性能（一个 Chunk 可能包含完整帧） | 高延迟（必须等 60KB 接收完） |

### 5.6 时间戳与 DTS/PTS 详解

**RTMP 时间戳规则**：
- 时间戳单位：**毫秒**
- 时间戳范围：32 位无符号整数（0 ~ 4294967295 ms ≈ 49.7 天）
- 时间戳必须**单调递增**（除了 Sequence Header，可以为 0）

**时间戳回绕处理**（超过 49.7 天的直播流）：
```
当时间戳达到 0xFFFFFFFF 后，下一个包回到 0：
  Packet N:   timestamp = 0xFFFFFFFF (4294967295 ms)
  Packet N+1: timestamp = 0x00000000 (0 ms)

播放器需检测回绕：
  if (current_ts < previous_ts && previous_ts > 0xF0000000) {
      // 发生回绕，累加 epoch
      epoch += 0x100000000;
  }
  actual_ts = epoch + current_ts;
```

**Composition Time Offset（CTO）**：
- 用于处理 **B 帧**（双向预测帧）
- B 帧的**解码顺序**（DTS）与**显示顺序**（PTS）不同

**示例**（包含 B 帧的 GOP）：
```
显示顺序（PTS）: I  P  B  B  P
解码顺序（DTS）: I  P  P  B  B
                ↑      ↑
              先解码 P，才能解码依赖它的 B

RTMP 时间戳 = DTS（解码时间）
Composition Time Offset = PTS - DTS

Packet 1: I 帧, timestamp=0,   CTO=0    (PTS=0)
Packet 2: P 帧, timestamp=33,  CTO=0    (PTS=33)
Packet 3: P 帧, timestamp=100, CTO=0    (PTS=100)
Packet 4: B 帧, timestamp=66,  CTO=0    (PTS=66)   ← 解码时间在 P 后，显示在 P 前
Packet 5: B 帧, timestamp=100, CTO=-34  (PTS=66)
```

**H.264 NALU Packet 中的 CTO**：
```
+-------------+-------------------+--------------+
| AVC Packet  | Composition Time  | NALU Data    |
|   Type(1B)  |   Offset (3B)     |              |
+-------------+-------------------+--------------+
  = 1（NALU）   = PTS - DTS（有符号）
```

**大多数直播场景不使用 B 帧**：
- OBS 默认编码配置：`tune=zerolatency`，不生成 B 帧
- CTO 始终为 0，时间戳即 PTS

### 5.7 RTMP 协议状态码（关键）

**onStatus 响应中的 code 字段**（播放器/推流端据此判断状态）：

**连接状态**：
| Code | 含义 | 触发场景 |
|------|------|----------|
| `NetConnection.Connect.Success` | 连接成功 | connect 命令成功 |
| `NetConnection.Connect.Rejected` | 连接被拒绝 | 鉴权失败、IP 黑名单 |
| `NetConnection.Connect.Failed` | 连接失败 | 网络错误、服务器崩溃 |
| `NetConnection.Connect.Closed` | 连接关闭 | 服务器主动断开 |
| `NetConnection.Connect.InvalidApp` | 应用名不存在 | `connect("invalid_app")` |

**推流状态**：
| Code | 含义 | 触发场景 |
|------|------|----------|
| `NetStream.Publish.Start` | 推流开始 | publish 命令成功 |
| `NetStream.Publish.BadName` | 流名称无效 | streamKey 为空或非法字符 |
| `NetStream.Publish.Idle` | 推流空闲 | 长时间未发送数据 |
| `NetStream.Publish.Denied` | 推流被拒绝 | 权限不足、流已被占用 |

**拉流状态**：
| Code | 含义 | 触发场景 |
|------|------|----------|
| `NetStream.Play.Start` | 播放开始 | play 命令成功 |
| `NetStream.Play.StreamNotFound` | 流不存在 | streamKey 不存在 |
| `NetStream.Play.Failed` | 播放失败 | 网络错误 |
| `NetStream.Play.Stop` | 播放停止 | 推流端断开 |
| `NetStream.Play.Reset` | 播放重置 | 服务器重新发送 Sequence Header |
| `NetStream.Play.PublishNotify` | 推流开始通知 | 等待的流开始推送 |
| `NetStream.Play.UnpublishNotify` | 推流停止通知 | 推流端停止 |

**错误处理示例**（Go 客户端）：
```go
func handleOnStatus(code string) error {
    switch code {
    case "NetStream.Publish.Start":
        log.Info("推流成功，开始发送媒体数据")
        return nil
    case "NetStream.Publish.BadName":
        return fmt.Errorf("streamKey 无效，请检查配置")
    case "NetStream.Publish.Denied":
        return fmt.Errorf("推流被拒绝，流可能已被占用")
    case "NetConnection.Connect.Rejected":
        return fmt.Errorf("鉴权失败，请检查推流密钥")
    default:
        log.Warn("未知状态码: %s", code)
        return nil
    }
}
```

### 5.8 Wireshark 抓包实战

**抓包 RTMP 推流流程**：

**1. 启动抓包**：
```bash
# macOS/Linux
sudo tcpdump -i any -w rtmp.pcap port 1935

# 或使用 Wireshark GUI
# 过滤器：tcp.port == 1935
```

**2. 开始推流**：
```bash
ffmpeg -re -i test.mp4 -c copy -f flv rtmp://localhost/live/test
```

**3. 停止抓包，用 Wireshark 分析 rtmp.pcap**

**抓包分析步骤**：

**① 找到 TCP 三次握手**：
```
过滤器：tcp.stream eq 0 and tcp.flags.syn == 1
```

**② 找到 RTMP 握手（C0/S0/C1/S1/C2/S2）**：
```
过滤器：rtmp
→ 找到 TCP payload 的前 1537 字节（C0 + C1）
→ 验证 C0 = 0x03（版本号）
```

**③ 找到 connect 命令**：
```
过滤器：rtmp.handshake_type == 0 and rtmp.command.name == "connect"
→ 右键 → Follow → TCP Stream
→ 看到 AMF0 编码的 "connect" 字符串（0x02 0x00 0x07 "connect"）
```

**④ 找到 Sequence Header**：
```
过滤器：rtmp.video
→ 找到 Frame Type = 5（配置帧）的包
→ 展开 RTMP Video Data：
   - Frame Type: 5 (video info/command frame)
   - Codec ID: 7 (H.264)
   - AVC Packet Type: 0 (AVC Sequence Header)
→ 提取 SPS/PPS（在 Hex 视图中查找 0x67/0x68 NAL 单元起始码）
```

**⑤ 找到第一个 I 帧**：
```
过滤器：rtmp.video && rtmp.video.frametype == 1
→ Frame Type = 1（关键帧）
→ AVC Packet Type = 1（NALU）
→ 看 timestamp 字段（应该接近 0ms）
```

**⑥ 统计流量**：
```
Statistics → Conversations → TCP
→ 看 1935 端口的流量
→ 计算平均码率：Bytes / Duration
```

**常见问题定位**：
| 现象 | 抓包特征 | 原因 |
|------|---------|------|
| 推流失败 | 只有 connect，无 publish | 鉴权失败或 streamKey 错误 |
| 播放花屏 | 缺少 AVC Sequence Header | 播放器未收到 SPS/PPS |
| 延迟高 | Timestamp 增长不均匀 | 编码器卡顿或网络抖动 |
| 断流重连 | TCP RST 包频繁出现 | 网络不稳定 |

---

## 六、RTMP 变种协议

| 变种 | 说明 | 端口 | 用途 |
|------|------|------|------|
| **RTMP** | 基础版本 | 1935 | 标准推流 |
| **RTMPS** | TLS 加密 | 443 | 安全推流（YouTube/Facebook 要求） |
| **RTMPT** | HTTP 隧道 | 80 | 穿透防火墙 |
| **RTMPE** | Adobe 私有加密 | 1935 | 已弃用 |
| **SRT** | 非 RTMP，替代方案 | 自定义 | 弱网推流 |

**RTMPS 配置（FFmpeg）**：
```bash
ffmpeg -i input.mp4 -c:v libx264 -c:a aac \
  -f flv "rtmps://live.twitch.tv/live/your-stream-key"
```

---

## 六、RTMP vs RTSP vs HLS

| 维度 | RTMP | RTSP | HLS |
|------|------|------|-----|
| **传输层** | TCP | TCP/UDP | HTTP |
| **延迟** | 1-3 秒 | 1-3 秒 | 10-30 秒 |
| **主要用途** | 推流（编码器→服务器） | 拉流（摄像头） | CDN 分发 |
| **防火墙穿透** | 好（1935/443） | 较差（UDP） | 极好（80/443） |
| **浏览器支持** | 无（Flash 已死） | 无 | 原生支持 |
| **CDN 支持** | 差 | 差 | 极好 |

**典型直播架构**：
```
[摄像头] --RTSP--> [流媒体服务器]
[OBS]   --RTMP--> [流媒体服务器] --HLS/DASH--> [CDN] --> [用户浏览器]
                                 --HTTP-FLV--> [监控播放器]
                                 --RTMP------> [内部播放器]
```

### RTMP 协议的局限性（技术债务）

虽然 RTMP 仍是推流标准，但协议本身存在历史包袱：

**1. 无法穿透严格防火墙**
- RTMP 使用非标准端口 **1935**，企业防火墙常屏蔽
- RTMPS（443）和 RTMPT（80 HTTP 隧道）可解决，但增加复杂度
- 对比：HLS 走标准 HTTP/HTTPS，天然穿透

**2. 不支持现代网络协议**
- 基于 TCP 单连接，无法利用 HTTP/2 多路复用
- 无法利用 QUIC 的 0-RTT 握手和连接迁移
- 对比：WebRTC 使用 UDP + DTLS，弱网表现更好

**3. CDN 缓存不友好**
- RTMP 是**有状态长连接**，CDN 必须保持连接
- 无法像 HLS 那样缓存 TS 分片到边缘节点
- 导致 RTMP 分发成本高，难以大规模 CDN 加速

**4. 浏览器原生不支持**
- Flash Player 已死（2020），浏览器无法直接播放 RTMP
- 必须转码为 HLS/HTTP-FLV/WebRTC
- 导致推流到播放的链路复杂化

**5. 编码格式扩展困难**
- 标准 RTMP 只支持 H.264 + AAC
- Enhanced RTMP 扩展了 H.265/AV1，但**不向后兼容**
- 对比：HLS 的 fMP4 容器天然支持任意编码

**6. 拥塞控制依赖 TCP**
- 丢包时 TCP 重传导致延迟抖动
- 弱网下（丢包率 > 5%）性能急剧下降
- SRT（UDP + ARQ）在弱网场景完胜 RTMP

**为什么 RTMP 还在用**：
- **工具链成熟**：OBS/FFmpeg/所有直播平台均支持
- **替代方案不成熟**：WebRTC 推流标准化晚（WHIP 2022 年才定稿）
- **惯性**：改造成本高，现有架构已围绕 RTMP 设计

**未来趋势**：
```
推流协议演进路线：
  RTMP（当前主流） 
    ↓
  SRT（弱网场景，2017）
    ↓
  WebRTC/WHIP（低延迟互动，2022+）
    ↓
  WebTransport（基于 QUIC，实验中）
```

---

## 七、动手实践

### 实践 1：部署 nginx-rtmp 服务器

```bash
# Ubuntu 安装
apt-get install nginx libnginx-mod-rtmp

# nginx.conf 添加 rtmp 配置
cat >> /etc/nginx/nginx.conf << 'EOF'
rtmp {
    server {
        listen 1935;
        chunk_size 4096;

        application live {
            live on;
            record off;

            # 转 HLS
            hls on;
            hls_path /var/www/html/hls;
            hls_fragment 2s;
            hls_playlist_length 10s;
        }
    }
}

http {
    server {
        listen 8080;
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /var/www/html;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }
    }
}
EOF

nginx -s reload
```

### 实践 2：FFmpeg 推流测试

```bash
# 1. 用本地文件循环推流（测试）
ffmpeg -re -stream_loop -1 \
  -i test.mp4 \
  -c:v libx264 -preset veryfast -b:v 2000k \
  -c:a aac -b:a 128k \
  -f flv rtmp://localhost/live/test

# 2. 摄像头推流（macOS）
ffmpeg -f avfoundation -framerate 30 -i "0:0" \
  -c:v libx264 -preset veryfast -b:v 1500k \
  -c:a aac -b:a 128k \
  -f flv rtmp://localhost/live/test

# 3. 屏幕录制推流（Linux）
ffmpeg -f x11grab -r 30 -s 1920x1080 -i :0.0 \
  -c:v libx264 -preset veryfast -b:v 2000k \
  -f flv rtmp://localhost/live/screen

# 4. 拉取 RTMP 流并保存
ffmpeg -i rtmp://localhost/live/test -c copy output.flv
```

### 实践 3：OBS Studio 推流配置

在 OBS 中设置：
```
设置 → 直播
  服务器：rtmp://localhost/live
  串流密钥：test
  
  (可选) 高级输出
  视频编码器：x264
  码率：2500 Kbps
  关键帧间隔：2 秒（GOP = 2s）
  Profile：Main
  Preset：veryfast
```

**关键参数说明**：
- **串流密钥**：即 streamKey，服务器用它区分不同直播流
- **关键帧间隔**：建议 2 秒，平衡延迟与质量
- **Preset**：`veryfast` 在 CPU 占用和质量间取得平衡

---

## 八、常见问题排查

### 问题 1：推流断线重连

**现象**：网络波动时推流中断，OBS 显示 "Connection to rtmp://... failed"。

**排查步骤**：
```bash
# 1. 检查网络连通性
ping <服务器IP>
telnet <服务器IP> 1935

# 2. 查看服务器日志
tail -f /var/log/nginx/error.log | grep rtmp

# 3. 抓包分析
tcpdump -i any port 1935 -w rtmp_debug.pcap
```

**OBS 客户端解决方案**：
```
设置 → 高级 → 网络
  启用网络优化：开
  自动重连：开
  重连延迟：2 秒（推荐）
  最大重试次数：20

设置 → 输出 → 串流
  编码器缓冲大小：自动（或手动设置为 3000-5000 KB）
```

**nginx-rtmp 服务器端配置**：
```nginx
application live {
    live on;
    wait_key on;           # 等待关键帧再发给拉流端
    wait_video on;         # 等待视频帧再开始
    sync 10ms;             # 音视频同步容忍时间
    
    drop_idle_publisher 10s;  # 10秒无数据自动断开推流端
    idle_streams off;         # 允许预创建流（推流端未到也能拉流）
}
```

**常见原因**：
- 上行带宽不足（推流码率超过带宽）
- 服务器 `max_connections` 限制
- 防火墙/安全组未放行 1935 端口

### 问题 2：延迟过高（超过 5 秒）

**目标**：RTMP 正常延迟 1-3 秒，超过 5 秒需排查。

**排查清单**：

**① 编码器关键帧间隔**（最常见）：
```bash
# FFmpeg 查看流的关键帧间隔
ffprobe -show_frames rtmp://localhost/live/test 2>&1 | grep "pict_type=I"

# OBS 设置：关键帧间隔 = 2 秒（推荐）
```

**② 服务器 GOP 缓存**：
```nginx
application live {
    live on;
    wait_key on;  # 启用 GOP 缓存，新观众秒开
                  # 关闭此项可降低 1-2 秒延迟，但新观众需等待下一个 I 帧
}
```

**③ 播放器缓冲设置**：
```javascript
// flv.js 播放器配置
let player = flvjs.createPlayer({
    type: 'flv',
    url: 'http://localhost:8080/live/test.flv',
    isLive: true
}, {
    enableWorker: true,
    enableStashBuffer: false,  // 禁用缓冲队列
    stashInitialSize: 128,     // 初始缓冲 128KB（默认 384KB）
    lazyLoad: false,
    lazyLoadMaxDuration: 0.2   // 最大缓冲 0.2 秒
});
```

**④ 网络延迟**：
```bash
# 检查 RTT（往返时间）
ping <服务器IP>
# 期望：< 50ms（国内），< 200ms（跨国）

# 使用 mtr 检查路由
mtr <服务器IP>
```

### 问题 3：推流成功但拉流无画面

**现象**：推流端显示正常，但播放器黑屏或报错 "Failed to load media"。

**排查步骤**：

**① 验证流是否存在**：
```bash
# 使用 FFmpeg 拉流测试
ffmpeg -i rtmp://localhost/live/test -c copy -f null -
# 如果能输出 "frame= xxxx"，说明流正常

# 使用 ffprobe 查看流信息
ffprobe rtmp://localhost/live/test
# 应该输出视频编码、分辨率、帧率等信息
```

**② 检查 Sequence Header**：
```bash
# 抓包查看是否有 AVC Sequence Header（Frame Type = 5）
tcpdump -i any port 1935 -w check.pcap
# 用 Wireshark 打开，过滤：rtmp.video.frametype == 5
```

**③ 检查编码格式兼容性**：
```bash
# 某些播放器不支持 H.265，必须用 H.264
ffprobe rtmp://localhost/live/test 2>&1 | grep "Video:"
# 输出应为：Video: h264 (High)
```

**④ 浏览器播放器兼容性**：
```javascript
// 检查浏览器是否支持 MSE
if (flvjs.isSupported()) {
    console.log('Browser supports flv.js');
} else {
    console.error('Browser does NOT support MSE!');
    // Safari 可能不支持，需使用 HLS
}
```

**⑤ CORS 跨域问题**（HTTP-FLV）：
```nginx
# nginx 配置
location /live {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, OPTIONS';
    add_header Access-Control-Allow-Headers 'Range';
}
```

### 问题 4：推流被拒绝（NetStream.Publish.BadName）

**现象**：OBS 报错 "Failed to connect to server" 或日志显示 "BadName"。

**原因与解决**：

**① streamKey 包含非法字符**：
```bash
# ❌ 非法：包含空格、中文、特殊符号
rtmp://localhost/live/my stream key
rtmp://localhost/live/直播间123

# ✅ 合法：只包含字母、数字、下划线、连字符
rtmp://localhost/live/my-stream-key
rtmp://localhost/live/live_123
```

**② 流名已被占用**：
```nginx
application live {
    live on;
    allow publish all;    # 允许所有 IP 推流
    deny publish all;     # 或使用白名单限制

    # 阻止重复推流
    on_publish http://localhost:8080/auth;  # 鉴权接口返回 403 拒绝
}
```

**③ 权限鉴权失败**：
```bash
# 带鉴权参数的推流 URL
rtmp://localhost/live/stream?sign=abc123&expire=1234567890

# nginx-rtmp 验证签名
on_publish http://localhost:8080/verify_sign;
```

### 问题 5：音视频不同步

**现象**：画面与声音延迟超过 200ms，说话与口型对不上。

**排查**：

**① 编码器音视频时间戳**：
```bash
# 使用 ffprobe 查看时间戳
ffprobe -show_packets rtmp://localhost/live/test 2>&1 | grep "pts="
# 音频和视频的 pts 应该递增且对齐
```

**② nginx-rtmp sync 参数**：
```nginx
application live {
    sync 10ms;   # 默认 300ms，设置过大会导致不同步
}
```

**③ 播放器同步策略**：
```javascript
// flv.js 播放器
{
    fixAudioTimestampGap: true,  // 修复音频时间戳跳变
    autoCleanupSourceBuffer: true // 自动清理过期缓冲
}
```

### 问题 6：Connection Refused（端口不通）

**现象**：推流端报错 "Connection refused" 或 "Unable to connect"。

**排查**：

**① 检查服务监听**：
```bash
# 检查 1935 端口是否监听
netstat -tlnp | grep 1935
# 或
ss -tlnp | grep 1935

# 应该输出：tcp  0  0  0.0.0.0:1935  0.0.0.0:*  LISTEN  <pid>/nginx
```

**② 检查防火墙**：
```bash
# Ubuntu/Debian (ufw)
sudo ufw status
sudo ufw allow 1935/tcp

# CentOS/RHEL (firewalld)
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=1935/tcp --permanent
sudo firewall-cmd --reload

# 云服务器：检查安全组是否放行 1935
```

**③ 检查 nginx 配置**：
```bash
# 验证配置语法
nginx -t

# 查看 nginx 状态
systemctl status nginx

# 查看错误日志
tail -f /var/log/nginx/error.log
```

**④ 本地测试**：
```bash
# 在服务器本地推流测试
ffmpeg -re -i test.mp4 -c copy -f flv rtmp://127.0.0.1/live/test
# 如果本地成功、远程失败，说明是防火墙问题
```

---

## 九、本篇小结

### 核心知识体系

| 主题 | 核心要点 |
|------|----------|
| **协议定位** | 推流标准；Flash 已死，推流协议仍是行业默认 |
| **握手流程** | TCP 三次握手 → RTMP 握手（C0/S0/C1/S1/C2/S2，共 3074 字节）|
| **应用层协议** | connect → createStream → publish/play → Sequence Header → 媒体流 |
| **Chunk 机制** | 消息切块传输，默认 128B；建议 4096B 减少 Header 开销 |
| **AMF 编码** | RTMP 命令使用 AMF0 序列化（类似 JSON 的二进制格式）|
| **FLV 容器** | RTMP 传输的本质是 FLV Tag 流，HTTP-FLV 低延迟的关键 |
| **Sequence Header** | SPS/PPS（视频）+ AAC Config（音频）必须先发，播放器才能解码 |
| **GOP 缓存** | 服务器缓存最近的 I 帧，新观众秒开（推荐关键帧间隔 2 秒）|
| **时间戳** | 毫秒单位，单调递增；Composition Time Offset 处理 B 帧 |
| **状态码** | NetStream.Publish.Start / NetStream.Play.StreamNotFound 等 |
| **变种协议** | RTMPS（TLS 加密）/ RTMPT（HTTP 隧道）|
| **协议局限** | 不支持 HTTP/2、CDN 不友好、浏览器原生不支持 |

### 关键技术要点

**1. 完整推流流程（必须掌握）**：
```
① TCP + RTMP 握手
② connect 命令（Transaction ID: 1）
③ _result 响应
④ createStream 命令（Transaction ID: 2）
⑤ _result 响应（返回 Stream ID: 1）
⑥ publish 命令（Stream ID: 1, streamKey, "live"）
⑦ onStatus 响应（NetStream.Publish.Start）
⑧ onMetaData（流元数据）
⑨ Video Sequence Header（SPS/PPS）
⑩ Audio Sequence Header（AAC Config）
⑪ Video I 帧 + Audio 帧（持续推流）
```

**2. Chunk Size 性能调优**：
- 默认 128B → 建议 4096B
- 减少 Chunk Header 开销：从 2.3% 降至 0.07%
- nginx 配置：`chunk_size 4096;`

**3. GOP 缓存原理**：
- 缓存内容：最近的 I 帧 + P/B 帧 + Sequence Header + onMetaData
- 作用：新观众加入时立即推送 I 帧，无需等待下一个关键帧
- 关键帧间隔推荐：2 秒（平衡延迟与码率）

**4. RTMP 与 FLV 的关系**：
- RTMP Video/Audio 消息（Type 8/9）= FLV Audio/Video Tag
- RTMP → FLV：去掉 Chunk Header，保留 Tag 数据
- HTTP-FLV 延迟低的原因：HTTP 长连接 + 流式推送 FLV Tag

**5. Wireshark 抓包关键点**：
- 过滤器：`tcp.port == 1935`
- 找到 connect 命令：AMF0 编码的 "connect" 字符串
- 找到 Sequence Header：Frame Type = 5, AVC Packet Type = 0
- 找到第一个 I 帧：Frame Type = 1, timestamp 接近 0ms

**6. 协议状态码（错误排查必备）**：
- `NetStream.Publish.Start`：推流成功
- `NetStream.Publish.BadName`：streamKey 无效
- `NetStream.Play.StreamNotFound`：流不存在
- `NetConnection.Connect.Rejected`：鉴权失败

**7. 常见问题定位**：
- 延迟过高：检查关键帧间隔（推荐 2 秒）、GOP 缓存、播放器缓冲
- 推流断线：检查上行带宽、编码器缓冲、OBS 自动重连设置
- 拉流无画面：验证 Sequence Header、检查编码格式兼容性、CORS 配置
- 音视频不同步：检查 sync 参数（推荐 10ms）、播放器同步策略

### 技术演进方向

```
推流协议的未来：
  RTMP（当前主流，2002-至今）
    ↓  弱网场景
  SRT（UDP + ARQ，2017+）
    ↓  低延迟互动
  WebRTC/WHIP（WebRTC 推流标准，2022+）
    ↓  基于 QUIC
  WebTransport（实验阶段）
```

**为什么 RTMP 还在用**：
- 工具链成熟（OBS/FFmpeg/所有直播平台均支持）
- 替代方案标准化晚（WHIP 2022 年才定稿）
- 历史惯性（现有架构围绕 RTMP 设计）

**但 RTMP 的局限**：
- 无法穿透严格防火墙（非标准端口 1935）
- 不支持 HTTP/2 多路复用和 QUIC
- CDN 缓存不友好（有状态长连接）
- 浏览器原生不支持（Flash 已死）
- 弱网表现差（TCP 丢包重传导致延迟抖动）

### 记忆口诀

- **RTMP = 推流标准**（Flash 死了，推流协议仍活）
- **端口 1935**（RTMPS 走 443）
- **核心流程**：握手 → connect → createStream → publish → Sequence Header → 媒体流
- **性能优化**：Chunk Size 4096 + 关键帧间隔 2 秒 + GOP 缓存开启
- **排查三件套**：ffprobe 看流信息 + Wireshark 抓包 + 日志看状态码

---

## 下一篇

下一篇将深入讲解 **SRT 协议**：如何在丢包率高达 30% 的弱网下实现稳定推流，SRT 的 ARQ 重传机制、延迟控制窗口，以及与 RTMP 的性能对比。

---

## 参考资料

1. [RTMP Specification 1.0 - Adobe](https://rtmp.veriskope.com/docs/spec/)
2. [nginx-rtmp-module - GitHub](https://github.com/arut/nginx-rtmp-module)
3. [FFmpeg RTMP 文档](https://ffmpeg.org/ffmpeg-protocols.html#rtmp)
4. [Enhanced RTMP - GitHub](https://github.com/veovera/enhanced-rtmp)
5. [OBS Studio - RTMP 推流设置](https://obsproject.com/wiki/Stream-Settings)

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| RTMP | Real-Time Messaging Protocol | 实时消息传输协议，Adobe 开发，主流推流协议 |
| Chunk | Chunk | RTMP 数据块，最小传输单位，默认128字节 |
| AMF | Action Message Format | RTMP 元数据编码格式（AMF0/AMF3）|
| Stream Key | Stream Key | 推流密钥，形如 `rtmp://server/live/<key>` |
| Enhanced RTMP | Enhanced RTMP | RTMP 扩展版本，支持 AV1/HEVC/VP9 编码 |
| RTMPS | RTMP Secure | RTMP over TLS，用于加密推流 |
| 握手 | Handshake | RTMP 连接建立时的三次握手过程 |
| Chunk Stream ID | Chunk Stream ID | 标识消息归属的逻辑通道 |
| Message Type | Message Type | RTMP 消息类型（音频/视频/元数据/控制）|
| 推流 | Push/Publish | 发布端主动向服务器推送媒体流 |
