# 第 13 篇：MCP 是什么，为什么值得学
### 从"N×M 各自焊"到"一条标准总线" · Part 5 开场

> 知识地图【E2】· Part 5 首篇
> 读完你能理解：没有 MCP 的世界乱在哪（N×M 问题）；MCP 的三个角色与协议白话版
> （server 声明 → client 获取 → 代模型调用 → 回结果）；JSON-RPC 2.0 与 HTTP 的区别；
> stdio 与 Streamable HTTP 怎么选、OAuth 摆在哪；2026 年生态现状——最后动手：给
> Claude Code 挂一个 example server，观察完整调用 trace。

## 本篇要回答的问题

1. 没有 MCP 的世界乱在哪？"N×M 问题"是什么？
2. MCP 的"宿主 / 客户端 / 服务端"各指什么？它和我们写了十篇的工具循环是什么关系？
3. 协议白话版：一次 MCP 工具调用从头到尾发生什么？JSON-RPC 2.0 和 HTTP 是什么关系？
4. 传输怎么选：stdio vs Streamable HTTP？OAuth 和授权摆在哪一层？
5. 2026 年 MCP 生态现状：规范版本、官方 SDK、参考 servers、MCP Registry？
6. 动手：给 Claude Code 挂一个 example server，从哪里看到完整的调用 trace？

---

## 1｜N×M 问题：每一次集成都重新发明轮子

先回看我们自己走过的路：第 8-9 篇，为了把管理后台接给助手，我们在 `tools.py` 里
手写了 5 个工具（login / list_users / list_roles / get_user_roles / assign_role），
配了 auth 注入、错误处理、确认钩子——**一套只属于"我们这个 agent"的私有集成**。

现在把镜头拉远。假设你希望同一个管理后台还能被别的"大脑"使用：

```text
没有 MCP 的世界（N × M 份集成）：

  我们的助手 ────┐
  Claude Code ───┼──→ 管理后台 API   （自己焊一份）
  Cursor ────────┼──→ 管理后台 API   （再焊一份）
  未来的 X ──────┴──→ 管理后台 API   （又一份）
  ……每个 agent 客户端 × 每个系统的集成，都要重写一遍
  （而且每家的工具描述格式、错误约定、认证方式还都不一样）

有了 MCP 的世界（N + M 份工作）：

  我们的助手 ─┐                        ┌─ 管理后台 server（写一次）
  Claude Code ─┼─── MCP 标准总线 ──────┼─ Git server（社区已有）
  Cursor ──────┘                        └─ 文件系统 server（官方参考）
  ……System 侧写 M 个 server，Agent 侧接 N 个 client，谁也不认识谁
```

**MCP（Model Context Protocol）就是这个标准总线**。它有一个人人都会举的类比——
LSP（Language Server Protocol）：当年每个编辑器×每种语言都要各自实现补全/跳转，
M×N 爆炸；LSP 把"编辑器 ↔ 语言工具"标准化之后，VS Code 支持一门新语言的成本从
"重写一遍"变成"接一个 LSP server"。MCP 的故事一模一样，只是主角从编辑器换成了
**agent 工具**。

对我们 Part 5 的意义：**P3 手写的那 5 个管理工具，要升级成一个 MCP server**——
升级之后，我们的助手能用、Claude Code 能挂、任何支持 MCP 的客户端都能用；到
P5·15，还能让 Go 后台自己提供这个 server（里程碑 M4 的另一半）。

---

## 2｜MCP 是什么：三个角色，一条总线

### 角色

```text
┌─────────────── Host（宿主应用）────────────────┐
│  Claude Code / 我们的 agent 程序 / IDE 插件     │
│   ┌─ Client ─┐  ┌─ Client ─┐  ┌─ Client ─┐    │   1 个宿主管 N 个 client
│   │  连接器  │  │  连接器  │  │  连接器  │    │   1 个 client 连 1 个 server
│   └────┬─────┘  └────┬─────┘  └────┬─────┘    │
└────────┼─────────────┼─────────────┼──────────┘
         ▼             ▼             ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐
    │ Server  │   │ Server  │   │ Server  │   本地子进程（stdio）
    │ 后台管理 │   │  Git    │   │ …       │   或远程服务（HTTP）
    └─────────┘   └─────────┘   └─────────┘
```

- **Host**：宿主应用（Claude Code、桌面客户端、我们自己的 agent）；
- **Client**：宿主内部的连接器，一个 client 对接一个 server，负责把宿主和 server
  之间的消息转发、协商、鉴权；
- **Server**：能力的提供方——**我们 P3 写的那 5 个工具，就是未来 server 的全部
  内容**。

Server 对外提供的能力叫**原语（primitives）**：本系列主线用 **Tools**（模型可调用的
动作）；规范还定义了 Resources（可读数据，更像"文件"）、Prompts（可复用提示模板）
等，用到再说（`<以官方 spec 为准>`）。

### 它和我们写的工具循环是什么关系？

这是"不神秘化 MCP"的关键一问。把 P2·7 的本地循环摊开对照：

| 我们的本地 loop（P2·7-P3） | MCP 里的对应物 |
|---|---|
| `TOOLS_SCHEMA`：本地写死的工具描述 | `tools/list`：server 声明的工具清单（name / description / inputSchema） |
| `registry`：本地函数字典 | **server 进程本身**：工具的真实执行者 |
| `run_tool(name, args)`：本地函数调用 | `tools/call`：一次 JSON-RPC 请求 |
| 函数返回值喂回模型 | `result.content` 喂回模型 |
| 工具报错（P3·9 的"喂回可决策文本"） | `isError: true` + 内容 |

看清楚了吗——**MCP 没有一个新概念**。它就是：把我们本地 loop 里的"工具三件套"
（schema / 执行者 / 调用通道）标准化，然后搬到**进程边界或网络边界**上。本地调用
`run_tool()` 变成了"发一条 JSON-RPC 消息给另一个进程"，其余一切（模型怎么选工具、
结果怎么回填）原封不动。

所以前面 12 篇不是白学的：**MCP 的每个设计，你都能在我们自己写过的循环里找到原型**。

---

## 3｜协议白话版：一次调用四拍

```text
① 握手      client → server:  initialize      （协议版本、能力协商）
② 列工具    client → server:  tools/list       ← 相当于加载 TOOLS_SCHEMA
③ 代模型调用 client → server:  tools/call       ← 模型决定调哪个之后
④ 回结果    server → client:  result           ← 喂回模型（我们的 tool_result 位）
```

一帧消息长什么样（示意，`<以官方 spec 为准>`）：

```json
// ③ client → server：代模型调用（对我们的 assign_role）
{"jsonrpc": "2.0", "id": 3, "method": "tools/call",
 "params": {"name": "assign_role", "arguments": {"user_id": "1001", "role_id": "2002"}}}

// ④ server → client：结果（成功）或 isError（失败也是"数据"，同 P3·9）
{"jsonrpc": "2.0", "id": 3,
 "result": {"content": [{"type": "text", "text": "已为 1001 分配角色 2002"}], "isError": false}}
```

### JSON-RPC 2.0 和 HTTP 是什么关系？

**别混淆两层**：JSON-RPC 是**消息格式**（这些 JSON 长什么样），传输是**怎么送**
（stdio 管道或 HTTP 请求）。表格对比它和我们熟悉的 REST：

| | HTTP REST（P3 的目标服务） | JSON-RPC 2.0（MCP 消息格式） |
|---|---|---|
| 风格 | 资源 + 方法语义（`GET /users/1001`） | 远程方法调用（`"method": "tools/call"`） |
| 方法名 | 双方按 REST 惯例约定 | **规范定义**（initialize / tools/list / tools/call…） |
| 返回 | 状态码 + body | `{"result": ...}` 或 `{"error": {...}}` |
| 配对 | 请求-响应天然成对 | 显式 `id` 字段配对（支持并发多请求） |
| 心智 | "取/改一个资源" | "调一个方法，等一个结果" |

JSON-RPC 比 REST**更"机械"**——没有资源建模的发挥空间，一切严格按 schema 来。这
正是协议要的：机器之间说的话，越没歧义越好。

### 协议版本协商

`initialize` 握手时双方交换**规范版本号**（如当前 `2026-07-28`）——client 说"我按
这个版本说话"，server 回"我支持到哪一版"，谈不拢就好聚好散。你以后升级 SDK/
挂新 server 报的"版本不兼容"，就是这一拍没谈成。

---

## 4｜传输与授权

同一套 JSON-RPC 消息，两种送法：

| | **stdio** | **Streamable HTTP** |
|---|---|---|
| server 在哪 | 宿主的**子进程**（同一个机器） | **远程服务**（跨机器/跨网络） |
| 通道 | stdin / stdout 管道 | HTTP POST（可选 SSE 流式回推） |
| 鉴权 | 基本不需要——**进程边界即信任边界** | 需要：OAuth 等（见下） |
| 适用 | 本地工具：文件系统、本地后台、开发调试 | 团队/公司共享服务、公网 SaaS 端点 |
| 我们的用法 | **P5·14 的 MCP server 用它** | P5·15 讨论桥接时再展开 |

（历史一句：早期规范用过 "HTTP + SSE 双端点"方案，后被 Streamable HTTP 取代——
`<以官方 spec 为准>`。）

**OAuth 与授权**：Streamable HTTP 的 server 在网络上，谁都能敲，就得回答"你是谁、
你能用它干什么"。所以远程 server 的鉴权走 OAuth 一族（`claude mcp add` 里就有
`--client-id` / `--client-secret` / `--callback-port` 这些 OAuth 参数——我们下面
实测的 help 里能看到）。本地 stdio **不需要**这一层——能起这个子进程的人本来就有
这台机器的权限。

最后一条认知（从 P3·10 长出来的）：**MCP server 就是一个权限边界**。server 暴露
哪些工具，客户端就最多能做哪些事——"工具集收窄"从单机的设计原则升级成了**跨进程
的架构原则**：以后别人（或别的团队）要用你的后台，你**别开放 API，开放一个收窄了
工具集的 MCP server**。授权体系的深水区（scope、token 传递、审批）留到 P5·15 和
P7·20。

---

## 5｜2026 现状速览（编写环境实测，2026-10-16）

| 项目 | 现状 |
|---|---|
| 规范版本 | **2026-07-28**（官方 specification/latest 实测指向该版本） |
| Python SDK | PyPI 包 `mcp`，版本 **2.3.0**（P5·14 的主角；v2 起主类更名 `MCPServer`，即 v1 的 FastMCP） |
| 官方 SDK 语言 | **10 个**：TypeScript / Python / Go / C# / Java / Kotlin / PHP / Ruby / Rust / Swift（servers 仓库 README 一手列出） |
| SDK 热度（GitHub ★ 实测） | python-sdk 24.6k / typescript-sdk 13.6k / **go-sdk 5.2k**（你 P5·15 的主战场） |
| 参考 servers 仓库 | `modelcontextprotocol/servers`，**91.3k ★**；官方维护 7 个参考实现：Everything（测试用）/ Fetch / Filesystem / Git / Memory / Sequential Thinking / Time |
| 找现成 server 去哪 | **MCP Registry**（registry.modelcontextprotocol.io）——官方已把 GitHub/Slack/SQLite 等早期示例转入归档，推荐去注册表找现成的 |
| 客户端支持 | Claude Code（`claude mcp add` 一等公民，下面实测）、Claude Desktop、Cursor 等 |

一句话判断：**规范定了、SDK 齐了、客户端接了、生态从"官方示范"走向"注册表分发"**
——现在学 MCP，正好卡在从"前沿"变"标配"的中点。

---

## 6｜动手实践：给 Claude Code 挂一个 example server

本节命令的**语法**已在编写环境核验（`claude mcp --help` / `claude mcp add --help`
实测）；挂载与观察在你的环境完成。我们挂官方参考 server 里的 **Memory**（知识图谱
记忆——正好像第 11 篇学过的"persist 长期记忆"，这次由 MCP server 代管）。

### 步骤

**① 挂载**（stdio 型：`--` 之后就是 server 的启动命令）：

```bash
claude mcp add memory -- npx -y @modelcontextprotocol/server-memory
```

（需要 node/npx。如果换 Python 系参考 server，官方 README 用 `uvx`，如
`uvx mcp-server-git`——语法同理。作用域用 `-s, --scope` 指定，可选 local / user /
project，默认 local——`--help` 实测。）

**② 查连接**：

```bash
claude mcp list        # 看 server 与连接状态
```

会话里输入 `/mcp` 也能查看 server 连接状态和它暴露的工具清单——**这份清单就是
`tools/list` 那一拍的可视化**。

**③ 驱动它**：在 Claude Code 里说一句自然语言，例如

> 用 memory 记住：我的教程项目在 tmp/agent-building-tutorial，主题是从零构建 agent。

**④ 观察完整 trace**，三个层次：

1. **界面上**：会出现一次工具调用块（server 名 + 工具名 + 参数）——这就是"模型
   决定 → client 代调 → 结果回填"的可视化，和我们 loop 的 `log_step` 是同一件事；
2. **server 侧**：stdio server 的日志走 stderr（stdout 是协议通道），可以在终端里
   直接运行同一命令观察：`npx -y @modelcontextprotocol/server-memory`——它会安静
   地等你发 JSON-RPC 消息；
3. **原始协议帧**（进阶）：官方调试器 **MCP Inspector**（`npx
   @modelcontextprotocol/inspector`，`<以官方 docs 为准>`）能可视化 initialize /
   tools/list / tools/call 的每一帧——本文 §3 的那些 JSON，在那里能看到真的。

**⑤ 玩别的**：想更直观，把 Filesystem server 挂到一个测试目录（**先建目录**，注意
server 的访问控制参数）：

```bash
mkdir -p /tmp/mcp-playground
claude mcp add fs -- npx -y @modelcontextprotocol/server-filesystem /tmp/mcp-playground
```

然后让 Claude"列出 /tmp/mcp-playground 里的文件"——观察它调 `fs` 的工具而不是
内置工具。

### 观察点小结（对应前文的每个概念）

- 你**没写一行工具代码**，Claude Code 就获得了新能力 → §1 的 N+M；
- `/mcp` 列出的工具清单来自 server 自报 → §3 的 `tools/list`；
- 工具调用的形态和我们 `agent/` 项目里的一模一样 → §2 的心智模型对照表。

---

## 本篇结论

1. **MCP 解决的是集成爆炸**：N 个 agent × M 个系统从 N×M 份重复集成变成 N+M——
   LSP 故事在 agent 时代的翻版；本 Part 的主线就是把我们 P3 的 5 个工具升级成
   server（第 14 篇）。
2. **MCP 没有新概念，只有标准化**：它就是本地工具循环（TOOLS_SCHEMA / registry /
   run_tool）挪到进程或网络边界——JSON-RPC 2.0 是消息格式（不是新 HTTP），stdio
   与 Streamable HTTP 是两种传输；**server 即权限边界**，工具集收窄从设计原则变为
   架构原则。
3. **生态已到"标配前夜"**（2026-10-16 实测）：规范 2026-07-28、10 语言官方 SDK、
   servers 仓库 91.3k ★、MCP Registry 分发、Claude Code 等客户端一等公民——现在
   学，正好。

---

**下一篇预告**：第 14 篇《把管理助手工具升级为 MCP server（MCPServer，原 FastMCP）》——用 Python
SDK（`mcp` 2.3.0）把我们 P3 的 5 个工具改写成 `@mcp.tool()` 声明式 server，agent
侧从"import 工具函数"改成"MCP client 连接"，同一句话验证两版等价。

**等我回复「继续」。**
