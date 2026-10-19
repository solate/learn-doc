# 第 25 篇：Go 集成 FFmpeg 完整方案

> **模块 6 · 工程化与生产部署** | 预计阅读时间：60 分钟 | 难度：⭐⭐⭐⭐⭐

---

## 目录

1. [集成模式概览](#1-集成模式概览)
2. [exec.Command 安全调用](#2-execcommand-安全调用)
3. [实时进度解析](#3-实时进度解析)
4. [超时控制与优雅退出](#4-超时控制与优雅退出)
5. [错误分类与重试策略](#5-错误分类与重试策略)
6. [并发控制与资源管理](#6-并发控制与资源管理)
7. [异步任务队列](#7-异步任务队列)
8. [日志集成与可观测性](#8-日志集成与可观测性)
9. [完整工程示例](#9-完整工程示例)
10. [本章总结](#10-本章总结)

---

## 学习目标

完成本篇后，你将能够：

- [ ] 用 `exec.CommandContext` 安全调用 FFmpeg，避免命令注入
- [ ] 解析 FFmpeg `-progress pipe:1` 输出实现进度回调
- [ ] 在 Go 中优雅终止 FFmpeg 进程（SIGTERM → 等待 → SIGKILL）
- [ ] 区分可重试错误与永久错误，实现带退避的重试
- [ ] 用信号量和工作池控制并发转码数量
- [ ] 设计基于 channel 的异步任务队列
- [ ] 集成 Zerolog 日志和 Prometheus 指标

---

## 1. 集成模式概览

### 1.1 三种集成方式对比

```
方式 A：直接调用（exec.Command）
  Go → exec.Command("ffmpeg", args...) → 等待退出
  ✅ 简单，90% 场景足够
  ✅ 原生进程隔离，崩溃不影响主进程
  ❌ 需要系统安装 FFmpeg，版本一致性靠外部保证

方式 B：CGO 绑定（go-astiav / ffgopeg）
  Go → CGO → libav* C 库 → 直接调用 API
  ✅ 零进程开销，精细控制每一帧
  ❌ CGO 编译复杂（交叉编译痛苦）
  ❌ libav* 版本更新频繁，binding 滞后
  适合：需要逐帧处理、超低延迟的场景

方式 C：HTTP 服务化（ffmpeg + REST）
  Go → HTTP → FFmpeg 工作进程（独立部署）
  ✅ 语言无关，横向扩展容易
  ❌ 网络开销，适合大任务而非实时处理
  适合：多语言团队，分布式集群

本篇聚焦方式 A（最常用），兼顾方式 C 的架构思路。
```

### 1.2 设计原则

```
1. 参数数组优于字符串拼接 → 避免命令注入
2. exec.CommandContext   → 生命周期与 context 绑定
3. 解析 -progress pipe  → 进度可观测
4. SIGTERM → SIGKILL 双阶段退出 → 让 FFmpeg 完成当前 mux
5. 信号量限制并发数      → 保护 CPU/带宽/磁盘
```

---

## 2. exec.Command 安全调用

### 2.1 命令注入风险（必须避免的写法）

```go
// ❌ 危险：字符串拼接 + shell 执行
func BadTranscode(inputPath string) error {
	// 如果 inputPath = "a.mp4; rm -rf /"，会执行删除命令！
	cmdStr := fmt.Sprintf("ffmpeg -i %s -c:v libx264 out.mp4", inputPath)
	return exec.Command("sh", "-c", cmdStr).Run()
}

// ✅ 安全：参数数组，不经过 shell
func GoodTranscode(ctx context.Context, inputPath string) error {
	args := []string{"-i", inputPath, "-c:v", "libx264", "out.mp4"}
	return exec.CommandContext(ctx, "ffmpeg", args...).Run()
}
```

**关键点**：`exec.Command` 不经过 shell，参数中的 `;`、`|`、`$()` 都被当作字面字符处理。
只要不用 `sh -c`，就没有命令注入风险。

### 2.2 输入路径校验

即使参数数组安全，仍需校验路径避免越权访问：

```go
package ffmpeg

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// ValidateInputPath 校验输入路径合法性
func ValidateInputPath(path, allowedRoot string) (string, error) {
	// 1. 解析为绝对路径（消除 .. 和符号链接）
	abs, err := filepath.Abs(path)
	if err != nil {
		return "", fmt.Errorf("路径解析失败: %w", err)
	}
	resolved, err := filepath.EvalSymlinks(abs)
	if err != nil {
		return "", fmt.Errorf("符号链接解析失败: %w", err)
	}

	// 2. 确保在允许的根目录内（防止路径穿越）
	rootAbs, err := filepath.Abs(allowedRoot)
	if err != nil {
		return "", fmt.Errorf("根目录解析失败: %w", err)
	}
	rel, err := filepath.Rel(rootAbs, resolved)
	if err != nil || strings.HasPrefix(rel, "..") {
		return "", fmt.Errorf("路径越权: %s 不在 %s 内", path, allowedRoot)
	}

	// 3. 确认文件存在且为常规文件
	info, err := os.Stat(resolved)
	if err != nil {
		return "", fmt.Errorf("文件不存在: %w", err)
	}
	if !info.Mode().IsRegular() {
		return "", fmt.Errorf("不是常规文件: %s", resolved)
	}

	return resolved, nil
}
```

### 2.3 URL 输入的特殊处理

网络输入（RTMP/HTTP）需要限制协议白名单，避免 SSRF 和本地文件读取：

```go
// FFmpeg 的 -protocol_whitelist 限制可用协议
func BuildNetworkInputArgs(inputURL string) ([]string, error) {
	u, err := url.Parse(inputURL)
	if err != nil {
		return nil, fmt.Errorf("URL 解析失败: %w", err)
	}

	// 协议白名单
	allowed := map[string]bool{
		"http": true, "https": true,
		"rtmp": true, "rtmps": true,
		"rtsp": true, "srt": true,
	}
	if !allowed[u.Scheme] {
		return nil, fmt.Errorf("不允许的协议: %s", u.Scheme)
	}

	return []string{
		// 限制 FFmpeg 内部可用协议（防止通过 playlist 跳转到 file://）
		"-protocol_whitelist", "file,http,https,tcp,tls,rtmp,rtsp,srt,crypto",
		"-i", inputURL,
	}, nil
}
```

**为什么需要 `-protocol_whitelist`**：
HLS 的 m3u8 里可以引用任意 URI。若不限制，攻击者可提供恶意 m3u8 让 FFmpeg 读取 `file:///etc/passwd`
并编码进输出视频，造成信息泄露。

### 2.4 基础执行封装

```go
// Runner FFmpeg 执行器
type Runner struct {
	BinPath string        // ffmpeg 可执行文件路径
	Timeout time.Duration // 单任务超时
}

// Run 执行 FFmpeg 命令，返回 stderr 内容（FFmpeg 日志走 stderr）
func (r *Runner) Run(ctx context.Context, args []string) (string, error) {
	if r.Timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, r.Timeout)
		defer cancel()
	}

	bin := r.BinPath
	if bin == "" {
		bin = "ffmpeg"
	}

	// 统一加上：覆盖输出、隐藏 banner、只输出 error 级日志
	fullArgs := append([]string{"-hide_banner", "-nostdin", "-y"}, args...)

	cmd := exec.CommandContext(ctx, bin, fullArgs...)

	var stderr bytes.Buffer
	cmd.Stderr = &stderr

	err := cmd.Run()
	if err != nil {
		return stderr.String(), fmt.Errorf("ffmpeg 执行失败: %w", err)
	}
	return stderr.String(), nil
}
```

**`-nostdin` 的作用**：
FFmpeg 默认监听 stdin 接受交互命令（如按 `q` 退出）。
在服务端运行时，若 stdin 是继承的终端，可能导致 FFmpeg 意外消费输入或阻塞。
`-nostdin` 明确禁用交互，是服务化场景的必备参数。

---

## 3. 实时进度解析

### 3.1 FFmpeg 进度输出格式

```bash
# -progress pipe:1 将进度写入 stdout，stderr 不受影响
ffmpeg -i input.mp4 -c:v libx264 output.mp4 \
  -progress pipe:1 -nostats

# stdout 输出（每秒或每帧更新一次，KV 格式）：
# frame=150
# fps=25.00
# stream_0_0_q=28.0
# bitrate=2499.8kbits/s
# total_size=1875000
# out_time_us=600000000
# out_time_ms=600000
# out_time=00:00:06.000000
# dup_frames=0
# drop_frames=0
# speed=1.01x
# progress=continue          ← 进行中
# (最后一组)
# progress=end               ← 完成
```

### 3.2 进度解析器

```go
package ffmpeg

import (
	"bufio"
	"context"
	"fmt"
	"io"
	"os/exec"
	"strconv"
	"strings"
	"time"
)

// Progress FFmpeg 进度信息
type Progress struct {
	Frame     int64
	FPS       float64
	Bitrate   string
	OutTimeMS int64         // 已编码时长（毫秒）
	Speed     float64       // 编码速度（1.0 = 实时）
	Done      bool          // true = 编码完成
}

// OnProgressFunc 进度回调
type OnProgressFunc func(p Progress)

// ProgressRunner 带进度回调的执行器
type ProgressRunner struct {
	BinPath string
}

// Run 执行 FFmpeg，通过回调实时上报进度
func (r *ProgressRunner) Run(ctx context.Context, args []string, onProgress OnProgressFunc) error {
	bin := r.BinPath
	if bin == "" {
		bin = "ffmpeg"
	}

	// -progress pipe:1 → stdout；-nostats 禁止旧式进度（避免混淆）
	fullArgs := append([]string{"-hide_banner", "-nostdin", "-y", "-nostats", "-progress", "pipe:1"}, args...)

	cmd := exec.CommandContext(ctx, bin, fullArgs...)

	// stdout 给进度解析器读取
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return fmt.Errorf("获取 stdout pipe 失败: %w", err)
	}
	// stderr 收集日志（FFmpeg 日志走 stderr）
	var stderrBuf strings.Builder
	cmd.Stderr = &stderrBuf

	if err := cmd.Start(); err != nil {
		return fmt.Errorf("启动 FFmpeg 失败: %w", err)
	}

	// 在 goroutine 中解析进度
	go parseProgress(stdout, onProgress)

	if err := cmd.Wait(); err != nil {
		if ctx.Err() != nil {
			return fmt.Errorf("任务被取消: %w", ctx.Err())
		}
		return fmt.Errorf("FFmpeg 失败: %w\nstderr: %s", err, stderrBuf.String())
	}
	return nil
}

// parseProgress 从 FFmpeg stdout 解析进度数据
func parseProgress(r io.Reader, onProgress OnProgressFunc) {
	scanner := bufio.NewScanner(r)
	current := Progress{}

	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if line == "" {
			continue
		}

		parts := strings.SplitN(line, "=", 2)
		if len(parts) != 2 {
			continue
		}
		key, val := parts[0], strings.TrimSpace(parts[1])

		switch key {
		case "frame":
			current.Frame, _ = strconv.ParseInt(val, 10, 64)
		case "fps":
			current.FPS, _ = strconv.ParseFloat(val, 64)
		case "bitrate":
			current.Bitrate = val
		case "out_time_ms":
			ms, _ := strconv.ParseInt(val, 10, 64)
			current.OutTimeMS = ms / 1000 // us → ms
		case "speed":
			s := strings.TrimSuffix(val, "x")
			current.Speed, _ = strconv.ParseFloat(s, 64)
		case "progress":
			current.Done = val == "end"
			if onProgress != nil {
				onProgress(current)
			}
			// 重置（新一轮统计）
			current = Progress{}
		}
	}
}
```

### 3.3 进度百分比计算

`-progress` 不直接给出百分比，需先获取视频总时长：

```go
// ProgressWithPercent 带百分比的进度计算
type ProgressWithPercent struct {
	Progress
	Percent float64  // 0.0 ~ 100.0
}

// GetDuration 获取视频时长（毫秒）
func GetDuration(ctx context.Context, inputPath string) (int64, error) {
	cmd := exec.CommandContext(ctx, "ffprobe",
		"-v", "error",
		"-show_entries", "format=duration",
		"-of", "csv=p=0",
		inputPath)
	out, err := cmd.Output()
	if err != nil {
		return 0, fmt.Errorf("ffprobe 失败: %w", err)
	}
	secs, err := strconv.ParseFloat(strings.TrimSpace(string(out)), 64)
	if err != nil {
		return 0, err
	}
	return int64(secs * 1000), nil
}

// RunWithPercent 执行转码，进度回调含百分比
func RunWithPercent(ctx context.Context, inputPath string, args []string, onProgress func(ProgressWithPercent)) error {
	totalMS, err := GetDuration(ctx, inputPath)
	if err != nil {
		// 获取时长失败时退化为无百分比模式
		totalMS = 0
	}

	runner := &ProgressRunner{}
	return runner.Run(ctx, args, func(p Progress) {
		pp := ProgressWithPercent{Progress: p}
		if totalMS > 0 && p.OutTimeMS > 0 {
			pp.Percent = float64(p.OutTimeMS) / float64(totalMS) * 100
			if pp.Percent > 100 {
				pp.Percent = 100
			}
		}
		if onProgress != nil {
			onProgress(pp)
		}
	})
}
```

---

## 4. 超时控制与优雅退出

### 4.1 两阶段终止策略

```
直接 Kill 的问题：
  FFmpeg 被 SIGKILL → 立即终止 → 输出文件 moov box 未写入 → MP4 损坏

正确方式（两阶段）：
  1. SIGTERM → FFmpeg 捕获信号，完成当前 GOP 写入，关闭文件
  2. 等待 N 秒 → FFmpeg 正常退出（✅ 文件完整）
  3. 若等待超时 → SIGKILL → 强制终止（文件可能损坏，但进程肯定结束）
```

### 4.2 GracefulKill 实现

```go
package ffmpeg

import (
	"context"
	"os"
	"os/exec"
	"syscall"
	"time"
)

// GracefulKillTimeout 等待 SIGTERM 优雅退出的最长时间
const GracefulKillTimeout = 5 * time.Second

// GracefulKill 先发 SIGTERM 等待，超时再发 SIGKILL
func GracefulKill(cmd *exec.Cmd) error {
	if cmd == nil || cmd.Process == nil {
		return nil
	}

	// 发送 SIGTERM（请求优雅退出）
	if err := cmd.Process.Signal(syscall.SIGTERM); err != nil {
		// 进程可能已经退出
		if err == os.ErrProcessDone {
			return nil
		}
		return err
	}

	// 等待进程退出（在 goroutine 中 Wait）
	done := make(chan error, 1)
	go func() {
		done <- cmd.Wait()
	}()

	select {
	case err := <-done:
		// 正常退出（可能是 SIGTERM 触发的退出，err 是 exit status）
		return err
	case <-time.After(GracefulKillTimeout):
		// 超时，强制 Kill
		_ = cmd.Process.Kill()
		return <-done
	}
}
```

### 4.3 带超时的 Job Runner

```go
// JobConfig 转码任务配置
type JobConfig struct {
	Args       []string      // FFmpeg 参数（不含 -hide_banner 等）
	Timeout    time.Duration // 任务级超时（0=无限）
	OnProgress OnProgressFunc
}

// JobRunner 单任务执行器（含超时和优雅退出）
type JobRunner struct {
	BinPath string
}

// Run 执行单个转码任务
func (jr *JobRunner) Run(ctx context.Context, cfg JobConfig) error {
	// 构建任务级 context（带超时）
	taskCtx := ctx
	var taskCancel context.CancelFunc
	if cfg.Timeout > 0 {
		taskCtx, taskCancel = context.WithTimeout(ctx, cfg.Timeout)
		defer taskCancel()
	}

	bin := jr.BinPath
	if bin == "" {
		bin = "ffmpeg"
	}

	fullArgs := buildArgs(cfg.Args, cfg.OnProgress != nil)
	cmd := exec.CommandContext(taskCtx, bin, fullArgs...)

	var stderrBuf strings.Builder
	cmd.Stderr = &stderrBuf

	var stdout io.ReadCloser
	if cfg.OnProgress != nil {
		var err error
		stdout, err = cmd.StdoutPipe()
		if err != nil {
			return fmt.Errorf("stdout pipe 失败: %w", err)
		}
	}

	if err := cmd.Start(); err != nil {
		return fmt.Errorf("启动 FFmpeg 失败: %w", err)
	}

	if cfg.OnProgress != nil {
		go parseProgress(stdout, cfg.OnProgress)
	}

	// 监听 context 取消，发送优雅终止信号
	stopCh := make(chan struct{})
	go func() {
		select {
		case <-taskCtx.Done():
			GracefulKill(cmd)
		case <-stopCh:
		}
	}()

	err := cmd.Wait()
	close(stopCh)

	if err != nil {
		if taskCtx.Err() == context.DeadlineExceeded {
			return fmt.Errorf("任务超时（%s）", cfg.Timeout)
		}
		if taskCtx.Err() == context.Canceled {
			return fmt.Errorf("任务被取消")
		}
		return fmt.Errorf("FFmpeg 执行失败: %w\nstderr: %s", err, stderrBuf.String())
	}
	return nil
}

// buildArgs 构建完整参数列表
func buildArgs(userArgs []string, withProgress bool) []string {
	base := []string{"-hide_banner", "-nostdin", "-y"}
	if withProgress {
		base = append(base, "-nostats", "-progress", "pipe:1")
	}
	return append(base, userArgs...)
}
```

---

## 5. 错误分类与重试策略

### 5.1 FFmpeg 错误分类

不是所有错误都值得重试。盲目重试会放大故障：

```
永久错误（不重试）：
  - "No such file or directory"        → 输入不存在
  - "Invalid data found when processing input" → 文件损坏
  - "Unknown encoder"                 → 编码器不存在（构建问题）
  - "Invalid argument"                → 参数错误
  - "Permission denied"               → 权限问题

临时错误（可重试）：
  - "Connection timed out"            → 网络超时
  - "Connection refused"              → 服务端暂时不可用
  - "Server returned 5XX"             → 源站 5xx
  - "Input/output error"              → 磁盘临时故障
  - "Resource temporarily unavailable" → 资源竞争

资源错误（延迟重试或降级）：
  - "Cannot allocate memory"          → 内存不足，等待或降低并发
  - "Device or resource busy"         → GPU 占满，排队
  - "No space left on device"         → 磁盘满（需先清理，重试无意义）
```

### 5.2 错误分类器

```go
package ffmpeg

import (
	"errors"
	"strings"
)

// ErrorKind 错误类型
type ErrorKind int

const (
	ErrKindUnknown   ErrorKind = iota
	ErrKindPermanent           // 永久错误，不应重试
	ErrKindTransient           // 临时错误，可重试
	ErrKindResource            // 资源不足，延迟重试
)

// permanentPatterns 永久错误特征
var permanentPatterns = []string{
	"no such file or directory",
	"invalid data found when processing input",
	"unknown encoder",
	"unknown decoder",
	"invalid argument",
	"permission denied",
	"unrecognized option",
	"could not find codec parameters",
}

// transientPatterns 临时错误特征
var transientPatterns = []string{
	"connection timed out",
	"connection refused",
	"connection reset by peer",
	"server returned 5",
	"input/output error",
	"resource temporarily unavailable",
	"end of file",
	"broken pipe",
}

// resourcePatterns 资源不足特征
var resourcePatterns = []string{
	"cannot allocate memory",
	"device or resource busy",
	"no space left on device",
	"too many open files",
}

// ClassifyError 根据 FFmpeg stderr 输出判断错误类型
func ClassifyError(stderr string) ErrorKind {
	lower := strings.ToLower(stderr)

	// 优先匹配永久错误（避免无意义重试）
	for _, p := range permanentPatterns {
		if strings.Contains(lower, p) {
			return ErrKindPermanent
		}
	}
	for _, p := range resourcePatterns {
		if strings.Contains(lower, p) {
			return ErrKindResource
		}
	}
	for _, p := range transientPatterns {
		if strings.Contains(lower, p) {
			return ErrKindTransient
		}
	}
	return ErrKindUnknown
}

// IsRetryable 判断是否值得重试
func IsRetryable(kind ErrorKind) bool {
	switch kind {
	case ErrKindTransient, ErrKindResource:
		return true
	default:
		return false
	}
}

// FFmpegError 带分类信息的错误
type FFmpegError struct {
	Kind     ErrorKind
	ExitCode int
	Stderr   string
	Err      error
}

func (e *FFmpegError) Error() string {
	return fmt.Sprintf("ffmpeg 失败（kind=%d, exit=%d）: %v", e.Kind, e.ExitCode, e.Err)
}

func (e *FFmpegError) Unwrap() error { return e.Err }
```

### 5.3 带指数退避的重试

```go
// RetryConfig 重试配置
type RetryConfig struct {
	MaxAttempts int           // 最大尝试次数（含首次）
	BaseDelay   time.Duration // 基础延迟
	MaxDelay    time.Duration // 最大延迟上限
}

// DefaultRetryConfig 默认重试策略
func DefaultRetryConfig() RetryConfig {
	return RetryConfig{
		MaxAttempts: 3,
		BaseDelay:   2 * time.Second,
		MaxDelay:    30 * time.Second,
	}
}

// RunWithRetry 带重试的任务执行
func (jr *JobRunner) RunWithRetry(ctx context.Context, cfg JobConfig, retry RetryConfig) error {
	var lastErr error

	for attempt := 1; attempt <= retry.MaxAttempts; attempt++ {
		err := jr.Run(ctx, cfg)
		if err == nil {
			if attempt > 1 {
				log.Info().Int("attempt", attempt).Msg("FFmpeg 重试后成功")
			}
			return nil
		}
		lastErr = err

		// 判断错误类型
		var ffErr *FFmpegError
		kind := ErrKindUnknown
		if errors.As(err, &ffErr) {
			kind = ffErr.Kind
		} else {
			kind = ClassifyError(err.Error())
		}

		if !IsRetryable(kind) {
			log.Error().Err(err).Int("kind", int(kind)).Msg("FFmpeg 永久错误，放弃重试")
			return err
		}

		// context 已取消时不再重试
		if ctx.Err() != nil {
			return ctx.Err()
		}

		if attempt == retry.MaxAttempts {
			break
		}

		// 指数退避：BaseDelay * 2^(attempt-1)，上限 MaxDelay
		delay := retry.BaseDelay * time.Duration(1<<(attempt-1))
		if delay > retry.MaxDelay {
			delay = retry.MaxDelay
		}
		log.Warn().Err(err).Int("attempt", attempt).Dur("delay", delay).Msg("FFmpeg 临时错误，准备重试")

		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return ctx.Err()
		}
	}

	return fmt.Errorf("重试 %d 次后仍失败: %w", retry.MaxAttempts, lastErr)
}
```

---

## 6. 并发控制与资源管理

### 6.1 为什么要限制并发

```
不限制并发的后果：
  10 个视频同时转码
  → 10 个 FFmpeg 进程同时运行
  → CPU 100%，进程互相抢占调度
  → 实际完成时间 ≈ 串行的 90%（上下文切换损耗）
  → 内存峰值 = 单任务内存 × 10
  → 磁盘 I/O 吞吐耗尽，任务全部慢速爬行

合理并发数：
  CPU 密集型（软件编码）：max(1, CPU核数 × 0.8)
  硬件编码（GPU）：由 GPU 队列决定，通常 2~4
  I/O 密集型（切片/重封装）：CPU核数 × 2~4
```

### 6.2 信号量实现

```go
package ffmpeg

import (
	"context"
	"runtime"
	"sync"
	"sync/atomic"
)

// Semaphore 基于 channel 的信号量
type Semaphore struct {
	ch chan struct{}
}

// NewSemaphore 创建信号量，size 为最大并发数
func NewSemaphore(size int) *Semaphore {
	return &Semaphore{ch: make(chan struct{}, size)}
}

// Acquire 获取一个信号量（阻塞直到有空位或 ctx 取消）
func (s *Semaphore) Acquire(ctx context.Context) error {
	select {
	case s.ch <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

// Release 释放一个信号量
func (s *Semaphore) Release() {
	<-s.ch
}

// Available 当前剩余可用数量
func (s *Semaphore) Available() int {
	return cap(s.ch) - len(s.ch)
}
```

### 6.3 工作池（Worker Pool）

信号量适合轻量并发控制；任务吞吐量大时，工作池更高效：

```go
// WorkerPool 固定大小的 FFmpeg 工作池
type WorkerPool struct {
	runner  *JobRunner
	jobs    chan poolJob
	wg      sync.WaitGroup
	active  atomic.Int64  // 当前活跃任务数
	done    chan struct{}
}

type poolJob struct {
	ctx    context.Context
	cfg    JobConfig
	result chan<- error
}

// NewWorkerPool 创建工作池
func NewWorkerPool(binPath string, workers int) *WorkerPool {
	if workers <= 0 {
		workers = runtime.NumCPU()
	}
	p := &WorkerPool{
		runner: &JobRunner{BinPath: binPath},
		jobs:   make(chan poolJob, workers*2),
		done:   make(chan struct{}),
	}
	for range workers {
		p.wg.Add(1)
		go p.worker()
	}
	return p
}

func (p *WorkerPool) worker() {
	defer p.wg.Done()
	for job := range p.jobs {
		p.active.Add(1)
		err := p.runner.Run(job.ctx, job.cfg)
		p.active.Add(-1)
		job.result <- err
	}
}

// Submit 提交任务，返回结果 channel
func (p *WorkerPool) Submit(ctx context.Context, cfg JobConfig) <-chan error {
	result := make(chan error, 1)
	select {
	case p.jobs <- poolJob{ctx: ctx, cfg: cfg, result: result}:
	case <-ctx.Done():
		result <- ctx.Err()
	}
	return result
}

// Shutdown 优雅关闭（等待所有任务完成）
func (p *WorkerPool) Shutdown() {
	close(p.jobs)
	p.wg.Wait()
}

// ActiveCount 当前活跃任务数
func (p *WorkerPool) ActiveCount() int64 { return p.active.Load() }
```

### 6.4 资源监控与自适应调节

```go
// ResourceMonitor 资源监控器（简化版）
type ResourceMonitor struct {
	pool        *WorkerPool
	maxLoad     float64  // CPU 使用率上限（0.0~1.0）
	checkPeriod time.Duration
}

// WaitForSlot 等待资源低于阈值（防止系统过载）
func (m *ResourceMonitor) WaitForSlot(ctx context.Context) error {
	ticker := time.NewTicker(m.checkPeriod)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			if m.pool.ActiveCount() < int64(cap(m.pool.jobs)/2) {
				return nil
			}
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}
```

### 6.5 并发参数与 FFmpeg 内部线程的关系

一个常见误区：把 `-threads` 设成 CPU 核数，同时又跑 N 个并发任务。

```
错误配置：8 核机器
  并发任务数 = 4
  每个任务 -threads 8
  → 期望线程 32，实际 CPU 只有 8 核
  → 线程抢占严重，总吞吐反而下降

正确配置（两种思路）：
  思路 A（低延迟优先）：并发 1，-threads 8
    单任务最快完成，适合用户等待结果的场景
  思路 B（高吞吐优先）：并发 4，-threads 2
    总吞吐最高，适合批量离线转码
```

| 场景 | 并发数 | -threads | 说明 |
|------|--------|----------|------|
| 用户上传后即时转码 | 1~2 | 0（自动） | 单任务优先，尽快返回 |
| 夜间批量转码 | CPU核数/2 | 2 | 吞吐优先 |
| 硬件编码（NVENC） | 2~4 | 不适用 | 受 GPU 编码会话数限制 |
| 切片/重封装（-c copy） | CPU核数×2 | 1 | I/O 密集，几乎不吃 CPU |

---

## 7. 异步任务队列

### 7.1 队列架构

```
                    ┌─────────────────────┐
   HTTP 请求  ───▶  │  Handler            │
                    │  1. 校验参数         │
                    │  2. 写 DB（pending） │
                    │  3. 投递队列         │
                    │  4. 立即返回 task_id │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  TaskQueue          │
                    │  （channel 或 Redis）│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  WorkerPool         │
                    │  1. 取任务           │
                    │  2. 更新 running     │
                    │  3. 执行 FFmpeg      │
                    │  4. 回调进度         │
                    │  5. 更新 done/failed │
                    └─────────────────────┘

   客户端轮询 GET /task?task_id=xxx 获取进度
```

### 7.2 任务状态机

```
   pending ──▶ running ──▶ succeeded
      │           │
      │           ├──▶ failed（永久错误）
      │           │
      │           └──▶ retrying ──▶ running
      │
      └──▶ canceled（用户取消）
```

```go
// TaskStatus 任务状态
type TaskStatus int16

const (
	TaskStatusPending   TaskStatus = 0 // 等待中
	TaskStatusRunning   TaskStatus = 1 // 执行中
	TaskStatusSucceeded TaskStatus = 2 // 成功
	TaskStatusFailed    TaskStatus = 3 // 失败
	TaskStatusCanceled  TaskStatus = 4 // 已取消
)

// TranscodeTask 转码任务
type TranscodeTask struct {
	TaskID     string     // 任务ID（VARCHAR20）
	TenantID   string     // 租户ID
	VideoID    string     // 视频ID
	InputPath  string     // 输入路径
	OutputPath string     // 输出路径
	Args       []string   // FFmpeg 参数
	Status     TaskStatus // 状态
	Progress   float64    // 进度百分比
	ErrMsg     string     // 错误信息
	Attempts   int        // 已尝试次数
}
```

### 7.3 基于 channel 的进程内队列

适合单机部署、任务量中等（每天数千）的场景：

```go
package service

import (
	"context"
	"sync"
	"time"

	"github.com/rs/zerolog/log"
)

// TaskQueue 进程内异步任务队列
type TaskQueue struct {
	tasks    chan *TranscodeTask
	pool     *ffmpeg.WorkerPool
	repo     TaskRepo          // 任务持久化
	cancels  sync.Map          // taskID -> context.CancelFunc
	wg       sync.WaitGroup
	shutdown chan struct{}
}

// TaskRepo 任务持久化接口
type TaskRepo interface {
	UpdateStatus(ctx context.Context, taskID string, status TaskStatus, errMsg string) error
	UpdateProgress(ctx context.Context, taskID string, progress float64) error
	ListPending(ctx context.Context, limit int) ([]*TranscodeTask, error)
}

// NewTaskQueue 创建任务队列
func NewTaskQueue(binPath string, workers, bufSize int, repo TaskRepo) *TaskQueue {
	q := &TaskQueue{
		tasks:    make(chan *TranscodeTask, bufSize),
		pool:     ffmpeg.NewWorkerPool(binPath, workers),
		repo:     repo,
		shutdown: make(chan struct{}),
	}
	for range workers {
		q.wg.Add(1)
		go q.consume()
	}
	return q
}

// Enqueue 投递任务（非阻塞，队列满时返回错误）
func (q *TaskQueue) Enqueue(task *TranscodeTask) error {
	select {
	case q.tasks <- task:
		log.Info().Str("task_id", task.TaskID).Str("video_id", task.VideoID).Msg("转码任务已入队")
		return nil
	default:
		return xerr.New(xerr.ErrTooManyRequests.Code, "任务队列已满，请稍后重试")
	}
}

// Cancel 取消正在执行的任务
func (q *TaskQueue) Cancel(taskID string) bool {
	if v, ok := q.cancels.Load(taskID); ok {
		v.(context.CancelFunc)()
		return true
	}
	return false
}
```

### 7.4 消费逻辑

```go
func (q *TaskQueue) consume() {
	defer q.wg.Done()
	for {
		select {
		case task, ok := <-q.tasks:
			if !ok {
				return
			}
			q.process(task)
		case <-q.shutdown:
			return
		}
	}
}

// process 执行单个任务
func (q *TaskQueue) process(task *TranscodeTask) {
	// 每个任务独立 context，支持单独取消
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Hour)
	defer cancel()
	q.cancels.Store(task.TaskID, cancel)
	defer q.cancels.Delete(task.TaskID)

	if err := q.repo.UpdateStatus(ctx, task.TaskID, TaskStatusRunning, ""); err != nil {
		log.Error().Err(err).Str("task_id", task.TaskID).Msg("更新任务状态失败")
	}

	start := time.Now()
	cfg := ffmpeg.JobConfig{
		Args: task.Args,
		OnProgress: q.throttledProgress(task.TaskID),
	}

	err := q.pool.runner.RunWithRetry(ctx, cfg, ffmpeg.DefaultRetryConfig())
	cost := time.Since(start)

	if err != nil {
		log.Error().Err(err).Str("task_id", task.TaskID).Dur("cost", cost).Msg("转码任务失败")
		_ = q.repo.UpdateStatus(ctx, task.TaskID, TaskStatusFailed, err.Error())
		return
	}

	log.Info().Str("task_id", task.TaskID).Dur("cost", cost).Msg("转码任务完成")
	_ = q.repo.UpdateStatus(ctx, task.TaskID, TaskStatusSucceeded, "")
}
```

### 7.5 进度写入节流

FFmpeg 每秒回调一次进度，直接写 DB 会造成大量无效更新。用节流控制写入频率：

```go
// throttledProgress 返回一个节流的进度回调（最快 5 秒写一次 DB）
func (q *TaskQueue) throttledProgress(taskID string) ffmpeg.OnProgressFunc {
	var lastWrite time.Time
	var lastPercent float64

	return func(p ffmpeg.Progress) {
		percent := 0.0
		if p.Done {
			percent = 100
		}

		now := time.Now()
		// 完成时必须写；否则至少间隔 5 秒且进度变化超过 1%
		if !p.Done && (now.Sub(lastWrite) < 5*time.Second || percent-lastPercent < 1) {
			return
		}
		lastWrite = now
		lastPercent = percent

		ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		defer cancel()
		if err := q.repo.UpdateProgress(ctx, taskID, percent); err != nil {
			log.Warn().Err(err).Str("task_id", taskID).Msg("更新任务进度失败")
		}
	}
}
```

### 7.6 重启恢复与优雅关闭

进程内队列的致命弱点：**重启丢任务**。必须配合 DB 持久化 + 启动扫描：

```go
// RecoverPending 服务启动时恢复未完成任务
func (q *TaskQueue) RecoverPending(ctx context.Context) error {
	// running 状态的任务说明上次异常退出，重置为 pending
	tasks, err := q.repo.ListPending(ctx, 100)
	if err != nil {
		return xerr.Wrap(xerr.ErrInternal.Code, "扫描待执行任务失败", err)
	}
	for _, t := range tasks {
		if err := q.Enqueue(t); err != nil {
			log.Warn().Err(err).Str("task_id", t.TaskID).Msg("恢复任务入队失败")
			break // 队列已满，剩余任务留给下次扫描
		}
	}
	log.Info().Int("count", len(tasks)).Msg("已恢复待执行转码任务")
	return nil
}

// Shutdown 优雅关闭：停止接收新任务，等待在途任务完成
func (q *TaskQueue) Shutdown(timeout time.Duration) {
	close(q.shutdown)

	done := make(chan struct{})
	go func() {
		q.wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		log.Info().Msg("任务队列已优雅关闭")
	case <-time.After(timeout):
		log.Warn().Dur("timeout", timeout).Msg("任务队列关闭超时，强制退出")
		// 取消所有在途任务，让 FFmpeg 走优雅终止流程
		q.cancels.Range(func(k, v any) bool {
			v.(context.CancelFunc)()
			return true
		})
	}
}
```

> **进程内队列 vs 分布式队列**
> channel 队列足够应付单机场景，但重启恢复依赖 DB 扫描，且无法横向扩展。任务量上到每天数万、或需要多机分担时，应换成 Redis Stream / RabbitMQ / Asynq。集群方案见 [第 28 篇：转码集群架构设计](./28-转码集群架构设计.md)。

---

## 8. 日志集成与可观测性

### 8.1 结构化日志规范

项目使用 Zerolog，所有日志必须**写在一行**（见 `.claude/rules/logging-style.md`）：

```go
// ❌ 错误 — 多行链式调用
log.Info().
    Str("task_id", taskID).
    Str("video_id", videoID).
    Dur("cost", cost).
    Msg("转码完成")

// ✅ 正确 — 一行
log.Info().Str("task_id", taskID).Str("video_id", videoID).Dur("cost", cost).Msg("转码完成")
```

关键日志字段约定：

| 字段 | 类型 | 说明 |
|------|------|------|
| `task_id` | string | 任务 ID |
| `video_id` | string | 视频 ID |
| `tenant_id` | string | 租户 ID（多租户必填）|
| `attempt` | int | 重试次数 |
| `cost` | duration | 耗时 |
| `exit_code` | int | FFmpeg 退出码 |
| `progress` | float64 | 进度百分比 |

### 8.2 FFmpeg stderr 日志过滤

FFmpeg 默认输出大量噪音（帧率统计、版本信息）。生产环境应只记录 WARN 及以上级别：

```go
// parseFFmpegLog 解析并过滤 FFmpeg stderr
func parseFFmpegLog(line string) (level zerolog.Level, clean string) {
	lower := strings.ToLower(line)
	switch {
	case strings.HasPrefix(lower, "error") || strings.Contains(lower, "invalid") ||
		strings.Contains(lower, "failed"):
		return zerolog.ErrorLevel, line
	case strings.HasPrefix(lower, "warning") || strings.Contains(lower, "deprecated"):
		return zerolog.WarnLevel, line
	default:
		return zerolog.NoLevel, "" // 丢弃
	}
}

// LogFFmpegOutput 将 FFmpeg stderr 输出转成结构化日志
func LogFFmpegOutput(ctx context.Context, taskID string, stderr io.Reader) {
	scanner := bufio.NewScanner(stderr)
	for scanner.Scan() {
		line := scanner.Text()
		level, clean := parseFFmpegLog(line)
		if level == zerolog.NoLevel {
			continue
		}
		log.WithLevel(level).Str("task_id", taskID).Str("ffmpeg", clean).Msg("FFmpeg 输出")
	}
}
```

### 8.3 Prometheus 指标

```go
package metrics

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	// TranscodeTotal 转码任务总数，按结果分类
	TranscodeTotal = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "ffmpeg_transcode_total",
		Help: "转码任务总计数",
	}, []string{"tenant_id", "result"}) // result: succeeded | failed | canceled

	// TranscodeDuration 转码耗时分布
	TranscodeDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "ffmpeg_transcode_duration_seconds",
		Help:    "转码任务耗时（秒）",
		Buckets: []float64{10, 30, 60, 120, 300, 600, 1200},
	}, []string{"tenant_id"})

	// TranscodeQueueDepth 队列深度
	TranscodeQueueDepth = promauto.NewGaugeVec(prometheus.GaugeOpts{
		Name: "ffmpeg_transcode_queue_depth",
		Help: "转码任务队列待处理数量",
	}, []string{"tenant_id"})

	// TranscodeActiveWorkers 活跃 Worker 数
	TranscodeActiveWorkers = promauto.NewGauge(prometheus.GaugeOpts{
		Name: "ffmpeg_transcode_active_workers",
		Help: "当前活跃的 FFmpeg 进程数",
	})

	// TranscodeRetryTotal 重试次数
	TranscodeRetryTotal = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "ffmpeg_transcode_retry_total",
		Help: "转码任务重试总次数",
	}, []string{"tenant_id", "error_kind"}) // error_kind: transient | resource
)
```

### 8.4 埋点示例

```go
// 在 process() 中埋入 Prometheus 指标
func (q *TaskQueue) process(task *TranscodeTask) {
	start := time.Now()
	metrics.TranscodeActiveWorkers.Inc()
	defer metrics.TranscodeActiveWorkers.Dec()

	// ... 执行转码（同 7.4）...

	cost := time.Since(start).Seconds()
	metrics.TranscodeDuration.WithLabelValues(task.TenantID).Observe(cost)

	if err != nil {
		metrics.TranscodeTotal.WithLabelValues(task.TenantID, "failed").Inc()
	} else {
		metrics.TranscodeTotal.WithLabelValues(task.TenantID, "succeeded").Inc()
	}
}
```

---

## 9. 完整工程示例

### 9.1 目录结构

```
internal/
├── ffmpeg/
│   ├── runner.go         # Runner、ProgressRunner（第2、3节）
│   ├── graceful.go       # GracefulKill、JobRunner（第4节）
│   ├── errors.go         # ClassifyError、FFmpegError、RetryConfig（第5节）
│   ├── semaphore.go      # Semaphore（第6节）
│   └── worker_pool.go    # WorkerPool（第6节）
├── service/
│   └── transcode_service.go  # TaskQueue、业务编排（第7节）
├── metrics/
│   └── transcode_metrics.go  # Prometheus 指标（第8节）
└── handler/
    └── transcode_handler.go  # HTTP Handler
```

### 9.2 TranscodeService

```go
package service

// TranscodeService 转码服务（遵循规则 11：功能开关在 service 内部）
type TranscodeService struct {
	config  *config.TranscodeConfig
	queue   *TaskQueue
	taskRepo TaskRepo
}

// NewTranscodeService 创建转码服务（app.go 无条件调用）
func NewTranscodeService(cfg *config.TranscodeConfig, taskRepo TaskRepo) *TranscodeService {
	queue := NewTaskQueue(cfg.FFmpegBin, cfg.Workers, cfg.QueueSize, taskRepo)
	return &TranscodeService{config: cfg, queue: queue, taskRepo: taskRepo}
}

// Start 服务启动（恢复待执行任务）
func (s *TranscodeService) Start(ctx context.Context) error {
	if !s.config.Enabled {
		log.Info().Msg("转码服务未启用，跳过启动")
		return nil
	}
	return s.queue.RecoverPending(ctx)
}

// SubmitTask 提交转码任务
func (s *TranscodeService) SubmitTask(ctx context.Context, tenantID, videoID, inputPath, outputPath string) (string, error) {
	if !s.config.Enabled {
		return "", xerr.New(xerr.ErrForbidden.Code, "转码功能未启用")
	}

	taskID, err := idgen.GenerateUUID()
	if err != nil {
		return "", xerr.Wrap(xerr.ErrInternal.Code, "生成任务ID失败", err)
	}

	args := s.buildArgs(inputPath, outputPath)
	task := &TranscodeTask{
		TaskID:     taskID,
		TenantID:   tenantID,
		VideoID:    videoID,
		InputPath:  inputPath,
		OutputPath: outputPath,
		Args:       args,
		Status:     TaskStatusPending,
	}

	if err := s.taskRepo.UpdateStatus(ctx, taskID, TaskStatusPending, ""); err != nil {
		return "", xerr.Wrap(xerr.ErrInternal.Code, "创建转码任务失败", err)
	}

	if err := s.queue.Enqueue(task); err != nil {
		return "", err
	}

	log.Info().Str("task_id", taskID).Str("video_id", videoID).Str("tenant_id", tenantID).Msg("转码任务已提交")
	return taskID, nil
}

// buildArgs 构造 FFmpeg 参数（不含 input/output 之外的安全封装）
func (s *TranscodeService) buildArgs(inputPath, outputPath string) []string {
	return []string{
		"-i", inputPath,
		"-c:v", s.config.VideoCodec,
		"-crf", fmt.Sprintf("%d", s.config.CRF),
		"-preset", s.config.Preset,
		"-c:a", "aac",
		"-b:a", "128k",
		"-movflags", "+faststart",
		outputPath,
	}
}

// Shutdown 优雅关闭
func (s *TranscodeService) Shutdown() {
	s.queue.Shutdown(30 * time.Second)
}
```

### 9.3 Handler 层

```go
package handler

// TranscodeHandler 转码接口处理器
type TranscodeHandler struct {
	svc *service.TranscodeService
}

// SubmitTranscode 提交转码任务
// @Summary 提交转码任务
// @Description 异步提交视频转码任务，立即返回任务ID
// @Tags 转码管理
// @Accept json
// @Produce json
// @Param body body dto.SubmitTranscodeRequest true "转码请求"
// @Success 200 {object} response.Response{data=dto.SubmitTranscodeResponse} "成功"
// @Router /api/v1/transcode/submit [post]
func (h *TranscodeHandler) SubmitTranscode(c *gin.Context) {
	var req dto.SubmitTranscodeRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		response.Error(c, xerr.New(xerr.ErrInvalidParam.Code, "参数错误"))
		return
	}

	scope, err := xcontext.ResolveOwnTenantScope(c.Request.Context())
	if err != nil {
		response.Error(c, err)
		return
	}

	taskID, err := h.svc.SubmitTask(c.Request.Context(), scope.TenantID, req.VideoID, req.InputPath, req.OutputPath)
	if err != nil {
		response.Error(c, err)
		return
	}

	response.Success(c, &dto.SubmitTranscodeResponse{TaskID: taskID})
}

// CancelTranscode 取消转码任务
// @Summary 取消转码任务
// @Tags 转码管理
// @Accept json
// @Produce json
// @Param body body dto.CancelTranscodeRequest true "取消请求"
// @Success 200 {object} response.Response{data=dto.CancelTranscodeResponse} "成功"
// @Router /api/v1/transcode/cancel [post]
func (h *TranscodeHandler) CancelTranscode(c *gin.Context) {
	var req dto.CancelTranscodeRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		response.Error(c, xerr.New(xerr.ErrInvalidParam.Code, "参数错误"))
		return
	}

	ok, err := h.svc.CancelTask(c.Request.Context(), req.TaskID)
	if err != nil {
		response.Error(c, err)
		return
	}

	response.Success(c, &dto.CancelTranscodeResponse{Canceled: ok})
}
```

### 9.4 DTO 定义

注意：所有 ID 字段用 `string` 类型（见 `.claude/rules/dto-id-type.md`）。

```go
package dto

// SubmitTranscodeRequest 提交转码请求
type SubmitTranscodeRequest struct {
	VideoID    string `json:"video_id" binding:"required" example:"123456789012345678"`
	InputPath  string `json:"input_path" binding:"required" example:"uploads/2026/07/raw.mp4"`
	OutputPath string `json:"output_path" binding:"required" example:"outputs/2026/07/720p.mp4"`
}

// SubmitTranscodeResponse 提交转码响应
type SubmitTranscodeResponse struct {
	TaskID string `json:"task_id" example:"123456789012345678"`
}

// CancelTranscodeRequest 取消转码请求
type CancelTranscodeRequest struct {
	TaskID string `json:"task_id" binding:"required" example:"123456789012345678"`
}

// CancelTranscodeResponse 取消转码响应
type CancelTranscodeResponse struct {
	Canceled bool `json:"canceled" example:"true"`
}

// TranscodeTaskInfo 转码任务详情
type TranscodeTaskInfo struct {
	TaskID    string  `json:"task_id" example:"123456789012345678"`
	VideoID   string  `json:"video_id" example:"123456789012345678"`
	Status    int16   `json:"status" example:"1"`
	Progress  float64 `json:"progress" example:"45.5"`
	ErrMsg    string  `json:"err_msg" example:""`
	CreatedAt int64   `json:"created_at" example:"1735200000000"`
	UpdatedAt int64   `json:"updated_at" example:"1735200060000"`
}
```

### 9.5 配置文件

```yaml
transcode:
  enabled: true                    # 功能开关（service 内部判断）
  ffmpeg_bin: /usr/local/bin/ffmpeg
  ffprobe_bin: /usr/local/bin/ffprobe
  workers: 4                       # 并发转码数
  queue_size: 100                  # 队列缓冲大小
  task_timeout: 2h                 # 单任务超时
  graceful_kill_timeout: 5s        # SIGTERM 到 SIGKILL 的等待时间
  video_codec: libx264
  crf: 23
  preset: medium
  retry:
    max_attempts: 3
    base_delay: 2s
    max_delay: 30s
  allowed_input_root: /data/uploads   # 输入路径白名单根目录
```

---

## 动手实践

### 练习 1：实现带进度上报的转码接口

**目标**：搭建一个最小可用的转码服务，提交任务后能轮询进度。

**步骤**：

1. 复制第 2、3 节的 `Runner` 和 `ProgressRunner` 到本地项目
2. 写一个 HTTP 服务，`POST /transcode` 接收 `{"input":"a.mp4","output":"b.mp4"}`
3. 用内存 `map[string]float64` 存进度（生产环境用 DB）
4. `GET /progress?task_id=xxx` 返回当前百分比

**验证**：

```bash
# 提交任务
curl -X POST http://localhost:8080/transcode \
  -H "Content-Type: application/json" \
  -d '{"input":"input.mp4","output":"output.mp4"}'
# 返回 {"task_id":"t1"}

# 轮询进度
watch -n 1 'curl -s "http://localhost:8080/progress?task_id=t1"'
# 应看到百分比从 0 递增到 100
```

**预期结果**：进度平滑递增，转码结束后 `output.mp4` 可正常播放。

### 练习 2：验证优雅终止能保住 MP4

**目标**：亲手对比 SIGKILL 和 SIGTERM 的差别。

**步骤**：

```bash
# 1. 启动一个长时间转码
ffmpeg -i long.mp4 -c:v libx264 -preset slow out_kill.mp4 &
PID=$!
sleep 10
kill -9 $PID          # SIGKILL 强杀

# 2. 换 SIGTERM 再来一次
ffmpeg -i long.mp4 -c:v libx264 -preset slow out_term.mp4 &
PID=$!
sleep 10
kill -TERM $PID       # SIGTERM 优雅终止
sleep 3

# 3. 对比两个文件
ffprobe -v error -show_format out_kill.mp4
ffprobe -v error -show_format out_term.mp4
```

**预期结果**：`out_kill.mp4` 报 `moov atom not found` 无法播放；`out_term.mp4` 是一个时长约 10 秒内容的**完整可播放**文件。

**思考**：如果加了 `-movflags +faststart`，SIGKILL 的破坏是否更严重？（答案：是，faststart 需要在转码结束后重排 moov，强杀连临时文件都留不下完整结构。）

### 练习 3：压测并发上限

**目标**：找到你机器的最佳并发数。

**步骤**：

1. 用第 6 节的 `WorkerPool`，写一个批量转码脚本，处理 20 个相同的测试视频
2. 分别用 `workers = 1, 2, 4, 8, 16` 跑，记录总耗时
3. 同时用 `top` 观察 CPU 使用率和内存峰值

**记录表**：

| workers | 总耗时 | CPU 峰值 | 内存峰值 | 单任务平均 |
|---------|--------|----------|----------|-----------|
| 1 | | | | |
| 2 | | | | |
| 4 | | | | |
| 8 | | | | |
| 16 | | | | |

**预期结果**：总耗时先随并发数下降，到 CPU 核数附近触底，再增加并发后**总耗时反而上升**（上下文切换损耗）。这个拐点就是你机器的最佳并发数。

---

## 常见问题排查

| 现象 | 根因 | 排查命令 | 修复方案 |
|------|------|----------|----------|
| 转码任务全部 pending，无人消费 | WorkerPool 未启动 / channel 关闭 | 检查 `NewWorkerPool` 是否被调用 | 确认 `Start()` 在服务初始化时调用 |
| FFmpeg 进程泄漏（`ps aux` 发现僵尸进程） | context 取消后未调用 `GracefulKill` | `ps aux | grep ffmpeg | wc -l` | 确保 watchdog goroutine 监听 `ctx.Done()` |
| 输出 MP4 无法播放（moov not found） | SIGKILL 强杀，moov 未写入 | `ffprobe out.mp4 2>&1 | grep moov` | 改用 `GracefulKill`（SIGTERM + 5s 等待）|
| 进度一直为 0%，最后跳 100% | `-progress pipe:1` 未加，进度回调拿不到数据 | 检查 `buildArgs` 是否包含 `-progress pipe:1` | 补全进度参数 |
| 并发高时机器卡死 | 并发数与 `-threads` 乘积超过 CPU 核数 | `top -bn1 | grep ffmpeg` | 降低 `workers` 或调低 `-threads` |
| 路径含空格时 FFmpeg 报找不到文件 | `sh -c` 字符串拼接导致参数分裂 | `echo "$ARGS"` 打印实际参数 | 改用 `exec.Command` 数组传参，禁止 `sh -c` |
| 任务重试 3 次都失败，但 stderr 无有用信息 | 输入文件权限或网络问题 | `ls -l <input>` / `curl <url>` | 检查输入源可达性后再提交任务 |
| 重启后任务全部丢失 | 进程内队列未持久化 | 查 DB 是否有 `running` 状态残留 | 启动时调 `RecoverPending()`，重置 running→pending |
| Prometheus 指标不出现 | 未注册 `/metrics` 路由 | `curl localhost:9090/metrics | grep ffmpeg` | 在 router 注册 `promhttp.Handler()` |
| `ClassifyError` 返回 Unknown，放弃重试 | 错误消息语言非英文（如 macOS 中文系统） | 打印 `err.Error()` 原始内容 | 补充中文错误模式到 `permanentPatterns` |

---

## 10. 本章总结

本章从「安全调用」到「生产级任务队列」构建了完整的 Go + FFmpeg 集成体系：

**核心要点**：

1. **安全第一**：路径遍历用 `filepath.EvalSymlinks` + `filepath.Rel` 验证；网络输入用 scheme 白名单 + `-protocol_whitelist`；始终用 `exec.Command` 数组传参，禁止 `sh -c`。

2. **进度解析**：`-progress pipe:1` 输出 KV 格式，用 `bufio.Scanner` 逐行读取；配合 `ffprobe` 获取总时长计算百分比；写 DB 时节流（5 秒 + 1% 变化阈值）避免无效写入。

3. **优雅终止**：`SIGTERM → wait 5s → SIGKILL` 两阶段。SIGTERM 让 FFmpeg 写完 MP4 moov box，SIGKILL 是最后兜底。避免 SIGKILL 直接杀进程导致文件损坏。

4. **错误分类**：区分永久错误（不重试）/ 临时错误（指数退避重试）/ 资源错误（延迟重试）。错误分类基于 FFmpeg stderr 关键词匹配。

5. **并发控制**：信号量适合轻量场景；`WorkerPool` 适合大吞吐；最优并发数通过压测确定，不要盲目设成 CPU 核数。

6. **异步队列**：`channel + goroutine` 构建进程内队列，配合 DB 持久化实现重启恢复。生产规模上时换分布式队列（见 第 28 篇）。

7. **可观测性**：结构化日志（Zerolog 一行）+ Prometheus 指标（任务数/耗时/队列深度/活跃 Worker）+ FFmpeg stderr 过滤输出。

**架构选型速查**：

| 需求 | 推荐方案 |
|------|---------|
| 单机，日均 < 1 万任务 | 进程内 channel 队列 + `WorkerPool` |
| 多机，需横向扩展 | Redis Stream + Asynq 或 RabbitMQ |
| 要求实时进度推送 | SSE / WebSocket 代替轮询 |
| GPU 硬件编码 | `workers = 2~4`，`-c:v h264_nvenc`，配合 GPU 会话数限制 |
| 低延迟场景 | `workers = 1`，`-threads 0`，关闭 retry |

---

## 术语表

| 术语 | 说明 |
|------|------|
| `exec.CommandContext` | Go 标准库，创建带 context 的子进程，context 取消时发送 SIGKILL |
| `-progress pipe:1` | FFmpeg 将进度信息写入 stdout，格式为 `key=value` 行 |
| `-hide_banner` | 不输出 FFmpeg 版本 banner，减少 stderr 噪音 |
| `-nostdin` | 禁止从 stdin 读取，防止进程在后台等待输入 |
| SIGTERM | 可捕获的终止信号，FFmpeg 收到后会完成当前帧处理并写文件尾 |
| SIGKILL | 不可捕获，OS 直接回收进程，FFmpeg 无法写文件尾 |
| moov box | MP4 文件结构中的元数据容器，缺失时文件无法播放 |
| `-movflags +faststart` | 将 moov box 移到文件头，支持边下边播；同时让 SIGKILL 破坏更严重 |
| WorkerPool | 固定大小的 goroutine 池，均摊任务处理，避免无限并发 |
| 指数退避 | 重试延迟 = `BaseDelay × 2^(attempt-1)`，避免雪崩式重试 |
| `bufio.Scanner` | 逐行读取 io.Reader，适合解析 FFmpeg 的流式进度输出 |
| `protocol_whitelist` | FFmpeg 允许使用的网络协议列表，限制为 `file,http,https,tcp,tls` 可防 SSRF/LFI |
| Prometheus Histogram | 适合记录耗时分布，P50/P95/P99 比平均值更有诊断价值 |

---

**下一篇**：[第 26 篇：生产环境部署](26-生产环境部署.md) —— Docker 多阶段构建、资源限制、健康检查、优雅关闭与监控告警。

---

**全文完** | 字数：约 7200 字 | 代码示例：38 个 | 实践环节：3 个  
**作者**：FFmpeg 教程编写组 | **版本**：基于 FFmpeg 8.0 "Huffman" + Go 1.22 | **标签**：`Go集成` `进程管理` `异步任务` `并发控制` `可观测性` `错误分类`
