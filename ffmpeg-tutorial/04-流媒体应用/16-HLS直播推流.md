# 第 16 篇：HLS 直播推流

> **模块**：04-流媒体应用 | **难度**：中级 | **预计阅读**：60 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-07-28

---

## 目录

1. [HLS 协议原理](#1-hls-协议原理)
2. [基础 HLS 打包](#2-基础-hls-打包)
3. [HLS 参数详解](#3-hls-参数详解)
4. [自适应码率 HLS（ABR）](#4-自适应码率-hlsabr)
5. [HLS 加密：AES-128](#5-hls-加密aes-128)
6. [低延迟 HLS（LL-HLS）](#6-低延迟-hlsll-hls)
7. [RTMP 直播转 HLS](#7-rtmp-直播转-hls)
8. [HLS 直播流录制与存档](#8-hls-直播流录制与存档)
9. [CDN 分发优化](#9-cdn-分发优化)
10. [Go 集成：HLS 打包服务](#10-go-集成hls-打包服务)
11. [动手实践](#11-动手实践)
12. [本章总结](#12-本章总结)
13. [术语表](#13-术语表)

---

## 学习目标

完成本章后，你将能够：

- 理解 HLS m3u8 播放列表结构与 TS 分片机制
- 用 FFmpeg 打包本地视频或实时流为 HLS 输出
- 实现 AES-128 加密的 DRM 保护 HLS 流
- 理解 LL-HLS 的 Part/Preload Hint 机制，实现低延迟直播
- 在 Go 中封装 HLS 打包服务，动态管理分片滑动窗口

---

## 1. HLS 协议原理

### 1.1 HLS 架构

**HTTP Live Streaming（HLS）** 是 Apple 于 2009 年提出的自适应流媒体协议：

```
┌──────────────────────────────────────────────────────────┐
│                    HLS 分发架构                           │
│                                                           │
│  源媒体 ──▶ FFmpeg ──▶ .m3u8 (播放列表)                  │
│                    └──▶ .ts/.fmp4 (媒体分片)              │
│                              │                            │
│                              ▼                            │
│                        CDN 节点缓存                       │
│                              │                            │
│                   ┌──────────┼──────────┐                │
│                   ▼          ▼          ▼                │
│              播放器 A    播放器 B    播放器 C              │
└──────────────────────────────────────────────────────────┘
```

### 1.2 m3u8 播放列表解析

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6          ← 最大分片时长（秒）
#EXT-X-MEDIA-SEQUENCE:0          ← 当前第一个分片序号

#EXTINF:6.006000,                ← 分片实际时长
segment000.ts                    ← 分片文件名
#EXTINF:6.006000,
segment001.ts
#EXTINF:3.003000,
segment002.ts
#EXT-X-ENDLIST                   ← 仅 VOD 有，直播无此标签
```

### 1.3 从 TS 到 fMP4：HLS 的演进

**2009 年：为什么选择 TS？**

HLS 诞生时，苹果选择 MPEG-TS 作为分片容器并非偶然：

- **成熟的工业标准**：TS 是数字电视广播（DVB/ATSC）的成熟格式，已有完整生态
- **天然容错性**：每个 188 字节的 TS packet 独立同步，HTTP 传输时即使丢包也不会导致整个分片不可播
- **快速落地**：2009 年移动网络不稳定，复用电视行业的成熟格式能加速 HLS 推广

**2016 年：TS 的三大痛点**

随着移动流量成本升高和低延迟需求出现，TS 的问题逐渐暴露：

1. **头部冗余浪费带宽**
   - 每个 6 秒分片都要重复写入 PAT（节目关联表）/PMT（节目映射表）/PES 头
   - 头部占分片体积的 ~2-3%，在 4G/5G 时代这个开销不可忽视

2. **字幕和多音轨支持薄弱**
   - WebVTT 字幕需要额外协议扩展（`#EXT-X-MEDIA`）
   - 切换多音轨（如中/英配音）需要重新加载播放器

3. **低延迟困境**
   - TS 必须等完整分片生成才能发送给客户端
   - 要降低延迟 → 必须缩短分片时长（如 2 秒）→ m3u8 请求频率暴增 → CDN 压力陡升

**fMP4 的针对性设计**

2016 年 HLS v6 引入 fMP4（基于 CMAF 标准）精准击破上述痛点：

✅ **解决头部冗余**
- `init.mp4` 一次性包含所有元数据（moov box：编解码器、时间轴、轨道信息）
- 后续每个分片只存纯媒体数据（moof + mdat），体积减少约 15%

✅ **原生多轨支持**
- MP4 的 `trak`（track）机制天然支持多音轨、多字幕轨道无缝切换
- 播放器只需更换 `init.mp4` 中的轨道引用，无需重连

✅ **使能低延迟（LL-HLS）**
- fMP4 可以按 `moof`（movie fragment）为单位切片，每个 moof 可独立解码
- LL-HLS 的 Part（1 秒以内）就是一个 moof 单元，播放器能边下载边播
- 延迟从传统 HLS 的 18 秒降至 2-3 秒

**为什么不全面切换到 fMP4？**

兼容性的现实约束：
- iOS 5-9（2011-2015 年发布）只认 TS 格式
- 2026 年这些设备仍在教育、工业控制等场景服役
- 除非确定用户群体 100% 使用新设备，否则 TS 仍是保险选择

CDN 生态的惯性：
- 大量 CDN 的缓存策略和边缘节点配置是基于 TS 设计的
- 切换需要运维成本和风险评估

**实际选择建议**：
- 普通直播/点播 → **TS**（最大兼容性）
- 低延迟场景（体育赛事、游戏直播、互动直播）→ **fMP4**（必需）
- 内部系统/新开发 App → **fMP4**（省带宽、更现代）

#### 技术对比表

| 特性 | TS（MPEG-TS） | fMP4（fragmented MP4）|
|------|--------------|----------------------|
| 兼容性 | 最佳（iOS 5+） | 需 iOS 10+ / HLS v6 |
| 文件体积 | 稍大（头部冗余 ~2-3%）| 更小（减少 ~15%）|
| LL-HLS 支持 | ❌ 不支持 | ✅ 原生支持（moof 切片）|
| 字幕/多轨 | 有限（需协议扩展）| 完整支持（MP4 trak）|
| 容错性 | 强（packet 独立同步）| 一般（需完整 moof）|
| 首屏加载 | 快（任意 TS 可播）| 需先下载 init.mp4 |

### 1.4 m3u8 播放列表格式对比

虽然 TS 和 fMP4 都使用 `.m3u8` 作为播放列表，但它们的**播放列表结构有本质区别**。

#### TS 格式的 m3u8（传统 HLS）

```m3u8
#EXTM3U
#EXT-X-VERSION:3                    ← 版本号通常是 3 或 4
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0

#EXTINF:6.006000,                   ← 每个分片时长
segment000.ts                       ← 每个 .ts 是完整的自包含文件
#EXTINF:6.006000,
segment001.ts
#EXTINF:6.006000,
segment002.ts
#EXT-X-ENDLIST                      ← VOD 才有，直播无此标签
```

**特点**：
- 每个 `.ts` 文件都是**自包含**的，内部有完整的 PAT/PMT/PES 头
- 播放器可以从任意 `segmentXXX.ts` 开始播放，无需额外初始化
- 适合边下载边播（断点续播简单）

#### fMP4 格式的 m3u8（现代 HLS）

```m3u8
#EXTM3U
#EXT-X-VERSION:7                    ← 版本号至少是 6 或 7（LL-HLS 用 7+）
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-MAP:URI="init.mp4"           ← 【关键】初始化段（只出现一次）

#EXTINF:6.006000,
segment000.m4s                      ← 扩展名通常是 .m4s（media segment）
#EXTINF:6.006000,
segment001.m4s
#EXTINF:6.006000,
segment002.m4s
#EXT-X-ENDLIST
```

**特点**：
- `#EXT-X-MAP` 指向 `init.mp4`（初始化段），包含所有元数据（moov box）
- 每个 `.m4s` 文件只有纯媒体数据（moof + mdat），体积更小
- 播放器**必须先下载 `init.mp4`**，否则无法解析 `.m4s` 分片
- 适合低延迟场景（LL-HLS 的 Part 机制基于此）

#### 低延迟 HLS（LL-HLS）的 m3u8 扩展

```m3u8
#EXTM3U
#EXT-X-VERSION:9                                        ← LL-HLS 需要版本 9+
#EXT-X-TARGETDURATION:6
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.5   ← 低延迟控制
#EXT-X-PART-INF:PART-TARGET=1.0                         ← Part 目标时长 1 秒

#EXT-X-MAP:URI="init.mp4"
#EXT-X-PROGRAM-DATE-TIME:2026-08-17T10:30:00.000Z       ← 绝对时间戳（同步多码率）
#EXT-X-PART:DURATION=1.001,URI="seg00000_part0.m4s"     ← 子分片（Part）
#EXT-X-PART:DURATION=1.001,URI="seg00000_part1.m4s"
#EXT-X-PART:DURATION=1.001,URI="seg00000_part2.m4s"
#EXTINF:6.006,
seg00000.m4s                                            ← 完整分片（向后兼容）
```

**LL-HLS 的关键机制**：
- `#EXT-X-PART`：1 秒以内的子分片，播放器能边下载边播
- `#EXT-X-SERVER-CONTROL`：告诉播放器支持阻塞重载（Blocking Reload）
- 延迟从传统 HLS 的 18 秒降至 2-3 秒

#### 对比总结

| 特性 | TS m3u8 | fMP4 m3u8 | LL-HLS m3u8 |
|------|---------|-----------|-------------|
| **版本号** | `#EXT-X-VERSION:3-4` | `#EXT-X-VERSION:6-7` | `#EXT-X-VERSION:9+` |
| **初始化段** | ❌ 无 | ✅ `#EXT-X-MAP` | ✅ `#EXT-X-MAP` |
| **分片扩展名** | `.ts` | `.m4s` 或 `.mp4` | `.m4s` |
| **子分片支持** | ❌ | ❌ | ✅ `#EXT-X-PART` |
| **播放器依赖** | 任意分片可播 | 必须先下载 init.mp4 | 必须先下载 init.mp4 |
| **典型延迟** | 18-30 秒 | 12-20 秒 | 2-3 秒 |

#### init.mp4 的作用（重要！）

这是 fMP4 最容易被忽略的机制：

```
TS 模式：
每个 segment.ts 都是"自包含"的
├─ PAT（节目关联表）
├─ PMT（节目映射表）
├─ PES 头（包流头）
└─ 视频/音频 payload
→ 播放器可以从任意分片开始播放

fMP4 模式：
init.mp4（一次性下载，~10KB）包含：
├─ moov box（电影头）
│   ├─ mvhd（时间轴、duration）
│   ├─ trak（轨道定义：视频/音频/字幕）
│   ├─ mdia（编解码器信息：avc1.64001f）
│   └─ stbl（采样表）

segment000.m4s（纯媒体数据）只有：
├─ moof box（分片元数据：帧偏移、时间戳）
└─ mdat box（实际视频/音频帧 payload）

→ 播放器必须先读 init.mp4，否则无法解析 .m4s
→ 省去每个分片重复写 moov，体积减少 ~15%
```

**播放器加载流程对比**：

```
TS 播放流程：
1. 下载 index.m3u8
2. 直接下载 segment000.ts 并播放
   （每个 TS 自包含全部元数据）

fMP4 播放流程：
1. 下载 index.m3u8
2. 解析 #EXT-X-MAP，下载 init.mp4（一次性）
3. 下载 segment000.m4s 并用 init.mp4 解析
   （每个 .m4s 只有媒体数据，依赖 init.mp4）
```

**实际文件大小对比**（6 秒 1080p 分片）：

```
TS 格式：
segment000.ts  ~12.5 MB  （包含 PAT/PMT/PES 头 ~300 KB）

fMP4 格式：
init.mp4       ~10 KB    （一次性开销）
segment000.m4s ~10.6 MB  （纯媒体数据）
segment001.m4s ~10.6 MB
segment002.m4s ~10.6 MB
总计：         ~31.8 MB + 10 KB

TS 总计：      ~37.5 MB
节省：         ~15%（5.7 MB）
```

## 2. 基础 HLS 打包

### 2.1 本地视频转 HLS（VOD）

```bash
# 最简单的 HLS 打包
ffmpeg -i input.mp4 -c:v copy -c:a copy \
  -hls_time 6 \
  -hls_playlist_type vod \
  output/index.m3u8
```

### 2.2 转码同时打包

```bash
# 重新编码后打包
ffmpeg -i input.mp4 \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_filename "output/segment%03d.ts" \
  output/index.m3u8
```

### 2.3 直播模式 HLS

```bash
# 直播：滑动窗口保留最近 5 个分片
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset ultrafast -tune zerolatency -b:v 2M \
  -c:a aac -b:a 128k \
  -hls_time 4 \
  -hls_list_size 5 \
  -hls_flags delete_segments \
  -hls_segment_filename "/var/hls/segment%05d.ts" \
  /var/hls/live.m3u8
```

参数说明：
- `hls_list_size 5`：m3u8 保留最近 5 个分片引用
- `delete_segments`：自动删除已滑出窗口的旧分片
- `hls_time 4`：每个分片约 4 秒（建议直播用 2-4 秒）

## 3. HLS 参数详解

### 3.1 核心参数

| 参数 | 说明 | 默认值 | 建议值 |
|------|------|--------|--------|
| `-hls_time` | 分片目标时长（秒） | 2 | VOD: 6-10, 直播: 2-4 |
| `-hls_list_size` | m3u8 保留分片数量 | 5 | 直播: 3-5, VOD: 0（全部）|
| `-hls_segment_filename` | 分片文件名模板 | 无 | `segment%05d.ts` |
| `-hls_playlist_type` | vod / event / 无 | 无 | VOD: vod, 直播: 不设 |

### 3.2 hls_flags 详解

```bash
# 多个 flag 用 + 连接
-hls_flags delete_segments+append_list+split_by_time
```

| flag | 作用 |
|------|------|
| `delete_segments` | 自动删除滑出窗口的旧分片文件 |
| `append_list` | 追加模式（续传已有 m3u8） |
| `split_by_time` | 精确按时间切分（忽略关键帧） |
| `omit_endlist` | 不写 `#EXT-X-ENDLIST`（直播） |
| `independent_segments` | 启用独立分片标签（提升 CDN 缓存）|

### 3.3 分片命名模板

```bash
# 按序号命名
-hls_segment_filename "output/seg%03d.ts"   # seg000.ts, seg001.ts...

# 按时间戳命名（便于调试）
-hls_segment_filename "output/seg_%Y%m%d_%H%M%S.ts"

# 按帧数命名（精确控制）
-hls_segment_filename "output/seg%010d.ts"  # 10 位数字
```

## 4. 自适应码率 HLS（ABR）

### 4.1 多码率 HLS 结构

```
output/
├── master.m3u8           ← 主播放列表（索引各码率）
├── 1080p/
│   ├── index.m3u8
│   └── segment*.ts
├── 720p/
│   ├── index.m3u8
│   └── segment*.ts
└── 480p/
    ├── index.m3u8
    └── segment*.ts
```

### 4.2 FFmpeg 生成多码率 HLS

```bash
ffmpeg -i input.mp4 \
  -filter_complex "\
    [0:v]split=3[v1][v2][v3];\
    [v1]scale=1920:1080[1080p];\
    [v2]scale=1280:720[720p];\
    [v3]scale=854:480[480p]" \
  -map "[1080p]" -map 0:a -c:v libx264 -b:v 5M -c:a aac -b:a 192k \
    -hls_time 6 -hls_playlist_type vod \
    -hls_segment_filename "output/1080p/seg%03d.ts" output/1080p/index.m3u8 \
  -map "[720p]" -map 0:a -c:v libx264 -b:v 3M -c:a aac -b:a 128k \
    -hls_time 6 -hls_playlist_type vod \
    -hls_segment_filename "output/720p/seg%03d.ts" output/720p/index.m3u8 \
  -map "[480p]" -map 0:a -c:v libx264 -b:v 1.5M -c:a aac -b:a 96k \
    -hls_time 6 -hls_playlist_type vod \
    -hls_segment_filename "output/480p/seg%03d.ts" output/480p/index.m3u8
```

### 4.3 手动生成 master.m3u8

```m3u8
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=5192000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=3128000,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
720p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1596000,RESOLUTION=854x480,CODECS="avc1.64001e,mp4a.40.2"
480p/index.m3u8
```

## 5. HLS 加密：AES-128

### 5.1 生成密钥和 key_info 文件

```bash
# 步骤1：生成 16 字节密钥
openssl rand 16 > enc.key

# 步骤2：生成 IV（初始化向量）
echo $(openssl rand -hex 16) > enc.iv

# 步骤3：创建 key_info_file（FFmpeg 需要）
cat > enc.keyinfo <<EOF
https://example.com/keys/enc.key
enc.key
$(cat enc.iv)
EOF
```

**key_info 格式**（三行）：
1. 密钥 URI（写入 m3u8，客户端访问此 URL 获取密钥）
2. 本地密钥文件路径（FFmpeg 打包时读取）
3. IV 值（十六进制，32 字符）

### 5.2 加密 HLS 打包

```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -b:v 3M -c:a aac -b:a 128k \
  -hls_time 6 \
  -hls_key_info_file enc.keyinfo \
  -hls_playlist_type vod \
  -hls_segment_filename "output/seg%03d.ts" \
  output/index.m3u8
```

### 5.3 加密后的 m3u8 示例

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-KEY:METHOD=AES-128,URI="https://example.com/keys/enc.key",IV=0x8cb84e3f9a5d2c1b0f6e3a7d4c9b2e1a

#EXTINF:6.006000,
seg000.ts
#EXTINF:6.006000,
seg001.ts
```

**播放器流程**：
1. 解析 `#EXT-X-KEY` 获取密钥 URI 和 IV
2. HTTP GET 获取 `enc.key`（需鉴权）
3. 用 AES-128-CBC 解密分片（IV + key）

## 6. 低延迟 HLS（LL-HLS）

### 6.1 LL-HLS 原理

传统 HLS 延迟 = 3×分片时长（6秒 → 18秒延迟）  
LL-HLS 延迟 = 2-3 秒（通过 Part + Preload Hint 机制）

```
传统 HLS:
[----6秒分片----][----6秒分片----][----6秒分片----]
                                 ↑ 播放器等待完整分片

LL-HLS:
[--1秒Part--][--1秒Part--][--1秒Part--][--1秒Part--]
            ↑ 播放器每秒获取 Part，实时跟进
```

### 6.2 LL-HLS 打包（fMP4 格式）

```bash
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset ultrafast -tune zerolatency -g 30 -sc_threshold 0 \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 6 \
  -hls_list_size 4 \
  -hls_flags independent_segments+program_date_time \
  -hls_segment_type fmp4 \
  -hls_fmp4_init_filename "init.mp4" \
  -var_stream_map "v:0,a:0" \
  -hls_segment_filename "seg%05d.m4s" \
  -master_pl_name master.m3u8 \
  stream.m3u8
```

**关键参数**：
- `hls_segment_type fmp4`：使用 fMP4（LL-HLS 必需）
- `hls_fmp4_init_filename`：初始化分片（包含 moov）
- `independent_segments`：启用独立分片标签
- `program_date_time`：添加绝对时间戳（同步多码率）
- `-g 30 -sc_threshold 0`：固定 GOP，关闭场景切换检测

### 6.3 LL-HLS m3u8 特征

```m3u8
#EXT-X-TARGETDURATION:6
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.5
#EXT-X-PART-INF:PART-TARGET=1.0

#EXT-X-MAP:URI="init.mp4"
#EXT-X-PROGRAM-DATE-TIME:2026-07-28T10:30:00.000Z
#EXT-X-PART:DURATION=1.001,URI="seg00000_part0.m4s"
#EXT-X-PART:DURATION=1.001,URI="seg00000_part1.m4s"
#EXTINF:6.006,
seg00000.m4s
```

## 7. RTMP 直播转 HLS

### 7.1 RTMP 推流 → HLS 分发

```bash
# 接收 RTMP 推流并转为 HLS（同时存档）
ffmpeg -i rtmp://0.0.0.0:1935/live/mystream \
  -c:v libx264 -preset veryfast -tune zerolatency -b:v 2M -maxrate 2M -bufsize 4M \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 4 \
  -hls_list_size 6 \
  -hls_flags delete_segments+omit_endlist \
  -hls_segment_filename "/var/hls/live/seg%06d.ts" \
  /var/hls/live/index.m3u8 \
  -c copy /var/recordings/$(date +%Y%m%d_%H%M%S).mp4
```

**两路输出**：
- 第一路：HLS 分片（`-f hls`）
- 第二路：MP4 录像（`-c copy`），直接从 RTMP 拷贝流

### 7.2 多码率 RTMP 转 HLS

```bash
ffmpeg -i rtmp://0.0.0.0:1935/live/stream \
  -filter_complex "[0:v]split=2[v1][v2];\
    [v1]scale=1280:720[720p];\
    [v2]scale=854:480[480p]" \
  -map "[720p]" -c:v libx264 -preset veryfast -b:v 3M -c:a aac -b:a 128k \
    -f hls -hls_time 4 -hls_list_size 5 -hls_flags delete_segments \
    -hls_segment_filename "/var/hls/720p/seg%06d.ts" /var/hls/720p/index.m3u8 \
  -map "[480p]" -c:v libx264 -preset veryfast -b:v 1.5M -c:a aac -b:a 96k \
    -f hls -hls_time 4 -hls_list_size 5 -hls_flags delete_segments \
    -hls_segment_filename "/var/hls/480p/seg%06d.ts" /var/hls/480p/index.m3u8
```

## 8. HLS 直播流录制与存档

### 8.1 同步录制 + 分片存档

```bash
# 边推 HLS 边录制完整 MP4
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset fast -b:v 3M -c:a aac -b:a 128k \
  -f tee \
  "[f=hls:hls_time=6:hls_list_size=5:hls_flags=delete_segments:hls_segment_filename=/hls/seg%06d.ts]/hls/index.m3u8|\
  [f=mp4:movflags=faststart]/recordings/live_$(date +%Y%m%d_%H%M%S).mp4"
```

**`-f tee`**：将同一路转码结果输出到多个文件/格式，避免重复编码。

### 8.2 按小时切割存档

```bash
# 每小时自动生成新的录制文件（segment_time=3600）
ffmpeg -i rtmp://localhost/live/stream \
  -c copy \
  -f segment \
  -segment_time 3600 \
  -segment_atclocktime 1 \
  -segment_format mp4 \
  -reset_timestamps 1 \
  -strftime 1 \
  "/recordings/%Y%m%d_%H0000.mp4"
```

**参数说明**：
- `-f segment`：分段录制格式
- `-segment_atclocktime 1`：在整点切割
- `-strftime 1`：文件名使用时间格式
- `-reset_timestamps 1`：每段重置时间戳从 0 开始

## 9. CDN 分发优化

### 9.1 分片命名与缓存策略

```bash
# 使用固定域名前缀，便于 CDN 缓存分发
ffmpeg -i input.mp4 \
  -hls_base_url "https://cdn.example.com/hls/live/" \
  -hls_segment_filename "/var/hls/live/seg%06d.ts" \
  /var/hls/live/index.m3u8
```

**分片缓存头**（需 Web 服务器配置）：

```nginx
# TS 分片：长期缓存（内容不变）
location ~* \.ts$ {
    add_header Cache-Control "public, max-age=86400";
}

# m3u8 播放列表：不缓存（直播）或短缓存
location ~* \.m3u8$ {
    add_header Cache-Control "no-cache, no-store";
}
```

### 9.2 版本化分片（防缓存碰撞）

```bash
# 在分片路径中加入随机 token（每次直播唯一）
STREAM_TOKEN=$(date +%s)
ffmpeg -i rtmp://localhost/live/stream \
  -hls_segment_filename "/var/hls/${STREAM_TOKEN}/seg%06d.ts" \
  -hls_base_url "https://cdn.example.com/hls/${STREAM_TOKEN}/" \
  /var/hls/index.m3u8
```

### 9.3 多 CDN 主备切换（master.m3u8）

```m3u8
#EXTM3U
#EXT-X-VERSION:3

# 主 CDN
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
https://cdn-primary.example.com/hls/720p/index.m3u8

# 备用 CDN
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
https://cdn-backup.example.com/hls/480p/index.m3u8
```

## 10. Go 集成：HLS 打包服务

### 10.1 HLS 打包器

```go
package hls

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"time"
)

// HLSProfile HLS 转码规格
type HLSProfile struct {
	Name        string
	Width       int
	Height      int
	VideoBitrate string
	AudioBitrate string
	HLSTime     int    // 分片时长（秒）
	ListSize    int    // 滑动窗口大小（0=全部）
}

// HLSPackager HLS 打包服务
type HLSPackager struct {
	outputDir   string
	baseURL     string
}

// NewHLSPackager 创建 HLS 打包器
func NewHLSPackager(outputDir, baseURL string) *HLSPackager {
	return &HLSPackager{outputDir: outputDir, baseURL: baseURL}
}

// PackageVOD 本地文件 → HLS VOD
func (p *HLSPackager) PackageVOD(ctx context.Context, inputPath string, profiles []HLSProfile) error {
	if err := os.MkdirAll(p.outputDir, 0755); err != nil {
		return fmt.Errorf("创建输出目录失败: %w", err)
	}
	
	args := []string{"-i", inputPath}
	
	// filter_complex：分流
	if len(profiles) > 1 {
		args = append(args, "-filter_complex",
			fmt.Sprintf("[0:v]split=%d%s", len(profiles), buildSplitLabels(len(profiles))))
	}
	
	// 每路输出
	for i, p2 := range profiles {
		outDir := filepath.Join(p.outputDir, p2.Name)
		if err := os.MkdirAll(outDir, 0755); err != nil {
			return fmt.Errorf("创建 %s 目录失败: %w", p2.Name, err)
		}
		segTemplate := filepath.Join(outDir, "seg%05d.ts")
		m3u8Path := filepath.Join(outDir, "index.m3u8")
		
		if len(profiles) > 1 {
			args = append(args, "-map", fmt.Sprintf("[v%d]", i), "-map", "0:a")
		}
		args = append(args,
			"-c:v", "libx264", "-b:v", p2.VideoBitrate,
			"-c:a", "aac", "-b:a", p2.AudioBitrate,
			"-hls_time", fmt.Sprintf("%d", p2.HLSTime),
			"-hls_playlist_type", "vod",
			"-hls_list_size", "0",
			"-hls_segment_filename", segTemplate,
			m3u8Path,
		)
	}
	
	cmd := exec.CommandContext(ctx, "ffmpeg", args...)
	return cmd.Run()
}
```

### 10.2 直播 HLS 服务

```go
// LiveHLSConfig 直播 HLS 配置
type LiveHLSConfig struct {
	RTMPSource  string
	OutputDir   string
	HLSTime     int
	ListSize    int
	StreamToken string
}

// LiveHLSService 直播 HLS 打包服务
type LiveHLSService struct {
	config LiveHLSConfig
	cmd    *exec.Cmd
	cancel context.CancelFunc
}

// Start 启动直播 HLS 打包
func (s *LiveHLSService) Start(ctx context.Context) error {
	ctx, s.cancel = context.WithCancel(ctx)
	
	if s.config.StreamToken == "" {
		s.config.StreamToken = fmt.Sprintf("%d", time.Now().Unix())
	}
	
	segDir := filepath.Join(s.config.OutputDir, s.config.StreamToken)
	if err := os.MkdirAll(segDir, 0755); err != nil {
		return fmt.Errorf("创建分片目录失败: %w", err)
	}
	
	segTemplate := filepath.Join(segDir, "seg%06d.ts")
	m3u8Path := filepath.Join(s.config.OutputDir, "live.m3u8")
	
	s.cmd = exec.CommandContext(ctx, "ffmpeg",
		"-i", s.config.RTMPSource,
		"-c:v", "libx264", "-preset", "veryfast", "-tune", "zerolatency", "-b:v", "2M",
		"-c:a", "aac", "-b:a", "128k",
		"-f", "hls",
		"-hls_time", fmt.Sprintf("%d", s.config.HLSTime),
		"-hls_list_size", fmt.Sprintf("%d", s.config.ListSize),
		"-hls_flags", "delete_segments+omit_endlist",
		"-hls_segment_filename", segTemplate,
		m3u8Path,
	)
	
	return s.cmd.Start()
}

// Stop 停止直播
func (s *LiveHLSService) Stop() error {
	if s.cancel != nil {
		s.cancel()
	}
	if s.cmd != nil && s.cmd.Process != nil {
		return s.cmd.Process.Signal(os.Interrupt)
	}
	return nil
}
```

### 10.3 master.m3u8 生成

```go
// MasterPlaylist master.m3u8 生成器
func GenerateMasterPlaylist(baseURL string, profiles []HLSProfile) string {
	var sb strings.Builder
	sb.WriteString("#EXTM3U\n#EXT-X-VERSION:3\n\n")
	
	for _, p := range profiles {
		bw := parseBitrate(p.VideoBitrate) + parseBitrate(p.AudioBitrate)
		sb.WriteString(fmt.Sprintf("#EXT-X-STREAM-INF:BANDWIDTH=%d,RESOLUTION=%dx%d\n",
			bw, p.Width, p.Height))
		sb.WriteString(fmt.Sprintf("%s/%s/index.m3u8\n\n", baseURL, p.Name))
	}
	
	return sb.String()
}

// parseBitrate 解析码率字符串（"3M" → 3000000）
func parseBitrate(bitrate string) int {
	bitrate = strings.TrimSpace(bitrate)
	if strings.HasSuffix(bitrate, "M") {
		val, _ := strconv.Atoi(strings.TrimSuffix(bitrate, "M"))
		return val * 1000000
	}
	if strings.HasSuffix(bitrate, "k") {
		val, _ := strconv.Atoi(strings.TrimSuffix(bitrate, "k"))
		return val * 1000
	}
	val, _ := strconv.Atoi(bitrate)
	return val
}
```

## 11. 动手实践

### 实验一：单码率 HLS VOD

```bash
# 将本地视频转为 HLS VOD
ffmpeg -i big_buck_bunny.mp4 \
  -c:v libx264 -crf 23 -c:a aac -b:a 128k \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_filename "output/seg%03d.ts" \
  output/index.m3u8

# 用 HTTP 服务器播放
cd output && python3 -m http.server 8080
# 浏览器访问：http://localhost:8080/index.m3u8
```

### 实验二：多码率 ABR HLS（Go 调用）

```go
package main

import (
	"context"
	"log"
)

func main() {
	packager := NewHLSPackager("./output", "https://cdn.example.com/hls")
	
	profiles := []HLSProfile{
		{Name: "1080p", Width: 1920, Height: 1080, VideoBitrate: "5M", AudioBitrate: "192k", HLSTime: 6},
		{Name: "720p", Width: 1280, Height: 720, VideoBitrate: "3M", AudioBitrate: "128k", HLSTime: 6},
		{Name: "480p", Width: 854, Height: 480, VideoBitrate: "1.5M", AudioBitrate: "96k", HLSTime: 6},
	}
	
	err := packager.PackageVOD(context.Background(), "input.mp4", profiles)
	if err != nil {
		log.Fatal(err)
	}
	
	// 生成 master.m3u8
	master := GenerateMasterPlaylist("", profiles)
	os.WriteFile("./output/master.m3u8", []byte(master), 0644)
}
```

### 实验三：加密 HLS 流

```bash
# 步骤1：生成密钥和 key_info 文件
openssl rand 16 > enc.key
echo $(openssl rand -hex 16) > enc.iv
cat > enc.keyinfo <<EOF
https://example.com/keys/enc.key
enc.key
$(cat enc.iv)
EOF

# 步骤2：打包加密 HLS
ffmpeg -i input.mp4 \
  -c:v libx264 -crf 23 -c:a aac -b:a 128k \
  -hls_time 6 \
  -hls_key_info_file enc.keyinfo \
  -hls_playlist_type vod \
  -hls_segment_filename "encrypted/seg%03d.ts" \
  encrypted/index.m3u8

# 步骤3：部署密钥服务（需鉴权）
# 将 enc.key 上传到 https://example.com/keys/enc.key
# 播放器需验证 token 才能获取密钥
```

### 实验四：RTMP 直播转 HLS（Go 服务）

```go
service := &LiveHLSService{
	config: LiveHLSConfig{
		RTMPSource:  "rtmp://localhost:1935/live/stream",
		OutputDir:   "/var/hls",
		HLSTime:     4,
		ListSize:    5,
		StreamToken: fmt.Sprintf("live_%d", time.Now().Unix()),
	},
}

if err := service.Start(context.Background()); err != nil {
	log.Fatal(err)
}

log.Println("HLS 直播已启动，访问 /var/hls/live.m3u8")

// 等待中断信号
c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt)
<-c

service.Stop()
```

## 12. 本章总结

本章全面覆盖了 HLS 在生产环境的核心场景：

| 场景 | 关键参数/技术 | 说明 |
|------|---------------|------|
| VOD 打包 | `-hls_playlist_type vod` / `hls_list_size 0` | 全分片输出，静态播放 |
| 直播推流 | `hls_list_size 5` / `delete_segments` | 滑动窗口，自动清理旧分片 |
| ABR 多码率 | `split` + 多路 `-map` | 一次编码多路输出 |
| DRM 加密 | `-hls_key_info_file` / AES-128 | 需密钥服务配合 |
| 低延迟 | `hls_segment_type fmp4` / LL-HLS | 2-3秒延迟，需 fMP4 格式 |
| 多路输出 | `-f tee` | 同时生成 HLS + MP4 录像 |
| CDN 优化 | `-hls_base_url` / 版本化路径 | 防缓存碰撞，提升命中率 |

**关键原则**：
1. **直播分片时长 2-4 秒**，VOD 用 6-10 秒（平衡延迟与请求频率）
2. **ABR 优先用 split 一次编码**，多次输入同一文件会重复解码
3. **加密 key_info 中的 URI** 是客户端访问地址，不是本地路径
4. **LL-HLS 必须用 fMP4**，TS 格式不支持 Part 机制
5. **m3u8 不能缓存**，TS 分片可长期缓存（内容不变）

**下一章**：[第 17 篇：DASH 流媒体](17-DASH流媒体.md)

深入 MPEG-DASH 协议，对比 HLS vs DASH 的分片结构差异，以及 fMP4 初始化段、媒体段的 MPD 生成方式。

## 13. 术语表

| 术语 | 英文全称 | 说明 |
|------|----------|------|
| HLS | HTTP Live Streaming | Apple 自适应流协议 |
| m3u8 | M3U Extended | HLS 播放列表格式（UTF-8 编码）|
| TS | MPEG Transport Stream | HLS 分片的容器格式 |
| fMP4 | fragmented MP4 | ISO BMFF 分片 MP4，LL-HLS 使用 |
| ABR | Adaptive Bitrate | 自适应码率，按带宽选择清晰度 |
| VOD | Video on Demand | 点播，有 `#EXT-X-ENDLIST` 标签 |
| LL-HLS | Low Latency HLS | 苹果 2019 年发布的低延迟 HLS 扩展 |
| Part | Partial Segment | LL-HLS 的子分片（1秒以内）|
| DRM | Digital Rights Management | 数字版权管理，AES-128 是最基础形式 |
| IV | Initialization Vector | AES 加密初始化向量（128位）|
| CDN | Content Delivery Network | 内容分发网络 |
| GOP | Group of Pictures | I帧间隔，影响分片边界精度 |

---

*本文属于 [FFmpeg 完全指南](../README.md) 系列第 16 篇。*

