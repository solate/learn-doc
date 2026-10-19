# 第 7 篇：手写一个「看得懂每行代码」的最小 agent

> **知识地图【A2/A3/B4】· Part 2** ⭐ 啊哈时刻
> 零件已备齐：第 3 篇给了 15 行循环骨架（[03-agent-loop与上下文](03-agent-loop与上下文.md)），第 4 篇填上了 `LLM()` 该填谁，第 6 篇备齐了消息四角色、`tool_call_id`、finish_reason、key 与超时（[06-调用LLM的三种姿势](06-调用LLM的三种姿势.md)）。本篇把零件装进骨架——桩函数换成真实 LLM + 两个演示工具，从今天起建立系列代码目录 `agent/`，此后每一篇都在它上面演进。逐行讲透四个要害：工具描述怎么"被模型看到"、参数怎么校验、"我调完了"怎么判定、四类必踩的坑怎么兜。Part 2 的里程碑 **M1** 在本篇动手实践收口。

## 本篇要回答的问题

1. 第 3 篇的 15 行骨架怎么补全成真 agent？工具描述（JSON Schema）是怎么"被模型看到"的？
2. 模型给出的参数怎么校验？幻觉调用、坏 JSON 空参、工具异常——每条错误路径怎么兜才不崩？
3. 模型怎么表达"我调完了"？终止判定为什么必须读结构化信号，循环里为什么必须回传 `reasoning_content`？
4. 结构化输出（JSON mode / `response_format`）是什么？它与 function calling 什么关系、什么时候用哪个？
5. 动手实践（**里程碑 M1**）：终端跑起属于你的最小 agent——预置问题 + 工具调用 + 循环可见日志 + 正常终止。

## 1｜组装：零件清单、配置点与工具层

读完本节你能理解：从骨架到成品差了哪几个文件、每个文件为什么长那样，以及工具描述如何变成 prompt——模型"看到"的工具和你写的函数根本是两个东西。

### 1.1 零件清单：五阶段 × 前五篇零件 × 本篇文件

第 3 篇动手实践的桩版本已经证明循环结构是对的，本篇只是把三处桩换成真家伙。逐阶段对号：

| 循环阶段（第 3 篇 §1.2） | 用到的零件（出处） | 本篇落点 |
|---|---|---|
| ① 获取输入 | system/user 消息角色（第 6 篇 §1.4） | `agent.py` 的 `run_agent` 开头 |
| ② 调 LLM | base_url/model/key、超时重试（第 6 篇 §2.3）、thinking 开关（第 4 篇 §1.3）、`tools` 参数 | `config.py` + `call_llm()` |
| ③ 解析/终止判定 | finish_reason 全集（第 6 篇 §1.3）、tool_calls 有无 | `run_agent` 的判定分支 |
| ④ 执行工具 | 工具注册表（第 3 篇 §2.3）、JSON Schema、参数校验 | `tools.py` + `execute_tool()` |
| ⑤ 追加回填 | 消息四快照、tool_call_id 配对（第 6 篇 §1.4）、reasoning_content 回传（第 4 篇 §1.5） | `to_assistant_dict()` + 回填循环 |
| 循环外保险 | 最大迭代数熔断、用户打断（第 3 篇 §4.2） | `config.MAX_ITERATIONS` + `main()` |

两个演示工具选 `get_time` 与 `calculate`：零依赖、零危险、结果肉眼可验证，且天然覆盖两种 schema 形态——无参工具（get_time）与多参带 enum 的工具（calculate）。第 8 篇才切后台 API 主场景，本篇不抢戏。

`agent/` 目录从这里开始随全系列演进（直连版 → MCP 版 → subagent 版 → 工程壳版），本篇的全部文件：

```
agent/
├── requirements.txt   # openai==3.31.0（Python 3.11+，第 5 篇系列约定）
├── .gitignore         # .venv/ .env __pycache__/
├── config.py          # 配置唯一入口：base_url / model / key / 超时 / 熔断上限
├── tools.py           # 两个演示工具：schema + 实现 + 最小校验
├── agent.py           # 总入口：agent loop 主循环（本篇主角）
└── json_mode_demo.py  # §4 的 JSON mode 演示（独立小脚本）
```

### 1.2 config.py：三个字符串 + 两个闸门

第 6 篇 §2.2 立过规矩：配置收敛到一个点。`config.py` 只有 27 行，但每行都对应前文的一条结论——`BASE_URL`/`MODEL` 是"换供应商改 2 个字符串"的那两个字符串，`TIMEOUT`/`MAX_RETRIES` 是第 6 篇要求显式收窄的 SDK 默认值，`MAX_ITERATIONS` 是第 3 篇四类终止条件里唯一不依赖模型自觉的那条。`api_key()` 在缺 key 时给出可操作的报错（第 5 篇实测过裸 `KeyError` 的体验，这里升级成人话）：

```python
# config.py — 全项目唯一的配置点
# 第 4 篇经验 1：模型名是易变接口，base_url / model / key 必须收敛到一处；
# 第 6 篇 §2.2 纪律：key 只走环境变量 / .env，不进 git。
import os

# OpenAI 兼容端点（第 6 篇 §1.2）。换供应商 = 只改这一个字符串（第 4 篇 §3.1）
BASE_URL = "https://api.deepseek.com"
# 现行模型名（第 4 篇 §1.2 考据：deepseek-chat / deepseek-reasoner 已于 2026-07-24 停用）
MODEL = "deepseek-flash"
# SDK 默认超时 10 分钟太宽（第 6 篇 §2.3）——agent loop 里单次调用收窄到 60 秒
TIMEOUT = 60.0
# SDK 默认就是 2，显式写出是让你知道它存在（第 6 篇 §2.3）
MAX_RETRIES = 2
# 终止条件二：最大迭代数熔断（第 3 篇 §4.2）——唯一不依赖模型自觉的保险
MAX_ITERATIONS = 10


def api_key() -> str:
    """从环境变量读 key；缺失时给出可操作的报错，而不是让 SDK 抛含糊的异常。"""
    key = os.environ.get("DEEPSEEK_API_KEY")
    if not key:
        raise SystemExit(
            "[配置错误] 未找到环境变量 DEEPSEEK_API_KEY。\n"
            "  1. 到 https://platform.deepseek.com 申请 API key；\n"
            "  2. export DEEPSEEK_API_KEY=sk-xxxxxxxx   # 只走环境变量/.env，不进 git"
        )
    return key
```

### 1.3 tools.py：schema 与实现为什么必须住在一起

第 3 篇 §2.3 给过工具注册件的定位："注册表同时是给模型的文档"。本篇把它落成代码，并加一条硬规矩：**schema（给模型看的说明书）与实现（真正干活的函数）放在同一个文件里成对维护**。原因到 §3 的踩坑表会看得很清楚——两者一旦不同步，模型立刻开始按旧描述幻觉调用。文件分三段：实现、schema 清单、最小校验器——第三段（校验器）的完整代码与逐条讲解在 §3.1。

```python
# tools.py — 两个演示工具：get_time / calculate（第 8 篇才切后台 API 主场景）
# 工具注册（第 3 篇 §2.3 六件套之 2）的最小形态：schema 与实现放在同一个文件里成对维护。
# 原因：工具描述是 prompt 的一部分——改了实现忘改描述，模型立刻按旧描述幻觉调用（§3 踩坑表）。
import datetime


# ---------- 工具实现：普通 Python 函数，agent.py 在阶段④ 按名字查表真正调用 ----------

def get_time() -> dict:
    """获取当前本地时间。"""
    return {"now": datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")}


def calculate(a: float, b: float, op: str) -> dict:
    """四则运算器。op 由 schema enum 约束为 + - * / 之一。
    b=0 且 op="/" 时抛 ZeroDivisionError——故意保留：它是 agent.py
    「工具异常兜底」路径（第 7 篇 §3 踩坑表第 4 行）的活教材。"""
    if op == "+":
        return {"result": a + b}
    if op == "-":
        return {"result": a - b}
    if op == "*":
        return {"result": a * b}
    return {"result": a / b}
```

```python
# ---------- 工具清单：随请求发给模型的 JSON Schema（tools 参数，第 6 篇 §1.2） ----------
# wire 层它就是一段 JSON 文本：模型"看到"的工具只有这份 schema，
# 你写的 Python 函数体模型一无所知（第 7 篇 §1.4 讲透这一点）。
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "get_time",
            "description": "获取当前本地时间（年-月-日 时:分:秒）。任何涉及当前时刻或日期的问题都必须先调它。",
            "parameters": {"type": "object", "properties": {}},
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "四则运算器：对 a、b 做一次 op 运算并返回 result。需要多步计算时，把上一步的结果作为下一步的输入继续调用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "a": {"type": "number", "description": "第一个操作数"},
                    "b": {"type": "number", "description": "第二个操作数"},
                    "op": {"type": "string", "enum": ["+", "-", "*", "/"],
                           "description": "运算符，四选一"},
                },
                "required": ["a", "b", "op"],
            },
        },
    },
]

# 名字 -> 实现：agent.py 阶段④ 按模型给的 function.name 查表执行
TOOL_REGISTRY = {"get_time": get_time, "calculate": calculate}

# 名字 -> schema：schema 校验时按名字取参数定义
SCHEMA_BY_NAME = {s["function"]["name"]: s["function"] for s in TOOL_SCHEMAS}
```

### 1.4 工具描述如何变成 prompt：模型"看到"的只是 schema

这是本篇第一个要害。读者常有的误解是"我把函数传给了模型"——没有的事。把这件事拆成三句：

**一、`tools` 就是请求体里的一个普通 JSON 字段。** 第 6 篇 §1.2 的 curl 里你已经见过它躺在 body 中间；`tools.py` 里那个 `TOOL_SCHEMAS` Python 列表，经 JSON 序列化后就是 wire 层那个字段本身——没有任何翻译层，你在 Python 里写的 dict 和模型收到的 JSON 逐字对应。

**二、模型看不到你的 Python 函数。** 它能看到的一切 = 随请求发过去的 schema 文本（名字、描述、参数类型、枚举）。官方 Tool Calls 指南原文说得很直白："the functionality of the `get_weather` function needs to be provided by the user. **The model itself does not execute specific functions**"（2026-10-15 实测）——模型只产出"我想调什么"的意图，执行永远发生在你的代码里（第 1 篇 §2.1 三方分工）。

**三、所以写 description 就是在写 prompt。** 第 3 篇说"工具注册表同时是给模型的文档"，现在你可以对着代码体会这句话了。本篇两个工具的描述里各藏了一句行为指令：

- get_time 的"**任何涉及当前时刻或日期的问题都必须先调它**"——不写这句，模型很可能凭训练记忆猜一个时间糊弄过去（幻觉的温床）；
- calculate 的"**需要多步计算时，把上一步的结果作为下一步的输入继续调用**"——这句直接支撑动手实践预置问题 3 的链式调用，改掉它你会看到模型开始自己心算（被系统提示词的第 2 条按住）。

参数描述同理：`calculate` 的 `op` 用 `enum` 约束成四选一，模型在生成参数时就会收窄到这四个值——但注意，**非 strict 模式下 schema 对模型是"软约束"**（生成时的倾向，不是协议级的强制），所以你的代码仍要在执行前校验，这是 §3 的主题。要"硬约束"也有办法：DeepSeek 的 strict 模式（Beta）让服务端强校验 schema，§3.2 展开。

## 2｜主循环 agent.py：每行都有为什么

读完本节你能理解：五阶段怎么变成 164 行可运行的代码——回填为什么长那样（含 `reasoning_content` 的 400 规则）、"我调完了"怎么判定才可信、工具的每条错误路径怎么兜。

### 2.1 完整代码

先给全文，随后逐块拆讲。第 3 篇说过"循环里没有一行聪明逻辑"——现在你可以逐行验证这句话：

```python
# agent.py — 第 7 篇成品：手写最小 agent（DeepSeek + openai SDK 直连，里程碑 M1）
# 依赖：pip install -r requirements.txt（openai 3.31.0；Python 3.11+，第 5 篇）
# 运行：
#   export DEEPSEEK_API_KEY=sk-xxx && python agent.py          # 跑 4 条预置问题（M1 验收路径）
#   python agent.py "37 乘以 45 等于多少？"                      # 单次提问
#   python agent.py --chat                                     # 交互模式（属于你的最小 agent）
#   python agent.py --think                                    # 思考档：验证 reasoning_content 回填（第 4 篇 §1.5）
# 诚实声明：循环逻辑（含四条错误兜底路径与熔断）已用桩模型本地实测（见文章实践节）；
#           对真实 DeepSeek 的调用 <未实测，以官方文档为准>
import argparse
import json

from openai import OpenAI

import config
import tools as toolset

# 系统提示词：第 4 篇 §2.2 反思指令的最简版——"引用数值必须原样来自工具结果"
# 是对"编造结果"最便宜的一道护栏；第 3 条错误自愈规则让 §3 的兜底路径真正能自愈。
SYSTEM_PROMPT = (
    "你是一个跑在终端里的最小 agent。规则：\n"
    "1. 事实性问题（当前时间、计算结果）必须调用工具获取，禁止凭记忆编造；\n"
    "2. 回答中引用的数值必须原样来自工具结果，不要自己改写或补全；\n"
    "3. 工具返回错误信息时，读懂错误、调整调用方式重试，不要原样重复同样的调用。"
)

PRESET_QUESTIONS = [
    "现在几点了？",
    "37 乘以 45 等于多少？",
    "先看看现在几点，再把当前的小时数乘以 7。",
    "用 calculate 算一下 1 除以 0，把工具返回的内容告诉我。",
]
```

```python
def create_client() -> OpenAI:
    """姿势二（第 6 篇 §2）：官方 SDK。三个字符串全部来自 config 唯一配置点。"""
    return OpenAI(api_key=config.api_key(), base_url=config.BASE_URL,
                  timeout=config.TIMEOUT, max_retries=config.MAX_RETRIES)


def call_llm(client: OpenAI, messages: list, think: bool = False):
    """阶段② 调 LLM：全部 messages + 工具清单一起发（第 3 篇：每轮全量重发）。"""
    kwargs = dict(model=config.MODEL, messages=messages, tools=toolset.TOOL_SCHEMAS)
    if think:   # 思考档（第 4 篇 §1.3）。默认不开：循环内普通调用用非思考档（第 4 篇 §4.4）
        kwargs["reasoning_effort"] = "high"      # SDK 原生参数，不走 extra_body（第 6 篇 §2.1）
        kwargs["extra_body"] = {"thinking": {"type": "enabled"}}
    else:
        kwargs["extra_body"] = {"thinking": {"type": "disabled"}}
    return client.chat.completions.create(**kwargs)


def to_assistant_dict(msg) -> dict:
    """把 SDK 返回的 assistant 消息固化成 dict（阶段⑤ 前半：回填模型意图）。
    官方做法是把整个 message 对象直接 append（官方 thinking 模式示例同构）；
    这里显式构造是为了看清回填了哪些字段——特别是 reasoning_content：
    带 tools 的请求必须完整回传它，否则 API 返回 400（第 4 篇 §1.5 官方原文）。
    getattr 防御性读取：旧版 SDK 类型可能未声明该字段，直接属性访问会 AttributeError。"""
    out = {"role": "assistant", "content": msg.content}
    reasoning = getattr(msg, "reasoning_content", None)
    if reasoning:
        out["reasoning_content"] = reasoning     # 非 thinking 档没有该字段，自然不带
    if msg.tool_calls:
        out["tool_calls"] = [
            {"id": c.id, "type": "function",
             "function": {"name": c.function.name, "arguments": c.function.arguments}}
            for c in msg.tool_calls
        ]
    return out
```

```python
def execute_tool(call) -> str:
    """阶段④ 执行单个工具调用。总原则：任何失败都不抛出——把错误文本当作工具结果
    回填给模型（它下一轮读到后自行纠正），循环永远不因工具问题崩掉（§3 表）。"""
    name = call.function.name
    if name not in toolset.TOOL_REGISTRY:        # 坑 1：幻觉调用——调了不存在的工具
        return (f"错误：不存在名为 {name!r} 的工具。"
                f"可用工具：{sorted(toolset.TOOL_REGISTRY)}")
    try:                                         # 坑 2：坏 JSON / 空参——arguments 是 JSON 字符串
        args = json.loads(call.function.arguments) if call.function.arguments else {}
    except json.JSONDecodeError as e:
        return f"错误：arguments 不是合法 JSON（{e}）。请重新发起调用。"
    ok, err = toolset.validate_args(toolset.SCHEMA_BY_NAME[name], args)   # 坑 3：schema 校验
    if not ok:
        want = json.dumps(toolset.SCHEMA_BY_NAME[name]["parameters"], ensure_ascii=False)
        return f"错误：参数校验失败（{err}）。参数要求：{want}"
    try:                                         # 坑 4：运行时异常，如除零
        result = toolset.TOOL_REGISTRY[name](**args)
    except Exception as e:   # 教学版从宽捕获；生产要按异常类型分级处理（第 9 篇展开）
        return f"错误：工具 {name} 执行异常：{type(e).__name__}: {e}"
    return json.dumps(result, ensure_ascii=False)
```

```python
def run_agent(client, user_input: str, think: bool = False) -> str:
    """一轮完整的 agent loop：第 3 篇五阶段的实体化。返回最终答复文本。"""
    # 阶段① 获取输入：system 纪律 + user 问题——agent 初始的内部状态
    messages = [{"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_input}]
    print(f"\n[用户] {user_input}")
    for i in range(1, config.MAX_ITERATIONS + 1):
        # 阶段② 调 LLM
        print(f"[loop] 第 {i} 轮 | 发送 {len(messages)} 条消息")
        resp = call_llm(client, messages, think=think)
        choice = resp.choices[0]
        msg, finish = choice.message, choice.finish_reason
        print(f"[loop] finish_reason={finish} | tool_calls={len(msg.tool_calls or [])} 个")
        if msg.content:
            print(f"[模型] {msg.content}")
        # 阶段⑤（前半）：先回填模型意图，工具结果才能与 tool_call_id 配对（第 6 篇硬规则 1）
        messages.append(to_assistant_dict(msg))
        # 阶段③ 终止判定：assistant 消息上没有 tool_calls = 模型宣告完成。
        # 判定读结构化信号（tool_calls 有无），不读文本——模型嘴上说"完成了"不算数（第 3 篇 §4.2）。
        if not msg.tool_calls:
            if finish != "stop":   # stop 之外的收尾（length=截断等，全集见第 6 篇 §1.3）
                print(f"[loop] 警告：finish_reason={finish}，回答可能不完整")
            print(f"[loop] 终止：共 {i} 轮")
            return msg.content
        # 阶段④ + ⑤（后半）：执行每一个工具调用、逐个配对回填。
        # 同一条消息里有多个 tool_calls 时也是这里逐个处理（官方 thinking 示例同构）；
        # 教学版顺序执行——并行化留到工具变成远程 HTTP 时（第 8 篇）再考虑。
        for call in msg.tool_calls:
            print(f"[工具] {call.function.name}({call.function.arguments})")
            result = execute_tool(call)
            print(f"[工具→] {result}")
            messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
    # 终止条件二：最大迭代数熔断——熔断后必须有明确行为，而不是静默返回空（第 3 篇 §4.2）
    raise RuntimeError(f"[熔断] 达到最大迭代数 {config.MAX_ITERATIONS}，强制终止")
```

```python
def run_safely(client, question: str, think: bool = False) -> str:
    """熔断等 RuntimeError 转成可见的答复文本——明确报错，不静默、不崩终端。"""
    try:
        return run_agent(client, question, think=think)
    except RuntimeError as e:
        return str(e)


def main():
    parser = argparse.ArgumentParser(description="手写最小 agent（第 7 篇，里程碑 M1）")
    parser.add_argument("question", nargs="?", help="单次提问；不给则跑预置问题集")
    parser.add_argument("--chat", action="store_true", help="交互模式；每行输入一次全新会话")
    parser.add_argument("--think", action="store_true", help="开思考档（第 4 篇 §1.5）")
    args = parser.parse_args()
    client = create_client()
    try:
        if args.chat:
            print("交互模式（输入 quit 退出）。注意：每次输入都是一次全新会话——"
                  "它为什么记不住上一句？P4·11 记忆篇回答。")
            while True:
                user_input = input("你> ").strip()
                if user_input.lower() in ("quit", "q", "exit"):
                    break
                if user_input:
                    print(f"[助手] {run_safely(client, user_input, think=args.think)}")
        elif args.question:
            print(f"[助手] {run_safely(client, args.question, think=args.think)}")
        else:                                # 预置问题集：M1 验收路径
            for q in PRESET_QUESTIONS:
                print(f"[助手] {run_safely(client, q, think=args.think)}")
    except KeyboardInterrupt:
        print("\n[退出] 用户中断（第 3 篇终止条件之四：用户确认/打断）")


if __name__ == "__main__":
    main()
```

### 2.2 回填为什么长这样：to_assistant_dict 与 reasoning_content 的 400 规则

`to_assistant_dict()` 是整个文件里最"多此一举"的一个函数——官方 thinking 模式示例直接写 `messages.append(response.choices[0].message)` 一行搞定，还明确说这与显式构造 dict **等价**（官方给出的等价展开正是 `role/content/reasoning_content/tool_calls` 四个字段，2026-10-15 实测）。那我们为什么显式写？两个原因：

1. **教学原因**：你亲眼看清"回填了哪些字段"。回填是 loop 的命门（第 3 篇实践里注释掉它循环就失忆），把它藏进 SDK 对象的序列化魔法里，你就失去了观察点。
2. **工程原因**：`reasoning_content` 不是 OpenAI 标准字段，不同版本的 openai SDK 对它的类型声明程度不同——直接属性访问在不声明该字段的版本上会 `AttributeError`。`getattr(msg, "reasoning_content", None)` 两种版本都安全。

而它的存在本身就是本系列反复预警的那条硬规则的落点（官方 Thinking Mode 页原文，2026-10-15 实测）：

> the reasoning_content **must be fully passed back to the API in all subsequent requests** — even for turns where the model did not perform a tool call. If your code does not correctly pass back reasoning_content, **the API will return a 400 error**.

翻译成 loop 视角：**只要请求带 `tools`（agent loop 每一轮都带），每一轮回填的 assistant 消息都必须原样带上思维链**。本篇默认非思考档，响应里没有这个字段、`if reasoning` 不成立、请求自然不带——一套代码两档通吃；你跑 `--think` 时它就开始干活，这是自检清单里的选做实验。动手实践桩测试里有一条专门验证它（S10）。

### 2.3 "我调完了"怎么判：结构化信号 + 非 stop 警告 + 熔断

终止判定（第 3 篇 §4.2 的"任务完成判定"）写成代码只有一行：`if not msg.tool_calls:`。三个设计决定值得逐个说：

**一、判定读 `tool_calls` 有无，不读文本。** 官方 thinking 示例的同款判定是 `if tool_calls is None: break`——协议层面的"完成"就是"这条 assistant 消息不再携带调用意图"。模型在 `content` 里写"我已经完成了"不算数：finish_reason 与 tool_calls 是结构化信号，content 是自然语言——前者能被程序判定，后者只能被人读。这是第 3 篇那条规则的代码形态。

**二、`finish_reason` 做"次级校验"，不做主判定。** 无 tool_calls 且 `finish_reason="stop"` 是教科书式收尾；但如果是 `length`（被截断）呢？回答可能是半句话。代码对非 stop 的收尾打警告而不吞掉——全集（stop/length/content_filter/…）第 6 篇 §1.3 已列，这里只处理最常见的一种。

**三、熔断在循环之外，官方示例里没有。** 值得停下来对照：官方 thinking 示例的循环是 `while True`，**没有最大迭代数**——示例为最小化省掉了工程保险，你的 loop 不能省（第 3 篇 §4.1 的事故现场：挂在线上跑过周末，周一账单四位数）。本篇 `for i in range(1, MAX_ITERATIONS + 1)` 跑满即 `raise RuntimeError`，`run_safely` 把它转成终端里可见的 `[熔断]` 文本——第 3 篇的要求"熔断后要有明确行为，而不是静默返回空"，就是这么落的。

### 2.4 错误路径总原则：把错误当结果，不当异常

`execute_tool()` 有四条出口，三条是错误，但**没有一条抛异常**——全部返回错误文本字符串，由调用方照常作为 `role:"tool"` 消息回填。为什么？对比两条路线：

- **崩掉**（抛出异常终止循环）：一次工具失败杀死整个任务，模型连"补救"的机会都没有。后端类比：下游 RPC 报错就整个请求 500，连降级响应都不给。
- **回喂**（错误文本回填，循环继续）：模型下一轮读到"错误：不存在名为 'get_weather' 的工具。可用工具：['calculate', 'get_time']"，立刻知道改口。错误变成了信息——这是 agent 独有的自愈通道，普通 Web 后端没有"把异常喂回去让调用方换种问法"的机制，因为调用方是死代码；而 loop 里的调用方是模型，它读得懂。

这也是系统提示词第 3 条（"工具返回错误信息时，读懂错误、调整调用方式重试"）存在的原因：指令 + 兜底配对，自愈通道才算真正打通。同理，坏参数也不静默修复——不要试图帮模型"猜"出正确参数，把校验错误连同参数要求原文一起回喂，让它在下一轮自己改。

`run_agent` 里还有一处容易漏看的细节：**多个 tool_calls 逐个执行、逐个配对回填**（第 6 篇硬规则 1——每个 tool_call 都要有自己的 tool 结果）。第 5 篇的 20 行版只处理了 `tool_calls[0]`，模型一次要两个工具就会协议违约；本篇的 `for call in msg.tool_calls:` 补上了。教学版顺序执行——两个本地微秒级工具并行毫无收益；等工具变成远程 HTTP（第 8 篇），并行化才进入考虑，届时一起来的还有并发结果排序与异常聚合的问题。

## 3｜schema 校验与踩坑清单

读完本节你能理解：模型给的参数在进函数之前经过哪几道闸、每道闸拦什么，以及 agent 工程里四个必然踩的坑各自的"现象/原因/防护"。

### 3.1 校验什么、在哪层拦

`validate_args()`（`tools.py` 末尾，即 §1.3 说的第三段）是刻意做薄的教学实现，完整代码如下：

```python
# ---------- 最小 schema 校验：模型给的参数在进函数之前先拦一道 ----------
def validate_args(function_schema: dict, args: dict):
    """校验模型给出的参数，返回 (ok, 错误信息)。
    只查模型最常见的三类坏参数：缺必填 / 类型不对 / 枚举外取值。
    这是教学最小实现；生产建议换 pydantic，或用 DeepSeek strict 模式让服务端
    强约束 schema（Beta，第 7 篇 §3.2）。"""
    params = function_schema["parameters"]
    props = params.get("properties", {})
    for name in params.get("required", []):
        if name not in args:
            return False, f"缺少必填参数 {name!r}"
    for name, value in args.items():
        spec = props.get(name)
        if spec is None:
            return False, f"传入了 schema 未定义的参数 {name!r}"
        checks = {
            "string": lambda v: isinstance(v, str),
            "number": lambda v: isinstance(v, (int, float)) and not isinstance(v, bool),
            "integer": lambda v: isinstance(v, int) and not isinstance(v, bool),
            "boolean": lambda v: isinstance(v, bool),
        }
        t = spec.get("type")
        if t in checks and not checks[t](value):
            return False, f"参数 {name!r} 应为 {t}，实际为 {type(value).__name__}"
        if "enum" in spec and value not in spec["enum"]:
            return False, f"参数 {name!r} 必须是 {spec['enum']} 之一，实际为 {value!r}"
    return True, ""
```

它只查模型最常见的三类坏参数：

1. **缺必填**——`required` 列表里的名字没出现在参数里（模型"忘了"填 `b`）；
2. **类型不对**——schema 声明 `number` 却传来字符串 `"14"`。注意一个 Python 陷阱：`bool` 是 `int` 的子类，`isinstance(True, int)` 为真，所以 number/integer 的检查要先把 `bool` 排除掉，否则 `op` 传个 `true` 都能溜进函数；
3. **枚举外取值**——`op` 传了 `"%"`。enum 在生成时是"软引导"，拦不住模型偶尔的任性。

外加一道"schema 外参数"检查（传了没定义的字段）。三类检查全部通过才真正调用函数——**校验的位置在"进函数之前"**，因为校验要防的是垃圾进（垃圾参数污染函数执行），不是垃圾出。

生产上这套手工校验有两个升级方向：① 换 **pydantic** 校验（本系列第 5 篇对比过的 pydantic-ai 正是把"类型注解 → 校验 + schema 生成"整条链自动化）；② 用 **DeepSeek strict 模式**把约束推到服务端（Beta，2026-10-15 官方文档实测）：`base_url` 换成 `https://api.deepseek.com/beta`，工具定义加 `"strict": true`，服务端会校验你提交的 JSON Schema 本身的合法性，并在生成时强约束输出符合 schema——代价是 schema 写法受限（如 object 的所有 properties 必须全部 required、`additionalProperties: false`），且依赖 Beta 端点的稳定性。两层不互斥：strict 管"模型尽量别写错"，你的校验管"写错了也进不了函数"。

### 3.2 踩坑清单：现象 / 原因 / 防护

这四坑（外加死循环）是手写 agent 的必修税，本篇的代码已经全部内置防护，表格里给出每条的复现方式：

| 现象 | 原因 | 防护（本篇落点） |
|---|---|---|
| 幻觉调用：模型调一个不存在的工具，或按已废弃的旧描述调用 | schema 与实现不同步（声明了没实现 / 改了实现忘改描述）；描述含糊诱导联想 | `execute_tool` 查注册表，查不到→错误文本回填；schema 与实现同文件成对维护（§1.3）。复现：自检清单选做实验 |
| 空参/坏 JSON：`arguments` 解析失败 | `arguments` 是 JSON **字符串**而非对象（第 6 篇 §1.3），模型可能输出空串、截断或非法格式 | `json.loads` 包 try，空串按 `{}` 处理（配无参工具）；错误回填让模型重发 |
| 参数不合规：缺必填/类型错/枚举外取值 | 非 strict 模式下 schema 是软约束，模型偶尔不按规矩填 | `validate_args` 进函数前拦截→校验错误+参数要求原文一起回喂，模型自愈 |
| 工具运行时异常：除零、断网、空指针 | 工具实现自身抛错，与模型无关 | try/except 包住执行→异常类型+信息回填，循环不崩（`calculate` 的除零就是活教材） |
| 死循环：反复调同一工具不收尾 | 模型自判失效（第 3 篇 §4.1 的"溢出→压缩→同循环"环） | `MAX_ITERATIONS` 熔断+明确报错；进阶判据见 OpenClaw"连续三次相同工具错误必停"（第 3 篇 §4.2 出处） |

一条共同的设计哲学：**所有防护都不以"崩掉"为终点**。错误要么变成回喂给模型的信息（前三条），要么变成终端里明确的人话报错（死循环）——demo 崩了可以重跑，线上 agent 崩一次的代价是一整个用户会话。

## 4｜结构化输出：JSON mode 与 function calling 的异同

读完本节你能理解：`response_format` / JSON Output 是什么、它和 function calling 是不是竞争关系，以及"什么时候用哪个"的判断口诀。

### 4.1 新名词 JSON mode：让模型"按给定结构交作业"

大白话先行：**function calling 是模型主动"点菜"（我要调什么工具），JSON mode 是你逼模型"按格式交作业"（不管答什么，必须是合法 JSON）**。

定义与出处（官方 JSON Output 页 + API Reference，2026-10-15 curl 实测）：把请求的 `response_format` 参数设为 `{"type": "json_object"}` 即开启 JSON Output——官方原话"ensures the model outputs valid JSON strings"；该参数取值只有 `text`（默认）与 `json_object` 两种。官方 Notice 列了三条使用要求，全部值得抄进工程习惯：

1. **prompt（system 或 user）必须包含 "json" 字样，并给出期望格式的示例**——否则模型不知道你要什么结构；
2. **合理设置 `max_tokens`**——JSON 字符串中途被截断就废了（API Reference 还特别警告：不指令模型产 JSON 时它可能"输出无尽空白直到 token 上限"，请求看起来像卡死）；
3. **官方承认的已知问题**：JSON Output 偶尔会返回空 content，官方建议改 prompt 缓解——连"保证合法 JSON"本身都不是 100% 可靠的，这是对"模型输出的任何保证都要留一手"的最好提醒。

### 4.2 与 function calling 的异同：什么时候用哪个

两者都拿 JSON Schema 表达结构，但服务的是相反的控制流需求：

| | function calling（本篇主线） | JSON mode（`response_format`） |
|---|---|---|
| 解决什么 | 模型**决定下一步做什么**（要能力） | 模型**按给定结构交作业**（要格式） |
| 控制流 | 多轮循环，模型决定调不调、调几次、什么顺序 | 单轮请求-响应，一次出结果 |
| 输出形态 | `tool_calls` 调用意图（要你执行后回填） | `content` 是合法 JSON 字符串，直接 `json.loads` |
| 结构保证 | 参数受 schema 约束（strict 模式可服务端加强） | 整体保证合法 JSON；**字段级**结构靠 prompt 里的示例引导 |
| 失败模式 | 幻觉调用/空参（§3 表） | 空 content（官方已知问题）/字段不合期望/截断 |
| 典型场景 | agent loop、查库、写后台、任何"要做动作"的任务 | 信息提取（从文本抽数据）、NL→固定字段、给下游代码消费 |
| 什么时候不用 | 只要固定格式不要动作——杀鸡用牛刀，JSON mode 更轻 | 需要模型自主决定多步动作——它给不了控制流 |

口诀一句话：**让模型"做动作"用 function calling，让模型"按格式说话"用 JSON mode**。"帮我把张三的角色改成 admin"是动作（function calling）；"从这段投诉文本里提取用户名和订单号给工单系统"是格式（JSON mode）。本系列主线是 agent loop，所以主线永远是 function calling；JSON mode 会在信息提取类子任务里出现（如 P4 记忆篇里"从对话提取偏好字段"）。

顺带一个本篇核验到的边界事实：现行 JSON Output 文档页**没有**列出与 Tool Calls 同用的限制说明（早期版本文档曾有相关提示）；如果业务里两者确实要同时出现，先按官方最新文档自行验证 `<以官方 docs 为准>`。

### 4.3 动手验证：json_mode_demo.py

`agent/json_mode_demo.py`（完整文件已在 §1.1 目录清单中）就是口诀里"信息提取"场景的最小实现：

```python
# json_mode_demo.py — 第 7 篇 §4：结构化输出（JSON mode / response_format）演示
# 与 function calling 的分工：function call 让模型"决定调什么"，
# JSON mode 让模型"按给定结构交作业"——后者适合信息提取类单轮任务（§4.2 对比表）。
# 运行：export DEEPSEEK_API_KEY=sk-xxx && python json_mode_demo.py
# 说明：写法逐字段对照官方 JSON Output 页示例（api-docs.deepseek.com/guides/json_mode，
#       2026-10-15 核验：官方示例即 model="deepseek-flash" + response_format={'type':'json_object'}）；
#       对真实 DeepSeek 的调用 <未实测，以官方文档为准>
import json

from openai import OpenAI

import config

client = OpenAI(api_key=config.api_key(), base_url=config.BASE_URL,
                timeout=config.TIMEOUT, max_retries=config.MAX_RETRIES)

resp = client.chat.completions.create(
    model=config.MODEL,
    messages=[
        # 官方 Notice 两条硬要求：prompt 必须包含 "json" 字样 + 给出期望格式的示例引导，
        # 否则模型可能输出无尽空白直到 token 上限（API Reference Important 原文）
        {"role": "system", "content": (
            '从用户输入中提取姓名和城市，输出 json，格式示例：'
            '{"name": "张三", "city": "杭州"}。只输出 json，不要输出其他内容。')},
        {"role": "user", "content": "我叫张三，下周去杭州出差，帮我看看那边的安排。"},
    ],
    response_format={"type": "json_object"},   # 官方取值只有 text（默认）/ json_object
)

# JSON mode 保证 content 是合法 JSON 字符串，可直接 loads 给后续代码用；
# 官方提醒：仍要合理设置 max_tokens，且注意 finish_reason="length" 时可能被截断
data = json.loads(resp.choices[0].message.content)
print(json.dumps(data, ensure_ascii=False, indent=2))
```

预期输出（真实调用 `<未实测，以官方文档为准>`；官方 JSON Output 页的示例输出同构——`content` 被直接 `json.loads` 成 Python 对象）：

```json
{
  "name": "张三",
  "city": "杭州"
}
```

对照着看两条通道的分野：这里有 `tools` 参数、没有循环、模型不"做事"只"交作业"；agent.py 里恰好反过来。同一个模型 API，两种结构化输出姿势，各管一段。

## 动手实践：里程碑 M1——终端跑起你的第一个 agent

目标：Part 2 的里程碑验收——终端里跑起"属于你的最小 agent"，亲眼看到预置问题被回答、工具被调用、循环有日志、终止有交代。做完你会验证本篇的四个设计决策：**工具描述是 prompt**（改描述就改行为）、**错误是信息**（除零后 agent 不崩反而自愈）、**判定读结构化信号**（终止行出现 finish_reason=stop）、**熔断是最后保险**（模型不说停也能停）。

诚实声明：`agent.py` 的循环逻辑（含四条错误兜底路径、多 tool_calls 配对、熔断、`reasoning_content` 回填）已用桩模型在本机实测通过（10 组场景 16 项断言全过，方法见文末核验说明）；`[loop]`/`[工具]` 日志行的格式与桩实测逐字一致。**对真实 DeepSeek 的调用 `<未实测，以官方文档为准>`**——`[模型]`/`[助手]` 的具体文本每次运行随模型浮动，下方预期输出中的模型文本为示意，判断验收看**结构**不看措辞。

### 第 0 步：准备

```bash
# 进入 agent/ 目录；环境复用第 5 篇的 venv（Python 3.11+），没有就重建
cd agent
python3.11 -m venv .venv && source .venv/bin/activate   # 已建过则只 source
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# key 只走环境变量（第 6 篇 §2.2；平台：platform.deepseek.com）
export DEEPSEEK_API_KEY=sk-xxxxxxxx

# 确认版本（系列约定 openai 3.31.0）
python -c "import openai; print(openai.__version__)"
```

预算：4 条预置问题全部非思考档、每条 1-3 轮调用，按第 4 篇 §4.2 的价格量级估算合计 **约 0.01-0.05 元**；选做实验另计（每个不超过几分钱）。

### 第 1 步：跑预置问题集（M1 验收路径）

```bash
python agent.py
```

预期输出（结构示意；`[loop]`/`[工具]` 行为桩实测格式，`[模型]`/`[助手]` 文本随模型浮动）。预置问题 1 的完整样子：

```
[用户] 现在几点了？
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_time({})
[工具→] {"now": "2026-10-15 20:31:07"}
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=stop | tool_calls=0 个
[模型] 现在是 2026-10-15 20:31:07。
[loop] 终止：共 2 轮
[助手] 现在是 2026-10-15 20:31:07。
```

预置问题 2（计算）形状相同，`[工具]` 行变成 `calculate({"a": 37, "b": 45, "op": "*"})`。预置问题 3（链式）会看到**两轮工具**，这是最值得盯着看的一条：

```
[用户] 先看看现在几点，再把当前的小时数乘以 7。
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_time({})
[工具→] {"now": "2026-10-15 20:31:19"}
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] calculate({"a": 20, "b": 7, "op": "*"})
[工具→] {"result": 140}
[loop] 第 3 轮 | 发送 6 条消息
[loop] finish_reason=stop | tool_calls=0 个
[模型] 现在是 20 点，20 × 7 = 140。
[loop] 终止：共 3 轮
[助手] 现在是 20 点，20 × 7 = 140。
```

预置问题 4（除零）是错误路径的现场：工具抛异常、错误文本回填、模型读到后改口——注意 `[工具→]` 的错误行之后循环**没有**停：

```
[用户] 用 calculate 算一下 1 除以 0，把工具返回的内容告诉我。
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] calculate({"a": 1, "b": 0, "op": "/"})
[工具→] 错误：工具 calculate 执行异常：ZeroDivisionError: division by zero
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=stop | tool_calls=0 个
[模型] 工具返回了错误：除数不能为 0（ZeroDivisionError），所以 1 除以 0 没有结果。
[loop] 终止：共 2 轮
[助手] 工具返回了错误：除数不能为 0（ZeroDivisionError），所以 1 除以 0 没有结果。
```

### 第 2 步：交互模式与单次提问

```bash
python agent.py --chat                          # 连续对话；每次输入都是全新会话
python agent.py "37 乘以 45 等于多少？"           # 单次提问，跑完即退
python agent.py --think "37 乘以 45 等于多少？"   # 思考档：明显变慢（第 4 篇 §4.3 的体感）
```

`--think` 的验证点：**不报 400** 就说明 `reasoning_content` 回填正确（§2.2 的硬规则被代码接住了）；另外思考档下 `[模型]` 行有时在工具轮为空——思维链走了 `reasoning_content`，正式内容要等收尾轮。

### 看懂输出：四个亲眼所见

1. **"发送 N 条消息"逐轮变大**（2→4→6）：第 3 篇"每轮全量重发"从图变成了你终端里的数字；预置 3 的 2→4→6 正好是"每轮 +1 条 assistant +1 条 tool"的算术。
2. **`[工具]` 行的参数来自模型、`[工具→]` 行的结果来自你的代码**——第 1 篇三方分工的实物化：模型只出意图，执行永远在你这（§1.4 认识二）。
3. **预置 4 的错误路径**：`[工具→] 错误：...` 之后模型没有原样重试也没有崩溃，下一轮直接改口——§2.4"错误即信息"的自愈通道，你现在亲眼见过它工作。
4. **`[loop] 终止：共 N 轮` 前面那行是 `finish_reason=stop`**：终止判定读的是结构化信号（tool_calls 归零 + finish_reason），不是模型嘴上说的"完成了"。

### 第 3 步：自检清单（M1 验收标准）

前三条是 M1 的验收线，全部通过即里程碑达成：

- [ ] `python agent.py` 跑完 4 条预置问题，每条都以 `[loop] 终止：共 N 轮` 结束（**正常终止**）；
- [ ] 预置 1/2/3 各至少出现一次 `[工具]` 行，且 `[助手]` 引用的数值能在 `[工具→]` 行里找到原文（**回答预置问题并调用工具**）；
- [ ] 全程每轮都有 `[loop]` 日志、`发送 N 条消息` 逐轮递增（**循环可见**）；
- [ ] 预置 3 出现两轮 `[工具]`（get_time → calculate 链式），`calculate` 的参数里 `a` 等于上一轮 `[工具→]` 返回的小时数；
- [ ] 预置 4 出现 `[工具→] 错误：...ZeroDivisionError...` 且 agent 没崩、最终给出解释（错误兜底可见）；
- [ ] `python agent.py --help` 与不设 key 的 `python agent.py` 都给出人话报错（配置闸门）；
- [ ] （选做）`--think` 跑预置 2：明显变慢、结果一致、无 400 报错（reasoning_content 回填验证）；
- [ ] （选做·复现幻觉调用）在 `tools.py` 的 `TOOL_SCHEMAS` 里加一个 `get_weather` 的 schema（照抄 `get_time` 改名即可）但**不**往 `TOOL_REGISTRY` 注册实现，然后问 `python agent.py "杭州天气如何？"`——你会看到 `[工具] get_weather(...)` 与 `[工具→] 错误：不存在名为 'get_weather' 的工具...`。这就是"schema 与实现不同步"事故的最小复现，也是 §1.3"成对维护"规矩的由来；
- [ ] （选做·复现熔断）把 `config.MAX_ITERATIONS` 改成 1 再跑预置 3：第 1 轮执行完工具就触发 `[助手] [熔断] 达到最大迭代数 1，强制终止`——模型从未说停，循环照样被你的代码刹住；看完改回 10；
- [ ] （选做）`python json_mode_demo.py` 输出两个字段的 JSON（§4 的结构化输出通道）。

---

到这里 Part 2 收口。回看这五篇的接力：第 3 篇的 15 行骨架、第 4 篇的模型选型、第 5 篇的环境与选型地图、第 6 篇的调用零件，全部在这份 `agent/` 里合体——里程碑 M1 的每个验收点（预置问题、工具调用、可见日志、正常终止）都不再依赖任何桩函数。但也要诚实地说清这份成品的边界：它只会玩两个演示工具，工具层薄得一层纸——没有认证、没有真实 API 的超时重试分级、没有危险操作的确认闸门。**下一 Part 会暴露什么问题**：把工具从 `get_time` 换成"登录后台 → 查用户 → 分配角色"的真实 REST 接口，你会立刻撞上工具工程真正的水深——token 会过期、4xx/5xx 要喂回模型、写操作需要确认——那正是 Part 3 的起点。

## 本篇结论

- 最小 agent = 第 3 篇骨架 + 真实 LLM + 两个演示工具：工具描述（JSON Schema）就是随请求发送的 prompt，模型看不到你的函数、只出调用意图，执行与校验永远在你这侧。
- 循环的正确性在三处硬细节：回填要带 `reasoning_content`（带 tools 请求缺了就 400）、终止判定读结构化信号（无 tool_calls + finish_reason）、熔断是唯一不依赖模型自觉的保险；工具的四条错误路径（幻觉调用/坏 JSON/参数不合规/运行时异常）一律以"错误文本回喂模型"兜底，循环不崩。
- JSON mode（`response_format={"type":"json_object"}`）与 function calling 是两种控制流需求：让模型做动作用 function calling，让模型按结构交作业用 JSON mode。里程碑 M1 达成——下一 Part 让它接上真实系统。

## 参考资料

1. [DeepSeek API Docs — JSON Output](https://api-docs.deepseek.com/guides/json_mode) — `response_format={'type':'json_object'}` 开启方式、"prompt 必须含 json 字样并给示例"、合理设 max_tokens、偶发空 content 的官方说明、官方 Python 示例（`model="deepseek-flash"`）。2026-10-15 curl 实测。
2. [DeepSeek API Docs — Tool Calls](https://api-docs.deepseek.com/guides/tool_calls) — "The model itself does not execute specific functions" 原文、`role:"tool"` + `tool_call_id` 回填官方示例、thinking 模式工具支持（V3.2 起）、Chat Completion API 不支持中途插入 tool calls、strict 模式（Beta：`/beta` base_url、`strict:true`、服务端校验 JSON Schema、支持类型清单）。2026-10-15 curl 实测。
3. [DeepSeek API Docs — Thinking Mode](https://api-docs.deepseek.com/guides/thinking_mode) — **带 tools 请求必须完整回传历史 reasoning_content 否则 400** 的官方原文、`messages.append(message)` 与显式 dict 回填的官方等价写法、官方工具调用示例循环（`tool_calls is None` 判停）与示例输出。2026-10-15 curl 实测。
4. [DeepSeek API Docs — API Reference（create-chat-completion）](https://api-docs.deepseek.com/api/create-chat-completion) — `response_format` 参数定义（取值 text/json_object、默认 text）与"不指令模型产 JSON 会输出无尽空白"的 Important 原文、`finish_reason="length"` 截断警告。2026-10-15 curl 实测。
5. [openai-python 官方仓库](https://github.com/openai/openai-python)（[PyPI](https://pypi.org/project/openai/) `openai 3.31.0`） — `OpenAI()` 构造参数（api_key/base_url/timeout/max_retries）、`chat.completions.create` 的 `tools`/`response_format`/`reasoning_effort` 参数。版本号沿用第 5/6 篇 2026-10-15 PyPI 核验。
6. 系列前序：[第 3 篇](03-agent-loop与上下文.md)（五阶段骨架、终止条件四类、工具注册六件套）、[第 4 篇](04-模型选型与推理.md)（thinking 开关与 reasoning_content 回传规则、非思考档默认）、[第 5 篇](05-Python选型地图.md)（venv 环境与 Python 3.11+ 约定）、[第 6 篇](06-调用LLM的三种姿势.md)（消息四快照、tool_call_id 硬规则、finish_reason 全集、超时重试默认值）。

> 核验说明：本篇所有 API 事实（JSON Output 三条要求、strict 模式细节、reasoning_content 400 规则、官方回填等价写法与示例循环、response_format 取值）均于 2026-10-15 用 curl 抓取 api-docs.deepseek.com 四个页面逐字核验；openai 3.31.0 沿用第 5/6 篇当日 PyPI 核验。`agent/` 四个 Python 文件通过 `python3 -m py_compile`；`agent.py` 循环逻辑（正常闭环/链式调用/多 tool_calls 配对/幻觉调用/坏 JSON/缺参/枚举外取值/除零异常/熔断/reasoning_content 回填，10 组场景 16 项断言）已用桩模型在本机实测全过——桩只替换模型响应，被测代码与文中逐字一致；对真实 DeepSeek 的调用未实测并已在文中与代码注释显式标注。预期输出中的 `[loop]`/`[工具]` 日志行与桩实测格式逐字一致，`[模型]`/`[助手]` 文本为结构示意。




















