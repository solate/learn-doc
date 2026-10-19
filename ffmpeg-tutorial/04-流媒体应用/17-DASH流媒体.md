# 第 17 篇：DASH 流媒体

> **模块**：04-流媒体应用 | **难度**：中级 | **预计阅读**：55 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-07-28

---

## 目录

1. [DASH 协议原理](#1-dash-协议原理)
2. [MPD 文件结构解析](#2-mpd-文件结构解析)
3. [基础 DASH 打包](#3-基础-dash-打包)
4. [自适应码率 DASH（ABR）](#4-自适应码率-dashabr)
5. [DASH 加密：CENC](#5-dash-加密cenc)
6. [低延迟 DASH（LL-DASH）](#6-低延迟-dashll-dash)
7. [HLS vs DASH 对比与选型](#7-hls-vs-dash-对比与选型)
8. [Go 集成：DASH 打包服务](#8-go-集成dash-打包服务)
9. [动手实践](#9-动手实践)
10. [本章总结](#10-本章总结)
11. [术语表](#11-术语表)

---

## 学习目标

完成本章后，你将能够：

- 理解 MPD（Media Presentation Description）文件的 Period/AdaptationSet/Representation 层级
- 用 FFmpeg `-f dash` 生成 DASH 点播和直播流
- 理解 CENC 加密标准与 DRM 集成方式
- 合理选择 HLS 或 DASH 作为流媒体格式
- 在 Go 中封装 DASH 打包服务

---

## 1. DASH 协议原理

### 1.1 DASH 架构

**MPEG-DASH（Dynamic Adaptive Streaming over HTTP）** 是 MPEG 于 2012 年发布的国际标准（ISO/IEC 23009）：

```
┌─────────────────────────────────────────────────────────┐
│                   DASH 分发架构                          │
│                                                          │
│  源媒体 → FFmpeg → manifest.mpd (MPD 清单)              │
│                  └─→ init-stream*.mp4 (初始化段)         │
│                  └─→ chunk-stream*-*.mp4 (媒体段)        │
│                              │                           │
│                        HTTP 服务器 / CDN                 │
│                              │                           │
│                         DASH 播放器                      │
│                    (dash.js / Shaka Player)              │
└─────────────────────────────────────────────────────────┘
```

### 1.2 DASH vs HLS 核心差异

| 特性 | HLS | DASH |
|------|-----|------|
| 标准化 | 私有（Apple）| 国际标准 ISO/IEC 23009 |
| 分片格式 | TS 或 fMP4 | 仅 fMP4（ISO BMFF）|
| 清单格式 | m3u8（文本）| MPD（XML）|
| iOS 原生支持 | ✅ AVPlayer | ❌ 需第三方播放器 |
| 加密标准 | AES-128 | CENC（Widevine/PlayReady）|
| 低延迟 | LL-HLS | LL-DASH（CTE）|

### 1.3 DASH 分片结构

```
DASH 输出：
├── manifest.mpd                     ← MPD 清单（描述所有码率）
├── init-stream0.m4s                 ← 视频初始化段（包含 moov）
├── init-stream1.m4s                 ← 音频初始化段
├── chunk-stream0-00001.m4s          ← 视频媒体段 #1
├── chunk-stream0-00002.m4s          ← 视频媒体段 #2
├── chunk-stream1-00001.m4s          ← 音频媒体段 #1
└── chunk-stream1-00002.m4s          ← 音频媒体段 #2
```

**初始化段（init segment）**：包含 `ftyp` + `moov` box，播放器必须先下载  
**媒体段（media segment）**：包含 `moof` + `mdat` box，携带实际音视频数据

---

## 2. MPD 文件结构解析

### 2.1 MPD 层级关系

```xml
<MPD>                                  ← 根节点
  <Period>                             ← 时间段（广告插入点）
    <AdaptationSet>                    ← 自适应集（视频/音频/字幕）
      <Representation>                 ← 具体码率（1080p/720p/480p）
        <SegmentTemplate>              ← 分片模板（命名规则）
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

### 2.2 完整 MPD 示例

```xml
<?xml version="1.0" encoding="utf-8"?>
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" type="static" mediaPresentationDuration="PT634.566S" minBufferTime="PT2S" profiles="urn:mpeg:dash:profile:isoff-on-demand:2011">
  <Period id="0">
    <!-- 视频 AdaptationSet -->
    <AdaptationSet id="0" contentType="video" segmentAlignment="true" bitstreamSwitching="true">
      <Representation id="0" mimeType="video/mp4" codecs="avc1.64001f" width="1920" height="1080" bandwidth="5000000">
        <SegmentTemplate timescale="90000" initialization="init-stream$RepresentationID$.m4s" media="chunk-stream$RepresentationID$-$Number%05d$.m4s" startNumber="1" duration="540000"/>
      </Representation>
      <Representation id="1" mimeType="video/mp4" codecs="avc1.64001f" width="1280" height="720" bandwidth="3000000">
        <SegmentTemplate timescale="90000" initialization="init-stream$RepresentationID$.m4s" media="chunk-stream$RepresentationID$-$Number%05d$.m4s" startNumber="1" duration="540000"/>
      </Representation>
    </AdaptationSet>
    
    <!-- 音频 AdaptationSet -->
    <AdaptationSet id="1" contentType="audio" segmentAlignment="true">
      <Representation id="2" mimeType="audio/mp4" codecs="mp4a.40.2" audioSamplingRate="48000" bandwidth="128000">
        <SegmentTemplate timescale="48000" initialization="init-stream$RepresentationID$.m4s" media="chunk-stream$RepresentationID$-$Number%05d$.m4s" startNumber="1" duration="288000"/>
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

## 3. 基础 DASH 打包

### 3.1 本地视频转 DASH（VOD）

```bash
# 最简单的 DASH VOD 打包
ffmpeg -i input.mp4 \
  -c:v libx264 -crf 23 \
  -c:a aac -b:a 128k \
  -f dash \
  -dash_segment_type mp4 \
  -seg_duration 6 \
  output/manifest.mpd
```

### 3.2 视频音频分轨

**为什么要分轨？**

DASH 的核心是**自适应码率切换**（ABR）：
- 网络从 5Mbps 降到 1Mbps 时，播放器只需切换视频轨（1080p → 360p），音频轨保持不变
- **如果音视频混在一起**：每次切码率都要重新下载完整分片（包括音频），浪费带宽
- **分轨后**：视频轨有 360p/720p/1080p 多个版本，音频轨只有一份 128kbps，播放器按需组合

同时支持多语言音轨（中文/英文/日文）、多字幕选择，播放器动态组合，不需要为每种组合预先生成完整文件。

**对比 HLS 的设计差异**：
- **HLS**：默认音视频混流（`.ts` 分片包含音视频），后来才加入 fMP4 + 分轨支持
- **DASH**：从设计之初就是**分轨优先**，音视频用独立的 AdaptationSet

```bash
# DASH 标准做法：视频音频分轨（最佳实践）
ffmpeg -i input.mp4 \
  -c:v libx264 -b:v 3M \
  -an -f dash \
  -init_seg_name "init_v.mp4" \
  -media_seg_name "chunk_v_$Number%05d$.m4s" \
  -seg_duration 6 \
  output/manifest.mpd
```

**注意**：上述命令只生成视频轨（`-an` 移除音频），实际应用需单独生成音频轨并在 MPD 中关联（见 4.1 多码率示例）。

### 3.3 直播 DASH

```bash
# 直播 DASH（动态 MPD）
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset veryfast -tune zerolatency -b:v 2M \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 4 \
  -window_size 5 \
  -extra_window_size 3 \
  -remove_at_exit 1 \
  /var/dash/live/manifest.mpd
```

**关键参数**：
- `window_size 5`：MPD 保留最近 5 个段（滑动窗口）
- `extra_window_size 3`：额外保留 3 个段（播放缓冲）
- `remove_at_exit 1`：退出时删除分片文件

## 4. 自适应码率 DASH（ABR）

### 4.1 多码率 DASH 打包

```bash
# 一次输入，生成多码率 DASH
ffmpeg -i input.mp4 \
  -map 0:v -map 0:v -map 0:a \
  -c:v:0 libx264 -b:v:0 5M -s:v:0 1920x1080 \
  -c:v:1 libx264 -b:v:1 3M -s:v:1 1280x720 \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -adaptation_sets "id=0,streams=v id=1,streams=a" \
  output/manifest.mpd
```

**参数说明**：
- `-map 0:v -map 0:v`：复制视频流两次（多码率源）
- `-c:v:0` / `-c:v:1`：分别设置两路视频参数
- `-adaptation_sets`：定义 AdaptationSet 分组（视频 id=0，音频 id=1）

### 4.2 使用 filter_complex 生成多码率

```bash
ffmpeg -i input.mp4 \
  -filter_complex "\
    [0:v]split=3[v1][v2][v3];\
    [v1]scale=1920:1080[1080p];\
    [v2]scale=1280:720[720p];\
    [v3]scale=854:480[480p]" \
  -map "[1080p]" -c:v:0 libx264 -b:v:0 5M \
  -map "[720p]" -c:v:1 libx264 -b:v:1 3M \
  -map "[480p]" -c:v:2 libx264 -b:v:2 1.5M \
  -map 0:a -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -adaptation_sets "id=0,streams=0,1,2 id=1,streams=3" \
  output/manifest.mpd
```

## 5. DASH 加密：CENC

### 5.1 CENC 原理

**CENC（Common Encryption）** 是 DASH 标准加密规范（ISO 23001-7），支持多 DRM 系统共用同一加密内容：

```
┌──────────────────────────────────────────────────────┐
│                   CENC 加密架构                        │
│                                                        │
│  明文 fMP4 ──▶ CENC 加密 ──▶ 密文 fMP4               │
│                    │                                   │
│              Key ID + Key                              │
│                    │                                   │
│         ┌──────────┼──────────┐                       │
│         ▼          ▼          ▼                       │
│     Widevine    PlayReady    FairPlay                  │
│    (Android)   (Windows)    (iOS/Safari)               │
└──────────────────────────────────────────────────────┘
```

| DRM | 平台 | 协议 |
|-----|------|------|
| Widevine | Android/Chrome | CENC/CBCS |
| PlayReady | Windows/Xbox | CENC |
| FairPlay | iOS/Safari | CBCS |

### 5.2 FFmpeg CENC 加密打包

```bash
# 生成加密密钥（Key ID + Key）
KEY_ID=$(openssl rand -hex 16)    # 16字节，32位hex
KEY=$(openssl rand -hex 16)       # 16字节，32位hex

# CENC 加密 DASH 打包
ffmpeg -i input.mp4 \
  -c:v libx264 -b:v 3M \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -encryption_scheme cenc-aes-ctr \
  -encryption_key "${KEY}" \
  -encryption_kid "${KEY_ID}" \
  output/manifest.mpd
```

**参数说明**：
- `-encryption_scheme cenc-aes-ctr`：AES-CTR 模式（CENC 标准）
- `-encryption_key`：AES 加密密钥（32位 hex）
- `-encryption_kid`：Key ID，DRM 系统用于匹配密钥

### 5.3 加密后的 MPD 片段

```xml
<ContentProtection schemeIdUri="urn:mpeg:dash:mp4protection:2011" value="cenc">
  <cenc:default_KID>a1b2c3d4-e5f6-7890-abcd-ef1234567890</cenc:default_KID>
</ContentProtection>
<ContentProtection schemeIdUri="urn:uuid:EDEF8BA9-79D6-4ACE-A3C8-27DCD51D21ED">
  <!-- Widevine PSSH box（由 DRM 服务商生成）-->
</ContentProtection>
```

### 5.4 完整 DRM 工作流

生产环境中，FFmpeg 只负责加密，密钥管理和许可证下发由 DRM 服务完成：

```
┌─────────┐   1.申请密钥    ┌──────────────┐
│ 转码服务 │ ─────────────▶ │  DRM 密钥服务  │
│ (FFmpeg)│ ◀───────────── │ (Widevine等)  │
└─────────┘   2.Key+KID     └──────────────┘
     │                              ▲
     │ 3.加密打包                    │ 5.验证+下发License
     ▼                              │
┌─────────┐   4.请求License   ┌──────────┐
│   CDN   │ ◀─────────────── │  播放器   │
└─────────┘   6.解密播放      └──────────┘
```

> **注意**：FFmpeg 的 CENC 支持仅覆盖加密本身，PSSH box 注入、License 服务需借助 Shaka Packager、Bento4 等专业工具。

---

## 6. 低延迟 DASH（LL-DASH）

### 6.1 CMAF Chunked 传输

LL-DASH 通过 **CMAF Chunk**（分块编码传输）把延迟从 10-30 秒降到 2-5 秒：

```
传统 DASH：等整个 segment 编码完才能下发
  ├──── 6s segment ────┤──── 6s segment ────┤
  编码完成才可用 ▲       编码完成才可用 ▲
  延迟 ≈ 2×6 = 12s+

LL-DASH：segment 内部切成 chunk，边编码边下发
  ├─c─┬─c─┬─c─┬─c─┬─c─┬─c─┤
   ▲ 每个 chunk（0.5s）立即可用
  延迟 ≈ 2s
```

### 6.2 FFmpeg LL-DASH 配置

```bash
ffmpeg -re -i rtmp://localhost/live/stream \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -g 30 -keyint_min 30 -sc_threshold 0 \
  -b:v 3M -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 2 \
  -frag_duration 0.5 \
  -frag_type duration \
  -ldash 1 \
  -streaming 1 \
  -use_template 1 -use_timeline 0 \
  -window_size 5 -remove_at_exit 1 \
  -utc_timing_url "https://time.akamai.com/?iso" \
  output/manifest.mpd
```

**关键参数**：

| 参数 | 作用 |
|------|------|
| `-ldash 1` | 启用 LL-DASH 模式（写入 availabilityTimeOffset） |
| `-streaming 1` | 启用 chunked 输出，边编码边写 |
| `-frag_duration 0.5` | 每个 chunk 0.5 秒 |
| `-frag_type duration` | 按时长切 chunk |
| `-use_timeline 0` | LL-DASH 必须用 SegmentTemplate+Number |
| `-utc_timing_url` | 时钟同步源，客户端据此计算播放位置 |

> **服务端要求**：必须支持 HTTP/1.1 Chunked Transfer-Encoding 或 HTTP/2，否则 chunk 无法提前下发。

### 6.3 LL-DASH 的 MPD 特征

```xml
<MPD type="dynamic"
     minimumUpdatePeriod="PT2S"
     availabilityStartTime="2026-07-28T10:00:00Z"
     timeShiftBufferDepth="PT10S"
     minBufferTime="PT1S">
  <ServiceDescription id="0">
    <Latency target="2000" min="1000" max="4000"/>
    <PlaybackRate min="0.96" max="1.04"/>
  </ServiceDescription>
  <UTCTiming schemeIdUri="urn:mpeg:dash:utc:http-iso:2014"
             value="https://time.akamai.com/?iso"/>
  <Period id="0" start="PT0S">
    <AdaptationSet contentType="video" segmentAlignment="true">
      <Representation bandwidth="3000000" codecs="avc1.64001f">
        <SegmentTemplate timescale="90000"
                         duration="180000"
                         availabilityTimeOffset="1.500"
                         availabilityTimeComplete="false"
                         media="chunk-$RepresentationID$-$Number%05d$.m4s"
                         initialization="init-$RepresentationID$.m4s"/>
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

**关键字段解读**：
- `availabilityTimeComplete="false"`：告诉客户端 segment 未编码完也可请求
- `availabilityTimeOffset="1.500"`：可提前 1.5 秒开始请求
- `<Latency target="2000">`：目标延迟 2000ms，播放器据此调整缓冲
- `<PlaybackRate min/max>`：允许 ±4% 变速追赶延迟

---

## 7. HLS 与 DASH 的选择

### 7.1 特性对比

| 维度 | HLS | DASH |
|------|-----|------|
| 标准化 | Apple 私有 → RFC 8216 | ISO/IEC 23009-1 国际标准 |
| 清单格式 | m3u8（文本行） | MPD（XML） |
| 分片容器 | TS / fMP4 | fMP4（CMAF） |
| iOS 原生 | ✅ 支持 | ❌ 需 JS 播放器 |
| Android 原生 | 部分支持 | ✅ ExoPlayer 原生 |
| 加密 | AES-128 / SAMPLE-AES | CENC（多 DRM） |
| 低延迟方案 | LL-HLS（Part） | LL-DASH（CMAF Chunk） |
| 编解码器灵活度 | 受 Apple 限制 | 任意编解码器 |
| CDN 兼容 | 极好（历史悠久） | 好 |

### 7.2 决策树

```
需要 iOS/Safari 原生播放？
  ├─ 是 ──▶ 必须提供 HLS
  │         └─ 同时要 DRM？──▶ HLS + FairPlay
  └─ 否 ──▶ 只做 Web/Android？
            ├─ 是 ──▶ DASH（编解码器更灵活）
            └─ 否 ──▶ 全平台覆盖
                      └─▶ CMAF 双清单方案（推荐）
```

### 7.3 CMAF 双清单方案（推荐）

用同一套 fMP4 分片，同时生成 m3u8 和 MPD，存储成本只增加清单文件：

```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -b:v 3M -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -hls_playlist 1 \
  -init_seg_name 'init-$RepresentationID$.m4s' \
  -media_seg_name 'chunk-$RepresentationID$-$Number%05d$.m4s' \
  output/manifest.mpd
```

`-hls_playlist 1` 会额外生成 `master.m3u8` + 各码率 m3u8，复用同一批 `.m4s` 分片。

### 7.4 技术选型建议

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 国内移动端为主 | HLS | iOS 市占率高，微信/浏览器原生支持 |
| 海外 Android 为主 | DASH | ExoPlayer 生态成熟，编解码器灵活 |
| 全球化平台 | CMAF 双清单 | 一套分片覆盖所有平台 |
| 需多 DRM 支持 | DASH + CENC | Widevine + PlayReady + FairPlay |
| 低延迟直播（< 3s） | LL-HLS / LL-DASH | 需服务端支持 chunked 传输 |
| 快速上线 MVP | HLS | 生态成熟，CDN 兼容性最好 |

---

## 8. Go 集成：DASH 打包服务

### 8.1 DASH 打包器

```go
package dash

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
)

// DASHProfile DASH 转码规格
type DASHProfile struct {
	Name         string
	Width        int
	Height       int
	VideoBitrate string
	AudioBitrate string
}

// DASHPackager DASH 打包服务
type DASHPackager struct {
	outputDir string
	baseURL   string
}

// NewDASHPackager 创建 DASH 打包器
func NewDASHPackager(outputDir, baseURL string) *DASHPackager {
	return &DASHPackager{outputDir: outputDir, baseURL: baseURL}
}

// PackageVOD 本地文件 → DASH VOD（多码率）
func (p *DASHPackager) PackageVOD(ctx context.Context, inputPath string, profiles []DASHProfile) error {
	if err := os.MkdirAll(p.outputDir, 0755); err != nil {
		return fmt.Errorf("创建输出目录失败: %w", err)
	}
	
	args := []string{"-i", inputPath}
	
	// filter_complex：分流
	if len(profiles) > 1 {
		splitFilter := fmt.Sprintf("[0:v]split=%d", len(profiles))
		labels := make([]string, len(profiles))
		for i := range profiles {
			labels[i] = fmt.Sprintf("[vsplit%d]", i)
		}
		splitFilter += strings.Join(labels, "")
		
		// 为每路生成 scale 滤镜
		scaleParts := make([]string, len(profiles))
		scaleLabels := make([]string, len(profiles))
		for i, prof := range profiles {
			scaleLabels[i] = fmt.Sprintf("[v%d]", i)
			scaleParts[i] = fmt.Sprintf("[vsplit%d]scale=%d:%d%s", i, prof.Width, prof.Height, scaleLabels[i])
		}
		
		filterComplex := splitFilter + ";" + strings.Join(scaleParts, ";")
		args = append(args, "-filter_complex", filterComplex)
		
		// 映射每路视频
		for i, prof := range profiles {
			args = append(args,
				"-map", scaleLabels[i],
				"-c:v:"+fmt.Sprintf("%d", i), "libx264", "-b:v:"+fmt.Sprintf("%d", i), prof.VideoBitrate,
			)
		}
		
		// 音频
		args = append(args, "-map", "0:a", "-c:a", "aac", "-b:a", profiles[0].AudioBitrate)
	} else {
		// 单码率
		prof := profiles[0]
		args = append(args,
			"-c:v", "libx264", "-b:v", prof.VideoBitrate, "-s", fmt.Sprintf("%dx%d", prof.Width, prof.Height),
			"-c:a", "aac", "-b:a", prof.AudioBitrate,
		)
	}
	
	// DASH 输出参数
	mpdPath := filepath.Join(p.outputDir, "manifest.mpd")
	args = append(args,
		"-f", "dash",
		"-seg_duration", "6",
		"-init_seg_name", "init-stream$RepresentationID$.m4s",
		"-media_seg_name", "chunk-stream$RepresentationID$-$Number%05d$.m4s",
	)
	
	if len(profiles) > 1 {
		// 构建 adaptation_sets
		videoStreams := make([]string, len(profiles))
		for i := range profiles {
			videoStreams[i] = fmt.Sprintf("%d", i)
		}
		adaptSets := fmt.Sprintf("id=0,streams=%s id=1,streams=%d", strings.Join(videoStreams, ","), len(profiles))
		args = append(args, "-adaptation_sets", adaptSets)
	}
	
	args = append(args, mpdPath)
	
	cmd := exec.CommandContext(ctx, "ffmpeg", args...)
	return cmd.Run()
}
```

### 8.2 直播 DASH 服务

```go
// LiveDASHConfig 直播 DASH 配置
type LiveDASHConfig struct {
	RTMPSource  string
	OutputDir   string
	SegDuration int
	WindowSize  int
	StreamToken string
}

// LiveDASHService 直播 DASH 打包服务
type LiveDASHService struct {
	config LiveDASHConfig
	cmd    *exec.Cmd
	cancel context.CancelFunc
}

// Start 启动直播 DASH 打包
func (s *LiveDASHService) Start(ctx context.Context) error {
	ctx, s.cancel = context.WithCancel(ctx)
	
	if s.config.StreamToken == "" {
		s.config.StreamToken = fmt.Sprintf("%d", time.Now().Unix())
	}
	
	if err := os.MkdirAll(s.config.OutputDir, 0755); err != nil {
		return fmt.Errorf("创建输出目录失败: %w", err)
	}
	
	mpdPath := filepath.Join(s.config.OutputDir, "live.mpd")
	
	s.cmd = exec.CommandContext(ctx, "ffmpeg",
		"-i", s.config.RTMPSource,
		"-c:v", "libx264", "-preset", "veryfast", "-tune", "zerolatency", "-b:v", "2M",
		"-c:a", "aac", "-b:a", "128k",
		"-f", "dash",
		"-seg_duration", fmt.Sprintf("%d", s.config.SegDuration),
		"-window_size", fmt.Sprintf("%d", s.config.WindowSize),
		"-extra_window_size", "3",
		"-remove_at_exit", "1",
		"-init_seg_name", "init-stream$RepresentationID$.m4s",
		"-media_seg_name", "chunk-stream$RepresentationID$-$Number%05d$.m4s",
		mpdPath,
	)
	
	return s.cmd.Start()
}

// Stop 停止直播
func (s *LiveDASHService) Stop() error {
	if s.cancel != nil {
		s.cancel()
	}
	if s.cmd != nil && s.cmd.Process != nil {
		return s.cmd.Process.Signal(os.Interrupt)
	}
	return nil
}

// GetMPDURL 获取 MPD 访问 URL
func (s *LiveDASHService) GetMPDURL(baseURL string) string {
	return fmt.Sprintf("%s/live.mpd", baseURL)
}
```

### 8.3 MPD 解析器

```go
// MPDParser MPD 文件解析器
type MPDParser struct{}

// RepresentationInfo 码率信息
type RepresentationInfo struct {
	ID        string
	Bandwidth int
	Width     int
	Height    int
	Codecs    string
	MimeType  string
}

// ParseMPD 解析 MPD 文件获取码率信息
func (p *MPDParser) ParseMPD(mpdPath string) ([]RepresentationInfo, error) {
	data, err := os.ReadFile(mpdPath)
	if err != nil {
		return nil, fmt.Errorf("读取 MPD 文件失败: %w", err)
	}
	
	var result []RepresentationInfo
	
	// 简化解析：用正则提取 Representation 属性
	repPattern := regexp.MustCompile(`<Representation[^>]+id="([^"]+)"[^>]+bandwidth="([^"]+)"[^>]+width="([^"]+)"[^>]+height="([^"]+)"`)
	matches := repPattern.FindAllStringSubmatch(string(data), -1)
	
	for _, match := range matches {
		if len(match) == 5 {
			bandwidth, _ := strconv.Atoi(match[2])
			width, _ := strconv.Atoi(match[3])
			height, _ := strconv.Atoi(match[4])
			
			result = append(result, RepresentationInfo{
				ID:        match[1],
				Bandwidth: bandwidth,
				Width:     width,
				Height:    height,
			})
		}
	}
	
	return result, nil
}
```

---

## 9. 动手实践

### 实验一：单码率 DASH VOD

```bash
# 将本地视频转为 DASH VOD
ffmpeg -i big_buck_bunny.mp4 \
  -c:v libx264 -crf 23 -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -init_seg_name "init-stream0.m4s" \
  -media_seg_name "chunk-stream0-$Number%05d$.m4s" \
  output/manifest.mpd

# 用 HTTP 服务器播放
cd output && python3 -m http.server 8080
# 浏览器访问：http://localhost:8080/manifest.mpd
# 使用 dash.js 或 Shaka Player 播放
```

### 实验二：多码率 ABR DASH（Go 调用）

```go
package main

import (
	"context"
	"log"
)

func main() {
	packager := NewDASHPackager("./output", "https://cdn.example.com/dash")
	
	profiles := []DASHProfile{
		{Name: "1080p", Width: 1920, Height: 1080, VideoBitrate: "5M", AudioBitrate: "192k"},
		{Name: "720p", Width: 1280, Height: 720, VideoBitrate: "3M", AudioBitrate: "128k"},
		{Name: "480p", Width: 854, Height: 480, VideoBitrate: "1.5M", AudioBitrate: "96k"},
	}
	
	err := packager.PackageVOD(context.Background(), "input.mp4", profiles)
	if err != nil {
		log.Fatal(err)
	}
	
	log.Println("DASH 打包完成：./output/manifest.mpd")
}
```

### 实验三：直播 DASH（Go 服务）

```go
service := &LiveDASHService{
	config: LiveDASHConfig{
		RTMPSource:  "rtmp://localhost:1935/live/stream",
		OutputDir:   "/var/dash",
		SegDuration: 4,
		WindowSize:  5,
		StreamToken: fmt.Sprintf("live_%d", time.Now().Unix()),
	},
}

if err := service.Start(context.Background()); err != nil {
	log.Fatal(err)
}

log.Println("DASH 直播已启动，访问", service.GetMPDURL("http://localhost:8080"))

// 等待中断信号
c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt)
<-c

service.Stop()
```

### 实验四：CMAF 双清单方案

```bash
# 一次打包生成 DASH + HLS 双清单（共享分片）
ffmpeg -i input.mp4 \
  -c:v libx264 -b:v 3M -s 1280x720 \
  -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -hls_playlist 1 \
  -init_seg_name 'init-$RepresentationID$.m4s' \
  -media_seg_name 'chunk-$RepresentationID$-$Number%05d$.m4s' \
  output/manifest.mpd

# 输出：
# - output/manifest.mpd（DASH 清单）
# - output/master.m3u8（HLS 主清单）
# - output/media_0.m3u8（HLS 子清单）
# - output/*.m4s（共享分片文件）

# iOS/Safari 访问：http://localhost:8080/master.m3u8
# Android/Web 访问：http://localhost:8080/manifest.mpd
```

---

## 10. 本章总结

本章全面覆盖了 DASH 在生产环境的核心场景：

| 场景 | 关键技术 | 说明 |
|------|----------|------|
| VOD 打包 | `-f dash` / `seg_duration` | 静态 MPD，完整分片列表 |
| 直播推流 | `window_size` / `extra_window_size` | 动态 MPD，滑动窗口 |
| ABR 多码率 | `filter_complex split` / `adaptation_sets` | 一次解码多路编码 |
| DRM 加密 | CENC / `encryption_scheme` | 需 DRM 服务配合 |
| 低延迟 | LL-DASH / `-ldash 1` / `-frag_duration` | 2-5 秒延迟，需 chunked 传输 |
| 双清单 | `-hls_playlist 1` / CMAF | 一套分片覆盖 HLS + DASH |
| MPD 结构 | Period/AdaptationSet/Representation | XML 层级，SegmentTemplate 定义分片规则 |

**关键原则**：
1. **DASH 只用 fMP4**，TS 格式不被 DASH 标准支持
2. **MPD 是 XML 格式**，比 m3u8 复杂但更灵活（支持多 Period 广告插入）
3. **CENC 加密仅覆盖加密本身**，PSSH box 和 License 服务需专业工具
4. **LL-DASH 必须服务端支持 chunked 传输**，否则无法提前下发 chunk
5. **iOS 不原生支持 DASH**，全平台覆盖优先选 CMAF 双清单方案
6. **AdaptationSet 分组很重要**，视频/音频/字幕必须分别放在不同 AdaptationSet

**HLS vs DASH 选型**：
- 国内移动端为主 → HLS（iOS 市占率高）
- 海外 Android 为主 → DASH（ExoPlayer 生态）
- 全球化平台 → CMAF 双清单（一套分片两套清单）
- 需多 DRM 支持 → DASH + CENC

**下一章**：[第 18 篇：RTMP/RTSP 实时流](18-RTMP-RTSP实时流.md)

深入 RTMP 推拉流、RTSP 摄像头接入、流媒体服务器（SRS/Nginx-RTMP）搭建，以及 Go 实现 RTMP 推流服务。

## 11. 术语表

| 术语 | 英文全称 | 说明 |
|------|----------|------|
| DASH | Dynamic Adaptive Streaming over HTTP | MPEG 国际标准流媒体协议 |
| MPD | Media Presentation Description | DASH 的 XML 清单文件 |
| fMP4 | fragmented MP4 | ISO BMFF 分片 MP4，DASH 使用 |
| Period | Period | MPD 时间段，广告插入点 |
| AdaptationSet | Adaptation Set | 自适应集（视频/音频/字幕分组）|
| Representation | Representation | 具体码率（1080p/720p/480p）|
| SegmentTemplate | Segment Template | 分片模板，定义命名规则 |
| CENC | Common Encryption | DASH 标准加密规范（ISO 23001-7）|
| Widevine | Widevine | Google DRM 系统（Android/Chrome）|
| PlayReady | PlayReady | 微软 DRM 系统（Windows/Xbox）|
| FairPlay | FairPlay | Apple DRM 系统（iOS/Safari）|
| LL-DASH | Low Latency DASH | 低延迟 DASH，基于 CMAF Chunk |
| CMAF | Common Media Application Format | 通用媒体应用格式（ISO 23000-19）|
| Chunk | Chunk | 分块编码传输，LL-DASH 核心 |
| init segment | Initialization Segment | 初始化段（包含 moov box）|
| media segment | Media Segment | 媒体段（包含 moof + mdat）|

---

*本文属于 [FFmpeg 完全指南](../README.md) 系列第 17 篇。*

