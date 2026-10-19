# DASH 与 CMAF 统一分片格式

> **适合读者**：后端开发者、流媒体工程师  
> **前置知识**：[第 10 篇：HLS 协议详解](10-HLS协议详解.md)

---

## 一、MPEG-DASH 是什么

**DASH（Dynamic Adaptive Streaming over HTTP）** 是 MPEG 组织 2011 年制定的开放标准（ISO/IEC 23009）。与 HLS 一样，DASH 也是基于 HTTP 的自适应码率流媒体协议。

### 1.1 DASH vs HLS 对比

| 特性 | HLS | DASH |
|------|-----|------|
| 制定者 | 苹果（私有，后开放） | MPEG（国际标准） |
| 分片格式 | TS（v3）/ fMP4（v7）| MP4 / TS |
| 索引文件 | M3U8（文本） | MPD（XML） |
| 浏览器原生支持 | Safari | Chrome/Firefox（需 MSE） |
| iOS Safari | 原生支持 | 需 dash.js |
| DRM 支持 | FairPlay | Widevine/PlayReady/FairPlay |
| 直播延迟 | 6-30s（LL-HLS 可达 2s） | 类似 HLS |

**为什么 DASH 没有成为唯一标准？**
- 苹果设备只原生支持 HLS，主流还是用 HLS
- DASH 在非苹果平台（Chrome、Android）表现更好
- Netflix、YouTube 主要用 DASH

### 1.2 DASH 核心概念

```
MPD（Media Presentation Description）
  └── Period（时间段，直播通常 1 个 Period）
        └── AdaptationSet（同类型流的集合，如"视频"或"音频"）
              ├── Representation（具体码率，如 720p@2Mbps）
              │     ├── SegmentTemplate（分片命名规则）
              │     └── SegmentTimeline（每段的时间戳）
              └── Representation（另一码率）
```

---

## 二、MPD 文件解析

### 2.1 直播 MPD 示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<MPD
  xmlns="urn:mpeg:dash:schema:mpd:2011"
  type="dynamic"
  minimumUpdatePeriod="PT2S"
  availabilityStartTime="2026-07-29T00:00:00Z"
  timeShiftBufferDepth="PT30S"
  minBufferTime="PT2S">

  <Period id="1" start="PT0S">

    <!-- 视频自适应集 -->
    <AdaptationSet mimeType="video/mp4" codecs="avc1.64001f"
                   frameRate="30" segmentAlignment="true">
      <SegmentTemplate
        timescale="90000"
        media="video_$RepresentationID$_$Number$.m4s"
        initialization="video_$RepresentationID$_init.mp4"
        startNumber="1">
        <SegmentTimeline>
          <S t="0" d="180000" r="99"/>  <!-- 2 秒分片（180000/90000=2s），重复 99 次 -->
        </SegmentTimeline>
      </SegmentTemplate>

      <Representation id="360p" width="640" height="360" bandwidth="800000"/>
      <Representation id="720p" width="1280" height="720" bandwidth="2800000"/>
      <Representation id="1080p" width="1920" height="1080" bandwidth="5000000"/>
    </AdaptationSet>

    <!-- 音频自适应集 -->
    <AdaptationSet mimeType="audio/mp4" codecs="mp4a.40.2" lang="zh">
      <SegmentTemplate
        timescale="44100"
        media="audio_$Number$.m4s"
        initialization="audio_init.mp4"
        startNumber="1">
        <SegmentTimeline>
          <S t="0" d="88200" r="99"/>  <!-- 2 秒音频分片 -->
        </SegmentTimeline>
      </SegmentTemplate>

      <Representation id="audio" bandwidth="128000" audioSamplingRate="44100"/>
    </AdaptationSet>

  </Period>
</MPD>
```

**关键属性**：
- `type="dynamic"`：直播（static 为点播）
- `minimumUpdatePeriod="PT2S"`：客户端每 2 秒刷新 MPD
- `timeShiftBufferDepth="PT30S"`：时移缓冲 30 秒（DVR 功能）
- `timescale`：时间刻度（分片时长 = duration / timescale）

### 2.2 分片命名规则

```
$RepresentationID$  → 替换为 "360p"、"720p" 等
$Number$            → 替换为分片序号 1, 2, 3...
$Time$              → 替换为时间戳

示例：
  初始化段：video_720p_init.mp4
  媒体段：  video_720p_1.m4s, video_720p_2.m4s, ...
```

---

## 三、CMAF：统一 HLS 和 DASH

### 3.1 历史困境

在 CMAF 出现之前，CDN 需要存储两套内容：

```
同一个视频：
  HLS 版本：  video_720p_seg001.ts, seg002.ts...
  DASH 版本： video_720p_1.m4s, 2.m4s...

存储成本翻倍，转码成本翻倍
```

### 3.2 CMAF 的解决方案

**CMAF（Common Media Application Format，MPEG-A 第 19 部分）** 定义了一种 **统一的分片格式**，既能用于 HLS，也能用于 DASH：

```
统一分片（CMAF Chunk / Segment）：
  使用 fMP4（Fragmented MP4）格式
  同一个 .m4s 文件，既可以被 HLS M3U8 引用，也可以被 DASH MPD 引用

HLS M3U8 引用 CMAF：
  #EXTM3U
  #EXT-X-VERSION:7          ← 版本 7+ 支持 fMP4
  #EXTINF:4.000000,
  video_720p_1.m4s          ← 直接用 .m4s 分片

DASH MPD 引用同样的文件：
  media="video_720p_$Number$.m4s"
```

### 3.3 fMP4 vs TS 分片格式

**TS（MPEG-TS）**：
```
[TS 包 188B][TS 包 188B]...[TS 包 188B]
```
- 固定 188 字节包大小
- 有较多开销（每包 4 字节头）
- 可以从任意位置开始解码（流式友好）

**fMP4（Fragmented MP4）**：
```
[ftyp box][moov box（初始化段）]
[moof box（fragment header）][mdat box（媒体数据）]
[moof box][mdat box]
...
```
- 初始化段（init segment）包含编解码器信息
- 每个 fragment 是独立的解码单元
- 更高效的压缩（约少 10% 开销）
- 支持字节范围请求（Byte-Range）

---

## 四、fMP4 内部结构

### 4.1 Box 层级

```
ftyp  (File Type)          - 文件类型声明
moov  (Movie)              - 初始化段（只在 init 分片中）
  ├── mvhd (Movie Header)  - 总体信息
  ├── trak (Track)         - 视频轨道
  │     └── mdia, minf, stbl...
  └── mvex (Movie Extends) - 标记这是 fMP4
        └── trex (Track Extends Default)

--- 以下在每个媒体分片中重复 ---
moof  (Movie Fragment)
  ├── mfhd (Movie Fragment Header)  - 序列号
  └── traf (Track Fragment)
        ├── tfhd (Track Fragment Header)
        └── trun (Track Run)        - 每帧的时长、大小、偏移

mdat  (Media Data)         - 实际视频/音频数据
```

### 4.2 用 FFmpeg 生成 fMP4 分片

```bash
# 生成 CMAF 格式的 HLS（fMP4 分片）
ffmpeg -i input.mp4 \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -hls_time 4 \
  -hls_segment_type fmp4 \          # 使用 fMP4 分片
  -hls_fmp4_init_filename "init.mp4" \
  -hls_segment_filename "seg%03d.m4s" \
  output.m3u8

# 生成的文件：
# init.mp4（初始化段）
# seg001.m4s, seg002.m4s...（媒体分片）
```

```bash
# 生成 DASH 输出
ffmpeg -i input.mp4 \
  -map 0:v -map 0:a \
  -c:v libx264 -b:v 2800k \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 4 \
  -init_seg_name "init_\$RepresentationID\$.mp4" \
  -media_seg_name "seg_\$RepresentationID\$_\$Number\$.m4s" \
  output.mpd
```

---

## 五、LL-DASH（低延迟 DASH）

DASH 同样有低延迟版本，基于 CMAF Chunks：

### 5.1 CMAF Chunk 机制

```
普通分片（4秒）：
[========4s media data========] → 4s 后客户端才能下载

CMAF Chunk（0.2秒一个 chunk，但属于同一个分片）：
[0.2s][0.2s][0.2s]...[0.2s]  → 共 20 个 chunk 组成 4s 分片
```

每个 Chunk 是一个完整的 moof+mdat，可以独立解码。

### 5.2 HTTP Chunked Transfer Encoding

服务器在分片完成之前就开始传输：

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a  ← chunk 大小（十六进制）
[moof+mdat for first 0.2s]
1a
[moof+mdat for next 0.2s]
...
0   ← 结束标记
```

客户端边下载边播放，不需要等整个分片下载完成。

---

## 六、DASH 与 DRM

DASH 原生支持 Common Encryption（CENC），可以同时用于 Widevine（Google）、PlayReady（Microsoft）和 FairPlay（Apple）：

### 6.1 Multi-DRM 架构

```xml
<ContentProtection
  schemeIdUri="urn:uuid:EDEF8BA9-79D6-4ACE-A3C8-27DCD51D21ED"
  value="Widevine">
  <cenc:pssh>AAAA...</cenc:pssh>
</ContentProtection>

<ContentProtection
  schemeIdUri="urn:uuid:9A04F079-9840-4286-AB92-E65BE0885F95"
  value="PlayReady">
  <mspr:pro>...</mspr:pro>
</ContentProtection>
```

### 6.2 加密流程

```
视频内容
  ↓ 用 CEK（Content Encryption Key）加密
加密分片
  ↓
CDN 分发

播放器 → License Server 请求密钥
  ↓      （Widevine/PlayReady/FairPlay 各自协议）
解密 CEK
  ↓
解密播放
```

---

## 七、选型指南

### 7.1 HLS vs DASH vs CMAF

| 场景 | 推荐格式 | 理由 |
|------|---------|------|
| iOS + 国内直播 | HLS (TS) | iOS 原生，CDN 成熟 |
| 全平台 + 版权保护 | CMAF | 一套内容，兼容 HLS/DASH + Multi-DRM |
| Netflix/YouTube 级别 | DASH + CMAF | 最佳自适应码率，多 DRM |
| 低延迟直播 | LL-HLS 或 LL-DASH | Part/Chunk 机制 |
| 企业内网 | HLS 或 RTMP | 简单够用 |

### 7.2 主流平台使用情况

| 平台 | 协议 |
|------|------|
| Netflix | DASH + CMAF（Widevine/PlayReady） |
| YouTube | DASH（自研格式） |
| Twitch | HLS |
| 国内视频平台 | HLS + 私有加密 |
| Disney+ | CMAF（Multi-DRM） |

---

## 八、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **DASH** | MPEG 开放标准，MPD（XML）描述，比 HLS 更灵活 |
| **CMAF** | 统一 HLS/DASH 分片格式，一套内容双协议复用 |
| **fMP4** | 比 TS 效率更高，支持字节范围请求 |
| **LL-DASH** | CMAF Chunk + HTTP Chunked Transfer，延迟 2-4s |
| **Multi-DRM** | CENC 加密，DASH 原生多 DRM 支持 |

---

## 下一篇

下一篇将深入讲解 **WebRTC 协议栈**：ICE/STUN/TURN 的穿透机制，SDP 协商过程，以及为什么 WebRTC 能实现 100ms 超低延迟。

---

## 参考资料

1. [MPEG-DASH 标准 - ISO/IEC 23009](https://www.iso.org/standard/83314.html)
2. [CMAF 标准 - ISO/IEC 23000-19](https://www.iso.org/standard/82529.html)
3. [dash.js - GitHub](https://github.com/Dash-Industry-Forum/dash.js)
4. [FFmpeg DASH Muxer](https://ffmpeg.org/ffmpeg-formats.html#dash-1)
5. [CMAF 与 HLS/DASH 统一 - Apple](https://developer.apple.com/documentation/http-live-streaming/about-the-common-media-application-format-with-http-live-streaming)

---

## 动手实践

### 练习1：FFmpeg 生成 DASH 流
```bash
ffmpeg -i input.mp4 \
  -map 0:v -map 0:a \
  -c:v libx264 -b:v:0 1500k -s:v:0 1280x720 \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 4 \
  -init_seg_name "init_$RepresentationID$.mp4" \
  -media_seg_name "seg_$RepresentationID$_$Number$.m4s" \
  manifest.mpd
```
**预期输出**：生成 `manifest.mpd` 和系列 `.mp4` init + `.m4s` 分片，dash.js 可播放。

### 练习2：生成 CMAF 兼容输出（同时服务 HLS 和 DASH）
```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -preset fast -g 60 \
  -c:a aac \
  -f dash \
  -seg_duration 4 \
  -hls_playlist 1 \
  -hls_master_name master.m3u8 \
  output/manifest.mpd
```
**预期输出**：同一目录下同时生成 `manifest.mpd`（DASH）和 `master.m3u8`（HLS），一套内容同时支持两种播放器。

### 练习3：验证 MPD 结构
```bash
# 用 curl 查看生成的 MPD 文件
curl http://localhost/output/manifest.mpd | python3 -m json.tool 2>/dev/null   || curl http://localhost/output/manifest.mpd | xmllint --format -
```
**预期输出**：看到 `<AdaptationSet>`、`<Representation>` 结构，包含视频/音频两组适配集。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| MPD 找不到初始化分片 | init 文件路径与 MPD 中声明不一致 | 检查 `initialization="..."` 属性的相对路径 |
| 播放器 codec 不支持 | fMP4 中封装了 AV1/HEVC，旧播放器不识别 | 添加 H.264 兼容层，或检查 Representation@codecs |
| ABR 不切换 | 带宽探测失败，播放器始终选最低质量 | 检查各 Representation 的 `@bandwidth` 值是否合理 |
| 时钟漂移导致音视频不同步 | 分片时间戳计算错误 | 确保 `-g` 与 `-seg_duration` 整除关系 |
| CORS 错误 | CDN 未透传 CORS 头 | 配置源站和 CDN 均返回 `Access-Control-Allow-Origin` |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| MPD | Media Presentation Description | DASH 清单文件，描述所有可用媒体流和分片位置 |
| AdaptationSet | Adaptation Set | MPD 中一组同类媒体（如所有视频码率为一组）|
| Representation | Representation | 单一码率/分辨率的媒体流 |
| CMAF | Common Media Application Format | 统一 HLS 和 DASH 的分片格式（ISO 23000-19）|
| fMP4 | Fragmented MP4 | 支持流式传输的 MP4，每段独立可解码 |
| InitSegment | Initialization Segment | 包含编解码器参数的初始化分片，播放前必须加载 |
| SegmentTemplate | Segment Template | MPD 中用模板生成分片 URL 的方式 |
| ABR | Adaptive Bitrate | 自适应码率，播放器依据带宽自动选择分辨率 |

---

> **全文完** | 上一篇：[第 10 篇 — HLS 协议详解](10-HLS协议详解.md) | 下一篇：[第 12 篇 — WebRTC 协议栈详解](12-WebRTC协议栈详解.md)
