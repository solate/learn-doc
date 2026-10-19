# CDN 分发与边缘加速

> **适合读者**：需要把视频/直播内容分发到大量用户的后端开发者  
> **核心问题**：CDN 如何工作？视频和直播在 CDN 上有什么特殊需求？怎么做到低回源、高命中、省成本？

---

## 一、为什么需要 CDN：单源站的三大瓶颈

> 读完本章你能理解：不加 CDN 直接对外提供视频服务会撞上哪几堵墙，从而知道 CDN 解决的是什么问题。

假设你只有一台源站（比如北京的一台 Nginx/SRS + OSS），直接让全国用户来拉流，会同时遇到三个问题：

```
问题 1 · 物理距离           问题 2 · 带宽                问题 3 · 单点可靠
新疆用户 → 北京 3500km     1000 人在线看直播             源站一挂 = 全网挂
往返延迟 ~60ms 以上        每人 2Mbps = 2Gbps 出网       DNS/升级/机房故障
卡顿、首屏慢                一般云服务器扛不住            没有兜底
```

**CDN 的解法就是把"内容"往用户身边放一份**：

- 边缘节点（PoP）遍布各地，用户永远连**最近**的节点，物理距离从上千公里缩到几十公里；
- 同一份热内容在边缘被成千上万人缓存复用，源站只被"打穿"一次，出网带宽压力从 `观看人数 × 码率` 降到 `内容更新频率`；
- 任何一个边缘节点挂了，智能调度把流量切去相邻节点，全网站点整体高可用。

> 一句话总结：**CDN 是用"分布式的缓存"换"延迟、带宽、可用性"三个指标。** 视频是体积最大、最热门的内容形态，所以视频业务是最典型的 CDN 场景。

---

## 二、CDN 工作原理：从用户请求到命中响应

> 读完本章你能理解：一次"看起来只是打开一个 URL"的背后，CDN 在 DNS、缓存、回源三个环节各做了什么。

### 1. 一次完整的 CDN 请求

```
用户请求 video.example.com/hls/stream.m3u8
    │
    ├─① DNS 解析 video.example.com
    │    CNAME 指向 CDN 调度域名（如 xxx.kunlunar.com）
    │    ↓
    ├─② GSLB 智能调度：根据用户 IP 归属地 + CDN 节点负载，
    │    解析出「离用户最近、且当前最空闲」的节点 IP
    │    ↓
    ├─③ 用户连上边缘节点，查缓存：
    │     HIT  ──► 直接把缓存内容返回（响应头 X-Cache: HIT）
    │     MISS ──► 节点向源站"回源"拉取 → 存到本地（并设置 TTL）→ 返回
    │    ↓
    └─④ 用户拿到内容；后续同一节点上的其他用户直接命中缓存
```

### 2. 三个关键技术点

**GSLB（Global Server Load Balancing）智能调度**，决定"连哪个节点"：
- 传统方案：DNS 按请求 IP 归属地解析到不同节点 IP（可以做到省级/城市级）；
- 更高级：**HTTPDNS**——客户端绕过运营商 DNS 直接向 HTTPDNS 服务请求最优节点 IP，避免运营商 Local DNS 缓存错误、被劫持导致的"连到外地节点"；
- 部分厂商用 **Anycast**（一个 IP 全网广播，路由层面自动就近）做调度层。

**缓存生命周期**：
```
首次请求(MISS) → 回源成功 → 缓存副本(记录 TTL) → TTL 内请求(HIT)
                                              → TTL 过期 → 重新回源校验
```

**命中率 / 回源率**是衡量 CDN 是否"白花钱"的两个核心指标：
- 命中率（Cache Hit Rate）= CDN 直接响应的请求比例，视频点播正常应在 **95% 以上**；
- 回源率 = 请求落到源站的比例，回源率高 = 缓存没用好 = 源站压力、源站带宽都在烧钱。

> ⚠️ **国内域名做 CDN 必须 ICP 备案**。未备案域名接阿里云/腾讯云 CDN 会被直接拒绝；出海业务（Cloudflare 等境外厂商）则不需要备案。这是选型前就要想清楚的硬约束。

---

## 三、为什么 HLS/HTTP-FLV 天生适合 CDN，而 RTMP/WebRTC 不适合

> 读完本章你能理解：同样是视频，有的协议天生适合大规模分发，有的只适合点对点——理解这个差异才知道"方案落地时该把什么推上 CDN"。

[第 10 篇](../01-底层技术基础/10-HLS协议详解.md)讲过 HLS 的本质：**m3u8 是文本索引，TS/fMP4 是分片文件，全部走普通 HTTP GET**。这一点决定了它在 CDN 上几乎零改造就能跑：

| 维度 | HLS / HTTP-FLV | RTMP / WebRTC |
|------|---------------|----------------|
| 传输协议 | 纯 HTTP（TCP:80/443） | RTMP 私有 TCP 协议 / WebRTC 用 UDP+DTLS |
| CDN 能否缓存 | 文件级缓存，天然适配 | 长连接流式数据，无法"缓存一份给下个人" |
| 缓存粒度 | 按分片缓存，精确到秒 | 整条流，只能转发不能缓存 |
| 断点续传/降级 | Range、304、多码率切换都支持 | 不支持 |
| 中间网络设备 | 只要能过 HTTP，就一定能过 CDN/代理/LB | 对 NAT、防火墙、运营商劫持敏感 |

所以业内有一个心照不宣的约定：

> **浏览器能直接播放的，都是 HTTP 承载的协议；CDN 也只负责分发 HTTP 的内容。** 因此所有"要大面积分发"的视频，最终都会被服务端转码成 HLS / HTTP-FLV / DASH（同为 HTTP）再挂到 CDN 后面。
>
> - 点播 → HLS/DASH（fMP4/CMAF 是 2026 年的默认分片格式，见[第 11 篇](../01-底层技术基础/11-DASH与CMAF统一分片格式.md)）
> - 直播 → HLS / LL-HLS / HTTP-FLV（HTTP-FLV 方案见[第 21 篇](./21-低延迟播放完整方案.md)）
> - RTSP/RTMP 只出现在"接入侧"（摄像头/推流器 → 流媒体服务器），不直接面向用户

---

## 四、视频缓存策略详解（本篇核心）

> 读完本章你能理解：m3u8 和 .ts 该分别设置什么 Cache-Control、为什么不能一刀切"不缓存"，以及"带签名的 URL 为什么会把命中率打到 0"这个最常见的坑。

### 1. 先搞清 Cache-Control 的语义

很多人把 `no-cache` 当成"不缓存"，**这是最大的误解**：

| 响应头 | 含义 | CDN 行为 |
|--------|------|---------|
| `public, max-age=300` | 强缓存 300 秒 | 300 秒内直接 HIT，不回源 |
| `no-cache` | **可以存储，但每次使用必须先回源校验** | 每次请求都带上 ETag 回源，源站校验没变返回 304，内容仍由缓存返回 |
| `no-store` | 禁止存储 | 完全不缓存，每次回源完整拉取 |
| `public` / `private` | 允许共享缓存 / 仅私有缓存 | CDN 属于共享缓存，`public` 才缓存 |

关键区别在于：**`no-cache` 保住了"内容存在边缘"这件事，只是多了一次 304 校验；`no-store` 连内容都不存边缘，每次全量回源。** 对视频源站来说，304 校验几乎不花带宽（只回源一个几十字节的响应头），而全量回源是很贵的。

### 2. 视频缓存策略矩阵

不同内容类型的缓存策略完全不同，正确配置如下：

| 内容 | 推荐响应头 | 原因 |
|------|-----------|------|
| **直播当前 m3u8**（实时更新） | `Cache-Control: no-cache` + ETag | 必须保证每次拿到最新片段列表，但用 304 免去重复下载 |
| **点播 master.m3u8**（静态，不变） | `public, max-age=3600` | 内容固定，放心缓存；换版本时改文件名即可 |
| **TS / fMP4 分片** | `public, max-age=分片时长`（如 4s，可略长） | 切片写完后**不可变**，按切片时长缓存正好匹配播放节奏 |
| **MP4 整文件（点播）** | `public, max-age=86400`+ | 大文件，长缓存减少回源 |
| **LL-HLS 未写完的 .part** | `Cache-Control: no-store` | 正在写入，缓存会导致播放器拿到残缺内容（见[第 10 篇 §LL-HLS](../01-底层技术基础/10-HLS协议详解.md)） |
| **签名/动态 URL** | 不靠响应头，靠**缓存键**处理（见 §4） | 每个 URL 都不同，见下 |

> ⚠️ **"m3u8 一律不缓存"是过度简化**——它只对直播成立。点播的 master.m3u8 是固定文件，缓存它是免费收益。带货了「先判断内容动不动，再决定缓存策略」的习惯以后，你会自然避开这个一刀切。

为了拿到 ETag，源站最好开启 `etag on`；如果源站是 OSS/S3 这类对象存储，本身就会自动带 ETag 和 Last-Modified，304 基本免费。

### 3. 直播 m3u8 的源站配置（含 304 校验）

```nginx
# 源站：直播 HLS
location /live/ {
    root /data;

    # 正在写入的 LL-HLS 分片，禁止缓存
    location ~* \.part$ {
        add_header Cache-Control "no-store";
    }

    # 直播 m3u8：可存但必须回源校验 → 走 ETag 304
    location ~* \.m3u8$ {
        etag on;
        add_header Cache-Control "no-cache";
    }

    # ts/fmp4 切片：写后不可变，按切片时长缓存
    location ~* \.(ts|m4s)$ {
        etag on;
        add_header Cache-Control "public, max-age=6";
    }
}
```

### 4. 缓存键（Cache Key）与签名 URL —— 最容易忽略的大坑

CDN 默认用**完整 URL（含 query string）**作为缓存键。这带来一个直接影响视频业务的问题：

> [第 22 篇旧版] 的鉴权方案是给 URL 拼 `?auth_key=时间戳-随机串-签名`。这个参数**每次请求都不同**——于是 CDN 眼中的"同一个视频文件"每次都长得不一样，**每个请求都 MISS、全量回源，命中率直接掉到接近 0**。

这就是**鉴权与缓存互相冲突**：签名防了盗链，却也把缓存打爆了。

解法（两层都要做，缺一不可）：
1. **CDN 侧：把鉴权参数从缓存键里剔除**。各家配置方式不同：
   - 阿里云 CDN → 「缓存配置 → 过滤参数」；
   - Cloudflare → Cache Rules 里 `Cache Key` 指定忽略 `auth_key` query param；
   - 腾讯云/CloudFront => 类似"忽略 URL 参数"/缓存键自定义。
2. **源站侧：签名只管鉴权，内容本身仍然稳定**。即 `auth_key` 变了，但真正的资源路径（`/hls/stream_00001.ts`）不变，所以剔除参数后能正确命中同一个缓存副本。

> 你可以在"动手实践 §练习2"里亲手验证这一现象。

### 5. 预热 / 预推送（热点内容主动分发）

直播、大型点播上架、大促专场这类"必然热门"的内容，**不能等用户来拉**——第一个用户会替所有人承担一次回源。做法：
- **手动预热**：开播前把 master.m3u8 和首批切片 URL 提交给 CDN 预热 API，CDN 立即回源填满各节点，开播即 HIT；
- **自动预热**：内容库落盘/生成时，后端调用厂商的 push 接口批量预热，见下文 §8 的示例。

---

## 五、回源优化：命中率升上去之后，剩下的成本在源站这一跳

> 读完本章你能理解：即便命中率 95%，每一跳"回源"也有讲究——Range、304、多源、QPS 限流，这些决定源站带宽和稳定性。

缓存没有覆盖到的请求会回源，回源又慢又费源站带宽。下面按重要性排序：

### 1. Range Request（范围请求）——视频观看的本质诉求

视频播放天然要**跳着看**、**看半截**：
- 用户拖进度条到 1:23:45，播放器 90% 不会从头开始，而是发 `Range: bytes=xxx-`；
- 直播 CDN 回源也应按分片回源，而不是整个 m3u8 把几百个切片地址全拉回来。

所以源站**必须支持并正确响应 HTTP Range**：

```
GET /vod/movie.mp4
Range: bytes=134217728-141557759
──────────►
206 Partial Content    ← 正常情况应返回 206，而不是 200（200 = 忽略 Range 给了整个文件）
Content-Range: bytes 134217728-141557759/1048576000
```

Nginx 默认支持 `mp4` 模块的 Range（`mp4` module + `mp4_max_buffer_size`），OSS/S3/CloudFront 原生支持。**自建 Nginx 时一定确认对视频扩展名开启了 Range**（`location ~ \.mp4$ { mp4; }`）。

### 2. 304 Not Modified（配合 ETag）——省"重复下载"

前面 §4 的 `Cache-Control: no-cache` 之所以能用，正是因为它靠 304 兜底：

```
用户 → CDN：网络慢，再加上 CDN 判断缓存过期
CDN → 源站：If-None-Match: W/"1a2b3c"
源站：内容没变 → 304 Not Modified（响应头，无 body）
CDN：继续用本地副本 → 返回用户
```

这条链路只消耗源站**几十字节的响应头**，几乎零带宽。相比一刀切 `no-store` 每次全量回源，是数量级的差距。

### 3. 回源的其他工程细节

| 问题 | 做法 |
|------|------|
| 源站突发被 CDN 打穿 | **回源限速**（Nginx `proxy_limit_rate` / CDN 控制台"回源 QPS 限制"） |
| 单源不可靠 | **多源回源**（CDN 配置主备/轮询回源域名），源站挂一个不影响整体 |
| 回源走内网/专线 | 回源 Host 与公网访问域名可以不同（`proxy_set_header Host origin.example.com`） |
| 回源也需鉴权 | 打开 CDN "回源鉴权"（回源头带 Key），防止源站被知道 IP 的人直接绕过 CDN 访问 |
| 回源超时 | CDN 回源超时（默认约 10s），慢源站要改大；配合"失败自动切换回源" |
| 回源监控 | 盯回源率/回源耗时；回源率异常升高 = 缓存策略或 cache key 出了问题 |

---

## 六、鉴权与防盗链：从 Referer 到签名 URL

> 读完本章你能理解：Referer 防盗链为什么挡不住盗链，签名 URL 的正确实现和最容易写错的"字段顺序"，以及推流侧的鉴权。

### 1. 为什么 Referer 不够

Referer 防盗链（见下）只对"正经浏览器"有效——HTTP 头可以随便伪造，一个 `curl -e https://myapp.com` 就能绕过。**视频 URL 一旦被收录/转走，Referer 白名单形同虚设。** 真正挡住盗链的是**签名 URL（Signed URL）**：

- 内容 URL 带上**过期时间 + 签名**；
- CDN 边缘校验：过期 → 403；签名对不上 → 403；
- 签名只在你的后端能生成，盗链者没有 private key 造不出合法 URL。

### 2. 阿里云 CDN 鉴权 Type A 正确实现

> ⚠️ **最容易写错的点**：签名串的 `{rand}` 与 `{uid}` **位置别换个顺序**——阿里云规范是
> `MD5({uri}-{timestamp}-{rand}-{uid}-{privateKey})`，`uid` 在 `rand` 之后。
> 顺序写反，签名对不上，**所有 URL 在边缘 403**。不同厂商规则各不相同，对接时以厂商文档为准。

```go
// 生成阿里云 CDN Type A 签名 URL
// 签名串:        {uri}-{timestamp}-{rand}-{uid}-{privateKey}
// 最终 URL:      {uri}?auth_key={timestamp}-{rand}-{uid}-{md5hash}
// timestamp: unix 秒 (过期时间)   rand: 随机串   uid: 用户ID(可填"0")
func SignCDNURL(baseURL, privateKey string, ttl time.Duration) (string, error) {
	u, err := url.Parse(baseURL)
	if err != nil {
		return "", err
	}

	now := time.Now()
	timestamp := now.Add(ttl).Unix()      // 过期时间，单位秒
	rand := strconv.FormatInt(now.UnixNano(), 36) // 随机串（36 进制够用）
	uid := "0"                              // 无业务需要填 0 即可

	// ① 按规范拼签名串（注意字段顺序）
	sign := fmt.Sprintf("%s-%d-%s-%s-%s", u.Path, timestamp, rand, uid, privateKey)
	// ② MD5 生成 32 位 hex 签名
	hash := fmt.Sprintf("%x", md5.Sum([]byte(sign)))
	// ③ 拼进 query（是否从缓存键剔除，见 §4.4）
	q := u.Query()
	q.Set("auth_key", fmt.Sprintf("%d-%s-%s-%s", timestamp, rand, uid, hash))
	u.RawQuery = q.Encode()
	return u.String(), nil
}
```

**使用要点**：
- **有效期控制在 5~30 分钟**，前端每次播放前向后端接口取新 URL；太长等于把钥匙发出去；
- URL 由**后端生成**，前端页面实时获取——前端无法拿到 privateKey；
- 生成后**必须配合 CDN 侧"过滤 auth_key 参数"**（§4.4），否则命中率为 0。

**其他厂商的签名风格**（对接时认准各自文档）：

| 厂商 | 方案 | 特点 |
|------|------|------|
| 阿里云 CDN | Type A：`MD5(uri-ts-rand-uid-key)` | 需在控制台开启"URL 鉴权" |
| 腾讯云 CDN/LVB | `txSecret=MD5(key+StreamName+txTime)`，`txTime` 为 hex 时间戳 | 推拉流共用一套 |
| Cloudflare | Token Auth（40 字节 HMAC-SHA256）+ Signed Requests | 支持自定义路径/过期 |
| AWS CloudFront | CloudFront Signed URL / Cookies（AWS SigV4 + 带策略） | 需配置签发方 |

### 3. 推流鉴权（RTMP/WHIP 上行）

别只防播放侧、漏了推流侧——**任何人都能往你的流媒体服务推流占资源**。推流 URL 同样要签名：

```bash
# 腾讯云直播推流地址（签名在 URL 上）
rtmp://push.livepush.myqcloud.com/live/{StreamName}?txSecret=MD5(KEY+{StreamName}+txTime)&txTime=7f5a3b41

# SRS 自建（HTTP 回调鉴权，见第 15 篇）
# 推流前 SRS 回调你的后端，后端验证签名/白名单/账号，返回 0 允许 / 非 0 拒绝
```

自建流媒体服务器（SRS/MediaMTX/ZLMediaKit）的推流鉴权通常走**HTTP 回调**而不是 URL 签名：
- SRS：`on_publish` HTTP 回调；MediaMTX：`authHTTP` / JWT；ZLMediaKit：Hook 鉴权（见[第 16.5 篇](./16.5-ZLMediaKit实战.md)）；
- 回调里做三件事：查该流是否被授权、校验鉴权参数、限制单流/单用户并发路数。

---

## 七、直播上 CDN：两种接入模式

> 读完本章你能理解：直播内容上 CDN 有"推流到 CDN"和"自建源站 + CDN 回源"两条路，各自的适用场景和延迟特征。

### 模式 A：推流到云厂商直播 CDN（省心，推荐国内直播）

直接把自己的流推给阿里云/腾讯云的**直播产品**，转码、多码率、分发、鉴权全交给平台：

```
编码器/OBS ──RTMP/SRT──► [厂商直播入口] ──转码→ 多码率 ──► [CDN 边缘] ──► 用户
                             ▲ 专区号+推流签权                       
```

```bash
# 推流（RTMP 到厂商直播入口，URL 带推流鉴权）
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 2000k \
  -c:a aac -b:a 128k \
  -f flv "rtmp://push.live-push.aliyuncs.com/live/{StreamName}?auth_key={推流签名}"

# 播放（HLS / HTTP-FLV / LL-HLS 由厂商分发，前端直接拉）
# HLS:       https://your-domain.cdn.com/live/{StreamName}.m3u8
# HTTP-FLV:  https://your-domain.cdn.com/live/{StreamName}.flv
# WebRTC:    厂商 WHEP 出口（低延迟，见第 21 篇）
```

### 模式 B：自建源站 + CDN 回源（可控，适合已有 SRS/MediaMTX）

自己搭流媒体源站，CDN 只负责"缓存 + 分发"。此时 CDN 对直播的回源，本质是**回源拉 m3u8 + 切片**（§4 的缓存策略就是为此设计）：

```
推流器 ──RTMP──► [SRS/MediaMTX 源站] ──HLS m3u8+ts──► [CDN 边缘] ──► 用户
                      ▲                                       │ 缓存
                      └────────── CDN 回源（上行） ←──────────┘
```

- **延迟**：路径上多了"CDN↔源站"一跳，直播延迟比模式 A 略高，但对点播/监控/录像无关紧要；
- **LL-HLS**：追求低延迟时注意 LL-HLS 的分片 `.part` 不能被 CDN 缓存（§4.2 的处理），否则会拿到旧内容。

> 判定依据：**找不到转码需求、要省运维** → 模式 A；**已有源站、要保留控制权、面向监控/点播** → 模式 B。两种模式在 CDN 侧的鉴权、缓存、回源优化都可以照搬上面各节。

---

## 八、ABR 多码率 + CDN：让边缘只缓存"用户真正要看的那一档"

> 读完本章你能理解：多码率内容挂 CDN 时，缓存、预热、切片对齐分别要注意什么。

多码率的生成在[第 18 篇](./18-FFmpeg流媒体转码配置.md)已完整讲过，本节只讲 CDN 侧的三件事：

```
master.m3u8                 # CDN 缓存即可
 ├── 1080p/stream.m3u8      # 3000kbps
 ├── 720p/stream.m3u8       # 1500kbps
 └── 360p/stream.m3u8       #  500kbps
```

1. **各码率档位独立缓存**：CDN 把 `1080p/xxx.ts` 和 `360p/xxx.ts` 当成不同文件，天然各存各的。播放器只拉当前档位的切片，所以**不要对 master.m3u8 或低档位做全量预热**，否则预热的分片可能根本没人看。
2. **切片边界的编码对齐**：转码时各档位**使用相同的关键帧间隔**（`-g`、`keyint` 一致，见[第 18 篇](./18-FFmpeg流媒体转码配置.md)），这样：
   - 播放器切换码率时，时间戳能无缝衔接，不花屏；
   - CDN 按时间对齐的分片缓存，能整段复用。
3. **按需预热**：大直播开播时，预热对象是 **master.m3u8 + 首屏档位（通常 720p/360p）的前十几个分片**，而不是全部档位全时间轴：

```bash
# 伪代码：批量预热
for seg in {1..30}; do
  prewarm("https://xxx.cdn.com/hls/720p/seg_${seg}.ts")
done
```

---

## 九、选型与成本：CDN 怎么选、怎么算账

> 读完本章你能理解：CDN 选型看哪几个维度，以及"带宽计费 vs 流量计费"这个能差一倍钱的成本问题。

### 1. 选型维度

| 维度 | 说明 | 关注点 |
|------|------|--------|
| **节点覆盖** | 是否覆盖你的目标用户群 | 纯国内 vs 出海：覆盖方向完全不同 |
| **接入复杂度** | 域名接入方式、是否强制备案 | 国内厂商都要求 ICP 备案 |
| **调度精度** | GSLB/HTTPDNS 精细化程度 | 对首屏时间影响大 |
| **计费方式** | 见下 §2 | 直接影响月度账单 |
| **视频能力** | 直播转码、鉴权、预热、回源优化 | 直播业务尤其要确认"直播 CDN"而非纯静态 CDN |
| **生态** | S3/OSS/对象存储 + push/pull 联动 | 用哪家存储就选哪家 CDN，历史证明省事 |

常见厂商速查：

| CDN | 适合 | 特点 |
|-----|------|------|
| 阿里云 CDN / 阿里云直播 | 国内视频、点播+直播一体 | 与 OSS 联动好，鉴权/预热 API 完善 |
| 腾讯云 CDN / LVB | 国内直播、腾讯系生态 | 直播云产品成熟，端上播放器配套多 |
| 火山引擎 CDN | 国内点播、高性价比 | 带宽便宜，视频场景多 |
| Cloudflare | 出海、可接受稍贵 | 全球节点、免费套餐起步、边缘计算强（Workers） |
| AWS CloudFront | 出海点播、S3 存量 | 与 S3/CloudFront 无缝、按量计费、安全性强 |
| 自建 Nginx | 内网/私有化部署 | 无备案、无按量费，但无任何就近调度可言 |

### 2. 成本：带宽计费 vs 流量计费

这是 CDN 账单里**差距能到一倍**的问题，先看两种模式：

| 计费模式 | 怎么算 | 曲线形状 | 适合 |
|---------|--------|---------|------|
| **流量计费** | 每 GB 多少钱，用多少收多少 | 跟随真实流量 | 流量平稳、突发少的业务 |
| **带宽计费（95 峰值）** | 取当月每 5 分钟采样带宽值，**去掉最高 5% 后**，按剩余最高值 × 单价 | 只看"尖峰"，日常低谷免费 | 有明显高峰（晚间档、直播专场） |

**坑就在这里**：
- 带宽计费是按**峰值**收的——哪怕一个月只有 1 个小时冲上 10Gbps，那 1 小时的峰值决定了整月账单；
- 直播大促、热点事件瞬间冲高，**带宽计费会非常贵**；
- 反过来，流量很平稳的话，流量计费反而更省，因为流量计费不抬高单价。

**对策**：
- 业务有明显尖峰 → 选 **流量计费**（把瞬时爆发摊到句子里，比按峰值计价便宜）；
- 业务平稳 → 两者差不多，按单价比较；
- 无论哪种，**热点自动或手动预热、命中率上 95%+**，才是省钱的根——回源带宽通常是 CDN 套餐外的独立费用。

### 3. 要盯的监控指标

| 指标 | 健康值 | CDN 控制台/日志里怎么看 |
|------|--------|------------------------|
| 缓存命中率 | > 95%（点播） | 控制台「命中率」报表，分域名/分 URL |
| 回源率 | 越低越好 | 回源报表；回源率突增 = cache key/响应头异常 |
| 首帧时间 / 播放卡顿率 | 越小越好 | 需播放器上报 + 视频质量监控平台 |
| 单 URL 请求量排名 | 识别热点、预热依据 | 访问日志分析 |
| 回源带宽 | 看是否被回源挤爆 | 回源带宽报表 |

---

## 十、边缘计算：把简单逻辑搬到用户身边

> 读完本章你能理解：边缘 Worker 能替源站挡掉哪些请求，以及"别把重活放边缘"。

CDN 边缘节点不止能缓存，还能**执行一段代码**（Edge Worker/Cloudflare Workers/阿里云边缘程序），对请求做前置处理：

```
用户 ──► [边缘节点] ──处理──► [源站/CDN 缓存]
              ▲ 可以在此处理：
              │  • 鉴权校验（参考签名 URL，见 §6）
              │  • 防盗链（Referer/UA/频控）
              │  • URL 重写（版本号、A/B 流量）
              └─ • 简单聚合/注入响应头
```

Cloudflare Workers 防盗链示例（校验 Referer，命中白名单才回源）：

```javascript
addEventListener('fetch', (event) => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const referer = request.headers.get('Referer') || '';
  const allowed = ['https://myapp.com', 'https://www.myapp.com'];

  // Referer 可伪造 → 只作第一道关卡，真正的门槛是签名 URL
  if (!allowed.some((d) => referer.startsWith(d))) {
    return new Response('Forbidden', { status: 403 });
  }
  return fetch(request);
}
```

**边界提醒**：
- 边缘适合**无状态、毫秒级、不涉及私有数据的逻辑**（鉴权缓存布尔值、URL 重写）；
- 边缘**不适合**重计算（视频转码、水印合成、AI 推理）——那应该在源站侧用专用服务做（见[第 35 篇](../03-行业应用场景/35-AI实时检测集成.md)），否则边缘节点 CPU 成本会反过来吃掉省下的带宽费。

---

## 十一、动手实践

> 3 个练习按顺序做，覆盖"源站配置 → 命中验证 → 签名"三个环节。第 4 个是点播 S3+CloudFront 的完整链路（对应大纲要求的实践）。

### 练习 1：源站 Nginx 配置带 304 校验 + 分层缓存

```nginx
server {
    listen 80;
    server_name origin.example.com;
    etag on;                     # 开启 ETag，让 no-cache 能走 304

    location /hls/ {
        root /var/www;
        # 直播 m3u8：允许存，但强制回源校验（304）
        location ~* \.m3u8$ {
            add_header Cache-Control "no-cache";
        }
        # TS 切片：写后不可变，按切片时长缓存
        location ~* \.ts$ {
            add_header Cache-Control "public, max-age=6";
        }
        add_header Access-Control-Allow-Origin *;
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }
    }
}
```

```bash
# 验证：第一次 200，第二次带 If-None-Match 应返回 304
curl -D - http://origin.example.com/hls/stream.m3u8 | grep -iE "etag|cache"
# 再取 ETag 后手动校验
curl -H "If-None-Match: <etag>" -D - http://origin.example.com/hls/stream.m3u8 | grep -iE "HTTP|etag"
# 期望输出：第二次 HTTP/1.1 304 Not Modified，无 body
```

### 练习 2：验证命中率，并复现"签名 URL 打爆缓存"

```bash
# ① 普通 URL 连续请求两次 → 第二次应 HIT
curl -I "https://cdn.example.com/hls/stream_00001.ts" | grep -iE "x-cache|age"
# X-Cache: MISS（第一次）  →  X-Cache: HIT, Age: 2（第二次）

# ② 带 auth_key 再连打两次 → 每次都 MISS！缓存键被签名参数炸了
curl -I "https://cdn.example.com/hls/stream_00001.ts?auth_key=1-abc-0-xxx1" | grep -i x-cache
curl -I "https://cdn.example.com/hls/stream_00001.ts?auth_key=2-abc-0-xxx2" | grep -i x-cache
# 两次都是 X-Cache: MISS —— 这就是 §4.4 说的大坑
# 然后：到 CDN 控制台打开「过滤参数/忽略 query」，再跑 ① ，第二次应 HIT
```

### 练习 3：Go 生成签名 URL（阿里云 Type A，顺序已校正确）

```go
package main

import (
	"crypto/md5"
	"fmt"
	"net/url"
	"time"
)

// 签名规则见 §6.2：MD5(uri-timestamp-rand-uid-privateKey)，
// 字段顺序必须与厂商文档完全一致，uid 在 rand 之后。
func SignCDNURL(baseURL, privateKey string, ttl time.Duration) (string, error) {
	u, err := url.Parse(baseURL)
	if err != nil {
		return "", err
	}
	now := time.Now()
	timestamp := now.Add(ttl).Unix()                       // 过期（秒）
	rand := fmt.Sprintf("%08x", now.UnixMicro())           // 随机串
	uid := "0"

	sign := fmt.Sprintf("%s-%d-%s-%s-%s", u.Path, timestamp, rand, uid, privateKey)
	hash := fmt.Sprintf("%x", md5.Sum([]byte(sign)))
	q := u.Query()
	q.Set("auth_key", fmt.Sprintf("%d-%s-%s-%s", timestamp, rand, uid, hash))
	u.RawQuery = q.Encode()
	return u.String(), nil
}

func main() {
	u, err := SignCDNURL("https://cdn.example.com/hls/stream_00001.ts",
		"my-secret-key", 30*time.Minute)
	if err != nil {
		panic(err)
	}
	fmt.Println(u)
	// https://cdn.example.com/hls/stream_00001.ts?auth_key=1798001234-xxxx-0-<32位hex>
}
```

**预期结果**：`auth_key` 过期或签名错误时，CDN 边缘直接返回 403；有效期内正常播放。

### 练习 4：S3 + CloudFront 部署点播 HLS（完整链路）

```bash
# ① 上传 HLS 点播文件到 S3（m3u8 也上传，是静态文件，可缓存）
aws s3 cp ./vod/ s3://my-video-bucket/vodm/ --recursive --content-type "application/vnd.apple.mpegurl" --no-guess-mime-type

# ② 建 CloudFront Distribution（源站选 S3，允许 GET/HEAD 与 HTTP 方法）
aws cloudfront create-distribution \
  --origin-domain-name my-video-bucket.s3.amazonaws.com \
  --default-root-object index.html

# ③ 客户端访问
# https://d1234abcd.cloudfront.net/vodm/master.m3u8
```

**要点**：
- S3 直接给 CDN 回源，天然带 ETag → 304 免费；
- `header Cache-Control` 在 S3 对象元数据里设置（`aws s3 cp ... --metadata cache-control="public,max-age=86400"` 对切片，`no-cache` 对 m3u8）；
- CloudFront 侧关掉 `Query String Forwarding`（等同"过滤参数"），避免 `auth_key` 打爆命中率。

---

## 十二、常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| m3u8 能播、.ts 全部 403 | CDN 防盗链配了 Referer 白名单，播放域名没加入 | 播放页域名加白名单；优先改用签名 URL（§6） |
| 直播延迟越来越大 | m3u8 被 CDN 缓存了旧片段列表 | 确认 m3u8 响应头 `no-cache`（§4.2），而不是 `no-store`+无 ETag |
| 命中率接近 0，回源率暴涨 | 签名 `auth_key` 混进了缓存键（§4.4） | CDN 配置忽略 query/过滤参数 |
| 命中率正常但源站带宽还是高 | 回源单位拉整文件，没用 Range（§5.1） | 源站/Nginx 开启 Range；切片级回源 |
| 拖进度条特别慢/首播 2 分钟才出来 | 点播 MP4 被边缘整文件缓存，seek 要整个文件 | 用 HLS/切片，或开启 mp4 Range 模块（§5.1） |
| 跨域播放失败 | 源站或 CDN 没配 CORS 头 | 源站与 CDN 都加 `Access-Control-Allow-Origin` |
| 全球节点延迟差异大 | PoP 覆盖不足/调度不生效 | 换覆盖目标地区的厂商；内容多地区预预热 |
| 签名 URL 泄露被盗播 | 有效太长/前端能拿到私钥 | 有效期 5-30min；签名只由后端生成 |
| 大促瞬间源站被打挂 | 热点内容没预热 + 回源无 QPS 限制 | 开播前预热（§4.5）；回源限速（§5.3）；多源兜底 |
| 账单比预估高一倍 | 95 峰值带宽计费撞上瞬时高峰 | 换成流量计费或削峰（§9.2） |
| LL-HLS 边缘拿旧分片 | `.part` 被 CDN 缓存了 | `.part` 用 `no-store`（§4.2） |

---

## 本篇小结

| 场景 | CDN 策略 |
|------|---------|
| 直播 | 推流签名 → CDN 直播（或自建源站 + 回源）；m3u8 `no-cache`+304，切片按分片时长缓存 |
| 点播 | 整文件/切片长缓存；master.m3u8 可缓存；预热热门内容 |
| 监控/内网 | 通常不需要外部 CDN，Nginx 直分发即可 |
| 出海 | CloudFront/Cloudflare + S3/OSS + 签名 URL |

画一张图记住本章：

```
内容 →(m3u8不缓存/304, 切片按切片时长缓存, MP4长缓存, .part不缓存)
      →(Range 分片回源, 304 免带宽, 多源+QPS限流防打穿)
      →(签名URL防盗链, auth_key 从缓存键剔除)
      →(预热热点) → 命中率>95%、回源率<5%、账单可控
```

核心原则：**能缓存的分片都缓存，会变的 m3u8 用 304 校验，签名只管鉴权别碰缓存键。**

---

## 关键术语速查表

| 术语 | 英文 | 说明 |
|------|------|------|
| CDN | Content Delivery Network | 内容分发网络，把内容缓存到各地边缘节点 |
| PoP | Point of Presence | 边缘节点机房，用户就近接入 |
| GSLB | Global Server Load Balancing | 按用户归属地把请求调度到最近/最空闲节点 |
| HTTPDNS | — | 绕过 Local DNS 的强调度，避免被劫持/乱缓存 |
| 回源 | Origin Pull | 缓存未命中时向源站拉取内容 |
| 命中率 | Cache Hit Rate | CDN 直接响应（未回源）的比例 |
| 回源率 | Origin Pull Rate | 请求真正打到源站的比例 |
| Cache Key | 缓存键 | CDN 判定"是不是同一个内容"的依据，默认含 query string |
| 预热 / 预推送 | Pre-warming / Push | 内容上架前主动分发到各边缘节点 |
| TTL | Time To Live | 缓存存活时间 |
| 强缓存 / 协商缓存 | Strong / Conditional Cache | `max-age` 直接返回 vs `no-cache`+ETag 回源校验(304) |
| Range Request | 范围请求 | 只取文件的字节区间，视频 seek 的基础 |
| 304 Not Modified | 304 / Not Modified | 内容没变，仅回响应头，免去 body 下载 |
| 签名 URL | Signed URL | 带过期时间与签名的 URL，CDN 校验后才放行 |
| 95 峰值计费 | 95th Percentile Billing | 按去掉最高 5% 采样后的峰值带宽计费 |

---

## 参考资料

- 阿里云 CDN 文档 · URL 鉴权（Type A/Type B）：https://help.aliyun.com/zh/cdn/user-guide/url-signature
- 阿里云 CDN 文档 · 过滤参数（缓存键配置）：https://help.aliyun.com/zh/cdn/user-guide/ignore-query-string
- 腾讯云 CDN 鉴权配置（txSecret/txTime）：https://console.cloud.tencent.com/cdn/security（控制台路径）
- Cloudflare · Signed URLs / Token Authentication：https://developers.cloudflare.com/cache/how-to/enable-token-authentication/
- AWS · CloudFront Signed URLs & Cache Key（Query String）：https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html
- Apple · HLS Authoring Specification（CDN 缓存建议）：https://developer.apple.com/streaming/
- rfc9110 · HTTP 缓存语义（Cache-Control / 304 / Range 的标准定义）：https://www.rfc-editor.org/rfc/rfc9110.html

---

> **全文完** | 上一篇：[第 21 篇 — 低延迟播放完整方案](21-低延迟播放完整方案.md) | 下一篇：[第 22.5 篇 — 流媒体安全：鉴权、防盗链与 DRM](22.5-流媒体安全鉴权防盗链与DRM.md)