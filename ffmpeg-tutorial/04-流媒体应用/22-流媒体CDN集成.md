# 第 20 篇：流媒体 CDN 集成

> **模块**：04-流媒体应用 | **难度**：高级 | **预计阅读**：55 分钟  
> **版本**：FFmpeg 8.0 "Huffman" | **更新**：2026-07-28

---

## 目录

1. [CDN 与流媒体分发原理](#1-cdn-与流媒体分发原理)
2. [HLS CDN 分发配置](#2-hls-cdn-分发配置)
3. [预签名 URL 鉴权](#3-预签名-url-鉴权)
4. [回源策略与缓存控制](#4-回源策略与缓存控制)
5. [多 CDN 负载均衡](#5-多-cdn-负载均衡)
6. [CDN 推流（RTMP/SRT）](#6-cdn-推流rtmpsrt)
7. [Go 集成：CDN URL 管理](#7-go-集成cdn-url-管理)
8. [动手实践](#8-动手实践)
9. [本章总结](#9-本章总结)
10. [术语表](#10-术语表)

---

## 学习目标

完成本章后，你将能够：

- 理解 CDN 在流媒体分发中的角色（边缘缓存/回源/BGP）
- 用 FFmpeg `-hls_base_url` 和路径版本化实现 CDN 友好的 HLS 输出
- 为 HLS/DASH 分片生成带过期时间的预签名 URL（HMAC-SHA256）
- 配置合理的 Cache-Control 头区分清单和分片的缓存策略
- 在 Go 中实现多 CDN 故障转移和 URL 签名服务

---

## 1. CDN 与流媒体分发原理

### 1.1 CDN 在流媒体中的位置

```
┌────────────────────────────────────────────────────────┐
│                流媒体 CDN 分发架构                       │
│                                                         │
│  推流端（OBS/App）                                      │
│       │ RTMP/SRT                                       │
│       ▼                                                 │
│  ┌──────────┐     回源    ┌──────────┐                 │
│  │  Origin  │ ──────────▶ │  CDN 节点 │                │
│  │  源站    │ ◀────────── │  (边缘)  │ ──▶ 观看端      │
│  └──────────┘    命中缓存  └──────────┘                 │
│       │           └── 直接返回，不回源                  │
│       │  HLS/DASH 打包                                  │
│       ▼                                                 │
│  .m3u8/.mpd + .ts/.m4s 分片                            │
└────────────────────────────────────────────────────────┘
```

### 1.2 缓存命中的关键：清单 vs 分片

HLS/DASH 有两类文件，缓存策略完全不同：

| 文件类型 | 示例 | 更新频率 | 缓存时长 |
|----------|------|----------|----------|
| 主清单 | master.m3u8 / manifest.mpd | 基本不变 | 长缓存（1小时+）|
| 子清单 | media_0.m3u8 | 每个分片（2-10s）更新一次 | 短缓存（= 分片时长）|
| 分片文件 | seg_001.ts / chunk-0-00001.m4s | 内容固定，名称唯一 | 永久缓存 |

**为什么分片文件能永久缓存？**

虽然分片文件数量最多（每秒可能产生多个），但恰恰是**最适合长期缓存**的：

1. **内容不可变（Immutable）**：一旦生成，`seg_001.ts` 的内容永远不会改变
2. **文件名唯一**：新分片用新文件名（`seg_002.ts`），不会覆盖旧文件
3. **无版本冲突**：即使有 1000 个分片，每个都是独立且不可变的资源

相比之下，**子清单虽然只有一个文件，但内容持续变化**（特别是直播）：

```m3u8
# 第 1 秒：
seg_100.ts
seg_101.ts
seg_102.ts  ← 最新分片

# 第 7 秒：清单更新，加入新分片
seg_101.ts
seg_102.ts
seg_103.ts  ← 新分片
```

如果子清单缓存 1 小时，客户端会一直拿到旧清单，看不到新分片。

**类比理解**：
- 分片 = 书的每一页（印刷后永不改变，页数虽多但每页可永久保存）
- 子清单 = 目录（每次加新页时必须更新）
- 主清单 = 书的封面（偶尔改版才变）

**核心原则**：
- 分片文件名唯一（含序号/时间戳） → `Cache-Control: max-age=31536000, immutable`
- 子清单频繁更新 → `Cache-Control: no-cache` 或 `max-age=2`
- 主清单很少变 → `Cache-Control: max-age=3600`

### 1.3 CDN 回源带宽成本

```
观看端直接请求源站（无 CDN）：
  1000 并发 × 3 Mbps = 3 Gbps 带宽 → 源站崩溃

使用 CDN：
  1000 并发 → 命中率 95% → 只有 50 次回源
  50 × 3 Mbps = 150 Mbps → 源站轻松应对
```

**CDN 的价值**：
- 降低源站带宽开销（95%+ 命中率）
- 就近接入（BGP Anycast），降低播放延迟
- 抗 DDoS、防盗链、HTTPS 卸载

---

## 2. HLS CDN 分发配置

### 2.1 基础 base_url 配置

FFmpeg 生成的 m3u8 默认是相对路径，需要用 `-hls_base_url` 指定 CDN 域名：

```bash
# 无 base_url（相对路径）
ffmpeg -i input.mp4 \
  -hls_segment_filename "seg_%03d.ts" \
  index.m3u8

# index.m3u8 内容：
# #EXTINF:6.0,
# seg_000.ts  ← 相对路径，浏览器请求 http://origin/seg_000.ts

# 有 base_url（CDN 绝对路径）
ffmpeg -i input.mp4 \
  -hls_base_url "https://cdn.example.com/live/stream1/" \
  -hls_segment_filename "/var/hls/stream1/seg_%03d.ts" \
  /var/hls/stream1/index.m3u8

# index.m3u8 内容：
# #EXTINF:6.0,
# https://cdn.example.com/live/stream1/seg_000.ts  ← CDN 地址
```

### 2.2 路径版本化（防缓存碰撞）

每次直播开始用不同路径，避免旧分片被缓存：

```bash
# 用时间戳作为路径版本号
STREAM_TOKEN=$(date +%s)
CDN_BASE="https://cdn.example.com/live/${STREAM_TOKEN}/"

ffmpeg -i rtmp://localhost:1935/live/stream \
  -c copy \
  -f hls \
  -hls_time 6 \
  -hls_list_size 5 \
  -hls_flags delete_segments \
  -hls_base_url "$CDN_BASE" \
  -hls_segment_filename "/var/hls/${STREAM_TOKEN}/seg_%05d.ts" \
  /var/hls/index.m3u8

# 观看地址：https://cdn.example.com/live/1785312645/index.m3u8
```

**优点**：
- 新直播用新路径，CDN 自动清理旧缓存
- 防止多次直播间的分片编号冲突

### 2.3 DASH CDN 配置

```bash
STREAM_TOKEN=$(date +%s)
CDN_BASE="https://cdn.example.com/dash/${STREAM_TOKEN}/"

ffmpeg -i rtmp://localhost:1935/live/stream \
  -c:v libx264 -b:v 2M -c:a aac -b:a 128k \
  -f dash \
  -seg_duration 6 \
  -window_size 5 \
  -init_seg_name "init-$RepresentationID$.m4s" \
  -media_seg_name "chunk-$RepresentationID$-$Number%05d$.m4s" \
  -utc_timing_url "https://time.akamai.com/?iso" \
  /var/dash/${STREAM_TOKEN}/manifest.mpd

# 手动给 MPD 加 BaseURL（FFmpeg DASH 不支持 base_url 参数）
sed -i "s|<Period|<BaseURL>${CDN_BASE}</BaseURL><Period|" /var/dash/${STREAM_TOKEN}/manifest.mpd
```

---

## 3. 预签名 URL 鉴权

### 3.1 为什么需要 URL 签名

```
不鉴权：https://cdn.example.com/live/stream.m3u8
  → 任何人都能访问，可能被盗链/爬取

鉴权后：https://cdn.example.com/live/stream.m3u8?sign=abc123&expires=1785312645
  → 签名错误或已过期 → CDN 返回 403
```

### 3.2 HMAC-SHA256 签名算法

主流 CDN（阿里云/腾讯云/Cloudflare）的鉴权本质是同一套逻辑：

```
sign = HMAC-SHA256(secret_key, path + expires)
URL  = https://cdn.example.com/path?sign=<hex>&expires=<unix_ts>
```

**签名验证流程**：

```
客户端请求：GET /live/stream.m3u8?sign=abc&expires=1785312645
      │
      ▼
CDN 边缘节点：
  1. 检查 expires 是否未过期（> 当前时间）
  2. 用相同密钥重新计算 sign
  3. 比较计算结果与请求中的 sign
  4. 一致 → 放行；不一致或过期 → 403
      │
      ▼
源站：不直接暴露，只接受 CDN 回源请求
```

### 3.3 m3u8 反向代理签名

HLS 有个特殊问题：播放器拿到 m3u8 后，会自动请求里面的分片 URL。
如果 m3u8 里的分片 URL 没有签名，分片就会 403。

**解决方案**：用反向代理动态注入分片签名：

```
播放器 → 反向代理 /sign → 源站 m3u8 → 注入签名 → 返回给播放器
播放器 → CDN 分片（带签名） → 验证通过 → 返回分片内容
```

```nginx
# Nginx 反向代理示例：自动给分片 URL 注入签名
location /signed-hls/ {
    # 转发到签名服务
    proxy_pass http://go-sign-service:8080/sign-m3u8/;
    proxy_set_header X-Original-URI $request_uri;
}
```

---

## 4. 回源策略与缓存控制

### 4.1 Nginx 源站 Cache-Control 配置

```nginx
server {
    listen 80;
    root /var/hls;

    # 子清单：短缓存（= 分片时长）
    location ~* \.m3u8$ {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Access-Control-Allow-Origin "*";
        add_header Pragma "no-cache";
        expires 0;
    }

    # 分片：永久缓存（内容固定）
    location ~* \.(ts|m4s|mp4|m4a)$ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Access-Control-Allow-Origin "*";
        expires 1y;
    }

    # MPD 主清单：中等缓存
    location ~* \.mpd$ {
        add_header Cache-Control "public, max-age=3600";
        add_header Access-Control-Allow-Origin "*";
        expires 1h;
    }
}
```

### 4.2 CDN 回源健康检测

```bash
# 检查 CDN 是否正确缓存（X-Cache: HIT = 命中缓存）
curl -I https://cdn.example.com/live/stream/seg_00001.ts

# 响应头示例：
# HTTP/2 200
# Cache-Control: public, max-age=31536000, immutable
# X-Cache: HIT                    ← 命中 CDN 缓存
# Age: 3600                       ← 已缓存 3600 秒
# CF-Ray: 8abc123def456789-HKG    ← Cloudflare 边缘节点 HKG（香港）

# 首次请求（回源）：
# X-Cache: MISS                   ← 未命中，CDN 回源获取
```

### 4.3 主动缓存预热

大型直播活动开始前预热 CDN：

```bash
# 预热脚本：提前拉取 m3u8 和前几个分片
MANIFEST_URL="https://cdn.example.com/live/stream/index.m3u8"

# 1. 拉取主清单（触发 CDN 缓存）
curl -s "$MANIFEST_URL" | while read line; do
    if [[ "$line" == *.ts ]] || [[ "$line" == *.m4s ]]; then
        # 2. 并发预热分片
        curl -s --head "${CDN_BASE}${line}" &
    fi
done
wait
echo "预热完成"
```

---

## 5. 多 CDN 负载均衡

### 5.1 为什么要多 CDN

```
单 CDN 风险：
  - 某 CDN 局部故障 → 部分地区无法观看
  - 议价能力弱 → 带宽成本高
  - 特定运营商覆盖差 → 卡顿

多 CDN 方案：
  - 主备切换：CDN-A 故障自动切 CDN-B
  - 按地区分流：国内用阿里云，海外用 Cloudflare
  - 按成本分流：按实时价格分配流量
```

### 5.2 客户端调度（推荐）

由播放器向调度服务请求最优 CDN 地址：

```
播放器 → GET /api/v1/stream/play-url?stream_id=xxx
              │
              ▼
       调度服务（Go）：
         1. 解析客户端 IP → 地域/运营商
         2. 查询各 CDN 实时质量（成功率/延迟）
         3. 按权重选择 CDN
         4. 生成带签名的播放 URL
              │
              ▼
       返回：{
         "primary":  "https://cdn-a.example.com/...?sign=...",
         "fallback": "https://cdn-b.example.com/...?sign=..."
       }
              │
              ▼
       播放器：primary 失败 → 自动切 fallback
```

### 5.3 DNS 调度（简单但切换慢）

```
# 通过 CNAME 权重分配
play.example.com  CNAME  cdn-a.example.com  (权重 70%)
play.example.com  CNAME  cdn-b.example.com  (权重 30%)

缺点：DNS 缓存导致切换延迟（TTL 60~300s）
优点：无需改播放器代码
```

---

## 6. RTMP/SRT 推流到 CDN

### 6.1 推流到 CDN 直播中心

主流云厂商都提供推流地址 + 鉴权：

```bash
# 阿里云直播推流（含鉴权参数）
PUSH_URL="rtmp://live-push.example.com/live/stream1?auth_key=1785312645-0-0-abcdef123456"

ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2500k -maxrate 2500k -bufsize 5000k \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -c:a aac -b:a 128k -ar 44100 \
  -f flv "$PUSH_URL"
```

**auth_key 格式**（阿里云）：`时间戳-随机数-uid-MD5值`

```
MD5值 = md5("/live/stream1-1785312645-0-0-推流密钥")
```

### 6.2 SRT 推流到 CDN（低延迟场景）

```bash
# SRT 推流（抗弱网，适合远程连线/户外直播）
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 3M -maxrate 3M -bufsize 6M -g 60 \
  -c:a aac -b:a 128k \
  -f mpegts \
  "srt://srt-push.example.com:9000?streamid=#!::h=stream1,m=publish&latency=300"
```

### 6.3 推流质量监控

```bash
# FFmpeg 输出 JSON 进度，便于程序解析
ffmpeg -re -i input.mp4 \
  -c:v libx264 -b:v 2500k -c:a aac -b:a 128k \
  -progress pipe:1 -nostats \
  -f flv "$PUSH_URL"

# 输出示例（每秒一组）：
# frame=150
# fps=25.0
# bitrate=2498.3kbits/s
# out_time_ms=6000000
# speed=1.0x           ← 关键指标：< 1.0 表示编码跟不上，会导致推流断流
# progress=continue
```

---

## 7. Go 实现：CDN URL 签名与调度服务

### 7.1 URL 签名器

```go
package cdn

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"net/url"
	"strconv"
	"time"
)

// SignerConfig CDN 签名配置
type SignerConfig struct {
	Domain    string        // CDN 域名，如 cdn.example.com
	SecretKey string        // 鉴权密钥
	TTL       time.Duration // 签名有效期
	Scheme    string        // http / https
}

// URLSigner CDN URL 签名器
type URLSigner struct {
	config SignerConfig
}

// NewURLSigner 创建签名器
func NewURLSigner(cfg SignerConfig) *URLSigner {
	if cfg.Scheme == "" {
		cfg.Scheme = "https"
	}
	if cfg.TTL == 0 {
		cfg.TTL = 30 * time.Minute
	}
	return &URLSigner{config: cfg}
}

// Sign 为指定路径生成带签名的完整 URL
// path 必须以 / 开头，如 /live/stream1/index.m3u8
func (s *URLSigner) Sign(path string) string {
	expires := time.Now().Add(s.config.TTL).Unix()
	sign := s.calcSign(path, expires)

	u := url.URL{
		Scheme: s.config.Scheme,
		Host:   s.config.Domain,
		Path:   path,
	}
	q := u.Query()
	q.Set("expires", strconv.FormatInt(expires, 10))
	q.Set("sign", sign)
	u.RawQuery = q.Encode()

	return u.String()
}

// Verify 校验签名是否有效（源站自建鉴权时使用）
func (s *URLSigner) Verify(path, sign string, expires int64) error {
	if time.Now().Unix() > expires {
		return fmt.Errorf("签名已过期")
	}
	if !hmac.Equal([]byte(sign), []byte(s.calcSign(path, expires))) {
		return fmt.Errorf("签名校验失败")
	}
	return nil
}

// calcSign 计算 HMAC-SHA256 签名
func (s *URLSigner) calcSign(path string, expires int64) string {
	mac := hmac.New(sha256.New, []byte(s.config.SecretKey))
	mac.Write([]byte(fmt.Sprintf("%s%d", path, expires)))
	return hex.EncodeToString(mac.Sum(nil))
}
```

### 7.2 多 CDN 调度服务

```go
// CDNProvider 单个 CDN 提供商
type CDNProvider struct {
	Name      string
	Signer    *URLSigner
	Weight    int     // 分流权重（0-100）
	HealthURL string  // 健康探测 URL
}

// CDNDispatcher 多 CDN 调度器
type CDNDispatcher struct {
	providers []*CDNProvider
	mu        sync.RWMutex
	weights   []int // 累积权重，用于快速抽签
}

// NewCDNDispatcher 创建调度器
func NewCDNDispatcher(providers []*CDNProvider) *CDNDispatcher {
	d := &CDNDispatcher{providers: providers}
	d.buildWeights()
	return d
}

// buildWeights 构建累积权重数组
func (d *CDNDispatcher) buildWeights() {
	d.mu.Lock()
	defer d.mu.Unlock()
	d.weights = make([]int, len(d.providers))
	total := 0
	for i, p := range d.providers {
		total += p.Weight
		d.weights[i] = total
	}
}

// GetPlayURL 为指定流路径返回主备播放 URL
func (d *CDNDispatcher) GetPlayURL(streamPath string) (primary, fallback string) {
	d.mu.RLock()
	defer d.mu.RUnlock()

	if len(d.providers) == 0 {
		return "", ""
	}

	// 按权重随机选主 CDN
	n := rand.Intn(d.weights[len(d.weights)-1])
	primaryIdx := 0
	for i, w := range d.weights {
		if n < w {
			primaryIdx = i
			break
		}
	}
	primary = d.providers[primaryIdx].Signer.Sign(streamPath)

	// 备用 CDN 选权重第二的
	if len(d.providers) > 1 {
		fallbackIdx := (primaryIdx + 1) % len(d.providers)
		fallback = d.providers[fallbackIdx].Signer.Sign(streamPath)
	}
	return primary, fallback
}
```

### 7.3 m3u8 代理签名服务

播放器拿到 m3u8 后需要访问里面的分片，代理服务负责动态签名：

```go
// M3U8SignProxy 代理服务，动态给 m3u8 中的分片 URL 注入签名
type M3U8SignProxy struct {
	signer *URLSigner
	client *http.Client
}

// ProxyM3U8 拉取上游 m3u8，替换分片 URL 为签名版本
func (p *M3U8SignProxy) ProxyM3U8(w http.ResponseWriter, r *http.Request) {
	upstreamURL := r.URL.Query().Get("url")
	if upstreamURL == "" {
		http.Error(w, "missing url param", http.StatusBadRequest)
		return
	}

	resp, err := p.client.Get(upstreamURL)
	if err != nil {
		http.Error(w, "fetch failed", http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)

	// 逐行替换分片 URL
	lines := strings.Split(string(body), "\n")
	for i, line := range lines {
		if strings.HasSuffix(line, ".ts") || strings.HasSuffix(line, ".m4s") {
			// 提取路径，注入签名
			parsed, err := url.Parse(line)
			if err == nil {
				lines[i] = p.signer.Sign(parsed.Path)
			}
		}
	}

	w.Header().Set("Content-Type", "application/vnd.apple.mpegurl")
	w.Header().Set("Cache-Control", "no-cache")
	w.Write([]byte(strings.Join(lines, "\n")))
}
```

### 7.4 Handler 集成示例

```go
// PlayURLHandler 播放地址接口
// @Summary 获取播放地址
// @Tags 流媒体
// @Param stream_id query string true "流ID" example("123456789012345678")
// @Success 200 {object} response.Response{data=dto.PlayURLResponse} "成功"
// @Router /api/v1/stream/play-url [get]
func (h *StreamHandler) PlayURLHandler(c *gin.Context) {
	var req dto.PlayURLRequest
	if err := c.ShouldBindQuery(&req); err != nil {
		response.Error(c, xerr.New(xerr.ErrInvalidParam.Code, "参数错误"))
		return
	}

	resp, err := h.streamService.GetPlayURL(c.Request.Context(), req.StreamID)
	if err != nil {
		response.Error(c, err)
		return
	}
	response.Success(c, resp)
}

// Service 层
func (s *StreamService) GetPlayURL(ctx context.Context, streamID string) (*dto.PlayURLResponse, error) {
	stream, err := s.repo.GetByID(ctx, streamID)
	if err != nil {
		log.Error().Err(err).Str("stream_id", streamID).Msg("查询流信息失败")
		return nil, xerr.Wrap(xerr.ErrInternal.Code, "查询流信息失败", err)
	}
	if stream == nil {
		return nil, xerr.New(xerr.ErrNotFound.Code, "流不存在")
	}

	hlsPath := fmt.Sprintf("/live/%s/index.m3u8", stream.StreamToken)
	primary, fallback := s.dispatcher.GetPlayURL(hlsPath)

	log.Info().Str("stream_id", streamID).Str("path", hlsPath).Msg("生成播放地址")

	return &dto.PlayURLResponse{
		StreamID:    streamID,
		HLSURL:      primary,
		FallbackURL: fallback,
		ExpiresAt:   time.Now().Add(30 * time.Minute).UnixMilli(),
	}, nil
}
```

---

## 8. 动手实践

### 实践 1：本地模拟 CDN 分发

用 Nginx 作为「边缘节点」，验证缓存头是否生效。

```bash
# 1. 启动源站（生成 HLS）
mkdir -p /tmp/hls
ffmpeg -re -f lavfi -i "testsrc=size=1280x720:rate=30" \
  -f lavfi -i "sine=frequency=1000" \
  -c:v libx264 -preset veryfast -b:v 2M -g 60 \
  -c:a aac -b:a 128k \
  -f hls -hls_time 4 -hls_list_size 5 \
  -hls_flags delete_segments \
  -hls_segment_filename "/tmp/hls/seg_%05d.ts" \
  /tmp/hls/index.m3u8

# 2. 另开终端，启动 Nginx（模拟边缘节点）
cat > /tmp/nginx-cdn.conf <<'EOF'
events { worker_connections 1024; }
http {
    include /usr/local/etc/nginx/mime.types;
    server {
        listen 8090;
        root /tmp/hls;
        location ~* \.m3u8$ {
            add_header Cache-Control "no-cache";
            add_header Access-Control-Allow-Origin "*";
        }
        location ~* \.ts$ {
            add_header Cache-Control "public, max-age=31536000, immutable";
            add_header Access-Control-Allow-Origin "*";
        }
    }
}
EOF
nginx -c /tmp/nginx-cdn.conf

# 3. 验证缓存头
curl -I http://localhost:8090/index.m3u8   # 应看到 Cache-Control: no-cache
curl -I http://localhost:8090/seg_00001.ts # 应看到 max-age=31536000, immutable
```

**验证目标**：
- m3u8 每次请求都是新内容（no-cache）
- ts 分片带永久缓存头（immutable）

### 实践 2：Go URL 签名验证

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	signer := NewURLSigner(SignerConfig{
		Domain:    "cdn.example.com",
		SecretKey: "my-secret-key-2026",
		TTL:       30 * time.Minute,
		Scheme:    "https",
	})

	// 生成签名 URL
	path := "/live/stream1/index.m3u8"
	signedURL := signer.Sign(path)
	fmt.Println("签名URL:", signedURL)
	// 输出：https://cdn.example.com/live/stream1/index.m3u8?expires=1785312645&sign=abc123...

	// 本地验证签名（源站鉴权场景）
	expires := time.Now().Add(30 * time.Minute).Unix()
	sign := signer.calcSign(path, expires)
	if err := signer.Verify(path, sign, expires); err != nil {
		fmt.Println("验证失败:", err)
	} else {
		fmt.Println("验证通过 ✓")
	}

	// 过期签名验证
	expiredExpires := time.Now().Add(-1 * time.Hour).Unix()
	expiredSign := signer.calcSign(path, expiredExpires)
	if err := signer.Verify(path, expiredSign, expiredExpires); err != nil {
		fmt.Println("过期签名:", err) // 签名已过期
	}
}
```

### 实践 3：多 CDN 权重调度

```go
func main() {
	dispatcher := NewCDNDispatcher([]*CDNProvider{
		{
			Name: "CDN-A（主力）",
			Signer: NewURLSigner(SignerConfig{
				Domain: "cdn-a.example.com", SecretKey: "key-a", TTL: 30 * time.Minute,
			}),
			Weight: 70,
		},
		{
			Name: "CDN-B（备用）",
			Signer: NewURLSigner(SignerConfig{
				Domain: "cdn-b.example.com", SecretKey: "key-b", TTL: 30 * time.Minute,
			}),
			Weight: 30,
		},
	})

	streamPath := "/live/room123/index.m3u8"
	primary, fallback := dispatcher.GetPlayURL(streamPath)
	fmt.Println("主播放地址:", primary)
	fmt.Println("备播放地址:", fallback)

	// 模拟 1000 次请求，验证分流比例
	countA, countB := 0, 0
	for i := 0; i < 1000; i++ {
		p, _ := dispatcher.GetPlayURL(streamPath)
		if strings.Contains(p, "cdn-a") {
			countA++
		} else {
			countB++
		}
	}
	fmt.Printf("CDN-A: %d次 (%.1f%%)\n", countA, float64(countA)/10)
	fmt.Printf("CDN-B: %d次 (%.1f%%)\n", countB, float64(countB)/10)
	// 期望：CDN-A ~70%，CDN-B ~30%
}
```

### 常见问题排查

| 问题 | 排查方法 | 解决方案 |
|------|----------|----------|
| 播放器收到 403 | `curl -I <url>` 查看响应码 | 检查签名是否过期，密钥是否正确 |
| 分片缓存不命中 | 响应头 `X-Cache: MISS` | 检查 Cache-Control 是否设置为 immutable |
| m3u8 更新延迟 | 检查 CDN 控制台缓存配置 | 确保 m3u8 路径规则 `Cache-Control: no-cache` |
| 播放器能播主清单但分片 403 | 检查分片是否有单独签名 | 部署 m3u8 代理签名服务 |
| 多 CDN 切换失败 | 检查 fallback URL 是否有效 | 实现客户端重试逻辑 |

---

## 9. 本章总结

### 核心要点

1. **缓存策略是 CDN 的灵魂**：m3u8 子清单用 `no-cache`（每次更新），分片用 `immutable`（永久缓存），主清单用 `max-age=3600`（小时级）
2. **路径版本化防缓存污染**：每次直播用新时间戳路径（`/live/1785312645/...`），避免旧分片干扰新直播
3. **URL 签名防盗链**：HMAC-SHA256 签名（`sign=hash(path+expires+secret)`），CDN 边缘节点验证后放行
4. **m3u8 代理签名解决分片 403**：播放器拿到的 m3u8 必须包含已签名的分片 URL，需代理动态注入
5. **多 CDN 提升容灾能力**：客户端调度（按地域/质量选 CDN）+ 主备切换，降低单点故障风险

### 缓存命中率优化 Checklist

- [ ] 源站 Nginx 配置 `Cache-Control` 响应头（.m3u8 no-cache / .ts immutable）
- [ ] FFmpeg 用 `-hls_base_url` 指定 CDN 域名
- [ ] 每次直播用新路径版本号（时间戳或随机 token）
- [ ] CDN 控制台配置缓存规则（路径匹配 / MIME 类型）
- [ ] 预热重要分片（首屏加载的前几个 segment）

### 性能指标

| 指标 | 目标值 | 监控方法 |
|------|--------|----------|
| 缓存命中率 | > 95% | CDN 控制台查看 HIT/MISS 比例 |
| 回源带宽占比 | < 5% | 总带宽 vs CDN 回源带宽 |
| 首屏加载时间 | < 3s | 浏览器开发者工具 Network 面板 |
| 签名验证成功率 | > 99.9% | 业务日志统计 403 比例 |

### 下一步方向

- **成本优化**：动态调整码率（ABR），低峰时段降低预热频率
- **安全加固**：Referer 白名单、播放 Token 一次性使用、IP 限速
- **全球加速**：海外用户接入国际 CDN（Cloudflare/Fastly/Akamai）
- **低延迟优化**：LL-HLS + CDN 边缘计算节点

**下一篇**：[第 21 篇：字幕与水印处理](../05-高级主题/21-字幕与水印处理.md) —— 进入高级主题模块，学习动态水印、硬字幕、ASS 样式、多语言字幕切换。

---

## 10. 术语表

| 术语 | 英文 | 解释 |
|------|------|------|
| CDN | Content Delivery Network | 内容分发网络，通过边缘节点缓存内容降低源站压力 |
| 边缘节点 | Edge Node | 靠近用户的缓存服务器，承担大部分流量 |
| 回源 | Origin Pull | CDN 缓存未命中时向源站请求内容 |
| 缓存命中率 | Cache Hit Rate | 从缓存直接返回的请求占比（越高越好）|
| immutable | - | Cache-Control 指令，表示内容永不变化，可永久缓存 |
| BGP Anycast | - | 多个节点共享同一 IP，路由自动选最近节点 |
| 预签名 URL | Pre-signed URL | 带时间戳和签名的 URL，防止盗链 |
| HMAC-SHA256 | - | 基于密钥的哈希算法，用于生成不可伪造的签名 |
| 鉴权回调 | Auth Callback | CDN 边缘节点调用业务服务器验证签名（旁路鉴权）|
| 多 CDN | Multi-CDN | 同时使用多家 CDN 厂商，按地域/成本/质量调度 |

---

**全文完** | 字数：约 7200 字 | 代码示例：12 个 | 实践环节：3 个  
**作者**：FFmpeg 教程编写组 | **版本**：基于 FFmpeg 8.0 "Huffman" | **标签**：`CDN` `HLS` `DASH` `URL签名` `缓存策略` `多CDN调度`
