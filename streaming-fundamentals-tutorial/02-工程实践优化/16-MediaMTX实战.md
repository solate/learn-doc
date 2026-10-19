# MediaMTX 实战（Go 项目首选）

> **适合读者**：Go 后端开发者、监控系统工程师  
> **前置知识**：[第 14 篇：流媒体服务器选型对比](14-流媒体服务器选型对比.md)

---

## 一、MediaMTX 核心优势回顾

MediaMTX（原名 rtsp-simple-server）是一个用 Go 编写的全协议流媒体服务器：

```
任意协议推流 → MediaMTX → 任意协议播放

支持：RTSP、RTMP、HLS、DASH、WebRTC、SRT、WHIP、WHEP
```

**为什么 Go 项目首选**：
- 纯 Go 实现，可直接嵌入 Go 进程或以子进程运行
- 完整的 HTTP API，动态管理流路径
- 单二进制，无依赖，Docker 镜像极小（~15MB）
- 零配置启动，一个 YAML 搞定所有功能

---

## 二、快速启动

### 2.1 Docker 启动

```bash
docker run -d \
  --name mediamtx \
  --restart unless-stopped \
  -p 8554:8554 \      # RTSP
  -p 1935:1935 \      # RTMP
  -p 8888:8888 \      # HLS/DASH
  -p 8889:8889 \      # WebRTC/WHIP/WHEP
  -p 8890:8890/udp \  # SRT
  -p 9997:9997 \      # HTTP API
  -v $(pwd)/mediamtx.yml:/mediamtx.yml \
  bluenviron/mediamtx:latest

# 测试推流（无需任何配置）
ffmpeg -re -i test.mp4 -c copy -f rtsp rtsp://localhost:8554/live
# 播放：vlc rtsp://localhost:8554/live
```

### 2.2 二进制启动

```bash
# 下载最新版本
wget https://github.com/bluenviron/mediamtx/releases/latest/download/mediamtx_linux_amd64.tar.gz
tar -xzf mediamtx_linux_amd64.tar.gz

# 直接运行（使用内置默认配置）
./mediamtx

# 或指定配置文件
./mediamtx mediamtx.yml
```

---

## 三、配置详解

### 3.1 完整配置文件

```yaml
# mediamtx.yml

# ===== 全局设置 =====
logLevel: info          # debug/info/warn/error
logDestinations: [stdout]

# ===== 协议端口 =====
rtspAddress: :8554
rtmpAddress: :1935
hlsAddress: :8888
webrtcAddress: :8889
srtAddress: :8890

# ===== API =====
api: yes
apiAddress: :9997

# ===== 认证 =====
# 推流鉴权（回调到 Go 后端）
authMethod: http
authHTTPAddress: http://backend:8081/api/v1/streams/auth

# ===== 路径配置 =====
paths:
  # 默认路径：接受任何路径的推流
  all:
    source: publisher  # publisher = 等待推流

  # 固定摄像头源（拉取 RTSP 流）
  camera/entrance:
    source: rtsp://admin:password@192.168.1.100/stream1
    sourceOnDemand: true    # 有人看才建立连接
    sourceOnDemandCloseAfter: 10s  # 无人看 10s 后断开

  # 带录制的直播流
  live/main:
    source: publisher
    # 录制到本地文件
    record: yes
    recordPath: /recordings/%Y/%m/%d/%H%M%S.mp4
    recordSegmentDuration: 1h   # 每小时一个文件
    recordDeleteAfter: 72h      # 保留 72 小时

  # 推流时触发回调
  live/events:
    source: publisher
    runOnPublish: "curl -X POST http://backend:8081/api/v1/streams/on_start -d '{\"stream\":\"$MTX_PATH\"}'"
    runOnPublishRestart: yes
    runOnUnpublish: "curl -X POST http://backend:8081/api/v1/streams/on_stop -d '{\"stream\":\"$MTX_PATH\"}'"
```

### 3.2 鉴权配置

```yaml
# 方式 1：固定用户名密码
authMethod: internal
authInternalUsers:
  - user: admin
    pass: secret
    ips: []           # 空表示不限 IP
    permissions:
      - action: publish
        path: live/*
      - action: read
        path: live/*

# 方式 2：HTTP 回调（推荐，与 Go 后端集成）
authMethod: http
authHTTPAddress: http://backend:8081/api/v1/streams/auth
```

**Go 后端鉴权接口**：
```go
type MediaMTXAuthRequest struct {
    User   string `json:"user"`     // 可能为空
    Pass   string `json:"pass"`     // 可能为空
    IP     string `json:"ip"`
    Action string `json:"action"`   // "publish" 或 "read"
    Path   string `json:"path"`     // 流路径
    Query  string `json:"query"`    // URL 查询参数（token=xxx）
}

// POST /api/v1/streams/auth
func (h *StreamHandler) MediaMTXAuth(c *gin.Context) {
    var req MediaMTXAuthRequest
    c.ShouldBindJSON(&req)

    // 从 query 参数中取 token
    params, _ := url.ParseQuery(req.Query)
    token := params.Get("token")

    if err := h.streamSvc.Auth(c.Request.Context(), req.Path, req.Action, token); err != nil {
        // 返回 4xx 表示拒绝
        c.JSON(http.StatusForbidden, gin.H{"error": "unauthorized"})
        return
    }
    // 返回 2xx 表示允许
    c.Status(http.StatusOK)
}
```

---

## 四、HTTP API 动态管理

MediaMTX 提供完整的 REST API，可动态添加/删除流路径。

### 4.1 API 操作示例

```bash
# 查询所有路径
curl http://localhost:9997/v3/paths/list

# 查询特定路径
curl http://localhost:9997/v3/paths/get/live/camera1

# 动态添加路径（不重启服务）
curl -X POST http://localhost:9997/v3/config/paths/add/camera/new \
  -H "Content-Type: application/json" \
  -d '{
    "source": "rtsp://admin:pass@192.168.1.200/stream",
    "sourceOnDemand": true
  }'

# 删除路径
curl -X DELETE http://localhost:9997/v3/config/paths/delete/camera/new

# 踢掉推流客户端
curl -X POST http://localhost:9997/v3/rtmpconns/kickall/live/stream
```

### 4.2 Go SDK 封装

```go
// pkg/mediamtx/client.go

type Client struct {
    baseURL    string
    httpClient *http.Client
}

func NewClient(baseURL string) *Client {
    return &Client{
        baseURL:    baseURL,
        httpClient: &http.Client{Timeout: 5 * time.Second},
    }
}

type PathConfig struct {
    Source             string `json:"source"`
    SourceOnDemand     bool   `json:"sourceOnDemand"`
    SourceOnDemandCloseAfter string `json:"sourceOnDemandCloseAfter,omitempty"`
    Record             bool   `json:"record,omitempty"`
    RecordPath         string `json:"recordPath,omitempty"`
}

// AddPath 动态添加摄像头路径
func (c *Client) AddPath(ctx context.Context, pathName string, cfg PathConfig) error {
    body, _ := json.Marshal(cfg)
    req, _ := http.NewRequestWithContext(ctx,
        http.MethodPost,
        c.baseURL+"/v3/config/paths/add/"+pathName,
        bytes.NewReader(body),
    )
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return xerr.Wrap(xerr.ErrInternal.Code, "添加流路径失败", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return xerr.New(xerr.ErrInternal.Code, "添加流路径失败")
    }
    return nil
}

// RemovePath 删除流路径（设备下线时调用）
func (c *Client) RemovePath(ctx context.Context, pathName string) error {
    req, _ := http.NewRequestWithContext(ctx,
        http.MethodDelete,
        c.baseURL+"/v3/config/paths/delete/"+pathName,
        nil,
    )
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return xerr.Wrap(xerr.ErrInternal.Code, "删除流路径失败", err)
    }
    defer resp.Body.Close()
    return nil
}

type PathInfo struct {
    Name    string `json:"name"`
    Ready   bool   `json:"ready"`   // 是否有推流
    Readers int    `json:"readers"` // 观看人数
}

// ListPaths 查询所有路径状态
func (c *Client) ListPaths(ctx context.Context) ([]PathInfo, error) {
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, c.baseURL+"/v3/paths/list", nil)
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, xerr.Wrap(xerr.ErrInternal.Code, "查询路径列表失败", err)
    }
    defer resp.Body.Close()

    var result struct {
        Items []PathInfo `json:"items"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.Items, nil
}
```

---

## 五、典型应用场景

### 5.1 场景：摄像头管理系统

```go
// service/camera_service.go

type CameraService struct {
    mediamtx   *mediamtx.Client
    cameraRepo repository.CameraRepo
}

// 上线摄像头：在 MediaMTX 注册 RTSP 源
func (s *CameraService) BringOnline(ctx context.Context, cameraID string) error {
    camera, err := s.cameraRepo.GetByID(ctx, cameraID)
    if err != nil {
        return err
    }

    pathName := "cameras/" + cameraID
    return s.mediamtx.AddPath(ctx, pathName, mediamtx.PathConfig{
        Source:                   camera.RTSPURL,
        SourceOnDemand:           true,
        SourceOnDemandCloseAfter: "30s",
    })
}

// 获取播放地址
func (s *CameraService) GetPlayURLs(ctx context.Context, cameraID string) (*dto.PlayURLs, error) {
    pathName := "cameras/" + cameraID
    baseURL := "https://streaming.example.com"

    return &dto.PlayURLs{
        HLS:      baseURL + ":8888/" + pathName + "/index.m3u8",
        WebRTC:   baseURL + ":8889/" + pathName,
        RTSP:     "rtsp://" + baseURL + ":8554/" + pathName,
    }, nil
}
```

### 5.2 场景：直播转录制

```yaml
# mediamtx.yml
paths:
  live/+:                     # + 匹配任意单级路径
    source: publisher
    record: yes
    recordPath: /recordings/$MTX_PATH/%Y%m%d_%H%M%S.mp4
    recordSegmentDuration: 30m
    runOnReady: curl -X POST http://backend/api/v1/live/start -d '{"path":"$MTX_PATH"}'
    runOnNotReady: curl -X POST http://backend/api/v1/live/stop -d '{"path":"$MTX_PATH"}'
```

---

## 六、Docker Compose 集成

```yaml
version: '3'
services:
  backend:
    build: .
    ports:
      - "8081:8081"

  mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8554:8554"
      - "1935:1935"
      - "8888:8888"
      - "8889:8889"
      - "9997:9997"
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - ./recordings:/recordings
    depends_on:
      - backend
    restart: unless-stopped
```

---

## 七、本篇小结

| 功能 | 实现方式 |
|------|---------|
| 摄像头 RTSP 拉流 | `source: rtsp://...`，`sourceOnDemand: true` |
| 推流鉴权 | `authMethod: http` 回调 Go 后端 |
| 动态添加摄像头 | `POST /v3/config/paths/add/:name` |
| 录制直播 | `record: yes`，配置 `recordPath` |
| 获取流状态 | `GET /v3/paths/list` |
| 事件回调 | `runOnPublish`/`runOnUnpublish` 执行命令 |

---

## 下一篇

下一篇讲解 **FFmpeg 流媒体推拉流实战**：快速上手 FFmpeg 常用命令，RTSP 拉流、RTMP 推流、格式转换的核心参数。

---

## 参考资料

1. [MediaMTX 官方文档](https://github.com/bluenviron/mediamtx/blob/main/README.md)
2. [MediaMTX API 文档](https://github.com/bluenviron/mediamtx/blob/main/apidocs/openapi.yaml)
3. [MediaMTX 配置文件参考](https://github.com/bluenviron/mediamtx/blob/main/mediamtx.yml)

---

## 动手实践

### 练习1：MediaMTX RTSP 转 HLS 一键转发
```yaml
# mediamtx.yml
paths:
  cam1:
    source: rtsp://192.168.1.100:554/stream
    sourceProtocol: tcp

# 启动
docker run -d -p 8888:8888 -p 8554:8554   -v $(pwd)/mediamtx.yml:/mediamtx.yml   bluenviron/mediamtx:latest
```
```bash
# 访问 HLS
curl http://localhost:8888/cam1/index.m3u8
# 或 WebRTC
# 浏览器打开 http://localhost:8889/cam1
```
**预期输出**：MediaMTX 自动从摄像头拉 RTSP，并提供 HLS/WebRTC/RTSP 多协议输出，无需任何编码。

### 练习2：配置推流鉴权回调（Go 业务集成）
```yaml
# mediamtx.yml
paths:
  "~^live/(.+)$":
    publishPermissions:
      - action: publish
        query: token=.*
    runOnPublish: >
      curl -s -X POST http://localhost:8080/api/v1/streams/on-publish
      -H "Content-Type: application/json"
      -d '{"path":"$MTX_PATH","query":"$MTX_QUERY"}'
    runOnUnpublish: >
      curl -s -X POST http://localhost:8080/api/v1/streams/on-unpublish
      -H "Content-Type: application/json"
      -d '{"path":"$MTX_PATH"}'
```
**预期输出**：推流时 MediaMTX 回调业务后端，后端可记录上线时间、触发 AI 任务等。

### 练习3：Go SDK 控制 MediaMTX（通过 API）
```go
// 查询活跃推流列表
resp, err := http.Get("http://localhost:9997/v3/paths/list")
if err != nil {
    log.Error().Err(err).Msg("查询 MediaMTX 路径失败")
    return
}
defer resp.Body.Close()

var result struct {
    Items []struct {
        Name   string `json:"name"`
        Ready  bool   `json:"ready"`
        Source struct {
            Type string `json:"type"`
        } `json:"source"`
    } `json:"items"`
}
json.NewDecoder(resp.Body).Decode(&result)
for _, item := range result.Items {
    if item.Ready {
        log.Info().Str("path", item.Name).Msg("活跃流")
    }
}
```
**预期输出**：输出当前所有活跃推流的路径名，可用于同步业务数据库中的摄像头在线状态。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| RTSP source 连接失败 | 摄像头 RTSP URL 需要认证 | `source: rtsp://user:pass@192.168.x.x/stream` |
| HLS 延迟高（>10s） | hlsSegmentDuration 太长 | `hlsSegmentDuration: 2s`，`hlsPartDuration: 200ms` |
| WebRTC ICE 失败 | 服务器无公网 IP | 配置 `webrtcICEServers` 中添加 TURN 服务器 |
| runOnPublish 不触发 | 配置路径正则不匹配 | 用 `~^` 开头的正则，确保 YAML 字符串加引号 |
| API 9997 端口无响应 | api 未启用 | `api: yes` 加入 mediamtx.yml |
| 多实例路径冲突 | 同名 path 被多个推流占用 | MediaMTX 同一 path 只允许一个推流源，断开旧连接后新推流才能接入 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| path | Path | MediaMTX 中的流路径，相当于频道名 |
| source | Source | path 的媒体来源，可以是 URL（主动拉流）或 publisher（等待推流）|
| runOnPublish | runOnPublish | 推流建立时 MediaMTX 执行的 shell 命令 |
| runOnUnpublish | runOnUnpublish | 推流断开时 MediaMTX 执行的 shell 命令 |
| MTX_PATH | $MTX_PATH | MediaMTX 环境变量，值为当前流的路径名 |
| MTX_QUERY | $MTX_QUERY | MediaMTX 环境变量，值为推流 URL 的 query string |
| API Port | API Port | MediaMTX 管理 API，默认端口 9997 |

---

> **全文完** | 上一篇：[第 15 篇 — SRS 部署与配置实战](15-SRS部署与配置实战.md) | 下一篇：[第 16.5 篇 — ZLMediaKit 实战（安防国标接入首选）](16.5-ZLMediaKit实战.md)
