# 第 7 篇：手写一个"看得懂每行代码"的最小 agent
### 里程碑 M1 · 终端跑起属于你的 agent

> 知识地图【A2/A3/B4】⭐ 啊哈时刻 · Part 2 收官 + 里程碑 M1
> 读完你能理解：把骨架 + 真 LLM + 两个工具拼成成品要写多少行；工具描述怎么变成模型的
> "prompt"；四个必踩的坑（幻觉调用/空参/工具异常/死循环）代码怎么接；结构化输出与
> function calling 的异同——最后亲手跑起 **M1**。

## 本篇要回答的问题

1. 从"骨架"到"成品"：拼装清单是什么？**agent/ 项目**怎么组织？
2. "工具描述变成模型的 prompt"发生在哪一行？**参数校验**守住哪几处？
3. 四个必踩的坑：**幻觉调用 / 空参 / 工具异常 / 死循环**——代码里怎么接？
4. **结构化输出**与 function calling 的异同？什么时候用哪个？
5. **M1 自检清单**：怎么确认"我真的做出了一个最小 agent"？

---

## 1｜拼装清单：三个原料 → 一个成品

第 7 篇没有新知识，全是"组装"——但组装本身就是一次考试：

| 原料 | 来自 | 在本篇的角色 |
|---|---|---|
| 五阶段 loop + 终止条件 | 第 3 篇（假模型骨架，已实测）| 骨架不动，换血肉 |
| 真实消息协议 + 工具调用格式 | 第 6 篇（REST/SDK/四角色）| `tools` 参数、`tool_calls`、`role:"tool"` 回填 |
| 模型接入 + key 管理 | 第 4 篇（DeepSeek）| `deepseek-flash` + `.env`（thinking 默认开启，本 loop 天然兼容）|

**成品 `agent/` 目录**（写作规范：从第 7 篇起项目同仓库演进——直连版 → MCP 版 →
subagent 版 → 工程壳版，后面每一 Part 都在它身上改）：

```text
agent/
├── .env            # DEEPSEEK_API_KEY=（把第 1 篇的复制过来）
├── .gitignore      # 一行：.env
├── tools.py        # 工具实现 + 注册表 + 给模型看的工具清单
└── main.py         # 五阶段 loop + 终止条件（≈70 行，本次全文给出）
```

安装依赖（国内源，沿用第 1 篇）：

```bash
cd agent
python3 -m venv .venv && source .venv/bin/activate
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple openai python-dotenv
```

---

## 2｜逐个文件看：**工具描述如何变成 prompt**

### tools.py：两个工具 + 一张"说明书"

```python
"""agent/tools.py — 工具实现 + 给模型看的工具清单（schema）。

两部分：
1. 工具的 Python 实现（真正干活的函数）
2. TOOLS_SCHEMA：工具的"说明书"，随每次请求发给模型——这就是"描述变成 prompt"的地方
"""
import ast
import operator as op
from datetime import datetime


def get_current_time():
    """无参工具：返回当前本地时间字符串。"""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


# ---- 安全计算器：只放开 + - * / % ** 和括号（ast 白名单，绝不 eval）----
_OPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
        ast.Div: op.truediv, ast.Mod: op.mod, ast.Pow: op.pow,
        ast.USub: op.neg}


def _eval_node(node):
    if isinstance(node, ast.Expression):        # 整个表达式
        return _eval_node(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value                       # 数字字面量
    if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_eval_node(node.left), _eval_node(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_eval_node(node.operand))
    raise ValueError(f"不支持的表达式片段: {ast.dump(node)}")   # 白名单之外的都拒绝


def calc(expression):
    """单参工具：计算数学表达式，如 '(23*17+5)/2'。"""
    return _eval_node(ast.parse(expression, mode="eval"))


# ---- 注册表：模型声明的"工具名" → 真正执行的函数 ----
registry = {
    "get_current_time": get_current_time,
    "calc": calc,
}

# ---- 工具清单（OpenAI 兼容 tools 参数格式）----
TOOLS_SCHEMA = [
    {
        "type": "function",
        "function": {
            "name": "get_current_time",
            "description": "获取当前日期和时间。用户问『现在几点/今天几号』时使用。",
            "parameters": {"type": "object", "properties": {}, "required": []},
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calc",
            "description": "计算一个数学表达式（支持 + - * / % ** 与括号）。算术类请求必须用本工具，不要心算。",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string",
                                   "description": "要计算的表达式，例如 '(23*17+5)/2'"},
                },
                "required": ["expression"],
            },
        },
    },
]
```

**逐点讲解**（每一处都在回应前几篇埋的伏笔）：

- **"工具描述如何变成 prompt"就在 `TOOLS_SCHEMA`**：它随每次请求的 `tools` 参数发给模型；
  模型"读"的就是每个工具的 `name`（叫什么）、`description`（干什么、什么时候用）、
  `parameters`（参数叫什么、什么类型、哪些必填）。**这三样写不清楚，模型就不选它、
  或选错、或填错参数**——第 3 篇说"工具描述是模型能看到的说明书"，这里就是实体。
  （写得规范不规范，第 10 篇有专门的设计规范；现在先记住"description 里要写清
  **何时使用**，比如那句『算术类请求必须用本工具，不要心算』就是防模型偷懒"。）
- **安全计算器为什么不用 `eval`**：模型生成什么表达式不受你控制，`eval` 等于把
  命令执行权交出去；ast 白名单只放行"数字 + 四则运算"，其余一律 `ValueError`。
  本环境实测：`calc("1/0")` 抛 `ZeroDivisionError`；`calc("__import__('os')…")`
  被白名单挡下——**工具是攻击面**（第 10 篇主题）的第一课。
- **`registry`（工具注册表）**：第 3 篇的雏形是 `{"get_time": …}`，形式没变——
  它是 harness"工具注册"服务的极小版。

### main.py：五阶段 + 四种兜底

```python
"""agent/main.py — 手写最小 agent：真 LLM + 2 个工具 + 五阶段 loop（里程碑 M1）。

五阶段：①取输入 ②组装(SYSTEM+历史+工具清单) ③调LLM ④解析 ⑤行动/回填 → 回到②
<本环境未实测——LLM 部分需 DEEPSEEK_API_KEY；工具层(calc)已单独实测>
"""
import json
import os
import sys

from dotenv import load_dotenv
from openai import OpenAI

from tools import TOOLS_SCHEMA, registry

load_dotenv()
client = OpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                base_url="https://api.deepseek.com")

SYSTEM = """你是一个最小可用助手。规则：
1. 需要当前时间或做算术时，调用对应工具，不要凭空猜测。
2. 拿到工具结果后再决定下一步；任务说完就给出最终答复。
"""

MAX_STEPS = 8   # 终止条件②：宿主硬刹车（防死循环）


def log_step(step, msg):
    """调试习惯：每轮打一条单行 JSON（最小 trace 雏形，第 18 篇升级为正式版）。"""
    names = [tc.function.name for tc in (msg.tool_calls or [])]
    print(json.dumps({"step": step, "tool_calls": names,
                      "content": (msg.content or "")[:60]}, ensure_ascii=False))


def call_model(messages):
    """②③ 组装 + 调 LLM：SYSTEM + 全部历史 + 工具清单，一起发出去。"""
    resp = client.chat.completions.create(
        model="deepseek-flash",
        messages=[{"role": "system", "content": SYSTEM}] + messages,
        tools=TOOLS_SCHEMA,
    )
    return resp.choices[0].message


def run_tool(tc):
    """⑤ 执行一个 tool_call；任何错误都变成文本喂回模型，绝不把循环搞崩。"""
    name = tc.function.name
    try:
        args = json.loads(tc.function.arguments or "{}")   # 守门①：空参/坏参
    except json.JSONDecodeError as e:
        return f"参数不是合法 JSON（{e}）。请修正后重试。"
    fn = registry.get(name)
    if fn is None:                                         # 守门②：幻觉调用
        return f"工具不存在：{name}。可用工具：{list(registry)}"
    try:
        return str(fn(**args))                             # 守门③：工具异常
    except Exception as e:
        return f"工具执行出错：{type(e).__name__}: {e}。如参数有误请修正后重试。"


def agent_loop(user_input):
    messages = [{"role": "user", "content": user_input}]   # ① 取输入
    for step in range(1, MAX_STEPS + 1):
        msg = call_model(messages)                         # ②③ 组装 + 调用
        log_step(step, msg)
        messages.append(msg)                               # ④ assistant 消息入库（可能带 tool_calls）
        if not msg.tool_calls:                             # 终止条件①：模型给最终答复 = 完成
            return msg.content
        for tc in msg.tool_calls:                          # ⑤ 行动：逐个执行 + 结果回填
            result = run_tool(tc)
            messages.append({"role": "tool", "tool_call_id": tc.id,
                             "content": result})
    return "（达到最大步数仍未完成，已停止）"                # 终止条件②触发


if __name__ == "__main__":
    question = " ".join(sys.argv[1:]) or "现在几点？再帮我算一下 (123+77)*3 等于多少。"
    print("用户:", question)
    print("agent:", agent_loop(question))
```

**三处与骨架版的关键差异**：

1. **`messages.append(msg)` 这一行是"记忆"的真实形态**：assistant 声明（含 `tool_calls`）
   和 `tool` 结果都进历史——下一轮请求带上全部，模型才能"看到刚才发生了什么"。
   注释掉它，模型立刻失忆（第 3 篇场景 1 让你验证过这件事）。
   顺带兑现第 4 篇的伏笔：这里 append 的是**整个 message 对象**（含 `content` /
   `reasoning_content` / `tool_calls`）——`deepseek-flash` 思考模式默认开启，而**带
   `tools` 的请求要求历史的 `reasoning_content` 完整回传，不回传会 400**；"append
   整个对象"正是官方推荐写法，本 loop 天然合规，你什么都不用改。
2. **"模型说我调完了"的判定**：`if not msg.tool_calls` ——模型只给文本、没有工具声明，
   就是完成信号；等价于第 6 篇的 `finish_reason` 检查，但更直接。
3. **校验是分层的**：本文的"轻量校验"= JSON 可解析 + 必填缺失时靠 Python `TypeError`
   兜底喂回；更严格的（按 `parameters` 逐字段校验，如 pydantic/jsonschema）留到 P7·20
   工程化——**原则现在就立住：校验失败的错，也要喂回模型让它自己修，而不是让循环崩**。

---

## 3｜四个必踩的坑与兜底

| 坑 | 长什么样 | 守门位置（上面代码里）| 处理原则 |
|---|---|---|---|
| **幻觉调用** | 模型调不存在的工具（如 `get_weather`）| `registry.get(name) is None` 分支 | 返回"工具不存在 + 可用清单"，模型通常自己改口 |
| **空参 / 坏参** | `arguments` 是空串、非法 JSON、缺必填 | `json.loads(arguments or "{}")` + `except` + 传参时 TypeError 兜底 | 错误文本喂回，让模型修正重试 |
| **工具异常** | 工具内部抛错（如 `calc("1/0")`）| `except Exception as e` 包住 `fn(**args)` | 把异常类型+信息喂回——模型能"读到错"并换招 |
| **死循环** | 模型反复调工具、原地打转 | `MAX_STEPS` 硬刹车 + `log_step` 可观察 | 宿主兜底必须永远保留（第 3 篇的"刹车"落地）|

**为什么统一处理成"喂回模型"而不是"抛异常崩掉"？** 因为 agent 的容错语义是"**把坏消息
告诉大脑，让它重新决策**"——工具报错也是"世界的一部分"。你把它咽掉或崩溃，agent 就
失去了自我修正的机会；你把它喂回去，模型往往能换个参数、换个工具、或者向用户解释。
这个模式，就是第 2 篇 utility-based"自查"在代码层的最早形态。

---

## 4｜结构化输出 vs function calling

本篇的 loop 是"行动型"用法（模型声明 → 宿主执行）。还有一类互补用法：**让模型直接
输出结构化数据**（不调任何工具，纯当"数据生成器"）：

```python
# <未实测，以官方 docs 为准>：JSON 模式（OpenAI 兼容接口普遍支持）
resp = client.chat.completions.create(
    model="deepseek-flash",
    messages=[
        {"role": "system", "content": "以 JSON 输出，字段：name(string), role(string)。"},
        {"role": "user", "content": "从这句话抽人名和角色：管理员张三应改为只读。"},
    ],
    response_format={"type": "json_object"},   # 要求模型输出合法 JSON
)
print(resp.choices[0].message.content)          # 形如 {"name": "张三", "role": "只读"}
```

| 维度 | function calling（本篇 loop）| 结构化输出（本节的 JSON 模式）|
|---|---|---|
| 模型输出什么 | "我想调 X 工具"的**声明** | 直接的**数据** |
| 谁来"执行" | 宿主执行工具（有副作用）| 没人执行；数据交给后续代码 |
| 典型场景 | agent 行动（查数据、写库）| 抽字段、打分、**生成计划**（P4·12 计划的载体就是一段 JSON）|
| 共同点 | 都是"用结构约束模型输出"——**能约束的地方，就别让它自由发挥** ||

---

## 里程碑 M1：终端跑起属于你的最小 agent

**目标**：第 3 篇的骨架 + 第 6 篇的协议 + 第 4 篇的模型 = 你自己的最小 agent。
按第 1 节的目录建好 `agent/`（`.env`、`.gitignore` 也备上），两个文件照抄上面全文。

**运行与预期输出**（LLM 输出每次略有不同，形态如下；工具层已实测）：

```text
$ cd agent && source .venv/bin/activate
$ python main.py "现在几点？再帮我算一下 (123+77)*3 等于多少。"
用户: 现在几点？再帮我算一下 (123+77)*3 等于多少。
{"step": 1, "tool_calls": ["get_current_time"], "content": ""}
{"step": 2, "tool_calls": ["calc"], "content": ""}
{"step": 3, "tool_calls": [], "content": "现在是 2026-10-16 10:24:37，(123+77)*3 = 600。"}
agent: 现在是 2026-10-16 10:24:37，(123+77)*3 = 600。
```

**M1 自检清单（六条全过才算里程碑达成）**：

- [ ] **单工具**：`python main.py "现在几点？"` → 答复含当前时间，日志出现 `get_current_time`
- [ ] **双工具复合**：跑上面的默认问题 → 日志里 `tool_calls` 至少出现两次（两个工具各一次）
- [ ] **计算正确**：`(123+77)*3` 的结果是 **600**（工具算对，非模型心算）
- [ ] **幻觉诱导**：`python main.py "请用 get_weather 工具查北京天气"` → 模型收到
      "工具不存在"后改用文字答复（**不崩、不装**——不会瞎编天气）
- [ ] **异常兜底**：`python main.py "帮我算一下 1/0"` → 工具异常被喂回，agent 不崩，
      并给出合理解释（如"除数不能为零"）
- [ ] **刹车可见**：把 `MAX_STEPS` 临时改成 `1` 再跑默认问题 → 看到"达到最大步数仍未完成"

六条里第 4、5 条最有价值——它们证明**你写的不是"顺利时才工作"的 demo，而是有兜底的
运行时**。

---

## 本篇结论

1. **M1 达成**：五阶段 loop + 真实协议 + 工具注册表 + 四种兜底 ≈ 70 行——agent 没有魔法，
   每一行都可指认（工具描述→`TOOLS_SCHEMA`、执行→`registry`、记忆→`messages`）。
2. **四坑一体兜底**：幻觉调用/空参/工具异常/死循环——统一"喂回模型 + 宿主刹车"，
   容错语义是"让大脑看到坏消息再决策"。
3. **结构化输出**是 function calling 的孪生：一个要行动、一个要数据；能约束就约束。

---

**Part 2 收口**：到这里你已拥有——loop 心智（第 3 篇）、选型判断力（第 4-5 篇）、
协议全貌（第 6 篇）、**能跑的最小 agent（第 7 篇）**。它现在是"人工智障"级的：只会
报时和算术。

**下一 Part 会暴露什么问题**：工具太玩具、接不上任何真实系统。P3 把它接上"后台管理
助手"主场景——登录、查用户、分配角色，你会立刻撞上真实世界的坑（token 过期、业务错误码、
危险写操作），第 9-10 篇逐个解决。

**等我回复「继续」。**（建议此时把上下文压缩一次——Part 2 已完整交付，认准 README
锚点即可无缝续接。）