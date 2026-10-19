# 第 5 篇：2026 年 Python 选型地图
### 手写 vs 厂商 SDK vs 框架 · "LangChain 呢" · 三段式结论

> 知识地图【E1/D1】· Part 2
> 读完你能理解：2026 年构建 agent 的几条主流路线各自什么定位；"LangChain 呢"的正面
> 答案是什么；以及本系列为什么选"手写直连起步、需要时再封装"的三段式路线。

## 本篇要回答的问题

1. 2026 年 Python 构建 agent 有几条路？各自什么**定位**？
2. **"LangChain 呢？"**——为什么 star 最多、却不是构建 agent 的主流选择？
3. **决策表**怎么选？本系列的三段式结论是什么？各框架怎么接 DeepSeek？
4. 各框架 **20 行内最小 agent** 长什么样？"从 import 到能跑"差几步？
5. 还会撞见的 **LlamaIndex / Semantic Kernel** 是什么定位？为什么不进主线？

> 本篇动手实践会让你**先跑通一个框架最小 agent 而不求甚解**——它"为什么能跑、底层是什么"
> 是第 6-7 篇的活。先把"市面上轮子长啥样"的实感拿到手。

---

## 1｜2026 年构建 agent 的五条路线

别被"框架大战"带偏——2026 年 Python 构建 agent，其实就五条路线，两条一组的关系很清晰：

| 路线 | 代表 | 一句话定位 | 你能拿到什么 |
|---|---|---|---|
| **手写直连** | openai SDK + 自己写 loop | 模型 API + 五阶段循环自己拼 | 最高可控性、看得懂每一行（第 7 篇成品） |
| **轻封装** | pydantic-ai | 类型安全、以 Python 类型为中心的最小封装 | 手写的百分之八十好处，省一半样板代码 |
| **厂商 SDK** | openai-agents / claude-agent-sdk | 官方把 loop + 工具 + guardrails 打包好 | 快，生态内开箱即用 |
| **图编排** | langgraph | 把 agent 拆成"节点 + 边"的图，复杂分支显式画出来 | 应对复杂多 agent / 循环控制 |
| （老牌聚合框架） | langchain 1.x | 曾经的生态整合者，链式 API 已退居二线 | 读它的生态库，别拿它直接搭 agent（见下节） |

两个坐标系看懂这张表：

- **上手成本**：手写直连 > langgraph > pydantic-ai ≈ openai-agents ≈ claude-agent-sdk（越往下越省事）；
- **可控性/可见性**：手写直连 > pydantic-ai ≈ langgraph > 厂商 SDK（越往上越透明）。

**没有全能冠军**——五条路线是"帮你挡复杂度"和"给你可见性"之间的权衡。选型的关键是
**你处在学习的哪个阶段**（要弄懂 → 手写；要快速出活 → 封装），而不是"哪个最火"。

---

## 2｜"LangChain 呢？"——star 最多，为什么不是主流选择

这是每个学 agent 的人都会问的第一句话，正面回答它：

**2026-10 实测数据表**（star 数 + PyPI 版本，GitHub/PyPI 实测）：

| 项目 | GitHub star | PyPI 最新 |
|---|---|---|
| langchain | 14.8w | 1.4.6 |
| langgraph | 4.3w | 1.2.15 |
| openai-agents | 3.0w | 0.24.0 |
| pydantic-ai | 2.0w | 2.56（pydantic-ai-slim）|
| claude-agent-sdk | 8k | 0.2.167 |

**LangChain 的 star 数遥遥领先，但"star 多 ≠ 主流构建方式"。** 原因有三：

1. **历史主流，不是当下主流**：2023-2024 年 LangChain 是"接大模型搞应用"的代名词，
   但它主推的**链式 API（LCEL 那套）已经退出"直接构建 agent"的路径**——链式是"把固定的
   步骤连成管线"，和 agent"让模型动态决定下一步"是两套思路。
2. **官方自己转身**：LangChain 官方在 1.x 时代把构建 agent 的主路径转向 **langgraph**，
   说明连作者都承认"链式不是搭 agent 的正解"。
3. **数据表直接说话**：一个 star 最多、一个官方主推，star 差 3 倍——如果 star 等于
   该用哪个，官方根本不需要另起 langgraph。**库大 ≠ 主流，生态大 ≠ 推荐主路径。**

**那 langchain 还有用吗？** 有，但定位变了：它的**周边生态**（文档加载器、向量库封装、
工具集成）依然丰富，是"工具收集器"；只是**不要再拿它的链式 API 去搭你的 agent 主骨架**。

**本系列的处理**：教程正文一律不教链式抽象；langchain 只作为"生态对照"出现，langgraph
作为"复杂多 agent 图编排"的候选在第 6 篇（P6·17）再登场——那是它真正该出现的场景。

---

## 3｜决策表 + 三段式结论

五个候选 × 00 指定的五个维度（上手成本 / 可控性 / 生态 / MCP 支持 / 长期维护）：

| | 上手成本 | 可控性 | 生态 | MCP 支持 | 长期维护 |
|---|---|---|---|---|---|
| **手写直连** | 中（loop 自己写）| 最高 | openai 系 SDK | 自建（P5）| 自己维护 |
| **pydantic-ai** | 低 | 高 | 较新但活跃 | 支持（以 docs 为准）| 活跃 |
| **openai-agents** | 低 | 中 | OpenAI 系，厂商强 | 内置 | OpenAI 官方 |
| **claude-agent-sdk** | 低 | 中（harness 内置略黑盒）| Anthropic 系 | 内置 | Anthropic 官方 |
| **langgraph** | 高 | 高（表现力换控制力）| LangChain 系 | 支持 | langchain-ai |

> MCP 掌握程度随版本演进，表中"支持"仅指官方存在对应能力，具体用官方文档为准。

**三段式结论（本系列锁定的路线）**：

1. **学习阶段：手写直连 loop。** 理由：可控性最高——loop 五阶段、工具调用、上下文管理
   全部看得见，日后用任何框架都知其所以然。这就是第 3 篇骨架 + 第 7 篇成品在干的事。
2. **需要封装时：优先 pydantic-ai / openai-agents。** 理由：pydantic-ai 类型安全、轻、
   保留手写的大部分好处；openai-agents 内置 MCP、guardrails、handoffs，且一行切换
   OpenAI 兼容供应商（DeepSeek 正好是）。
3. **多 agent / 复杂图编排才上 langgraph。** Part 6（P6·16-17）给模式落地；不到那一步
   **不必提前引入**——它是"表达力换控制力"，复杂度守恒。

**claude-agent-sdk 的定位要单独说**：它的模型绑定 Claude（不适配 DeepSeek），但它的
**harness 概念最完整**（loop、权限、checkpoint、hooks 全内置），所以它在本系列只做
**第 6 篇拆解 Claude Code 时的概念对照**，不当默认方案。

**各框架对 DeepSeek（OpenAI 兼容）的适配方式**：

| 框架 | 怎么接 DeepSeek（以各官方 docs 为准）|
|---|---|
| 手写直连 | `base_url="https://api.deepseek.com"` + `model="deepseek-flash"`（第 4 篇已见）|
| openai-agents | `OPENAI_BASE_URL` / `OPENAI_API_KEY` 环境变量指向 DeepSeek，模型名用 `deepseek-flash` |
| pydantic-ai | OpenAI 兼容 provider 配置 `base_url` + `api_key` |
| claude-agent-sdk | 不适配（模型绑定 Claude），仅拆解参考 |
| langgraph | 用内置的 OpenAI 兼容 LLM 类传 `base_url` |

> 关于 claude-agent-sdk 一行补充：DeepSeek 官方另有 **Anthropic 格式端点**
> （`base_url` 换 `https://api.deepseek.com/anthropic`）——那是让 `anthropic` SDK
> 直连 DeepSeek 用的（第 6 篇细讲），与 claude-agent-sdk 无关：后者的 harness 与
> 模型/生态绑定 Claude，本系列只做概念对照。

---

## 顺带：还会撞见的 LlamaIndex / Semantic Kernel

选型地图之外，你在外部资料里大概率还会撞见两个"大名字"。给一句话定位，避免混淆：

| 框架 | 一句话定位 | 为什么不进主线 |
|---|---|---|
| **LlamaIndex** | RAG/数据检索平台（Python，[GitHub 5.3w star](https://github.com/run-llama/llama_index)，PyPI 0.14.25）——主场是把"文档变成可检索的知识库"，**不是搭 agent 循环** | 本系列主线场景是"调用后台接口"，不涉及文档检索；第 11 篇的"RAG 分界线"处会再提一句它代表的正路 |
| **Semantic Kernel** | 微软系 AI SDK（[GitHub 2.9w star](https://github.com/microsoft/semantic-kernel)，PyPI 1.45.0）——.NET/C# 优先、有 Python 版，偏企业微软栈 | 语言/生态与本主线（Python + DeepSeek）弱相关；真要用时按你的团队技术栈另行自学 |

（以上 star 与版本号为 2026-10-15 在线实测核验。）记住一句话即可区分：**LlamaIndex 是
"检索工具厂"、Semantic Kernel 是"微软企业栈"——都不碰"agent 循环"这块主线**，所以它们
在本系列只出现这一次（再加第 11 篇的 LlamaIndex 一句）。

---

## 4｜20 行内最小 agent 并排：看形态，别背 API

> 下面五个块是**形态示意**（对比"几百行 vs 十几行"的结构差异），**均未在本环境实测**，
> API 以各自官方 docs 为准、随版本演化——读的时候别抄代码，读"它用几个概念就转起来了"。

**① 手写直连（openai SDK + 自己 loop）**——概念：client、messages、model

```python
from openai import OpenAI
import os
from dotenv import load_dotenv
load_dotenv()
client = OpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                base_url="https://api.deepseek.com")

def agent(msg):
    r = client.chat.completions.create(
        model="deepseek-flash",
        messages=[{"role": "user", "content": msg}])
    return r.choices[0].message.content

print(agent("一句话介绍你自己"))
```

**② openai-agents（厂商 SDK）**——概念：Agent、Runner；改环境变量即切供应商

```python
import os
from dotenv import load_dotenv
load_dotenv()
os.environ["OPENAI_API_KEY"] = os.environ["DEEPSEEK_API_KEY"]
os.environ["OPENAI_BASE_URL"] = "https://api.deepseek.com"   # 以官方 docs 为准

from agents import Agent, Runner
assistant = Agent(name="assistant", instructions="你是简洁的助手。")
result = Runner.run_sync(assistant, "一句话介绍你自己")
print(result.final_output)
```

**③ claude-agent-sdk（厂商 SDK，Harness 最全）**——概念：ClaudeAgent 一个类管 loop+工具

```python
# 示意形态：真实 API 以官方 docs 为准
import os
from dotenv import load_dotenv
load_dotenv()  # ANTHROPIC_API_KEY
from claude_agent_sdk import ClaudeAgent
agent = ClaudeAgent(instructions="你是简洁的助手。")
result = agent.run("一句话介绍你自己")
print(result)
```

**④ pydantic-ai（类型安全轻封装）**——概念：Agent + 类型注解出 schema

```python
# 示意形态：供应商配置以官方 docs 为准
from pydantic_ai import Agent
agent = Agent(
    "openai:deepseek-flash",        # 具体 base_url 配置见官方 docs
    system_prompt="你是简洁的助手。",
)
print(agent.run_sync("一句话介绍你自己").output)
```

**⑤ langgraph（图编排）**——概念：StateGraph、node、edge、compile

```python
from typing import TypedDict
from langgraph.graph import StateGraph

class State(TypedDict):
    question: str
    answer: str

def echo_node(state: State):
    return {"answer": f"回声：{state['question']}"}

g = StateGraph(State)
g.add_node("echo", echo_node)
g.set_entry_point("echo")
g.set_finish_point("echo")
app = g.compile()
print(app.invoke({"question": "你好"}))
```

**"从 import 到能跑"差几步**（教学估算，以各自官方 quickstart 为准）：

| 方案 | 最少步骤（装依赖 → 跑通）| 首次跑通前你要理解的概念 |
|---|---|---|
| 手写直连 | 2（pip install openai + 已备的 .env）| client / messages / model |
| openai-agents | 3（装 + 环境变量 + 写）| Agent / Runner |
| claude-agent-sdk | 2（装 + key）| ClaudeAgent |
| pydantic-ai | 3（装 + 供应商配置 + 写）| Agent / 类型 |
| langgraph | 4+（装 + 建图 + 节点边 + 编译）| StateGraph / node / edge |

**读法**：大家"从 import 到能跑"都差不多近——**差的不在"跑起来"的速度，而在"跑起来了，
你搞不搞得懂它在干嘛"**。手写直连最慢的是"你自己写的"，也正因为如此，它最透明。

---

## 动手实践：装一个框架，跑一个最小 agent（先跑通不求甚解）

**目标**：把第 3 节的三段式结论里第 2 条"需要封装时优先 openai-agents"落一次地——装、
写八行、跑通，**不要求你现在看懂它内部**（那是第 6-7 篇的拆解工作）。

环境已在第 1 篇备好（Python 3.10+、`.env` 里有 `DEEPSEEK_API_KEY`）。装 `openai-agents`
（国内源）并运行：

```bash
source .venv/bin/activate
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple openai-agents
python demo_framework_agent.py
```

新建 `demo_framework_agent.py`（约 12 行，跑通即可，看完不求甚解）：

```python
"""第 5 篇动手实践：用 openai-agents 跑一个最小 agent（先跑通，不求甚解）。

<本环境未实测，供应商 env 配置方式以 openai-agents 官方 docs 为准>
"""
import os
from dotenv import load_dotenv
load_dotenv()

# 切到 DeepSeek（OpenAI 兼容）：只改两个环境变量
os.environ["OPENAI_API_KEY"] = os.environ["DEEPSEEK_API_KEY"]
os.environ["OPENAI_BASE_URL"] = "https://api.deepseek.com"

from agents import Agent, Runner
assistant = Agent(name="assistant", instructions="你是简洁的助手。")
result = Runner.run_sync(assistant, "用一句话介绍你自己，再报一下今天的日期。")
print(result.final_output)
```

预期输出（以当次运行为准）：

```text
我是你的 AI 助手，负责简洁地回答问题。今天是 2026 年 10 月 15 日。（示例输出，具体以模型为准）
```

**跑通之后的三句话备忘**（现在看不懂没关系）：

1. 这 12 行里"Agent + Runner"背后就是第 3 篇的 loop——**框架替你写好了**，你马上会在
   第 7 篇亲手拆出来；
2. "instructions"就是你之前写的 system prompt 的显式入口；
3. 如果跑挂了，九成是环境变量或国内网络——回头检查 `.env` 和上面三行 `os.environ`，
   别怀疑自己。

---

## 本篇结论

1. **五条路线**是"挡复杂度 vs 给可见性"的权衡，没有全能冠军；"star 最多 ≠ 主流"，LangChain
   链式 API 已退居二线、官方转向 langgraph。
2. **三段式锁定**：学习用手写直连 → 封装优先 pydantic-ai / openai-agents → 复杂多 agent
   才 langgraph；claude-agent-sdk 只做第 6 篇拆解参考。
3. **DeepSeek 适配**：全走 OpenAI 兼容，"改 base_url + 模型名"一处到底；三套框架的
   "从 import 到能跑"其实差不多，差别在跑起来后你懂不懂它。

---

**下一篇预告**：第 6 篇《调用 LLM 的三种姿势》——直连 REST / 官方 SDK / agent SDK
三条到达模型的路径、消息结构与流式，配两个可运行脚本次接触模型。本篇先跑通的那 12 行，
下一篇开始拆。

**等我回复「继续」。**