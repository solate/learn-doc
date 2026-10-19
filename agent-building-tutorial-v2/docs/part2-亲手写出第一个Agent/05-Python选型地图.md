# 第 5 篇：harness 框架怎么选（读完就有结论）

> **知识地图【E1/D1】· Part 2**
> 面向需求：写一个现代 harness 用来维护既有系统（比如调用你现有的后台 API），要**能接不同大模型、能用上多 agent / 多 skill / 多种运行模式、功能全面、封装好、能上生产**。
> 本篇把选型直接给出结论：选什么、为什么选它、为什么不选别的；不铺全景、不列没用的对比表。
> 版本号与事实以 2026-10-19 核验为准（PyPI / GitHub 官方源）。

## 本篇要回答的问题

1. 你要的 "harness" 和"随便一个 agent 库"、和 LangGraph 到底差在哪？
2. 成熟方案（Claude Code / Codex / OpenHands / OpenClaw / DeepSeek Harness）的 harness 都由哪些部分组成？我选的框架覆盖全了吗？
3. 在 deepagents / Microsoft Agent Framework / DeepSeek Harness 之间，按**多模型 / 多 agent / 多 skill / 多模式**四个能力轴怎么打分？**首选为什么是 deepagents？**
4. 为什么不选 LangGraph / claude-agent-sdk / openai-agents / Google ADK / AWS Strands / CrewAI / 手写？
5. 动手：首选方案 + DeepSeek 的最小跑通长什么样？

## 1｜先对齐：harness ≠ 随便一个库 ≠ LangGraph

第 3 篇定义过 **harness（agent loop 的工程外壳）**：循环调度、工具注册、事件流、权限闸门、持久化、可观测这六件套。现代 agent 的公式是 **Model + Harness = Agent**——你缺的不是模型（第 4 篇已选好），是 harness。

三条硬话：

1. **LangGraph 是图运行时，不是 harness**。它只给你"把控制流画成状态图"的执行引擎；工具、权限、会话、上下文管理这些 harness 件它一件都不打包，全要你自己组装。`deepagents` 就是 LangChain 家族在它上面**打包好的 harness**——这就是为什么你该在 deepagents / 微软 / DeepSeek 里选，而不是 LangGraph。
2. **harness 的"装好就能用"程度是你的第一筛子**。给的零件越多、要你拼的越少，封装越好。
3. **你的硬标准**（读后面打分表之前先对齐这六条）：
   - **多模型无关**：DeepSeek / OpenAI / Claude / 本地 Ollama 切换不是重写，只是换一个配置点；
   - **多 agent**：能拆子 agent、委派给隔离上下文的子任务，而不是所有事挤在一个循环里；
   - **多 skill**：能把复用能力（读文档、查后台、写审计……）打包成可命名的技能，按需加载；
   - **多种运行模式**：能适配不同场景——单轮问答 / 长任务执行 / 只读模式 / 写操作审批模式 / 极简跑基准，不要一套 loop 打天下；
   - **功能全面 + 封装好**：工具、上下文管理、记忆、权限/HITL、MCP、可观测都有，且一个入口建出完整 agent；
   - **生产可维护**：长任务能断点续、危险操作能审批、每次调用能 trace。

## 2｜成熟方案是怎么做的：harness 八件套 + 四个能力轴

Claude Code、Codex、Cursor、OpenHands、OpenClaw、DeepSeek Harness 形态各不相同，但**收敛到同一张"harness 件清单"**。你要选的框架至少得齐这八件：

| # | harness 件 | deepagents | MAF | DeepSeek Harness |
|---|---|---|---|---|
| 1 | loop + 终止条件（有界不跑飞） | ✅ | ✅ | ✅ |
| 2 | 工具注册（签名→Schema，不手写 JSON） | ✅ | ✅ | ✅ |
| 3 | 权限 / HITL 审批 | ✅ | ✅ | ✅ |
| 4 | 会话 / 持久化 / 断点恢复 | ✅（LangGraph checkpointer） | ✅ | ✅ |
| 5 | 上下文管理（压缩/摘要） | ✅ | ✅ | ✅ |
| 6 | 子 agent | ✅ | ✅ | ✅ |
| 7 | MCP | ✅ | ✅ | ✅ |
| 8 | 可观测（trace/成本） | ✅（LangSmith） | ✅（OpenTelemetry） | ✅（遥测插件） |

再按你加的四条需求打分：

| 能力轴 | deepagents | Microsoft Agent Framework | DeepSeek Harness |
|---|---|---|---|
| **多模型** | ✅ 换 chat model 对象 1 处 | ✅ 多 provider（OpenAI/Anthropic/Google/Bedrock/Ollama + 兼容端点） | ✅ 多 provider（偏 DeepSeek 原生） |
| **多 agent** | ✅ `subagents=` 内置 | ✅✅ teams / handoff / group 全套流程 | ✅ 子 agent（完整 profile；极简 profile 会裁掉） |
| **多 skill** | ✅ `skills=` 内置 | ✅ Agent Skills 官方一等公民 | ✅ 技能/指令发现走插件 |
| **多种模式** | △ 不内置"模式"菜单，靠你组合（prompt + 工具集 + 子agent 变体） | △ declarative / process / hosted 等形态，偏企业运维 | ✅✅ **内置多种运行模式**（如标准 / PTC / 极简 / 创造，`<以官方文档为准>`） |

**读法**：八件套三家都齐；四种能力里 deepagents 靠"参数 + 你组合"补齐，MAF 靠"企业形态"补齐，**DeepSeek Harness 是在"多模式 + 插件化"上最极致的那个——但它付出的代价是 developer preview**。

## 3｜决策：选哪个

读这一张就够：

| 优先级 | 框架 | 为什么选 / 为什么不选 |
|---|---|---|
| ✅ **首选** | **deepagents**（LangChain 官方） | 开箱即用 harness：`create_deep_agent(...)` 一个函数，默认带文件系统、子 agent、上下文管理、持久记忆、HITL 审批、skills、任意 MCP server，底层 LangGraph（checkpoint / 流式 / 持久化复用），LangSmith 自带 trace/eval。**多模型**：换一个 chat model 对象即可（DeepSeek 用 `langchain-deepseek` 的 `ChatDeepSeek`）。**多 agent / 多 skill**：`subagents=`、`skills=` 都是命名参数。**模式**：不内置"模式"菜单，但模式本来就是你组合出来的（只读模式 = 只挂查询工具；计划模式 = 先输出计划再执行；审批模式 = `interrupt_on=`）。生态最大、中文资料多、和你在学的 LangGraph 一脉相承。 |
| ✅ **备选** | **Microsoft Agent Framework** | 场景换成**企业治理 / 多语言 / 强流程化**时上它：teams / handoff / group 多 agent 流程、中间件、checkpoint、HITL、Agent Skills、OpenTelemetry，Python / .NET / Go 一套 API；多 provider 原生，DeepSeek 走 `OpenAIChatCompletionClient` + `OPENAI_BASE_URL`。代价：概念重（workflow / middleware / process），不绑 Azure 时配置自理。 |
| ⚠️ **暂不选 / 预研** | **DeepSeek Harness** | 它在"多模式 + 插件化 + 完整 harness 件"上其实是三家最全的——**唯一拦路的是稳定性**：developer preview，官方原话 "THERE WILL BE COMPATIBILITY-BREAKING CHANGES"。**什么时候选它**：你主锁 DeepSeek、或等稳定版、或愿意为"官方完整运行时"吃 break 的坑（先 POC 再定）。此刻更适合当**架构参考**读（第 16 篇拆 harness 的素材）。 |
| ❌ 不选 | **LangGraph（裸框架）** | 图运行时，不是 harness——八件套全要自己拼。 |
| ❌ 不选 | **claude-agent-sdk** | 模型绑定 Claude，你的多模型需求直接排除它。 |
| ❌ 不选 | **openai-agents** | OpenAI 优先；接 DeepSeek 有一行行"适配税"（`AsyncOpenAI` + `OpenAIChatCompletionsModel` + 关 tracing），多模型不顺。 |
| ❌ 不选 | **Google ADK / AWS Strands** | 分别绑定 GCP、AWS；你不绑云，等于背一堆用不上的部署件。 |
| ❌ 不选 | **CrewAI** | 多 agent 角色编排好使，但不是"通用 harness"——角色/任务抽象偏业务剧本，模式/工具面不如 deepagents 直接。 |
| ❌ 不选 | **手写（生产）** | 学习期必写（第 6-7 篇，看懂 loop 每一行）；生产期手写是把八件套全部重复造一遍——那是拆成熟 agent（第 16 篇）才轮得到的工程量。 |

**一句话决策规则**：

- 要**立刻稳定上生产 + 生态大 + 封装一个函数** → **deepagents**；
- 要**企业治理 + 多语言 + 流程化多 agent** → **Microsoft Agent Framework**；
- 要**多模式开箱即用 + 主锁 DeepSeek + 能接受 preview（先 POC）** → **DeepSeek Harness**。

## 4｜最小跑通：deepagents + DeepSeek（首选方案）

```python
from deepagents import create_deep_agent
from langchain_deepseek import ChatDeepSeek

def get_time() -> str:
    """获取当前时间（演示用，返回固定值）"""
    return "2026-10-15 12:00:00"

agent = create_deep_agent(
    model=ChatDeepSeek(model="deepseek-flash"),   # 换模型就换这一处
    tools=[get_time],
    system_prompt="用一句话简洁回答",
)
result = agent.invoke({"messages": "现在几点了？"})
print(result["messages"][-1].content)
```

- 安装：`pip install deepagents langchain-deepseek`（Python 3.11+）；`DEEPSEEK_API_KEY` 走环境变量。
- 这 14 行背后是完整 harness：默认还带 `ls/read_file/write_file/edit_file/glob/grep/execute/task` 等工具、上下文管理、子 agent、checkpoint——对比第 7 篇你手写的 20 行循环，能看到"同一个 loop，deepagents 替你包好了哪些件"。
- 你的四个需求在 `create_deep_agent` 的命名参数上怎么展开（`<以官方文档为准>`）：
  - **多模型**：`model=` 换 `ChatOpenAI(base_url=...)` / 本地 Ollama 的 LangChain `init_chat_model`；
  - **多 agent**：`subagents=[...]` 给各子任务配隔离上下文（第 17 篇的主题）；
  - **多 skill**：`skills=[...]` 把复用能力命名加载（第 16 篇 Claude Code 就有这套）；
  - **多模式**：deepagents 不内置"模式"按钮——你自己用"不同 system prompt + 工具子集 + `interrupt_on`"组合出只读模式、审批模式、计划模式。想要"开箱即用的运行模式"，去看 DeepSeek Harness 的多种内置运行模式（如标准 / PTC / 极简 / 创造，`<以官方文档为准>`）。
- 生产化前把默认件收口：`permissions=` 收紧文件系统 → 后台工具走统一工具出口或 MCP server（Part 5）→ `interrupt_on=` 开写操作审批 → `checkpointer=` / LangSmith 接断点续跑与 trace。
- MAF 版最小写法见官方 quickstart（`Agent` + `OpenAIChatCompletionClient` + `OPENAI_BASE_URL`），这里不展开。

## 动手实践：把首选方案跑起来

目标：验证"一个函数装好一整套 harness"这一个判断，和手写版形成体感对比。

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install deepagents langchain-deepseek -i https://pypi.tuna.tsinghua.edu.cn/simple
export DEEPSEEK_API_KEY=sk-xxx        # key 只走环境变量，不进 git
# 保存上方 §4 代码为 agent_harness.py
python agent_harness.py
```

- 预期输出：一句回答（现在是 2026-10-15 12:00:00）。
- 对比练习（选做）：再用 openai SDK 手写 20 行跑同一任务（第 7 篇成品），写出"deepagents 替我写了哪几件"清单——这是把黑盒变透明的关键一步。
- 说明：真实 API 调用未逐个实测 `<未实测，以官方文档为准>`；安装 + 无 key 报错行为可在本机复测。

## 本篇结论

- 你的场景（现代 harness + 多模型 / 多 agent / 多 skill / 多模式 + 维护既有系统）**首选 deepagents**：八件套齐、多模型换一处、`subagents=` / `skills=` 开箱即有、模式用组合实现，LangGraph 底座 + LangSmith 观测，封装和生态都在第一档。
- **企业治理 / 多语言 / 流程化再上 Microsoft Agent Framework**；**DeepSeek Harness 的"多模式 + 插件化"最全，但 developer preview 拦路——先当架构参考 / POC，等稳定版再上**；LangGraph 裸框架、claude-agent-sdk、openai-agents、绑云的 ADK / Strands、CrewAI 都不是这个需求的首选。
- 学习仍先手写（第 6-7 篇），看懂 loop 之后再换生产 harness，避免黑盒。

## 参考资料

1. [Deep Agents 官方文档](https://docs.langchain.com/oss/python/deepagents/overview) + [仓库](https://github.com/langchain-ai/deepagents) — "batteries-included agent harness"、`create_deep_agent` 参数（tools / subagents / skills / permissions / interrupt_on / checkpointer / memory）均引自 README/源码，2026-10-19 核验；PyPI `deepagents 0.7.28`
2. [langchain-deepseek（DeepSeek 官方集成文档）](https://docs.langchain.com/oss/python/integrations/providers/deepseek) — PyPI `langchain-deepseek 1.1.1`（2026-10-19 核验，仓库为 langchain-ai/langchain 官方仓库的一部分）；`ChatDeepSeek` 默认 `api_base` 指向 DeepSeek 兼容端点
3. [LangGraph 官方文档](https://docs.langchain.com/oss/python/langgraph/overview) — 定位为图运行时，deepagents 的底座；PyPI `langgraph 1.2.16`
4. [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) + [官方文档](https://learn.microsoft.com/en-us/agent-framework/) — 多 provider、teams / handoff / group 流程、Agent Skills、中间件、checkpoint、HITL、OTel 与 `OPENAI_BASE_URL`/`OPENAI_MODEL` 引自 README/示例，2026-10-19 核验；PyPI `agent-framework 1.22.0`
5. [DeepSeek Harness 仓库](https://github.com/deepseek-ai/deepseek-harness) + [Python SDK](https://pypi.org/project/deepseek-harness-sdk/) — developer preview、"一切皆插件"、profile/bundle、运行模式与 `DeepSeekHarness(provider=...)` 引自官方 README/文档，2026-10-19 核验；npm `0.2.0-rc.2` / PyPI `0.2.1rc2`
6. PyPI JSON API（版本与 requires_python 核验）— `https://pypi.org/pypi/<包名>/json`，2026-10-19 当日查询
7. [DeepSeek 开放平台](https://platform.deepseek.com/)（现行模型清单与思考模式参数，考据见第 4 篇 §1.2）