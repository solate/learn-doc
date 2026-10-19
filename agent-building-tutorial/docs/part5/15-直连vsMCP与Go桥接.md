# 第 15 篇：直连 vs MCP — 选型与 Go 后台桥接
### 六维对比给出最终选型 · 官方 Go SDK 最小 server · Part 5 收口 · 里程碑 M4

> 知识地图【E2/F1】· Part 5 收口
> 读完你能：在"直连 REST"与"MCP server"之间做有依据的选型（六维对比 + 场景清单）；
> 用官方 Go SDK（v1.8.0）写声明式 MCP server；让 P5·14 的 Python 宿主**一行不改**
> 连上 Go server 完成跨语言调用；复验里程碑 M4。

## 本篇要回答的问题

1. 直连 vs MCP 六维对比（学习成本 / 标准化 / 调试 / 部署 / 多客户端复用 / 权限模型）
   ——什么时候直连就够了？
2. 什么信号出现时必须上 MCP？生产环境的最终选型建议是什么？
3. 官方 Go SDK 长什么样：30 行量级怎么写出一个声明式 server？
4. Python Agent ↔ Go MCP Server 怎么接——宿主为什么可以一行不改？
5. 路线 ①（教程脚手架）与路线 ②（真实后台落地）是什么关系？里程碑 M4 复验。

---

## 0｜全景：我们已经走过的两条路

Part 5 开头（13 篇）说过：**Part 5 的目标，是把 M2 的 5 个工具从"Python import"升级为
MCP 标准**。三篇走到现在，手里有两份对照物：

```text
路线 A · 直连 REST（P3 的形态）
  agent/main.py（宿主）
    ├── 工具实现（5 个函数 + token/重登/_shape）   ← 全在宿主进程里
    └── requests ──HTTP──▶ 目标后台 REST API

路线 B · MCP（P5·14 的形态）
  agent/main.py（宿主）
    └── MCPToolbox ──stdio(JSON-RPC)──▶ admin_mcp_server/server.py
                                          └── HTTP ──▶ 目标后台

本篇之后 · 路线 B 换一颗"心脏"（M4 的另一半）
  agent/main.py（宿主）               ← 一行都不用改（只换启动命令）
    └── MCPToolbox ──stdio──▶ admin-mcp-go（Go 写的 server）
                                └── HTTP ──▶ Go 后台（真实落地目标）
```

注意第三张图的重点：**宿主侧完全不动**。MCPToolbox 只会"起子进程 + 讲 JSON-RPC"，
它不关心（也无法知道）server 是 Python 还是 Go 写的——这就是 13 篇 LSP 类比里
"协议是唯一接口"的字面意义，也是里程碑 M4 要验收的东西。

**路线 ①/② 一句话**：P3 的 FastAPI `target_service` 是**路线 ①
的教学脚手架**（只为把概念跑通）；**路线 ② 才是真实落地**——给真实 Go 后台加 MCP 入口。
本篇用一个**最小 Go module** 演示路线 ② 的形态；照抄到真实后台、加上权限与审计的完整
工程化，留到你学完后的落地设计文档（这是本系列一开始就划好的边界）。

---

## 1｜六维对比：什么时候直连够、什么时候必须 MCP

不做宗教争论，只看六个维度。每一行都给"哪条路线更占优 + 代价是什么"：

| 维度 | 直连 REST（P3） | MCP server（P5） |
|---|---|---|
| **学习成本** | 会 HTTP 就会——`requests` + JSON，没有新概念 | 多学协议/SDK/进程边界；但声明式把手写 schema 省了回来（14 篇：34 行 → 0 行） |
| **标准化** | 每个 agent × 每个系统一份私有集成（N×M） | 写一次，**任何 MCP 客户端都能用**（N+M，13 篇的起点问题） |
| **调试** | 单进程：print、断点，所见即所得 | 多一层：进程树 / Inspector / "stdout 是协议通道"三规矩（14 篇）；换来的是 SDK 帮你校验参数（非法参数进不了函数） |
| **部署** | 一个进程，起 agent 就行 | 多一个部署单元；本地 stdio 零网络配置，远程要 Streamable HTTP + OAuth（13 篇） |
| **多客户端复用** | ❌ 工具代码绑死在这个 Python 宿主里；Claude Code 用不上 | ✅ 核心红利：同一 server 挂 Claude Code / IDE / 别的 agent（14 篇步骤五的做法） |
| **权限模型** | 靠代码自觉；"能干什么"散在 5 个函数里 | **server 即权限边界**（13 篇）：暴露哪些工具，客户端就最多做哪些事；敏感状态还能留在 server 进程（14 篇 token 升级） |

### 选型清单（贴墙版）

**直连就够的信号**（满足 ≥3 条就别急着上 MCP）：

- 只有一个 agent 消费这组能力，且短期内没有第二个客户端；
- 自己团队开发维护，接口和 agent 一起迭代；
- 还在快速原型期——需求每周变，协议层会拖慢你；
- 认证/会话逻辑本来就属于 agent 宿主（比如"用当前用户的 token 做事"）。

**该上 MCP 的信号**（出现 ≥1 条就值得上）：

- ① **同一能力要被 ≥2 个客户端用**——特别是"还想让 Claude Code / IDE 用"；
- ② **跨团队/跨语言**——后台团队维护 server，agent 团队只消费（本篇的主角场景）；
- ③ **现成生态直接吃**——13 篇的 memory/filesystem/git server 挂上就能用；
- ④ 需要"**工具集 = 审批过的能力清单**"这种可审计的权限边界；
- ⑤ **凭证/敏感状态要离开 agent 进程**（14 篇的"更小的盒子"）。

### 生产最终选型建议（本系列的结论）

> **原型期直连、生产期 MCP、混合期并存。** 具体到"后台管理"这类**长期存在、天然会被
> 多方使用**的系统：**按 MCP server 设计**——先从只读工具放开（`list_users` 先上），
> 写工具收窄+审批（`assign_role` 后置），身份用 service account（见第 4 节）。
> 这不是教条：如果你的 agent 永远只有自己用、永不外接，直连的简单性就是正确答案——
> **代价换收益，先把收益说清楚再换**。

---

## 2｜Go 侧入场：官方 Go SDK（v1.8.0）

你的主战场到此为止都在 Python；从这一节起，**后台侧用 Go**——毕竟真实系统的这一头，
本来就是 Go 工程师的地盘。

> 一手事实（2026-10-16 官方 README / releases 核验）：
> `github.com/modelcontextprotocol/go-sdk`，**5,231 ★**，最新 **v1.8.0（2026-09-14）**；
> **v1.7.0+ 支持规范 `2026-07-28`**（与 Python SDK 2.3.0 同一代）；SDK 是**单一主包
> `mcp`**（另有 `jsonrpc` / `auth` / `oauthex` 三个辅助包）；安装
> `go get github.com/modelcontextprotocol/go-sdk/mcp`。官方策略：新版本只支持仍受
> 支持的 Go 版本——用较新的 Go 工具链（编写环境核验时用的是 go1.27.1）。

### API 速览：和 Python 版对照着看

官方 README 的入门示例就是全部心智模型："create an `mcp.Server` instance, add features
to it, and then run it over an `mcp.Transport`"。和 14 篇逐一对照：

| 概念 | Python SDK（`mcp` 2.3.0） | Go SDK（v1.8.0） |
|---|---|---|
| server 实例 | `mcp = MCPServer("name")` | `server := mcp.NewServer(&mcp.Implementation{Name: "…", Version: "…"}, nil)` |
| 注册工具（声明式） | `@mcp.tool()` + 类型注解 + docstring | `mcp.AddTool(server, &mcp.Tool{Name, Description}, handler)`；参数用 **Input 结构体**声明 |
| schema 从哪来 | 类型注解自动生成 | Input 结构体自动推断（官方："If `Tool.InputSchema` is unset, the input schema is **inferred from the `In` type**"）；`json` 标签 = 参数名，`jsonschema` 标签 = 参数说明 |
| 返回纯文本 | `return "文本"` | `return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: "…"}}}, nil, nil` |
| 工具报错 | `raise ToolError(...)` → `is_error` | `return nil, nil, err` → 官方："stored in the CallToolResult and **IsError** is set to `true`" |
| 启动（stdio） | `mcp.run()` | `server.Run(ctx, &mcp.StdioTransport{})` |

结论一句话：**声明式是 SDK 的共性，不是 Python 的特权**。"类型声明即契约"在两边都成立
——Python 靠注解、Go 靠结构体，SDK 都从声明里生成 `tools/list` 的 `inputSchema`。

---

## 3｜30 行量级：写一个 Go 版 admin MCP server

口径先说清：M4 清单里说的"30 行"指**最小骨架**——去掉 import、注释、把错误分支压扁
之后的核心量。下面这版把错误处理写全（70 行左右，含 import 与注释），因为这是 P3·9 教过的事：
"能跑"和"敢用"之间差的就是这些行。

新建 `admin_mcp_go/main.go`：

```go
// admin_mcp_go/main.go —— 后台管理 MCP server（Go 最小版）。
// 声明式工具 + stdio 传输；Python 宿主用 P5·14 的 MCPToolbox 原样连接。
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

// 入参结构体：json 标签 = 参数名；jsonschema 标签 = 给模型的参数说明。
// SDK 从这里推断 inputSchema —— 与 Python 版"类型注解即契约"是同一个设计。
type ListUsersInput struct {
	Keyword string `json:"keyword" jsonschema:"用户名或姓名，可为空字符串"`
}

func listUsers(ctx context.Context, req *mcp.CallToolRequest, in ListUsersInput) (*mcp.CallToolResult, any, error) {
	base, token := os.Getenv("ADMIN_API_BASE"), os.Getenv("ADMIN_TOKEN")
	if base == "" || token == "" {
		// 子进程环境不继承（P5·14 规矩①）：宿主必须用 env= 显式传
		return nil, nil, fmt.Errorf("缺少 ADMIN_API_BASE / ADMIN_TOKEN，宿主需用 env= 显式传入")
	}
	httpReq, err := http.NewRequestWithContext(ctx, "GET",
		base+"/users?keyword="+url.QueryEscape(in.Keyword), nil)
	if err != nil {
		return nil, nil, err
	}
	httpReq.Header.Set("Authorization", "Bearer "+token)
	resp, err := http.DefaultClient.Do(httpReq)
	if err != nil {
		return nil, nil, fmt.Errorf("调用后台失败：%v", err)
	}
	defer resp.Body.Close()
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, nil, err
	}
	if resp.StatusCode != 200 {
		// 与 P3·9 的 _shape 同一规矩：错误塑成"可决策文本"，SDK 会自动置 IsError
		return nil, nil, fmt.Errorf("调用失败：HTTP %d（%s）", resp.StatusCode, body)
	}
	// 目标服务的外层信封是 {"code":0,"data":…}；给模型的文本只取 data（与 P3 的 _shape 契约一致）
	var envelope struct {
		Data json.RawMessage `json:"data"`
	}
	if err := json.Unmarshal(body, &envelope); err != nil {
		return nil, nil, fmt.Errorf("响应不是预期 JSON：%v", err)
	}
	return &mcp.CallToolResult{
		Content: []mcp.Content{&mcp.TextContent{Text: string(envelope.Data)}},
	}, nil, nil
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "admin-mcp-go", Version: "0.1.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{
		Name:        "list_users",
		Description: "按用户名或姓名查询后台用户，返回 user_id 等信息。",
	}, listUsers)
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

几个和前面文章咬合的点：

- **handler 的三段返回**：`(*mcp.CallToolResult, any, error)` 是官方 `AddTool` 的通用
  形态（`Out` 用 `any` 表示"不走结构化输出，我自己构造结果"）；返回 `TextContent`
  就是 P3/P5·14 的"给模型的文本"契约——模型侧看到的东西两边完全一样。
- **错误 = 数据**：`fmt.Errorf` 返回的错误，SDK 自动"stored in the CallToolResult and
  IsError is set to true"——**换语言不换铁律**（P3·9）：任何失败都变形为可读文本喂回
  模型，它自己决策重试还是放弃。真实版照 P3·9 把 hint 表补全（"404 别重试"这类处置
  建议是集成知识，与用什么语言写 server 无关）。
- **配置从 env 读**：这不是学 Python 版的样子，而是 stdio 子进程世界的规定（14 篇
  规矩①：环境白名单、不继承）——host 不给，server 就只能报"缺变量"。

### 编译

```bash
mkdir -p admin_mcp_go && cd admin_mcp_go
go mod init admin-mcp-go
go get github.com/modelcontextprotocol/go-sdk/mcp    # v1.8.0
go build -o admin-mcp-go .                           # 产出二进制
```

> 为什么要先 `go build` 成二进制、而不是让宿主 `go run .`？两个现实原因：`go run`
> 首次要现场编译（秒级到几十秒，而 P5·14 外壳的单次调用超时是 30 秒）；`go` 工具链
> 自己的构建缓存又依赖 `HOME` 等环境——恰好是子进程白名单里有的那几个，但没必要赌。
> **编译产物 + 绝对路径**是最稳的挂法。

---

## 4｜桥接验证：宿主一行不改，只换"启动命令"

现在兑现第三张架构图。新增 `agent/bridge_test.py`——注意它和 P5·14 的
`mcp_tools.py` **一个字都没改**：

```python
"""bridge_test.py —— M4 桥接验证：P5·14 的 MCPToolbox 原样复用，这次连的是 Go server。"""
import os

import requests
from dotenv import load_dotenv

from mcp_tools import MCPToolbox

load_dotenv()                                     # 读 agent/.env（P2·7 起的约定）

BASE = os.environ.get("ADMIN_API_BASE", "http://127.0.0.1:8000/api/v1")
SERVER = os.path.join(os.path.dirname(os.path.abspath(__file__)),
                      "..", "admin_mcp_go", "admin-mcp-go")

# 宿主先登录拿 token（演示版形态；生产形态见本节末"真实落地的三个差异"）
tok = requests.post(f"{BASE}/login", json={
    "username": os.environ["TARGET_USERNAME"],
    "password": os.environ["TARGET_PASSWORD"],
}).json()["data"]["token"]

box = MCPToolbox(
    command=SERVER,
    args=[],
    env={"ADMIN_API_BASE": BASE, "ADMIN_TOKEN": tok},   # 子进程白名单：显式传（P5·14 规矩①）
)
print("工具清单：", [t.name for t in box.tools])
print("调用返回：", box.call("list_users", {"keyword": "张三"}))
box.close()
```

运行（终端 1 保持 P3·8 的 `target_service` 在跑）：

```bash
cd agent && python bridge_test.py
```

**预期输出**：

```text
工具清单： ['list_users']
调用返回： {"total": 1, "list": [{"user_id": "1001", "username": "zhangsan", "display_name": "张三", "status": 1}]}
```

第二行数据来自一个 Go 进程——而宿主代码和连 Python server 时**完全相同**。这就是
"MCP 把 N×M 变成 N+M"在你手里落地的瞬间：`MCPToolbox(command=…)` 换成任意语言的
server 都成立，因为协议是唯一接口。

### 真实落地的三个差异（演示版 → 生产版）

1. **登录/凭证归 server**：演示版宿主帮忙登录再传 token；真实形态里，Go server 用
   **service account** 在进程内部完成认证（就像 P5·14 的"更小的盒子"讨论）——
   宿主连 token 概念都不需要。这也是"真实落地里模型不该管凭证"的方向：P3 把 `login`
   当工具是**教学脚手架**（让模型体验凭证错误的恢复）；生产系统里那是服务身份的事，
   到 P7·20 的身份映射一节展开。
2. **目标服务从 mock 换成真实后台**：把 `ADMIN_API_BASE` 指向真实 Go 后台（或干脆
   在 server 内部直连其数据层）。教程到此为止——给真实后台加 MCP 入口的完整落地，
   是学完系列后另开设计文档的事（路线 ②）。
3. **P3·9 的工程件照补**：401 自动重登、hint 处置表、审计——同一套规矩用 Go 重写一遍
   而已；**设计不变，语言随意**，这也是 Part 5 全部三篇想说的话。

---

## 5｜Part 5 收口：里程碑 M4 复验

M4 的定义（13 篇开篇时立下）：**MCP 版助手跑通 + Python Agent ↔ Go MCP Server
桥接成功**。复验四条（全过即 M4 完成）：

- [ ] **① MCP 版助手回归**：P5·14 步骤四的"给张三加上运营"仍完整跑通（含确认钩子
      提示、`audit.jsonl` 宿主审计）
- [ ] **② Go 桥接成功**：本篇 `bridge_test.py` 输出张三数据（上面那两行）
- [ ] **③ 宿主零改动的证据**：`git diff agent/mcp_tools.py` 为空——桥接复用的就是
      P5·14 的同一个文件（"标准化"的直接证据）
- [ ] **④ 多客户端（可选加验）**：`claude mcp add admin-go -- /绝对路径/admin_mcp_go/admin-mcp-go`
      （env 的传法以 `claude mcp add --help` 为准），Claude Code 也能查同一批数据——同一 server、两个宿主

### Part 5 五篇回头一看

13 篇讲清"为什么"（N×M、总线、server 即权限边界）→ 14 篇完成"搬"（工具进 server、
宿主接 client、P3 五设计点体检）→ 15 篇完成"选"与"跨"（六维选型、Go 桥接、M4 收口）。
**到这里，你的 agent 已经是"标准接口"的一等公民了**：Python 写、Go 写、给 Claude Code
挂、给未来的任何 host 挂——工具层的故事到此闭环。

**但下一个问题已经不是"能力"，而是"组织"**：工具越挂越多、上下文越挤越满、一个
复杂任务（"把这个租户的权限重审一遍"）单线程的 loop 开始捉襟见肘。下一 Part：先拆开
**Claude Code / Cursor / OpenHands** 看成熟系统怎么组织——然后落地我们自己的多智能体。

---

## 本篇结论

1. **选型有据可依**：原型直连、生产（且会被多方复用时）MCP；判断信号看"客户端数量、
   团队/语言边界、生态复用、权限边界、凭证归属"五条——本系列对"后台管理"类系统的
   最终建议是：**MCP server + 只读先行 + service account**。
2. **Go SDK 把"声明式"带到了后台侧**：`mcp.NewServer` + `mcp.AddTool` + Input 结构体，
   schema 照样自动生成；handler 返回文本/错误的行为与 Python 版一一对应——**换语言
   不换设计**。
3. **M4 达成路径**：宿主只换一行启动命令就连上 Go server（协议是唯一接口）；复验四条
   全过，Part 5 闭环。

---

**下一篇预告**：第 16 篇《拆解 Claude Code / Cursor / OpenHands / OpenClaw》（Part 6 开场 ·
实例先行）：三层架构共性图（模型适配 / harness / 应用层）、subagent 与 hooks、权限
提示与 checkpoint——对照我们第 7-15 篇的成品"已自然实现哪些、缺哪些"。

**等我回复「继续」。**
