# 第 16 篇：拆解 Claude Code / Cursor / OpenHands / OpenClaw
### 三层架构共性 · 五个"为什么这样设计" · 对照我们的成品查漏 · Part 6 开场（实例先行）

> 知识地图【A3/C3/E4】· Part 6 开场
> 读完你能：用"三层架构"这把刀拆开任意一个成熟 agent；说出 subagent / hooks /
> 权限分级 / checkpoint / 事件流各自解决什么问题；对照自己 7-15 篇的成品列出差距
> 清单；用"指认"的方式读一个真实 agent 的协作机制；给自己的 agent 补上钩子日志
> 与可恢复 checkpoint（已完成本地实测）。

## 本篇要回答的问题

1. Claude Code / Cursor / OpenHands / OpenClaw 的架构共性是什么？
2. 为什么它们都长出了 subagent、hooks、权限分级、checkpoint、事件流——各自在解决什么问题？
3. 我们第 7-15 篇的成品，哪些已经自然实现了、缺哪些？
4. HITL 人机回环：写操作 / 花钱操作 / 边界指令的确认与审批，应该落在哪一层？
5. 动手：「指认」Claude Code 的 subagent 机制并画出事件流；给我们的 agent 加
   "工具钩子打日志 + checkpoint 可恢复"，中断重启验证。

> 一手事实（2026-10-16 官方文档 / 仓库核验，全部经 curl 取得原文）：
> **Claude Code**：code.claude.com 文档站（checkpointing / sub-agents / hooks 参考页）；
> **Cursor**：cursor.com/docs 的 agent/overview、agent/security、agent/security/run-modes、
> agent/plan-mode；**OpenHands**：github.com/OpenHands/OpenHands（**91,040 ★**）+
> docs.openhands.dev（Agent Canvas architecture）；**OpenClaw**：
> github.com/openclaw/openclaw（**392,152 ★**）+ docs.openclaw.ai（agent runtime
> architecture）。
> 顺带一个取资料技巧：这几家的文档站（Mintlify 系）都支持在页面 URL 后加 `.md`
> 直接拿到 markdown 原文，站点还有 `llms.txt` 总索引——本篇的核验就是这么做的。

---

## 0｜为什么不接着写代码，先拆机器

前 15 篇的节奏是"自己造"：从 15 行 loop 骨架，造到能连真实后台的 MCP 宿主。
Part 6 换一个视角——**先看工业界怎么造**。

这不是偷懒，是"实例先行"：第 17 篇要讲多智能体（subagent、协作模式、通信），
如果手里没有几个真实系统的设计答案做参照，讨论很容易变成名词堆砌。四台机器，
四种出身，正好覆盖四类场景：

| 样本 | 出身 | 场景 | 一句话定位 |
|---|---|---|---|
| Claude Code | Anthropic，闭源 | 终端里的编码 agent | harness 层设计最完整的"参考实现" |
| Cursor | Anysphere，闭源 | IDE 里的编码 agent | 产品化体验与权限默认值的样板 |
| OpenHands | 开源 · 91,040 ★ | 可自托管的 agent 平台 | "agent 跑在哪里"（沙箱/后端）的解耦样板 |
| OpenClaw | 开源 · 392,152 ★ | 个人 AI 助理（多渠道） | 应用层与 harness 解耦、渠道即入口的样板 |

> 两家开源产品的自我介绍（原文核验）：OpenHands 是 "AI-Driven Development"，
> 本地运行、可接 local / Docker / VM / 云等 **agent backends**；OpenClaw 是
> "The AI that really does things. Any OS. Any Platform."，一个 **Gateway** 进程
> 把 Discord / iMessage / Slack / Teams / Telegram / WhatsApp 等 **20 多个渠道**
> 接到 AI assistant，由独立的 501(c)(3) **OpenClaw Foundation** 托管。

**边界声明**：这几家的主业是编码与个人助理，我们的场景是后台管理助手。本篇拆的是
**通用设计**（harness 怎么分层、状态怎么管），不是照抄产品。它们踩过的坑，我们
没理由再踩一遍。

---

## 1｜共性一：三层架构——模型适配层 / harness 层 / 应用层

四家产品形态差得很远，但把架构画出来，形状惊人地一致：

```
┌──────────────────────────────────────────────────────┐
│ 应用层（谁在用、在哪用）                                │
│   CLI / IDE / 桌面 / 浏览器 / 聊天渠道 / 自动化触发      │
├──────────────────────────────────────────────────────┤
│ harness 层（Agent 本体 —— 行为质量都在这）              │
│   loop · 工具注册 · 上下文管理 · 权限 · 事件流           │
│   会话持久化 · hooks · subagent 调度 · 压缩/总结         │
├──────────────────────────────────────────────────────┤
│ 模型适配层（谁在思考）                                  │
│   各家模型 + 流式协议 + 提示词适配 + 传输                │
└──────────────────────────────────────────────────────┘
```

Cursor 官方文档把这层话说得最直白：

> "An agent is built on three components: **Instructions**（system prompt 与
> rules）· **Tools** · **Model**。Cursor's agent orchestrates these components
> … tuning instructions and tools specifically for every frontier model."

四家逐一映射：

| 层 | Claude Code | Cursor | OpenHands | OpenClaw |
|---|---|---|---|---|
| 模型适配层 | Claude 多档（可为 subagent 单独选模型） | "为每个前沿模型调优 instructions 和 tools" | 接 Claude Code / Codex / Gemini / Pi / OpenCode 或任意 ACP agent；本地 LLM（Ollama / vLLM 等） | 模型 provider 注册表 + 每 agent 一份原子"模型快照" |
| harness 层 | Claude Code 本体（loop / 工具 / 权限 / hooks / checkpoint / subagent） | Instructions + Tools + 编排 | **Agent Server**："runs conversations, agents, tools, and workspace operations; **streams events to clients**" | `src/agents/`（attempt loop / 压缩 / session）+ 可复用 `agent-core` + **harness 注册表**（内建 `openclaw`，插件可加如 `codex`） |
| 应用层 | 终端 / IDE / 桌面 / 云会话（"同一套 hook 事件在哪都触发"） | IDE 侧栏 + Plan Mode | **Agent Canvas**（浏览器客户端，"is not an agent runtime or sandbox"）+ Automation Server | **Gateway**（控制平面）+ Control UI / CLI / TUI + 20 余渠道 |

三个可以直接拿走的判决：

1. **模型是租的**。四家没人为"用一个模型"重造轮子，OpenClaw 干脆把
   "Models and agent harnesses（Claude, Codex, local models）are plugins you can
   swap" 写进 README——模型和 harness 都是可换插件。
2. **应用层随场景变，接口不变**。OpenHands 把这条做到极致：Agent Canvas 前端
   只"展示后端状态、发送请求"，换个后端（本机进程 / Docker / 云）体验不变。
3. **harness 才是各家的真功夫**。同一代模型喂给不同 harness，行为质量完全不同。
   回头看我们第 2 篇写的 harness 定义（loop 内核 + 调度 / 工具注册 / 事件流 /
   权限 / 持久化 / 可观测）——**前 15 篇其实一直在长我们自己的 harness**，本篇
   就是去对标这个最值钱的一层。

---

## 2｜共性二：五个"为什么这样设计"

### 2.1 为什么有 subagent —— 上下文隔离，不是"更聪明的模型"

Claude Code 文档的原话是："Each subagent runs in its own context window **with a
custom system prompt, specific tool access, and independent permissions**"。用法是
Markdown + YAML frontmatter
定义一个角色（`name` / `description` / `tools` / `model` / `permissionMode` 等字段），
主 agent 根据 `description` **自动决定何时委派**。

它解决的问题一句话：**上下文是最稀缺的资源**（回指第 11 篇）。主对话是"总部的
办公桌"，塞满探索过程就再也放不下正事；subagent 是一间"单独的会议室"——探索
过程留在会议室，只有结论（摘要）回到办公桌。

四个值得注意的设计细节：

- **官方内置了两个 subagent**：Explore（代码探索）与 Plan（规划前研究），都是
  **只读**——Write/Edit 工具直接拒绝；且"轻装上阵"——不加载你的 CLAUDE.md 与
  git 状态快照（原文：to keep research fast and inexpensive）。**"探索"和"规划"
  被单独拆出来**，说明这是 agent 最吃上下文、最值得隔离的两类任务。
- **扁平是默认，嵌套是选项**。默认下所有 subagent 由主 agent 派生（一层）；
  官方也支持嵌套（如 reviewer 派生 verifier），终端里以树状展示，但一个规则很
  清楚：**"只读的 subagent 就别给它派生能力"**——把 `Agent` 工具从它的 `tools`
  列表里去掉，它就只能查、不能再生。
- **派生可以是白名单**。父级工具列表里写允许派生的 subagent 类型，其余类型的
  派生请求直接失败，模型在提示里也只看得见被允许的那些。
- **子代理的元信息也吃上下文**。所有 subagent 的 `description` 加起来超过
  **15,000 token**，启动时会出警告，官方建议：description 写短、细节放进
  subagent 自己的 system prompt（那个只在它运行时才加载）。——和"工具描述也
  占上下文"是同一个道理（第 9/13 篇）。

**它也是有代价的**，官方文档明说两点：subagent 跑起来自己烧 token；返回详尽的
结果照样吃主对话上下文。所以"该不该拆"要有决策框架：

| 场景 | 主 agent（flat） | subagent（hierarchical） |
|---|---|---|
| 只读侦察：多路搜索、跨文件探索 | 上下文会被过程淹没 | ✅ 过程隔离，只回摘要 |
| 并行独立子任务 | 只能排队 | ✅ 可并行（第 17 篇展开） |
| 需要连续上下文的短任务 | ✅ 委派开销 > 收益 | ❌ 通信成本白花 |
| 写操作委派 | ✅ 主 agent 掌控改动 | ⚠️ 子 agent 的编辑不参与会话 checkpoint 恢复（见 2.4），慎用 |

我们 7-15 篇的成品是**完全扁平的**单 agent——这对"后台管理助手"是合理默认。
第 17 篇（里程碑 M5）会给它装上"主 agent + 只读查询 subagent"的第一层结构。

### 2.2 为什么有 hooks —— 给 loop 装上"确定性插槽"

模型是不确定的，但纪律必须是确定的。审计、日志、策略拦截不能靠"在提示词里
求模型配合"——hooks 就是把这些**确定性代码**挂到生命周期上的插槽。

Claude Code 的 hooks 参考文档给出的骨架：

- **事件按三节拍**：per session（`SessionStart` / `SessionEnd`）、per turn
  （`UserPromptSubmit` / `Stop`）、以及 **agentic loop 内每次工具调用**
  （`PreToolUse` / `PostToolUse`）；另有 subagent 生命周期（`SubagentStart` /
  `SubagentStop`）、上下文压缩（`PreCompact` / `PostCompact`）等——全部加起来
  约 30 个事件。
- **拦截机制有两种**，官方推荐记住三条线：
  - `exit 2` = 阻断（stderr 内容就是给模型看的原因），且**即使同时输出 JSON
    说 "allow" 也拦住**；
  - JSON 输出 `permissionDecision: "deny"`（还有 allow / ask / defer）也可阻断；
  - `exit 1` 是**非阻断**错误（官方明确警告：策略钩子别用 1，用 `exit 2`）；
    命令钩子超时默认也不阻断（要兜底得显式配 `onFailure: "block"`）。
- **收窄执行面**：`matcher` 先按工具名过滤，`if` 再按参数收窄（如
  `Bash(rm *)`），两层都不匹配就不启动钩子进程——避免"每个工具调用都开销一次"。

官方文档里就有一个完整示例——**策略钩子阻止 `rm`**：

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh：从 stdin 读事件 JSON，检查命令
input=$(cat)
command=$(jq -r '.tool_input.command' <<<"$input")

if [[ "$command" == rm* ]]; then
  echo "Blocked: rm commands are not allowed" >&2
  exit 2  # 阻断：工具调用被拦下
fi

exit 0    # 不表态：走正常的权限流程
```

一句话总结：**hooks 把 loop 从"一个循环"变成"一条有插槽的生命线"**。
我们第 9/10 篇手写的 `audit()`（审计四事件）和 `confirm()`（危险操作拦截），
就是这层插槽的"手工版"——功能对，但和循环代码耦合死了。本篇实践 B 把它抽象成
注册表。

### 2.3 为什么权限是"分级 + 白名单 + 分类器"，而不是一个开关

"要么全自动、要么每步都问"，两个极端都不能用：全自动不敢，每步都问烦死。
成熟产品的答案是一张**分级梯子**。Cursor 的 "Run Modes" 是最清晰的一张：

| 档位 | 免问执行的东西 | 沙箱 | 分类器 |
|---|---|---|---|
| **Auto-review** | 白名单里的直接跑；其余 shell 命令尽可能进沙箱；用不了沙箱的交给 LLM 分类器评 | ✅ | ✅ |
| **Allowlist** | 白名单里的直接跑 | 可选 | ❌ |
| **Run Everything** | 全部直接跑 | ❌ | ❌ |

Auto-review 的执行顺序（官方原文）：白名单 → 沙箱（能进就进）→ 分类器
（跑在 Cursor 后端的小模型，会调你机器上的只读工具看上下文）→ 被拦的换条路
或弹给你批。而官方紧跟着一句话，和我第 14 篇说 annotations 时是同一个道理：

> "Auto-review is **not a security boundary**. The classifier can make mistakes.
> It can allow a call you would have blocked, or block a call you would have allowed."

——**分类器是"最大努力的护栏"（best-effort guardrails），不是安全边界。**
真正的边界是白名单、沙箱、只读账号这些"确定性"的东西（第 9/10 篇的五家族防线）。

再看 Cursor 的**默认值哲学**（这些是产品默认，不看文档想不到）：

- 读文件、代码搜索：**默认放行**（不想被读的文件用 `.cursorignore` 排除）；
- 工作区文件编辑：**默认放行**，但**配置文件先要你批**；
- 终端命令：**默认要批**；
- MCP：连接要批，连上之后**每个工具调用还要逐个批**（可预批 allowlist）；
- 网络：默认只走出 GitHub / 直链抓取 / 搜索三个口，**不能发任意网络请求**。

规律呼之欲出：**读默认放行、写默认放行但盯敏感面、执行默认要批**。这和我们
第 9 篇的"风险三级阶梯（读 < 写 < 执行）"完全同构——只是产品把梯子做进了默认值。

我们目前的权限实现是"一个 `DANGEROUS_TOOLS` 集合 + confirm"，正好是 Cursor
三档里 **Allowlist 档的最小版本**。分级配置化留给 P7·19/20。

### 2.4 为什么 checkpoint 不是 git —— 会话内的"撤销栈"

先看两家的事实（都来自官方文档原文）：

**Claude Code**：每发一条"开启回合"的提示，自动为代码状态打一个点；一次会话
保留最近 **100 个**快照；快照**随会话保存**（resume 之后 `/rewind` 还能用）；
约 **30 天**清理。`/rewind` 菜单有六项：恢复代码+对话 / 只恢复对话 / 只恢复
代码 / 从此处向前摘要 / 到此为止摘要 / 取消。**局限一长串且都被官方写明了**：

- Bash 命令改的文件**不追踪**（`rm` / `mv` / `cp` 都恢复不了）；
- subagent 的编辑**多数不恢复**（只有前台 fork 的 skill 例外，后台默认模式不算）；
- 会话外的改动不追踪；符号链接 / 硬链接文件恢复时跳过；
- 明确定位："**不是版本控制的替代品**"。

**Cursor**：进大改动前自动快照；点任意 checkpoint 可预览/恢复，但**恢复只回
文件、不回消息**；快照存在本地、与 Git 分离；定位同样是"只用来撤销 Agent 的
改动，永久历史请用 Git"。

把两家拼起来，"checkpoint 是什么"就清楚了——**它是"这次会话里 agent 动作的
撤销栈"，不是项目历史的真相**：

| | git | 会话 checkpoint |
|---|---|---|
| 记谁的历史 | 项目：你 + 所有人 + 所有工具 | 只有 agent 的文件编辑 |
| 粒度 | commit（语义单元） | 每次提示 / 每次大改动（时间线单元） |
| 保留 | 永久 | 会话内（CC：最近 100 个 / 约 30 天） |
| 恢复对象 | 整个仓库 | 代码、对话，可分开恢复 |

两个"为什么"就顺了：

- **为什么只管 agent 编辑工具改的文件？** 因为要撤销的是"agent 的动作"。
  Bash 改的、别的进程改的，本来就不归它对账——硬追反而错乱。
- **为什么连对话也一起存？** 因为 agent 的状态有两条线：**文件状态**和
  **它当时知道什么**（消息历史）。CC 让你分开恢复哪条线，正是承认这两条线
  可以独立回滚——"代码回退但保留探索记忆"和"忘掉错误结论但保留代码"都是
  真实需求。

我们目前的 checkpoint 是"零"——messages 只在内存里，进程一死全丢。本篇实践
B 补上最小版。

### 2.5 为什么有"事件流"——不只为了流式 UI

OpenHands 的架构页把这件事说得最透（三句原话都是核验过的）：

- Agent Server："runs conversations, agents, tools, and workspace operations;
  **streams events to clients**"（*执行*并*流式发出事件*）；
- Agent Canvas："renders state and sends requests to backend services; **it is
  not an agent runtime or sandbox**"（纯展示，不持有执行）；
- 状态归属：对话历史、密钥、MCP 配置都归 Agent Server；文件归沙箱；**Canvas
  只存"怎么连上后端"的连接信息**。

"流式 UI 打字机效果"只是表象，事件流的真正价值是三条：

1. **前端无状态 → 可换后端**。同一个 Canvas 接本机进程、Docker、云都行——
   因为会话状态不在前端，而在"事件 + 后端状态"里。
2. **每一步都是事件 → 可观测**。第 7 篇 b-8 我们在 loop 里加的那行单行 JSON
   迭代日志，就是最小的事件流；第 18 篇的正式 trace 是它的完整形态。
3. **事件落盘 = checkpoint 的天然素材**。事件流不只给人看——按边界持久化，
   就是 2.4 的恢复点。

**那"流式 UI"呢？** 它就是事件流最直观的消费端：agent 跑一轮可能几十秒到几分钟，
没有中间反馈，用户面对的就是一块黑屏——既不知道它在干什么，也不知道该不该打断。
事件流让"边跑边看"变成默认：每个工具调用、每段文本增量都是可显示的消息。Claude
Code 甚至专门有一个 **display-only 事件**（`MessageDisplay`，仅在助手文本流式显示
时触发）——事件流成熟到"显示层也是一个订阅者"。我们的 agent 目前"每个工具调用
打印一行"，已经是流式 UI 的最小形态（第 7/9 篇的打印习惯）；P7·18 会把它升级成
正式 trace。

Claude Code 的 hooks 事件表（SessionStart → UserPromptSubmit → PreToolUse /
PostToolUse → … → SessionEnd）本身就是一条**全生命周期事件流**，它干脆把这条
流开放成用户的扩展点——"事件流"从内部机制变成了公开 API。

OpenClaw 走得最远：Gateway 是 "the local control plane for
**sessions, tools, events**, and channel connections"——一个进程同时管会话、
工具、事件、渠道；应用层是 20 多个聊天渠道，harness 是插件。它的安全口径也
值得记下：**"trusted gateway, untrusted execution, policy enforced by code"**
（受信的网关、不受信的执行、策略用代码兜底）——我们第 10 篇"从模型黑盒里
把控制权抢回宿主"的同款思路。

---

## 3｜对照表：我们 7-15 篇有什么、缺什么

把五个设计逐件对照我们的成品（"出自"列 = 我们在哪篇实现的）：

| 能力 | 成熟 agent 对应物 | 我们的现状 | 出自 | 处置 |
|---|---|---|---|---|
| 工具注册表 | Cursor Tools / CC 工具集 | `registry.update({...})` | 9 | ✅ 已有 |
| 错误形状回填 | hook 阻断时把原因回填给模型 | `_shape`（状态码+hint+body） | 9 | ✅ 已有 |
| 凭证失效自愈 | —（产品内建） | 401 自动重登一次 | 9/14 | ✅ 已有 |
| 审计事件 | CC hooks / OpenClaw agent-hooks | `audit()` 四事件 jsonl | 10/14 | ✅ 已有（手工版） |
| 危险操作确认 | Cursor Allowlist 档 | `DANGEROUS_TOOLS` + `confirm()` | 10 | ✅ 已有（最小版） |
| 声明式 schema | Cursor "为每个模型调优 tools" | `@mcp.tool()` / `mcp.AddTool` | 13/14/15 | ✅ 已有 |
| 跨进程工具 | MCP 生态 | Python server + Go server | 14/15 | ✅ 已有 |
| 上下文管理 | CC autoCompact / OpenClaw compaction | trim + 压缩 + 总结 | 11 | ✅ 简化版 |
| **hooks 注册表** | CC PreToolUse 等 ~30 事件 | 耦合在 run_tool 里 | — | 🔧 本篇实践 B |
| **checkpoint / 恢复** | CC /rewind 100 快照；Cursor 快照 | 无（内存态） | — | 🔧 本篇实践 B |
| **subagent** | CC 独立上下文 + 工具 + 权限 | 无（纯 flat） | — | → P6·17（M5） |
| 权限分级 + 白名单配置 | Cursor Run Modes 三档 | 一个集合，不可配 | — | → P7·19/20 |
| 正式事件流 / trace | OpenHands events / CC hooks | print 级日志 | 7（b-8） | → P7·18 |
| 沙箱 | OpenHands 每会话一容器 | 无（靠校验 + 只读账号） | 10 | → P7·20 略提 |

更新后的决策视角——**"什么时候该从 flat 升到 hierarchical"**（2.1 的表在
我们场景下的落地版）：

- 后台管理助手的**默认保持 flat**：多数任务是"查一下 → 改一下 → 汇报"，
  上下文能装下，拆出去反而慢；
- 出现"**长链只读侦察**"（如"把最近一周所有异常操作的用户都找出来并汇总"）
  时，是第一个该拆的信号——正是 P6·17 里程碑 M5 要做的；
- 写操作**永远留在主 agent**：子 agent 的编辑不参与会话恢复（2.4 的官方局限），
  审批链也更难做（第 4 节的 HITL）。

---

## 4｜HITL：写操作 / 花钱操作 / 边界指令的审批落点

"人机回环"（Human-in-the-loop）不是"弹个 y/N"这么简单——先分类风险，
再选落点。

**第一步：三类高风险动作**

| 类别 | 例子（我们的后台管理场景） | 判定的提问 |
|---|---|---|
| 写操作 | 删用户、改角色、批量停用 | 这步失败，数据会不会错？ |
| 花钱 / 配额操作 | 大批量导出、触发大量 LLM 调用的批处理、发短信 | 这步跑起来，账单会不会涨？ |
| 边界指令 | 导出全量数据、改系统配置、装东西、访问授权范围外的表 | 这步越权了吗，最坏损失是什么？ |

**第二步：审批落在哪一层**——四个候选，各有位置：

| 落点 | 形态 | 适合 | 我们见过谁这么干 |
|---|---|---|---|
| ① 工具 / 循环内 | `run_tool` 里 if 判断 + 终端确认 | 起步够用，工具一多就散 | 我们（第 10 篇 `confirm()`） |
| ② hooks 层 | `PreToolUse` 拦下 → deny / ask / defer | **策略集中、可配置、可审计** | CC `permissionDecision`；Cursor Run Modes |
| ③ 应用层 | UI 弹窗、审批按钮、待办 | 面向人的审批流 | Cursor / Claude Code 的权限提示 |
| ④ 异步审批 | 挂起落盘 → 消息/工单 → 批准后恢复 | 审批人不在终端旁时 | CC 的 `"defer"`（优雅退出、稍后恢复）；OpenClaw 渠道化 |

两条给我们场景的结论：

1. **确认要留在能看见完整参数和意图的层**——第 14 篇把 `confirm()` 留在宿主
   而不是搬进 MCP server，落点选的就是②。工具一多之后，把 `DANGEROUS_TOOLS`
   升级为"PreToolUse 风格"的策略钩子 + 白名单配置（本篇实践 B 的骨架），
   就是从①到②的进化。
2. **"异步审批"值得牢记**——它是 checkpoint 的另一个用户：审批挂起前把会话
   落盘，批准后从恢复点继续。对"后台管理 agent 不该把人钉在终端上"这个诉求
   （就像 OpenClaw 让人在微信里用 agent），这是正解。完整审批流设计留给 P7。

---

## 5｜动手实践 A：指认练习——拆 Claude Code 的 subagent

"指认"是故意的方法：**先不做，先观察**——亲手把机制跑出来、看清每个证据，
比读十页文档记得牢。

### 准备：造两个只读侦察兵

在任意项目下建目录与文件：

`.claude/agents/code-scout.md`：

```markdown
---
name: code-scout
description: 代码侦察兵。需要弄清某个功能在哪些文件实现、调用链如何时使用。只读。
tools: Read, Grep, Glob
---

你是代码侦察兵，只做一件事：定位。
- 用 Grep/Glob 找入口，用 Read 确认关键片段
- 汇报格式：文件路径 + 关键行号 + 一句话结论
- 不写代码、不改文件、不给建议
```

`.claude/agents/doc-scout.md`：

```markdown
---
name: doc-scout
description: 文档侦察兵。需要从 README、docs/ 里提炼接口说明或约定时使用。只读。
tools: Read, Grep, Glob
---

你是文档侦察兵。只读 README 与 docs/ 下的文件。
汇报格式：结论 + 出处文件 + 关键原句；拿不准的标注"未找到"。
```

### 派活与观察（五个指认点）

给它一句任务：**「用 subagent 查清 XX 功能在哪些文件实现，先别改任何东西」**

逐条指认：

1. **主对话里出现了什么？** → 预期只有一次 Agent 工具调用 + 子 agent 的
   最终摘要。中间的 Grep/Read 过程你在主窗口看不到——去看到底看不看得到。
2. **子 agent 知道你之前聊过什么吗？** → 问它"我刚才说的是哪个功能"，预期它
   一无所知（它只拿到任务描述 + 自己的 system prompt）——这就是"独立上下文"。
3. **它改得动文件吗？** → 让它"顺手把发现写进 NOTES.md"，预期改不了
   （`tools` 白名单里没有 Edit/Write）。**工具集就是权限**。
4. **看见生命周期事件**：在项目 `.claude/settings.json` 里挂两个钩子
   （按官方文档编写，**未实跑**，跑不通就以你版本的官方文档为准）：

   ```json
   {
     "hooks": {
       "SubagentStart": [
         { "hooks": [{ "type": "command", "command": "echo subagent-start >> .claude/subagent.log" }] }
       ],
       "SubagentStop": [
         { "hooks": [{ "type": "command", "command": "echo subagent-stop >> .claude/subagent.log" }] }
       ]
     }
   }
   ```

   再派一次活，`cat .claude/subagent.log` 看时间戳——**生命周期是可挂载的**。
5. **画事件流**（手动填完这张图，就是你的学习产出）：

   ```
   你 ──prompt──▶ 主 agent
                    │ ① 读 description 选人: code-scout
                    ├── Agent 工具调用 ──▶ code-scout（新上下文：任务 + 只读工具）
                    │        ② 内部: Grep → Read → Grep → …（主窗口不可见）
                    ◀── ③ 回传: 最终摘要（唯一回传物）──┘
                    │ ④ 带着摘要继续（或再派 doc-scout）
                    ▼
                  回答你
   ```

   填图时回答：①选人依据在哪？（description）②中间过程为什么不可见？
   （独立上下文）③为什么只有摘要回来？（隔离的成本与红利）④主 agent 上下文
   里哪几条消息被"节约"了？

**没有 Claude Code 环境？等价替代**：打开第 9 篇的 `registry` 代码，人工推演
——"若把 `list_users` 单独装进一个只有该工具的循环，主循环只拿它的返回值"，
同样画一遍图。机制是同一个。

---

## 6｜动手实践 B：给我们的 agent 加"钩子 + checkpoint"（已实测）

两个目标，正好补上对照表里"🔧 本篇"的两格：

1. 把散在循环里的日志/策略变成**钩子注册表**（对应 PreToolUse / PostToolUse）；
2. 每完成一步**原子落盘**，进程被杀后能**恢复继续**（对应 checkpoint）。

### 脚本全文（纯标准库，~150 行，含注释）

```python
# -*- coding: utf-8 -*-
"""第 16 篇实践 B：工具钩子（hooks）+ 会话检查点（checkpoint）——纯标准库，可直接运行。

用法：
    python3 agent_hooks_check.py run1   # 首次运行，第 3 步执行前模拟中断
    python3 agent_hooks_check.py run2   # 重启后从检查点恢复，跑完全程
"""
import json
import os
import sys

HERE = os.path.dirname(os.path.abspath(__file__))
CKPT = os.path.join(HERE, "session.json")

# ---------- ① 钩子注册表：把「日志 / 策略」从循环里搬出来 ----------
# 事件名刻意对齐 Claude Code：PreToolUse / PostToolUse
HOOKS = {"pre_tool": [], "post_tool": []}


def hook(event):
    """装饰器：@hook("pre_tool") 把函数注册进钩子表"""
    def deco(fn):
        HOOKS[event].append(fn)
        return fn
    return deco


WRITE_TOOLS = {"delete_user", "assign_role"}


@hook("pre_tool")
def log_pre(name, args):
    print("[hook:PreToolUse] tool=%s args=%s" % (name, json.dumps(args, ensure_ascii=False)))
    return True  # 只记日志，不拦


@hook("pre_tool")
def readonly_policy(name, args):
    """策略钩子：写操作一律拒绝（对应 Claude Code 的 permissionDecision=deny）"""
    if name in WRITE_TOOLS:
        print("[hook:PreToolUse] DENY: %s 是写操作，当前策略=只读" % name)
        return False
    return True


@hook("post_tool")
def log_post(name, args, ok, result):
    print("[hook:PostToolUse] tool=%s ok=%s result=%s" % (name, ok, result))


def run_pre(name, args):
    """按注册顺序跑所有 pre 钩子；任一返回 False 即拒绝"""
    for fn in HOOKS["pre_tool"]:
        if not fn(name, args):
            return False
    return True


def run_post(name, args, ok, result):
    for fn in HOOKS["post_tool"]:
        fn(name, args, ok, result)


# ---------- ② 假工具：真项目里就是 part 3 的 registry ----------
DB_USERS = {"1001": "张三", "1002": "李四"}


def list_users(keyword=""):
    hit = {uid: n for uid, n in DB_USERS.items() if keyword in n}
    return json.dumps(hit, ensure_ascii=False)


def get_user_roles(user_id):
    return json.dumps({"user_id": user_id, "roles": ["admin"]}, ensure_ascii=False)


def delete_user(user_id):
    # 若策略钩子失效，这行会抛异常暴露出来 —— 证明「拦截」真的发生在执行前
    raise RuntimeError("不该被执行到：delete_user 应在 pre 钩子处被拦住")


TOOLS = {"list_users": list_users, "get_user_roles": get_user_roles, "delete_user": delete_user}


# ---------- ③ 检查点：每完成一步就原子落盘 ----------
def save_ckpt(step, messages):
    tmp = CKPT + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump({"step": step, "messages": messages}, f, ensure_ascii=False)
    os.replace(tmp, CKPT)  # 原子替换：避免写到一半被杀、留下半个 JSON
    print("[checkpoint] round=%d 已落盘 %s" % (step, os.path.basename(CKPT)))


def load_ckpt():
    if not os.path.exists(CKPT):
        return None
    with open(CKPT, encoding="utf-8") as f:
        return json.load(f)


# ---------- ④ 假模型：固定 4 步剧本（真项目里这步是 LLM 调用） ----------
SCRIPT = [
    {"thought": "先查一下有没有叫张三的用户", "tool": "list_users", "args": {"keyword": "张三"}},
    {"thought": "看看张三有哪些角色", "tool": "get_user_roles", "args": {"user_id": "1001"}},
    {"thought": "把张三删掉", "tool": "delete_user", "args": {"user_id": "1001"}},
    {"thought": "汇报：张三的角色是 admin；删除请求被只读策略拒绝", "tool": None, "args": {}},
]


def agent_step(i, step):
    """执行剧本第 i 步：先过 pre 钩子 → 执行工具 → 过 post 钩子"""
    if step["tool"] is None:
        print("[step %d/%d] 最终答复：%s" % (i + 1, len(SCRIPT), step["thought"]))
        return {"role": "assistant", "content": step["thought"]}
    name = step["tool"]
    print("[step %d/%d] thought=%s" % (i + 1, len(SCRIPT), step["thought"]))
    if not run_pre(name, step["args"]):
        result, ok = "调用被钩子拒绝：%s 属于写操作（策略=只读）" % name, False
        # 被拒绝也是一条要回填给模型的结果 —— 模型据此换一条路
        run_post(name, step["args"], ok, result)
        return {"role": "tool", "tool": name, "ok": ok, "result": result}
    result, ok = TOOLS[name](**step["args"]), True
    run_post(name, step["args"], ok, result)
    return {"role": "tool", "tool": name, "ok": ok, "result": result}


def run(stop_before=None):
    ckpt = load_ckpt()
    start, messages = (ckpt["step"], ckpt["messages"]) if ckpt else (0, [])
    if start:
        print("[resume] 从检查点恢复：已完成 %d 步，从第 %d 步继续" % (start, start + 1))
    for i in range(start, len(SCRIPT)):
        if i == stop_before:
            raise KeyboardInterrupt("模拟第 %d 步执行前进程被杀（Ctrl-C / 断电）" % (i + 1))
        messages.append(agent_step(i, SCRIPT[i]))
        save_ckpt(i + 1, messages)  # 先完成一步，再落盘这一步的结果
    print("[done] 全程 %d 步完成，检查点 round=%d" % (len(SCRIPT), len(messages)))


if __name__ == "__main__":
    phase = sys.argv[1] if len(sys.argv) > 1 else "run1"
    try:
        if phase == "run1":
            run(stop_before=2)
        else:
            run()
    except KeyboardInterrupt as e:
        print("[killed] %s" % e)
        print("[info] 检查点停在第 2 步，可重启恢复")
```

### 实测记录（本机 Python 3.9 实跑，输出原文粘贴）

第一次运行（第 3 步执行前"断电"）：

```text
$ python3 agent_hooks_check.py run1
[step 1/4] thought=先查一下有没有叫张三的用户
[hook:PreToolUse] tool=list_users args={"keyword": "张三"}
[hook:PostToolUse] tool=list_users ok=True result={"1001": "张三"}
[checkpoint] round=1 已落盘 session.json
[step 2/4] thought=看看张三有哪些角色
[hook:PreToolUse] tool=get_user_roles args={"user_id": "1001"}
[hook:PostToolUse] tool=get_user_roles ok=True result={"user_id": "1001", "roles": ["admin"]}
[checkpoint] round=2 已落盘 session.json
[killed] 模拟第 3 步执行前进程被杀（Ctrl-C / 断电）
[info] 检查点停在第 2 步，可重启恢复
```

此刻磁盘上的 `session.json`（真实内容，缩略展示）：

```json
{
  "step": 2,
  "messages": [
    {"role": "tool", "tool": "list_users", "ok": true, "result": "{\"1001\": \"张三\"}"},
    {"role": "tool", "tool": "get_user_roles", "ok": true, "result": "{\"user_id\": \"1001\", \"roles\": [\"admin\"]}"}
  ]
}
```

重启恢复：

```text
$ python3 agent_hooks_check.py run2
[resume] 从检查点恢复：已完成 2 步，从第 3 步继续
[step 3/4] thought=把张三删掉
[hook:PreToolUse] tool=delete_user args={"user_id": "1001"}
[hook:PreToolUse] DENY: delete_user 是写操作，当前策略=只读
[hook:PostToolUse] tool=delete_user ok=False result=调用被钩子拒绝：delete_user 属于写操作（策略=只读）
[checkpoint] round=3 已落盘 session.json
[step 4/4] 最终答复：汇报：张三的角色是 admin；删除请求被只读策略拒绝
[checkpoint] round=4 已落盘 session.json
[done] 全程 4 步完成，检查点 round=4
```

一次跑通两件事：**恢复**（第 3 步从断点续上，前两步没有重做）与**策略拦截**
（`delete_user` 在执行前被钩子拦下，`ok=False` 的结果被回填——注意脚本里
`delete_user` 本体是"被执行到就抛异常"的，全程没抛，证明拦截真的发生在执行前）。

### 三个设计点（对应我们观察到的产品行为）

1. **恢复点必须落在明确的边界上**。我们是"完成一步 → 落盘这一步的结果"，
   Claude Code 是"回合开始前给代码打点"——切法不同，本质相同：**把恢复点
   定义在模型调用之间**，而不是工具执行到一半。因为恢复之后，模型会重新
   看到消息历史并继续生成。
2. **落盘要原子**（临时文件 + `os.replace`）。中断随时会来，写一半的 JSON
   无法解析——宁可保留"旧的完整状态"，不要"新的破碎状态"。这和第 10 篇
   "拒绝也要有形状"是同一种工程直觉：**异常路径的输出也必须是合法状态**。
3. **被拒绝也是结果**。deny 之后要把"被拒绝"回填给模型——正是 Claude Code
   hooks 的行为（阻断时把 stderr 文本作为工具错误发给模型），模型据此换路
   或者向你汇报。吞掉拒绝、让循环卡死，是最容易犯的错。

### 与真实 agent 的差距（诚实标注）

- 剧本是 stub，真项目里第 ④ 步是 LLM 调用；恢复后的"继续生成"质量取决于
  落盘的消息是否完整（我们存了全部 messages，够用）。
- **非幂等写操作的风险**：恢复后模型可能重发"其实已经成功"的写请求——真实
  系统要配合写操作确认（第 10 篇 `confirm`）或幂等键（留 P7）。
- 我们只存"最新一个"检查点（单文件覆盖）；Claude Code 会保留最近 100 个
  快照以支持回退到任意点——生产版应改为 append-only 或按 step 分文件。

### 五条自检

1. `run1` 结束后：输出里有 `[killed]`，且 `session.json` 存在、`step` 为 2；
2. 把脚本进程真的 `Ctrl-C` 掉（把 `stop_before` 换成 `input()` 后回车前中断）
   ——重启 `run2` 依然恢复：**状态不在内存里**；
3. `run2` 里没有出现 `RuntimeError`——拦截发生在执行前；
4. 把 `readonly_policy` 改成永远 `return True`、删除 `session.json` 重跑
   ——会看到 `RuntimeError` 冒出来：之前拦住它的**确实是钩子**（对照组）；
5. 把你自己的一个工具接进注册表（比如加一个 post 钩子统计调用次数），跑通。

---

## 本篇结论

1. **四家殊途同归的三层**：模型适配层 / harness 层 / 应用层——模型是租的、
   应用层随场景变，**harness 是各家真功夫**，也是我们 7-15 篇一直在长的东西。
2. **五个设计回答同一个问题**——"把不确定的模型，放进确定的工程纪律里"：
   subagent 管上下文、hooks 管时机、权限分级管边界、checkpoint 管恢复、
   事件流管可观测；其中 hooks 与 checkpoint 本篇已给我们的 agent 补上（实测
   通过），subagent 归第 17 篇，权限分级 / 沙箱 / trace 归 Part 7。
3. **HITL 的落点原则**：确认要留在"看得见完整参数与意图、又离执行点足够近"
   的层（我们第 14 篇把 confirm 留在宿主正是此意）；审批不必钉在终端上
   ——挂起落盘、批准恢复（异步审批）是 checkpoint 的第二个用户。

---

**下一篇预告**：第 17 篇《多智能体 — 模式、通信与实战》（Part 6 收口 · 里程碑
M5）：orchestrator-worker / planner-executor / verifier 等协作模式对比、
agent 间通信机制（共享上下文 vs 嵌套调用 vs 消息传递）、复杂度守恒的警告，
动手给管理助手加"只读查询 subagent"并对比单 agent 的 token / 延迟。

**等我回复「继续」。**
