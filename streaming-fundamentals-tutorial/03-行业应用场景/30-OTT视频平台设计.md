# OTT 视频平台设计（Netflix/YouTube 风格）

> **适合读者**：构建大规模点播（VOD）平台的架构师和后端开发者  
> **前置知识**：[第 10 篇：HLS 协议详解](../01-底层技术基础/10-HLS协议详解.md)、[第 11 篇：DASH 与 CMAF](../01-底层技术基础/11-DASH与CMAF统一分片格式.md)、[第 18 篇：FFmpeg 多码率转码](../02-工程实践优化/18-FFmpeg流媒体转码配置.md)、[第 22 篇：CDN 分发](../02-工程实践优化/22-CDN分发与边缘加速.md)、[第 22.5 篇：鉴权与 DRM](../02-工程实践优化/22.5-流媒体安全鉴权防盗链与DRM.md)  
> **核心问题**：同一部片子，怎么做到"4K 电视、手机、弱网用户"都能流畅看？又怎么让 100 万并发首播不把源站打爆？

---

## 一、OTT 的工程本质：一次上传，四个阶段

读完本节，你能说清 OTT 点播与直播在工程上的根本差异，并建立全链路地图。

OTT（Over-The-Top）指**绕过传统运营商、直接走互联网分发**的视频服务，技术形态是**点播（VOD）**。和直播（第 28 篇）不同，点播内容**预先录制好、随时可看**——这个"预先"是 YouTube/Netflix 一切工程选择的起点。

| 维度 | 直播（Live） | OTT 点播（VOD） |
|------|-------------|----------------|
| 内容产生 | 实时产生，边采集边分发 | 预先录制，可反复加工 |
| 转码 | 实时、低延迟、低档位 | **离线转码农场**，几百档任意精算 |
| 用户到达 | 分摊到整场直播 | **同时刻百万级突刺**（新片首播） |
| 峰值预测 | 提前约 20 分钟热场 | **上架即峰值**，必须提前准备 |
| 看一段 | 播过的无法回看（除非另录） | 任意拖动进度条 |

整条链路四个阶段：

```
上传接入 → 转码/打包 → CDN 分发 → 播放器 ABR
   │           │          │          │
 断点续传     一条原片    热门内容    按网速
 前端直传     N 份产物     预推边缘    自动选档
```

> **一句话认知**：OTT 的成本几乎全来自"一条原片 → 多份产物"这一步。为了兼容不同网络（码率）和设备（编码器），系统把同一内容复制成 **编码器 × 分辨率 × 封装** 的产物矩阵，后续存储、转码、CDN 成本全部由它决定。本篇二~四节讲怎么克制这份矩阵，五节讲怎么用好它。

---

## 二、上传接入：断点续传 + 前端直传

读完本节，你能设计"大文件上传不占业务带宽、断网能续"的接入方案。

上传是整条链路最"业务无关"的部分，但做错会拖垮后端。核心设计两条：

1. **前端直传对象存储，不经过业务后端**：后端只签发**预签名 URL**（预签名 URL 只有指定时长内有效、只对指定 key 有效），浏览器/客户端直接 PUT 给 OSS/S3，避免视频流量压垮 Go 服务。
2. **分片 + 断点续传**：大文件切成 5-50MB 分片逐个上传，失败只重传该分片。

```
客户端          业务后端               对象存储(OSS/S3)
  │  ① 申请上传(文件名/大小)  →       │
  │  ② 返回 upload_id + 首个预签名URL  │
  │  ③ 对第 N 片要预签名URL  ←────────┼── 最多签一个 URL，控制成本
  │  ④ 分片若无签名/PUT 直传 OSS  ────→ 直接落对象存储
  │  ⑤ 全部传完 → CompleteUpload       │
  │         └─ 后端合并分片 → 触发转码 ─┘
```

```go
// ① 初始化：创建 OSS 分片上传任务，返回 upload_id
func (s *VideoService) InitMultipartUpload(ctx context.Context, req dto.UploadInitRequest) (*dto.UploadInitResponse, error) {
    uploadID, err := s.storage.CreateMultipartUpload(ctx, req.ObjectKey)
    if err != nil {
        return nil, xerr.Wrap(xerr.ErrInternal.Code, "创建上传任务失败", err)
    }
    // 只签发第一个分片的 URL：其余等到客户端上报"即将传第 N 片"再逐个签发
    firstURL, err := s.storage.PreSignPartUpload(ctx, uploadID, 1, 15*time.Minute)
    return &dto.UploadInitResponse{UploadID: uploadID, FirstPartURL: firstURL}, err
}

// ② 客户端传出第 N 片时，取第 N 片的预签名 URL（前端直传，不经后端）
func (s *VideoService) GetPartUploadURL(ctx context.Context, uploadID string, partNumber int) (string, error) {
    return s.storage.PreSignPartUpload(ctx, uploadID, partNumber, 15*time.Minute)
}

// ③ 全部传完：合并分片 → 落库 → 触发离线转码
func (s *VideoService) CompleteUpload(ctx context.Context, uploadID string, parts []storage.Part) error {
    if err := s.storage.CompleteMultipartUpload(ctx, uploadID, parts); err != nil {
        return xerr.Wrap(xerr.ErrInternal.Code, "合并分片失败", err)
    }
    return s.transcodeSvc.Enqueue(ctx, uploadID) // 进队列，见第三节
}
```

要点：分片并行上传 + 失败重试放客户端做；服务端永远不要代传大文件。

---

## 三、转码农场：一条原片 → 多码率 × 多编码器

读完本节，你能设计从原片到"产物矩阵"的转码流水线，并理解为什么 AV1 值得做。

### 1. 编码阶梯（Transcode Ladder）

观众网络从 4K 光纤到 4G 弱网，差别两个数量级。转码的目标是产出**一组分辨率/码率逐级递增的档位**，让播放器随时切换：

| 档位 | 分辨率 | H.264 码率 | AV1 码率 | 定位 |
|------|--------|-----------|---------|------|
| 4K | 3840×2160 | 16 Mbps | 8 Mbps | 旗舰电视 |
| 1080p | 1920×1080 | 6 Mbps | 3 Mbps | 默认高清 |
| 720p | 1280×720 | 3 Mbps | 1.5 Mbps | 中端网络 |
| 480p | 854×480 | 1.5 Mbps | 0.8 Mbps | 移动弱网 |
| 360p | 640×360 | 0.8 Mbps | 0.4 Mbps | 兜底 |

**AV1 的价值**（第 2 篇讲了压缩原理）：同画质码率约为 H.264 的一半 → **带宽成本砍半**。代价是编码极慢（需 SVT-AV1 多核或硬件），且老设备不支持硬解。OTT 的通常做法是双编码轨：**H.264 做全设备兼容兜底 + AV1 给支持硬解的现代设备**。响号大、偏新片首发的平台还可给 AV1 更高配比。

### 2. 任务编排：队列 + Worker 池

转码是**纯 CPU/GPU 计算**，一个 2 小时 4K 片源可算出分钟级别的耗时——绝对不能放在上传请求里同步等。正确形态是**消息队列驱动、独立转码工人池**：

```
上传完成 → 写 job 表(pending) → 发布队列消息
                                   ↓
                           N 个转码 Worker 消费
                          ┌─────────┴─────────┐
                          ├ 拉原片 → 多路并行编码 → 上传产物 → 标记 done
                          └ 失败 → 重试(指数退避) → 仍失败标记 failed + 告警
```

Go 侧只需管理状态机（底层的多码率 FFmpeg 命令直接用[第 18 篇](../02-工程实践优化/18-FFmpeg流媒体转码配置.md)的脚本，这是转码配置的完整实战）：

```go
type TranscodeStatus string // pending → processing → done / failed
const (
    TranscodeStatusPending    TranscodeStatus = "pending"
    TranscodeStatusProcessing TranscodeStatus = "processing"
    TranscodeStatusDone       TranscodeStatus = "done"
    TranscodeStatusFailed     TranscodeStatus = "failed"
)

// 上传完成后入队（不阻塞请求）
func (s *TranscodeService) Enqueue(ctx context.Context, videoID string) error {
    if err := s.repo.UpdateStatus(ctx, videoID, TranscodeStatusProcessing); err != nil {
        return err
    }
    msg, _ := json.Marshal(TranscodeJob{VideoID: videoID, Ladder: s.cfg.Ladder})
    return s.mq.Publish(ctx, "transcode.jobs", msg)
}

// Worker 消费端（异步进程），产物上传完更新最终状态
func (s *TranscodeService) RunTranscode(ctx context.Context, job TranscodeJob) error {
    if err := s.ffmpeg.RunLadder(ctx, job, s.storage); err != nil {
        s.repo.UpdateStatus(ctx, job.VideoID, TranscodeStatusFailed)
        return xerr.Wrap(xerr.ErrInternal.Code, "转码失败", err)
    }
    return s.repo.UpdateStatus(ctx, job.VideoID, TranscodeStatusDone)
}
```

> 为什么不在 Go 进程里 `go func(){ doTranscode() }()`？进程重启任务就丢了、无法并发限流、无法横向扩容 —— OTT 的转码量必须落到**持久化队列 + 独立 Worker**。

---

## 四、CMAF 统一打包：一份产物，HLS / DASH 通吃

读完本节，你能把产物矩阵的存储量减半，并理解它为何同时提升 CDN 命中率。

第二节说产物矩阵按"编码器 × 分辨率"膨胀。还有一个隐藏的膨胀因子：**封装格式**。如果一个平台同时要 HLS（第 10 篇）和 DASH（第 11 篇），传统做法是各打包一份 —— HLS 用 TS 分片、DASH 用 fMP4 分片，**存储和 CDN 缓存全部翻倍**。

CMAF（Common Media Application Format）解决它：**用同一份带标准分片边界（chunk）的 fMP4，同时服务 HLS v7 和 DASH**。

```
传统双打包：  H.264×5档 × {TS给HLS, fMP4给DASH} = 10 份
CMAF 单打包： H.264×5档 × {fMP4 两者共用}        = 5 份   → 存储减半
```

除存储外还有一个容易忽略的好处：**CDN 缓存命中率翻倍**。HLS 和 DASH 客户端现在请求的是同一批 `.m4s` 文件，冷门内容无论用哪个协议看，首次回源后就被两个协议的后续请求共享。

打包本身不改变转码结果，只是输出参数（`-hls_segment_type fmp4`），在动手实践里会现场验证。CMAF 的 `cbcs` 加密模式还能让一份密文被三种 DRM 解密（见第七节，详参[第 22.5 篇](../02-工程实践优化/22.5-流媒体安全鉴权防盗链与DRM.md)）。

---

## 五、ABR 自适应码率：播放器是怎么"自动选画质"

读完本节，你能讲清三种主流 ABR 算法各自的取舍，并知道哪些坑其实出在服务端。

ABR（Adaptive Bitrate，自适应码率）是整个 OTT 播放体验的心脏。**服务端只负责"把多档摆出来"，选哪一档是播放器每 4~6 秒（一个分片）做一次的事**。整个过程三步：

```
① 带宽估计：统计最近 N 个分片的下载速率（
   下载耗时 < 分片时长 → 带宽够用；超时 → 不够）
② 选档决策：在"画质更高"与"别卡顿"之间选择
③ 切档：无缝切到新档（依赖第三节说的分片对齐）
```

### 三种主流决策算法

| 算法 | 怎么估计带宽 | 怎么选档 | 优点 | 缺点 |
|------|------------|---------|------|------|
| **吞吐量法** | 最近分片的平均下载速率 | 选 ≤ 估计带宽 × 0.8（留余量）的最高档 | 简单、直观，hls.js 默认 | 抖动剧烈时反应慢，无法预防卡顿 |
| **BOLA** | 不测带宽，只看**播放缓冲区水位** | 缓冲高→升档，缓冲低→降档 | 有数学保证（Lyapunov），抗抖档、天然防卡顿 | 冷启动（缓冲空）阶段偏保守 |
| **MPC** | 历史吞吐序列 + **对未来预测** | 用 QoE 模型（画质+卡顿+切档成本）对后续几片做最优规划 | 决策全局更优、轨迹平滑 | 复杂，需多步预下载采样；高端播放器才用 |

> 工程结论：**绝大多平台用"吞吐量 + 缓冲区"混合**：带宽估计决定可选上限，缓冲区水位决定要不要实际升档；缓冲区见底则无条件降档。纯理论算法（BOLA/MPC）是权威实现（dash.js、Shaka）里的可选项，不必自研。

### 服务端真正要做的四件事（做错播放器再聪明也没用）

1. **BANDWIDTH 标注必须是峰值而非平均**：master m3u8 里 `#EXT-X-STREAM-INF:BANDWIDTH=6000000` 写的是该档**峰值码率**（含音频与复用开销）。写小了播放器会选档过低、写大了会切上去就卡。
2. **各档分片对齐 + 关键帧对齐**：所有档用相同的 `-g`/切分边界，ABR 切档才能无缝。这是转码脚本约束（第 18 篇），不是播放器的事。
3. **档位数量克制**：每加一档 = 转码 + 存储 + CDN 成本。5~6 档足够覆盖 90% 网络，没必要做成 Netflix 的几十档。
4. **不要自己发明切档逻辑**：hls.js / dash.js 已带完备 ABR。除非你的播放器有强定制需求，否则改 ABR 是负优化起点。

```
master.m3u8（服务端产物，播放器据此做 ABR）
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=6288000,RESOLUTION=1920x1080,CODECS="avc1.640029,mp4a.40.2"
1080p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3188000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=898000,RESOLUTION=640x360,CODECS="avc1.42c01e,mp4a.40.2"
360p/index.m3u8
```

---

## 六、CDN 分发与"两级预加载"

读完本节，你能区分"全网预热的 CDN 预推"和"给单个用户的下一集预加载"两条不同手段。

第 22 篇已详述 CDN 的缓存策略与回源优化，本节只讲 OTT 特有的两个动作：**热内容预推** 和 **看片前预加载**。它们的共同敌人是第一节说的"上架即峰值"：第一个用户不该替所有人承担一次回源。

### 第一级：热门内容预推（谁下架都需要的动作）

新片全网首发、热门剧更新，是**可预知的流量尖峰**。做法：上架/生成产物时，调用 CDN 厂商的预热 API，把 `master.m3u8` 与首批分片 URL 批量提交，CDN **主动回源填满各边缘节点**，开播即 HIT（第 22 篇 §四.5 的完整示例）。

Netflix 的极端形态是 **Open Connect**：自研内容分发设备（OCA）直接放进各家 ISP 机房，热门剧提前落在离用户一跳的网络里——足够大的流量让"自建 CDN"反而更省。中小平台买商业 CDN 预热 API 即可，别碰自建。

### 第二级：个性化预加载（"下一集"秒开）

这是 OTT 特有的心机手法。观众看到第 3 集 80% 时，系统已经知道**大概率要看第 4 集**——提前把下一集的 manifest + 前几个分片拉到他所在的边缘/设备上，点"下一集"时**无需冷启动**。

```
观看进度事件(≥60% 该集时长)
        │
        ├── 后端查：是否有"NEXT"（系列剧才有）
        │      ├─ 无 → 忽略（电影/单集）
        │      └─ 有 →
        │           ├─ 告诉 CDN 预取下一集 manifest + 首几个分片（发往最近节点）
        │           └─ 播放器空闲时预解析 manifest、预拉起首分片到本地缓冲
        ▼
用户点"下一集" → manifest/分片已在手边 → 秒开
```

```go
// 观看进度上报 → 决定是否触发"下一集预加载"
func (s *PlaybackService) OnProgress(ctx context.Context, userID, videoID string, posMs, durMs int64) error {
    // 系列剧且播放到 60% 以上，才开始预加载；低于此值的冷门片不预取，防浪费
    if float64(posMs)/float64(durMs) < 0.6 {
        return nil
    }
    next, ok, err := s.repo.NextEpisode(ctx, videoID)
    if err != nil {
        return err
    }
    if !ok {
        return nil // 没有下一集（电影或已到最终话）
    }
    log.Info().Str("user", userID).Str("next", next.ID).Msg("预加载下一集")
    // 预取的是"分发层"：manifest + 前几个分片，不是"重新转码"
    return s.cdn.Preload(ctx, next.ManifestURL, next.FirstSegments)
}
```

> 预加载的边界：**只预分发，不预转码**。转码仍按上架队列统一跑；预加载针对的是"播放那一刻的冷启动"。个性化预加载对非系列内容（电影）是无用的，判断条件（有无 NEXT）是它的开关。

---

## 七、内容保护：按内容价值选 DRM 档位

读完本节，你能为 OTT 内容定出"上到什么程度的安全"。

完整的鉴权、AES-128 HLS、Widevine/FairPlay/PlayReady 三体系和 License 服务实现，**全部在第 22.5 篇**，这里只给 OTT 特有的决策表。原则：**按内容价值递增投入，不要一步到位**。

| 内容价值 | 推荐组合 | 说明 |
|---------|---------|------|
| 顶级版权影视（Netflix 级） | **DRM 三体系 + License 服务 + CMAF cbcs** | 版权方硬性要求；一份 cbcs 密文三体系共享，转码/存储不翻倍 |
| 中高价值（赛事、新片首发） | DRM + 密钥即指纹 + 隐式水印 | 防盗录搬运可追溯 |
| 付费课程 / 一般片库 | HLS AES-128 + 签名 URL + 显式水印 | 成本低，防顺手盗链与分享 |
| 免费 / 公开内容 | CDN 签名 URL 防盗链即可 | ≈0 成本 |

OTT 特有的两个注意点：

1. **地区许可约束**：版权内容常按地区销售，License 服务或 CDN 按 IP/地区拒绝签发——热映期对"哪里的 IP 能看哪部片"的校验，比画质优化更容易被业务追问。
2. **兼容性测试是运营项**：DRM 三家体系 × 浏览器 × 机型 × 有无 TEE（L1/L3）的矩阵是真金白银的测试成本，上线前留预算。

---

## 动手实践

### 练习1：MinIO + FFmpeg 小型 OTT 平台（本篇核心）

搭一个跑通"上传 → 多码率转码 → CMAF 打包 → 本地播放"的最小 OTT 平台，验证 ABR 全链路。

```bash
# 1. 启动 MinIO（S3 兼容存储，模拟 OSS/S3）
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin \
  minio/minio server /data --console-address ":9001"

# 2. 建桶 + 上传原片（模拟"前端直传对象存储"）
mc alias set local http://localhost:9000 minioadmin minioadmin
mc mb local/ott-videos
mc cp sample.mp4 local/ott-videos/source/sample.mp4   # 短视频源，4K 档很慢可先去掉

# 3. 转码：多档位 CMAF HLS（H.264），产物在 rend/ 下
ffmpeg -y -i sample.mp4 \
  -map 0:v:0 -map 0:v:0 -map 0:v:0 -map 0:a:0 \
  -c:v libx264 -preset veryfast \
  -b:v:0 6000k -maxrate:v:0 6000k -bufsize:v:0 12000k -s:v:0 1920x1080 \
  -b:v:1 3000k -maxrate:v:1 3000k -bufsize:v:1 6000k  -s:v:1 1280x720 \
  -b:v:2 800k  -maxrate:v:2 800k  -bufsize:v:2 1600k  -s:v:2 640x360 \
  -c:a aac -b:a 128k \
  -g 120 -keyint_min 120 -sc_threshold 0 \              # 关键帧固定 4s → 各档对齐
  -var_stream_map "v:0,a:0 v:1,a:0 v:2,a:0" \
  -hls_time 4 -hls_playlist_type vod -hls_segment_type fmp4 \
  -hls_master_playlist_name master.m3u8 \
  -hls_segment_filename "rend/%v/seg_%03d.m4s" \
  -f hls "rend/%v/index.m3u8"

# 4. 本地提供播放（生产 = 源站/CDN；先验证码用 python 静态服务器）
python3 -m http.server 8080 --directory rend
# VLC 打开 http://localhost:8080/master.m3u8 → 手动切画质 / 限速观察自动切档
```

**预期输出**：`rend/` 下有 `master.m3u8` + `0/1/2/` 三个子目录（各含 `index.m3u8` 与 `.m4s` 分片）；VLC 能播且能切换 1080p/720p/360p 三个档位。若浏览器 MSE 报 MIME 错，换成 Caddy/nginx 静态服务即可（VLC 不受影响）。

> 练习延伸：把 `rend/` 上传回 MinIO（`mc cp -r rend local/ott-videos/`）并把桶设为公开读 → 产物就能被任意设备拉取，这等于"对象存储即源站"的最小版。

### 练习2：验证 CMAF 复用 + 观察 ABR 切档

```bash
# a) 确认产物确实是 fMP4 且 HLS/DASH 可共用（同一份 m4s）
file rend/0/seg_000.m4s          # 看到 ISO Media（fMP4）
# 用 VLC 播放 master.m3u8，在"播放→渲染→网络"里限速：
#    从 1080p 降到 720p 再升回，观察是否无缝（无黑屏/花屏 = 分片对齐生效）

# b) 检查 master 的 BANDWIDTH 标注（应为峰值，见第五节）
cat rend/master.m3u8
```

**预期输出**：分片是 fMP4 而非 `.ts`；限速时播放器自动降档、恢复后自动升档，**切换瞬间不破声不黑屏** —— CMAF 复用与关键帧对齐两个特性的直接验证。

### 练习3：下一集预加载的触发逻辑（Go 单元测试）

```go
func TestPreloadNextEpisode(t *testing.T) {
    svc := NewPlaybackService(repo: fakeRepo..., cdn: &fakeCDN{})

    // 进度 50% → 不预加载
    svc.OnProgress(ctx, "u1", "ep3", 150e3, 300e3)
    if cdn.preloded != 0 { t.Fatal("未到阈值不应预加载") }

    // 进度 80% 且有下一集 → 预加载 ep4
    svc.OnProgress(ctx, "u1", "ep3", 240e3, 300e3)
    if cdn.preloaded != "ep4" { t.Fatal("应预加载下一集 ep4") }

    // 是电影（无 NEXT）→ 即使 100% 也不预加载
    svc.OnProgress(ctx, "u1", "movie", 120e3, 120e3)
    if cdn.preloaded != "ep4" { t.Fatal("无下一集不应触发") }
}
```

**预期输出**：三个分支断言全过——不到阈值不加载、到阈值加载下一集、无 NEXT（电影/最终话）即使看完也不加载。

### 练习4：VMAF 转码质量验收

```bash
# 对标"AV1 码率减半"：同码率下对比 H.264 与 AV1 的画质
ffmpeg -i original.mp4 -i transcoded_av1.mp4 \
  -lavfi "[0:v][1:v]libvmaf=log_path=vmaf.json:log_fmt=json:n_threads=4" -f null -

python3 -c "
import json
d = json.load(open('vmaf.json'))
print(f\"VMAF 平均分: {d['pooled_metrics']['vmaf']['mean']:.2f}\")
"
```

**预期输出**：AV1 用 H.264 一半码率时 VMAF 仍 ≥ 90（若 < 85 说明码率给太低）——这就是第五节讲 AV1 省带宽的量化依据。

---

## 常见问题排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| 播放器频繁在 1080↔720 来回跳 | 带宽估计噪声大 + 播放器未配迟滞阈值 | 播放器配置最小切档间隔与迟滞；同时检查 master 的 BANDWIDTH 是否写了均值 |
| 切档瞬间黑屏/花屏 | 各档 **GOP/分片未对齐** | 转码统一 `-g` + `-sc_threshold 0`，保证切分边界一致 |
| 分片上传完文件损坏 | CompleteMultipartUpload 的 Parts 顺序错误 | 按 PartNumber 升序提交流程，前端别乱序提交 |
| 新片全网首播首帧卡顿 | 未做**上架预热**，第一个用户承担回源 | 上架时调 CDN 预热 API 预推 master + 首批分片 |
| 低端安卓机播 AV1 黑屏 | 硬件不支持 AV1 解码 | 编码阶梯保留 H.264 兜底档，按设备能力下发 AV1/H.264 |
| 存储成本比预期翻倍 | 同一内容 HLS(TS) 和 DASH(fMP4) 双打包 | 切 CMAF 单打包（`-hls_segment_type fmp4`），存储减半 |
| 转码队列积压、机器全忙 | 软件编码吃满 CPU 且无限流 | 转码走队列 + Worker 池限并发，4K/AV1 用硬件编码（NVENC/SVT-AV1，见第 19 篇） |
| 版权片在禁播地区还能看 | License 未按 IP/地区校验 | CDN 加地理限制，License 服务拒绝签发对应地区 |

---

## 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| OTT | Over-The-Top | 绕过运营商直连互联网的视频分发，典型形态是 VOD 点播 |
| VOD | Video on Demand | 点播：预录内容随时回放，与直播相对 |
| 编码阶梯 | Transcode Ladder | 一组分辨率/码率逐级递增的档位，播放器按网速切换 |
| 产物矩阵 | Rendition Matrix | 同一内容在"编码器×分辨率×封装"三个维度上的全部产物 |
| ABR | Adaptive Bitrate | 播放器按实时带宽在多档产物间自动切换的机制 |
| BOLA | Buffer-Occupancy based Lyapunov Algorithm | 基于播放缓冲区水位选档的 ABR 算法 |
| MPC | Model Predictive Control | 用 QoE 模型对未来几片做最优规划的 ABR 算法 |
| 迟滞 | Hysteresis | 切档阈值带缓冲，防画质频繁来回抖动的机制 |
| CMAF | Common Media Application Format | 同一份 fMP4 同时服务 HLS 与 DASH 的统一打包格式 |
| cbcs | Cipher Block Chaining Segment | CMAF 的通用加密模式，一份密文可被多种 DRM 解密 |
| 预推/预热 | Preload / Prewarm | 上架时把热内容主动派发到 CDN 边缘，开播即命中 |
| Open Connect | Netflix Open Connect | Netflix 把自研缓存设备放进 ISP 机房的自建 CDN 方案 |
| 个性化预加载 | Personalized Preload | 根据观看历史预取下一集，实现点"下一集"秒开 |

---

> **全文完** | 上一篇：[第 29.6 篇 — 远程医疗视频会诊](29.6-远程医疗视频会诊.md) | 下一篇：[第 31 篇 — 云游戏流媒体技术](31-云游戏流媒体技术.md)