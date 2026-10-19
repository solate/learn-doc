# HLS 协议详解（CDN 分发标准）

> **适合读者**：后端开发者、流媒体工程师、对直播/点播架构感兴趣的技术人员  
> **前置知识**：HTTP 基础、[第 03 篇：音频编码与视频封装格式](03-音频编码与视频封装格式.md)

---

## 一、HLS 是什么

**HLS（HTTP Live Streaming）** 是苹果公司 2009 年推出的流媒体传输协议。它的核心思路是：**把视频切成小文件（TS 分片），通过普通 HTTP 服务器传输**。

```
传统流媒体：视频数据通过专用协议持续推送
HLS：把视频切成 2-6 秒的小文件，客户端轮询下载
```

**HLS 的本质是一个"伪流媒体"**——视频内容存在普通 HTTP 服务器上，播放器不断地下载下一个分片，造成连续播放的效果。

### 1.1 为什么 HLS 成为 CDN 分发标准？

| 问题 | RTMP 的困境 | HLS 的解法 |
|------|------------|-----------|
| 防火墙穿透 | 使用 1935 端口，常被封锁 | 走 443/80，无任何阻碍 |
| CDN 支持 | 需要专用 RTMP CDN | 任意 HTTP CDN 都能分发 |
| 苹果设备 | iOS Safari 不支持 Flash | iOS 原生支持 HLS |
| 服务器压力 | 维持长连接，并发有限 | 无状态 HTTP，CDN 缓存 |
| 自适应码率 | 不支持 | 原生支持 ABR |

---

## 二、M3U8 格式深度解析

### 2.1 播放列表文件（M3U8）

M3U8 是 HLS 的核心——一个文本文件，指向一系列 TS 分片：

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0

#EXTINF:6.000000,
segment000.ts
#EXTINF:6.000000,
segment001.ts
#EXTINF:5.960000,
segment002.ts
#EXT-X-ENDLIST
```

**标签说明**：

| 标签 | 说明 |
|------|------|
| `#EXTM3U` | M3U8 文件头（必须第一行） |
| `#EXT-X-VERSION:3` | 协议版本（常用 3 或 7） |
| `#EXT-X-TARGETDURATION:6` | 每个分片的最大时长（秒） |
| `#EXT-X-MEDIA-SEQUENCE:0` | 第一个分片的序号 |
| `#EXTINF:6.0,` | 下一个分片的时长 |
| `#EXT-X-ENDLIST` | 点播结束标记（直播不加此标签） |

### 2.2 直播 M3U8 vs 点播 M3U8

**直播 M3U8**（滑动窗口）：
```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:2847      ← 窗口起始序号持续增大

#EXTINF:4.000000,
segment2847.ts
#EXTINF:4.000000,
segment2848.ts
#EXTINF:4.000000,
segment2849.ts
                                 ← 无 #EXT-X-ENDLIST
```

**直播窗口策略**：
- 通常保留最近 3-6 个分片（12-30 秒的内容）
- 客户端每隔一个 `TARGETDURATION` 刷新 M3U8
- 旧分片从 M3U8 移除，但文件可能保留一段时间

**点播 M3U8**：
- 包含所有分片，有 `#EXT-X-ENDLIST`
- 可以随机跳转（Seek）

### 2.3 主播放列表（Master Playlist）

自适应码率的入口文件：

```m3u8
#EXTM3U

#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360,CODECS="avc1.42c01e,mp4a.40.2"
360p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
720p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p.m3u8
```

**字段说明**：
- `BANDWIDTH`：峰值码率（bps）
- `RESOLUTION`：视频分辨率
- `CODECS`：编码格式（视频+音频）

播放器根据当前网速自动选择最合适的码率层。

### 2.4 TS 分片内部结构

每个 `.ts` 文件是一个 MPEG-TS 容器：

```
TS 包（188 字节固定大小）:
[同步字节 0x47][传输标记][PID][...][负载数据]

PID 0x0000: PAT（Program Association Table）- 节目关联表
PID 0x0001: CAT（Conditional Access Table）
PID 0x0100: PMT（Program Map Table）- 节目映射表
PID 0x0101: 视频流（H.264/H.265）
PID 0x0102: 音频流（AAC/MP3）
```

**为什么用 TS 而不是 MP4？**
- TS 是流式格式，可以从任意位置开始解码（适合分片传输）
- MP4 的 moov box 在文件末尾（需完整下载才能播放）
- HLS 版本 7+ 开始支持 fMP4（Fragmented MP4）分片

---

## 三、分片策略对延迟的影响

### 3.1 分片时长与延迟的关系

**延迟来源**：
```
总延迟 = 编码延迟 + 分片时长 + 缓冲分片数 × 分片时长 + 传输延迟

标准 HLS（6 秒分片）：
  编码延迟：1-2s
  分片时长：6s
  缓冲分片：3 个
  总延迟：~20-25 秒

短分片 HLS（2 秒分片）：
  编码延迟：0.5-1s
  分片时长：2s
  缓冲分片：3 个
  总延迟：~8-10 秒

Low-Latency HLS（0.2 秒分片）：
  使用 CMAF Chunks
  总延迟：~2-4 秒
```

### 3.2 分片时长权衡

| 分片时长 | 延迟 | HTTP 请求数/分钟 | CDN 压力 | 适合场景 |
|---------|------|----------------|---------|---------|
| 10 秒 | 30-40s | 6 | 低 | 点播、录播 |
| 6 秒 | 20-25s | 10 | 中 | 普通直播 |
| 2 秒 | 8-12s | 30 | 高 | 低延迟直播 |
| 0.2 秒（LL-HLS） | 2-4s | 300 | 极高 | 超低延迟场景 |

### 3.3 客户端缓冲策略

播放器通常预缓冲 3 个分片才开始播放：

```
分片 1 [已下载]
分片 2 [已下载]
分片 3 [已下载] → 开始播放
分片 4 [下载中]
```

**ABR 切换策略**：
```
带宽估算：
  当前下载速度 > 当前码率 × 1.5 → 考虑升码率
  当前下载速度 < 当前码率 × 0.8 → 立即降码率

切换时机：
  码率升降在分片边界（关键帧）处进行
  避免在分片中间切换（会产生花屏）
```

---

## 四、Low-Latency HLS（LL-HLS）

苹果 2019 年推出 LL-HLS（HLS 版本 9），将延迟降到 2-4 秒：

### 4.1 核心技术：Part 和 Preload Hint

**Part（分块）**：把一个分片分成更小的 Part（0.1-0.5 秒）：

```m3u8
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-TARGETDURATION:6
#EXT-X-PART-INF:PART-TARGET=0.2

#EXTINF:6.000000,
#EXT-X-PART:DURATION=0.2,URI="part0.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.2,URI="part1.mp4"
#EXT-X-PART:DURATION=0.2,URI="part2.mp4"
...
segment100.ts

#EXT-X-PRELOAD-HINT:TYPE=PART,URI="part30.mp4"  ← 预告下一个 Part
```

**Blocking Playlist Reload（阻塞刷新）**：
```
传统 HLS：客户端每隔 T 秒轮询，服务器立即返回
LL-HLS：客户端请求带 _HLS_msn 和 _HLS_part 参数
         服务器在新 Part 出现前 Hold 请求（长轮询）
         新 Part 一出现立即响应，减少空轮询
```

```http
GET /stream.m3u8?_HLS_msn=100&_HLS_part=5 HTTP/1.1
```

### 4.2 LL-HLS 与普通 HLS 延迟对比

```
普通 HLS（6s 分片）：
  时间轴：|---6s---|---6s---|---6s---|
  播放点：           在此处播放（延迟 ~18s）

LL-HLS（6s 分片 + 0.2s Part）：
  时间轴：|0.2|0.2|0.2|...|0.2|0.2|
  播放点：仅延迟 2-3 个 Part = 0.4-0.6s + 网络传输
  实际总延迟：~2-4 秒
```

---

## 五、HLS 加密

### 5.1 AES-128 加密分片

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://keyserver.example.com/key",IV=0x00000000000000000000000000000001

#EXTINF:6.0,
segment001.ts  ← 内容用 AES-128-CBC 加密
```

**工作流程**：
1. 播放器请求 `URI` 获取 16 字节密钥
2. 用密钥 + IV 解密 TS 分片
3. 密钥服务器可以做鉴权（Token 验证）

**生成加密 HLS（FFmpeg）**：
```bash
# 生成密钥
openssl rand 16 > enc.key
echo "https://keyserver.example.com/enc.key" > enc.keyinfo
echo "enc.key" >> enc.keyinfo

# 生成加密 HLS
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -hls_key_info_file enc.keyinfo \
  -hls_time 6 \
  -f hls output.m3u8
```

### 5.2 Sample-AES（更安全）

Sample-AES 对每个 NAL 单元单独加密，即使分片被缓存，也无法绕过解密。

```m3u8
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://key-id",KEYFORMAT="com.apple.streamingkeydelivery"
```

---

## 六、用 FFmpeg 生成 HLS

### 6.1 基础 HLS 生成

```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_filename "segment%03d.ts" \
  output.m3u8
```

### 6.2 多码率自适应 HLS

```bash
#!/bin/bash
INPUT="input.mp4"
OUTPUT_DIR="hls_output"
mkdir -p "$OUTPUT_DIR"

ffmpeg -i "$INPUT" \
  # 360p
  -vf scale=640:360 -c:v libx264 -b:v 800k -c:a aac -b:a 96k \
  -hls_time 6 -hls_segment_filename "$OUTPUT_DIR/360p_%03d.ts" \
  "$OUTPUT_DIR/360p.m3u8" \
  # 720p
  -vf scale=1280:720 -c:v libx264 -b:v 2800k -c:a aac -b:a 128k \
  -hls_time 6 -hls_segment_filename "$OUTPUT_DIR/720p_%03d.ts" \
  "$OUTPUT_DIR/720p.m3u8" \
  # 1080p
  -vf scale=1920:1080 -c:v libx264 -b:v 5000k -c:a aac -b:a 192k \
  -hls_time 6 -hls_segment_filename "$OUTPUT_DIR/1080p_%03d.ts" \
  "$OUTPUT_DIR/1080p.m3u8"

# 生成 Master Playlist
cat > "$OUTPUT_DIR/master.m3u8" << 'EOF'
#EXTM3U

#EXT-X-STREAM-INF:BANDWIDTH=900000,RESOLUTION=640x360,CODECS="avc1.42c01e,mp4a.40.2"
360p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
720p.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5200000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p.m3u8
EOF
```

### 6.3 直播推流转 HLS

```bash
# 接收 RTMP 推流并切成 HLS（SRS/Nginx-RTMP 更推荐，这里演示 FFmpeg）
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -c:a aac \
  -hls_time 4 \
  -hls_list_size 5 \                    # 直播窗口保留 5 个分片
  -hls_flags delete_segments \          # 自动删除旧分片
  -hls_segment_filename "seg%d.ts" \
  /var/www/html/live/stream.m3u8
```

---

## 七、HLS 与 CDN 配合

### 7.1 CDN 缓存策略

**M3U8 文件**（直播）：不能缓存，每次必须回源：
```
Cache-Control: no-cache, no-store
```

**M3U8 文件**（点播）：可以长期缓存：
```
Cache-Control: public, max-age=31536000
```

**TS 分片**：内容不变，可以永久缓存：
```
Cache-Control: public, max-age=31536000
```

### 7.2 CDN 分发架构

```
[推流端]
    ↓ RTMP/SRT
[源站流媒体服务器 - SRS/MediaMTX]
    ↓ 生成 M3U8 + TS 分片
[源站 HTTP 服务器 / 对象存储]
    ↓
[CDN 源节点（Origin）]
    ↓ 全球分发
[CDN 边缘节点（Edge）]
    ↓
[播放器] 就近访问边缘节点
```

**关键优化**：
- TS 分片上传到对象存储（S3/OSS），CDN 直接回源拉取
- 边缘节点缓存 TS 分片后，后续播放零回源
- M3U8 必须实时获取（直播），TS 分片可缓存

### 7.3 HTTPS 的必要性

```
iOS ATS（App Transport Security）要求：
- 所有 HTTP 连接必须使用 HTTPS
- HLS 的 M3U8 和 TS 分片都必须走 HTTPS

浏览器 Mixed Content 规则：
- HTTPS 页面不能加载 HTTP 的 M3U8（会被阻止）
```

---

## 八、HLS 播放器集成

### 8.1 浏览器端（hls.js）

```html
<video id="video" controls></video>
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script>
const video = document.getElementById('video');

if (Hls.isSupported()) {
  const hls = new Hls({
    // 自定义配置
    maxBufferLength: 30,        // 最大缓冲 30 秒
    lowLatencyMode: false,      // 普通延迟模式
    startLevel: -1,             // -1 表示自动选择初始码率
  });
  hls.loadSource('https://cdn.example.com/live/stream.m3u8');
  hls.attachMedia(video);
  hls.on(Hls.Events.MANIFEST_PARSED, () => video.play());
} else if (video.canPlayType('application/vnd.apple.mpegurl')) {
  // Safari 原生 HLS 支持
  video.src = 'https://cdn.example.com/live/stream.m3u8';
  video.play();
}
</script>
```

### 8.2 Go 后端生成 HLS API

```go
// 点播 HLS：返回 M3U8 播放地址
func (h *VideoHandler) GetPlayURL(c *gin.Context) {
    var req dto.GetPlayURLRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        response.Error(c, xerr.Wrap(xerr.ErrBadRequest.Code, "参数错误", err))
        return
    }

    url, err := h.videoSvc.GetHLSPlayURL(c.Request.Context(), req.VideoID)
    if err != nil {
        response.Error(c, err)
        return
    }

    response.Success(c, dto.PlayURLResponse{PlayURL: url})
}

// Service 层：生成带签名的 CDN 播放地址
func (s *VideoService) GetHLSPlayURL(ctx context.Context, videoID string) (string, error) {
    video, err := s.videoRepo.GetByID(ctx, videoID)
    if err != nil {
        return "", xerr.Wrap(xerr.ErrInternal.Code, "获取视频失败", err)
    }
    if video == nil {
        return "", xerr.New(xerr.ErrNotFound.Code, "视频不存在")
    }

    // m3u8 key: videos/{videoID}/master.m3u8
    key := fmt.Sprintf("videos/%s/master.m3u8", videoID)
    signedURL, err := s.storageProvider.SignURL(ctx, key, 3600) // 1 小时有效
    if err != nil {
        return "", xerr.Wrap(xerr.ErrInternal.Code, "生成播放地址失败", err)
    }

    return signedURL, nil
}
```

---

## 九、HLS 常见问题排查

### 问题 1：播放器一直转圈不播放

**排查步骤**：
```bash
# 1. 检查 M3U8 是否可访问
curl -I https://cdn.example.com/live/stream.m3u8

# 2. 检查 CORS 头（浏览器跨域）
curl -H "Origin: https://player.example.com" \
     -I https://cdn.example.com/live/stream.m3u8
# 应该返回 Access-Control-Allow-Origin: *

# 3. 检查 TS 分片是否可访问
curl -I https://cdn.example.com/live/segment001.ts
```

**常见原因**：
- CDN 没有正确配置 CORS
- M3U8 中分片路径是相对路径但 CDN 解析错误
- 防火墙拦截了某些分片

### 问题 2：延迟越来越大（直播）

**原因**：播放器缓冲了太多分片，没有及时跳跃。

**hls.js 配置**：
```javascript
const hls = new Hls({
  liveMaxLatencyDurationCount: 3,  // 最多缓冲 3 个分片
  liveSyncDurationCount: 2,        // 目标保持 2 个分片缓冲
});
```

### 问题 3：iOS Safari 播放 HLS 加密视频失败

**原因**：iOS Safari 对密钥服务器有严格的 CORS 和 SameSite 要求。

**解决**：
```
密钥服务器响应头：
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET
```

### 问题 4：分片无缝衔接有杂音（音频撕裂）

**原因**：FFmpeg 切片时没有在关键帧处切割，或音频帧边界不对齐。

**解决**：
```bash
ffmpeg -i input.mp4 \
  -c:v libx264 \
  -force_key_frames "expr:gte(t,n_forced*6)" \  # 强制每 6 秒一个关键帧
  -hls_time 6 \
  output.m3u8
```

---

## 十、本篇小结

| 主题 | 核心要点 |
|------|----------|
| **HLS 本质** | 把视频切成小 TS 文件，通过 HTTP 分发 |
| **M3U8 格式** | 指向分片列表；直播滑动窗口，点播带 ENDLIST |
| **分片策略** | 分片越短延迟越低，但请求数越多；6s 是常规选择 |
| **LL-HLS** | Part 机制 + 阻塞刷新，延迟降至 2-4s |
| **CDN 配合** | TS 分片永久缓存；M3U8 直播不缓存 |
| **自适应码率** | Master Playlist + 多码率 M3U8，自动切换 |

**适用场景选择**：
- 全球分发点播 → HLS（CDN 友好）
- 国内大规模直播 → HLS（CDN 成熟）
- 要求超低延迟 → LL-HLS 或 WebRTC
- 企业内网监控 → RTSP/RTMP（HLS 延迟过高）

---

## 下一篇

下一篇将深入讲解 **DASH 与 CMAF**：MPEG-DASH 的动态自适应流，CMAF 如何统一 HLS 和 DASH 的分片格式，以及 fMP4 分片与 TS 分片的对比。

---

## 参考资料

1. [RFC 8216 - HTTP Live Streaming - IETF](https://datatracker.ietf.org/doc/html/rfc8216)
2. [Apple HLS Authoring Specification](https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices)
3. [Low-Latency HLS - Apple Developer](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-hls)
4. [hls.js - GitHub](https://github.com/video-dev/hls.js)
5. [FFmpeg HLS Muxer](https://ffmpeg.org/ffmpeg-formats.html#hls-1)

---

## 动手实践

### 练习1：FFmpeg 生成 HLS 直播流
```bash
# 推一路 RTSP 摄像头流为 HLS，输出到本地 web 目录
ffmpeg -i "rtsp://camera/stream" \
  -c:v libx264 -preset veryfast -g 60 \
  -c:a aac -b:a 128k \
  -f hls -hls_time 4 -hls_list_size 10 \
  -hls_flags delete_segments \
  /var/www/hls/cam1/index.m3u8
```
**预期输出**：`/var/www/hls/cam1/` 下持续产生 `index.m3u8` 和 `index000N.ts` 文件，VLC 打开 `http://server/cam1/index.m3u8` 可播放。

### 练习2：生成多码率 HLS（ABR）
```bash
# 同时生成 360p/720p/1080p 三路
ffmpeg -i input.mp4 \
  -map 0:v -map 0:a -map 0:v -map 0:a -map 0:v -map 0:a \
  -c:v libx264 -preset veryfast \
  -s:v:0 640x360 -b:v:0 500k \
  -s:v:1 1280x720 -b:v:1 1500k \
  -s:v:2 1920x1080 -b:v:2 3000k \
  -c:a aac -b:a 128k \
  -f hls -hls_time 4 -hls_list_size 0 \
  -master_pl_name master.m3u8 \
  -hls_segment_filename "seg_%v_%04d.ts" stream_%v.m3u8
```
**预期输出**：生成 `master.m3u8`（含三路链接）和三个子 m3u8 + .ts 分片。

### 练习3：启用 LL-HLS 低延迟
```bash
ffmpeg -i rtmp://localhost/live/stream \
  -c:v libx264 -preset ultrafast -tune zerolatency -g 30 \
  -c:a aac \
  -f hls \
  -hls_time 1 -hls_list_size 10 \
  -hls_flags +delete_segments+split_by_time \
  -hls_segment_type fmp4 \
  -hls_fmp4_init_filename init.mp4 \
  /var/www/ll-hls/index.m3u8
```
**预期输出**：延迟降至 2-4s，hls.js 0.14+ 以 LL-HLS 模式加载。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| m3u8 返回 404 | HLS 目录权限或路径错误 | 检查 Nginx root 配置，`chmod 755` 目录 |
| 播放器报"no valid segments" | m3u8 内 ts 文件路径不匹配 | 确认相对路径，或改绝对 URL |
| 延迟超过 30s | hls_list_size 设置太小，客户端缓冲过多 | 减小 `hls_time`，检查播放器 `maxBufferLength` |
| 频繁卡顿重缓冲 | 分片时长过短 + 网络抖动 | `hls_time` 改为 4-6s，增大 CDN 缓存 |
| CORS 跨域错误 | Nginx 未配置跨域头 | 添加 `add_header Access-Control-Allow-Origin *;` |
| 加密 HLS 密钥 403 | Key 服务器未授权播放器 IP | 检查 Key Server 认证逻辑，添加来源白名单 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| m3u8 | M3U8 Playlist | HLS 索引文件，包含分片 URL 和元信息 |
| TS 分片 | Transport Stream Segment | HLS 视频分片文件，通常 2-10s 一段 |
| EXT-X-TARGETDURATION | Target Duration | m3u8 中声明的最大分片时长 |
| EXT-X-MEDIA-SEQUENCE | Media Sequence | 当前 m3u8 中第一个分片的序号 |
| LL-HLS | Low-Latency HLS | 苹果低延迟 HLS 扩展，使用部分分片（Part）|
| fMP4 | Fragmented MP4 | LL-HLS 和 CMAF 使用的分片格式，代替传统 .ts |
| EXT-X-KEY | Encryption Key | m3u8 中的加密密钥声明 |
| ABR | Adaptive Bitrate | 自适应码率，播放器根据网络自动切换分辨率 |

---

> **全文完** | 上一篇：[第 9 篇 — SRT 协议与弱网推流](09-SRT协议与弱网推流.md) | 下一篇：[第 11 篇 — DASH 与 CMAF 统一分片格式](11-DASH与CMAF统一分片格式.md)
