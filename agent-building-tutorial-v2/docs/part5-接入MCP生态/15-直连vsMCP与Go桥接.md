# 第 15 篇：直连 vs MCP — 选型与 Go 后台桥接

> **知识地图【E2/F1】· Part 5** · 第 14 篇把工具搬进了 Python 版 MCP server，本篇算清这笔账：多一个进程、多一层协议，到底换到了什么？什么时候直连就够、什么时候必须 MCP？最后把桥接落到真实形态——官方 Go SDK 写 server，Python agent 跨语言调用（**里程碑 M4**）。这也是「成熟方案三条路线」里路线 ② 的正式落地。

## 本篇要回答的问题

1. 直连 vs MCP，六个维度（学习成本 / 标准化 / 调试 / 部署 / 多客户端复用 / 权限模型）各差在哪？每一维「什么时候不选它」？
2. 生产上的最终选型是什么——条件怎么判，本项目（Go 后台、内部管理助手）选哪条路？
3. 真实 Go 后台怎么桥接：两种架构各长什么样？官方 go-sdk 怎么用？
4. 里程碑 M4 怎么验收：Python agent 调通 Go MCP server，证据看什么？

---

## 1｜六维对比：直连 vs MCP

先摆总表，再逐维展开。表里的每一格都能在前四篇里找到实证——对比不是拍脑袋，是把已经跑过的两条路摆在一起看。

| 维度 | 直连（第 8-10 篇） | MCP（第 13-14 篇） | 什么时候不选它 |
|---|---|---|---|
| 学习成本 | 零新概念：HTTP + JSON，后端工程师的主场 | +协议（JSON-RPC）/传输（stdio/HTTP）/握手/发现，一套新词汇 | 团队只有一个 agent、一种语言、没有第二消费方——学协议的钱花不回来 |
| 标准化 | 私有协议：TOOL_SCHEMAS 只在本进程有意义 | 开放标准：任何 MCP client 即插即用，工具描述与实现跨进程跨语言 | 永远不会有第二个 client——标准化换来的「复用权」没人来兑现 |
| 调试 | 单进程函数调用栈，断点直接打 | 跨进程：报文在管道里，看得见（stdio 可抓、有 Inspector）但多一层；stdout 纪律这类新坑（第 14 篇 §2.3） | 排障预算紧、工具又经常变——每多一层间接就多一层排查成本 |
| 部署 | 工具与 agent 同进程，一起起一起停 | server 独立进程：可跨机、跨语言、独立升级；但要管子进程生命周期（第 14 篇常驻连接的取舍） | 一次性脚本、本地小工具——为它常驻一个 server 进程是杀鸡用牛刀 |
| 多客户端复用 | 无：换语言/换客户端只能重写工具层 | 一份 server，所有 client 共用：Claude Code / Cursor / 自研 agent 连的是同一个 | 只有一个消费方时，「复用」不存在 |
| 权限模型 | agent 进程内自管：闸门/校验/审计都在自己手里（第 9-10 篇） | 分层治理：校验审计随工具进 server（任何 client 自动获得），闸门留 client（策略各归各），双审计各记各账（第 14 篇 §4） | 权限需求极简（全只读工具）——分层治理的设计成本没有对价 |

逐维展开：

**① 学习成本**。直连版（第 8 篇）的全部新知识是「把 REST 接口包成函数注册进 TOOL_REGISTRY」——一个 urllib 调用加一份 schema。MCP 版（第 13-14 篇）新增了协议、传输、握手、发现、isError 语义、stdout 纪律一整套词汇。这笔学费不白交（它是「跨进程标准化」的地基），但如果你永远只用一种语言一个 client，学费就纯是成本。

**② 标准化**。直连的工具清单是私有的：`TOOL_SCHEMAS` 的 shape 只有你的 agent 进程认识。MCP 把它变成公共协议：`tools/list` 的返回（第 13 篇报文逐字段讲过）任何 MCP client 都认识。第 14 篇场景 3 的「动态发现」就是标准化的直接红利——server 加了工具，client 重连一次就能看到，不用改 client 代码。

**③ 调试**。直连时工具抛异常，调用栈一屏看完；MCP 时错误要跨进程传回（isError + 错误文本回喂，第 14 篇 §3.2），中间多一层协议。补偿是**报文可观察**：stdio 信道可以原样抓取（第 13 篇 raw_client.py 就是这么干的），协议栈的黑白盒程度介于「函数调用」和「RPC 黑盒」之间——偏白盒，但要先学会看。

**④ 部署**。直连 = 工具随 agent 走；MCP = server 是独立部署单元。独立带来三个能力：跨语言（本篇 §3 的 Go server）、跨机器（Streamable HTTP 传输，第 13 篇 §3 的远程形态）、独立升级（server 改版不用动 agent）。代价是进程管理：第 14 篇已经付过一笔（常驻连接、优雅关停、子进程随会话生命周期）。

**⑤ 多客户端复用**。这是 MCP 的核心卖点，也是 N×M→N+M 那张图（第 13 篇 §1）的落地形态：M 个系统各出一个 server，N 个 agent 各做 client，工具实现份数从 N×M 降到 M。判断标准很硬：**出现第二个消费方的那一刻**（比如「想在 Claude Code 里直接查后台用户」），MCP 的边际成本就回本了。

**⑥ 权限模型**。直连时保护全在 agent 进程里，agent 的作者说了算；MCP 把保护拆成两层（第 14 篇 §4 分层表）：server 端的校验与审计是**任何 client 都绕不过**的底线，client 端的确认策略**各归各**。对治理的意义：后台团队把校验放进 server，就不用信任每个 client 作者的自觉——权限从「约定」升级为「结构」。

## 2｜生产选型：条件化判定 + 本项目的答案

**判定条件**（按出现顺序问自己）：

1. 只有一个 agent、一种语言、工具不外传？→ **直连**。学习成本为零，第 8 篇一天就能闭环。
2. 出现第二个消费方（第二个 agent / IDE 集成 / 跨语言团队）？→ **MCP**。第二份工具实现的成本，就是回本点。
3. 后台由专门团队维护、要统一权限与审计？→ **MCP，且 server 跟后台走**（校验审计进 server，第 14 篇分层表的治理含义）。
4. 要对外提供服务、跨网络？→ **MCP + Streamable HTTP 传输**（stdio 只适合本机父子进程，第 13 篇 §3 场景表）。

**本项目的答案**：Go 后台（用户/角色/权限域，JWT + RBAC + 多租户）、内部管理助手、Python agent 起步。

- **起步（现在）**：直连。路线 ① 在 M2/M3 已验证——工具层包 HTTP client，不动老系统，最快可验证（三条路线里官方指南推荐的最简做法）。
- **标准化窗口（第二个消费方出现时）**：上 MCP，**server 用 Go 写**（语言随系统走——后台团队维护自己的工具出口，这正是路线 ② 的分工）。本篇 §3 就是这一步的预演。
- **规模化之后**：治理平台化（路线 ③，身份/成本/审计平台）——第 21 篇用 Agentic OS 视角收口时再回来对照。

一句话版：**直连是验证想法的路，MCP 是沉淀能力的路；从 1 到 2 个消费方的那一步，就是换路的信号。**

## 3｜Go 后台桥接：两种架构与 go-sdk

### 3.1 两种架构图

先明确一点：第 8 篇的 FastAPI 脚手架（mock_backend）只是路线 ① 的**前置验证**——用最便宜的方式证明「agent 能驱动一个 REST 后台」。它不落地。真实落地从本篇开始，而且形态换成 Go（与真实后台同语言）。

**架构 B（现状，路线 ①）**——agent 直连 REST：

```
┌─────────────────────┐   HTTP/JSON + JWT    ┌──────────────────────┐
│  Python agent 进程   │ ───────────────────→ │  Go 后台（REST API）  │
│  loop + 工具函数     │   /api/v1/users ...  │  Gin + JWT + RBAC    │
│  （工具=urllib 调用） │ ←─────────────────── │                      │
└─────────────────────┘     JSON 响应         └──────────────────────┘
  工具实现住在 agent 进程里（admin_tools.py）——Go 侧对 agent 一无所知
```

**架构 A（本篇落地，路线 ②）**——Go 进程当 MCP server，Python agent 连：

```
┌─────────────────────┐  stdio（JSON-RPC）   ┌──────────────────────┐   HTTP/JSON + JWT
│  Python agent 进程   │ ───────────────────→ │  Go MCP server 进程   │ ────────────────→ ┌─────────┐
│  loop（零改动）      │  tools/list / call  │  mcpserver 包 + 工具   │   Go 后台 REST     │ Go 后台  │
│  MCPToolset client  │ ←─────────────────── │  （校验/审计住这层）   │ ←──────────────── │         │
└─────────────────────┘   工具结果（文本）    └──────────────────────┘     JSON 响应       └─────────┘
  工具实现搬到 Go 侧——「包内部服务时，server 语言随系统语言走」（后台团队维护工具出口）
  同一个 Go server 还可以被 Claude Code / Cursor / 任何 MCP client 复用（N×M→N+M）
```

两张图的关键差异只有一处：**工具实现住在哪一侧**。架构 B 里它是 agent 的私产（Python）；架构 A 里它是后台的标准出口（Go），agent 只剩一个通用 client。

### 3.2 官方 go-sdk：用法片段

官方 Go SDK（`github.com/modelcontextprotocol/go-sdk`，规格清单 2026-10-15 核验存在；包结构 `mcp` / `mcpclient` / `mcpserver`，支持 OAuth）。以下代码按官方 README 的公开用法写成，**本环境未实际构建运行，API 细节 `<以官方文档为准>`**——动手前先对照仓库当前版 README：

```go
// go-mcp-server/main.go — M4：官方 go-sdk 写的最小 admin 工具 MCP server（约 45 行）
// 架构 A 的 Go 侧：工具实现调后台 REST（这里仍指向 mock_backend 做演示——
// 接真实 Go 后台时只改 BASE_URL 与各接口的 path/字段映射）。
// <未实测，以官方文档为准>：go-sdk 的类型名/函数签名以
// github.com/modelcontextprotocol/go-sdk 当前 README 为准。
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"os"
	"strings"

	"github.com/modelcontextprotocol/go-sdk/mcp"
	"github.com/modelcontextprotocol/go-sdk/mcpserver"
)

const baseURL = "http://127.0.0.1:8000" // 同第 8 篇：本地脚手架；真实后台只改此行

var token string // 登录态进程内存持有（第 8 篇决定二，第 14 篇 §4.1：持有者换成 server 进程）

// call 后台的统一出口（对位 admin_tools._call 的极简版）：自动带 Authorization
func call(method, path string, body map[string]any) (map[string]any, error) {
	var reqBody strings.Reader
	if body != nil {
		b, _ := json.Marshal(body)
		reqBody = *strings.NewReader(string(b))
	} else {
		reqBody = *strings.NewReader("")
	}
	req, _ := http.NewRequest(method, baseURL+path, &reqBody)
	req.Header.Set("Content-Type", "application/json")
	if token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	var out map[string]any
	if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
		return nil, err
	}
	return out, nil
}

func main() {
	// 声明 server + 注册工具：对位第 14 篇 admin_mcp_server.py 的 MCPServer + @mcp.tool()
	// 具体签名（NewServer / AddTool / Run）以官方 README 为准 <以官方文档为准>
	server := mcpserver.NewServer(&mcp.Implementation{Name: "admin-tools-go", Version: "0.1"}, nil)

	mcpserver.AddTool(server, &mcp.Tool{Name: "login",
		Description: "登录后台管理系统。演示账号 admin/admin123。令牌由本 server 自动保管。"},
		func(ctx context.Context, req *mcp.CallToolRequest, args struct {
			Username string `json:"username"`
			Password string `json:"password"`
		}) (*mcp.CallToolResult, any, error) {
			out, err := call("POST", "/api/v1/auth/login",
				map[string]any{"username": args.Username, "password": args.Password})
			if err != nil {
				return mcp.NewToolResultError(err.Error()), nil, nil // 错误回喂，不炸连接（第 7 篇同哲学）
			}
			if t, ok := out["access_token"].(string); ok {
				token = t
				delete(out, "access_token") // 令牌摘除：不进报文、不进 client（第 8 篇纪律）
				out["status"] = "ok"
			}
			b, _ := json.Marshal(out)
			return mcp.NewToolResultText(string(b)), nil, nil
		})

	mcpserver.AddTool(server, &mcp.Tool{Name: "get_users",
		Description: "分页查询后台用户列表，keyword 按用户名模糊搜索。"},
		func(ctx context.Context, req *mcp.CallToolRequest, args struct {
			Keyword string `json:"keyword"`
		}) (*mcp.CallToolResult, any, error) {
			q := url.Values{"keyword": {args.Keyword}, "page": {"1"}, "page_size": {"20"}}
			out, err := call("GET", "/api/v1/users?"+q.Encode(), nil)
			if err != nil {
				return mcp.NewToolResultError(err.Error()), nil, nil
			}
			b, _ := json.Marshal(out)
			return mcp.NewToolResultText(string(b)), nil, nil
		})

	// stdio 传输：stdin/stdout 收发 JSON-RPC（与第 14 篇 Python server 同一形态）
	if err := server.Run(context.Background(), os.Stdin, os.Stdout); err != nil {
		fmt.Fprintln(os.Stderr, "server 退出：", err) // 日志走 stderr——stdout 纪律同第 14 篇 §2.3
	}
}
```

三个对位读法（Go 侧 ≈ Python 侧的什么）：

| go-sdk 构件 | 第 14 篇 Python 对位 | 说明 |
|---|---|---|
| `mcpserver.NewServer` + `AddTool` | `MCPServer(...)` + `@mcp.tool()` | 声明式注册；Go 用结构体 tag 做 schema 注解（`json:"username"`），对位 pydantic 类型注解 |
| `mcp.NewToolResultError` | 返回 error dict + isError | 错误回喂不炸连接 |
| `server.Run(stdin, stdout)` | `mcp.run()`（stdio） | 同一根管道、同一套报文——client 完全不需要知道 server 是什么语言写的 |

**这就是跨语言价值的直接证据**：Python agent 的 `MCPToolset` 连接命令从 `[python, admin_mcp_server.py]` 换成 `[go-mcp-server 可执行文件]`，loop、注册件接口、闸门、审计一行不改。协议是两边的公共语言。

### 3.3 与 FastAPI 脚手架的解耦（再强调一次）

mock_backend（FastAPI）存在的唯一理由是「让第 8-14 篇每一步都可在本机跑通」——它是路线 ① 的验证道具。本篇的 Go server 仍然调它做演示，但**把 `baseURL` 指向真实 Go 后台、按真实接口改 path/字段映射**之后，架构 A 的其余部分（Python client、保护分层、agent loop）原样成立。真实后台的 JWT / RBAC / 多租户如何与 agent 侧身份对齐，是第 20 篇 F1 治理清单的主菜。

## 4｜动手实践：里程碑 M4——Python agent 调通 Go MCP server

> M4 验收口径：Python agent 通过 MCP 协议调用 Go 写的 server 完成一次真实工具往返——**跨语言桥接成立**。本实践本环境未实际构建运行（网络拉取 Go 依赖受限），全部命令按可复现步骤给出，`<未实测，以官方文档为准>`；已在第 14 篇实测的部分（Python 侧连接机制）沿用其证据。

### 第 1 步：构建 Go server

```bash
cd agent/go-mcp-server
go mod init admin-tools-mcp            # 或你喜欢的 module 名
go get github.com/modelcontextprotocol/go-sdk@latest
# 国内网络加速：go env -w GOPROXY=https://goproxy.cn,direct
go build -o admin-mcp-server .         # 产出可执行文件 admin-mcp-server
```

依赖版本以 `go list -m all` 输出为准；go-sdk 的当前版本号 `<以官方文档为准>`（仓库 releases 页）。

### 第 2 步：Python agent 连接 Go server

第 14 篇的 `MCPToolset.build(command=...)` 支持自定义 server 命令——这就是当时埋的口。最小验证脚本（不花 LLM token，验证的是桥本身）：

```python
# go_bridge_check.py — M4 最小验证：Python client × Go server 的一次真实往返
# <未实测，以官方文档为准>：Go server 构建成功后本脚本应可直接跑通
import json
import mcp_client_tools

ts = mcp_client_tools.build(command=["./go-mcp-server/admin-mcp-server"])
print(f"已连接 {ts.server_info}（协议 {ts.protocol_version}），"
      f"发现 {len(ts.TOOL_SCHEMAS)} 个工具：{sorted(ts.TOOL_REGISTRY)}")
print(json.loads(ts.TOOL_REGISTRY["login"](username="admin", password="admin123")))
print(json.loads(ts.TOOL_REGISTRY["get_users"](keyword="张三")))
ts.close()
```

### 第 3 步：完整闭环（agent 主入口）

```bash
# 终端 1：后台脚手架照旧（演示阶段仍用 mock_backend；接真实后台时换成真实服务）
python -m uvicorn mock_backend.main:app --port 8000
# 终端 2：agent 以 MCP 模式跑，server 是 Go 写的
python agent.py --mcp "李四现在是什么角色？"     # --mcp 的 server 命令默认指向 Python 版；
                                                  # 连 Go 版需把 DEFAULT_SERVER_COMMAND 指到
                                                  # ./go-mcp-server/admin-mcp-server（或用第 2 步脚本先验证桥）
```

预期输出（结构示意；具体措辞以实跑为准，`<未实测>`）：

```text
[MCP] 已连接 server {'name': 'admin-tools-go', 'version': '0.1'}（协议 2025-11-25），发现 2 个工具：['get_users', 'login']
[loop] 第 1 轮 | 发送 2 条消息
[工具] login({"username": "admin", "password": "admin123"})
[工具→] {"status": "ok", "assignable_roles": [...], ...}
[loop] 第 2 轮 | 发送 4 条消息
[工具] get_users({"keyword": "李四"})
[工具→] {"total": 1, "users": [{"user_id": "u-1002", ...}]}
...
```

### 第 4 步：把 server 加回完整工具集（选做）

本篇示例只包了 login + get_users 两个工具（控篇幅）；补齐 `get_user_roles` / `assign_role` 时，**保护分层照搬第 14 篇的表**：语义白名单（user_id 正则 + role 枚举）写在 Go 工具函数里（server 端），确认闸门不需要写——它本来就住在 Python client 的 `execute_tool` 里，Go server 无需知情。双审计同理：Go server 记到达的调用，client 记本侧决定。

### 自检清单

1. `go build` 产出可执行文件，`go list -m all` 里 go-sdk 版本可查；
2. go_bridge_check.py 能打印 `serverInfo: admin-tools-go` 与 tools/list 发现结果——**跨语言握手成立**（M4 核心证据）；
3. login 往返后打印的 JSON 里没有 access_token 字段（令牌摘除纪律在 Go 侧同样成立）；
4. agent 主入口跑一句话任务，loop 日志正常终止；
5. Go server 的日志只出现在 stderr/stdout 协议流之外（stdout 纪律）；
6. 第 14 篇回归全绿：p9/p10/p11/p12/p14（Python 侧没有因连接 Go server 改动任何核心逻辑——改的只是连接命令）。

## 本篇结论

1. 六维对比的胜负手只有一条：**是否出现第二个消费方**。单 agent 直连够，第二消费方出现即 MCP，server 语言随后台走。
2. 本项目选型：起步直连（M2/M3 已验证），标准化窗口到了用 go-sdk 包一层 Go server（本篇架构 A），治理平台化留给规模化之后（路线 ③）。
3. M4 的意义：同一套 MCP 报文，Python client 连 Go server——工具实现从此跟后台走、跟语言解耦，这就是第 13 篇 N×M→N+M 图的第一次真实兑现。

## Part 5 收口：里程碑 M4 复验与下一 Part 预告

**M4 复验**：吸收 Part 5 增量（Python 版 server → 直连/MCP 对照 → Go 桥接）后，前序里程碑仍立——selfcheck 回归锚点（p9/p10/p11/p12/p14 共 213 项断言）在 Part 5 全程保持全绿；M2 闭环（直连）与 MCP 版（第 14 篇 demo_mcp_equiv 7 项）各自可跑；M4 的跨语言往返按本篇实践验收（Go 侧构建状态如实标注）。

**下一 Part 会暴露什么问题**：到现在为止，我们只有一个 agent——上下文、记忆、规划、工具调用全挤在一个循环里。任务一复杂，上下文互相污染、token 预算互相挤占。成熟系统（Claude Code / Cursor / OpenHands）是怎么拆的？第 16 篇先拆解它们的设计，第 17 篇把模式落到自己的助手（主 agent + subagent）。

## 参考资料

1. [MCP 官方 Go SDK（modelcontextprotocol/go-sdk）](https://github.com/modelcontextprotocol/go-sdk) — mcp / mcpclient / mcpserver 包，OAuth 支持。仓库存在性 2026-10-15 核验（系列规格清单）；**本篇 Go 代码未经本机构建，API 签名以官方 README 当前版为准**。
2. [MCP 规范仓库（modelcontextprotocol/modelcontextprotocol）](https://github.com/modelcontextprotocol/modelcontextprotocol) — 规范 tag `2026-07-28`（2026-10-15 GitHub tags API 核验）；tools/list / tools/call 报文定义。
3. [第 13 篇：MCP 是什么，为什么值得学](13-MCP是什么.md) — 六维对比表里「标准化/传输/报文」论断的上游证据。
4. [第 14 篇：用 FastMCP（现 MCPServer）把管理助手工具升级为 MCP server](14-FastMCP包装.md) — 保护分层表（本篇 Go 侧工具补齐的施工图）、MCPToolset.build(command=...) 的连接口、双审计纪律。
5. [第 8 篇：把后台 API 变成 Agent 的工具](../part3-接上真实系统/08-后台API变成工具.md) — 直连版（路线 ①）与 FastAPI 脚手架的定位声明（本篇 §3.3 解耦声明的上游）。
6. [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 「集成 agent 与现有系统从最简做法起步」的工程判断出处（路线 ① 的官方指南背书；第 1 篇已核验）。

> 核验说明：本篇对比表的论断均来自第 8-14 篇已实测/已核验的系列事实（出处逐条回链）；Go 侧代码与构建命令**未经本机实测**（网络拉取 Go 依赖受限），全部标注 `<未实测，以官方文档为准>`——这是系列写作规范第 6 条「拿不准的一律显式标注」的执行，不是含糊其辞。
