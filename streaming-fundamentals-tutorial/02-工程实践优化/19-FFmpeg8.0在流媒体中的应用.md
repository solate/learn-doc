# FFmpeg 8.0 在流媒体中的应用

> **适合读者**：已有 FFmpeg 基础，想了解 8.0 新特性的开发者  
> **篇幅**：精简版，聚焦新变化

---

## 一、FFmpeg 8.0 主要更新

FFmpeg 8.0（2025年发布）带来几项对流媒体影响较大的更新：

| 更新点 | 影响 |
|--------|------|
| **AV1 编码性能提升 50%** | libaom/SVT-AV1 大幅优化，云端转码成本降低 |
| **WHIP 协议支持** | 原生 WebRTC 推流，无需第三方信令 |
| **AVFrame 线程模型重写** | 多路并发转码内存占用降低 |
| **VVC（H.266）解码支持** | 新一代视频编解码标准（编码仍在开发） |
| **ffmpeg-python API 改进** | 更易集成到 Python/Go 管理脚本 |

---

## 二、AV1 编码实战

FFmpeg 8.0 中 AV1 编码已可用于生产：

```bash
# SVT-AV1 编码（速度最快，适合直播转码）
ffmpeg -i input.mp4 \
  -c:v libsvtav1 -preset 8 -crf 35 \    # preset 越大越快，质量越低
  -c:a libopus -b:a 96k \
  -f hls -hls_time 4 \
  output.m3u8

# libaom-av1（质量最好，适合点播）
ffmpeg -i input.mp4 \
  -c:v libaom-av1 -cpu-used 4 -crf 30 \
  -c:a libopus \
  output.webm
```

**AV1 vs H.264 vs H.265 对比**（相同画质）：

| 编解码器 | 相对码率 | 编码速度 | 解码硬件支持 |
|---------|---------|---------|------------|
| H.264 | 100% | 极快 | 全设备 |
| H.265 | ~50% | 较快 | 2019+设备 |
| AV1 | ~35% | 较慢（软编）| 2021+设备 |

**何时用 AV1**：
- 点播 VOD（有时间慢慢转码）
- 面向 Chrome/Firefox/Android 用户
- 需要节省 CDN 带宽成本（比 H.264 省 35-50%）

---

## 三、WHIP 原生推流

FFmpeg 8.0 支持原生 WHIP 推流，无需 pion/webrtc：

```bash
# 直接推流到支持 WHIP 的服务器（如 MediaMTX、Cloudflare）
ffmpeg -re -i input.mp4 \
  -c:v libvpx-vp8 -b:v 2000k \
  -c:a libopus -b:a 128k \
  -f whip "https://server:8889/live/stream/whip"

# 带鉴权
ffmpeg -re -i input.mp4 \
  -c:v libvpx-vp8 \
  -c:a libopus \
  -headers "Authorization: Bearer <token>" \
  -f whip "https://server/whip/stream-key"
```

**注意**：WHIP 推流必须用 VP8/Opus 或 H264/Opus（WebRTC 限制）。

---

## 四、多路并发转码优化

FFmpeg 8.0 改进了并发场景的内存管理：

```bash
# 启动多个 FFmpeg 进程的最佳实践
# （每路独立进程，避免一个崩溃影响其他）

for camera in camera1 camera2 camera3; do
  ffmpeg \
    -rtsp_transport tcp \
    -i "rtsp://admin:pass@$camera/stream" \
    -c:v libx264 -preset veryfast -b:v 1500k \
    -c:a aac -b:a 128k \
    -f hls -hls_time 4 -hls_list_size 10 \
    -hls_flags delete_segments \
    "/var/www/html/$camera/stream.m3u8" \
    2>"/var/log/ffmpeg/$camera.log" &
done
```

---

## 五、Go 中管理 FFmpeg 进程池

```go
// FFmpeg 进程管理器
type FFmpegManager struct {
    mu       sync.RWMutex
    procs    map[string]*exec.Cmd
}

func (m *FFmpegManager) StartTranscode(ctx context.Context, id, inputURL, outputDir string) error {
    m.mu.Lock()
    defer m.mu.Unlock()

    if _, exists := m.procs[id]; exists {
        return nil // 已在运行
    }

    cmd := exec.CommandContext(ctx, "ffmpeg",
        "-loglevel", "warning",
        "-rtsp_transport", "tcp",
        "-i", inputURL,
        "-c:v", "libx264", "-preset", "veryfast", "-b:v", "1500k",
        "-c:a", "aac", "-b:a", "128k",
        "-f", "hls",
        "-hls_time", "4",
        "-hls_list_size", "10",
        "-hls_flags", "delete_segments",
        filepath.Join(outputDir, "stream.m3u8"),
    )

    if err := cmd.Start(); err != nil {
        return xerr.Wrap(xerr.ErrInternal.Code, "启动转码失败", err)
    }

    m.procs[id] = cmd
    log.Info().Str("id", id).Int("pid", cmd.Process.Pid).Msg("转码进程启动")

    // 异步监控进程退出
    go func() {
        err := cmd.Wait()
        m.mu.Lock()
        delete(m.procs, id)
        m.mu.Unlock()
        if err != nil {
            log.Error().Err(err).Str("id", id).Msg("转码进程异常退出")
        }
    }()

    return nil
}

func (m *FFmpegManager) StopTranscode(id string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    if cmd, ok := m.procs[id]; ok {
        cmd.Process.Signal(syscall.SIGTERM)
    }
}
```

---

## 六、版本选择建议

| 版本 | 状态 | 建议 |
|------|------|------|
| FFmpeg 7.x | 稳定 | 生产环境首选 |
| FFmpeg 8.0 | 最新 | 新项目可用，需测试 |
| FFmpeg 6.x | 老稳定 | 可升级 |
| FFmpeg 4.x | 过时 | 建议升级 |

```bash
# 检查版本和编译特性
ffmpeg -version | head -5
ffmpeg -buildconf 2>&1 | grep -E "av1|hevc|nvenc|qsv"
```

---

## 本篇小结

| 新特性 | 使用建议 |
|--------|---------|
| SVT-AV1 | 点播节省带宽，用 `-preset 8 -crf 35` |
| WHIP 原生支持 | 替代复杂信令，推流到 MediaMTX/Cloudflare |
| 多路优化 | 每路独立进程 + Go 进程池管理 |

---

## 参考资料

1. [FFmpeg 8.0 发布说明](https://ffmpeg.org/index.html#news)
2. [SVT-AV1 编码参数指南](https://gitlab.com/AOMediaCodec/SVT-AV1/-/blob/master/Docs/Parameters.md)
3. [AV1 浏览器支持情况](https://caniuse.com/av1)

---

## 动手实践

### 练习1：FFmpeg 8.0 SVT-AV1 编码测试
```bash
# 验证 FFmpeg 版本及 AV1 编码器支持
ffmpeg -version | head -2
ffmpeg -encoders | grep -E "av1|svt"

# 用 SVT-AV1 编码，preset 8 = 速度优先（直播场景）
ffmpeg -i input.mp4 \
  -c:v libsvtav1 -preset 8 -crf 35 \
  -c:a libopus -b:a 96k \
  output_av1.mp4

# 对比文件大小与质量
ffprobe output_av1.mp4 2>&1 | grep -E "Video|Duration"
ls -lh input.mp4 output_av1.mp4
```
**预期输出**：AV1 文件比 H.264 小40-60%，Chrome/Firefox 直接播放。

### 练习2：FFmpeg 8.0 WHIP 直接推流（新增 -f whip）
```bash
# FFmpeg 7.0+ 原生支持 WHIP 输出
ffmpeg -re -stream_loop -1 -i test.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -c:a libopus -b:a 64k \
  -f whip http://localhost:8889/live/stream/whip
```
**预期输出**：MediaMTX 日志显示 WHIP 会话建立，WebRTC 播放延迟 < 500ms，无需中间 RTMP 跳转。

### 练习3：使用新版 hstack/vstack 滤镜合并多路流
```bash
# 将2路摄像头流合并为左右拼接画面
ffmpeg \
  -i rtsp://cam1/stream \
  -i rtsp://cam2/stream \
  -filter_complex "[0:v][1:v]hstack=inputs=2[out]" \
  -map "[out]" -map 0:a \
  -c:v libx264 -preset veryfast \
  -f flv rtmp://localhost/live/merged
```
**预期输出**：输出1路左右拼屏画面，分辨率是单路的2倍宽。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| `libsvtav1 not found` | FFmpeg 编译时未包含 SVT-AV1 | 使用 `brew install ffmpeg` 或下载官方 static build |
| AV1 视频浏览器无法播放 | Safari 不支持 AV1（macOS 13以前）| 检查 `caniuse.com/av1`，按需提供 H.264 备选流 |
| WHIP 推流连接被拒 | MediaMTX 版本过低（需 1.x+）| 升级 MediaMTX 至 1.0 以上 |
| SVT-AV1 CPU 占用100% | preset 过小导致超实时编码 | 直播用 `preset 8`，点播用 `preset 4-6` |
| 多路滤镜导致音视频不同步 | 各路输入帧率不一致 | 加 `-fps_mode cfr` 统一帧率 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| SVT-AV1 | Scalable Video Technology for AV1 | Intel/Netflix 开源高性能 AV1 编码器 |
| AV1 | AOMedia Video 1 | 开源免专利视频编码标准，压缩率优于 H.265 |
| libdav1d | dav1d Decoder | VideoLAN 开发的高性能 AV1 解码库 |
| -f whip | WHIP output format | FFmpeg 7+ 原生 WHIP 输出格式 |
| hstack | Horizontal Stack | FFmpeg 滤镜，将多路视频水平拼接 |
| CRF | Constant Rate Factor | 恒定质量编码，数值越低质量越高（AV1 推荐28-35）|

---

> **全文完** | 上一篇：[第 18 篇 — FFmpeg 流媒体转码配置](18-FFmpeg流媒体转码配置.md) | 下一篇：[第 20 篇 — 多分屏并发性能优化](20-多分屏并发性能优化.md)
