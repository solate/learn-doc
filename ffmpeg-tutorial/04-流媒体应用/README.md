# 模块 4：流媒体应用

> 掌握 HLS/DASH 自适应流生成、RTMP/RTSP 实时推拉流、直播录制与 CDN 分发。

## 本模块目标

- 掌握 HLS 切片生成、加密与低延迟（LL-HLS）配置
- 学会生成 DASH 自适应流，理解 CMAF 与 HLS 共用切片
- 掌握 RTMP/RTSP 推拉流、断线重连与低延迟调优
- 实现 7x24 直播录制与分段存储策略
- 学会流媒体接入 CDN：签名鉴权、缓存策略、多 CDN 容灾

## 文章列表

16. **[HLS 直播推流](16-HLS直播推流.md)**  
    m3u8 + TS 分片原理、`-hls_time`/`-hls_flags` 调参、AES-128 加密、LL-HLS、fMP4 切片

17. **[DASH 流媒体](17-DASH流媒体.md)**  
    MPD 清单结构、多码率 ABR 梯队、CMAF 与 HLS 共用切片、与 HLS 的选型对比

18. **[RTMP/RTSP 实时流](18-RTMP-RTSP实时流.md)**  
    RTMP 推流与 RTSP 拉流实战、`-rtsp_transport tcp`、断线重连、超时参数、低延迟调优

19. **[直播录制与切片](19-直播录制与切片.md)**  
    `-f segment` 滚动录像、`-strftime` 按时间命名、7x24 录制策略、断点续录

20. **[流媒体 CDN 集成](20-流媒体CDN集成.md)**  
    URL 签名鉴权、Cache-Control 缓存策略、多 CDN 调度与故障切换、回源优化

## 前置知识

- 已完成模块 1（命令行语法、转封装）
- 已完成模块 2（编码参数、码率控制）—— ABR 梯队设计依赖第 11 篇
- 了解 HTTP 基础（Range 请求、缓存头）

---

[← 模块3：滤镜系统](../03-滤镜系统/README.md) | [返回总目录](../README.md) | [模块5：高级主题 →](../05-高级主题/README.md)
