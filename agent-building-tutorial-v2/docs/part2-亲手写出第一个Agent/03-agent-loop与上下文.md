# 第 3 篇：Agent 的心脏 — agent loop（aka harness）与上下文管理

> **知识地图【A3/A4】· Part 2** ⭐ 工程地基
> 本篇是全系列的工程地基。第 1 篇给了 tool call 时序图、第 2 篇给了 PMTA 坐标系（[01-全景与学习路线地图](../part1-打开黑盒/01-全景与学习路线地图.md)、[02-经典理论映射](../part1-打开黑盒/02-经典理论映射.md)），本篇把它们落成代码形状：PMTA §3.4 对照表里的 **M**（Memory→第 3 节上下文管理）与 **A**（Action→第 1 节的工具执行与回填）两行从本篇起有代码对应物；P（Planning）与 T（Tool use）分别在 P4·12、P2·6 落地。agent loop 与 harness 在第 1 篇词表里只各带过一句，本篇展开讲透。

## 本篇要回答的问题

1. agent loop（智能体循环）到底长什么样？五阶段每一步谁在干活、数据怎么流动？
2. harness（智能体运行时）凭什么被叫"agent 的地基"——除了模型和工具，成熟系统还在循环外面加了哪些基础设施？
3. agent 为什么会"忘事"？上下文窗口（context window）的 token 预算怎么算，满了怎么办？
4. 循环什么时候停？没有终止条件（termination condition）会发生什么事故？
5. 不接 LLM、用桩函数把 agent loop 骨架跑通——亲眼看五阶段转起来、看死循环被熔断。

## 1｜agent loop：把"循环"拆成五阶段

读完本节你能理解：循环的每个阶段各自做什么、数据在哪里、终止判定发生在哪一步——第 7 篇的手写最小 agent 就是按这节逐行补全的。

### 1.1 从一句伏笔说起：回填 messages = agent 的内部状态

第 2 篇 §2.2 留过一句话：「把工具结果与模型回复回填进 messages 再问一次，你就迈进了 model-based 档：messages 列表就是 agent 的内部状态」。本篇回收这句话，把它落成 loop 的起点：

- **每轮把完整 messages 发给模型**——模型"看到"全部历史，这是它不"失忆"的唯一手段；
- **每轮把新东西追加回 messages**——模型的调用意图（assistant 消息）、工具结果（tool 消息）依次回填；
- **循环本身没有任何智能**——它只是机械地"发出去、收回来、追加、再发出去"。

"智能"全部发生在阶段②那一次模型调用里。所以第 1 篇才说 agent 是编排层：**loop 是骨架，模型调用是心脏的每一次跳动**。后端类比：这像一个 `for` 循环版的中间件链——每轮"请求"进模型、"响应"回来再进工具，直到某个响应说"到站了"。

### 1.2 五阶段拆解：数据流视角

五阶段的消息流（消息结构沿用第 1 篇 §2.3 的 OpenAI 兼容四角色 system/user/assistant/tool，本篇不再重复）：

```
 用户输入
    │
    ▼
① 获取输入：user 消息进 messages（agent 的内部状态，见 1.1）
    │
    └────────────── 回到循环顶 ◄────────────────────┐
                                                   │
    ② 调 LLM：把【全部】messages + 工具清单发给模型服务 │
    │                                              │
    ▼                                              │
③ 解析 finish_reason（终止判定就藏在这一步）           │
    │                                              │
    ├── 要工具（finish_reason = "tool_calls"）        │
    │       │                                      │
    │       ▼                                      │
    │   ④ 执行工具：真发 HTTP / 真查库 / 真写文件        │
    │       │                                      │
    │       ▼                                      │
    │   ⑤ 追加回填：模型意图 + 工具结果依次进 messages ──┘
    │
    └── 不调了（finish_reason = "stop"）→ 输出最终答复，循环结束
```

每个阶段的分工，用后端视角逐个对号：

| 阶段 | 谁在干活 | 做什么 | 后端类比 |
|---|---|---|---|
| ① 获取输入 | harness | 用户输入包装成 user 消息，放进 messages | Controller 收请求体 |
| ② 调 LLM | harness → 模型服务 | 发送全部 messages + 工具清单，等响应 | 调一次下游 RPC |
| ③ 解析 | harness | 读 finish_reason：要工具还是收尾 | 解析响应码、分支路由 |
| ④ 执行工具 | harness | 真正发 HTTP/查库/写文件——唯一改变世界的步骤 | Service 真正调下游/写库 |
| ⑤ 追加回填 | harness | assistant 意图、tool 结果追加进 messages | 回填会话上下文 |

三个要点值得钉死：

1. **五个阶段里四个（①③⑤加上循环控制）是纯确定性代码**，只有 ② 涉及模型——模型的不确定性被关在一个函数调用里，这正是第 1 篇"agent 不是玄学"立场的代码体现。
2. **messages 是唯一跨轮次存活的东西**（循环变量）。所以第 3 节的上下文管理，管的其实就是它的体积。
3. **终止判定在 ③，但它依赖模型"自觉"**——工程上还需要外挂的终止条件，第 4 节展开。

### 1.3 伪代码：循环的骨架（第 7 篇照此填真肉）

```
messages = [user("把张三的角色换成 admin")]          # ① 获取输入
while True:
    resp = LLM(messages, tools)                     # ② 调 LLM
    if resp.finish_reason == "tool_calls":          # ③ 解析
        messages.append(resp.assistant_message)     # ⑤ 回填模型意图
        for call in resp.tool_calls:
            result = execute(call.name, call.args)  # ④ 执行工具
            messages.append(tool_result(call.id, result))  # ⑤ 回填工具结果
    else:                                           # "stop"：模型宣告完成
        messages.append(resp.assistant_message)
        return resp.content                         # 终止条件之一（第 4 节）
```

循环里没有一行"聪明"逻辑——聪明全在 `LLM()` 和 `execute()` 里面。这份骨架 15 行左右：本篇动手实践先用桩函数跑通它（看清结构），第 7 篇把 `LLM()` 换成真模型、`execute()` 换成真工具，就是完整的最小 agent。

### 1.4 实锤：成熟系统文档里的"同一个循环"

五阶段不是本系列自创的拆法。两家成熟 agent 的官方文档对"循环"的描述拆法不同、形状同构（均为 2026-10-15 实测）：

**Claude Code**（官方文档称 **agentic loop**，按"工作性质"拆三相位）："When you give Claude a task, it works through three phases: **gather context, take action, and verify results**. These phases blend together."（三相位会互相融合）。并明确 "You're part of this loop too"——用户随时可以打断改向，这就是第 4 节第四类终止条件的官方出处。

**OpenClaw**（官方文档直接称 **agent loop**，按"运行环节"拆六步）："The agent loop is the serialized, per-session run that turns a message into actions and a reply: **intake, context assembly, model inference, tool execution, streaming, persistence**."（每个会话串行执行的一轮： intake → 上下文组装 → 模型推理 → 工具执行 → 流式输出 → 持久化）。

三套拆法是同一个循环的三个镜头：

| 本篇（数据流五阶段） | OpenClaw（运行环节六步） | Claude Code（工作相位三段） |
|---|---|---|
| ① 获取输入 | intake | gather context |
| ② 调 LLM | context assembly + model inference | （跨相位） |
| ③ 解析 | run sequence 里的终止判定 | take action 的决策点 |
| ④ 执行工具 | tool execution | take action |
| ⑤ 追加回填 | streaming + persistence | verify results 的依据（结果喂回供下一步判断） |

读法：Claude Code 按"这步工作属于哪种性质"归类，OpenClaw 按"数据经过哪个流水线工位"排序，本篇按"messages 什么时候变"切分——三镜头映射到的是同一个 while 循环，看懂任何一个，另外两个自动看懂。

## 2｜harness：模型和工具之外，循环外面还有什么

读完本节你能理解：为什么"会写 while 循环"离"能用的 agent"还差一整层——harness 的六个组件各自拦哪起事故、成熟系统分别怎么做。

### 2.1 新名词 harness：循环的工程化载体

**agent loop** 与 **harness** 经常连用（甚至被当同义词），准确关系是：

- **agent loop** 是那个 while 循环本身——§1 的五阶段；
- **harness** 是让这个循环在生产环境真正能跑的**全部外围基础设施**——循环只是 harness 的主线程，外围还包括工具注册、上下文管理、权限、持久化、可观测（本节展开）。

大白话先行：**loop 是发动机，harness 是整辆车**。

定义与出处：Claude Code 官方文档对 agentic harness 的原文——"The agentic loop is powered by two components: models that reason and tools that act. **Claude Code is the layer around the model that provides the tools and manages the context the model sees. This surrounding layer is what the term agentic harness refers to.**"（环绕模型、提供工具并管理模型所见上下文的那一层，就叫 agentic harness；2026-10-15 实测）。行业对 harness 的边界尚无唯一权威定义，本系列采用与官方表述一致的约定（第 1 篇词表的一句话，到此完成展开）。

后端类比一步到位：**harness 之于 agent，就像 Gin 之于一个 HTTP 服务**。Gin 不提供你的业务逻辑，但把路由注册、中间件链、参数绑定、panic 恢复、日志这些"每个服务都要"的事做掉；harness 不提供"智能"，但把循环、工具、上下文、权限、日志这些"每个 agent 都要"的事做掉。

### 2.2 通用 harness 分层图

把 Claude Code、OpenClaw 的公开文档对照归纳，通用 harness 可以画成三层（第 16 篇拆解成熟系统时逐层回访）：

```
┌──────────────────────────────────────────────────────────┐
│ 应用层（产品形态）：CLI / 编辑器 / Web / 消息渠道            │
│   Claude Code 的终端界面；OpenClaw 的 Control UI 与各渠道   │
├──────────────────────────────────────────────────────────┤
│ harness 层（本篇主角）                                    │
│                                                          │
│   ┌─ agent loop（§1 五阶段）：①→②→③→④→⑤→回到② ─┐        │
│   └────────────────────────────────────────────────┘      │
│   循环每一步都依赖这六件套：                                │
│     1 调度         谁先跑、并发怎么串行                     │
│     2 工具注册     工具清单与 schema 的管理、注入           │
│     3 事件流       过程与结果的对外广播                    │
│     4 权限         哪些动作放行、哪些要问人                │
│     5 持久化       会话与上下文落盘、可恢复                │
│     6 可观测       每步留痕、事后可查                     │
│   （第 7 件：上下文管理，管住 messages 体积 → §3 专节）      │
├──────────────────────────────────────────────────────────┤
│ 模型适配层：抹平供应商差异                                 │
│   （OpenAI 兼容 / Anthropic 原生 / 本地 Ollama 等）        │
└──────────────────────────────────────────────────────────┘
```

读图自下而上：模型适配层解决"怎么调到模型"，harness 层解决"怎么把循环跑稳"，应用层解决"人怎么用"。你第 7 篇手写的最小 agent 只有中间层的一小块（loop + 极简工具注册）——这完全正常，harness 是随规模长出来的，不是一上来就全建。

### 2.3 六件套逐个过：每个组件都在拦一起事故

**1｜调度（scheduling / queueing）——拦"并发互踩"**
解决什么：多个请求同时进来，同一会话的循环必须排队，否则两轮循环互相踩对方的 messages。事故现场：用户连发两句话，两轮循环并发跑，工具结果回填错乱、会话状态损坏。后端类比：按会话键分片的单 worker 任务队列——同会话串行、跨会话并行。实锤：OpenClaw 文档明确 "Runs are serialized per session key (session lane) and optionally through a global lane, preventing tool/session races"（2026-10-15 实测，下同）。

**2｜工具注册（tool registry）——拦"模型不认识你的能力"**
解决什么：模型只认识"随请求发过去的工具清单"（第 1 篇 §2.3），注册表负责把工具名、JSON Schema、描述统一管理并注入阶段②。事故现场：工具描述写得含糊，模型该调不调、乱填参数——工具描述本质是 prompt 的一部分，注册表同时是"给模型的文档"。后端类比：路由表——Gin 注册路由时，路径和参数签名一并声明。实锤：Claude Code 内置工具按五大类组织（文件操作/搜索/执行/Web/代码智能），另有 subagent、向用户提问等编排类工具。工具怎么设计才"可被模型理解"，第 8-10 篇由点及面展开；升级为跨语言标准即 MCP（P5）。

**3｜事件流（event stream）——拦"黑屏等待"**
解决什么：一轮循环可能跑几分钟，UI 不能干等最终答复——过程（正在调哪个工具、工具返回什么、模型正在说什么）要以事件形式对外广播。事故现场：没有事件流，用户面对黑屏三分钟，要么干等要么以为程序死了。后端类比：SSE/WebSocket 推送——请求-响应之外的单向进度广播。实锤：OpenClaw 的 Gateway "Exposes a typed WS API (requests, responses, server-push events)"，事件类型包括 agent、chat、presence、health 等；其 agent loop 文档还定义了生命周期事件 `phase: "start" | "finishing" | "end" | "error"`。你在 Claude Code 里看到的流式输出与"正在读取文件"提示，都是事件流在驱动。

**4｜权限（permission gate）——拦"模型好心办坏事"**
解决什么：阶段④"真动手"之前，有些动作必须先问人或直接拦下（写库、删数据、花钱、执行命令）。事故现场：模型"好心"帮你执行了 `rm -rf`、把生产表清了——权限闸门是 agent 与事故之间最后一道人工关卡。后端类比：网关的鉴权/限流中间件——先过闸再放行。实锤：Claude Code 的 permission modes 四档——Auto（后台分类器自动放行多数操作、拦下危险操作）/ Manual（文件编辑与命令执行前必问）/ Accept edits（文件编辑与部分文件系统命令不问，其余还问）/ Plan（只探索与出计划、不改源文件）；改文件前还有 checkpoints 先快照、可回滚。OpenClaw 同样有 exec approvals（命令执行前审批）与沙箱配置。人机协同的完整展开（何时该停下来问、三种介入方式）在 P7·19；这里记住一句：**权限闸门是 harness 的标准部件，不是可选装饰**。

**5｜持久化（persistence）——拦"重启即失忆"**
解决什么：会话断了要能接上、历史要能查——messages 不能只活在内存里。事故现场：进程重启，半小时的排查上下文全丢，用户从头再来。后端类比：会话落库 / 事件溯源（event sourcing）——把每条消息当事件追加存储。实锤：Claude Code "Each message, tool use, and result is written to a plaintext JSONL file under ~/.claude/projects/, which enables rewinding, resuming, and forking sessions"（JSONL 落盘支撑回退、续跑、分叉）；OpenClaw 的 compaction 文档同样强调 "The full conversation history stays on disk"。第 16 篇的"checkpoint 可恢复"实践就落在这里。

**6｜可观测（observability）——拦"出事只能靠猜"**
解决什么：模型"想"了什么、调了什么、结果是什么、花了多少 token——每一步都要留痕。事故现场：线上 agent 答非所问，没有 trace，你连它当时"看到"了什么都不知道。后端类比：请求日志 + 分布式追踪——每个阶段打点、按 run 串起来。实锤：OpenClaw 内建 OpenTelemetry export（导出 span 与诊断事件）；本系列 P7·18 的做法是自建 JSONL trace——每次 LLM/工具调用、耗时、成本各记一行。第 4 节排查死循环，靠的也是它。

### 2.4 一个反直觉的实锤：harness 本身可以是个插件

多数人以为 harness 和模型是绑死的。OpenClaw 的 README 原话恰恰相反："**Models and agent harnesses (Claude, Codex, local models) are plugins you can swap without changing anything else.**"（模型与 agent harness——Claude、Codex、本地模型——都是可以替换的插件，其他一切不动；2026-10-15 实测），其文档里还有专门的 "Agent harness plugins" 章节。

这说明三层已经彻底解耦：**模型可换**（适配层）、**harness 可换**（驱动循环的运行时）、**应用形态可换**（CLI/渠道）。你第 7 篇手写的 loop 是最简 harness；P2·5 要对比的 pydantic-ai / openai-agents / claude-agent-sdk，本质都是"别人写好的 harness + 模型适配层"——选框架，就是在选"要不要自己造车"。

## 3｜上下文管理：agent 为什么"忘事"，满了怎么办

读完本节你能理解：context window 是什么、token 预算怎么算、"忘事"的两种机制，以及裁剪/压缩/隔离三种手段——这是 PMTA 里 M 要素在单会话内的全部工程。

### 3.1 新名词 context window 与 token：模型的工作记忆及其计量

**context window（上下文窗口）** 大白话：模型一次能"看进去"的内容总量是有限的——像一块固定大小的白板，写满了就得擦掉旧的才能写新的。

定义与出处：OpenClaw compaction 文档的定义最直白——"**Every model has a context window: the maximum number of tokens it can process.** When a conversation approaches that limit, OpenClaw compacts older messages into a summary so the chat can continue."（每个模型都有上下文窗口：它能处理的最大 token 数；2026-10-15 实测）。计量单位是 token。

**token** 大白话：模型世界的"字节"——文本被切成的小片段。工程粗估：1 个英文常见词 ≈ 1 个 token，1 个汉字 ≈ 1~2 个 token（随 tokenizer 与上下文浮动，精确计数用各家 tokenizer/count 接口，`<以官方 docs 为准>`）。计费、限长、性能预算都按它算；当前主流模型的窗口在十万到百万 token 量级（具体数值随模型不同，`<以官方 docs 为准>`）。

**token budget（token 预算）**：把窗口当内存预算来管。关键在于 messages 的用法：**每一轮都是全量重发**（第 1 篇时序图的阶段②每次带全部历史）。后端类比：这不是可以只发增量的缓存，而是每轮都要把整张调用栈重新传一遍——轮次越多、工具结果越大，预算烧得越快。

### 3.2 上下文里到底装了什么：一份官方清单

Claude Code 官方文档列了它的 context window 里装的东西（2026-10-15 实测）：**对话历史、文件内容、命令输出、CLAUDE.md、auto memory、已加载的技能、系统指令**。对照五阶段看：阶段②每次发出的是全部；阶段④的工具结果（命令输出、文件内容）是增长最快的部分——**一次读入一个大文件，预算可能直接烧掉一大截**。管理预算的第一步是"看见开销"，第二步才是"省"。

### 3.3 "忘事"的两种机制：没写进去 vs 写进去又被挤掉

把"agent 忘事"拆成两种完全不同的病：

1. **跨会话失忆（没写进去）**：会话结束，messages 生命周期终止——下一轮对话从空白开始。这不是 bug，是 in-context 这层记忆的天然边界。解药是把记忆搬到会话外（P4·11 的三层记忆：工作记忆进 Redis、长期记忆进 Postgres/pgvector），本篇不展开。
2. **会话内被挤掉（写进去了，但窗口满了）**：单次会话跑长了，上下文逼近上限——这是本节的主战场。Claude Code 官方文档的提醒很直白：随着工作推进 context fills up，自动压缩后"早期对话里的详细指令可能丢失"，所以官方建议把持久性规则放进 CLAUDE.md 而不是依赖对话历史（2026-10-15 实测）。

第 2 篇说过"messages 列表就是 agent 的内部状态"——现在补上后半句：**这块内部状态装在一块有预算的内存里，装不下时会失真**。类比：寄存器数量有限，写满了必须先把某些值换出（spill）——换出的代价就是精度损失。

### 3.4 三种手段：裁剪、压缩、隔离

| 手段 | 做法 | 优点 | 代价 / 什么时候不选它 |
|---|---|---|---|
| 裁剪（truncation） | 直接丢掉最旧的消息 | 零成本、实现五行 | 丢掉的内容永久消失；只适合"旧的确实没用了"（如超长工具日志） |
| 压缩（compaction） | 把旧对话总结成摘要，替换原文 | 保留要点、可聚焦（指定保留什么） | **有损**：细节丢失、摘要可能漏关键约束；多花一次模型调用 |
| 隔离（isolation） | 把整块工作交给 subagent——它有自己的窗口 | 主上下文几乎不涨 | 跨窗口只能靠"摘要回传"沟通；多一次模型成本（P6·16 展开） |

三者不互斥，成熟 harness 打的是组合拳。Claude Code 的官方做法分两层（2026-10-15 实测）："It clears older tool outputs first, then summarizes the conversation if needed"——**先清旧工具输出**（裁剪：工具输出通常最大且时效最短），**不够再总结对话**（压缩），并且 "Your requests and key code snippets are preserved"（用户的请求与关键代码片段优先保留）。

### 3.5 实测行为：/compact 与 /context

Claude Code 里这两个机制都能手动触达（2026-10-15 实测，官方文档描述）：

- **/compact**：手动触发压缩，还可以带聚焦指令——如 `/compact focus on the API changes`（只重点保留与 API 变更相关的要点）。对应一个设计认识：**压缩是有损的，所以要让用户指定"保什么"**。
- **/context**：查看当前窗口被什么占用。对应另一条：**预算管理第一步是先看见开销**。

两个工程细节值得直接抄进你的 harness：

1. **工具调用与结果必须成对保留**。OpenClaw 明确它在选压缩切分点时 "keeps assistant tool calls paired with their matching toolResult entries"——把 assistant 的 tool_calls 从历史裁掉、却留下孤立的 tool 结果，会把模型搞糊涂（协议上这些消息必须配对，第 1 篇 §2.3）。
2. **磁盘全量、窗口有损**。OpenClaw 原文："The full conversation history stays on disk. Compaction only changes what the model sees on the next turn."（压缩只改"模型下一轮看到什么"，不改磁盘历史）——**可见性**与**有损性**分离，永远可以回去查原文。

还有一条官方明说的失败模式：如果单个文件或工具输出大到"每次压缩完立刻又满"，Claude Code 会在尝试几次后停止自动压缩并报错（官方称 thrashing error），而不是无限循环。**压缩本身也要有终止条件**——第 4 节的主题在这里提前露了一脸。

### 3.6 设计启示：什么该留在上下文

本节收束成三条设计规则（P4·11 工程化为三层记忆时直接复用）：

1. **当前任务的必要事实留在上下文**——用户这轮的目标、刚拿到的工具结果、关键中间结论；
2. **确定性规则不要依赖对话历史**——放进系统指令/持久文件（Claude Code 建议放 CLAUDE.md 的正是"每次会话都要生效"的规则），因为压缩可能把早期指令挤掉；
3. **超大工具结果先裁再进**——在工具端就做截断/摘要（只返回前 N 条、只回传关键字段），别指望窗口装得下。

一句话回收 PMTA：**本节管的是 M 要素里"in-context 这一层"**——上下文字符串的记忆；跨会话的工作记忆/长期记忆、向量检索与 RAG 分界线，P4·11 展开。

## 4｜终止条件：循环不会自己停

读完本节你能理解：为什么"模型自觉收尾"不可信，四类终止条件各自拦什么事故——这是 harness 层最重要的安全网。

### 4.1 新名词 termination condition：让循环必须停的硬规矩

大白话：**给 while 循环加的那几个"必须刹车"的开关**。

定义：终止条件（termination condition）是 harness 在循环外强制施加的停止规则，与模型自己的"我完成了"判定（阶段③ 的 finish_reason）共同决定循环何时结束。前者是工程兜底，后者是模型自判——**工程上永远假设后者会失效**。

事故场景先摆出来：模型陷入"调同一个工具 → 失败 → 原样重试"的循环，每轮都烧 token；一个缺了最大迭代数的 agent 挂在线上跑过周末，周一账单四位数。这不是吓唬——OpenClaw 的循环检测文档原话："**Break context-overflow -> compaction -> same-loop cycles instead of letting them run indefinitely**"（把"上下文溢出 → 压缩 → 同样的循环"这种环打破，而不是任其无限跑下去；2026-10-15 实测）。连"压缩省空间"这件事本身都需要防循环保护。

### 4.2 四类终止条件

**一、任务完成判定（模型自判，发生在阶段③）**
模型不再要求调工具（finish_reason = "stop"，第 1 篇 §2.3）即视为完成。注意两件事：① 这是"模型认为完成了"，不等于"真的完成了"——对不对要靠验收（P4·12 的反思与 verifier、P7·18 的 eval）；② 判定必须读结构化信号（finish_reason 变没变），**不能读内容**——模型在文本里说"我完成了"不算数。

**二、最大迭代数（harness 兜底，硬顶）**
无论模型多"自信"，超过 N 轮强制熔断。这是唯一不依赖任何外部判断的终止条件，也是新手最容易漏的一个。N 取多少：经验上普通工具型任务给 10~20 轮已很宽裕 `<业界常见做法，按任务复杂度调整>`；关键是**熔断后要有明确行为**（报错/降级/转人工），而不是静默返回空。实锤：OpenClaw 内置了一条窄护栏——"OpenClaw always stops a turn after **three consecutive identical tool errors**: the same tool, arguments, and error result."（连续三次完全相同的工具错误后必停；2026-10-15 实测）；更强的滚动历史循环检测（tools.loopDetection）可配置开启。本篇动手实践的演示 2 就是你亲手复现这个兜底。

**三、工具超时（单步级）**
每次工具调用设超时，卡死的工具不能拖死整个循环。粒度别搞混：这是"**单次调用**最多等多久"，最大迭代数是"**整个任务**最多跑几轮"。实锤：OpenClaw 的 agent loop 文档写明它 "enforces the current model attempt's timeout (aborting that attempt on expiry)"——连模型调用本身都有单次超时，到点中止该次尝试（2026-10-15 实测）。超时之后怎么重试、怎么退避，第 9 篇展开。

**四、用户确认与打断（人兜底，HITL 的起点）**
两类入口：**事前**——危险动作先停下来等人批准（§2.3 权限闸门的职责）；**事中**——人随时可以喊停或改向。Claude Code 官方明确 "You're part of this loop too"：按 Esc 立即停止，运行中的工具调用被取消、模型等你的下一条指令（2026-10-15 实测）。完整的人机协同设计（介入时机、三种介入方式、多轮打断的取消语义）是 P7·19 的主题，这里先埋住位置。

### 4.3 小结：四类终止 × 谁判定 × 拦什么事故

| 终止条件 | 谁判定 | 拦住的事故 | 本系列落点 |
|---|---|---|---|
| 任务完成判定 | 模型（finish_reason） | 误判"没完成就停"由其余三条兜底；"说完成了但没完成"靠验收 | P2·7 / P4·12 / P7·18 |
| 最大迭代数 | harness（硬顶） | 死循环烧钱；同错重试不进展 | 本篇实践演示 2 |
| 工具超时 | harness（单步） | 卡死的工具/模型调用拖死整轮 | P3·9 |
| 用户确认/打断 | 人 | 危险动作真动手；方向跑偏收不回来 | P7·19（闸门见 §2.3） |

一句话：**模型的"我好了"只是四种停止方式里最不可信的一种**——harness 的价值，就在于其余三种都不依赖模型的自觉。

## 动手实践：主循环跑通 agent loop——不接 LLM，先看清结构

目标：亲手看到五阶段转起来——桩 LLM 第 1 轮"点菜"、你的代码执行工具并回填、第 2 轮模型宣告完成；再亲手触发一次死循环，看最大迭代数熔断。做完你会验证两个设计决策：**回填是循环能继续的原因**（不回填，下一轮模型就"忘了"）；**熔断是模型自判失效时的唯一保险**。

> 按系列规划，`agent/` 项目目录从第 7 篇才开始建；本篇是独立脚本，放任意目录都能跑。

### 准备

纯 Python 标准库，零依赖、零网络、零 token 消耗（本篇实测 Python 3.9，3.8+ 均可）：

```bash
python --version   # 确认 3.8+，然后把下面的文件保存到任意目录
```

### 完整代码

保存为 `agent_loop_skeleton.py`（已实测，输出见下节）。`run_agent` 去掉日志与注释后核心控制流只有 12 行——这就是"15 行主循环"的完整形态：

```python
# agent_loop_skeleton.py
# 第 3 篇动手实践：手写 agent loop 核心骨架（不接真实 LLM，用桩函数模拟模型响应）
# 运行：python agent_loop_skeleton.py   （纯标准库，零依赖）
import json

FINISH_TOOL = "tool_calls"   # finish_reason 取值：模型想调工具（OpenAI 兼容接口）
FINISH_STOP = "stop"         # finish_reason 取值：模型收尾，不再调工具

# ---------- 桩 LLM 的两份"剧本"：代替真实模型 API（第 6 篇换成真的） ----------
def script_normal():
    """正常剧本：第 1 轮点菜（调工具），第 2 轮基于结果收尾。"""
    return [
        {"finish_reason": FINISH_TOOL,
         "tool_calls": [{"id": "call_1", "name": "get_user",
                         "arguments": {"name": "张三"}}]},
        {"finish_reason": FINISH_STOP,
         "content": "张三（ID 1001）当前是 admin 角色。"},
    ]

def script_stuck():
    """死循环剧本：模型反复用同样参数调工具——终止条件缺失时会发生什么。"""
    return [{"finish_reason": FINISH_TOOL,
             "tool_calls": [{"id": f"call_{n}", "name": "get_user",
                             "arguments": {"name": "张三"}}]}
            for n in range(1, 10)]

def fake_llm(messages, script):
    """桩：真实实现是一次 HTTP 调用，把【全部】messages + 工具清单发出去。"""
    resp = script.pop(0)
    print(f"  [阶段2 调LLM] 模型看到 {len(messages)} 条消息 -> finish_reason={resp['finish_reason']}")
    return resp

# ---------- 工具注册表：名字 -> 真正干活的函数（真实实现是 HTTP/SQL，第 8 篇展开） ----------
def get_user(name):
    return {"id": 1001, "name": name, "status": "active", "role": "admin"}

TOOLS = {"get_user": get_user}

# ---------- agent loop 本体：全篇的主角 ----------
def run_agent(user_input, script, max_iterations=5):
    messages = [{"role": "user", "content": user_input}]          # 阶段1 获取输入
    print(f"[阶段1 获取输入] messages = {len(messages)} 条")

    for i in range(1, max_iterations + 1):
        print(f"\n--- 第 {i} 轮 ---")
        resp = fake_llm(messages, script)                         # 阶段2 调 LLM

        if resp["finish_reason"] == FINISH_TOOL:                  # 阶段3 解析
            messages.append({"role": "assistant",                 # 阶段5 回填模型意图
                             "tool_calls": resp["tool_calls"]})
            print(f"  [阶段3 解析] 要调 {len(resp['tool_calls'])} 个工具")
            for call in resp["tool_calls"]:
                print(f"  [阶段4 执行] {call['name']}({call['arguments']})")
                result = TOOLS[call["name"]](**call["arguments"]) # 阶段4 真正动手
                print(f"  [阶段4 结果] {json.dumps(result, ensure_ascii=False)}")
                messages.append({"role": "tool", "tool_call_id": call["id"],
                                 "content": json.dumps(result, ensure_ascii=False)})  # 阶段5 回填结果
            print("  [阶段5 回填] 带着工具结果回到循环顶")
        else:                                                     # 终止条件之一：模型宣告完成
            messages.append({"role": "assistant", "content": resp["content"]})  # 阶段5 回填最终答复
            print(f"  [阶段3 解析] 无 tool_calls，任务完成：{resp['content']}")
            print(f"  [终态] messages 轨迹: {' -> '.join(m['role'] for m in messages)}")
            return "done"

    # 终止条件之二：最大迭代数熔断
    print(f"\n[熔断] 达到最大迭代数 {max_iterations}，强制终止；messages 已积累 "
          f"{len(messages)} 条却没有答案")
    return "max_iterations"

if __name__ == "__main__":
    print("========== 演示 1：正常闭环（点菜 -> 做菜 -> 回填 -> 收尾） ==========")
    run_agent("帮我查一下张三", script_normal())
    print("\n========== 演示 2：死循环现场（最大迭代数熔断兜底） ==========")
    run_agent("帮我查一下张三", script_stuck(), max_iterations=3)
```

### 运行与预期输出

```bash
python agent_loop_skeleton.py
```

预期输出（本篇已实测，Python 3.9，逐字一致）：

```
========== 演示 1：正常闭环（点菜 -> 做菜 -> 回填 -> 收尾） ==========
[阶段1 获取输入] messages = 1 条

--- 第 1 轮 ---
  [阶段2 调LLM] 模型看到 1 条消息 -> finish_reason=tool_calls
  [阶段3 解析] 要调 1 个工具
  [阶段4 执行] get_user({'name': '张三'})
  [阶段4 结果] {"id": 1001, "name": "张三", "status": "active", "role": "admin"}
  [阶段5 回填] 带着工具结果回到循环顶

--- 第 2 轮 ---
  [阶段2 调LLM] 模型看到 3 条消息 -> finish_reason=stop
  [阶段3 解析] 无 tool_calls，任务完成：张三（ID 1001）当前是 admin 角色。
  [终态] messages 轨迹: user -> assistant -> tool -> assistant

========== 演示 2：死循环现场（最大迭代数熔断兜底） ==========
[阶段1 获取输入] messages = 1 条

--- 第 1 轮 ---
  [阶段2 调LLM] 模型看到 1 条消息 -> finish_reason=tool_calls
  [阶段3 解析] 要调 1 个工具
  [阶段4 执行] get_user({'name': '张三'})
  [阶段4 结果] {"id": 1001, "name": "张三", "status": "active", "role": "admin"}
  [阶段5 回填] 带着工具结果回到循环顶

--- 第 2 轮 ---
  [阶段2 调LLM] 模型看到 3 条消息 -> finish_reason=tool_calls
  [阶段3 解析] 要调 1 个工具
  [阶段4 执行] get_user({'name': '张三'})
  [阶段4 结果] {"id": 1001, "name": "张三", "status": "active", "role": "admin"}
  [阶段5 回填] 带着工具结果回到循环顶

--- 第 3 轮 ---
  [阶段2 调LLM] 模型看到 5 条消息 -> finish_reason=tool_calls
  [阶段3 解析] 要调 1 个工具
  [阶段4 执行] get_user({'name': '张三'})
  [阶段4 结果] {"id": 1001, "name": "张三", "status": "active", "role": "admin"}
  [阶段5 回填] 带着工具结果回到循环顶

[熔断] 达到最大迭代数 3，强制终止；messages 已积累 7 条却没有答案
```

### 看懂输出：四个"亲眼所见"

1. **演示 1 的"模型看到 3 条消息"**：第 2 轮发给模型的不是 1 条新问题，而是全部历史（user + assistant 的调用意图 + tool 结果）——回填就是在这里起作用的。把两处 `messages.append(...)` 注释掉再跑一遍，你会看到桩 LLM 永远只看到 1 条消息、循环永远出不来结果：**没有回填，agent 就失忆**。
2. **演示 1 的终态轨迹 `user -> assistant -> tool -> assistant`**：这就是第 2 篇说的"messages 列表 = agent 的内部状态"的实物长相。
3. **演示 2 的"模型看到 1 → 3 → 5 条消息"**：死循环里上下文每轮都在变大——这正是第 3 节"预算烧得快"的现实来源；真实场景里窗口很快见底，接着触发压缩、压缩完继续错——OpenClaw 要专门写文档防的就是这个环。
4. **演示 2 的熔断**：模型从没说"我好了"（finish_reason 一直是 tool_calls），循环却停了——这就是四类终止条件里唯一不依赖模型自觉的那条。真实 harness（OpenClaw）用的是更聪明的判据："连续三次完全相同的工具错误必停"。

## 本篇结论

agent loop 是"调模型→解析→执行工具→回填→再调"的 while 循环，messages 列表是它唯一的内部状态；harness 是让循环能上生产的全部外围设施（调度/工具注册/事件流/权限/持久化/可观测，外加上下文管理）。
上下文窗口有 token 预算，满了用裁剪/压缩/隔离顶住——压缩有损、磁盘留全量；循环靠四类终止条件刹车：模型自判、最大迭代数、工具超时、用户确认，后三者不依赖模型自觉。
第 4 篇回答"循环里那个 `LLM()` 该填什么模型"：通用 vs 推理模型怎么选、国产模型怎么接，并给可运行的实测对比脚本。

## 参考资料

1. [Claude Code 官方文档 — How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) — agentic loop 三相位、"agentic harness" 定义原文、context window 构成与自动压缩行为（先清旧工具输出、再总结）、/compact 与 /context、thrashing error、permission modes 四档、会话 JSONL 落盘、subagent 独立上下文、Esc 打断。2026-10-15 curl 实测。
2. [OpenClaw 官方文档 — Agent loop](https://docs.openclaw.ai/concepts/agent-loop) — agent loop 官方定义（intake / context assembly / model inference / tool execution / streaming / persistence）、per-session 串行队列、attempt 超时、生命周期事件与 loop 内钩子（before_tool_call / before_compaction 等）。2026-10-15 curl 实测。
3. [OpenClaw 官方文档 — Compaction](https://docs.openclaw.ai/concepts/compaction) — context window 定义原文、压缩语义（旧轮总结、近轮保留、tool 调用与结果成对保留）、"全量历史留在磁盘，压缩只改模型所见"、溢出错误触发压缩重试。2026-10-15 curl 实测。
4. [OpenClaw 官方文档 — Tool-loop detection](https://docs.openclaw.ai/tools/loop-detection) — "连续三次完全相同工具错误必停"的内置护栏、防"溢出→压缩→同循环"的设计动机。2026-10-15 curl 实测。
5. [OpenClaw GitHub 仓库（openclaw/openclaw）](https://github.com/openclaw/openclaw) — "Models and agent harnesses are plugins you can swap"（harness 即插件）、"Gateway is the local control plane for sessions, tools, events, and channel connections"、MIT 许可与 OpenClaw Foundation 治理背景。2026-10-15 curl 实测（star 数 392.1k 为当日页面显示值，随时间变化）。
6. [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — agent 与 workflow 的分界（控制流归谁，第 1 篇已核验引用，本篇 §1.1 立场延续）。

> 核验说明：本篇所有涉及 Claude Code 与 OpenClaw 的行为描述均于 2026-10-15 用 curl 抓取官方页面逐字核验（code.claude.com / docs.openclaw.ai / github.com/openclaw/openclaw）；上下文窗口 token 数量级未写死具体数值（`<以官方 docs 为准>`）。动手实践代码已在本机 Python 3.9.6 实际运行，预期输出为实测逐字记录。

