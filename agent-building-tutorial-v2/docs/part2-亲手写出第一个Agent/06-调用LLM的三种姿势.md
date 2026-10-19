# 第 6 篇：调用 LLM 的三种姿势

> **知识地图【A2】· Part 2**
> 第 5 篇把选型地图落了地：默认方案（DeepSeek + openai SDK）的最小 agent 已经在你终端里跑通（[05-Python选型地图](05-Python选型地图.md)）。但那 20 行里你只见过一种姿势——`client.chat.completions.create(...)`。本篇把「调用 LLM」这件事摊开看全貌：同一个端点的三种姿势、三种姿势背后共享的四件必修事（key 管理 / 超时重试 / 流式 / 消息结构）、一张对比表。第 7 篇手写最小 agent 要用的零件，本篇备齐。

## 本篇要回答的问题

1. 「调用 LLM」到底是在调什么？直连 REST（姿势一）发出去的 HTTP 长什么样——URL、header、body、工具清单各是什么？响应里哪些字段要紧？
2. 官方 Python SDK（姿势二：openai / anthropic）替你沉淀了哪些协议细节？它在默认情况下替你做了哪些你没看见的事（超时、重试）？
3. 通过 agent SDK 调用（姿势三：openai-agents / pydantic-ai）时，messages 数组和循环去哪了？DeepSeek 怎么接？
4. 三种姿势都绕不开的四件必修事：API key 怎么管才不进 git、超时与重试怎么配、流式与非流式怎么选（思考模式的流式行为）、消息结构四角色与 tool_call_id 怎么关联？
5. 动手实践：裸 HTTP 与 openai SDK 对同一个工具各触发一次 tool call，把两边的请求/响应 JSON 并排对比——验证「SDK 只是协议的语法糖」。

## 1｜姿势一：直连 REST——先把协议看裸

读完本节你能理解：调用 LLM 就是一次普通的 HTTPS POST——每个字段都是你写后台接口时熟悉的那几类东西。第 1 篇那张时序图（[01-全景与学习路线地图](../part1-打开黑盒/01-全景与学习路线地图.md)）里的箭头，本节给它填上真实字节。

### 1.1 为什么从最底层学起：一个后端类比

三种姿势的关系，用你最熟的 Web 后端栈类比，一层比一层少写协议细节，但也离协议越远：

| 姿势 | 是什么 | Web 后端类比 |
|---|---|---|
| 姿势一：直连 REST | 自己拼 URL、header、JSON body，用 `curl` / 标准库 HTTP client 发出去 | 裸 TCP / 直接用 `net/http` 手写请求 |
| 姿势二：官方 SDK | 模型厂商官方包（openai / anthropic），把 URL、序列化、重试、超时打包成函数调用 | 顺手好用的 HTTP client 封装 |
| 姿势三：agent SDK | 在姿势二之上再包一层：你给「工具 + 任务」，它管 messages 和循环（agent loop，第 3 篇） | Web 框架：路由/中间件/生命周期全替你管 |

先补一个贯穿全篇的词。**endpoint（端点）**大白话：你把请求发给的那个确切网址。定义：REST API 中「资源 + 动作」对应的 URL 路径——本篇的主角是 DeepSeek 的 OpenAI 兼容端点 `https://api.deepseek.com/chat/completions`（官方首次调用页原文，2026-10-15 实测）。它不只是 DeepSeek 的地址：所有 OpenAI 兼容供应商（第 4 篇 §3.1）都在各自的 base_url 后面挂同一个形状的端点。

三种姿势不是三选一，而是同一件事的三个观察高度——**它们最终发出的都是同一份 wire JSON**（wire：线上字节，即真正出现在网络上的请求/响应原文；本篇动手实践会逐字节对比）。选哪层取决于你此刻想看见什么。

### 1.2 一个请求的完整解剖：curl 逐行读

下面这条 curl 改自官方首次调用页的示例（2026-10-15 实测原文，仅按官方 Tool Calls 指南补了 `tools` 字段），它就是姿势一的全部——没有任何魔法：

```bash
curl https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${DEEPSEEK_API_KEY}" \
  -d '{
    "model": "deepseek-flash",
    "messages": [
      {"role": "system", "content": "你是后台管理助手。本请求用于协议演示。"},
      {"role": "user", "content": "现在几点了？用一句话回答。"}
    ],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_time",
        "description": "获取当前时间（演示用，返回固定值）",
        "parameters": {"type": "object", "properties": {}}
      }
    }],
    "thinking": {"type": "disabled"},
    "stream": false
  }'
```

逐行读（每一行都是你 HTTP 课的老朋友）：

| 位置 | 字段 | 作用 |
|---|---|---|
| URL | `/chat/completions` | OpenAI 兼容的对话补全端点，POST |
| header | `Authorization: Bearer ...` | 认证。Bearer token 是 HTTP 认证的标准方案（RFC 6750）——和你后台 JWT 的 `Authorization: Bearer <token>` 一模一样 |
| body | `model` | 模型名，现行 `deepseek-flash`（第 4 篇 §1.2 考据） |
| body | `messages` | 消息数组——agent 的全部状态（§1.4 专讲） |
| body | `tools` | 工具清单：JSON Schema 描述「有什么工具、怎么调」。API Reference 对 `tools` 的定义原文："A list of tools the model may call. Currently, only functions are supported as a tool."（2026-10-15 实测） |
| body | `thinking` | 思考开关，DeepSeek 的非 OpenAI 标准字段——注意：**在 wire 层它就是个普通顶层 JSON 字段**（姿势二里它怎么变成 `extra_body`，§2.1 揭晓） |
| body | `stream` | 流式开关，`false` = 一次性返回完整 JSON（§2.4） |

注意一个与 Go 后台直觉一致的事实：**`tools` 只是把工具的「说明书」发给模型，模型不会执行任何东西**——它会返回「我想调 get_time」的意图，执行的永远是你（第 1 篇 §2.1 的三方分工）。

### 1.3 响应的字段：哪里是消息、哪里是元数据

非流式响应是一个完整 JSON（结构依官方 API Reference，2026-10-15 逐字段核验；`id`/`created`/`usage` 的具体值每次调用都不同）：

```jsonc
{
  "id": "call-mock-61167",          // 本次响应的唯一 ID
  "object": "chat.completion",
  "created": 1760000000,            // 时间戳（秒）
  "model": "deepseek-flash",
  "choices": [{
    "index": 0,
    "message": {                    // ◄─ 模型的发言：回填 messages 的就是它（第 3 篇阶段⑤）
      "role": "assistant",
      "content": null,              // 要调工具时为 null；普通回答时是文本
      "tool_calls": [{              // ◄─ 模型的「调用意图」清单
        "id": "call_mock_0001",     //    id：string，本次调用的唯一标识（回填时要用，§1.4）
        "type": "function",         //    取值当前只有 "function"（API Reference 原文）
        "function": {
          "name": "get_time",
          "arguments": "{}"         //    注意：是 JSON 字符串，不是对象——要自己 json.loads
        }
      }]
    },
    "finish_reason": "tool_calls"   // ◄─ 终止原因：循环判定的结构化信号
  }],
  "usage": { "prompt_tokens": 37, "completion_tokens": 9, "total_tokens": 46, "...": "..." }
}
```

**finish_reason 的全集**比第 1 篇出现过的两个值要多（官方 API Reference，2026-10-15 实测）：

| 取值 | 含义 | loop 该怎么处理 |
|---|---|---|
| `tool_calls` | 模型要调工具 | 执行工具、回填、继续循环 |
| `stop` | 自然收尾 | 输出最终答复，循环结束 |
| `length` | 达到 max_tokens 或上下文上限 | 警惕：内容可能被截断，不是正常完成 |
| `content_filter` | 内容被过滤 | 按业务降级处理 |
| `insufficient_system_resource` / `aborted` | 推理资源不足被中断 / 生成被打断 | 异常路径：按 §2.3 的错误类型捕获，视情况重试或降级 |

`usage` 里的 `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` / `completion_tokens_details.reasoning_tokens` 三个计费相关字段第 4 篇 §4.2 已拆过，不重复。

### 1.4 消息结构：四角色与 tool_call_id 的关联（四件必修事之一）

第 1 篇 §2.3 给过四角色一句话版、第 3 篇把 messages 定为「agent 唯一的内部状态」，这里不再重复定义；本节补的是**一次 tool call 往返中 messages 数组的完整演变**——第 7 篇的循环会把下面四个快照原样变成代码。

```
时刻 0 ─ 初始请求（1 条）              时刻 1 ─ 模型要调工具（新增 assistant，finish_reason="tool_calls"）
messages = [                          messages = [
  {role:"system", content:"…"},         {role:"system", content:"…"},
  {role:"user",   content:"现在几点…"}    {role:"user",   content:"现在几点…"},
]                                       {role:"assistant", content:null,      ◄── 新增
                                          tool_calls:[{id:"call_abc", type:"function",
                                                       function:{name:"get_time",
                                                                 arguments:"{}"}}]},
                                      ]

时刻 2 ─ 你的代码执行工具、回填结果，第二次请求（4 条；工具真正执行发生在这一步之前，第 1 篇时序图 (4)(5)）
messages = [
  …前 3 条【原样】重发（第 3 篇：每轮全量重发）…,
  {role:"tool", tool_call_id:"call_abc",        ◄── 新增：tool_call_id 必须等于上面 tool_calls[].id
   content:"\"2026-10-15 12:00:00\""},
]

时刻 3 ─ 模型基于结果收尾（finish_reason="stop"，新增最终答复）
messages = [
  …前 4 条原样…,
  {role:"assistant", content:"现在是 2026-10-15 12:00:00。"}   ◄── 新增
]
```

从这四个快照里读出三条硬规则（都是协议约束，不是风格建议）：

1. **id 成对**：`tool` 消息的 `tool_call_id` 必须对上 `assistant.tool_calls[].id`——模型靠它把结果对回到那次调用。模型一次可以返回多个 `tool_calls`，每个都得有自己的 `tool` 结果。官方示例的回填写法就是 `messages.append({"role": "tool", "tool_call_id": tool.id, "content": ...})`（DeepSeek Tool Calls 指南原文，2026-10-15 实测）。
2. **成对裁剪**：压缩/裁剪上下文时，`assistant(tool_calls)` 与它对应的 `tool` 结果必须一起动——孤立地留下半边会把模型搞糊涂（第 3 篇 §3.5 OpenClaw 的配对约束，出处在那边）。
3. **中途插入工具调用不支持**：官方 Tool Calls 指南明说——Chat Completion API 不支持在历史中间插入模型没有生成过的 tool call（Anthropic API 与 Responses API 才支持；2026-10-15 实测）。想「替模型伪造一次工具调用」的捷径在这条路线上不存在。

再加一条与 thinking 模式的交叉规则：**带 `tools` 的请求必须完整回传历史 `reasoning_content`，否则 400**——时刻 1 的新增 assistant 消息在思考模式下还要带 `reasoning_content` 字段。第 4 篇 §1.5 已逐字展开，此处只留位置。

## 2｜姿势二：官方 SDK——协议细节换工程默认

读完本节你能理解：官方 SDK 把姿势一的哪些东西收走了、收走之后**默认替你做了哪些你没点过头的事**（超时 10 分钟、自动重试 2 次），以及四件必修事里剩下的三件（key 管理 / 超时重试 / 流式）怎么落到代码上。

### 2.1 openai SDK 接 DeepSeek：三个字符串与 extra_body 的真相

**SDK（Software Development Kit，软件开发工具包）**大白话：厂商把「拼 URL、发请求、解析响应」这些每个用户都要写的代码预先写好，打包成一个 pip 包。定义到本篇语境：官方 SDK = 厂商维护的官方 client 库，openai 包（PyPI `openai 3.31.0`，requires-python ≥3.10，2026-10-15 PyPI 核验）与 anthropic 包是两家的代表；「官方」相对的第三种是 agent SDK（§3），那是另一层东西。

接 DeepSeek 只需要三个字符串（第 4 篇 §3.1 的「OpenAI 兼容」口径，官方首次调用页的 Python 示例原文同构）：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],   # 第一个字符串：key
    base_url="https://api.deepseek.com",      # 第二个字符串：端点
)
resp = client.chat.completions.create(
    model="deepseek-flash",                   # 第三个字符串：模型名
    messages=[...], tools=[...],              # 与姿势一 body 里的字段一一对应
    extra_body={"thinking": {"type": "disabled"}},
)
```

两个当场要讲透的细节：

1. **`api_key` 不会被自动读到**。`OpenAI()` 构造函数默认读的是环境变量 `OPENAI_API_KEY`（官方 README 原文：`api_key=os.environ.get("OPENAI_API_KEY")  # This is the default and can be omitted`，2026-10-15 实测）——`DEEPSEEK_API_KEY` 不在它的视野里，不显式传就直接报错。这是 DeepSeek + openai SDK 组合的第一坑。
2. **`extra_body` 是 SDK 的逃生舱**。`thinking` 不是 OpenAI 标准参数，所以 SDK 签名里没有它；`extra_body` 的官方 docstring 原文是 "Add additional JSON properties to the request"（openai-python 源码 `resources/chat/completions/completions.py`，2026-10-15 核验）——它的内容被**原样并进请求体顶层**。你在姿势一 §1.2 已经亲眼看到 `thinking` 在 wire 层就是个普通顶层字段，两个姿势在这里严丝合缝。顺带：`reasoning_effort` 是 SDK 的原生参数（签名里就有），不用走 `extra_body`。

wire 字段与 SDK 参数的对应关系（对照 §1.2 的表读）：

| wire（姿势一） | SDK 参数（姿势二） | 说明 |
|---|---|---|
| URL `/chat/completions` | `base_url` + 方法名 `chat.completions.create` | URL 拆进了对象路径 |
| header `Authorization: Bearer ...` | `api_key=...` | header 生成被收走 |
| body `model` / `messages` / `tools` | 同名参数 | 一一对应 |
| body `thinking` | `extra_body={"thinking": ...}` | 非 OpenAI 标准字段走逃生舱 |
| body `stream` | `stream=True/False` | 默认不发送（§2.4；本篇实践会实测这条差异） |

### 2.2 API key 管理：环境变量 / .env / 不进 git（四件必修事之二）

解决什么问题：key 就是钱（按 token 计费）+ 权限（能用你的配额调模型）。它一旦进了 git——哪怕后来删掉——**仍留在 git 历史里，必须当作已泄漏处理：立即吊销重发**，而不是删掉那行代码了事。

三个层级，从临时到工程：

```bash
# 层级一：环境变量（临时/一次性实验，本篇实践都用它）
export DEEPSEEK_API_KEY=sk-xxxxxxxx        # platform.deepseek.com 申请；sk-xxx 为占位
```

```python
# 层级二：.env 文件 + python-dotenv（工程默认；官方推荐）
# .env 内容（一行）：
#   DEEPSEEK_API_KEY=sk-xxxxxxxx
import os
from dotenv import load_dotenv   # pip install python-dotenv（PyPI 1.2.4，2026-10-15 核验）
load_dotenv()                    # 把 .env 内容注入环境变量，之后照常 os.environ 读
key = os.environ["DEEPSEEK_API_KEY"]
```

```gitignore
# 层级三：.gitignore 兜底（没有这条，前两层都是摆设）
.env
```

为什么 `.env` 算「比环境变量工程化」而不是「更危险」：openai 官方 README 对 key 的官方建议原文就是 "we recommend using python-dotenv to add `OPENAI_API_KEY="My API Key"` to your `.env` file **so that your API key is not stored in source control**"（2026-10-15 实测）——`.env` 的价值在于「机器上有、仓库里没有」：每台机器/每个环境放自己的 `.env`，git 里只进 `.env.example`（写占位 `sk-xxx` 的模板）。

两条配套纪律（本系列全程执行）：

1. **示例代码里 key 一律占位 `sk-xxx`**，读它的代码只写 `os.environ[...]`——第 5 篇实践里「没设 key 就干净的 `KeyError`」正是这条纪律的顺手副产品：报错发生在发出任何请求之前。
2. **配置收敛**（第 4 篇 §1.2 经验 1 的延续）：`base_url` / `model` / `api_key` 三个字符串收进代码里唯一的一个配置点。第 7 篇的手写 agent 会建一个几行的 `config.py` 专门放它。

### 2.3 超时与重试：SDK 默认在发生什么、怎么显式配（四件必修事之三）

解决什么问题：LLM API 是一次**可能要跑几十秒的外部 HTTP 调用**（思考模式一轮可长达数十秒，第 4 篇 §4.3）——不设超时，一次网络抖动就能把 agent loop 卡死在第 3 篇说的「工具超时」之前；不看重试语义，一次 429 限流可能在你的循环里变成空指针。

**先看默认值**（openai 包，README + 源码 `_constants.py` 双重核验，2026-10-15）：

| 默认项 | 值 | 出处 |
|---|---|---|
| timeout | **10 分钟**（`DEFAULT_TIMEOUT = httpx2.Timeout(timeout=600, connect=5.0)`） | README："By default requests time out after 10 minutes"；`_constants.py` |
| max_retries | **2** | README 注释 "# default is 2"；`_constants.py: DEFAULT_MAX_RETRIES = 2` |
| 会重试的错误 | 连接失败、408、409、429、≥500 | README："...are all retried by default" |
| 重试前提 | "Requests are retried only when their body can be safely resent" | README 原文 |
| 超时本身 | 也会被重试（"requests that time out are retried twice by default"） | README 原文 |

两个对 agent loop 直接有影响的默认行为：

1. **10 分钟太宽**。第 3 篇终止条件里的「工具超时」粒度是单步；LLM 调用作为「最贵的单步」，10 分钟意味着一次挂起拖死整轮。显式收窄到秒级/十秒级：`OpenAI(timeout=60.0)` 或单次 `client.with_options(timeout=5.0).chat.completions.create(...)`（README 原文示例同构）。
2. **流式请求不自动重试**。README 原文："Stream consumption is not automatically retried, because replaying a request could duplicate output already delivered to your application."——道理与第 3 篇事件流一致：内容已经发给用户了，重放就重复。要重试得自己重来（§2.4）。

超时后抛 `APITimeoutError`，连不上抛 `APIConnectionError`，服务端 4xx/5xx 抛 `APIStatusError`（带 `status_code` 属性）——README 明确让你 catch 这三类 SDK 异常而不是底层 HTTP 异常。curl 侧对应物是 `--max-time`（curl 手册：整体操作的最长秒数）。

第 9 篇会展开「重试几次之后放弃、把错误喂回模型改道」的兜底链，这里只立规则：**默认值要知道、要收窄、要在 loop 外想清楚重试上限**。

### 2.4 流式 vs 非流式：stream=True 与思考模式的流式行为（四件必修事之四）

大白话先行：**非流式像同步 RPC——等全部生成完，一口气收一个大 JSON；流式像 SSE 推送——生成多少就先收到多少，一块一块到**。

准确定义与出处：官方 README 原文 "We provide support for streaming responses using Server-Sent Events (SSE)."（2026-10-15 实测）。**SSE（Server-Sent Events）**是基于 HTTP 的单向服务器推送：响应不是一次返回，而是按事件块（**chunk**，数据块）持续下发，最终以 `data: [DONE]` 结束（DeepSeek API Reference 原文，2026-10-15 实测）。SDK 侧把参数写成 `stream=True`，循环里逐 chunk 消费：

```python
stream = client.chat.completions.create(model="deepseek-flash", messages=messages,
                                        stream=True, stream_options={"include_usage": True})
for chunk in stream:                                   # 每个 chunk 是「一小片回答」
    delta = chunk.choices[0].delta                     # delta = 本次增量（相对完整 message）
    if delta.content:
        print(delta.content, end="", flush=True)       # 逐段往外吐——打字机效果就是这么来的
```

两个容易踩的参数细节（均官方 API Reference 原文，2026-10-15 实测）：

- `stream_options` **必须与 `stream: true` 一起设置**，否则 API 返回 400；`include_usage: true` 让 usage 只出现在**最后一个 chunk**（其余 chunk 里为 null）；
- 思考模式的流式行为：**思维链先流出**——`chunk.choices[0].delta.reasoning_content` 先逐段到达，之后 `delta.content` 才开始（官方思考模式页示例代码原文，2026-10-15 实测）。给前端做「正在思考」提示（第 3 篇事件流六件套之 3）接的正是这个字段。

怎么选：

| | 非流式（stream=false） | 流式（stream=true） |
|---|---|---|
| 代码量 | 最小：一个对象全解析 | 循环消费 chunk，拼装/转发自己管 |
| 首字时延 | 高（等全部生成完） | 低（先到先显示） |
| 错误处理/重试 | 简单；超时会被 SDK 自动重试（§2.3） | 已送达内容不可重放——**SDK 不自动重试**，要自己兜 |
| usage 获取 | 响应里直接有 | 要显式 `stream_options={"include_usage": true}`，且只在末 chunk |
| 适合 | loop 内部的中间调用、批处理、测试 | 面向用户的最终输出、长回答、过程可见（思考/打字机） |

本系列第 7 篇手写 agent 用非流式（loop 内部调用，简单可判定）；「流式透传给前端」留给第 20 篇的部署形态（FastAPI+SSE）。

### 2.5 anthropic SDK 对照（一条，不展开）

两家官方包长相高度相似（同源生成器，见下），差异点才是值得记的：

```python
import os
from anthropic import Anthropic            # pip install anthropic；PyPI 1.14.0，≥3.10（2026-10-15 核验）

client = Anthropic()                       # 默认读 ANTHROPIC_API_KEY（README："can be omitted"）
msg = client.messages.create(              # 端点是 /v1/messages，不是 /chat/completions
    model="claude-opus-5-5",
    max_tokens=1024,                       # max_tokens 是必填参数（openai 包里可选）——最重要的差异
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
```

- **默认值同源**：anthropic 包 `_constants.py` 同样是 `DEFAULT_TIMEOUT` 10 分钟（`timeout=10*60, connect=5.0`）+ `DEFAULT_MAX_RETRIES = 2`（2026-10-15 源码核验）——两个包出自同一代码生成器，§2.3 的结论原样适用。
- **DeepSeek 也能用 anthropic SDK 接**：官方给出 Anthropic 兼容 base_url `https://api.deepseek.com/anthropic`（首次调用页原文，2026-10-15 实测）——「OpenAI 兼容」之外的第二条兼容线。
- 本系列不再展开：Claude 模型的调用在第 4 篇实践中给过可选形态；claude-agent-sdk 属于姿势三/agent SDK 层，第 16 篇拆解 Claude Code 时正式讲。

## 3｜姿势三：通过 agent SDK 调用——messages 一起交出去

读完本节你能理解：agent SDK 换掉的不是「调谁」而是「谁来调」——messages 数组和循环依然存在，只是从你的代码搬进了框架内部；以及为什么 DeepSeek 在这一层依然只是「OpenAI 兼容供应商」。

### 3.1 换掉的是「调用者」

第 5 篇 §5.2/§5.3 给过两个 ≤20 行的 agent SDK 版本（pydantic-ai 11 行、openai-agents 17 行），它们的调用形状是：

```python
result = agent.run_sync("现在几点了？")        # pydantic-ai：一行把整个 loop 跑完
result = await Runner.run(agent, "现在几点了？")  # openai-agents：Runner 接管循环
```

对照姿势二，被「交出去」的东西精确清点如下——第 3 篇的手写件逐一对号：

| 你不再写的 | 去了哪 | 代价 |
|---|---|---|
| `messages` 数组的创建、append、回填（§1.4 四快照） | Runner / `run_sync` 内部自建自维护 | 看不见每轮全量重发了什么，出问题要靠框架的日志/trace |
| agent loop 五阶段循环 | 同上 | 第 3 篇说它「十几行」，框架收走后就是黑盒 |
| 工具 JSON Schema 手写 | 装饰器从类型注解/docstring 生成 | Schema 质量取决于注解质量 |

但有一条不能误判：**姿势三发出的仍是同一份 wire JSON**。openai-agents 的模型层最终也是 `AsyncOpenAI(base_url=...)` 走 Chat Completions（第 5 篇 §2.3 的「适配税」那几行就是在配这个 client）——本篇实践的自检清单留了一道选做题，用姿势二同款的 `with_raw_response` 手法把 agent SDK 发出的请求抓出来对一遍。

### 3.2 DeepSeek 的接入：还是那个 OpenAI 兼容供应商

姿势三层面对 DeepSeek 的接入方式，第 5 篇 §4.2 的表格已经逐行核验过（手写 2 个字符串 / pydantic-ai 1 处前缀 / openai-agents 约 3 处适配税），本篇不重复代码，只补一句本篇视角的解读：

- pydantic-ai 的 `DeepSeekProvider` / `'deepseek:'` 前缀，本质是把「base_url + 鉴权习惯」封装成具名 provider——provider 抽象就是姿势二 `OpenAI(base_url=...)` 那两行的框架化；
- openai-agents 的 `OpenAIChatCompletionsModel(model="deepseek-flash", openai_client=client)`，则直接暴露了它的底座就是姿势二的 client——**三个姿势是叠层关系，不是三套协议**。

所以「换模型供应商」这件事在第 3 篇、本篇、第 5 篇里的答案是同一个：OpenAI 兼容让供应商可插拔（第 4 篇 §3.1），框架层只是把这个可插拔再封装了一次。

## 4｜三种姿势对比表与选型

读完本节你能拿到本篇的核心交付物：一张「看得见什么 / 看不见什么 / 写多少代码 / 适合谁」的对比表，和三句选型结论。

### 4.1 对比表

| | 姿势一：直连 REST | 姿势二：官方 SDK | 姿势三：agent SDK |
|---|---|---|---|
| 看得见什么 | **协议全貌**：每个字段、每次全量重发、每个错误码原文 | `messages` 与响应对象、参数签名；默认超时/重试有据可查（README+源码） | 任务与工具（业务语言）；`result.output` |
| 看不见什么 | 无隐藏——协议裸奔，代价是全要自己管 | HTTP 构造、序列化、重试、连接管理（文档可查、源码可翻、`with_raw_response` 可穿透） | `messages` 数组、loop、Schema 生成——全被接管（可从框架侧透出，默认不看） |
| 写多少代码 | 一个请求约 20 行 body+header；超时/重试/流式全部自管 | client 3 行 + `create()` 一次调用；超时/重试默认已带 | ≤20 行全包（第 5 篇实测 11/17 行） |
| 适合谁 | 第一次学协议；排查 wire 问题；非 Python 环境（任何语言都能发 HTTP） | **日常开发主力**；手写 loop 的底座（本系列主线） | 要快速出活、吃现成 harness 的产品场景（第 5 篇决策表） |
| 什么时候不选它 | 日常开发——每个供应商都要重写一遍样板 | 需要逐字节控制连接层时（其实也能下潜到姿势一） | 学习 loop 原理时（黑盒）；需要精确控制 messages 演变时 |

### 4.2 选型三句话

1. **学习期：姿势一进眼睛，姿势二进代码**。先看裸协议一次（本节之前你已完成），之后日常全部姿势二——本篇动手实践的顺序就是这个原则。
2. **姿势三在需要「别人写好的 harness」时上**，选型依据第 5 篇决策表不变；它的「看不见」是特性也是代价（第 5 篇 §5.6 的黑盒论）。
3. **三种姿势是叠层关系**：理解自下而上（协议→SDK→框架），使用自上而下（框架→SDK→协议）——排查问题时随时下潜一层，本篇实践给你的 `with_raw_response` 就是下潜工具。

## 动手实践：两个脚本、一次 tool call、JSON 对比等价性

目标：亲手验证本篇的中心论断「三种姿势发的是同一份 wire JSON」。脚本一用标准库裸 HTTP（姿势一，零依赖），脚本二用 openai SDK 触发**同一次 tool call**（姿势二），并抓出 SDK 实际发出/收到的字节级 JSON 与脚本一并排对比。预期结论：请求体语义完全等价、响应结构完全一致——SDK 只是协议的语法糖。

诚实声明：两个脚本在本机对**本地模拟端点**（按官方文档结构返回的小服务）实测跑通，下方「预期输出」为实测记录；对真实 DeepSeek 的调用 `<未实测，以官方文档为准>`——真实调用时响应里 `id`/`created`/`usage`/文本内容随每次调用浮动，字段结构已按官方 API Reference 逐字段核验。实测环境 Python 3.9.6 + openai 2.48.0（`with_raw_response` 接口 2.x/3.x 一致，已对 3.x 源码核验；系列要求 Python 3.11+ / openai 3.31.0，第 5 篇环境搭建直接复用）。

### 第 0 步：准备

```bash
# 环境沿用第 5 篇：venv + openai SDK（脚本一其实零依赖，脚本二才需要 openai 包）
source .venv/bin/activate    # 若按第 5 篇建过 venv；没有则 python3 -m venv .venv && source .venv/bin/activate
pip install openai -i https://pypi.tuna.tsinghua.edu.cn/simple
export DEEPSEEK_API_KEY=sk-xxxxxxxx    # key 只走环境变量（§2.2）
```

### 脚本一：裸 HTTP 触发 tool call（姿势一）

保存为 `01_rest_tool_call.py`（已实测，输出见下）。要点：URL、header、body 全部自己拼——§1.2 那条 curl 的 Python 版；用标准库 `urllib` 而非 requests，零依赖且拿到的响应体一字节不动。产物 `rest_request.json` / `rest_response.json` 是后面对比的基准：

```python
# 01_rest_tool_call.py
# 第 6 篇动手实践·脚本一：姿势一——标准库裸 HTTP 触发一次 tool call（零依赖，协议全裸）
# 依赖：无（Python 3.9+ 标准库）
# 运行：export DEEPSEEK_API_KEY=sk-xxx && python 01_rest_tool_call.py
# 产物：rest_request.json（本脚本实际发出的请求体）
#       rest_response.json（服务实际返回的响应原文）
# 说明：本机对「本地模拟端点」实测跑通全流程；真实 DeepSeek 调用 <未实测，以官方文档为准>
import json
import os
import urllib.request

URL = "https://api.deepseek.com/chat/completions"  # OpenAI 兼容端点：URL、header、body 全部自己拼

BODY = {
    "model": "deepseek-flash",  # 现行模型名（第 4 篇考据）
    "messages": [
        {"role": "system", "content": "你是后台管理助手。本请求用于协议演示。"},
        {"role": "user", "content": "现在几点了？用一句话回答。"},
    ],
    "tools": [{
        "type": "function",
        "function": {
            "name": "get_time",
            "description": "获取当前时间（演示用，返回固定值）",
            "parameters": {"type": "object", "properties": {}},
        },
    }],
    "thinking": {"type": "disabled"},  # 非思考档（第 4 篇结论）；wire 层它就是一个普通 JSON 字段
    "stream": False,  # 显式写 false：后面对比 SDK 时你会看到这个字段怎么消失的
}


def main():
    req = urllib.request.Request(
        URL,
        data=json.dumps(BODY, ensure_ascii=False).encode("utf-8"),
        headers={
            "Content-Type": "application/json",
            "Authorization": "Bearer " + os.environ["DEEPSEEK_API_KEY"],  # key 只走环境变量（§2.2）
        },
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=60) as resp:  # timeout=60：姿势一的超时要自己给（§2.3）
        wire = resp.read().decode("utf-8")  # 响应原文，一字节不动

    with open("rest_request.json", "w", encoding="utf-8") as f:
        json.dump(BODY, f, ensure_ascii=False, indent=2)
    with open("rest_response.json", "w", encoding="utf-8") as f:
        json.dump(json.loads(wire), f, ensure_ascii=False, indent=2)

    data = json.loads(wire)
    choice = data["choices"][0]
    print("== 请求体（已存 rest_request.json）==")
    print(json.dumps(BODY, ensure_ascii=False, indent=2))
    print("\n== 响应原文（已存 rest_response.json）==")
    print(json.dumps(data, ensure_ascii=False, indent=2))
    print("\n== 关键字段 ==")
    print("finish_reason:", choice["finish_reason"])
    for c in choice["message"].get("tool_calls") or []:
        print("tool_call:", c["function"]["name"],
              "arguments =", c["function"]["arguments"], "id =", c["id"])


if __name__ == "__main__":
    main()
```

### 脚本二：openai SDK 触发同一次 tool call，抓出 wire JSON 并对比（姿势二）

保存为 `02_sdk_tool_call.py`（已实测，输出见下）。关键工具是 `with_raw_response`——**SDK 的「下潜舱口」**：大白话是把「SDK 替你发的那次 HTTP」的原始请求/响应整个交回你手上；准确定位：`client.chat.completions.with_raw_response.create(...)` 返回原始响应包装对象，其 `.http_response`（原始 HTTP 响应）、`.http_response.request`（原始请求）、`.text`（响应体原文）、`.parse()`（再解析成 SDK 对象）经 openai-python 源码核验（`src/openai/_legacy_response.py`，2026-10-15）。

```python
# 02_sdk_tool_call.py
# 第 6 篇动手实践·脚本二：姿势二——openai 官方 SDK 触发同一次 tool call，抓出 wire JSON 并对比
# 依赖：pip install openai -i https://pypi.tuna.tsinghua.edu.cn/simple
# 运行：export DEEPSEEK_API_KEY=sk-xxx && python 02_sdk_tool_call.py
#       （先跑过脚本一，rest_*.json 已在同目录）
# 产物：sdk_request.json / sdk_response.json（SDK 实际发出/收到的字节级 JSON）
# 说明：with_raw_response 属性路径经 openai-python 源码核验；本机对「本地模拟端点」实测跑通；
#       真实 DeepSeek 调用 <未实测，以官方文档为准>
import json
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
    timeout=60.0,   # 默认 10 分钟太宽：agent loop 里单次调用给明确上限（§2.3）
    max_retries=2,  # 默认就是 2，写出来是让你知道它存在（§2.3）
)

KWARGS = {
    "model": "deepseek-flash",
    "messages": [
        {"role": "system", "content": "你是后台管理助手。本请求用于协议演示。"},
        {"role": "user", "content": "现在几点了？用一句话回答。"},
    ],
    "tools": [{
        "type": "function",
        "function": {
            "name": "get_time",
            "description": "获取当前时间（演示用，返回固定值）",
            "parameters": {"type": "object", "properties": {}},
        },
    }],
    # thinking 不是 OpenAI 标准字段：SDK 用 extra_body 把它原样并进请求体顶层（§2.1）
    "extra_body": {"thinking": {"type": "disabled"}},
}


def norm(v):
    """对比前的归一化：丢弃 null 字段——只比『发了什么、值是什么』，不比字节序与转义。"""
    if isinstance(v, dict):
        return {k: norm(x) for k, x in v.items() if x is not None}
    if isinstance(v, list):
        return [norm(x) for x in v]
    return v
```

```python
def value_diff(a, b, path=""):
    """请求体用：逐路径找『值不同 / 一方独有』。"""
    if isinstance(a, dict) and isinstance(b, dict):
        out = []
        for k in sorted(a.keys() | b.keys()):
            sub = (path + "." if path else "") + k
            if k not in a:
                out.append("  " + sub + "：仅 SDK 侧有 = " + repr(b[k]))
            elif k not in b:
                out.append("  " + sub + "：仅 REST 侧有 = " + repr(a[k]))
            else:
                out += value_diff(a[k], b[k], sub)
        return out
    if isinstance(a, list) and isinstance(b, list):
        if len(a) != len(b):
            return ["  " + (path or "/") + "：列表长度 %d != %d" % (len(a), len(b))]
        out = []
        for i, (x, y) in enumerate(zip(a, b)):
            out += value_diff(x, y, "%s[%d]" % (path, i))
        return out
    return [] if a == b else ["  " + (path or "/") + "：" + repr(a) + " != " + repr(b)]


def shape_diff(a, b, path=""):
    """响应体用：只比结构（键集合/列表长度），不比值——两次独立调用，值必然不同。"""
    if isinstance(a, dict) and isinstance(b, dict):
        out = []
        for k in sorted(a.keys() | b.keys()):
            sub = (path + "." if path else "") + k
            if k not in a:
                out.append("  " + sub + "：仅 SDK 响应有此键")
            elif k not in b:
                out.append("  " + sub + "：仅 REST 响应有此键")
            else:
                out += shape_diff(a[k], b[k], sub)
        return out
    if isinstance(a, list) and isinstance(b, list):
        if len(a) != len(b):
            return ["  " + (path or "/") + "：列表长度 %d != %d" % (len(a), len(b))]
        out = []
        for i, (x, y) in enumerate(zip(a, b)):
            out += shape_diff(x, y, "%s[%d]" % (path, i))
        return out
    return []


def main():
    # with_raw_response：把「SDK 替你发的那次 HTTP」的原始请求/响应整个交回你手上
    raw = client.chat.completions.with_raw_response.create(**KWARGS)
    wire_req = json.loads(raw.http_response.request.content.decode("utf-8"))
    wire_resp = json.loads(raw.text)

    with open("sdk_request.json", "w", encoding="utf-8") as f:
        json.dump(wire_req, f, ensure_ascii=False, indent=2)
    with open("sdk_response.json", "w", encoding="utf-8") as f:
        json.dump(wire_resp, f, ensure_ascii=False, indent=2)

    resp = raw.parse()  # 再解析成 SDK 对象：属性访问、类型提示等便利层从这一步才开始
    msg = resp.choices[0].message
    print("== SDK 解析后的世界（你平时看到的）==")
    print("finish_reason:", resp.choices[0].finish_reason)
    for c in (msg.tool_calls or []):
        print("tool_call:", c.function.name, "arguments =", c.function.arguments, "id =", c.id)

    print("\n== wire 对比（SDK 实际发的 vs 脚本一裸发的）==")
    with open("rest_request.json", encoding="utf-8") as f:
        rest_req = norm(json.load(f))
    with open("rest_response.json", encoding="utf-8") as f:
        rest_resp = json.load(f)
    req_diffs = value_diff(rest_req, norm(wire_req))
    print("请求体（值级）差异 %d 处：" % len(req_diffs))
    print("\n".join(req_diffs) if req_diffs else "  （无——语义完全等价）")
    resp_diffs = shape_diff(rest_resp, wire_resp)
    print("响应体（结构级）差异 %d 处：" % len(resp_diffs))
    print("\n".join(resp_diffs) if resp_diffs else "  （无——同一个接口、同一份响应结构）")


if __name__ == "__main__":
    main()
```

### 运行与预期输出

```bash
python 01_rest_tool_call.py   # 先跑姿势一
python 02_sdk_tool_call.py    # 再跑姿势二（读脚本一的产物做对比）
```

预期输出（实测记录：对按官方文档结构返回的本地模拟端点运行；真实 DeepSeek 调用时响应值浮动 `<未实测，以官方文档为准>`）。脚本一（节选，响应部分完整结构见 §1.3）：

```
== 请求体（已存 rest_request.json）==
{
  "model": "deepseek-flash",
  "messages": [
    { "role": "system", "content": "你是后台管理助手。本请求用于协议演示。" },
    { "role": "user", "content": "现在几点了？用一句话回答。" }
  ],
  "tools": [ ...同 §1.2... ],
  "thinking": { "type": "disabled" },
  "stream": false
}

== 响应原文（已存 rest_response.json）==  （结构见 §1.3，此处略）
== 关键字段 ==
finish_reason: tool_calls
tool_call: get_time arguments = {} id = call_mock_0001
```

脚本二（完整实测输出）：

```
== SDK 解析后的世界（你平时看到的）==
finish_reason: tool_calls
tool_call: get_time arguments = {} id = call_mock_0001

== wire 对比（SDK 实际发的 vs 脚本一裸发的）==
请求体（值级）差异 1 处：
  stream：仅 REST 侧有 = False
响应体（结构级）差异 0 处：
  （无——同一个接口、同一份响应结构）
```

### 看懂结果：三个亲眼所见

1. **请求体只差一处：`stream`**。SDK 对你没设的参数**根本不发送**（姿势二 §2.1 对应表里预告过）——语义等价（不设就是非流式），字节不同。除此之外，`model`/`messages`/`tools`/`thinking` 与裸 HTTP 逐字段一致，`extra_body` 并进去的 `thinking` 与姿势一手写的完全同位。
2. **响应结构零差异**。两个脚本各自独立调用，`id`/`created`/文本必然不同（所以只比结构不比值），而结构完全一致——两个姿势收的是同一个接口吐的同一份 JSON。
3. **「解析后的世界」与 wire 是两回事**。脚本二前半段 `resp.choices[0].message.tool_calls[0].function.name` 那种顺畅的属性访问，只存在于 `.parse()` 之后；wire 层是 `function.arguments` 是字符串、字段要自己掏的原始 JSON。SDK 的价值是这层便利 + §2.3 的默认工程行为，**不是另一套协议**。

### 自检清单

- [ ] 四个 json 文件都在：`rest_request.json` / `rest_response.json` / `sdk_request.json` / `sdk_response.json`；
- [ ] 亲手解释脚本二那 1 处请求差异（`stream`）为什么不影响语义；
- [ ] 把两个脚本的 `get_time` 换成带一个 string 参数的工具（如 `get_user(name)`），对比 `function.arguments` 在 wire 层的字符串形态；
- [ ] （选做）把脚本二 `KWARGS` 里加 `"reasoning_effort": "low"` 再对比——它会以顶层字段出现在 wire 请求里（§2.1：原生参数，不走 `extra_body`）；
- [ ] （选做）用第 5 篇 §5.3 的 openai-agents 17 行版，把它的 `AsyncOpenAI` client 换成 `client.chat.completions.with_raw_response` 手法抓一次请求——验证姿势三发出的也是同一份 wire JSON。

---

到这里，「调用 LLM」的三个观察高度打通了：协议层你已经看裸（§1），SDK 层的默认行为你知道去哪查证（§2），框架层知道它藏了什么、怎么下潜（§3）。零件清单也齐了——消息四角色、`tool_call_id` 关联、finish_reason 判定、key 管理、超时重试、流式。第 7 篇把这些零件装进第 3 篇的循环骨架：手写一个「看得懂每行代码」的最小 agent，跑起 Part 2 的里程碑 M1。

## 本篇结论

- 三种姿势调的是同一个端点、同一份 wire JSON：直连 REST 看协议全貌，官方 SDK 拿工程默认（超时 10 分钟/重试 2 次要显式收窄，流式不自动重试），agent SDK 把 messages 与 loop 一起接管——叠层关系，不是三套协议。
- 四件必修事：key 只走环境变量/.env 且不进 git；超时重试要知道默认、显式收敛；流式选型看「要不要过程可见」（思考模式的 `delta.reasoning_content` 先行）；messages 四快照的 `tool_call_id` 成对是协议硬约束。
- 通用工具定义、请求/响应字段在两个姿势下逐字段一致（本篇实测）——「SDK 只是协议的语法糖」现在是你亲手验证过的结论。

## 参考资料

1. [DeepSeek API Docs — Your First API Call（首次调用 API）](https://api-docs.deepseek.com/quick_start/overview) — base_url（OpenAI 格式 `https://api.deepseek.com` / Anthropic 格式 `https://api.deepseek.com/anthropic`）、官方 curl 示例（`/chat/completions` + `Bearer` + `thinking`/`reasoning_effort`/`stream` 字段）、Python 示例（`extra_body` 写法）。2026-10-15 curl 实测。
2. [DeepSeek API Docs — Tool Calls（工具调用）](https://api-docs.deepseek.com/guides/tool_calls) — `role:"tool"` + `tool_call_id` 回填的官方示例、模型不执行函数的声明、「Chat Completion API 不支持中途插入 tool calls」、strict 模式（Beta）。2026-10-15 curl 实测。
3. [DeepSeek API Docs — API Reference（create-chat-completion）](https://api-docs.deepseek.com/api/create-chat-completion) — `finish_reason` 全集（stop/length/content_filter/tool_calls/insufficient_system_resource/aborted）、`tool_calls[].id/type/function` 字段定义、`tools`/`tool_choice` 参数、`stream_options` 必须随 `stream:true`、`data: [DONE]` 流式终止。2026-10-15 curl 实测。
4. [DeepSeek API Docs — Thinking Mode（思考模式）](https://api-docs.deepseek.com/guides/thinking_mode) — 流式下 `delta.reasoning_content` 先于 `delta.content` 的官方示例、带 tools 请求必须回传 `reasoning_content` 否则 400。2026-10-15 curl 实测。
5. [openai-python 官方 README](https://github.com/openai/openai-python)（[PyPI](https://pypi.org/project/openai/) `openai 3.31.0`） — 默认 timeout 10 分钟 / `max_retries` 默认 2、重试条件（连接错误/408/409/429/≥500、body 可安全重发）、流式消费不自动重试原文、`api_key` 默认读 `OPENAI_API_KEY`、python-dotenv 官方建议、SSE 流式与 `with_options` 用法。2026-10-15 curl 实测。
6. [openai-python 源码](https://github.com/openai/openai-python/tree/main/src/openai) — `_constants.py`（`DEFAULT_TIMEOUT = httpx2.Timeout(timeout=600, connect=5.0)`、`DEFAULT_MAX_RETRIES = 2`）、`_legacy_response.py`（`LegacyAPIResponse.http_response/.http_request/.text/.parse`）、`resources/chat/completions/completions.py`（`create()` 签名含 `reasoning_effort` 原生参数、`extra_body` docstring "Add additional JSON properties to the request"）。2026-10-15 curl 核验。
7. [anthropic-sdk-python](https://github.com/anthropics/anthropic-sdk-python)（README + 源码 `_constants.py`；[PyPI](https://pypi.org/project/anthropic/) `anthropic 1.14.0`，requires-python ≥3.10） — `Anthropic()` 默认读 `ANTHROPIC_API_KEY`、`messages.create` 的 `max_tokens` 必填、与 openai 包同源的默认超时/重试常量。2026-10-15 curl 实测。
8. [curl 官方手册](https://curl.se/docs/manpage.html) — `--max-time` 等选项定义。2026-10-15 核验 200。
9. [python-dotenv（PyPI）](https://pypi.org/project/python-dotenv/) — 版本 1.2.4（2026-10-15 查询）；`.env` 用法见其文档。
10. 系列前序：[第 1 篇](../part1-打开黑盒/01-全景与学习路线地图.md)（tool call 时序图与消息角色）、[第 3 篇](03-agent-loop与上下文.md)（五阶段、messages=内部状态、配对裁剪）、[第 4 篇](04-模型选型与推理.md)（deepseek-flash 与 thinking 参数、reasoning_content 回传规则）、[第 5 篇](05-Python选型地图.md)（openai-agents/pydantic-ai 接 DeepSeek 的写法与 20 行并排）。

> 核验说明：本篇所有 API 字段、参数、默认值、URL 均于 2026-10-15 用 curl 抓取官方文档/源码逐字核验（api-docs.deepseek.com ×4 页、openai-python/anthropic-sdk-python 的 README 与源码、PyPI JSON API）；动手实践两个脚本在本机对本地模拟端点实测跑通（含 wire 抓取与对比逻辑），对真实 DeepSeek 的调用未实测并已显式标注。
