# 第 6 篇：调用 LLM 的三种姿势
### 直连 REST · 官方 SDK · agent SDK · 消息结构与工程四件套

> 知识地图【A2】· Part 2
> 读完你能理解：三种"让模型干活"的调用姿势分别能看到什么；真实的消息结构
> （system / user / assistant / tool）长什么样；以及 API key 管理、超时重试、流式选择
> 这些工程细节怎么做。

## 本篇要回答的问题

1. 调模型的**三种姿势**各是什么？分别看得见什么、什么时候用？
2. **真实的消息结构**长什么样？`tool_calls` 与工具结果怎么往返？（第 3 篇简化版的正式版）
3. **API key 怎么管**？**超时与重试**怎么设？
4. **流式 vs 非流式**怎么选？

> 动手实践会让同一件事（触发一次 tool_use）分别用"直连 REST"和"官方 SDK"做两遍，
> 打印 JSON 对比两者**结构等价**——亲眼验证"SDK 只是协议的包装"。

---

## 1｜三种姿势总览：从协议到封装的三层

| 姿势 | 你亲手碰的是哪一层 | 你能看到什么 | 什么时候用 |
|---|---|---|---|
| **① 直连 REST** | HTTP 请求/响应 | **协议全貌**：每个字段都在你眼前 | 理解原理；SDK 行为对不上预期时对照排查 |
| **② 官方 SDK**（openai / anthropic） | Python 函数 | 参数、超时、重试由 SDK 管，代码干净 | **日常开发主路径**（本系列默认） |
| **③ agent SDK**（openai-agents 等） | 配置 + 声明 | loop、工具、guardrails 全被包走 | 快速出活（第 5 篇已跑通一个） |

递进关系一句话：**① 是真实协议，② 是它的标准包装，③ 是包装再加一层 harness**。
你把 ① 看懂，②③ 的行为就都可解释；本系列主线用 ②，① 用来对照，③ 已体验过。

---

## 2｜姿势一：直连 REST（看得见协议全貌）

同一件事——"问北京天气，带上工具清单"——裸 HTTP 就是一次 POST：

```bash
curl -s https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -d '{
    "model": "deepseek-flash",
    "messages": [
      {"role": "system", "content": "当用户问天气时，调用 get_weather 工具。"},
      {"role": "user",   "content": "北京今天冷不冷？"}
    ],
    "tools": [
      {"type": "function",
       "function": {
         "name": "get_weather",
         "description": "查询指定城市的当前天气",
         "parameters": {"type": "object",
                        "properties": {"city": {"type": "string"}},
                        "required": ["city"]}}} 
    ]
  }' | python3 -m json.tool
```

（端点路径为 base_url + `/chat/completions`，字段细节以 DeepSeek 官方 docs 为准
`<以官方 docs 为准>`。本命令需 key，本环境未实测。）

响应体（省去无关字段）：

```json
{
  "id": "…", "object": "chat.completion", "model": "deepseek-flash",
  "choices": [
    {
      "index": 0,
      "finish_reason": "tool_calls",
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"北京\"}"
            }
          }
        ]
      }
    }
  ],
  "usage": {"prompt_tokens": 128, "completion_tokens": 24, "total_tokens": 152}
}
```

几个字段决生死（前三条是老朋友，后两条是 2026 的新坑）：

- `finish_reason: "tool_calls"`：这一轮**没说完**，模型要调工具——loop 据此决定"不终止、
  去执行"（第 3 篇五阶段里第 ④ 步的正式判据）；
- `message.tool_calls[].id`：这次调用的**身份证**——工具结果回填时必须带回来配型（见下）；
- `message.tool_calls[].function.arguments`：**JSON 字符串**（注意不是对象）——要
  `json.loads()` 后才能用，这就是"参数校验"该守住的位置；
- **`tool_calls` 是数组**：一轮里模型可以声明**多个**工具调用（"并行调用"）——收到多个
  就得逐个执行、逐个用 `tool_call_id` 配对回填（第 8 篇的工具注册表按这个写）；
- **`message.reasoning_content`（思考模式默认开启时出现）**：模型"想"的过程文本，与
  `content` 同级。它有个硬规则——**带 `tools` 的请求，历史的 `reasoning_content` 必须
  完整回传，否则 400**；`messages.append(整个 message 对象)` 就不会错（第 4 篇详讲）。

### 消息结构：正式版（第 3 篇简化版的"真身"）

第 3 篇的假模型里，工具结果我们用了一个自定义角色名 `tool_result`——那只是**我们循环里的
约定**。真实 OpenAI 兼容协议里，消息结构是四种角色：

| role | 谁产出 | 说明 |
|---|---|---|
| `system` | 你 | 系统提示词：身份、规则、工具使用要求 |
| `user` | 用户 | 用户输入 |
| `assistant` | 模型 | 模型的答复；**要调工具时**，这条消息带 `tool_calls`、`content` 为 null |
| `tool` | 宿主（你的代码）| **工具执行结果回填**：必须带 `tool_call_id` 指回是哪次调用（配 `tool_calls[].id`）|

把工具结果回填后，第二轮请求的消息尾巴长这样：

```json
[
  {"role": "assistant", "content": null,
   "tool_calls": [{"id": "call_abc123", "type": "function",
                   "function": {"name": "get_weather", "arguments": "{\"city\": \"北京\"}"}}]},
  {"role": "tool", "tool_call_id": "call_abc123", "content": "{\"temp_c\": 5, \"sky\": \"多云\"}"}
]
```

**两点必须记住**：① `role: "tool"` + `tool_call_id` 是"配对"关系，不配对协议就报错；
② 从下一轮请求起，**整段历史（含 tool_calls 和 tool 结果）都要带上**——这就是第 3 篇说的
"结果回填让模型看到世界"。至此，第 1 篇的时序图、第 3 篇的假模型、这里的真实协议，
三者的对应关系全部闭合。

---

## 3｜姿势二：官方 SDK（日常主路径）

同一个请求换成 `openai` SDK（我们一直用的）：

```python
import os, json
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
    timeout=30,        # 单请求超时（秒）
    max_retries=2,     # SDK 自动重试（针对 429/5xx/连接错误）
)

resp = client.chat.completions.create(
    model="deepseek-flash",
    messages=[
        {"role": "system", "content": "当用户问天气时，调用 get_weather 工具。"},
        {"role": "user",   "content": "北京今天冷不冷？"},
    ],
    tools=[{"type": "function",
            "function": {"name": "get_weather",
                         "description": "查询指定城市的当前天气",
                         "parameters": {"type": "object",
                                        "properties": {"city": {"type": "string"}},
                                        "required": ["city"]}}}],
)
print(json.dumps(resp.choices[0].message.tool_calls[0].model_dump(), ensure_ascii=False, indent=2))
```

`timeout` / `max_retries` 两个参数名以 openai SDK 官方 docs 为准；SDK 内部做的正是
"姿势一"那条 HTTP + 自动重试 + 结构化解析。**claude 官方 SDK 的形态对照**（只作了解，
模型绑定 Claude、国内可跑性有限，不进入主线）：它的消息结构略有不同（system 单独参数、
工具结果以 `user` 消息里的 `tool_result` 内容块回传、`tools` 参数直接是数组）——
各家协议"神似形不似"，以 Anthropic 官方 docs 为准。

> 顺带一个桥：DeepSeek 也提供 **Anthropic 格式端点**（`base_url` 换成
> `https://api.deepseek.com/anthropic`）——同一把 key，`anthropic` SDK 就能直连
> DeepSeek。它是"格式翻译层"：接口形状是 Anthropic 的，模型还是 DeepSeek 的。你知道
> 有这座桥即可，本系列主线继续走 OpenAI 兼容（以官方 docs 为准）。

---

## 4｜姿势三：agent SDK（把前两种都包掉）

第 5 篇你跑的那 12 行 openai-agents 就是姿势三：**它内部做的就是本节全部内容**——
帮你发请求（姿势一）、给你对象（姿势二）、再套上 loop 和工具管理。现在回看那段代码，
你应该能说出"Agent = 什么、Runner = 什么"了。

一句话收口三姿势：**①看得全 ②用得顺 ③省得多；理解靠①，日常写②，赶路用③。**

---

## 5｜工程细节：key 管理 / 超时重试 / 流式选择

### API key 管理（红线级）

- **只放环境变量 / `.env`**：第 1 篇已建好 `.env` + `.gitignore`；代码里永远写
  `os.environ["..."]`，**绝不硬编码**；
- **不进 git、不进日志、不进截图**：key 一旦 push 出去等于泄露，**立即去平台吊销重发**；
- **泄漏处理预案**：平台后台可吊销旧 key 再建新 key（轮换）——团队场景一人一把 key。

### 超时与重试

| 设置 | 建议 | 理由 |
|---|---|---|
| `timeout` | 单请求 30s 级（按任务调）| 防一个请求挂死拖死整个 loop |
| `max_retries` | 2~3 次（SDK 自动，指数退避）| **429（限流）/ 5xx / 连接错误**值得重试 |
| 不该重试的 | 4xx 参数错误（400/401/404）| 重试一百次也一样——改代码，别重试 |

重试是"网络层面的韧性"；"工具层面的韧性"（工具失败怎么喂回模型）在第 9 篇讲。

### 流式 vs 非流式

- **非流式**（默认）：等模型把整段输出生成完，一次性返回——**agent/工具调用场景首选**：
  工具参数是 JSON，本来就要等完整才能解析；
- **流式**（`stream=True`）：服务器用 SSE 分块推增量（形如 `data: {...}` 的片段，以
  `data: [DONE]` 结束），用户能**逐字看到**回答——适合面向人的聊天界面（第 20 篇 Web
  形态再上）；在 agent loop 里用流式 = 自己拼碎片，收益低、复杂度高，现阶段不做。

---

## 动手实践：两个脚本触发同一个 tool_use，对比 JSON 等价性

**目标**：同一件事（问天气 → 模型声明 `get_weather(city=北京)`），分别用 curl（姿势一）
和 SDK（姿势二）各做一次，打印 JSON——亲眼看"两份输出描述的是同一份协议"。

> 两个脚本都需 `DEEPSEEK_API_KEY`（本环境未实测，接口以官方 docs 为准）。预算友好：
> 各触发一次 tool_use 即停，不进入工具执行、不回传第二轮。

**脚本 A（直连 REST）**：新建 `rest_tool_use.sh`：

```bash
#!/usr/bin/env bash
# 第 6 篇动手实践 A：curl 直连 REST 触发一次 tool_use
set -euo pipefail
source .env 2>/dev/null || true   # 载入 DEEPSEEK_API_KEY（或在终端自行 export）

curl -s https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}" \
  -d '{
    "model": "deepseek-flash",
    "messages": [{"role": "user", "content": "北京今天冷不冷？"}],
    "tools": [{"type": "function",
               "function": {"name": "get_weather",
                            "description": "查询指定城市的当前天气",
                            "parameters": {"type": "object",
                                           "properties": {"city": {"type": "string"}},
                                           "required": ["city"]}}}]
  }' | python3 -m json.tool
```

运行：`bash rest_tool_use.sh`。预期输出（节选）：

```json
{
  "choices": [{"index": 0, "finish_reason": "tool_calls",
    "message": {"role": "assistant", "content": null,
      "tool_calls": [{"id": "call_…", "type": "function",
        "function": {"name": "get_weather", "arguments": "{\"city\": \"北京\"}"}}]}}],
  "usage": {"prompt_tokens": 97, "completion_tokens": 21, "total_tokens": 118}
}
```

**脚本 B（官方 SDK）**：新建 `sdk_tool_use.py`：

```python
"""第 6 篇动手实践 B：openai SDK 触发一次 tool_use（与前一个脚本做同一件事）"""
import json, os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI(api_key=os.environ["DEEPSEEK_API_KEY"],
                base_url="https://api.deepseek.com")

resp = client.chat.completions.create(
    model="deepseek-flash",
    messages=[{"role": "user", "content": "北京今天冷不冷？"}],
    tools=[{"type": "function",
            "function": {"name": "get_weather",
                         "description": "查询指定城市的当前天气",
                         "parameters": {"type": "object",
                                        "properties": {"city": {"type": "string"}},
                                        "required": ["city"]}}}],
)
tc = resp.choices[0].message.tool_calls[0]
print(json.dumps({"name": tc.function.name,
                  "arguments": json.loads(tc.function.arguments)},
                 ensure_ascii=False, indent=2))
```

运行：`python sdk_tool_use.py`。预期输出（节选）：

```json
{
  "name": "get_weather",
  "arguments": {"city": "北京"}
}
```

**对比等价性（本实践的核心）**：把 A 的输出看成一棵树，`choices[0].message.tool_calls[0]`
那一段，和 B 打印的 `name/arguments` **是同一份数据**——B 只是用对象属性（`tc.function.name`）
替你做了 A 里要手动做的 `json.loads` 和取下标。`id`、`created`、token 数每次运行不同，
但**结构永远等价**。结论一句话：**SDK=协议的便利贴，没魔法；直连能跑通，任何 SDK
的问题你都能定位到协议层**。

---

## 本篇结论

1. **三种姿势**是同一协议的三层：直连 REST（看得全）→ 官方 SDK（日常主路径）→
   agent SDK（连 loop 一起包）；sdk 的问题都能回到协议层定位。
2. **消息结构正式版**：system / user / assistant（可带 `tool_calls`）/ tool（带
   `tool_call_id` 配对回填）；`finish_reason: "tool_calls"` 是 loop 的正式判据；
   `tool_calls` 是数组（可并行多调用）；思考模式下 `reasoning_content` 必须随历史
   回传（append 整个 message 对象即合规）。
3. **工程四件套**再钉一遍：消息结构四角色（tool 结果带 `tool_call_id` 配对）；key 只进
   `.env` 不进 git；`timeout`+`max_retries` 只重试值得重试的；工具/agent 场景非流式、
   逐字显示才上流式。

---

**下一篇预告**：第 7 篇《手写一个"看得懂每行代码"的最小 agent》——把第 3 篇的骨架 +
第 6 篇的真实协议 + 真模型拼起来，**里程碑 M1 就在那里**：终端里跑起属于你的最小 agent。

**等我回复「继续」。**