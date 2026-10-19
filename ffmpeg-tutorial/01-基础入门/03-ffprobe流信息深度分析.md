# 第 3 篇：ffprobe 流信息深度分析

> **读完本篇你将理解**：ffprobe 的核心用途、如何分析流信息、容器信息、帧信息，如何诊断视频问题、编写自动化分析脚本。

---

## 目录

- [ffprobe 用途与定位](#ffprobe-用途与定位)
- [输出格式选择](#输出格式选择)
- [分析流信息](#分析流信息)
- [分析容器信息](#分析容器信息)
- [分析帧信息](#分析帧信息)
- [分析数据包](#分析数据包)
- [实用分析命令合集](#实用分析命令合集)
- [动手实践](#动手实践)
- [本篇小结](#本篇小结)
- [关键术语速查表](#关键术语速查表)

---

## ffprobe 用途与定位

### 为什么需要 ffprobe？

**ffprobe 是诊断一切视频问题的起点**。在遇到以下问题时，第一步永远是用 ffprobe 检查：

- 为什么转码失败？→ 检查输入文件的编码格式
- 为什么播放卡顿？→ 检查关键帧间隔（GOP）
- 为什么音视频不同步？→ 检查时间戳（PTS/DTS）
- 为什么文件这么大？→ 检查码率和分辨率
- 为什么某些播放器播不了？→ 检查编码器 profile/level

### ffprobe vs ffmpeg -i

```bash
# 方式 1：用 ffmpeg -i（不推荐，输出混乱）
ffmpeg -i video.mp4

# 方式 2：用 ffprobe（推荐，输出结构化）
ffprobe video.mp4
```

**ffprobe 的优势**：
- 输出格式可控（JSON/XML/CSV/Plain Text）
- 不会尝试转码，纯分析
- 可以精确选择要查看的信息
- 适合脚本自动化处理

---

## 输出格式选择

### 默认文本格式

```bash
ffprobe video.mp4
```

**输出示例**（部分）：
```
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'video.mp4':
  Duration: 00:01:30.00, start: 0.000000, bitrate: 5000 kb/s
  Stream #0:0(und): Video: h264 (High), yuv420p, 1920x1080, 30 fps
  Stream #0:1(und): Audio: aac (LC), 48000 Hz, stereo, 128 kb/s
```

**适用场景**：人工快速查看

---

### JSON 格式（推荐用于脚本）

```bash
ffprobe -v quiet -print_format json -show_format -show_streams video.mp4
```

**输出示例**（部分）：
```json
{
  "streams": [
    {
      "index": 0,
      "codec_name": "h264",
      "codec_type": "video",
      "width": 1920,
      "height": 1080,
      "r_frame_rate": "30/1",
      "avg_frame_rate": "30/1",
      "pix_fmt": "yuv420p",
      "bit_rate": "5000000"
    },
    {
      "index": 1,
      "codec_name": "aac",
      "codec_type": "audio",
      "sample_rate": "48000",
      "channels": 2,
      "bit_rate": "128000"
    }
  ],
  "format": {
    "filename": "video.mp4",
    "format_name": "mov,mp4,m4a,3gp,3g2,mj2",
    "duration": "90.000000",
    "size": "56250000",
    "bit_rate": "5000000"
  }
}
```

**适用场景**：
- Go/Python/Node.js 脚本解析
- 自动化批量分析
- 数据库存储

---

### XML 格式

```bash
ffprobe -v quiet -print_format xml -show_format -show_streams video.mp4
```

**适用场景**：需要 XML 格式的系统集成

---

### CSV 格式（适合统计）

```bash
ffprobe -v quiet -print_format csv -show_streams video.mp4
```

**输出示例**：
```
stream,0,0,h264,video,,,1920,1080,30/1,yuv420p,5000000
stream,1,1,aac,audio,48000,2,,,128000
```

**适用场景**：导入 Excel/数据库批量统计

---

### flat 格式（键值对）

```bash
ffprobe -v quiet -print_format flat -show_format -show_streams video.mp4
```

**输出示例**：
```
streams.stream.0.codec_name="h264"
streams.stream.0.width=1920
streams.stream.0.height=1080
format.duration="90.000000"
format.bit_rate="5000000"
```

**适用场景**：Shell 脚本直接读取（grep/awk）

---

## 分析流信息

### 查看所有流

```bash
ffprobe -v quiet -show_streams video.mp4
```

---

### 只查看视频流

```bash
ffprobe -v quiet -select_streams v -show_streams video.mp4
```

**关键字段解读**：

| 字段 | 含义 | 示例值 |
|------|------|--------|
| `codec_name` | 编码器名称 | h264, hevc, vp9 |
| `codec_type` | 流类型 | video, audio, subtitle |
| `width` / `height` | 分辨率 | 1920 × 1080 |
| `r_frame_rate` | 真实帧率（容器声明） | 30/1 = 30fps |
| `avg_frame_rate` | 平均帧率（计算得出） | 29.97/1 ≈ 29.97fps |
| `pix_fmt` | 像素格式 | yuv420p, yuv444p, nv12 |
| `bit_rate` | 码率（bps） | 5000000 = 5Mbps |
| `profile` | 编码器 profile | High, Main, Baseline |
| `level` | 编码器 level | 40 = 4.0, 51 = 5.1 |

---

### 只查看音频流

```bash
ffprobe -v quiet -select_streams a -show_streams video.mp4
```

**关键字段解读**：

| 字段 | 含义 | 示例值 |
|------|------|--------|
| `codec_name` | 编码器名称 | aac, mp3, opus |
| `sample_rate` | 采样率（Hz） | 48000, 44100 |
| `channels` | 声道数 | 2 = 立体声, 1 = 单声道 |
| `channel_layout` | 声道布局 | stereo, 5.1, mono |
| `bit_rate` | 码率（bps） | 128000 = 128kbps |

---

### 只查看字幕流

```bash
ffprobe -v quiet -select_streams s -show_streams video.mkv
```

**关键字段**：
- `codec_name`：srt, ass, webvtt, pgssub（蓝光图形字幕）
- `language`：字幕语言（eng, zho, jpn）

---

### 快速提取单个字段

```bash
# 提取视频编码器
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name \
  -of default=noprint_wrappers=1:nokey=1 video.mp4

# 输出：h264

# 提取分辨率
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height \
  -of csv=s=x:p=0 video.mp4

# 输出：1920x1080

# 提取时长（秒）
ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 video.mp4

# 输出：90.000000
```

---

## 分析容器信息

### 查看容器元数据

```bash
ffprobe -v quiet -show_format video.mp4
```

**关键字段解读**：

| 字段 | 含义 | 用途 |
|------|------|------|
| `format_name` | 容器格式 | mov,mp4,m4a,3gp,3g2,mj2 |
| `duration` | 总时长（秒） | 90.000000 |
| `size` | 文件大小（字节） | 56250000 ≈ 53.6MB |
| `bit_rate` | 总码率（bps） | 5000000 = 5Mbps |
| `nb_streams` | 流数量 | 2（1视频+1音频） |
| `tags` | 元数据标签 | 标题、作者、创建时间等 |

---

### 查看元数据标签

```bash
ffprobe -v quiet -show_format -show_entries format_tags video.mp4
```

**常见标签**：
- `title`：标题
- `artist`：作者
- `creation_time`：创建时间
- `encoder`：编码器软件（如 "Lavf58.76.100"）

---

## 分析帧信息

### 为什么要分析帧？

帧级别的分析可以诊断：
- 关键帧间隔（GOP 长度）
- 时间戳问题（PTS/DTS 错乱）
- B 帧数量（影响延迟）

---

### 查看所有帧信息

```bash
ffprobe -v quiet -show_frames video.mp4
```

**警告**：输出非常多（每帧一条记录），建议只看前几帧：

```bash
ffprobe -v quiet -show_frames video.mp4 | head -100
```

---

### 只查看关键帧

```bash
ffprobe -v quiet -select_streams v -show_frames \
  -show_entries frame=pkt_pts_time,pict_type,key_frame \
  video.mp4 | grep key_frame=1
```

**字段解读**：
- `pkt_pts_time`：播放时间戳（秒）
- `pict_type`：帧类型（I/P/B）
- `key_frame`：是否关键帧（1=是，0=否）

---

### 统计 GOP 长度

```bash
# 统计关键帧间隔
ffprobe -v quiet -select_streams v -show_frames \
  -show_entries frame=pkt_pts_time,key_frame video.mp4 \
  | grep key_frame=1 \
  | awk -F= '{print $2}' \
  | awk 'NR>1 {print ($0-prev)*30} {prev=$0}'

# 输出：每个 GOP 的帧数（假设 30fps）
```

---

## 分析数据包

### 数据包 vs 帧的区别

- **帧（Frame）**：解码后的画面/音频样本
- **数据包（Packet）**：压缩数据，未解码

**适用场景**：
- 分析码率波动（Packet size）
- 检查关键帧标记（Packet flags）

---

### 查看数据包信息

```bash
ffprobe -v quiet -show_packets video.mp4 | head -50
```

**关键字段**：
- `pts_time`：播放时间戳
- `dts_time`：解码时间戳
- `size`：数据包大小（字节）
- `flags`：标记（K = 关键帧）

---

### 统计码率波动

```bash
# 每秒的码率（kbps）
ffprobe -v quiet -select_streams v -show_packets \
  -show_entries packet=pts_time,size video.mp4 \
  | grep -E "pts_time|size" \
  | awk 'NR%2==1 {time=$0} NR%2==0 {size=$0; print time, size}'
```

---

## 实用分析命令合集

### 1. 检查视频是否有 B 帧

```bash
ffprobe -v quiet -select_streams v -show_frames \
  -show_entries frame=pict_type video.mp4 \
  | grep "pict_type=B"

# 如果有输出，说明有 B 帧
# 如果没有输出，说明只有 I/P 帧（低延迟编码）
```

**用途**：判断视频是否适合低延迟场景

---

### 2. 统计帧类型分布

```bash
ffprobe -v quiet -select_streams v -show_frames \
  -show_entries frame=pict_type video.mp4 \
  | grep pict_type \
  | sort | uniq -c

# 输出示例：
#  30 pict_type=I
# 120 pict_type=P
#  50 pict_type=B
```

---

### 3. 检查音视频是否对齐

```bash
# 提取视频第一帧的 PTS
video_pts=$(ffprobe -v error -select_streams v:0 -show_frames \
  -show_entries frame=pkt_pts_time video.mp4 | head -2 | tail -1 | cut -d= -f2)

# 提取音频第一帧的 PTS
audio_pts=$(ffprobe -v error -select_streams a:0 -show_frames \
  -show_entries frame=pkt_pts_time video.mp4 | head -2 | tail -1 | cut -d= -f2)

# 计算差值
echo "音视频偏差: $(echo "$audio_pts - $video_pts" | bc) 秒"
```

---

### 4. 检查 duration 是否准确

```bash
# 容器 duration
container_duration=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 video.mp4)

# 视频流 duration
stream_duration=$(ffprobe -v error -select_streams v:0 \
  -show_entries stream=duration \
  -of default=noprint_wrappers=1:nokey=1 video.mp4)

echo "容器时长: $container_duration 秒"
echo "视频流时长: $stream_duration 秒"

# 如果两者相差 > 1 秒，可能有问题
```

---

### 5. 批量检查目录下所有视频

```bash
#!/bin/bash
# 批量分析视频编码信息

for file in *.mp4; do
    echo "=== $file ==="
    
    # 编码器
    codec=$(ffprobe -v error -select_streams v:0 \
      -show_entries stream=codec_name -of default=noprint_wrappers=1:nokey=1 "$file")
    
    # 分辨率
    resolution=$(ffprobe -v error -select_streams v:0 \
      -show_entries stream=width,height -of csv=s=x:p=0 "$file")
    
    # 码率
    bitrate=$(ffprobe -v error -show_entries format=bit_rate \
      -of default=noprint_wrappers=1:nokey=1 "$file")
    bitrate_mbps=$(echo "scale=2; $bitrate / 1000000" | bc)
    
    # 时长
    duration=$(ffprobe -v error -show_entries format=duration \
      -of default=noprint_wrappers=1:nokey=1 "$file")
    
    echo "编码器: $codec"
    echo "分辨率: $resolution"
    echo "码率: ${bitrate_mbps}Mbps"
    echo "时长: ${duration}秒"
    echo ""
done
```

---

## 动手实践

### 实践 1：分析一个实际视频文件

```bash
# 1. 生成测试视频
ffmpeg -f lavfi -i testsrc=duration=10:size=1920x1080:rate=30 \
       -f lavfi -i sine=frequency=1000:duration=10 \
       -c:v libx264 -g 60 -c:a aac test.mp4

# 2. 用 ffprobe 分析（JSON 格式）
ffprobe -v quiet -print_format json -show_format -show_streams test.mp4 > info.json

# 3. 查看 JSON 内容
cat info.json | jq .

# 4. 提取关键信息
echo "编码器: $(jq -r '.streams[0].codec_name' info.json)"
echo "分辨率: $(jq -r '.streams[0].width' info.json)x$(jq -r '.streams[0].height' info.json)"
echo "帧率: $(jq -r '.streams[0].r_frame_rate' info.json)"
```

---

### 实践 2：检查关键帧间隔

```bash
# 统计关键帧时间点
ffprobe -v quiet -select_streams v -show_frames \
  -show_entries frame=pkt_pts_time,key_frame test.mp4 \
  | grep -B1 "key_frame=1" \
  | grep pkt_pts_time \
  | cut -d= -f2

# 输出：0.000000, 2.000000, 4.000000, 6.000000, 8.000000
# 说明：每 2 秒一个关键帧（-g 60，30fps × 2秒 = 60帧）
```

---

### 实践 3：编写批量分析脚本（Go 版本）

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
)

type ProbeInfo struct {
    Streams []struct {
        CodecName string `json:"codec_name"`
        CodecType string `json:"codec_type"`
        Width     int    `json:"width"`
        Height    int    `json:"height"`
        BitRate   string `json:"bit_rate"`
    } `json:"streams"`
    Format struct {
        Duration string `json:"duration"`
        Size     string `json:"size"`
        BitRate  string `json:"bit_rate"`
    } `json:"format"`
}

func analyzeVideo(path string) (*ProbeInfo, error) {
    cmd := exec.Command("ffprobe",
        "-v", "quiet",
        "-print_format", "json",
        "-show_format", "-show_streams",
        path)
    
    output, err := cmd.Output()
    if err != nil {
        return nil, err
    }
    
    var info ProbeInfo
    if err := json.Unmarshal(output, &info); err != nil {
        return nil, err
    }
    
    return &info, nil
}

func main() {
    if len(os.Args) < 2 {
        fmt.Println("用法: go run main.go <video_file>")
        return
    }
    
    videoPath := os.Args[1]
    info, err := analyzeVideo(videoPath)
    if err != nil {
        fmt.Printf("分析失败: %v\n", err)
        return
    }
    
    fmt.Printf("=== %s ===\n", filepath.Base(videoPath))
    
    for _, stream := range info.Streams {
        if stream.CodecType == "video" {
            fmt.Printf("视频编码器: %s\n", stream.CodecName)
            fmt.Printf("分辨率: %dx%d\n", stream.Width, stream.Height)
            fmt.Printf("视频码率: %s bps\n", stream.BitRate)
        } else if stream.CodecType == "audio" {
            fmt.Printf("音频编码器: %s\n", stream.CodecName)
            fmt.Printf("音频码率: %s bps\n", stream.BitRate)
        }
    }
    
    fmt.Printf("总时长: %s 秒\n", info.Format.Duration)
    fmt.Printf("文件大小: %s 字节\n", info.Format.Size)
}
```

**运行**：
```bash
go run analyze.go test.mp4
```

---

### 实践 4：检测损坏的视频文件

```bash
#!/bin/bash
# 检测视频文件是否损坏

check_video() {
    local file="$1"
    
    # 尝试分析文件
    if ! ffprobe -v error "$file" &> /dev/null; then
        echo "❌ $file - 损坏或不支持的格式"
        return 1
    fi
    
    # 检查是否有视频流
    video_streams=$(ffprobe -v error -select_streams v -show_entries stream=codec_type \
        -of default=noprint_wrappers=1:nokey=1 "$file" | wc -l)
    
    if [ "$video_streams" -eq 0 ]; then
        echo "❌ $file - 没有视频流"
        return 1
    fi
    
    # 检查 duration 是否有效
    duration=$(ffprobe -v error -show_entries format=duration \
        -of default=noprint_wrappers=1:nokey=1 "$file")
    
    if [ -z "$duration" ] || [ "$duration" = "N/A" ]; then
        echo "⚠️  $file - duration 未知（可能是流文件）"
        return 0
    fi
    
    echo "✅ $file - 正常"
    return 0
}

# 批量检查
for file in *.mp4; do
    check_video "$file"
done
```

---

## 本篇小结

1. **ffprobe 是诊断工具**：遇到任何视频问题，第一步用 ffprobe 检查

2. **输出格式选择**：
   - 人工查看：默认文本格式
   - 脚本解析：JSON 格式（`-print_format json`）

3. **三大分析目标**：
   - 流信息（`-show_streams`）：编码格式、分辨率、码率
   - 容器信息（`-show_format`）：时长、文件大小
   - 帧信息（`-show_frames`）：关键帧、时间戳

4. **实用场景**：
   - 检查关键帧间隔（GOP）
   - 统计帧类型分布（I/P/B）
   - 检测音视频对齐
   - 批量分析视频库

5. **Go/Python 集成**：用 `exec.Command` 调用 ffprobe，解析 JSON 输出

---

## 关键术语速查表

| 术语 | 英文 | 解释 |
|------|------|------|
| 流 | Stream | 容器内的独立轨道（视频/音频/字幕） |
| PTS | Presentation Time Stamp | 播放时间戳（何时显示这一帧） |
| DTS | Decode Time Stamp | 解码时间戳（何时解码这一帧） |
| GOP | Group of Pictures | 一组帧，从一个 I 帧到下一个 I 帧 |
| 帧类型 | Frame Type | I（关键帧）/ P（预测帧）/ B（双向预测帧） |
| 码率 | Bit Rate | 每秒传输的比特数（bps/kbps/Mbps） |
| 像素格式 | Pixel Format | YUV420P、YUV444P、RGB 等 |
| profile | Codec Profile | 编码器配置（Baseline/Main/High） |
| level | Codec Level | 编码器级别（3.1/4.0/5.1） |
| 容器 | Container | 文件的封装格式（MP4/MKV/FLV） |

---

**下一步**：[第 4 篇：命令行语法完全解析](04-命令行语法完全解析.md)

---

**最后更新**：2026-07-28  
**字数**：约 6500 字
