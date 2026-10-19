# 第 16 篇：拆解 Claude Code / Cursor / OpenHands / OpenClaw

> **知识地图【A3/C3/E4】· Part 6** · 到第 15 篇为止，我们的助手已经五脏俱全：loop、工具、记忆、规划、MCP、保护分层。但它们全挤在**一个** agent 里。本篇换个视角——去读四个成熟系统（Claude Code / Cursor / OpenHands / OpenClaw）的公开设计与文档，看它们为什么长成那样；对照之下，我们的成品缺什么、下一步补什么。第 17 篇再把学到的模式落地成主 agent + subagent。

## 本篇要回答的问题

1. 成熟 agent 系统的共性架构是什么？三层各管什么——我们第 3 篇画的 harness 六件套对得上号吗？
2. 四个系统逐个看：subagent、hooks、权限白名单、checkpoint 恢复、事件流——每个设计背后的「为什么」是什么？
3. 对照我们第 7-15 篇的成品：哪些已经自然实现、哪些还缺？层级化（hierarchical）什么时候比扁平（flat）更对？
4. HITL 人机回环（写/花钱/边界操作的确认与审批）在这些系统里落在哪？
5. 动手：指认 Claude Code 的 subagent 机制并画事件流；给自己的 agent 补上「工具钩子 + checkpoint 可恢复」两块缺失。

---

## 1｜三层架构：四个系统的共性骨架

把四个系统的公开文档与设计放在一起，会看到同一个骨架（术语各家不同，分层一致）：

```
┌─────────────────────────────────────────────────────────────┐
│ 应用层（application）     各家长得最不一样的地方              │
│   Claude Code：CLI + 终端 UI + 权限提示 + /命令              │
│   Cursor：编辑器内嵌 agent（diff 视图、内联审查）             │
│   OpenHands：Web UI + 浏览器/终端面板                        │
│   OpenClaw：可插拔 harness 的多形态宿主                      │
├─────────────────────────────────────────────────────────────┤
│ harness 层（agent runtime / 内核）  ← 我们的主战场            │
│   agent loop（感知→思考→行动→观察循环）                       │
│   工具注册 / 上下文管理（含 compaction）/ 权限闸门            │
│   事件流（可观测）/ 持久化（checkpoint、会话恢复）            │
│   subagent 调度（部分系统）                                   │
├─────────────────────────────────────────────────────────────┤
│ 模型适配层（model providers）                                │
│   provider 抽象 / base_url 切换 / 流式解析 —— 第 5 篇已见     │
└─────────────────────────────────────────────────────────────┘
```

**对上号了**：第 3 篇 §2.2 画的 harness 六件套（调度 / 工具注册 / 事件流 / 权限 / 持久化 / 可观测）正是这个中间层。我们不是在造一个玩具然后仰望成品——第 7-15 篇逐篇补齐的东西，恰好按同样的次序铺在这同一层上：

| harness 六件套（第 3 篇） | 我们的实现 | 篇 |
|---|---|---|
| 调度（loop） | run_agent 五阶段 | 3、7 |
| 工具注册 | TOOL_SCHEMAS/REGISTRY → MCPToolset | 7、8、14 |
| 上下文管理 | 裁剪/摘要（记忆层） | 3、11 |
| 权限 | guard 确认闸门 + 语义白名单 + 分层 | 9、10、14 |
| 持久化 | 记忆 JSONL/Redis —— **会话 checkpoint 还没有** | 11 |
| 可观测 | 审计 JSONL —— **结构化事件流还没有** | 9 |

## 2｜逐个「为什么这样设计」

### 2.1 Claude Code：把 harness 做成产品

Claude Code 的官方文档（code.claude.com，第 2/3 篇已核验）把它自己的循环叫 agentic loop、把整体叫 agentic harness。逐个设计看动机：

- **subagent（子代理）**：官方支持在 `.claude/agents/` 下用带 frontmatter（name / description / tools 等，字段以官方文档为准）的文件定义子代理；主 agent 通过任务派发让它独立跑，**子代理有自己的上下文窗口，主对话只收到最终报告**。动机一句话：上下文隔离——查十个文件的过程日志会撑爆主上下文，而主 agent 只需要结论。这正是我们第 11 篇「token 预算」问题在多任务形态下的解法，第 17 篇落地。
- **hooks**：官方提供工具调用前后等生命周期挂载点，可编程干预（阻止、注入上下文、打日志）。动机：把「策略」从 prompt 约束升级为代码强制——我们的 guard.confirm 本质就是一个写死的 before-hook，本篇实践把它泛化。
- **权限提示与白名单**：首次执行某类操作时询问，允许「仅此一次 / 整个会话 / 永久（写入配置）」。动机：在安全与流畅之间给用户三个档位——我们的闸门只有 y/N 两档，白名单是我们缺的档位（第 19 篇补）。
- **会话 checkpoint 与恢复**：对话落盘为 JSONL（第 3 篇核验过文档表述），支持恢复与回溯。动机：长任务的进程边界——agent 跑了半小时被打断，不该从零开始。我们完全没做，本篇实践补最小版。
- **/compact 与 auto memory**：上下文满时压缩（第 3 篇讲过机制）；跨会话记忆有大小上限（第 2 篇核验过 200 行/25KB 量级的文档表述）。对位：我们的滚动摘要（第 11 篇）与长期记忆槽。
- **事件流与流式 UI**：每个工具调用、每段模型输出都是事件，UI 流式渲染。动机：长任务的**可见性**——用户要看到「它在干活」而不是盯着一动不动的终端（第 19 篇长任务问题的地基）。

### 2.2 Cursor：应用层形态决定语义

Cursor 把 agent 内嵌进编辑器（官方站点行为描述，细节以官网为准）。有意思的不是它多了什么，而是**编辑器形态改变了 agent 的语义**：文件修改以 diff 呈现、用户逐块 Accept/Reject（写操作的确认粒度从「整个工具调用」细化到「每一处改动」）；终端命令单独确认；代码库检索是工具而非人肉贴上下文。对位我们：应用层差异会反过来重塑 harness 的权限粒度——第 20 篇部署形态（CLI vs Web）会再遇到这条。

### 2.3 OpenHands：sandbox 优先的世界模型

OpenHands（开源，All-Hands-AI/OpenHands，规格清单核验存在）的设计公开文档把「代码执行」放在世界观中心：agent 的主要行动方式是写代码并在**沙箱**里执行，而不是逐个调用细粒度工具。动机：代码是万能胶水——一个「执行代码」工具的表达力 ≈ 无数个专用工具，代价是必须解决沙箱隔离（文件系统/网络/进程的边界）。对位我们：第 10 篇工具谱系里「代码执行（沙箱）」一档的风险等级最高（R3），OpenHands 是把这一档做成主路径的极端样本——安全边界与表达力的一体两面。

### 2.4 OpenClaw：harness 即插件

OpenClaw（开源，第 3 篇核验过其文档站）有两个对我们最有用的公开设计：

- **「harness 即插件」**（其 README 原话，第 3 篇引用过）：把 harness 本身做成可替换、可扩展的插件体系，而不是固化的产品。动机：不同任务形态（CLI 工具、聊天助手、自动化流水线）需要不同的 harness 配置，但共享同一套 loop 骨架——分层解耦的极致版。
- **loop 的六环节拆法**（intake → context assembly → model inference → tool execution → streaming → persistence，第 3 篇三镜头对照表引用过）：比我们的五阶段多了 streaming 与 persistence 两个**显式环节**——因为我们把可观测和持久化做成了「旁挂」（审计、记忆），它们把这两件事升格为 loop 的一等环节。另有其 loop detection 文档（工具级防死循环，第 3 篇已引）——比我们的最大迭代数熔断更细。

## 3｜对照我们的成品：已实现 / 缺失 / 层级化决策

### 3.1 对照总表

| 构件 | 成熟系统的做法 | 我们的位置（篇） | 状态 |
|---|---|---|---|
| agent loop | 各家五/六环节 | run_agent（3、7） | ✅ 已有 |
| 工具注册 | 注册件 + MCP 标准化 | 注册件同形（8、14） | ✅ 已有 |
| 上下文管理 | 裁剪/摘要/compact | 滚动摘要（3、11） | ✅ 已有 |
| 权限 | 闸门 + 白名单三档 | 闸门两档（9） | 🟡 有起步版，白名单缺（19 篇） |
| 审计/事件流 | 结构化事件流 | 审计 JSONL（9、14） | 🟡 有事实记录，无统一事件流（18 篇） |
| 会话 checkpoint | JSONL 落盘 + 恢复 | — | ❌ 缺（**本篇实践补**） |
| hooks | 生命周期可编程挂载 | guard 是特例 | 🟡 特例已有，泛化缺（**本篇实践补**） |
| subagent | 独立上下文的子代理 | — | ❌ 缺（**17 篇落地**） |
| 沙箱 | 执行隔离 | — | ❌ 不做（第 10 篇已声明边界） |

结论很提气：九个构件里五个已自然实现，两个本篇实践补上，subagent 留给下一篇，沙箱明确不做——**我们没有走弯路，只是还没走完**。

### 3.2 hierarchical vs flat：什么时候需要层级

扁平（flat）：一个 loop 装所有工具——我们的现状，M2/M3 一直这么跑。层级（hierarchical）：主 agent 派活给 subagent，各有一套上下文——Claude Code 的形态。

决策信号（第 17 篇展开，这里给判定）：

- 任务内步骤的中间结果**对后续无价值**（查十个文件只为找一个事实）→ 层级，中间日志留在子上下文里烧完就扔；
- 工具结果**大而脏**（搜索、日志、代码库扫描）→ 层级，主上下文只收结论；
- 步骤间强耦合、每步输出都是下一步的输入 → 扁平，拆开反而要在层间搬运上下文；
- 只有一个任务形态、token 预算宽裕 → 扁平，层级的调度成本（派发/汇总的额外轮次）是纯开销。

一句话：**层级是上下文预算的压力反应，不是架构审美**。

## 4｜HITL 人机回环：确认与审批的落点

第 9 篇的闸门是 HITL 最小形态。四个系统的做法给全了光谱：

| 风险级 | 操作类型 | 我们（第 9 篇） | Claude Code | Cursor | 落点趋势 |
|---|---|---|---|---|---|
| 高 | 删除/付费/出站发送 | 未覆盖（后台只有覆盖式分配） | 权限提示，可永久放行 | 逐块 diff 审查 | 分级审批流（19 篇） |
| 中 | 写操作 | y/N 闸门（fail-closed） | 询问 + 白名单档位 | 命令确认 | 白名单降频（19 篇） |
| 低 | 只读 | 不打扰 | 默认允许 | 默认允许 | 一致 |

三个观察：①**粒度可以更细**（Cursor 的 diff 级）也可以**更粗**（批量预授权一段任务）；②**档位是体验问题**——每次都问会把用户训练成无脑 y（第 9 篇警报疲劳），永不问等于没有闸门，白名单是中间态；③**审批流是组织形态**——「agent 提出申请 → 人审批 → agent 继续」把同步 y/N 变成异步工单，P7·19 展开。本系列到 19 篇前的形态保持终端 y/N。

## 5｜动手实践：指认 subagent + 补齐 hooks 与 checkpoint

### 实践①：指认 Claude Code 的 subagent 机制（零代码）

1. 在任意项目建 `.claude/agents/` 目录，写一个最小子代理定义（frontmatter 字段以官方文档为准）：

```markdown
---
name: repo-scout
description: 只读侦察仓库结构与关键文件，回答事实性问题
tools: Read, Grep, Glob
---

你是仓库侦察员。只做只读检索，返回简明事实清单，不做修改。
```

2. 让主 agent 派一个任务给它（比如「用 repo-scout 查这个项目用了哪些依赖管理工具」）；
3. 观察：主对话里**只出现派发与最终报告**，中间的检索过程留在子代理自己的上下文里；
4. 把观察到的事件流画下来（这就是 subagent 协作的时序）：

```
 主 agent 上下文                          subagent 上下文（独立窗口）
 ─────────────                          ─────────────────────
 [1] 收到用户任务
 [2] 判定：交给 repo-scout
 [3] 派发任务 ──────────────────────→  [4] 收到任务，装 system prompt
                                        [5] 独立 loop：调 Read/Grep...
                                        [6] （过程日志留在这里，烧完即弃）
 [7] 收到最终报告 ←──────────────────  [8] 汇总结论，结束
 [9] 基于报告答复用户
        ↑ 主上下文只花了 [1][2][3][7][9] 的 token——[5][6] 的账记在子窗口
```

### 实践②：给我们的 agent 补「工具钩子 + checkpoint 可恢复」

两块缺失一次补齐，改动刻意保持最小（新增 `hooks.py` 一个文件 + `agent.py` 一处插入），完整代码如下。**py_compile 已过；中断重启的行为验证步骤在代码后，预期输出为结构示意 `<未实测，以官方文档为准>`**。

`hooks.py`（新增，工具生命周期钩子的最小泛化——guard 是它的特例）：

```python
# hooks.py — 第 16 篇：工具生命周期钩子（最小泛化版）
# 对位 Claude Code hooks / OpenClaw 插件体系的「可编程干预点」：
#   before(name, args) -> None（放行）或 str（拦截，返回值是回喂文本）
#   after(name, args, result) -> str（可改写结果，如脱敏/截断）
# 第 9 篇的 guard.confirm 是一个写死的 before-hook；本文件把它泛化成注册式，
# 且与 guard 并存：guard 管确认策略，hooks 管观测与改写（单一职责）。
# 用法（agent.py 已接入）：
#   import hooks
#   hooks.register_before(lambda n, a: f"拦截：{n} 被策略禁止")   # 拦截一切（示例）
#   hooks.register_after(lambda n, a, r: r[:200])                 # 结果截断（示例）
_BEFORE, _AFTER = [], []


def register_before(fn):
    """fn(name, args) -> None 放行 / str 拦截（返回值回喂模型）。"""
    _BEFORE.append(fn)


def register_after(fn):
    """fn(name, args, result) -> str，返回改写后的结果文本。"""
    _AFTER.append(fn)


def run_before(name: str, args: dict):
    """逐个执行 before 钩子；任何一个返回 str 即拦截（fail-closed：钩子异常也拦截）。"""
    for fn in _BEFORE:
        try:
            verdict = fn(name, args)
        except Exception as e:   # 钩子自身出错不能放行——与 guard 同一红线
            return f"错误：before 钩子异常（{type(e).__name__}），本次调用被拦截。"
        if verdict:
            return verdict
    return None


def run_after(name: str, args: dict, result: str) -> str:
    """逐个执行 after 钩子（观测/改写）；钩子异常不改写、不拦截（观测不能拖垮执行）。"""
    for fn in _AFTER:
        try:
            result = fn(name, args, result)
        except Exception:
            pass   # 明确不影响的场景才允许吞错：after 钩子失败不影响已完成的工具调用
    return result
```

`agent.py` 的两处插入（其余零改动；`--resume` 与 checkpoint 见下）：

```python
# ① execute_tool 内、guard 闸门之后、执行之前：
    verdict = hooks.run_before(name, args)            # 第 16 篇：通用 before 钩子
    if verdict:
        audit.log_tool_call(name, args, '{"error": true, "hook_denied": true}')
        return verdict
# ... 原有执行逻辑 ...
    result_str = hooks.run_after(name, args, result_str)   # 第 16 篇：通用 after 钩子
    audit.log_tool_call(name, args, result_str)            # （原审计行不变）

# ② run_agent 新增 checkpoint 参数（每轮结束落盘，进程被杀也能从最近一轮恢复）：
def run_agent(client, user_input, think=False, messages=None,
              checkpoint_file: str | None = None) -> str:
    ...
    for i in range(1, config.MAX_ITERATIONS + 1):
        ...
        for call in msg.tool_calls:
            ...（原逻辑）
        if checkpoint_file:                            # 第 16 篇：会话 checkpoint
            with open(checkpoint_file, "w", encoding="utf-8") as f:
                json.dump(messages, f, ensure_ascii=False)   # 每轮全量快照（教学版）
```

`main()` 相应加两个参数并接线（约 10 行，要点如下；完整版见磁盘 `agent.py`）：

```python
    parser.add_argument("--checkpoint", metavar="FILE",
                        help="每轮结束把会话消息快照落盘（第 16 篇，可恢复）")
    parser.add_argument("--resume", metavar="FILE",
                        help="从 checkpoint 文件恢复会话继续跑（第 16 篇）")
    ...
    if args.resume:                       # 恢复模式：读回快照，从最近一轮继续
        with open(args.resume, encoding="utf-8") as f:
            messages = json.load(f)
        print(f"[checkpoint] 已恢复 {len(messages)} 条消息，继续任务")
        print(f"[助手] {run_safely(client, '（继续之前的任务）', think=args.think, messages=messages)}")
    elif args.question:                   # 原路径 + 透传 checkpoint
        ...（原逻辑，run_safely 增加 checkpoint_file=args.checkpoint）
```

中断重启验证（行为 `<未实测>`，步骤按可复现写）：

```bash
# 终端 1 起后台；终端 2 跑一个多轮任务，中途 Ctrl+C 打断
python agent.py --checkpoint cp.json "把张三提升为管理员"
#   Ctrl+C 打断在第 2-3 轮之间
python agent.py --resume cp.json        # 从最近一轮的消息快照继续，不重跑已完成的工具调用
```

预期：resume 后的 loop 日志从「第 N 轮 | 发送 M 条消息」继续（M = 快照里的消息数），已执行过的 login/查询不会重复计费——**checkpoint 的价值就是让「打断」从事故降级为操作**。

### 自检清单

1. 实践①：主对话里确实看不到子代理的中间检索过程（事件流图的 [5][6] 在子窗口）；
2. 实践②：`python -m py_compile hooks.py agent.py` 通过；
3. 注册一个 before 钩子拦截 get_users，跑一句话任务，模型收到拦截文本后能改道（或如实告知）；
4. checkpoint 文件能被 `--resume` 读回，消息数与打断时一致；
5. 回归：p9/p10/p11/p12/p14 全绿（hooks 与 checkpoint 是纯增量，不改既有路径）；
6. 对照 §3.1 总表：九个构件里你自己的 agent 已有七个（含本篇补的两个）。

## 本篇结论

1. 四个系统共享三层骨架（模型适配 / harness / 应用），我们的六件套铺在 harness 层且次序一致——架构没走弯路，缺的是 subagent、事件流、白名单三块，各有归属篇。
2. 每个成熟设计的动机都能还原成我们经历过的问题：subagent=上下文预算（11 篇）、hooks=策略代码化（9 篇）、checkpoint=打断从事故变操作（本篇实践）、白名单=警报疲劳的中间态（19 篇）。
3. 层级化不是审美是压力反应：中间结果无价值/工具结果大而脏时上 subagent，强耦合任务保持扁平。下一篇把 subagent 真的落到我们的助手（里程碑 M5）。

## 参考资料

1. [Claude Code 官方文档（code.claude.com）](https://code.claude.com/) — agentic loop / agentic harness / subagent / hooks / checkpoint / 权限。第 2、3 篇已核验该站多页（docs.anthropic.com 301 跳转至此）；subagent frontmatter 字段与 hooks 挂载点以当前文档为准。
2. [OpenHands（All-Hands-AI/OpenHands）](https://github.com/All-Hands-AI/OpenHands) — sandbox 优先的代码 agent，事件流架构。仓库存在性 2026-10-15 核验（规格清单）；设计表述以仓库当前文档为准。
3. [OpenClaw（openclaw/openclaw）](https://github.com/openclaw/openclaw) — 「harness 即插件」、agent loop 六环节、loop detection。第 3 篇核验其文档站（docs.openclaw.ai）并引用原文。
4. [Cursor](https://www.cursor.com/) — 编辑器形态 code agent。行为描述基于官方站点（规格清单 ⚠️ 项），细节以官网为准。
5. [第 3 篇：Agent 的心脏](../part2-亲手写出第一个Agent/03-agent-loop与上下文.md) — harness 六件套与三镜头对照表（本篇 §1、§2.4 的上游）。
6. [第 11 篇：记忆设计](../part4-记忆与规划/11-记忆设计.md) — token 预算与滚动摘要（subagent 动机的上游）。
7. [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 多 agent 编排的适用边界（第 1 篇已核验；第 17 篇展开引用）。

> 核验说明：本篇对四个系统的描述，凡标「第 N 篇核验」者为系列已实测/已抓取的官方原文；Cursor/OpenHands 的行为层描述基于其官方公开站点与仓库（规格清单核验存在），细节处已标 `<以官方…为准>`；实践②的代码 py_compile 通过、行为未实测已标注。
