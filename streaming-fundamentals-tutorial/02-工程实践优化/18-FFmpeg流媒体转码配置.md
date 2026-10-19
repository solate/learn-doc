# FFmpeg 流媒体转码配置（多码率 HLS）

> **适合读者**：需要生产多码率 HLS 的开发者  
> **篇幅**：精简版，聚焦配置方案

---

## 一、为什么需要多码率 HLS

用户网络质量差异巨大，单一码率无法兼顾：

```
4G 用户：可能只有 1Mbps 带宽
光纤用户：可以流畅播放 10Mbps 内容

多码率 ABR：自动匹配用户网络，减少卡顿 + 节省带宽
```

---

## 二、一键生成多码率 HLS 脚本

```bash
#!/bin/bash
# gen_hls.sh - 生成三档码率 HLS

INPUT="$1"
OUTPUT_DIR="${2:-./hls_output}"
mkdir -p "$OUTPUT_DIR"

ffmpeg -i "$INPUT" \
  -filter_complex \
    "[0:v]split=3[v1][v2][v3]; \
     [v1]scale=1920:1080[v1out]; \
     [v2]scale=1280:720[v2out]; \
     [v3]scale=640:360[v3out]" \
  \
  -map "[v1out]" -c:v:0 libx264 -b:v:0 5000k -maxrate:v:0 5500k -bufsize:v:0 10000k \
    -profile:v:0 high -level:v:0 4.1 \
  -map "[v2out]" -c:v:1 libx264 -b:v:1 2800k -maxrate:v:1 3000k -bufsize:v:1 5600k \
    -profile:v:1 main -level:v:1 3.1 \
  -map "[v3out]" -c:v:2 libx264 -b:v:2 800k  -maxrate:v:2 900k  -bufsize:v:2 1600k \
    -profile:v:2 baseline -level:v:2 3.0 \
  \
  -map 0:a -map 0:a -map 0:a \
  -c:a aac -b:a:0 192k -b:a:1 128k -b:a:2 96k \
  -ar 44100 -ac 2 \
  \
  -force_key_frames "expr:gte(t,n_forced*4)" \
  -f hls \
  -hls_time 4 \
  -hls_playlist_type vod \
  -hls_flags independent_segments \
  -hls_segment_type mpegts \
  -hls_segment_filename "$OUTPUT_DIR/stream_%v_%03d.ts" \
  -master_pl_name master.m3u8 \
  -var_stream_map "v:0,a:0,name:1080p v:1,a:1,name:720p v:2,a:2,name:360p" \
  "$OUTPUT_DIR/stream_%v.m3u8"

echo "生成完成：$OUTPUT_DIR/master.m3u8"
```

**使用**：
```bash
chmod +x gen_hls.sh
./gen_hls.sh input.mp4 ./output
```

---

## 三、生成的文件结构

```
output/
  ├── master.m3u8          ← 自适应码率入口（播放器加载此文件）
  ├── stream_1080p.m3u8    ← 1080p 播放列表
  ├── stream_720p.m3u8     ← 720p 播放列表
  ├── stream_360p.m3u8     ← 360p 播放列表
  ├── stream_1080p_000.ts
  ├── stream_1080p_001.ts
  ├── stream_720p_000.ts
  └── ...
```

**master.m3u8 内容**（FFmpeg 自动生成）：
```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=5192000,RESOLUTION=1920x1080,CODECS="avc1.640029,mp4a.40.2"
stream_1080p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2928000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
stream_720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=896000,RESOLUTION=640x360,CODECS="avc1.42c01e,mp4a.40.2"
stream_360p.m3u8
```

---

## 四、直播多码率转码

```bash
# 接收 RTMP 推流，实时转码成三档 HLS
ffmpeg -i rtmp://localhost/live/input \
  -filter_complex \
    "[0:v]split=2[v1][v2]; \
     [v1]scale=1280:720[v1out]; \
     [v2]scale=640:360[v2out]" \
  \
  -map "[v1out]" -c:v:0 libx264 -preset veryfast -b:v:0 2800k -tune zerolatency \
  -map "[v2out]" -c:v:1 libx264 -preset veryfast -b:v:1 800k  -tune zerolatency \
  -map 0:a -map 0:a \
  -c:a aac -b:a:0 128k -b:a:1 96k \
  \
  -f hls -hls_time 4 -hls_list_size 10 -hls_flags delete_segments \
  -hls_segment_filename /var/www/html/live/%v/seg%03d.ts \
  -master_pl_name /var/www/html/live/master.m3u8 \
  -var_stream_map "v:0,a:0,name:720p v:1,a:1,name:360p" \
  /var/www/html/live/%v/stream.m3u8
```

---

## 五、硬件加速转码

CPU 转码在多路场景消耗极大，使用 GPU 可降低 CPU 占用 80%：

```bash
# NVIDIA GPU（NVENC）
ffmpeg -hwaccel cuda -hwaccel_output_format cuda \
  -i input.mp4 \
  -vf "scale_cuda=1280:720" \
  -c:v h264_nvenc -preset p4 -b:v 2800k \
  -c:a aac -b:a 128k \
  -f flv rtmp://server/live/key

# Intel Quick Sync
ffmpeg -hwaccel qsv \
  -i input.mp4 \
  -c:v h264_qsv -b:v 2800k \
  -f flv rtmp://server/live/key

# Apple VideoToolbox（macOS）
ffmpeg -i input.mp4 \
  -c:v h264_videotoolbox -b:v 2800k \
  -f flv rtmp://server/live/key
```

**硬件加速性能对比**（1080p@5Mbps 转 720p@2.8Mbps）：

| 方式 | CPU 占用 | 速度 | 延迟增量 |
|------|---------|------|---------|
| libx264 软编 | ~80% | ~1.2x 实时 | 约 50ms |
| h264_nvenc | ~10% | ~30x 实时 | 约 20ms |
| h264_qsv | ~15% | ~20x 实时 | 约 15ms |

---

## 六、fMP4 格式 HLS（版本 7）

```bash
# 生成 CMAF 兼容的 fMP4 分片 HLS
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -hls_time 4 \
  -hls_segment_type fmp4 \
  -hls_fmp4_init_filename init.mp4 \
  -hls_segment_filename "seg%03d.m4s" \
  -hls_playlist_type vod \
  output.m3u8
```

fMP4 比 TS 节省约 10% 带宽，且与 DASH 分片格式兼容。

---

## 七、转码质量调优

```bash
# CRF 恒定质量模式（点播首选）
ffmpeg -i input.mp4 \
  -c:v libx264 -crf 23 \         # 0-51，越小质量越好
  -preset slow \                  # 慢预设=更好压缩
  output.mp4

# 两遍编码（最优质量）
ffmpeg -i input.mp4 -c:v libx264 -pass 1 -f null /dev/null
ffmpeg -i input.mp4 -c:v libx264 -pass 2 -b:v 2000k output.mp4
```

---

## 本篇小结

| 场景 | 方案 |
|------|------|
| 点播多码率 | `-var_stream_map` + `-master_pl_name` |
| 直播实时转码 | `-preset veryfast -tune zerolatency` |
| GPU 加速 | NVIDIA 用 `h264_nvenc`，Intel 用 `h264_qsv` |
| CMAF 兼容 | `-hls_segment_type fmp4` |

---

## 参考资料

1. [FFmpeg HLS Muxer 文档](https://ffmpeg.org/ffmpeg-formats.html#hls-1)
2. [FFmpeg H.264 编码指南](https://trac.ffmpeg.org/wiki/Encode/H.264)
3. [FFmpeg NVENC 硬件加速](https://trac.ffmpeg.org/wiki/HWAccelIntro)

---

## 动手实践

### 练习1：生成三码率 HLS 自适应流
```bash
ffmpeg -i input.mp4 \
  -filter_complex \
    "[0:v]split=3[v1][v2][v3]; \
     [v1]scale=1920:1080[v1out]; \
     [v2]scale=1280:720[v2out]; \
     [v3]scale=640:360[v3out]" \
  -map "[v1out]" -c:v:0 libx264 -b:v:0 4000k -preset fast \
  -map "[v2out]" -c:v:1 libx264 -b:v:1 1500k -preset fast \
  -map "[v3out]" -c:v:2 libx264 -b:v:2 500k  -preset fast \
  -map 0:a -c:a aac -b:a 128k \
  -f hls -hls_time 4 -hls_list_size 0 \
  -master_pl_name master.m3u8 \
  -hls_segment_filename "seg_%v_%04d.ts" \
  stream_%v.m3u8
```
**预期输出**：生成 `master.m3u8` 和三组流，`hls.js` 自适应播放器能根据带宽切换。

### 练习2：硬件加速转码（NVENC/VideoToolbox）
```bash
# NVIDIA GPU（Linux）
ffmpeg -hwaccel cuda -hwaccel_output_format cuda   -i input.mp4   -c:v h264_nvenc -preset p4 -b:v 2000k   -c:a copy output.mp4

# Apple Silicon（macOS）
ffmpeg -i input.mp4   -c:v h264_videotoolbox -b:v 2000k   -c:a copy output.mp4
```
**预期输出**：`top` 或 `nvidia-smi` 显示 GPU 负载上升，转码速度比软编快3-10倍。

### 练习3：实时转码推流（直播场景）
```bash
# RTSP 摄像头 → 720p30fps → RTMP 推流
ffmpeg -rtsp_transport tcp   -i "rtsp://camera/stream"   -c:v libx264 -preset veryfast -tune zerolatency   -vf "scale=1280:720"   -b:v 1500k -maxrate 1500k -bufsize 3000k   -g 30 -keyint_min 30   -c:a aac -b:a 128k   -f flv rtmp://localhost/live/transcoded
```
**预期输出**：实时转码推流，`ffprobe` 拉流验证分辨率为 1280x720，延迟 < 2s。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| 转码后音视频不同步 | GOP 设置与 `-hls_time` 不整除 | 确保 `-g`（GOP） = 帧率 × `hls_time` |
| NVENC 提示"no capable devices" | NVIDIA 驱动版本过低 | 更新驱动至 ≥ 470，安装 `cuda` 和 `nvidia-video-codec` |
| 多码率 HLS 只有最低分辨率 | `master.m3u8` bandwidth 值计算错误 | 检查各 variant 的 `BANDWIDTH` 是否真实反映码率 |
| CPU 占满导致丢帧 | 转码 preset 太慢 | 改用 `preset veryfast`，或切硬件编码 |
| 转码后视频绿屏/花屏 | hwaccel 格式不兼容，GPU 解码后未正确传至编码器 | 检查 `-hwaccel_output_format` 与编码器输入格式匹配 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| preset | Encoding Preset | 编码速度与压缩率的权衡（ultrafast→veryslow）|
| GOP | Group of Pictures | I 帧间隔，影响随机访问和延迟（直播推荐 = 帧率）|
| -tune zerolatency | Zero Latency Tune | x264 调优参数，减小编码缓冲，降低延迟 |
| -maxrate | Max Bitrate | CBR 模式的最大码率上限 |
| -bufsize | VBV Buffer Size | 码率控制缓冲区大小，通常设为 maxrate 的2倍 |
| NVENC | NVIDIA Encoder | NVIDIA GPU 硬件编码器，支持 H.264/H.265/AV1 |
| VideoToolbox | VideoToolbox | Apple 平台硬件编解码框架 |

---

> **全文完** | 上一篇：[第 17 篇 — FFmpeg 流媒体推拉流实战](17-FFmpeg流媒体推拉流实战.md) | 下一篇：[第 19 篇 — 流媒体延迟优化实战](19-流媒体延迟优化实战.md)
