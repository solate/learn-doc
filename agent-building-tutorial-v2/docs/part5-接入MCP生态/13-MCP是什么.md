# 第 13 篇：MCP 是什么，为什么值得学

> **知识地图【E2】· Part 5**
> Part 5 开篇。[第 12 篇](../part4-记忆与规划/12-规划能力设计.md)收口时留了钩子：M3 助手的记忆、规划、护栏都齐了，但工具仍是 **python import 的私有协议**——换语言要重写、外部客户端无从接入。本篇补「标准化」这块拼图：MCP（Model Context Protocol，模型上下文协议）解决什么问题、协议长什么样、传输怎么选、授权怎么办、2026 年生态到哪一步。读完你就能回答「为什么不把工具函数直接 import 进去就完事了」，并亲眼看过一次 MCP 的完整报文。正经写 server 和桥接 Go 后台分别是第 14、15 篇的事——本篇只用一个 12 行的最小演示 server 观察协议，不写任何业务工具。

## 本篇要回答的问题

1. **解决什么问题**：第 7 篇那套 `TOOL_SCHEMAS` + `TOOL_REGISTRY` 是私有协议，它缺了什么？为什么没有标准化，N 个 agent 接 M 个系统就会写出 N×M 套重复胶水？MCP 怎么把它压成 N+M？
2. **协议白话版**：server 声明工具清单 → client 获取 → 代模型调用 → 回结果，这四步怎么落在 JSON-RPC 2.0 报文上？对着你熟的 HTTP/REST，`initialize` / `tools/list` / `tools/call` 每个 method、报文里每个字段对应什么？
3. **传输怎么选**：stdio（本地子进程）和 Streamable HTTP（远程服务）各适合什么场景？曾经无处不在的 SSE 去哪了？远程 server 的 OAuth 授权是谁给谁授权、什么场景才需要？
4. **2026 现状**：规范最新版本是多少、官方 SDK 覆盖哪些语言、参考 servers 仓库里有哪些现成可挂的 server？
5. **动手实践**：亲手用裸 JSON-RPC 驱动一个最小 MCP server，看全「握手 → 列工具 → 调工具 → 回结果」的每一条报文；再把同一个 server 挂进 Claude Code，看外部客户端怎么用同一份工具。

## 1｜为什么需要 MCP：从我们的私有协议说起

读完本节你能理解：第 7 篇的工具层是「只活在一个进程里」的私有协议；一旦出现第二个 agent 或第二个系统，集成的边数就会按乘法爆炸；MCP 的全部价值 = 把乘法变加法。

### 1.1 私有协议的寿命：一个进程

回顾第 7 篇搭的工具层（[07-手写最小agent.md](../part2-亲手写出第一个Agent/07-手写最小agent.md) §1.3）——三件套缺一不可：

```python
TOOL_SCHEMAS   = [...]                     # 给模型看的 JSON Schema（进请求体的 tools 字段）
TOOL_REGISTRY  = {"get_time": get_time, ...}  # 名字 -> Python 函数（loop 阶段④ 查表执行）
SCHEMA_BY_NAME = {...}                     # 名字 -> schema（执行前参数校验）
```

第 8-12 篇一路演进（登录即工具、确认闸门、审计、记忆、规划），这套形状没变过——变的只是注册进来的工具更多、闸门更厚。它好学、可控、零依赖，**在学习阶段是优点；在生产阶段是三重枷锁**：

- **绑语言**：`TOOL_REGISTRY` 的值是 Python 函数对象。你的后台是 Go（本教程项目的真实后台就是），想让 agent 用上它的接口，要么用 Python 重写一遍 HTTP client（第 8 篇的 FastAPI 脚手架就是这么来的——它是「为验证 agent 而生的替身」，不是真后台），要么换 Go 重写 agent。工具逻辑与 agent 语言被焊死。
- **绑进程**：schema 和实现都在同一个 Python 进程的内存里。Claude Code、Cursor、同事写的另一个 agent——任何**跑在别的进程里**的客户端，都无法 import 你的 `TOOL_REGISTRY`。
- **绑格式**：`tools` 字段是 OpenAI/DeepSeek 的对话补全格式，换成别家模型的 API 就是另一套工具定义格式（第 6 篇对比过 OpenAI 兼容与 Anthropic 兼容两种接入面）。同一个工具要接两家模型，schema 就得维护两份。

一句话：**私有协议 = 工具只能被「写它的那个 agent」用**。这不是代码质量问题，是集成模型问题——下面用算术看它多贵。

### 1.2 N×M 集成爆炸 → N+M

设想生态里有 N 个 agent 应用（你的后台助手、Claude Code、Cursor、同事写的运维 agent……）和 M 个想被调用的系统（你的 Go 后台、公司邮箱、数据库、Jira……）。没有统一协议时，**每个 agent 想用每个系统，就得为「那个 agent 的语言/格式」写一遍该系统的工具适配**：

```text
没有标准化：N × M 条私有集成线（每条 = 一套 schema + 一套实现 + 一套错误处理）

   agent A(Python 后台助手) ────┬──── 后台 REST (Go)
   agent B(Claude Code)     ────┼──── 后台 REST (Go)      ← 同一系统，B 又写一遍
   agent C(Cursor)          ────┼──── 后台 REST (Go)      ← 再写一遍
   agent A                  ────┼──── 邮箱 / 数据库 / Jira
   agent B                  ────┴──── 邮箱 / 数据库 / Jira
   ……
        3 agent × 4 系统 = 12 套胶水代码，格式各异、各自维护、各自出 bug
```

这你很熟——它就是 **API 没有统一规范年代的集成地狱**：每个系统一套私有 API、每个调用方一套 client。后来 OpenAPI（Swagger）把「一个 HTTP 服务怎么被描述」标准化了，才有了「写一份 spec，所有语言的 client 生成器都认」。MCP 在 agent 工具层做的事与此同构：**在中间插一层人人认同的协议，把 N×M 压成 N+M**：

```text
有了 MCP：N + M 条标准线（每条 = 说一遍 MCP）

   agent A ──┐                      ┌── MCP server（包着后台 REST）
   agent B ──┼── 都会说 MCP ────────┼── MCP server（包着邮箱）
   agent C ──┘   （MCP client 侧）  └── MCP server（包着数据库）
                                        （MCP server 侧）

   N 个 agent 各写一次「说 MCP 的 client」（N 条）
   M 个系统各包一次「说 MCP 的 server」（M 条）
   中间任意组合都成立：3 agent × 4 系统 = 3 + 4 = 7 份协议实现
```

省掉的不只是代码量，还有**不一致性**：第 8 篇给工具描述定了规矩（描述即流程、参数可校验），私有协议下每个集成方各自执行、质量参差；MCP 把「工具怎么声明、参数怎么校验、结果怎么回传」写进规范，一份工具实现在所有 client 处行为一致。生态里现成的 MCP server（第 5 节盘点）因此可以即插即用——挂上就能用，这就是「标准」的复利。

### 1.3 MCP 一句话定义（不神秘化）

**一句话大白话**：MCP 是一套约定——工具提供方（server）用统一格式自报家门「我有哪些工具、参数长什么样」，agent 侧（client）用统一报文问清单、传调用、收回执。你的工具逻辑一行不改，包一层协议就获得了跨语言、跨进程、跨客户端的接入能力。

**它不是什么**：不是模型厂商的私有接口、不是又一个 agent 框架、也不接触模型本身——模型既看不到 MCP 也说不了 MCP，**把它翻译成对话补全 `tools` 字段的仍是你的 agent 代码**（第 7 篇 §1.4 的结论在 MCP 下原样成立，只是「schema 来源」从进程内 dict 换成了 `tools/list` 响应）。类比：MCP 之于 agent 工具，约等于 OpenAPI 之于 REST 服务 + 设备驱动之于操作系统——第 21 篇的 Agentic OS 视角会回收这个类比（MCP server 像设备驱动、系统调用层）。

**准确定义与出处**：MCP（Model Context Protocol）是一个开放协议，为「应用程序向 LLM 提供上下文与工具」定义标准化的连接方式，采用 JSON-RPC 2.0 作为报文格式（出处：MCP 规范仓库 [github.com/modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol)，现行最新规范 tag `2026-07-28`，2026-10-16 经 GitHub tags API 核验仍是最新——本篇所有协议细节均以该版本规范为准，下同）。协议由 Anthropic 于 2024 年 11 月开源并规范发布，2025-2026 年成为主流 agent/IDE 的事实标准（第 1 篇路线图里「P5 接入 MCP」指的就是它）。

### 1.4 私有协议 vs MCP：一张对照表收束

| 维度 | 第 7 篇私有协议（TOOL_SCHEMAS + REGISTRY） | MCP（本 Part 起的标准层） |
|---|---|---|
| schema 定义在哪 | agent 进程内的 Python dict | server 端声明，`tools/list` 拉取 |
| 谁能调用这些工具 | 只有 import 它的那个 Python agent | 任何说 MCP 的 client（agent、IDE、CLI） |
| 换模型/换 agent 语言 | schema 格式随模型厂商变，工具实现跟语言走 | client 负责翻译成各家模型的 tools 格式，工具实现不动 |
| 加一个新系统 | 在 agent 里新增 N 处代码 | 包一个 server（M 侧 +1），全部 agent 立即可用 |
| 错误处理/审计/闸门 | agent 进程内自定义（第 9-10 篇） | 协议层有标准回传通道（`isError`），闸门策略仍在各端 |
| 学习成本 | 零（就是写函数） | 多学一层协议（本篇 + 第 14 篇） |
| 适合谁 | 单 agent、单语言、快速验证 | 多 agent / 多客户端 / 跨语言 / 需要生态复用 |

表里最后一行是成本项——本篇不回避它：MCP 是**加在工具和 agent 之间的一层间接**，第 15 篇会给「什么时候直连就够、什么时候必须 MCP」的完整选型结论。先把协议本身看懂，选型才有依据。

## 2｜协议白话版：JSON-RPC 2.0 是 MCP 的普通话

读完本节你能理解：MCP 报文就是 JSON-RPC 2.0 信封 + MCP 语义的 method 名；对着你熟的 HTTP/REST，每个字段都有对位物；一次会话 = 握手、列工具、调工具三段。

### 2.1 先看信封：JSON-RPC 2.0 的请求、响应、通知

**一句话大白话**：JSON-RPC 2.0 是一种「用 JSON 写的信封格式」——把「我想调哪个方法、传什么参数」装进一个 JSON 对象发过去，对方回一个装着 `result` 或 `error` 的 JSON 对象。它和 HTTP 是两回事：HTTP 解决「字节怎么送到对面」，JSON-RPC 解决「信里写什么」。MCP 选择它作为唯一报文格式（stdio 上一行一条，HTTP 上一个 POST 一条）。

三种信封（出处：[JSON-RPC 2.0 规范](https://www.jsonrpc.org/specification)，字段名逐字核对过）：

```text
① 请求（Request）——期待响应               ② 响应（Response）——成功
{"jsonrpc": "2.0",                        {"jsonrpc": "2.0",
 "method": "tools/call",                   "result": {...},
 "params": {...},                          "id": 2}
 "id": 2}

③ 通知（Notification）——不带 id、不期待响应     ④ 响应——失败
{"jsonrpc": "2.0",                        {"jsonrpc": "2.0",
 "method": "notifications/initialized"}    "error": {"code": -32601,
                                                    "message": "Method not found"},
                                           "id": 1}
```

对着你熟的 HTTP/REST 建立对位（本系列的对位习惯：新协议不长在真空里）：

| JSON-RPC 2.0 | 你熟悉的 REST 对位 | 说明 |
|---|---|---|
| `jsonrpc: "2.0"` | 无（固定值） | 版本标记，像 HTTP 报文里的 `HTTP/1.1` 首行 |
| `method` | 「URL + 动词」合体 | 命名空间用 `/` 分隔（`tools/list`），一个字符串说清「调什么」 |
| `params` | 请求体 + path/query 参数 | 对象（命名参数）或数组（位置参数），MCP 全用对象 |
| `id` | 无直接对位（多路复用里的 stream id 最接近） | 请求与响应的关联键；同一连接上可并发多个未决请求，靠 id 配对 |
| `result` / `error` | 200 响应体 / 4xx-5xx 响应体 | **二者互斥**；`error` 里 `code` 是负整数（`-32601` = Method not found，JSON-RPC 规范预定义段） |
| 通知（无 `id`） | fire-and-forget 的单向推送（如 webhook） | 发了就走、永不回包——MCP 用它发「我准备好了」「清单变了」这类单向告知 |

两个后端视角的要点：其一，`id` 让**单条连接承载并发请求**成为可能（不像 HTTP/1.1 的请求-响应串行）；其二，「响应里 `result`/`error` 互斥、通知永不响应」是 JSON-RPC 的硬规则——本篇动手实践里你会亲眼看到一个真实踩坑：initialize 请求忘了带 `id`，server 把它当通知「收到不回」，client 卡死在等响应上。

### 2.2 一次会话四步走：声明 → 获取 → 代调用 → 回结果

**一句话大白话**：MCP 会话就是一个四步对话——client 启动 server 后先握手确认「咱们说哪个版本的协议、各自支持哪些能力」，再问「你有哪些工具」，然后**模型每次想调工具时，由 client 把调用转给 server 执行、把结果带回来**。时序：

```text
  你的 agent 进程（MCP host/client）                MCP server 进程
  ┌────────────────────────────┐                ┌──────────────────────┐
  │ agent loop（第 3/7 篇）      │                │ 工具实现（任意语言）    │
  │  模型想调工具时……            │   ① initialize │                      │
  │  loop 阶段④ 不再查本地 REGISTRY─┼───────────────>│ 校验版本、交换能力      │
  │  而是把调用转给 MCP client    │<───────────────┤ 报 protocolVersion/   │
  │                            │   ② initialized │ capabilities/serverInfo│
  │                            │──(单向通知)─────>│                      │
  │                            │   ③ tools/list  │                      │
  │  拉到清单 → 拼进 tools 字段   │───────────────>│ 返回 name/description/│
  │  （模型看到的仍是第 7 篇那    │<───────────────┤ inputSchema 清单       │
  │   种 JSON Schema）          │   ④ tools/call  │                      │
  │  代模型执行：转发调用+参数     │───────────────>│ 真正执行工具函数        │
  │  把结果回喂进 messages       │<───────────────┤ 返回 content/isError  │
  └────────────────────────────┘                └──────────────────────┘
        ①②只发生在连接建立时；③④在会话里反复发生（③的结果可缓存）
```

三个核心 method（命名空间前缀就是「能力域」）：`initialize`（握手）、`tools/list`（列工具）、`tools/call`（调工具）。逐条看真实报文——以下 JSON 全部来自本篇动手实践的**本机实测捕获**（不是示意），先用官方示例的 get_weather 场景。

**① initialize 请求**（client → server；出处：规范 lifecycle 章 2025-06-18 版示例，字段结构核验）：

```jsonc
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",     // client 想说的协议版本（规范：应报 client 支持的最新版）
    "capabilities": {},                   // client 能力声明：roots / sampling / elicitation…
    "clientInfo": {                       // client 自报家门（纯描述性，用于展示与排查）
      "name": "raw-learner",
      "version": "0.1"
    }
  }
}
```

**initialize 响应**（server → client，同样带 `id: 0`）：

```jsonc
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2025-06-18",   // server 确认/还价后的版本（见 §2.3）
    "capabilities": {                  // server 能力声明——本次实测的真实值
      "prompts":   {"listChanged": false},
      "resources": {"listChanged": false, "subscribe": false},
      "tools":     {"listChanged": false}
    },
    "serverInfo": {"name": "weather", "version": ""}
  }
}
```

REST 对位：这一步 ≈ 「先 GET /version + /capabilities 再开干」——但 MCP 把它做成了必经仪式，双方从此知道彼此的边界。`capabilities` 里的 `tools.listChanged` 表示「我支持工具清单变更通知」（`notifications/tools/list_changed`）——server 加了新工具时主动喊一声，client 不用轮询。

随后 client 必发一条**通知**（无 `id`、无响应）：`{"jsonrpc": "2.0", "method": "notifications/initialized"}`——单向告知「握手完成，开始干活」。

**③ tools/list**（client 请求 `{"method": "tools/list", "params": {}}`，server 响应）：

```jsonc
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "查询指定城市的当前天气。city 传中文城市名，如 '杭州'。",
        "inputSchema": {                  // 就是第 7 篇给模型看的 JSON Schema，同一个东西
          "type": "object",
          "properties": {"city": {"title": "City", "type": "string"}},
          "required": ["city"],
          "title": "get_weatherArguments"    // SDK 从 Python 类型注解自动生成
        }
      }
    ]
    // 分页游标 nextCursor（工具多时 server 分页返回）、字段 title/annotations 可选
  }
}
```

这份 `inputSchema` 与第 7 篇 `TOOL_SCHEMAS` 里的 `parameters` 是**同构的 JSON Schema**——所以第 14 篇把 M2 工具包装成 server 后，agent 侧 loop 拿到 `tools/list` 结果填进 `tools` 字段，模型侧行为完全不变。REST 对位：`tools/list` ≈ `GET /tools`（一份自描述的目录）。

**④ tools/call**（client → server，**代模型**执行）：

```jsonc
// 请求：name + arguments（模型产出的参数由 client 原样转发）
{"jsonrpc": "2.0", "id": 2, "method": "tools/call",
 "params": {"name": "get_weather", "arguments": {"city": "杭州"}}}

// 响应（实测）：
{"jsonrpc": "2.0", "id": 2,
 "result": {
   "content": [{"type": "text",
                "text": "{\n  \"city\": \"杭州\",\n  \"weather\": \"晴\",\n  \"temp_c\": 26\n}"}],
   "isError": false
 }}
```

三个对位要点：其一，`tools/call` ≈ `POST /tools/get_weather/invoke`——`params.name` 定位工具、`params.arguments` 是参数体；其二，**执行发生在 server 进程**，client（以及模型）只拿到文本结果——第 1 篇「模型不执行函数、代码才执行」的三方分工，在 MCP 里变成「client 代模型调、server 代 client 执行」；其三，`content` 是数组且每项带类型（`text` / `image` / `audio` / `resource`，出处：规范 `schema/2026-07-28/schema.ts` 的 `TextContent`/`ImageContent`/`AudioContent`/`EmbeddedResource` 接口），工具结果可以是多模态的；`isError` 是「工具执行失败」的标记（动手实践补充实验二实测了它的行为）。

### 2.3 能力协商与版本还价：实测看协议怎么自保

`initialize` 的两个协商规则（出处：规范 lifecycle 章 *Version Negotiation* / *Capability Negotiation*，2025-06-18 版）：

- **版本**：client 必须报一个它支持的版本（应报最新）；server 支持同版则原样回应，否则**报自己支持的其他版本（应报其最新）**；client 若也不支持 server 的还价，就断开。本质和你熟悉的 HTTP 内容协商（`Accept` 头）同构。
- **能力**：双方各自声明「支持哪些可选特性」（client 侧：`roots` / `sampling` / `elicitation`；server 侧：`tools` / `resources` / `prompts` / `logging` 等），**此后双方只准用协商过的能力**——这就像 HTTP 的 `OPTIONS` 预检 + 一次性约定，避免「用到一半发现对面不支持」。

本篇动手实践的补充实验一实测了还价：client 报一个不存在的未来版本 `2099-01-01`，server（官方 SDK 实现，其握手期最新为 `2025-11-25`）**还价到 `2025-11-25`**——与规范「应报 server 支持的最新版」逐字吻合。协议的向前兼容不靠猜，靠这条确定性规则。

## 3｜传输层：stdio 与 Streamable HTTP

读完本节你能理解：同一套 JSON-RPC 报文，怎么分别跑在「本地管道」和「HTTP POST」上；两类传输各自的适用场景；SSE 从主角降级为「请求内通道」的历史；以及为什么这个选择基本等价于「本地进程间调用 vs 远程服务」。

**一句话大白话**：传输（transport）决定「报文字节走哪条路」，与报文内容无关——就像你不会因为改用 gRPC 就重写业务 JSON。

### 3.1 stdio：本地子进程，管道上的换行分隔 JSON

```text
  client 进程（agent）          fork/exec               server 进程
  ┌──────────────┐   启动子进程    ┌──────────────┐
  │ 写入 stdout──┼────管道──────>  stdin 读一行     │
  │ 读取 stdin <─┼────管道──────  stdout 写一行    │
  └──────────────┘   JSON 行      （stderr = 日志）  └──────────────┘
```

规则（出处：规范 transports/stdio 章，2026-07-28 版）：client 把 server 作为**子进程**启动；server 从 stdin 读 JSON-RPC、往 stdout 写 JSON-RPC，**一行一条报文、行内不得有换行**；stderr 随便写（纯日志，client 可转发可忽略）；server 不得往 stdout 写任何非报文字节——第 7 篇你写工具时「print 调试会污染输出」的纪律，在这里升格为协议义务。

对位：stdio 就是**本地函数调用/Unix 管道思想**——无网络、无端口、无鉴权（信任本机），进程生命周期即会话生命周期。Claude Code 挂本地 server 走的就是这条（动手实践实操）。

### 3.2 Streamable HTTP：一个 POST 端点包打天下

远程场景报文内容不变，路换成 HTTP：server 暴露**一个 MCP 端点**，client 把每条 JSON-RPC 报文作为独立 POST 发过去；server 用**单个 JSON 对象**直接应答，或对该请求开一条**请求作用域的 SSE 流**（先推请求相关的通知、最后推正式响应）。出处：规范 transports/streamable-http 章，2026-07-28 版（2026-10-16 核验）。

**SSE 的历史地位，一句话讲清**：2024-11-05 初版规范里远程传输是「HTTP POST + SSE 长连接」双端点设计；2025-03-26 规范用 Streamable HTTP **取代**了它（出处：2025-03-26 Key Changes 原文 "Replaced the previous HTTP+SSE transport with a more flexible Streamable HTTP transport"，PR #206，2026-10-16 从 modelcontextprotocol.io 核验）；到 2026-07-28 规范进一步删掉了 GET 流端点与跨请求会话（`Mcp-Session-Id` 头），SSE 只剩「单个请求内的流式应答通道」这一个角色。今天你仍会见到老的 SSE 端点（Claude Code 的 `--transport sse` 还能挂），但**新做 server 一律选 Streamable HTTP**。

### 3.3 选型场景表

| 维度 | stdio | Streamable HTTP |
|---|---|---|
| server 形态 | 本地子进程，client 启动/停止 | 独立进程/独立主机，可多 client 同时连 |
| 生命周期 | 随 client 会话生死 | 与 client 解耦，独立部署/升级/扩容 |
| 鉴权 | 无网络面，凭据经环境变量传入（client 启动时给） | OAuth 2.1（§4）/ Bearer 头 / 网关 |
| 状态 | 连接内会话（握手一次） | 2026-07-28 起无状态化：每请求自报版本与能力 |
| 延迟/部署成本 | 极低，零网络 | 一个 HTTP 服务（网关/证书/域名） |
| 多客户端复用 | 差（谁启动谁独享） | 好（一个 URL 服务全公司） |
| 典型场景 | 本地文件系统、Git、个人工具链 | 企业后台、SaaS、需要集中治理的内部服务 |
| 本项目落点 | 第 14 篇（M2 工具包成 stdio server，Python 同进程生态演示） | 第 15 篇（Go 后台包成远程 server，官方 go-sdk） |

一句话选型：**工具跟人走、数据在本机 → stdio；工具跟系统走、要被多方共享 → Streamable HTTP**。这与「本地函数调用 vs 远程 RPC」的同构不是巧合——传输选型本来就继承自你在后端领域已有的经验。

## 4｜授权：stdio 放环境变量，HTTP 走 OAuth 2.1

读完本节你能理解：MCP 的授权是传输层职责，两类传输各有一套答案；远程场景下「谁给谁授权」的四个角色怎么对位到你熟的 OAuth 术语。

**一句话大白话**：stdio 的信任模型是「本机、子进程、你亲手启动的」——没网络面，授权退化为「启动时把凭据塞进环境变量」；Streamable HTTP 的 server 是公网/内网上的独立服务，得回答「谁在调我、凭什么调我」——MCP 规范给的答案是 OAuth 2.1。

角色对位（出处：规范 authorization 章，2025-06-18 版，2026-10-16 核验；原文明确四点：MCP 授权是**可选**能力；HTTP 传输**应当**遵循本章；stdio 传输**不应**走本章流程、应从环境取凭据）：

| OAuth 2.1 角色 | 在 MCP 里是谁 | 对位你熟的 |
|---|---|---|
| Resource Server（资源服务器） | MCP server（保护着的工具/数据） | 你的 Go 后台 API 服务 |
| Client（客户端） | MCP client（agent / Claude Code） | 调你 API 的前端或服务 |
| Authorization Server（授权服务器） | 签发 access token 的一方（可与 server 同部署或独立，如公司 SSO/IdP） | 你后台的 JWT 签发端 / 公司 SSO |
| Resource Owner（资源所有者） | 用户（人） | 后台用户 |

授权流和你熟的 OAuth 完全一致：client 拿着用户授权向授权服务器要 token → 携 token 调 MCP server → server 验 token 放行。规范明确基于 OAuth 2.1 草案 + 三个 RFC（8414 服务器元数据、7591 动态客户端注册、9728 受保护资源元数据）。Claude Code 的落地命令已实证存在：`claude mcp login <name>`（对 HTTP/SSE server 发起 OAuth 登录）、`claude mcp logout`（清除凭据）（出处：code.claude.com MCP 文档与 `claude mcp --help` 实测，2026-10-16）。

**什么时候需要哪套**（授权场景表）：

| 场景 | 用什么 | 一句话理由 |
|---|---|---|
| 本地 stdio server（个人 Git/文件工具） | 环境变量传凭据 | 无网络面；OAuth 是杀鸡用牛刀（规范明说 stdio 不该走它） |
| 内网远程 server（企业内部服务） | OAuth 2.1（对内部 IdP）/ 网关鉴权 / Bearer | 要「哪个 agent、代表谁」的身份，才能接到公司 RBAC |
| 公网 SaaS server（Notion/GitHub 官方等） | OAuth 2.1（用户亲手授权） | 跨组织信任：用户授权 agent 代理访问自己的账号 |
| 本教程 M2 助手连 Go 后台 | 用户 JWT（第 8 篇起的既有设计，P7·20 讲身份映射） | agent 是「代表用户动作」的 client，用户身份即授权来源 |

注意最后一行与本系列主线的衔接：**MCP 的 OAuth 解决「agent 客户端 ↔ MCP server」的授权；MCP server 收到调用后以什么身份调你的 Go 后台，是另一层问题**（第 15 篇讲桥接时展开，P7·20 讲治理收口）。

## 5｜2026 现状：规范时间线、官方 SDK 与参考生态

读完本节你能理解：规范从初版到 `2026-07-28` 的演进主线（尤其「无状态化」这次大改）、官方 SDK 覆盖、现成 server 去哪找——以及一个诚实提醒：**最新规范 ≠ 部署现实**。

### 5.1 规范时间线与 2026-07-28 的无状态化

版本号就是日期（date-based），截至 2026-10-16（GitHub tags API 核验）共五个正式版：

| 规范版本 | 一句话记忆点 |
|---|---|
| 2024-11-05 | 初版：stdio + HTTP+SSE；工具/资源/提示词三能力定型 |
| 2025-03-26 | OAuth 2.1 授权框架；Streamable HTTP 取代 HTTP+SSE |
| 2025-06-18 | 修订完善（本篇握手示例与授权章引用的版本）；生态主力版本 |
| 2025-11-25 | 输入校验错误归为工具执行错误（喂回模型可自愈，第 7 篇「错误回喂」的协议化）；SSE 流轮询 |
| **2026-07-28（现行最新）** | **无状态化改造**（见下）；工具结果缓存提示（`ttlMs`）；OpenTelemetry trace 传播约定 |

**2026-07-28 改了什么、为什么值得你知道**（出处：该版 Key Changes，2026-10-16 从规范仓库核验）：把 MCP 从「有会话的协议」改成「无状态的协议」——删除 `initialize`/`notifications/initialized` 握手与 `Mcp-Session-Id` 会话头，改为**每个请求自带信封**：`params._meta` 里携带 `io.modelcontextprotocol/protocolVersion`、`io.modelcontextprotocol/clientCapabilities` 等字段；server 必须实现新的 `server/discover` 方法自报身份与版本。动机与大型分布式系统演进同构：会话态让 server 难以水平扩容与负载均衡，无状态化后每个请求可打向任意实例——`tools/list` 的结果从此不随连接而变，中间层可以放心缓存。

**诚实提醒（本系列的准确性传统）**：新规范落地需要时间。部署现实是分层兼容的——官方 Python SDK（2.3.0 实测）同时内置两套语义：握手期协议（`2024-11-05` 到 `2025-11-25`，走 initialize）与无状态期协议（`2026-07-28`，走 `_meta` 信封 + `server/discover`），按对端版本自动切换；Claude Code 等主流 client 与大量存量 server 目前仍以握手期协议为主（其文档仍引用 `notifications/roots/list_changed` 等握手期方法）。**所以本篇动手实践教你的是「你会真实看到的握手期报文」，同时你必须知道最新规范已经走向无状态**——第 14 篇写 server 时会给两套的落点。协议仍在快速演化，写生产代码前先查当前版规范。

### 5.2 官方 SDK 与参考 servers（2026-10-16 核验）

**官方 SDK**（出处：[modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) README 列表 + 各仓库 Releases API）：

| SDK | 仓库 / 版本 | 本系列用到 |
|---|---|---|
| Python | [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)，PyPI `mcp` 2.3.0（2026-10-02；Python ≥ 3.10） | 第 14 篇：把 M2 工具包成 server |
| TypeScript | [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk)，v2.3.1（2026-10-05） | 参考 servers 多数由它实现 |
| **Go（官方）** | [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk)，v1.8.0（2026-09-14；与 Google 协作维护） | 第 15 篇：Go 后台包 MCP server |
| 另有 C#/Java/Kotlin/PHP/Ruby/Rust/Swift 共 10 个语言 SDK | 同见 servers README | 本系列不展开 |

**Python SDK 2.x 的重要更名（直接影响第 14 篇，先打个预防针）**：mcp 2.x 起 `FastMCP` 更名为 `MCPServer`（`from mcp.server.mcpserver import MCPServer`；老名字 `mcp.server.fastmcp` 在 2.x 会抛出指向迁移指南的 `ModuleNotFoundError`，出处：本机安装 mcp 2.3.0 后实测 + SDK 源码内迁移说明）。网上大量教程用的是 1.x 的 `FastMCP` 名字——读完本篇你就有能力识别「这教程说的是旧版协议栈」。

**参考 servers**（出处：[modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)，91,302 stars，2026-10-16 GitHub API 核验；定位是**教学参考实现**而非生产件，README 原文自述）。当前在维护的 7 个：**Everything**（测试用瑞士军刀，演示全部能力）、**Fetch**（抓网页转 Markdown）、**Filesystem**（限定目录的文件操作）、**Git**（读/搜 Git 仓库）、**Memory**（知识图谱记忆）、**Sequential Thinking**（思维链式问题分解）、**Time**（时间/时区换算）。早年明星 server（GitHub、Slack、PostgreSQL、Puppeteer 等）已归档至 servers-archived，多数由官方/社区另出独立实现。**找 server 的新入口是 MCP Registry**（[registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io/)——已发布 server 的官方目录）。

**调试工具**：MCP Inspector（[modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector)，11,084 stars）——官方可视化测试工具，`npx @modelcontextprotocol/inspector` 起一个本地 UI，手动列工具/调工具/看报文，是「本机装不了 Claude Code 时的替代观察路径」（动手实践给出）。

## 动手实践：亲眼看一次 MCP 的完整调用 trace

本实践回答一个问题：**「第 2 节那些报文是真的吗？」**——你将不用任何 MCP client 库，手写 JSON-RPC 驱动一个最小 server，把 §2.2 的每条报文亲眼打出来；再用 Claude Code 挂载同一个 server，看外部客户端复用同一份工具。全部代码与输出均为 2026-10-16 本机实测记录（Python 3.12，mcp 2.3.0）；换版本报文字段可能微调，以你跑出来的为准。

**准备环境**（约 2 分钟；沿用第 5 篇的 venv 习惯）：

```bash
mkdir -p mcp13 && cd mcp13
python3.12 -m venv .venv && source .venv/bin/activate
pip install "mcp"     # 实测装到 2.3.0；国内镜像：pip install -i https://pypi.tuna.tsinghua.edu.cn/simple mcp
```

> 注意 mcp 2.x 的更名：如果你跟着老教程装了 `mcp<2`，本节的 `MCPServer` 写法要换成 `FastMCP`（`from mcp.server.fastmcp import FastMCP`），其余报文演示完全一致——因为报文由协议决定，与 SDK 哪个版本包装无关。

**文件 1：`weather_server.py`** —— 12 行的最小 server（一个工具；`mcp.run()` 默认走 stdio 传输）：

```python
# weather_server.py — 最小 MCP server（官方 python-sdk 2.x；mcp<2 请用 FastMCP）
from mcp.server.mcpserver import MCPServer

mcp = MCPServer("weather")

@mcp.tool()
def get_weather(city: str) -> dict:
    """查询指定城市的当前天气。city 传中文城市名，如 '杭州'。"""
    return {"city": city, "weather": "晴", "temp_c": 26}

if __name__ == "__main__":
    mcp.run()  # stdio 传输：stdin/stdout 上收发 JSON-RPC
```

注意它和第 7 篇 `TOOL_REGISTRY` 的关系：`@mcp.tool()` 一行干了两件事——把函数**注册**进 server、把类型注解+docstring **翻译**成 JSON Schema（tools/list 时返回的 `inputSchema` 就是它生成的）。第 14 篇把 M2 四个工具搬过来，做的就是同样的事。

**文件 2：`raw_client.py`** —— 手写 JSON-RPC 的观察脚本（把 server 当子进程，逐阶段收发报文；约 70 行）：

```python
# raw_client.py — 用「裸 JSON-RPC」驱动 weather_server.py，把 wire 层每条报文打出来。
# 不用任何 MCP client 库：initialize / tools/list / tools/call 全部手写。
# 关键协议点：请求带 id、通知不带 id —— initialize 若忘了带 id，
# server 会把它当通知「收到但不回」，client 就会卡死在 readline()（本脚本初版真实踩过）。
import json
import subprocess
import sys
from pathlib import Path

proc = subprocess.Popen(
    [sys.executable, str(Path(__file__).with_name("weather_server.py"))],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True, bufsize=1,
)

next_id = 0

def request(method: str, params: dict, label: str) -> None:
    """JSON-RPC 请求：带 id，必有响应。"""
    global next_id
    msg = {"jsonrpc": "2.0", "id": next_id, "method": method, "params": params}
    next_id += 1
    line = json.dumps(msg, ensure_ascii=False)
    print(f"--> {label}: {line}")
    proc.stdin.write(line + "\n")
    proc.stdin.flush()

def notify(method: str, label: str) -> None:
    """JSON-RPC 通知：不带 id，不期待响应（发了就走）。"""
    line = json.dumps({"jsonrpc": "2.0", "method": method}, ensure_ascii=False)
    print(f"--> {label}: {line}")
    proc.stdin.write(line + "\n")
    proc.stdin.flush()

def recv(label: str) -> dict:
    line = proc.stdout.readline()
    print(f"<-- {label}: {line.rstrip()}")
    return json.loads(line)

# ---- 阶段 1：initialize 握手（客户端报协议版本 + 能力 + 自报家门）----
request("initialize", {
    "protocolVersion": "2025-06-18",
    "capabilities": {},
    "clientInfo": {"name": "raw-learner", "version": "0.1"},
}, "initialize 请求")
init_result = recv("initialize 响应")
print(f"    [server 自报：{init_result['result']['serverInfo']}, "
      f"协商版本 {init_result['result']['protocolVersion']}]")

# ---- 阶段 2：initialized 通知（无 id、无响应——单向告知握手完成）----
notify("notifications/initialized", "initialized 通知")

# ---- 阶段 3：tools/list（问 server 有哪些工具）----
request("tools/list", {}, "tools/list 请求")
tools_result = recv("tools/list 响应")
for t in tools_result["result"]["tools"]:
    print(f"    [工具 {t['name']}: {t['description']}]")
    print(f"    [inputSchema: {json.dumps(t['inputSchema'], ensure_ascii=False)}]")

# ---- 阶段 4：tools/call（代模型调一次工具）----
request("tools/call", {"name": "get_weather", "arguments": {"city": "杭州"}},
        "tools/call 请求")
call_result = recv("tools/call 响应")
print(f"    [结果 content: {json.dumps(call_result['result']['content'], ensure_ascii=False)}]")

proc.stdin.close()
proc.wait(timeout=5)
print("=== trace 完毕 ===")
```

**运行与预期输出**（`python -u raw_client.py`；以下是 2026-10-16 实测捕获，`#` 注释为后加分析）：

```text
--> initialize 请求: {"jsonrpc": "2.0", "id": 0, "method": "initialize", "params": {"protocolVersion": "2025-06-18", "capabilities": {}, "clientInfo": {"name": "raw-learner", "version": "0.1"}}}
<-- initialize 响应: {"jsonrpc":"2.0","id":0,"result":{"capabilities":{"prompts":{"listChanged":false},"resources":{"listChanged":false,"subscribe":false},"tools":{"listChanged":false}},"protocolVersion":"2025-06-18","serverInfo":{"name":"weather","version":""}}}
    # id=0 精确配对；server 逐字确认版本 2025-06-18；capabilities 里 tools 在场
--> initialized 通知: {"jsonrpc": "2.0", "method": "notifications/initialized"}
    # 无 id、无响应——§2.1 的通知语义
--> tools/list 请求: {"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}
<-- tools/list 响应: {"jsonrpc":"2.0","id":1,"result":{"tools":[{"description":"查询指定城市的当前天气。city 传中文城市名，如 '杭州'。","inputSchema":{"properties":{"city":{"title":"City","type":"string"}},"required":["city"],"type":"object","title":"get_weatherArguments"},"name":"get_weather"}]}}
    # docstring 变成 description、类型注解变成 inputSchema——你写的是 Python，模型看到的是 Schema
--> tools/call 请求: {"jsonrpc": "2.0", "id": 2, "method": "tools/call", "params": {"name": "get_weather", "arguments": {"city": "杭州"}}}
<-- tools/call 响应: {"jsonrpc":"2.0","id":2,"result":{"content":[{"text":"{\n  \"city\": \"杭州\",\n  \"weather\": \"晴\",\n  \"temp_c\": 26\n}","type":"text"}],"isError":false}}
=== trace 完毕 ===
    # dict 返回值被序列化成 text content；isError:false = 执行成功
```

对照着读：这四条报文就是 §2.2 时序图的 ①-④。**agent 侧的完整翻译链**（第 14 篇代码化）：client 把 `tools/list` 的结果填进对话补全请求的 `tools` 字段 → 模型吐出 tool call → client 查「这名字属于哪个 MCP server」→ 转发成 `tools/call` → 把 `content` 文本作为 tool 结果回喂进 messages。模型从头到尾没见过 JSON-RPC——**MCP 是 client 与 server 之间的协议，不是模型与任何人的协议**。

**文件 3：`extra_experiment.py`** —— 两个补充实验（版本还价 + 错误喂回；约 45 行，同目录运行 `python -u extra_experiment.py`）。它复用 raw_client 的收发函数，只列差异部分：

```python
# extra_experiment.py — 两个补充实验（同目录三件套之三）：
# ① 客户端报一个 server 不认识的未来版本号，看 server 怎么"还价"（§2.3）
# ② tools/call 传不符合 schema 的参数，看协议层怎么把错误喂回来（isError 路径）
# （request / notify / recv 与 raw_client.py 完全一致，此处省略——照抄 raw_client.py 即可）

# ① 还价实验：报一个 server 不认识的未来版本
r = request("initialize", {
    "protocolVersion": "2099-01-01",
    "capabilities": {},
    "clientInfo": {"name": "raw-learner", "version": "0.1"},
}, "initialize(未来版本) 请求")
print(f"    [server 还价到：{r['result']['protocolVersion']}]")
notify("notifications/initialized", "initialized 通知")

# ② 错误参数实验：schema 要求 city 是 string，故意传 list
r = request("tools/call", {"name": "get_weather", "arguments": {"city": ["杭州"]}},
            "tools/call(参数类型错误) 请求")
print(f"    [isError: {r['result'].get('isError')}]")
print(f"    [content: {json.dumps(r['result'].get('content'), ensure_ascii=False)}]")
```

**实测输出**（2026-10-16）：

```text
--> initialize(未来版本) 请求: {"jsonrpc": "2.0", "id": 0, "method": "initialize", "params": {"protocolVersion": "2099-01-01", ...}}
    [server 还价到：2025-11-25]
    # 规范说"应报 server 支持的最新版"——官方 SDK 的握手期最新正是 2025-11-25（§5.1）
--> tools/call(参数类型错误) 请求: {"jsonrpc": "2.0", "id": 1, "method": "tools/call", "params": {"name": "get_weather", "arguments": {"city": ["杭州"]}}}
    [isError: True]
    [content: [{"text": "Error executing tool get_weather: 1 validation error for get_weatherArguments\ncity\n  Input should be a valid string ...", "type": "text"}]]
```

第二个实验值得多看一眼：**参数不合法不是 JSON-RPC 层的 `error`，而是正常 `result` 里 `isError: true`**——2025-11-25 规范明确把输入校验错误归为「工具执行错误」（Tool Execution Error）而非协议错误，理由就是让模型能读到错误文本、修正参数重试（出处：2025-11-25 Key Changes 第 4 条，SEP-1303）。这正是第 7 篇「错误回喂自愈」的协议化：私有协议下你自己决定怎么回喂，MCP 把这条最佳实践写进了协议。还价实验则演示了向前兼容：老 client 也能和新 server 对上话，因为版本不匹配时有确定性的还价规则。

### 给 Claude Code 挂载同一个 server（观察外部客户端接入）

上面是「自己当 client」。现在换 Claude Code 当 client——同一个 `weather_server.py`，一行注册命令（**命令语法经 `claude mcp --help` 与 code.claude.com MCP 文档核验，2026-10-16；挂载后的会话体验因版本而异**）：

```bash
# 在 mcp13 目录（确保 weather_server.py 可用；「--」之后的参数原样传给 server）
claude mcp add --transport stdio weather -- <venv绝对路径>/bin/python <mcp13绝对路径>/weather_server.py

claude mcp list        # 应看到 weather 一行；首次连过之后显示连接状态
claude mcp get weather # 看它的详细配置与健康状态
```

然后在 Claude Code 会话里问一句「杭州现在天气怎么样」，工具就会以 **`mcp__weather__get_weather`** 的名字进入它的工具清单（MCP 工具在 Claude Code 里的命名规则是 `mcp__<server名>__<工具名>`，出处：code.claude.com MCP 文档，2026-10-16 核验）；输入 `/mcp` 打开面板能看到该 server 的连接状态与工具数。注册信息落在配置文件里：交互式 `claude mcp add` 默认写 local scope（仅本机本目录），`--scope project` 写项目根的 **`.mcp.json`**（可提交 git 团队共享），`--scope user` 写本机全局——`.mcp.json` 的形状：

```jsonc
{
  "mcpServers": {
    "weather": {
      "command": "<venv绝对路径>/bin/python",       // stdio server = 一条启动命令
      "args": ["<mcp13绝对路径>/weather_server.py"]
    }
  }
}
```

对比着体会这次「接入」有多薄：你**没有为 Claude Code 写任何代码**——它作为 MCP client 发起同样的 initialize/tools/list/tools/call，你的 server 无法区分（也不需要区分）对面是自研 raw_client 还是 Claude Code。第 1.2 节说的「N+M 的加法」，这里就是你亲手按下的那次加一。

**本机装不了 Claude Code 时的替代观察路径**：其一，跑上面文件 2/3——裸 JSON-RPC 观察的正是 Claude Code 在底下做的事；其二，用官方可视化工具 MCP Inspector：`npx @modelcontextprotocol/inspector <venv路径>/bin/python <mcp13路径>/weather_server.py`，浏览器里手动 Connect → Tools → Run，同样能看到列工具与调用的完整往返（Inspector 用法见其 README，<以官方 README 为准>）。此外可挂规范仓库的参考 server 体验真实生态：`npx -y @modelcontextprotocol/server-everything`（Everything 测试 server）或 `uvx mcp-server-time`（Time server，出处：servers 仓库 README，2026-10-16 核验），注册命令同上。

**真实踩坑记录（协议在细节处 bite）**：本实践脚本初版在 `initialize` 报文里漏了 `id` 字段，现象是 client 卡死在 `readline()`——因为按 JSON-RPC 语义，无 `id` 的报文是**通知**，server 「收到但不回」。修复 = 请求带 `id`、通知不带。这不是 SDK 的 bug，是 §2.1 信封规则的严格执行；把它记进你的调试清单：**MCP 会话「卡住」时先看第一条报文带没带 id**。

### 自检清单

- [ ] raw_client 输出里，`initialize` 响应的 `id` 与请求的 `id` 是否一致？（信封配对）
- [ ] `tools/list` 返回的 `inputSchema` 与你在 `weather_server.py` 里写的类型注解逐项对应吗？（声明式生成）
- [ ] 把 `extra_experiment.py` 里错误参数换成 `{"wrong_key": "杭州"}` 再跑，观察 pydantic 校验错误文本怎么变（仍是 `isError: true` 路径吗？）
- [ ] （有 Claude Code 环境）`/mcp` 面板里 weather 显示几个工具？问一句天气问题，观察它调用的是不是 `mcp__weather__get_weather`？
- [ ] （选做）把 `get_weather` 加个参数再重启 Claude Code 会话，看工具清单是否更新——如果 server 声明了 `tools.listChanged`，还可以不重启（§2.2 的通知机制）

## 本篇结论

- MCP 把「工具」从 agent 进程的私有协议变成跨语言跨进程的标准：N 个 agent × M 个系统的 N×M 套胶水，压缩为 N+M 份「会说 MCP」的实现——与 OpenAPI 之于 REST 同构。
- 协议本体 = JSON-RPC 2.0 信封（请求带 `id`、通知不带、`result`/`error` 互斥）+ 三个核心 method：`initialize` 握手协商版本与能力、`tools/list` 拉取与第 7 篇同构的 JSON Schema、`tools/call` 代模型执行并带回 `content`/`isError`。
- 传输按「本地 vs 远程」二选一（stdio / Streamable HTTP，后者 2025-03-26 起取代 HTTP+SSE）；授权按传输二分：stdio 走环境变量，HTTP 走 OAuth 2.1（MCP server = resource server）。规范最新 tag `2026-07-28` 已走向无状态（每请求 `_meta` 信封 + `server/discover`），但部署现实仍以握手期协议为主——学协议看最新版，写代码先查当下生态。

## 参考资料

1. [MCP 规范仓库 modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol) — 全篇协议细节的最终出处。现行最新 tag `2026-07-28`（2026-10-16 经 GitHub tags API 核验，无更新 tag）；本篇引用的 lifecycle / transports / authorization / tools 各章与 Key Changes 均出自仓库 `docs/specification/` 对应版本目录；§2.2 报文字段经 `schema/2026-07-28/schema.ts` 与 2025-06-18 版 lifecycle 示例逐字核验。
2. [MCP 官方规范站 modelcontextprotocol.io](https://modelcontextprotocol.io/) — 规范的渲染版；2025-03-26 Key Changes（Streamable HTTP 取代 HTTP+SSE、OAuth 2.1 框架）于 2026-10-16 从该站实测抓取核验。
3. [JSON-RPC 2.0 规范（jsonrpc.org）](https://www.jsonrpc.org/specification) — §2.1 信封字段（`jsonrpc`/`method`/`params`/`id`/`result`/`error`、通知无 id、`-32601` Method not found）逐字核验出处（2026-10-16 实测抓取）。
4. [MCP 官方 Python SDK（modelcontextprotocol/python-sdk）](https://github.com/modelcontextprotocol/python-sdk) — PyPI `mcp` 2.3.0（2026-10-02 发布，2026-10-16 PyPI API 核验）；`mcp.server.mcpserver.MCPServer` 与 FastMCP 更名说明经本机安装实测 + 源码核验；握手期/无状态期协议并存经 `src/mcp-types/mcp_types/version.py` 源码核验。
5. [MCP 官方 Go SDK（modelcontextprotocol/go-sdk）](https://github.com/modelcontextprotocol/go-sdk) — v1.8.0（2026-09-14，2026-10-16 GitHub Releases API 核验）；与 Google 协作维护（仓库 README 自述）。第 15 篇的主角。
6. [MCP 参考 servers 仓库（modelcontextprotocol/servers）](https://github.com/modelcontextprotocol/servers) — 7 个参考 server 清单、10 个官方 SDK 列表、MCP Registry 入口（2026-10-16 README + GitHub API 核验：91,302 stars）。
7. [Claude Code MCP 文档（code.claude.com/docs/en/mcp）](https://code.claude.com/docs/en/mcp) — `claude mcp add` 语法、`--transport`/`--scope`、`.mcp.json` 格式、`/mcp` 面板、`mcp__server__tool` 命名规则（2026-10-16 实测抓取 + `claude mcp --help` 对照核验）。
8. [MCP Inspector（modelcontextprotocol/inspector）](https://github.com/modelcontextprotocol/inspector) — 官方可视化调试工具（11,084 stars，2026-10-16 核验）；动手实践的替代观察路径。
9. 系列前序：[第 1 篇](../part1-打开黑盒/01-全景与学习路线地图.md)（路线图中 MCP 的位置）、[第 6 篇](../part2-亲手写出第一个Agent/06-调用LLM的三种姿势.md)（tools 字段的 wire 层位置）、[第 7 篇](../part2-亲手写出第一个Agent/07-手写最小agent.md)（TOOL_SCHEMAS + REGISTRY 私有协议——本篇的被标准化对象）、[第 8 篇](../part3-接上真实系统/08-后台API变成工具.md)（M2 工具四件套——第 14 篇的包装对象）、[第 12 篇](../part4-记忆与规划/12-规划能力设计.md)（Part 4 收口钩子——本篇开头的被接住处）。

> 核验说明：参考资料 1-8 的 URL、版本号、命令语法均于 2026-10-16 在线核验（GitHub API / PyPI API / 规范站与文档站实测抓取 / 本机 CLI）；动手实践三件套在本机（Python 3.12.15 + mcp 2.3.0）完整跑通，文中所有报文输出为实测记录；Claude Code 挂载小节的命令语法经 `claude mcp --help` 与官方文档双源核验，挂载后的会话内体验 <未实测，以官方文档为准>，文中已标注；MCP Inspector 的 UI 操作 <以官方 README 为准>。规范 tag `2026-07-28` 为核验时最新；若你读到时已有新版本，以新规范为准并留意 §5.1 的兼容性说明。

