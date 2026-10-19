# FFmpeg 流媒体推拉流实战（快速上手）

> **适合读者**：需要快速掌握 FFmpeg 流媒体命令的开发者  
> **篇幅**：精简版，聚焦实用命令

---

## 一、安装与验证

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
apt install ffmpeg

# 验证版本
ffmpeg -version | head -3

# 验证编解码器支持
ffmpeg -codecs 2>/dev/null | grep -E "libx264|libx265|libvpx|aac"
```

---

## 二、推流核心命令

### 2.1 文件推 RTMP

```bash
# 实时速率推流（-re 模拟实时）
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 2000k \
  -c:a aac -b:a 128k \
  -f flv rtmp://server/live/stream-key

# 循环推流（测试用）
ffmpeg -re -stream_loop -1 -i input.mp4 \
  -c copy -f flv rtmp://server/live/stream-key

# 直接复制，不转码（源已是 H264+AAC）
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://server/live/key
```

### 2.2 摄像头推流

```bash
# macOS 摄像头推 RTMP
ffmpeg -f avfoundation -i "0:0" \          # 0=摄像头, 0=麦克风
  -c:v libx264 -preset ultrafast -b:v 1500k \
  -c:a aac -b:a 128k \
  -f flv rtmp://server/live/camera

# Linux V4L2 摄像头
ffmpeg -f v4l2 -i /dev/video0 \
  -f alsa -i default \
  -c:v libx264 -preset ultrafast \
  -c:a aac \
  -f flv rtmp://server/live/camera
```

### 2.3 推 SRT 流

```bash
# SRT 推流（latency 单位微秒）
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast \
  -c:a aac \
  -f mpegts "srt://server:9000?latency=500000&streamid=live/main"
```

---

## 三、拉流核心命令

### 3.1 RTSP 拉流保存

```bash
# 拉取 RTSP 流，保存为 MP4
ffmpeg -i rtsp://admin:pass@192.168.1.100/stream \
  -c copy -f mp4 output.mp4

# 每 1 小时切一个文件
ffmpeg -i rtsp://camera/stream \
  -c copy \
  -f segment -segment_time 3600 \
  -strftime 1 "recording_%Y%m%d_%H%M%S.mp4"

# 拉 RTSP 推 RTMP（协议转换）
ffmpeg -i rtsp://camera/stream \
  -c copy -f flv rtmp://server/live/camera1
```

### 3.2 拉流转 HLS

```bash
# 拉 RTMP 流，切成 HLS
ffmpeg -i rtmp://server/live/stream \
  -c:v libx264 -preset veryfast \
  -c:a aac \
  -hls_time 4 \
  -hls_list_size 10 \
  -hls_flags delete_segments \
  /var/www/html/live/stream.m3u8
```

---

## 四、常用参数速查

### 4.1 视频参数

```bash
# 分辨率缩放
-vf scale=1280:720          # 固定分辨率
-vf scale=-2:720            # 保持宽高比，高度 720

# 帧率控制
-r 30                       # 输出 30fps

# 码率控制
-b:v 2000k                  # 目标码率 2Mbps
-maxrate 3000k -bufsize 4000k  # 最大码率 + 缓冲

# 编码器预设（速度 vs 质量）
-preset ultrafast   # 最快（质量最差）
-preset veryfast    # 推流常用
-preset fast        # 转码常用
-preset medium      # 默认
-preset slow        # 高质量

# H264 Profile
-profile:v baseline    # 低延迟兼容性最好
-profile:v main        # 通用
-profile:v high        # 高压缩率

# 强制关键帧（每 N 秒一个，HLS 切片必用）
-force_key_frames "expr:gte(t,n_forced*4)"  # 每 4 秒
```

### 4.2 音频参数

```bash
-c:a aac -b:a 128k         # AAC 128kbps（直播常用）
-c:a libopus -b:a 64k      # Opus 64kbps（WebRTC）
-ac 2                       # 立体声
-ar 44100                   # 采样率 44.1kHz
-af "volume=2.0"            # 音量调整
```

### 4.3 延迟优化参数

```bash
# 低延迟推流配置
-tune zerolatency           # 零延迟调优（x264）
-preset ultrafast           # 最快编码
-x264-params keyint=60:min-keyint=60  # 关键帧间隔

# 输入实时流，减少缓冲
-rtsp_transport tcp         # RTSP 用 TCP 传输
-rtbufsize 64M              # 实时缓冲大小
-thread_queue_size 512      # 线程队列大小
-fflags nobuffer            # 不缓冲（减少延迟）
-flags low_delay
```

---

## 五、调试技巧

```bash
# 查看流信息（不播放）
ffprobe -v error -show_streams -print_format json rtsp://camera/stream

# 静默推流（隐藏进度，脚本中用）
ffmpeg -loglevel error -i input.mp4 -c copy -f flv rtmp://server/key

# 显示推流统计（用于监控）
ffmpeg -re -i input.mp4 \
  -c copy -f flv rtmp://server/key \
  -progress pipe:1 2>/dev/null | \
  grep -E "speed|fps|bitrate"

# 测试连接（不推流，只探测）
ffprobe -v quiet -print_format json -show_format rtmp://server/live/test
```

---

## 六、Go 中调用 FFmpeg

```go
// 使用 os/exec 调用 FFmpeg
func startRTSPTrans(ctx context.Context, rtspURL, rtmpURL string) error {
    cmd := exec.CommandContext(ctx, "ffmpeg",
        "-loglevel", "warning",
        "-rtsp_transport", "tcp",
        "-i", rtspURL,
        "-c", "copy",
        "-f", "flv", rtmpURL,
    )

    cmd.Stderr = &logWriter{logger: log.With().Str("rtsp", rtspURL).Logger()}

    if err := cmd.Start(); err != nil {
        return xerr.Wrap(xerr.ErrInternal.Code, "启动 FFmpeg 失败", err)
    }

    go func() {
        if err := cmd.Wait(); err != nil {
            log.Error().Err(err).Str("rtsp", rtspURL).Msg("FFmpeg 进程退出")
        }
    }()

    return nil
}
```

---

## 本篇小结

| 场景 | 命令要点 |
|------|---------|
| 文件推 RTMP | `-re -c:v libx264 -preset veryfast -f flv` |
| RTSP 转 RTMP | `-i rtsp://... -c copy -f flv` |
| RTSP 转 HLS | `-hls_time 4 -hls_list_size 10 -hls_flags delete_segments` |
| 低延迟 | `-tune zerolatency -preset ultrafast -fflags nobuffer` |
| SRT 推流 | `-f mpegts "srt://server:9000?latency=500000"` |

---

## 参考资料

1. [FFmpeg 官方文档](https://ffmpeg.org/ffmpeg.html)
2. [FFmpeg Wiki - Streaming Guide](https://trac.ffmpeg.org/wiki/StreamingGuide)

---

## 动手实践

### 练习1：RTSP 拉流转推 RTMP（摄像头接入最常用场景）
```bash
# -c copy 零转码转发，不消耗 CPU
ffmpeg -rtsp_transport tcp   -i "rtsp://admin:pass@192.168.1.100:554/h264/ch1/main/av_stream"   -c copy   -f flv rtmp://localhost/live/cam1

# 带断线重连的生产脚本
while true; do
  ffmpeg -rtsp_transport tcp     -i "rtsp://admin:pass@192.168.1.100:554/..."     -c copy -f flv rtmp://localhost/live/cam1
  echo "断线，3s 后重连..." && sleep 3
done
```
**预期输出**：MediaMTX/SRS 收到推流，延迟约1-2s（TCP RTSP + RTMP转发开销）。

### 练习2：本地文件循环推流（测试/演示用）
```bash
# -re 按实际帧率读取（模拟实时流），-stream_loop -1 无限循环
ffmpeg -re -stream_loop -1   -i /path/to/test_video.mp4   -c:v libx264 -preset veryfast -tune zerolatency   -b:v 1500k -maxrate 1500k -bufsize 3000k   -c:a aac -b:a 128k   -f flv rtmp://localhost/live/test
```
**预期输出**：流媒体服务器持续收到视频流，直到 Ctrl+C 中断。

### 练习3：FFmpeg 拉取 HLS 流并保存为 MP4
```bash
# 从直播源录制30分钟
ffmpeg -i http://server/live/stream.m3u8   -c copy   -t 1800   output_$(date +%Y%m%d_%H%M%S).mp4

# 验证录制文件
ffprobe output_*.mp4 2>&1 | grep -E "Duration|Video|Audio"
```
**预期输出**：生成约1800s的MP4文件，`ffprobe` 显示正确时长和编码格式。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| `Connection refused` 无法推流 | 服务器未启动或端口错误 | `telnet localhost 1935` 验证端口可达 |
| 推流后播放有花屏 | `-c copy` 时起始关键帧缺失 | 等待第一个 I 帧或加 `-force_key_frames "expr:gte(t,0)"` |
| RTSP 连接超时 | 摄像头只支持 UDP，被防火墙拦截 | 改用 `-rtsp_transport tcp` |
| 推流频繁断开 | 网络不稳定 + 无重连逻辑 | 使用 `while true; do ffmpeg ...; sleep 3; done` 自动重连 |
| 视频帧率不稳定 | 缺少 `-re` 导致以最快速度读文件 | 推本地文件时必须加 `-re` |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| -re | Read at native frame rate | 按文件原始帧率读取，避免推流过快 |
| -c copy | Stream copy | 零转码模式，直接复制编码数据，不解码重编 |
| -stream_loop | Stream loop | 循环读取输入文件，`-1` 表示无限循环 |
| -rtsp_transport | RTSP Transport | 指定 RTSP 传输层协议（tcp/udp/udp_multicast）|
| -f flv | Force format FLV | 强制输出 FLV 容器格式（RTMP 推流必须）|
| lavf | libavformat | FFmpeg 的格式层库，负责封装/解封装 |

---

> **全文完** | 上一篇：[第 16.5 篇 — ZLMediaKit 实战（安防国标接入首选）](16.5-ZLMediaKit实战.md) | 下一篇：[第 18 篇 — FFmpeg 流媒体转码配置（多码率 HLS）](18-FFmpeg流媒体转码配置.md)
