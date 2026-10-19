# SRS 部署与配置实战

> **适合读者**：后端开发者、运维工程师  
> **前置知识**：[第 14 篇：流媒体服务器选型对比](14-流媒体服务器选型对比.md)

---

## 一、SRS 安装

### 1.1 Docker 方式（推荐）

```bash
# 启动 SRS 5.0（最新稳定版）
docker run -d \
  --name srs \
  --restart unless-stopped \
  -p 1935:1935 \    # RTMP
  -p 8080:8080 \    # HTTP（HLS/HTTP-FLV）
  -p 1985:1985 \    # HTTP API
  -p 8000:8000/udp \ # WebRTC
  -v $(pwd)/conf/srs.conf:/usr/local/srs/conf/srs.conf \
  ossrs/srs:5

# 验证运行状态
curl http://localhost:1985/api/v1/versions
```

### 1.2 从源码编译

```bash
git clone -b 5.0release https://gitee.com/ossrs/srs.git
cd srs/trunk
./configure
make -j4
./objs/srs -c conf/srs.conf
```

---

## 二、基础配置详解

### 2.1 最小配置：RTMP 推流 + HLS 播放

```nginx
# conf/srs.conf
listen              1935;
max_connections     1000;
daemon              off;    # Docker 中不要 daemon
pid                 ./objs/srs.pid;
srs_log_tank        console;

http_server {
    enabled     on;
    listen      8080;
    dir         ./objs/nginx/html;
}

vhost __defaultVhost__ {
    # HLS 分发
    hls {
        enabled         on;
        hls_path        ./objs/nginx/html;
        hls_fragment    4;      # 4 秒分片
        hls_window      60;     # 保留 60 秒
        hls_cleanup     on;     # 自动删除旧分片
    }
}
```

**推流测试**：
```bash
# OBS 或 FFmpeg 推流
ffmpeg -re -i test.mp4 \
  -c:v libx264 -preset veryfast -b:v 1500k \
  -c:a aac -b:a 128k \
  -f flv rtmp://localhost/live/test

# 播放
# RTMP：rtmp://localhost/live/test
# HLS：http://localhost:8080/live/test.m3u8
```

### 2.2 完整生产配置

```nginx
listen              1935;
max_connections     3000;
daemon              off;
pid                 ./objs/srs.pid;
srs_log_tank        console;
srs_log_level       warn;      # 生产环境用 warn

# HTTP API（供后端调用）
http_api {
    enabled     on;
    listen      1985;
    # 生产环境加 HTTPS 和 auth
}

# HTTP 服务（HLS/HTTP-FLV 分发）
http_server {
    enabled     on;
    listen      8080;
    dir         ./objs/nginx/html;
}

# WebRTC 信令
rtc_server {
    enabled     on;
    listen      8000;      # UDP 端口
    candidate   1.2.3.4;  # 公网 IP（必须设置！）
}

vhost __defaultVhost__ {
    # 推流鉴权
    http_hooks {
        enabled     on;
        on_publish  http://backend:8081/api/v1/streams/on_publish;
        on_unpublish http://backend:8081/api/v1/streams/on_unpublish;
    }

    # HLS
    hls {
        enabled         on;
        hls_path        ./objs/nginx/html;
        hls_fragment    4;
        hls_window      60;
        hls_on_error    continue;  # 出错继续，不断流
    }

    # HTTP-FLV（低延迟约 1-3s）
    http_remux {
        enabled     on;
        mount       [vhost]/[app]/[stream].flv;
        fast_cache  30;    # 30 秒快速缓存，减少延迟
    }

    # WebRTC（超低延迟 <500ms）
    rtc {
        enabled         on;
        rtmp_to_rtc     on;    # RTMP 推流 → WebRTC 播放
        rtc_to_rtmp     on;    # WebRTC 推流 → RTMP 播放
    }
}
```

---

## 三、HTTP Hooks 推流鉴权

### 3.1 Go 后端实现鉴权回调

```go
// handler/stream_hook_handler.go

type OnPublishRequest struct {
    Action    string `json:"action"`     // "on_publish"
    ClientID  string `json:"client_id"`
    IP        string `json:"ip"`
    Vhost     string `json:"vhost"`
    App       string `json:"app"`        // "live"
    Stream    string `json:"stream"`     // 推流 key
    Param     string `json:"param"`      // URL 查询参数，如 ?token=xxx
}

// POST /api/v1/streams/on_publish
// @Summary SRS 推流鉴权回调
// @Success 200 {object} response.Response{data=dto.StreamAuthResponse}
func (h *StreamHookHandler) OnPublish(c *gin.Context) {
    var req OnPublishRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        // SRS 要求：code != 0 则拒绝推流
        c.JSON(http.StatusOK, gin.H{"code": 400, "data": "参数错误"})
        return
    }

    // 解析 token（从 URL 参数中获取）
    // 推流地址：rtmp://server/live/stream?token=xxx
    params, _ := url.ParseQuery(req.Param)
    token := params.Get("token")

    if err := h.streamSvc.AuthPublish(c.Request.Context(), req.Stream, token); err != nil {
        log.Warn().Str("stream", req.Stream).Str("ip", req.IP).Msg("推流鉴权失败")
        c.JSON(http.StatusOK, gin.H{"code": 403, "data": "无推流权限"})
        return
    }

    // code=0 表示允许推流
    c.JSON(http.StatusOK, gin.H{"code": 0, "data": ""})
}

// POST /api/v1/streams/on_unpublish
func (h *StreamHookHandler) OnUnpublish(c *gin.Context) {
    var req OnPublishRequest
    c.ShouldBindJSON(&req)
    h.streamSvc.HandleUnpublish(c.Request.Context(), req.Stream)
    c.JSON(http.StatusOK, gin.H{"code": 0})
}
```

---

## 四、SRS HTTP API 使用

### 4.1 查询在线流列表

```bash
# 查询所有在线流
curl http://localhost:1985/api/v1/streams

# 查询单个 vhost 的流
curl http://localhost:1985/api/v1/streams?vhost=__defaultVhost__
```

**Go 代码集成**：
```go
type SRSStream struct {
    ID        string `json:"id"`
    Name      string `json:"name"`
    App       string `json:"app"`
    Clients   int    `json:"clients"`   // 当前观看人数
    Frames    int64  `json:"frames"`
    SentBytes int64  `json:"send_bytes"`
    RecvBytes int64  `json:"recv_bytes"`
    Kbps      struct {
        Recv30s int `json:"recv_30s"` // 最近 30s 收流码率(kbps)
        Send30s int `json:"send_30s"`
    } `json:"kbps"`
}

func (s *StreamService) GetOnlineStreams(ctx context.Context) ([]SRSStream, error) {
    resp, err := http.Get("http://srs:1985/api/v1/streams")
    if err != nil {
        return nil, xerr.Wrap(xerr.ErrInternal.Code, "查询流列表失败", err)
    }
    defer resp.Body.Close()

    var result struct {
        Code    int         `json:"code"`
        Streams []SRSStream `json:"streams"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.Streams, nil
}
```

### 4.2 踢掉客户端

```bash
# 断开指定客户端（停止推流）
curl -X POST http://localhost:1985/api/v1/clients/your-client-id \
  -H "Content-Type: application/json" \
  -d '{"action": "kickoff"}'
```

---

## 五、SRS 集群部署

### 5.1 Edge-Origin 集群

```
多个 Edge 节点从 Origin 按需拉流，分散播放压力：

Origin 节点（1台）：
  - 接收所有推流
  - 不对外提供播放（减少带宽）

Edge 节点（N台）：
  - 只负责播放分发
  - 按需从 Origin 拉流
  - 可无限扩展
```

**Origin 配置**：
```nginx
# origin.conf
listen  1935;
vhost __defaultVhost__ {
    # Origin 不需要太多配置
    cluster {
        mode    local;    # 本地模式（不是 edge）
    }
}
```

**Edge 配置**：
```nginx
# edge.conf
listen  1935;
vhost __defaultVhost__ {
    cluster {
        mode        remote;            # Edge 模式
        origin      origin-server:1935 # Origin 地址
        token_traverse  on;            # 透传 token
    }
    http_remux {
        enabled on;
        mount   [vhost]/[app]/[stream].flv;
    }
    hls {
        enabled on;
        hls_path ./objs/html;
    }
}
```

### 5.2 Docker Compose 集群示例

```yaml
version: '3'
services:
  srs-origin:
    image: ossrs/srs:5
    ports:
      - "1935:1935"  # 推流端口
    volumes:
      - ./origin.conf:/usr/local/srs/conf/srs.conf

  srs-edge-1:
    image: ossrs/srs:5
    ports:
      - "8081:8080"  # 华北播放
    volumes:
      - ./edge.conf:/usr/local/srs/conf/srs.conf

  srs-edge-2:
    image: ossrs/srs:5
    ports:
      - "8082:8080"  # 华南播放
    volumes:
      - ./edge.conf:/usr/local/srs/conf/srs.conf
```

---

## 六、常见问题排查

### 问题 1：WebRTC 播放没有画面

**原因**：`candidate` 未配置公网 IP。

```nginx
rtc_server {
    enabled on;
    listen  8000;
    candidate 1.2.3.4;  # ← 必须填公网 IP，不能用 127.0.0.1
}
```

### 问题 2：HLS 播放延迟过高

**优化**：减小 `hls_fragment`，增加 Edge 节点：
```nginx
hls {
    hls_fragment    2;  # 2 秒分片（默认 10s）
    hls_window      12; # 保留 6 个分片
}
```

### 问题 3：大量推流时 CPU 飙升

**排查**：SRS 本身 CPU 消耗极低，飙升通常是 FFmpeg 转码。

```bash
# 检查是否有转码进程
ps aux | grep ffmpeg

# SRS 纯转发不转码
# 如果需要转码，改用硬件加速
ffmpeg -hwaccel cuda -i input.mp4 ...
```

---

## 七、本篇小结

| 功能 | 配置要点 |
|------|---------|
| RTMP 接收 | `listen 1935`，默认开启 |
| HLS 分发 | `hls { enabled on; hls_fragment 4; }` |
| HTTP-FLV | `http_remux { enabled on; }` |
| WebRTC | `rtc { enabled on; }` + 设置 `candidate` 公网 IP |
| 推流鉴权 | `http_hooks { on_publish ... }` 回调 Go 后端 |
| 集群 | Edge 节点 `cluster { mode remote; origin ... }` |

---

## 下一篇

下一篇讲解 **MediaMTX 实战**：Go 项目最佳搭档，零配置接入摄像头 RTSP 流并转换为 HLS/WebRTC，以及 API 动态管理流路径。

---

## 参考资料

1. [SRS 官方中文文档](https://ossrs.net/lts/zh-cn/docs/v6/doc/getting-started)
2. [SRS GitHub](https://github.com/ossrs/srs)
3. [SRS Docker 部署](https://ossrs.net/lts/zh-cn/docs/v6/doc/getting-started-docker)

---

## 动手实践

### 练习1：SRS Docker 一键部署并验证推拉流
```bash
# 启动 SRS（开启 HLS + HTTP-FLV + 控制台）
docker run -d --name srs   -p 1935:1935 -p 1985:1985 -p 8080:8080   ossrs/srs:4 ./objs/srs -c conf/docker.conf

# 推流测试
ffmpeg -re -i /path/to/test.mp4 -c copy -f flv rtmp://localhost/live/test

# 验证 HLS
curl -I http://localhost:8080/live/test.m3u8

# 验证 HTTP-FLV
ffplay http://localhost:8080/live/test.flv
```
**预期输出**：SRS Console（`http://localhost:8080`）中看到活跃流，HLS 200 OK，FLV 实时播放。

### 练习2：配置自定义 SRS vhost + 录制
```nginx
# /etc/srs/srs.conf
vhost __defaultVhost__ {
    hls {
        enabled on;
        hls_path ./objs/nginx/html;
        hls_fragment 4;
        hls_window 60;
    }
    dvr {
        enabled on;
        dvr_path ./objs/nginx/html/[app]/[stream]/[2006][01][02]-[15][04][05].flv;
        dvr_plan segment;
        dvr_duration 300;  # 每5分钟一个文件
    }
}
```
**预期输出**：推流后，每5分钟在 `dvr_path` 生成一个 FLV 录像文件。

### 练习3：SRS 集群（Origin + Edge）
```nginx
# Edge 配置（从 Origin 回源）
vhost __defaultVhost__ {
    cluster {
        mode remote;
        origin 192.168.1.100:1935;
    }
}
```
**预期输出**：客户端连接 Edge 拉流时，Edge 自动从 Origin 回源，Origin 只需处理一路拉流压力。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| SRS API `/api/v1/streams` 返回空 | 没有活跃推流 | 确认推流 URL 正确，检查 SRS 日志 `objs/srs.log` |
| HLS m3u8 有内容但 .ts 404 | hls_path 权限问题 | `chmod 755 ./objs/nginx/html` |
| 录像文件大小为0 | DVR 目录不可写 | 检查 dvr_path 目录是否存在且可写 |
| Edge 无法回源 | Origin IP/Port 配置错误 | `telnet <origin_ip> 1935` 确认连通性 |
| 内存持续增长 | HLS 窗口设置过大 | 减小 `hls_window`（推荐 30-60s）|

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| DVR | Digital Video Recording | SRS 内置的流媒体录制功能 |
| vhost | Virtual Host | SRS 虚拟主机，隔离不同业务配置 |
| HTTP-FLV | HTTP-FLV | 通过 HTTP 长连接传输 FLV 数据，低延迟约2s |
| 回源 | Origin Pull | Edge 节点向 Origin 拉取媒体流的行为 |
| SRS Console | SRS Console | SRS 内置 Web 管理界面，端口 8080 |
| hls_window | HLS Window | m3u8 中保留的总时长（秒），控制直播窗口大小 |

---

> **全文完** | 上一篇：[第 14 篇 — 流媒体服务器选型对比](14-流媒体服务器选型对比.md) | 下一篇：[第 16 篇 — MediaMTX 实战（Go 项目首选）](16-MediaMTX实战.md)
