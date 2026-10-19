# 第 14 篇：把管理助手工具升级为 MCP server
### 声明式工具 · 5 个工具搬进独立进程 · 同步 loop 接 async SDK · Part 5 第二篇

> 知识地图【E2】· Part 5 第二篇
> 读完你能：用 `@mcp.tool()` 声明式定义工具（schema 从类型注解自动生成）；把 P3 的
> `tools.py` 改造成 stdio MCP server；在同步 agent loop 里接入异步 SDK（后台线程桥接）；
> 说清 P3 的五项工程设计（token / 重登 / 错误整形 / 确认钩子 / 审计）在 MCP 架构下各去
> 了哪里；动手：完整目录跑通，同一句话验证直连版与 MCP 版等价。

## 本篇要回答的问题

1. `@mcp.tool()` 声明式定义 vs P3 手写 `TOOLS_SCHEMA`，到底省掉了什么？
2. 5 个工具搬进独立 server 进程后，token、401 重登、错误整形分别在哪运行？
3. 确认钩子和审计该留在宿主还是搬进 server？——P3「装在宿主层」的原则接受检验。
4. agent 端从"import 工具函数"改成"MCP client 连接"：同步 loop 怎么接 async SDK？
5. 动手：完整目录（宿主 + server + 模拟后台）跑通，同一句话验证两版等价。

---

## 0｜先纠一个名：FastMCP 已更名为 MCPServer

本系列第 13 篇的生态表里，Python SDK 写的是 `mcp` 2.3.0。写本篇前敲定代码时，
官方文档第一眼就泼了盆冷水：**2025 年满世界的 `FastMCP`，在 2.x 里已更名为 `MCPServer`**。

```python
# ❌ 2025 年的写法（mcp 2.x 里直接 ModuleNotFoundError）
from mcp.server.fastmcp import FastMCP

# ✅ 2026 年的写法（mcp 2.3.0 官方文档）
from mcp.server import MCPServer
```

官方 migration 文档的原话是"renamed to better reflect its role as the main server
class in the SDK"——改名的理由不重要，重要的是这就是学"活着的技术"的日常：**先以官方
Get started 为准，再参考网上的教程**。本篇所有代码按 2.3.0 的现名编写。

> 核对记录（2026-10-16）：PyPI `mcp` **2.3.0**（版本线 2.0.0 → 2.3.0）；纯库用法
> `pip install mcp` 即可，`mcp dev/run/install` 命令行工具来自 `mcp[cli]` extra；
> 要求 **Python ≥ 3.11**（官方仓库 pyproject 实测——PyPI 元数据尚写 3.10，以后者为准
> 有疑时以 pyproject 为准）。`MCPServer` 官方两处文档分别写作
> `from mcp.server import MCPServer`（first-steps）与 `mcp.server.mcpserver`
> （migration）——同一个类，本篇用前者。

---

## 1｜声明式：schema 不再手写

先回到 P3 的现场。当时为了让模型认识 5 个工具，我们手写了这样的东西（截取 2 个）：

```python
TOOLS_SCHEMA += [
    {"type": "function", "function": {
        "name": "list_users",
        "description": "按用户名或姓名查询后台用户，返回 user_id 等信息。用户提到人名时先用本工具把名字转换成 user_id。必须先登录。",
        "parameters": {"type": "object",
                       "properties": {"keyword": {"type": "string",
                                                  "description": "用户名或姓名，可为空字符串"}},
                       "required": []}}},
    ...
]
```

一个问题当时没细说：**同一件事写了两遍**——函数的 docstring 给人看（"按用户名或姓名查用户…"），
`TOOLS_SCHEMA` 里的 description 给模型看；参数类型在函数签名上有一遍，schema 里又写一遍。
两处文案会漂移（改了 docstring 忘了 schema），而这种漂移对模型来说就是"工具说明书对不上实物"。

MCP SDK 把这两遍合成了一遍。同一个 `list_users`，MCP 版：

```python
@mcp.tool()
def list_users(keyword: str = "") -> str:
    """按用户名或姓名查询后台用户，返回 user_id 等信息。用户提到人名时先用本工具把名字转换成 user_id。必须先登录。"""
    return _shape(_request("GET", "/users", params={"keyword": keyword}))
```

官方文档一句话总结：*"no JSON Schema (`a: int, b: int` **is** the schema), no request
parsing, no validation code, no protocol handling."* SDK 从函数上读三样东西：

| SDK 读什么 | 来自哪里 | 对应 P3 手写的 |
|---|---|---|
| 工具名 | 函数名 | `"name": "list_users"` |
| description | docstring | `"description": "…"` |
| 参数 schema | **类型注解**（+默认值） | `"parameters": {…}` 整块 |

类型注解不是注释，是**契约**。官方原文："Type hints aren't documentation here. They
are the **contract**."——客户端发一个 `{"keyword": 123}`，SDK 在进入你的函数**之前**就
拒掉，模型会收到一条能读懂、能自修正的参数错误。P3 里我们没有这层校验（模型传错类型，
要等 `requests` 报错才暴露，还多烧一个回合）。

### 参数可选性 = Python 默认值

官方 tools 文档的规则：`limit: int`（无默认值）→ schema 里进 `"required"`；
`limit: int = 10` → 移出 required、并加上 `"default": 10`。对应我们的
`keyword: str = ""`：`required` 自动为空——与 P3 手写的 `"required": []` 完全一致。
**Python 怎么写，契约就怎么长**。

> 实践里你会看到 schema 的实物：MCP Inspector（`mcp dev server.py`）把每个参数渲染成
> 表单输入框——那份表单就是 `tools/list` 返回的 `inputSchema` 渲的，也就是 P3 手写、
> 现在自动生成的东西。
>
> 进阶留一条线：`@mcp.tool()` 也接受 `title=` / `annotations=` 覆盖。其中 `annotations`
> 在第 4 节有一场和"确认钩子"的对质。

---

## 2｜搬家：tools.py → 独立 server 进程

本篇完成后，目录长这样：

```text
agent-building-tutorial/
├── agent/                     # 宿主：我们写的 agent 程序
│   ├── main.py                # loop 不变；接线方式变了（第 3 节）
│   ├── mcp_tools.py           # 新增：同步外壳（第 3 节）
│   ├── audit.py               # 新增：宿主侧审计（第 4 节解释为什么拆出来）
│   └── .env                   # 凭证等（不进 git）
├── admin_mcp_server/          # 新增：MCP server（P3 的 tools.py 搬家到这里）
│   └── server.py
└── target_service/            # P3 的 FastAPI 模拟后台，原样不动
```

**server.py 全文**——它就是 P3·9 的 `tools.py`，加三处改造（下面逐段标出），其余照抄：

```python
"""admin_mcp_server/server.py —— 后台管理 MCP server（stdio）。

第 14 篇：P3 的 tools.py 搬家到独立进程。token、重登、错误整形全部原样，
只是"执行者"从宿主进程内，变成了宿主拉起的子进程。
"""
import json
import os
import time

import requests
from mcp.server import MCPServer

# 改造点 ①：配置从环境变量读（默认值与原常量一致）
API_BASE = os.environ.get("ADMIN_API_BASE", "http://127.0.0.1:8000/api/v1")
AUDIT_FILE = os.environ.get("ADMIN_AUDIT_FILE", "audit_server.jsonl")
_TIMEOUT = 10
_token = None                 # 登录 token：现在只生活在 server 进程内存（第 4 节）
_creds = None                 # 供 401 自动重登的凭证

mcp = MCPServer("admin-mcp-server")   # 改造点 ②：server 实例（这个名字会亮在客户端 UI 里）


def audit(event):
    """审计日志：每行一条 JSON 追加落盘——给"事后的人"看，不是给模型看。"""
    line = {"ts": time.strftime("%Y-%m-%d %H:%M:%S"), **event}
    with open(AUDIT_FILE, "a", encoding="utf-8") as f:
        f.write(json.dumps(line, ensure_ascii=False) + "\n")


def _do_login(username, password):
    global _token, _creds
    resp = requests.post(f"{API_BASE}/login",
                         json={"username": username, "password": password},
                         timeout=_TIMEOUT)
    if resp.status_code != 200:
        return f"登录失败：HTTP {resp.status_code} {resp.text}"
    _token = resp.json()["data"]["token"]
    _creds = (username, password)
    return "登录成功，token 已在本进程内存持有（有效期 1 小时）。"


def _auth_headers():
    return {"Authorization": f"Bearer {_token}"}


def _request(method, path, **kw):
    """统一请求入口：自动带 token；曾登录过且收到 401 → 自动重登一次再重试。"""
    url = f"{API_BASE}{path}"
    resp = requests.request(method, url, headers=_auth_headers(), timeout=_TIMEOUT, **kw)
    if resp.status_code == 401 and _creds:
        audit({"event": "relogin", "reason": "got 401, relogin and retry once"})
        _do_login(*_creds)
        resp = requests.request(method, url, headers=_auth_headers(), timeout=_TIMEOUT, **kw)
    return resp


def _shape(resp):
    """把响应统一塑成"给模型的文本"：成功给 data；失败给 状态码+body+处置提示。"""
    if resp.status_code == 200:
        return json.dumps(resp.json()["data"], ensure_ascii=False)
    hint = {400: "请求参数不合法，修正后重试",
            401: "认证失败（自动重登也没救回来，检查凭证）",
            404: "目标不存在，不要重试",
            500: "服务端错误，可稍后重试"}.get(resp.status_code, "未知错误")
    return f"调用失败：HTTP {resp.status_code}（{hint}）{resp.text}"


# 改造点 ③：每个工具加 @mcp.tool() + 类型注解；
# 原来写在 TOOLS_SCHEMA 里的 description，搬进 docstring（那才是模型看到的那句）
@mcp.tool()
def login(username: str, password: str) -> str:
    """登录后台管理系统。调用任何其他后台工具前必须先登录。幂等：已有 token 时直接复用。"""
    if _token:
        return "已登录（复用现有 token）。"
    return _do_login(username, password)


@mcp.tool()
def list_users(keyword: str = "") -> str:
    """按用户名或姓名查询后台用户，返回 user_id 等信息。用户提到人名时先用本工具把名字转换成 user_id。必须先登录。"""
    return _shape(_request("GET", "/users", params={"keyword": keyword}))


@mcp.tool()
def list_roles() -> str:
    """查询后台全部角色目录（role_id/role_code/role_name）。用户提到角色名时先用本工具把角色名转换成 role_id。必须先登录。"""
    return _shape(_request("GET", "/roles"))


@mcp.tool()
def get_user_roles(user_id: str) -> str:
    """查询指定用户当前拥有的角色列表。必须先登录。"""
    return _shape(_request("GET", f"/users/{user_id}/roles"))


@mcp.tool()
def assign_role(user_id: str, role_id: str) -> str:
    """给指定用户分配一个角色（写操作，幂等：已有该角色则不变更）。必须先登录；role_id 需先经 list_roles 查得。"""
    resp = _request("POST", f"/users/{user_id}/roles", json={"role_id": role_id})
    if resp.status_code == 200:
        return "分配成功：" + json.dumps(resp.json()["data"], ensure_ascii=False)
    return _shape(resp)


if __name__ == "__main__":
    mcp.run()          # 默认 stdio：被宿主当子进程拉起，stdin/stdout 就是协议通道
```

对照 P3·9 的 tools.py：`audit` / `_do_login` / `_auth_headers` / `_request` / `_shape`
**逐字未改**；删掉的是 `registry.update(...)` 和整个 `TOOLS_SCHEMA` 块——那两段的职责，
被协议吃掉了（第 3 节在宿主端补回来，但不用手写）。

### ⚠️ 三个子进程世界的规矩

**① 子进程不继承你的环境变量。** 官方文档明确：stdio server 只拿到一份最小白名单
（`HOME` / `LOGNAME` / `PATH` / `SHELL` / `TERM` / `USER`），"so nothing sensitive leaks
into a process you may not have written"——**这是安全默认值，不是 bug**。这正是我们把
`API_BASE` 改成从 env 读的原因：要传，得用 `env=` 显式传（第 3 节）。反过来想：以后你
挂别人写的 server，你的密钥也不会被它顺走——这条规则双向保护。

**② stdout 是协议通道，不是你的屏幕。** JSON-RPC 消息走 stdout；一句 `print()` 混进去
就是往协议里掺垃圾。官方 SDK 会把"已 flush 的 stdout 输出"改道到 stderr 兜底，但正确
习惯是：**日志走 logging（它写 stderr）**。我们的 server 本来就没有 print，audit 写文件
——搬家后这两条习惯直接受益。

**③ `python server.py` 直接跑会"安静地卡住"**——那不是死机，是它在等 stdin 上的第一条
JSON-RPC 消息（官方 run 文档原样演示过这个现象："Nothing prints, and it doesn't return.
It is waiting on stdin for a host to speak first."）。这份"安静"就是 stdio 的形态。

### token 的隔离升级

P3·8 立过一条规矩：**token 存在进程内存，不进对话历史**——模型从头到尾看不到 token 原文。
搬家之后，这条规矩自动升了一个档：

```text
P3（进程内）：  宿主进程 = agent 逻辑 + token + 工具实现   ← token 与"模型能影响的一切"同屋
P5（跨进程）：  宿主进程 = agent 逻辑                     ← token 不在这
                server 进程 = token + 工具实现            ← 单独一间房
```

宿主进程里现在连 `_token` 这个变量都不存在了——**敏感状态被关进了更小的盒子**。这正是
第 13 篇"server 即权限边界"的物理形态：边界不只"暴露哪些工具"，还包括"哪些秘密连宿主
都摸不到"。

---

## 3｜宿主端：registry → MCP client（同步外壳）

改造后的数据流（对照 P2·7 那张"loop 调用本地函数"的图看）：

```text
agent/main.py（同步）
   └→ MCPToolbox.call()（同步外壳）
        └→ run_coroutine_threadsafe → 后台线程里的 asyncio 事件循环
             └→ Client（官方 SDK，async）
                  └→ stdio（JSON-RPC over 子进程管道）
                       └→ server.py 的 @mcp.tool() 函数
```

**先解决一个现实矛盾：SDK 是 async 的，P2·7 的 loop 是同步的。** 官方 `Client` 的用法是
`async with Client(...) as client:`——官方原话："async with is the lifecycle. Entering it
connects and negotiates; leaving it disconnects."。连接的生命周期 = 进程的生命周期，
**不是"一次调用一次连接"**。三条路线：

| 路线 | 结论 |
|---|---|
| ① 把整个 loop 改成 async | loop 大改，且后面每篇示例都要背着 asyncio 走——成本高 |
| ② 每次调用 `asyncio.run(…)` 新起一次连接 | ❌ 反模式：握手/清单每次都重来，会话本来该长活 |
| ③ **后台线程常驻事件循环 + 同步投递** | ✅ 本篇选择：外壳 ~70 行（含注释），loop 一行不改 |

第 ③ 条的实现就是新增文件 `agent/mcp_tools.py`：

```python
"""mcp_tools.py —— 把 MCP server 的工具，接进同步 agent loop 的同步外壳。

SDK 的 Client 是 async 的；我们的 loop 是同步的。桥接方式：后台线程跑一个
私有 asyncio 事件循环，所有协议调用经 run_coroutine_threadsafe 投进去执行，
本线程阻塞等结果——对 loop 而言，它就是一个普通的同步对象。
"""
import asyncio
import threading

from mcp import Client, StdioServerParameters
from mcp.types import TextContent


class MCPToolbox:
    def __init__(self, command, args, env=None):
        self._params = StdioServerParameters(command=command, args=args, env=env)
        # 启动后台事件循环线程（daemon：主线程退出时自动收摊）
        self._loop = asyncio.new_event_loop()
        threading.Thread(target=self._loop.run_forever, daemon=True).start()
        # 长驻协程持有 Client 的 async with —— 进入/退出必须是同一个任务（见下文）
        self._ready = threading.Event()
        self._stop = None
        self._run_future = asyncio.run_coroutine_threadsafe(self._serve(), self._loop)
        if not self._ready.wait(timeout=15):
            err = self._run_future.exception() if self._run_future.done() else None
            raise RuntimeError(f"MCP server 启动/连接失败：{err or '连接超时'}")
        # 启动时拉一次 tools/list —— 相当于 P3「加载 TOOLS_SCHEMA」的动作
        self.tools = self._sync(self._list_tools())

    async def _serve(self):
        """长驻协程：整个生命周期持有 Client（连接 = 进程寿命）。"""
        async with Client(self._params) as client:
            self._client = client
            self._stop = asyncio.Event()
            self._ready.set()
            await self._stop.wait()          # 挂起，等 close() 喊停

    async def _list_tools(self):
        result = await self._client.list_tools()
        return result.tools

    def _sync(self, coro):
        """把协程投递到后台事件循环执行，同步等结果（超时兜底防死等）。"""
        return asyncio.run_coroutine_threadsafe(coro, self._loop).result(timeout=30)

    def tools_schema(self):
        """把 server 的 tools/list 转成 DeepSeek/OpenAI 的 tools 参数格式。"""
        return [{"type": "function", "function": {
                    "name": t.name,
                    "description": t.description or "",
                    "parameters": t.input_schema}}       # 官方就是 JSON Schema，直通
            for t in self.tools]

    def tool_names(self):
        return {t.name for t in self.tools}

    def call(self, name, args):
        """同步调用工具；返回"给模型的文本"——输出与 P3 run_tool 同构。"""
        result = self._sync(self._client.call_tool(name, args))
        texts = [b.text for b in result.content if isinstance(b, TextContent)]
        body = "\n".join(texts)
        if result.is_error:                       # 工具报错也是"数据"（第 7/9 篇同一铁律）
            return f"工具执行出错：{body}"
        return body

    def close(self):
        """退出时调用：喊停长驻协程（退出 async with → SDK 负责关掉子进程）。"""
        if self._stop is not None:
            self._loop.call_soon_threadsafe(self._stop.set)
            self._run_future.result(timeout=10)
        self._loop.call_soon_threadsafe(self._loop.stop)
```

四个值得停一下的设计点：

- **为什么"进出必须同一个任务"值得小心**：SDK 内部用任务组管理 stdin/stdout 的读写
  协程，这类资源要求"进入的协程"和"退出的协程"是同一个。所以外壳不是"手动
  connect()/close()"，而是**一个长驻协程把 `async with` 从头包到尾**，`close()` 只是
  让它自然退出，SDK 再负责收尾（官方原话："Leaving the block shuts the subprocess
  down: close stdin, wait, kill if it lingers. You never clean it up yourself."）。
- **`isinstance` 那道过滤**：`result.content` 是内容块列表（Text / Image / Audio…），
  官方文档明确要先用 `isinstance` 收窄、再读 `.text`——这不是洁癖，是类型系统在防你
  "把图片当文本读"。
- **`input_schema` 直通**：server 给的 schema 本来就是 JSON Schema（2020-12），和 P3
  手写的 `parameters` 同构——所以 `tools_schema()` 只是**换了个信封**
  （`{"type": "function", "function": {...}}`），字段一个没改。这解释了为什么"换协议"
  没有动 loop 里任何一行。
- **`structured_content` 是什么？** 官方 `CallToolResult` 还有这个字段（给**代码**读的
  结构化版）。我们的 loop 只需要"给模型的文本"，只用 `content`——多出来的那一半，第 13
  篇对照表里没有对应物，等用到再学。

### main.py 的改动：三处 + 一个收尾

```python
# ① 顶部 import：删掉 from tools import TOOLS_SCHEMA, registry, audit，换成：
import sys
from audit import audit
from mcp_tools import MCPToolbox

# ② 启动时连上 server（会拉起子进程），工具清单从 server 拉取：
_HERE = os.path.dirname(os.path.abspath(__file__))
toolbox = MCPToolbox(
    command=sys.executable,        # 同一个解释器 = 同一个 venv（server 要用 mcp/requests）
    args=[os.path.join(_HERE, "..", "admin_mcp_server", "server.py")],
)
TOOLS_SCHEMA = toolbox.tools_schema()   # 这行之后，下面所有代码看到的工具和 P3 一模一样

# ③ run_tool 整段替换（agent_loop 及其余代码一行不动）：
def run_tool(tc):
    """⑤ 执行一个 tool_call；任何错误都变成文本喂回模型，绝不把循环搞崩。"""
    name = tc.function.name
    try:
        args = json.loads(tc.function.arguments or "{}")
    except json.JSONDecodeError as e:
        return f"参数不是合法 JSON（{e}）。请修正后重试。"
    if name not in toolbox.tool_names():
        return f"工具不存在：{name}。可用工具：{sorted(toolbox.tool_names())}"
    if name in DANGEROUS_TOOLS:                   # 护栏：危险操作先过确认钩子（原样保留）
        if not confirm(name, args):
            audit({"event": "rejected", "tool": name, "args": args})
            return "用户拒绝了本次操作，未执行。请向用户说明未做任何变更，不要重试。"
    audit({"event": "tool_call", "tool": name, "args": args})
    try:
        result = toolbox.call(name, args)         # ← 唯一的实质变化：本地字典调用 → MCP 调用
    except Exception as e:                        # 兜底网：server 挂了等一切未处理错误
        result = f"工具执行出错：{type(e).__name__}: {e}。如参数有误请修正后重试。"
    audit({"event": "tool_result", "tool": name, "result": result[:120]})
    return result
```

对比 P3·9 的 run_tool：`registry.get(name)` 换成 `toolbox.tool_names()` 判断，
`str(fn(**args))` 换成 `toolbox.call(name, args)`——**两行**。`DANGEROUS_TOOLS`、
`confirm`、audit、兜底 except 全部原样。所谓"agent 端改造"，改的就是这两行 + 工具
从哪来；`agent_loop`（模型对话循环）本身零改动。

收尾别忘了（脚本退出前）：

```python
try:
    ...                    # 原有的 main 流程
finally:
    toolbox.close()        # 停 server 子进程
```

### 顺手兑现 13 篇的对照表

第 13 篇给过一张"心智模型对照表"，本节逐一落到实处：

| 13 篇说的对应物 | 本篇代码里的实体 |
|---|---|
| `TOOLS_SCHEMA` ↔ `tools/list` | `TOOLS_SCHEMA = toolbox.tools_schema()`（从 server 拉、换信封） |
| `registry` ↔ server 进程 | `admin_mcp_server/server.py` 里那 5 个 `@mcp.tool()` 函数 |
| `run_tool(name, args)` ↔ `tools/call` | `toolbox.call(name, args)` |
| 函数返回值 ↔ `result.content` | 拼接 `b.text`；`is_error` 分支同 P3 兜底网 |

---

## 4｜P3 五设计点体检：MCP 架构下各去哪了？

搬家最值钱的部分在这里——把 P3 熬出来的工程设计逐条放进新架构体检，"升了 / 死了 / 搬家了"
一目了然：

| # | P3 设计点 | P3 位置 | MCP 版位置 | 判定 |
|---|---|---|---|---|
| 1 | 登录当工具（模型可重试/换凭证） | tools.py 工具 | server 工具 | **不变**——协议不吃语义 |
| 2 | token 进程内存、不进上下文 | 宿主进程内存 | **server 进程内存** | **升级**：宿主不再持有 |
| 3 | 401 自动重登（只重试一次） | `_request` | server `_request` | 位置不变；**宿主侧完全无感**（连审计都看不到） |
| 4 | 错误整形 `_shape`（状态码+hint+body） | 工具实现 | server 工具实现 | **不变**——错误文本是"工具的一部分" |
| 5 | 确认钩子（DANGEROUS_TOOLS+confirm） | 宿主 run_tool | **仍然宿主 run_tool** | **铁律验证**——见下 |
| 6 | 审计 audit.jsonl | 宿主 tools.py | **拆两处**：宿主 + server | 新知识——见下 |

**⑤ 确认钩子为什么必须留在宿主？** 这是本篇最重要的一条。MCP 其实给了它一个"近亲"：
`annotations`——工具可以声明 `read_only_hint` / `destructive_hint` 等**行为提示**，
好客户端会据此决定"要不要先问用户"。但官方紧接着就是一句警告：

> *They are hints, not security. Never rely on a client honouring them.*
> （它们是提示，不是安全机制。永远不要假定客户端会遵守。）

（给我们的 `assign_role` 加上 `annotations=ToolAnnotations(destructive_hint=True)`——
`from mcp.types import ToolAnnotations`——是合理的锦上添花，让别家客户端更聪明地对待它。
但"提示"拦不住任何东西。）

**拦截必须发生在决策的地方**。在 MCP 架构里：

```text
模型决定调 assign_role
   ↓
宿主 run_tool 检查 DANGEROUS_TOOLS → confirm() 问人 ← 唯一能"拒绝执行"的位置
   ↓（批准后）
tools/call 到达 server        ← server 根本不知道这次调用曾被 question；被拒的调用它永远收不到
```

P3·9 说过"确认钩子装在宿主层，因为工具只管做事、宿主管政策"。换架构后**结论一字未改**
——"政策层"本来就该是"离用户最近的那层"。何况第 13 篇说过：server 将来可能不是你写的、
是别人家的——你还能把钩子装到别人的进程里去吗？

**⑥ 审计为什么拆成两处？** 因为**审计跟着"事件在哪个进程发生"走**：

- **宿主侧**（`agent/audit.py`，内容就是把 P3 的 `audit()` 抄过来）：`tool_call` /
  `tool_result` / `rejected`——决定发生在这里。特别注意 `rejected`：**被宿主拦下的
  调用，server 永远看不到**——如果审计只在 server 侧，所有"拒绝"就都丢了。宿主日志
  是它们唯一的记录点。
- **server 侧**（`server.py` 里的 `audit()`，写 `audit_server.jsonl`）：`relogin`——
  重登发生在 server 内，对宿主**完全透明**（P3·9 夸的"模型无感"，MCP 版升级成"连宿主
  都无感"），所以它只能自己记。

两份日志文件名不同（`audit.jsonl` / `audit_server.jsonl`），原因很实际：两个进程工作
目录相同，同名会互相踩。这对应分布式审计的常见形态——每个进程记自己的、集中式服务
聚合（P7 展开）。`.gitignore` 记得补一行。

---

## 5｜动手实践：同一句话，验证两版等价

> **实测状态说明**：本节代码按官方文档（python-sdk 2.3.0 的 migration / get-started /
> client / tools / run / transports 页）逐条核对编写；编写环境为 Python 3.9 且未安装
> `mcp`，**未实跑**（本篇代码依赖 Python ≥ 3.11 + `mcp` 实跑环境，请以你的运行结果为准；
> 若 API 有出入，对照 `python -c "import mcp; print(mcp.__file__)"` 旁的源码与官方文档）。
> 这也是"2.x 更名"这类事实的第一手核对方式。

### 准备

```bash
# agent 的 venv 需 Python ≥ 3.11，然后：
pip install mcp          # server 与宿主的共同依赖；requests 此前已有
```

（想用 Inspector 可视化调试，再加 cli extra：`pip install "mcp[cli]"`，
然后 `mcp dev admin_mcp_server/server.py`，浏览器里会给每个工具渲染调用表单。）

### 步骤一：先跑一遍直连版，记下基线

```bash
# 终端 1：模拟后台（P3·8 的 target_service，原样）
cd target_service && uvicorn main:app --port 8000

# 终端 2：直连版 —— 改造前，先留一份基线输出
cd agent && python main.py "给张三加上运营"
```

走到确认提示输入 `y`，记下整段过程（模型说了什么、工具调用顺序、audit.jsonl 内容）。

### 步骤二：改造

1. 新建 `admin_mcp_server/server.py`（第 2 节全文）；
2. 新建 `agent/mcp_tools.py`（第 3 节全文）；`agent/audit.py`（把 P3 的 `audit()` 抄过来
   ——拆分理由见第 4 节）；
3. `agent/main.py`：按第 3 节改三处 + `finally: toolbox.close()`；
4. `.gitignore` 加一行 `audit_server.jsonl`。

### 步骤三：server 冒烟（可选但推荐）

```bash
mcp dev admin_mcp_server/server.py      # 打开它打印的 Inspector 地址
```

Tools 标签应看到 **5 个工具**，点开 `assign_role`——参数的输入表单就是自动生成的
`inputSchema`（对照 §1：手写 schema 的时代结束了）。

### 步骤四：同一句话，MCP 版

```bash
cd agent && python main.py "给张三加上运营"
```

**预期（与直连版对照）**：

| 观察点 | 预期 |
|---|---|
| 对话体验 | 与直连版一致：先 login → 查 user_id → 查 role_id → confirm 提示 → 汇报结果 |
| 进程树 | agent 进程下多一个 `python …/server.py` 子进程（`ps -ef | grep server.py`） |
| `audit.jsonl`（宿主） | `tool_call` / `tool_result` / `rejected`——与直连版**同构** |
| `audit_server.jsonl`（server） | 平时为空；发生重登时才有 `relogin` 行 |

### 步骤五：两个 MCP 专属验证（直连版做不到的）

1. **N+M 兑现**：把同一个 server 挂给 Claude Code——

   ```bash
   claude mcp add admin -- /path/to/venv/bin/python /绝对路径/admin_mcp_server/server.py
   ```

   （python 写 venv 的绝对路径——Claude Code 起子进程时不走你的 `source activate`。）
   然后在 Claude Code 里说"列出张三的角色"——它用的就是你刚写的 server。**同一个
   server，两个宿主**（我们的 agent + Claude Code），这就是第 13 篇 §1 的 N+M。
2. **隔离可视化**：在 `agent/` 里翻遍代码也找不到 `_token`——因为它只活在 server
   进程里（对照 §2 末的"更小的盒子"）。

### 自检清单（六条）

- [ ] **等价性**：步骤四的对话体验与步骤一直连版一致，结果数据一致
- [ ] **确认钩子仍生效**：输 `n` → agent 报"未执行"，且 `audit_server.jsonl` 里**没有**
      这次 assign_role 记录（被拒的调用 server 看不到）——第 4 节结论的实证
- [ ] **审计双轨**：宿主 `audit.jsonl` 三类事件齐全；用 P3·9 的"手动破坏 token"技巧
      触发一次重登，`audit_server.jsonl` 出现 `relogin`
- [ ] **schema 自动生成**：Inspector 里 5 个工具的参数与 §1 结论一致（`keyword` 可选）
- [ ] **loop 未动**：`git diff agent/main.py` 中 `agent_loop` 函数零改动
- [ ] **多宿主**：Claude Code 经 `claude mcp add` 能调用同一 server

---

## 本篇结论

1. **声明式红利**：`@mcp.tool()` + 类型注解 + docstring 三件套，替换掉手写
   `TOOLS_SCHEMA`——schema 从"抄一遍"变成"生成"（P3 三十多行 → 零行手写），还白得了
   参数校验。注意 2026 事实：2.x 起 `FastMCP` 已更名 `MCPServer`，旧导入直接报错。
2. **搬家不搬设计**：token 内存持有 / 401 重登 / `_shape` 全部随工具进 server（token
   隔离还升级成"宿主都摸不到"）；**确认钩子留在宿主、审计拆成两处**——"政策离用户最近、
   审计跟事件走"两条原则在跨进程架构下被原样验证。
3. **同步 loop 接 async SDK 有成熟套路**：后台线程常驻事件循环 + 长驻协程持有
   `async with` + `run_coroutine_threadsafe` 同步投递——约 70 行外壳（含注释），`agent_loop`
   一行不改。

---

**下一篇预告**：第 15 篇《直连 vs MCP——选型与 Go 后台桥接》（Part 5 收口 · 里程碑 M4）：
六维对比表给出生产选型建议；然后用**官方 Go SDK** 写一个 30 行的 MCP server，让 Python
agent 跨语言调用我们真实的 Go 后台——P3 到 P5 那条"脚手架 → 真实后台"的伏笔，到这里收线。

**等我回复「继续」。**
