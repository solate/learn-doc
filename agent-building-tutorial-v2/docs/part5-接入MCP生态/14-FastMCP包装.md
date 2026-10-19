# 第 14 篇：用 FastMCP（现 MCPServer）把管理助手工具升级为 MCP server

> **知识地图【E2】· Part 5** · 第 13 篇在纸面上看完了协议：JSON-RPC 报文、stdio 传输、握手与发现。本篇把 M2/M3 助手的四个后台工具（login / get_users / get_user_roles / assign_role）真的搬进一个 MCP server 进程，agent 端从 `import admin_tools` 换成 stdio 连接。真正的大戏不在「包装」本身，而在**保护分层**：第 9-10 篇攒下的确认闸门、两段式校验、审计日志搬家后各住哪一层、怎么证明没有一层被静默丢掉。

## 本篇要回答的问题

1. 第 13 篇说的 FastMCP，为什么本篇代码里 `import` 进来的是 `MCPServer`？版本口径怎么定、为什么教程要锁版本？
2. `@mcp.tool()` 的「签名即 schema」和第 7 篇手写的 `TOOL_SCHEMAS` 是什么关系——同一份信息的两种写法，各维护什么、差在哪？
3. 工具搬进独立进程后，agent 端怎么连（stdio 子进程）、怎么发现（tools/list）、怎么调用（tools/call）？直连版与 MCP 版的代码到底差几行？
4. 确认闸门、两段式校验、审计日志这三层保护搬家后各住哪层？「server 没有终端可问 y/n」这道约束会把闸门逼到哪一侧？
5. 怎么用同一句话验证两版等价——不等价的迹象该去哪里找？

---

## 1｜先交代更名：FastMCP → MCPServer

### 1.1 更名始末

第 13 篇 §5.2 已经打过预防针，这里正式落地：官方 Python SDK `mcp` 从 2.x 起，把原先的 `FastMCP` 类**更名为 `MCPServer`**。更名不是换了包——包名还是 `pip install mcp`，只是入口类搬家：

```python
# mcp 1.x 时代（老教程里随处可见的写法）
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("admin-tools")

# mcp 2.x（本篇实测 2.3.0，2026-10-16 PyPI 核验现行版）
from mcp.server.mcpserver import MCPServer
mcp = MCPServer("admin-tools", version="0.1")
```

老名字在新版里会直接 `ModuleNotFoundError`，错误信息指向官方迁移指南（第 13 篇本机实测）。所以当你照着 2025 年的博客抄 `from mcp.server.fastmcp import FastMCP` 而环境装的是 2.x，第一行就报错——这不是你抄错了，是教程过时了。本篇标题保留 FastMCP（它是社区通用叫法），代码一律用 `MCPServer`。

### 1.2 本篇锁定的版本口径

- **SDK 版本**：`mcp==2.3.0`（写进 `requirements.txt`，锁死）。
- **协议语义**：跑**握手期**（handshake era，≤2025-11-25 的语义：`initialize` 握手 → 能力协商 → 会话内 `tools/list` / `tools/call`）。第 13 篇核验过：规范 `2026-07-28` 版已删握手走向无状态（每请求 `_meta` 信封 + `server/discover`），但当下生态部署现实仍以握手期为主——「学协议看最新版，写代码查当下生态」，SDK 2.3.0 两个时代都支持，我们跑握手期。
- **实测证据**：本篇所有 stdio 往返在本机真跑过（Python 3.12.15 + mcp 2.3.0），握手协商出的协议版本是 `2025-11-25`（selfcheck_p14 场景 2 的断言实测值）。

为什么教程要锁版本：MCP 规范与 SDK 都在快速演化（一年内两次大改），锁版本是「跑得起来」的最低保障；版本写进 requirements，读者环境与本篇输出才能对上。

## 2｜@mcp.tool()：签名即 schema

### 2.1 同一份信息的两种写法

第 7 篇手写过工具清单：`TOOL_SCHEMAS` 里每个工具一份 JSON（name / description / parameters 的 JSON Schema），再往 `TOOL_REGISTRY` 注册函数实现。**信息有两份**——描述在 schema 里，实现在函数里，靠人肉保持同步：改了函数签名忘了改 schema，模型就会拿着过时的说明书调用（第 7 篇踩坑清单的变种）。

`@mcp.tool()` 把这两份合成一份：**函数签名生成 schema，docstring 生成描述，函数体就是实现**。

| 维度 | 第 7 篇手写（TOOL_SCHEMAS + TOOL_REGISTRY） | @mcp.tool() 声明式 |
|---|---|---|
| 工具名 | dict 里的 `name` 字段 | 函数名 |
| 描述 | `description` 字符串 | docstring |
| 参数 schema | 手写 JSON Schema | 类型注解（`str` → `string`，`int` → `integer`） |
| 参数说明 | properties 里每个字段手写 `description` | `pydantic.Field(description=...)` |
| 必填/默认 | `required` 数组手写 | 有默认值 = 可选，无默认值 = 必填（由签名直接得出） |
| 漂移风险 | schema 与实现分离，可漂移 | 同居一体，改签名即改 schema |
| 生成物 | 直接就是 OpenAI 信封（第 7 篇直用） | JSON Schema（inputSchema），client 翻译后同构 |

「同一份信息」的证据在 selfcheck_p14 场景 4：两种写法产出的 schema 逐工具对账——必填列表、属性名、属性类型（pydantic `int` → JSON Schema `integer`、`str` → `string`）全部一致，4 工具 × 3 项 = 12 条断言。

### 2.2 把四个工具包装成 server：admin_mcp_server.py

server 端全文如下。四个要点先划出来，读代码时对照：

- **工具实现不重写**：函数体直接委托 `admin_tools` 的原实现（login / get_users / get_user_roles / assign_role 一行调用）——server 只是给同一批实现换了进程和声明方式。
- **token 搬家**：第 8 篇「决定二：token 在进程内存持有」原样成立，只是持有者从 agent 进程换成了 server 进程（`admin_tools._token` 现在活在 server 子进程里）。
- **语义白名单随实现住 server**：`assign_role` 的函数体里复用 `admin_tools.EXTRA_VALIDATORS`（第 10 篇的 user_id 正则 + role 枚举），规则与实现同居一处，拒绝以 error dict 返回（回喂自纠，不抛异常——第 7 篇同一哲学）。
- **描述里删掉了闸门句**：对比直连版，`assign_role` 的 docstring 不再有「系统会在执行时请求用户在终端确认」——server 不替 client 承诺（§4.2 详述），这句话由 client 在拼装时追加。

```python
# admin_mcp_server.py — 第 14 篇：把 M2 四个后台工具包装成 MCP stdio server
# 直连版（admin_tools.py）的工具实现与 agent 同进程；本文件把同一批实现搬进独立进程，
# 用 mcp 2.x 的 MCPServer（原 FastMCP，更名说明见第 13 篇 §5.2 / 本篇 §1.2）声明式包装：
#   @mcp.tool() + 类型注解 + pydantic Field → tools/list 的 inputSchema 自动生成；
#   docstring → description。手写 TOOL_SCHEMAS 的活，从「维护一份 JSON」变成「写一个函数签名」。
# 保护分层（第 14 篇文章 §4 的表在代码里的落点）：
#   server 端留下：两段式校验（第一段 schema 校验由 SDK 按 inputSchema 自动做——第 13 篇
#   补充实验二实测过它走 isError 文本回喂；第二段语义白名单复用 admin_tools.EXTRA_VALIDATORS，
#   规则与实现同居一处）+ 执行审计（本文件落到 audit_mcp_server.jsonl）+ 登录态
#   （token/凭证仍进程内存持有，只是进程从 agent 换成了本 server——第 8 篇决定二原样成立）。
#   server 端没有、也不该有：终端确认闸门——stdio server 没有终端可问 y/n，闸门是 client
#   的策略（agent.py execute_tool 原样生效），本文件对它一无所知（对比：直连版的
#   assign_role 描述里那句「系统会请求终端确认」在本文件里消失了，它由 mcp_client_tools
#   在 client 侧拼回 description）。
# 日志纪律（经典坑）：stdio 传输下 stdout 是 JSON-RPC 专线，一个字节的 print 都会污染
#   协议流、让 client 解析炸掉——调试日志一律走 stderr 或文件。MCPServer 的 stdio 运行
#   默认不往 stdout 写任何非协议内容，本文件也不配置任何 stdout handler。
# 运行（两种方式等价）：由 client 作为子进程拉起（agent.py --mcp，推荐）；或独立手动启动
#   （调试用）：cd agent && .venv/bin/python admin_mcp_server.py   # stdin/stdout 收发 JSON-RPC
# 依赖：pip install mcp（实测 2.3.0）；后台依赖 mock_backend（BASE_URL 同 admin_tools）。
# 诚实声明：本文件已在本机实测（selfcheck_p14.py 场景 1-7 + demo_mcp_equiv.py，
#           Python 3.12.15 + mcp 2.3.0；stdio 往返报文为实测捕获）。
import json
from pathlib import Path

from pydantic import Field

from mcp.server.mcpserver import MCPServer

import admin_tools
import audit

# server 进程的审计写到自己的文件：client 进程也在写自己的审计（audit_log.jsonl），
# 两个进程不共写一个文件；两份记录的差异本身就是保护分层的观察点——server 只见得到
# 「到达它的调用」，client 的拒绝记录无处可去、必须留在本地（文章 §4 对照表的证据）。
audit.AUDIT_FILE = str(Path(__file__).with_name("audit_mcp_server.jsonl"))

mcp = MCPServer("admin-tools", version="0.1")


def _audited(name: str, args: dict, result: dict) -> dict:
    """server 端审计：每个真正执行到工具函数的调用（含被语义校验拒绝的）都留一行。
    直连版里「校验拒绝不写审计」（agent.execute_tool 在 validate_args 处提前返回）；
    MCP 版语义校验住在工具函数体内，拒绝成了执行结果的一部分——server 如实在案。
    敏感字段打码由 audit._sanitize 负责（password → ***），与直连版同一纪律。"""
    audit.log_tool_call(name, args, json.dumps(result, ensure_ascii=False))
    return result


# ---------- 四个工具：签名即 schema，docstring 即描述，函数体 = 校验 + 委托 admin_tools ----------

@mcp.tool()
def login(
    username: str = Field(description="登录用户名"),
    password: str = Field(description="登录密码"),
) -> dict:
    """登录后台管理系统，获取访问令牌。在调用任何其他后台工具之前必须先成功登录一次。
    演示账号：用户名 admin，密码 admin123。登录成功后令牌由本 server 自动保管并在后续
    调用中自动携带，你无需向用户提及令牌，也不要询问密码以外的任何凭证信息。"""
    result = admin_tools.login(username=username, password=password)
    return _audited("login", {"username": username, "password": password}, result)


@mcp.tool()
def get_users(
    page: int = Field(default=1, description="页码，从 1 开始，默认 1"),
    page_size: int = Field(default=20, description="每页条数，默认 20"),
    keyword: str = Field(default="", description="按用户名模糊过滤，可为空串"),
) -> dict:
    """分页查询后台用户列表，返回 users（每人含 user_id、username、status）与 total。
    keyword 按用户名模糊搜索——不知道 user_id 时，先用它把人搜出来。"""
    result = admin_tools.get_users(page=page, page_size=page_size, keyword=keyword)
    return _audited("get_users", {"page": page, "page_size": page_size, "keyword": keyword},
                    result)


@mcp.tool()
def get_user_roles(
    user_id: str = Field(description="用户 ID，来自 get_users"),
) -> dict:
    """查询指定用户当前拥有的角色列表。user_id 从 get_users 的结果里来。
    在给用户改角色之前，先调它看一眼现状。"""
    result = admin_tools.get_user_roles(user_id=user_id)
    return _audited("get_user_roles", {"user_id": user_id}, result)


@mcp.tool()
def assign_role(
    user_id: str = Field(description="用户 ID，来自 get_users"),
    role_id: str = Field(description="角色 ID，来自 login 返回的 assignable_roles"),
) -> dict:
    """把指定用户的角色设置为 role_id 对应的角色。这是写操作，且是覆盖式：该用户原有的
    角色会被整体替换，role_id 从 login 返回的 assignable_roles 里选；user_id 必须是 u-开头
    的用户编码、role_id 必须是已知角色编码，格式不对会被本 server 的校验层拒绝（请求不会
    发到后台）；执行前先调 get_user_roles 确认现状，并向用户复述一遍将发生的变更。"""
    # 第二段校验（语义白名单）随实现住在 server：复用 admin_tools 第 10 篇的校验器，
    # 不复制第二份规则。拒绝以 error dict 返回（回喂自纠，不抛异常——第 7 篇同一哲学）。
    ok, err = admin_tools.EXTRA_VALIDATORS["assign_role"](
        {"user_id": user_id, "role_id": role_id})
    if not ok:
        return _audited("assign_role", {"user_id": user_id, "role_id": role_id},
                        {"error": True, "code": "rejected", "detail": err})
    result = admin_tools.assign_role(user_id=user_id, role_id=role_id)
    return _audited("assign_role", {"user_id": user_id, "role_id": role_id}, result)


if __name__ == "__main__":
    mcp.run()  # stdio 传输：stdin/stdout 上收发 JSON-RPC（第 13 篇 §3.1）
```

`_audited` 包装是 server 端审计的落点：每个真正执行到工具函数的调用（**含被语义校验拒绝的**）都写一行到 `audit_mcp_server.jsonl`。注意与直连版的一处语义差异：直连版里校验拒绝发生在 `execute_tool` 的 `validate_args` 处、根本不写审计；MCP 版语义校验住进了工具函数体内，拒绝成了执行结果的一部分——server 如实在案。敏感字段打码（password → `***`）由 `audit._sanitize` 负责，与直连版同一纪律。

### 2.3 stdio 的 stdout 纪律（经典坑）

stdio 传输下，**stdout 是 JSON-RPC 专线**：client 和 server 靠这根管道收发报文。工具函数或调试代码里任何一个 `print()`，都会把非协议字节混进报文流，client 侧解析直接炸掉——表现为「连接莫名奇妙断开」或「报文解析错误」，而且炸点在 client，根因在 server 的一个 print，非常难查。

纪律一句话：**server 进程里，日志一律走 stderr 或文件，stdout 一个字节都不许碰**。`MCPServer` 的 stdio 运行模式默认不往 stdout 写非协议内容，我们自己的代码也别配 stdout handler——上面全文里没有任何 print，就是这个原因。

## 3｜agent 端改造：从 import 到 stdio 连接

### 3.1 连接层 MCPConnection：把 async SDK 包成同步接口

官方 client API 是 async 的（`stdio_client` 异步上下文管理器 + `ClientSession` 的 async 方法），而本系列的 loop 从第 3 篇起就是同步教学形态。与其把整个 loop 改成 async，不如用一个后台线程跑 asyncio 循环、`run_coroutine_threadsafe` 把会话包成同步调用（业界常见做法）——loop 一行不改。

另一个设计决定是**常驻连接**：为什么不每次工具调用临时起一条连接？因为 server 子进程的进程内存里持有登录态（token / 凭证），连接断开 = 子进程退出 = 登录态丢失，第 9 篇 401 自动重登的原料（`_credential`）就没了。所以「连接与 agent 同寿命」，`close()` 优雅关停（stdin EOF → 子进程自行退出）。

```python
# mcp_client_tools.py — 第 14 篇：agent 端的 MCP toolset（直连版 admin_tools 的进程外替身）
# 与 admin_tools 暴露同一组注册件接口（TOOL_SCHEMAS / TOOL_REGISTRY / SCHEMA_BY_NAME /
# validate_args），agent.py 的 loop 一行核心逻辑不改——第 8 篇 §3.1「换注册表内容、不换
# 循环结构」的红利在这里第二次兑现：上一次换的是工具集（demo → admin），这一次换的是
# 注册件来源（进程内 dict → MCP server 的 tools/list + tools/call）。
# 三处来源差异（与直连版逐一对应）：
#   TOOL_SCHEMAS   不是手写的 dict，是连接期 tools/list 的发现结果换上 OpenAI 信封；
#   TOOL_REGISTRY  函数体不再干活，把参数原样转发成 tools/call，由 server 子进程执行；
#   validate_args  只做第一段 schema 形状校验（tools.validate_args 是通用校验器，不关心
#                  schema 是谁写的）——语义白名单已随实现搬进 server。
# 保护分层（第 14 篇文章 §4 对照表）：确认闸门留在 client（guard.WRITE_TOOLS 仍在本侧，
#   agent.execute_tool 原样生效，server 不知情）；闸门的提示文案也归 client——server 描述
#   不能替客户端承诺「会有人确认」（不同 client 策略不同），所以 GATE_NOTICE 在拼
#   TOOL_SCHEMAS 时由本模块追加（对比：直连版这句话写死在 admin_tools 的描述里）。
# 依赖：pip install mcp（实测 2.3.0）。官方 client API（2026-10-16 签名核验）：
#   mcp.client.stdio.StdioServerParameters(command, args, env, cwd, encoding,
#                                         encoding_error_handler) + stdio_client(params)
#   异步上下文管理器，产出 (read_stream, write_stream) 二元组；
#   mcp.client.session.ClientSession(read, write)：initialize() / list_tools() /
#   call_tool(name, arguments) 三个方法；返回类型字段是 snake_case（is_error /
#   input_schema，线上报文仍是 isError / inputSchema，见第 13 篇报文）。
# 诚实声明：本文件已在本机实测（selfcheck_p14.py 全场景 + demo_mcp_equiv.py）。
import asyncio
import sys
import threading
from pathlib import Path

from mcp.client.session import ClientSession
from mcp.client.stdio import StdioServerParameters, stdio_client

import guard
import tools  # 只借第 7 篇的通用 schema 校验器；不 import admin_tools——工具实现只在 server 进程

STARTUP_TIMEOUT = 30.0  # 起 server 子进程 + 握手的预算（秒）
CALL_TIMEOUT = 60.0     # 单次 tools/call 的预算（与直连版 TIMEOUT 量级一致）

# 默认 server 命令：用当前解释器把 admin_mcp_server.py 作为子进程拉起（stdio 的标准形态，
# 第 13 篇 §3.1：client 启动 server、报文走管道、进程生命周期即会话生命周期）。
DEFAULT_SERVER_COMMAND = [sys.executable, str(Path(__file__).with_name("admin_mcp_server.py"))]

# client 的闸门提示文案：原直连版 assign_role 描述的最后一句。MCP 版里它由 client 追加——
# 「执行前会有人确认」是 client 侧策略的承诺，server 只描述工具本身（文章 §4）。
GATE_NOTICE = ("系统会在执行时请求用户在终端确认：用户拒绝就是最终决定，"
               "不要重试、也不要换参数绕过。")


class MCPConnection:
    """一条常驻 stdio 连接：后台线程跑 asyncio 循环，把 async 的 client 会话变成同步调用。
    为什么不每次工具调用临时起一条连接：server 子进程的进程内存里持有登录态（token/凭证，
    第 8 篇决定二原样成立——只是搬进了 server 进程），连接断开 = 子进程退出 = 登录态丢失，
    第 9 篇 401 自动重登的原料（_credential）就没了。所以「连接与 agent 同寿命」。
    为什么 agent 主循环还是同步代码：第 3 篇起 loop 就是同步形态，教学不引入 async——
    用一个后台线程 + run_coroutine_threadsafe 把 async SDK 包成同步接口（业界常见做法）。"""

    def __init__(self, command: list[str]):
        self._params = StdioServerParameters(command=command[0], args=command[1:])
        self._loop = None
        self._thread = None
        self._session = None
        self._started = threading.Event()
        self._closed = None  # asyncio.Event，在 _serve 里创建（归属后台循环）
        self._start_error = None

    def start(self) -> None:
        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(target=self._run, daemon=True, name="mcp-client")
        self._thread.start()
        if not self._started.wait(STARTUP_TIMEOUT):
            raise RuntimeError(
                f"MCP server 连接超时（{STARTUP_TIMEOUT}s）——确认命令可执行：{self._params.command}")
        if self._start_error is not None:
            raise RuntimeError(f"MCP server 连接失败：{self._start_error}")

    def _run(self) -> None:
        asyncio.set_event_loop(self._loop)
        try:
            self._loop.run_until_complete(self._serve())
        except Exception as e:  # 连接期失败（子进程起不来/握手失败）：唤醒主线程如实报错
            if not self._started.is_set():
                self._start_error = e
                self._started.set()

    async def _serve(self) -> None:
        async with stdio_client(self._params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()   # 握手：版本协商 + 能力交换（第 13 篇 §2.2 ①②）
                self._session = session
                self._closed = asyncio.Event()
                self._started.set()
                await self._closed.wait()    # 挂起在这里——连接存活到 close() 被调

    def list_tools(self):
        """tools/list 的同步封装。返回 ListToolsResult（.tools 是 Tool 列表）。"""
        return asyncio.run_coroutine_threadsafe(self._session.list_tools(),
                                                self._loop).result(CALL_TIMEOUT)

    def call_tool(self, name: str, arguments: dict):
        """tools/call 的同步封装。返回 CallToolResult（.content / .is_error）。"""
        return asyncio.run_coroutine_threadsafe(
            self._session.call_tool(name, arguments), self._loop).result(CALL_TIMEOUT)

    def protocol_version(self) -> str:
        """握手协商出的协议版本（第 13 篇 §2.3 的还价结果，实测 2025-11-25）。"""
        return self._session.protocol_version

    def server_info(self) -> dict:
        """server 自报家门（serverInfo：name/version）。"""
        info = self._session.server_info
        return {"name": info.name, "version": info.version}

    def close(self) -> None:
        """优雅关停：唤醒 _serve → async with 退出（stdio_client 关 stdin、回收子进程）→
        线程结束 → 关循环。若主进程直接退出也没关系：daemon 线程随进程消亡，子进程读
        stdin 得到 EOF 会自行退出（第 13 篇 §3.1 的生命周期规则）。"""
        if self._closed is not None and self._loop is not None and self._loop.is_running():
            self._loop.call_soon_threadsafe(self._closed.set)
        if self._thread is not None:
            self._thread.join(timeout=10)
        if self._loop is not None and not self._loop.is_closed():
            try:
                self._loop.close()
            except Exception:
                pass


class MCPToolset:
    """直连版 toolset 的 MCP 替身。用法：ts = MCPToolset(); ts.connect(); 之后
    ts.TOOL_SCHEMAS / ts.TOOL_REGISTRY / ts.SCHEMA_BY_NAME / ts.validate_args 与
    admin_tools 完全同形，agent.py 的 call_llm / execute_tool / planner 全部无感。"""

    def __init__(self, command: list[str] | None = None):
        self._conn = MCPConnection(list(command or DEFAULT_SERVER_COMMAND))
        self.TOOL_SCHEMAS: list[dict] = []
        self.TOOL_REGISTRY: dict[str, object] = {}
        self.SCHEMA_BY_NAME: dict[str, dict] = {}

    def connect(self) -> None:
        """启动连接 + tools/list 发现工具 + 拼注册件。发现是动态的：server 加了新工具，
        这里重连一次就能看到——直连版要改代码注册，MCP 版只是重新问一遍（§1.3 表）。"""
        self._conn.start()
        result = self._conn.list_tools()
        for t in result.tools:
            function = self._to_openai_function(t)
            self.TOOL_SCHEMAS.append({"type": "function", "function": function})
            self.SCHEMA_BY_NAME[t.name] = function
            self.TOOL_REGISTRY[t.name] = self._make_invoker(t.name)

    @staticmethod
    def _to_openai_function(t) -> dict:
        """MCP Tool（name/description/input_schema）→ OpenAI 函数格式。翻译做三件事：
        ① input_schema 本来就是 JSON Schema（第 13 篇 §2.2 的同构结论），装进 parameters；
        ② 剥掉顶层 title（pydantic 自动生成的 "xxxArguments" 对模型无信息量；属性里的
           title/description/default 保留——那是 server 作者写的参数说明，原样转发）；
        ③ 写操作追加闸门提示（client 的承诺，client 来说，§头部注释）。"""
        params = dict(t.input_schema or {"type": "object", "properties": {}})
        params.pop("title", None)
        description = t.description or ""
        if t.name in guard.WRITE_TOOLS:
            description = (description + GATE_NOTICE) if description else GATE_NOTICE
        return {"name": t.name, "description": description, "parameters": params}

    def _make_invoker(self, name: str):
        """给一个工具名造同步函数：(**arguments) → tools/call → 提取文本。
        返回值形状与直连版对齐：直连版工具返回 dict、execute_tool 统一 json.dumps；
        MCP 版 server 已经把 dict 序列化成文本（JSON，带缩进），client 原样取出——
        最终回喂模型的都是这份 JSON 的语义（缩进不同，demo_mcp_equiv 里可对照）。"""
        def invoke(**arguments) -> str:
            r = self._conn.call_tool(name, arguments)
            texts = [c.text for c in r.content if getattr(c, "type", "") == "text"]
            text = texts[0] if texts else ""
            if r.is_error and not text:
                return "错误：MCP server 报告工具执行失败且未返回内容"
            # is_error=True 且有文本（server 端 pydantic 校验拒绝，第 13 篇补充实验二）：
            # 文本本身就是给模型读的错误说明，原样回喂——与直连版「错误不抛出、回喂」同哲学
            return text
        return invoke

    def validate_args(self, function_schema: dict, args: dict):
        """与 admin_tools.validate_args 同形，但只有第一段：schema 形状校验（缺必填 /
        类型不对 / 枚举外）。语义白名单（user_id 正则、role_id 枚举）已随实现搬进 server
        ——保护分层表里「语义校验」一行在 server 侧的原因：规则与实现同居一处，任何
        client 接入都自动获得同一套校验，不用每个 client 重写一遍（文章 §4）。"""
        return tools.validate_args(function_schema, args)

    @property
    def protocol_version(self) -> str:
        return self._conn.protocol_version()

    @property
    def server_info(self) -> dict:
        return self._conn.server_info()

    def close(self) -> None:
        self._conn.close()


def build(command: list[str] | None = None) -> MCPToolset:
    """连接 + 发现，一步到位（agent.py --mcp 的入口）。"""
    toolset = MCPToolset(command)
    toolset.connect()
    return toolset
```

### 3.2 注册件同形：MCPToolset

`MCPToolset` 与 `admin_tools` 暴露**同一组注册件接口**（`TOOL_SCHEMAS` / `TOOL_REGISTRY` / `SCHEMA_BY_NAME` / `validate_args`），agent.py 的 `call_llm` / `execute_tool` / planner 全部无感。这是第 8 篇 §3.1「换注册表内容、不换循环结构」红利的第二次兑现：上一次换的是工具集（demo → admin），这一次换的是注册件来源（进程内 dict → MCP server 的 tools/list + tools/call）。

三个来源差异（`mcp_client_tools.py` 头部注释逐条对应）：

- **TOOL_SCHEMAS** 不是手写的 dict，是连接期 `tools/list` 的发现结果换上 OpenAI 信封。翻译做三件事：① `input_schema` 本来就是 JSON Schema（第 13 篇 §2.2 的同构结论），装进 `parameters`；② 剥掉顶层 `title`（pydantic 自动生成的 `XxxArguments` 对模型无信息量；属性里的 title/description/default 保留——那是 server 作者写的参数说明，原样转发）；③ 写操作追加闸门提示（client 的承诺，client 来说，§4.2）。
- **TOOL_REGISTRY** 函数体不再干活，把参数原样转发成 `tools/call`，由 server 子进程执行。`is_error=True` 且有文本时（server 端校验拒绝），文本本身就是给模型读的错误说明，原样回喂——与直连版「错误不抛出、回喂」同哲学。
- **validate_args** 只做第一段 schema 形状校验（`tools.validate_args` 是通用校验器，不关心 schema 是谁写的）——语义白名单已随实现搬进 server（§4.3）。

### 3.3 agent.py 的 --mcp 开关与惰性加载

agent.py 的改动集中在两处：`toolset` 从模块级常量改为**惰性加载**（`_default_toolset()`：第一次用时才 `import admin_tools`），以及 `--mcp` 开关（连接 server → 动态发现 → 注册件同形装载）。

为什么必须惰性加载：如果 agent.py 一 import 就加载 `admin_tools`，那 `--mcp` 模式下工具实现还是进了 agent 进程，「工具只活在 server 子进程」就成了空话。惰性加载后，`--mcp` 模式的 agent 进程从头到尾不 import `admin_tools`——这是保护分层的**物理前提**（selfcheck_p14 场景 1 的两条断言验证的就是它）。

```python
# agent.py — 手写最小 agent（第 7 篇 M1 建立；第 8 篇 M2 主线闭环；第 9 篇工程护栏；
#            第 11 篇接入记忆：交互模式默认带记忆；第 12 篇加 --plan 规划模式；
#            第 14 篇加 --mcp：同一 loop，工具从进程内 import 换成 MCP server）
# 依赖：pip install -r requirements.txt（openai 3.31.0；Python 3.11+，第 5 篇；
#       --mcp 需 mcp 2.3.0，第 14 篇加入 requirements）
# 运行：
#   终端 1：python -m uvicorn mock_backend.main:app --port 8000   # 先起后台脚手架（第 8 篇）
#   终端 2：
#   export DEEPSEEK_API_KEY=sk-xxx && python agent.py "把张三提升为管理员"   # M2 主线路径
#   python agent.py --mcp "把张三提升为管理员"   # MCP 版（第 14 篇）：工具走 stdio 子进程
#   python agent.py                            # 跑后台预置问题集
#   python agent.py --plan "把赵六提升为管理员"  # 规划模式：先列计划再执行（第 12 篇）
#   python agent.py --chat --plan              # 交互模式也支持规划（M3 联合验收）
#   python agent.py --demo                     # 切回第 7 篇演示工具集（M1 复验）
#   python agent.py --chat                     # 交互模式（第 11 篇起默认带记忆，jsonl 后端）
#   python agent.py --chat --user u-alice      # 指定记忆归属用户（隔离键，第 11 篇）
#   python agent.py --chat --mem-backend redis_pg   # Redis+Postgres/pgvector 后端（实践②）
#   python agent.py --chat --no-mem            # 关闭记忆（第 10 篇前的行为）
#   python agent.py --think                    # 思考档（第 4 篇 §1.5）
#   python selfcheck_p9.py                     # 第 9 篇桩测试（零 token，见实践节）
#   python selfcheck_p11.py                    # 第 11 篇桩测试（零 token 零网络，见实践节）
#   python selfcheck_p12.py                    # 第 12 篇桩测试（零 token 零网络，见实践节）
#   python selfcheck_p14.py                    # 第 14 篇 MCP 桩测试（零 LLM token，见实践节）
#   python demo_mcp_equiv.py                   # 第 14 篇两版等价对照（零 LLM token，自管后台）
# 诚实声明：loop 逻辑（含四条错误兜底路径与熔断）已用桩模型本地实测；
#           M2 闭环已对本地 mock_backend 实测（桩 LLM + 真实 HTTP，见第 8 篇实践节）；
#           第 9 篇新增的确认闸门/审计落盘/自动重登已桩测 + 真实 HTTP 实测（selfcheck_p9.py
#           与第 9 篇实践节）；第 11 篇记忆链路已桩测（selfcheck_p11.py 59 断言）+
#           零依赖 demo 实测（demo_memory.py）；第 12 篇规划链路已桩测（selfcheck_p12.py）+
#           桩 LLM × 真实 HTTP demo 实测（demo_planning.py，见第 12 篇实践节）；
#           第 14 篇 MCP 链路已桩测（selfcheck_p14.py）+ 两版等价实测（demo_mcp_equiv.py）；
#           对真实 DeepSeek 的调用与 redis_pg 后端 <未实测，以官方文档为准>
import argparse
import json
import os

from openai import OpenAI

import audit
import config
import guard
import memory
import memory_redis_pg
import planner
import tools as demo_tools   # 演示工具集：--demo 复验 M1 时使用

# 第 8 篇主线切换：工具集从演示工具（tools.py）换成后台管理工具（admin_tools.py）。
# 两个模块暴露同一组注册件接口（TOOL_SCHEMAS / TOOL_REGISTRY / SCHEMA_BY_NAME /
# validate_args），所以下面的 loop 代码一行核心逻辑都不用改——这就是第 3 篇 §2.3
# 说"工具注册件是形状"的红利：换的是注册表内容，不是循环结构。
# 第 14 篇把这条红利推到进程边界：--mcp 时 toolset 是 mcp_client_tools.MCPToolset，
# 注册件来自 MCP server 的 tools/list / tools/call，接口同形，loop 依旧零改动。
# 直连注册件改为「用时才加载」（_default_toolset）：--mcp 模式下 agent 进程从头到尾
# 不 import admin_tools——工具实现只活在 server 子进程里（保护分层的物理前提，
# selfcheck_p14 场景 1 有断言）。
toolset = None


def _default_toolset():
    """惰性加载直连注册件：demo/selfcheck 直接调本模块 execute_tool 的既有路径
    （第 9-12 篇）在此拿到 admin_tools，行为与之前逐字节一致。"""
    global toolset
    if toolset is None:
        import admin_tools
        toolset = admin_tools
    return toolset

# 系统提示词随工具集绑定切换（main 里 --demo 时三者一起换）——它们与工具描述同属
# "给模型看的说明书"（第 7 篇 §1.4），改工具不改说明，模型立刻按旧世界观干活。
# 第 9 篇升级两条：规则 3 = 先只读后写（信息没查清楚之前不动手写，文章 §3.3）；
# 规则 4 = 把确认闸门的存在与"拒绝即最终决定"讲给模型（闸门本身是代码级强制，文章 §3.2）。
SYSTEM_PROMPT = (
    "你是一个后台管理助手，通过工具操作用户管理系统。规则：\n"
    "1. 涉及用户、角色的事实必须调用工具获取，禁止编造 user_id / role_id；\n"
    "2. 调用其他后台工具前必须先 login（演示账号写在工具说明里）；令牌由系统自动保管，"
    "不必向用户提及，也不要在回复里复述任何令牌；\n"
    "3. 先只读后写：执行写操作前，先用查询工具把现状查清楚；查不到目标（如用户不存在）"
    "就向用户如实说明，不要凭猜测强行写；\n"
    "4. assign_role 是写操作，系统会在终端请求用户确认：用户拒绝就是最终决定，"
    "不要重试、不要换参数绕过；\n"
    "5. 回答里引用的 ID、角色名必须原样来自工具结果；工具返回错误时读懂错误、"
    "调整后重试，同样的错误不要原样重发。"
)

PRESET_QUESTIONS = [
    "把张三提升为管理员",            # M2 验收主路径：登录→搜人→查角色→分配（写操作）
    "李四现在是什么角色？",           # 只读路径：登录→搜人→查角色
    "系统里现在有哪些用户？",         # 只读路径：登录→分页列表
]

# 第 7 篇演示工具集的原提示词与原预置问题：--demo 复验 M1 时整组换回，内容原样保留
DEMO_SYSTEM_PROMPT = (
    "你是一个跑在终端里的最小 agent。规则：\n"
    "1. 事实性问题（当前时间、计算结果）必须调用工具获取，禁止凭记忆编造；\n"
    "2. 回答中引用的数值必须原样来自工具结果，不要自己改写或补全；\n"
    "3. 工具返回错误信息时，读懂错误、调整调用方式重试，不要原样重复同样的调用。"
)

DEMO_PRESET_QUESTIONS = [
    "现在几点了？",
    "37 乘以 45 等于多少？",
    "先看看现在几点，再把当前的小时数乘以 7。",
    "用 calculate 算一下 1 除以 0，把工具返回的内容告诉我。",
]


def create_client() -> OpenAI:
    """姿势二（第 6 篇 §2）：官方 SDK。三个字符串全部来自 config 唯一配置点。"""
    return OpenAI(api_key=config.api_key(), base_url=config.BASE_URL,
                  timeout=config.TIMEOUT, max_retries=config.MAX_RETRIES)


def call_llm(client: OpenAI, messages: list, think: bool = False):
    """阶段② 调 LLM：全部 messages + 工具清单一起发（第 3 篇：每轮全量重发）。"""
    ts = toolset if toolset is not None else _default_toolset()
    kwargs = dict(model=config.MODEL, messages=messages, tools=ts.TOOL_SCHEMAS)
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


def execute_tool(call) -> str:
    """阶段④ 执行单个工具调用。总原则：任何失败都不抛出——把错误文本当作工具结果
    回填给模型（它下一轮读到后自行纠正），循环永远不因工具问题崩掉（第 7 篇 §2.4）。
    第 9 篇在 validate_args 之后挂了两道关卡：
      确认闸门：写操作先过 guard.confirm，用户终端 y/N，拒绝文本回喂模型（文章 §3.2）；
      审计落盘：每次真实执行（成功/失败/被拒）都写一行 audit_log.jsonl（文章 §3.4）。
    幻觉调用与坏参数不写审计：它们没到工具层，属于模型的调用失误——那是 P7·18
    trace 的管辖范围，本篇审计只记"真正发生了什么"。"""
    name = call.function.name
    ts = toolset if toolset is not None else _default_toolset()
    if name not in ts.TOOL_REGISTRY:             # 坑 1：幻觉调用——调了不存在的工具
        return (f"错误：不存在名为 {name!r} 的工具。"
                f"可用工具：{sorted(ts.TOOL_REGISTRY)}")
    try:                                         # 坑 2：坏 JSON / 空参——arguments 是 JSON 字符串
        args = json.loads(call.function.arguments) if call.function.arguments else {}
    except json.JSONDecodeError as e:
        return f"错误：arguments 不是合法 JSON（{e}）。请重新发起调用。"
    ok, err = ts.validate_args(ts.SCHEMA_BY_NAME[name], args)             # 坑 3：schema 校验
    if not ok:
        want = json.dumps(ts.SCHEMA_BY_NAME[name]["parameters"], ensure_ascii=False)
        return f"错误：参数校验失败（{err}）。参数要求：{want}"
    if guard.is_write_tool(name):                # 代码级闸门：提示词可以不听，闸门绕不过
        allowed, feedback = guard.confirm(name, args)
        if not allowed:
            audit.log_tool_call(name, args, '{"error": true, "denied": true}')
            return feedback
    try:                                         # 坑 4：运行时异常 + HTTP 层漏网的未知异常
        result = ts.TOOL_REGISTRY[name](**args)
    except Exception as e:   # 兜底链最后一环（第 9 篇 §2.4）：未知异常也变成可回喂文本
        result = f"错误：工具 {name} 执行异常：{type(e).__name__}: {e}"
    result_str = result if isinstance(result, str) else json.dumps(result, ensure_ascii=False)
    audit.log_tool_call(name, args, result_str)  # 审计：每次真实执行都留痕（含被拒调用）
    return result_str


def run_agent(client, user_input: str, think: bool = False,
              messages: list | None = None) -> str:
    """一轮完整的 agent loop：第 3 篇五阶段的实体化。返回最终答复文本。
    第 11 篇：messages 可由记忆层（MemoryManager.build_messages）预装——传入时阶段①
    的"拼初始上下文"换成读记忆的结果（system+记忆区+摘要+近 N 轮+本轮输入）；
    不传时行为与第 8 篇逐字节一致（M2 回归保真，selfcheck_p9/p10 验证）。"""
    # 阶段① 获取输入：system 纪律 + user 问题——agent 初始的内部状态
    if messages is None:
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
        # 工具已是远程 HTTP（第 8 篇），教学版仍顺序执行——同消息多工具并行发送属于
        # 时延优化项，本系列不做（M2 流程每轮通常只来一个调用）。
        for call in msg.tool_calls:
            print(f"[工具] {call.function.name}({call.function.arguments})")
            result = execute_tool(call)
            print(f"[工具→] {result}")
            messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
    # 终止条件二：最大迭代数熔断——熔断后必须有明确行为，而不是静默返回空（第 3 篇 §4.2）
    raise RuntimeError(f"[熔断] 达到最大迭代数 {config.MAX_ITERATIONS}，强制终止")


def run_safely(client, question: str, think: bool = False,
               messages: list | None = None) -> str:
    """熔断等 RuntimeError 转成可见的答复文本——明确报错，不静默、不崩终端。"""
    try:
        return run_agent(client, question, think=think, messages=messages)
    except RuntimeError as e:
        return str(e)


def build_memory_manager(args) -> memory.MemoryManager:
    """第 11 篇：按 --mem-backend 组装记忆门面（会话 + 长期两后端，接口同形可互换）。
    jsonl：零依赖教学后端（本机实测，selfcheck_p11 场景 1-5 全覆盖）；
    redis_pg：Redis 会话 + Postgres/pgvector 长期（实践②，docker compose up -d 后可用，
    <未实测，以官方文档为准>；接口契约已由 selfcheck_p11 场景 6 桩验证）。"""
    if args.mem_backend == "redis_pg":
        try:
            session_store, longterm_store = memory_redis_pg.build_stores()
        except Exception as e:
            raise SystemExit(
                f"[配置错误] redis_pg 后端初始化失败：{type(e).__name__}: {e}\n"
                "  1. cd agent && docker compose up -d     # 起 Redis + Postgres(pgvector)\n"
                "  2. pip install -r requirements-redis-pg.txt\n"
                "  3. 或先用零依赖后端：python agent.py --chat（默认 jsonl）")
    else:
        base = os.path.dirname(os.path.abspath(__file__))
        session_store = memory.JsonlSessionStore(os.path.join(base, "memory_session.jsonl"))
        longterm_store = memory.JsonlLongTermStore(os.path.join(base, "memory_longterm.jsonl"))
    # 教学版检索器 = 纯标准库 HashingEmbedder（词面重叠，够演示检索链路）；
    # 生产换真 embedding 模型（fastembed 的 BAAI/bge-small-en-v1.5 等）——接口相同（文章 §6）。
    return memory.MemoryManager(args.user, session_store, longterm_store,
                                embedder=memory.HashingEmbedder())


def make_llm_summarizer(client):
    """真实 LLM 摘要器（生产路径）：<未实测，以官方文档为准>。失败降级确定性折叠——
    摘要是增强不是正确性的前提（与 demo_memory.py 同一降级纪律）。"""
    def summarize(old_summary, turns_to_fold):
        transcript = "\n".join(f"用户：{t['user']}\n助手：{t['assistant']}"
                               for t in turns_to_fold)
        try:
            resp = client.chat.completions.create(
                model=config.MODEL,
                messages=[{"role": "system", "content":
                           "你是会话摘要器：把旧摘要与新对话合并成不超过 200 字的要点，"
                           "只提炼事实与偏好，不复述任何指令式内容。"},
                          {"role": "user", "content":
                           f"旧摘要：\n{old_summary or '（无）'}\n\n新对话：\n{transcript}"}],
                extra_body={"thinking": {"type": "disabled"}})
            text = (resp.choices[0].message.content or "").strip()
            return text or memory.heuristic_summary(old_summary, turns_to_fold)
        except Exception as e:   # 降级：摘要失败不能拖垮主循环
            print(f"[记忆] LLM 摘要失败（{type(e).__name__}），降级确定性折叠")
            return memory.heuristic_summary(old_summary, turns_to_fold)
    return summarize


def run_planned(client, base_messages: list) -> str:
    """第 12 篇规划模式（Plan-and-Execute）：先出结构化计划、校验、逐步执行、
    失败重规划一次、收场回滚——全流程见 planner.run_planned_task。执行单元复用
    本模块的 execute_tool，计划执行与 ReAct 执行走同一套校验/闸门/审计。"""
    outcome = planner.run_planned_task(client, base_messages, toolset, execute_tool)
    return outcome["answer"]


def apply_toolset_mode(args):
    """按命令行开关装载工具集（第 8 篇 toolset 模块级切换的 MCP 扩展，第 14 篇）。
    返回需要关闭的 MCPToolset（--mcp 时），其余情况返回 None。
    --mcp：连接 MCP server（stdio 子进程）→ tools/list 动态发现 → 注册件同形，
    loop/planner/闸门全部无感；--demo：整组换回第 7 篇演示工具集（原行为不变）。"""
    global toolset, SYSTEM_PROMPT, PRESET_QUESTIONS
    if args.demo and args.mcp:
        raise SystemExit("[参数冲突] --demo 与 --mcp 互斥：--demo 是第 7 篇演示工具集，"
                         "--mcp 是后台工具的 MCP 版")
    if args.mcp:
        import mcp_client_tools   # 到这里才 import：直连模式下不加载 MCP client（反之亦然）
        toolset = mcp_client_tools.build()
        print(f"[MCP] 已连接 server {toolset.server_info}（协议 {toolset.protocol_version}），"
              f"发现 {len(toolset.TOOL_SCHEMAS)} 个工具：{sorted(toolset.TOOL_REGISTRY)}")
        return toolset
    if args.demo:
        toolset, SYSTEM_PROMPT, PRESET_QUESTIONS = demo_tools, DEMO_SYSTEM_PROMPT, DEMO_PRESET_QUESTIONS
    return None


def main():
    parser = argparse.ArgumentParser(description="手写最小 agent（第 7 篇 M1 → 第 8 篇 M2 → 第 11 篇带记忆 → 第 12 篇带规划 → 第 14 篇 --mcp）")
    parser.add_argument("question", nargs="?", help="单次提问；不给则跑预置问题集")
    parser.add_argument("--plan", action="store_true",
                        help="规划模式：先出结构化计划再执行（第 12 篇 --plan 对照开关）")
    parser.add_argument("--chat", action="store_true",
                        help="交互模式（第 11 篇起默认带记忆；--no-mem 关闭）")
    parser.add_argument("--think", action="store_true", help="开思考档（第 4 篇 §1.5）")
    parser.add_argument("--mcp", action="store_true",
                        help="工具走 MCP server（第 14 篇）：stdio 子进程连接 admin_mcp_server，"
                             "工具发现用 tools/list；确认闸门仍在 client（本进程）生效")
    parser.add_argument("--demo", action="store_true",
                        help="切回第 7 篇演示工具集 get_time/calculate（M1 复验用）")
    parser.add_argument("--user", default="u-demo",
                        help="记忆归属用户（第 11 篇隔离键；真实租户隔离在 P7·20）")
    parser.add_argument("--mem-backend", choices=("jsonl", "redis_pg"), default="jsonl",
                        help="记忆后端：jsonl=零依赖教学版（默认）；redis_pg=Redis+Postgres/pgvector")
    parser.add_argument("--no-mem", action="store_true",
                        help="关闭记忆（第 10 篇前的行为，每次输入都是全新会话）")
    args = parser.parse_args()
    mcp_toolset = apply_toolset_mode(args)   # 工具集装载：--mcp / --demo / 默认直连（第 14 篇）
    client = create_client()
    try:
        if args.chat:
            # 第 11 篇主线：交互模式接入记忆。写策略先跑（显式「记住：」入长期记忆 +
            # 偏好槽），读策略再拼上下文（记忆区 + 摘要 + 近 N 轮），答复后落会话轮。
            # --demo 与 --no-mem 保持旧行为：M1/M2 回归路径零改动。
            mgr = None if (args.no_mem or args.demo) else build_memory_manager(args)
            if mgr is not None:
                print(f"交互模式（输入 quit 退出）。记忆已开启：用户 {args.user}，"
                      f"后端 {args.mem_backend}。说「记住：……」可写入长期记忆；"
                      "--no-mem 关闭记忆。")
            else:
                print("交互模式（输入 quit 退出）。本次运行不使用记忆——"
                      "每次输入都是一次全新会话（第 10 篇前的行为）。")
            while True:
                user_input = input("你> ").strip()
                if user_input.lower() in ("quit", "q", "exit"):
                    break
                if not user_input:
                    continue
                if mgr is None:
                    print(f"[助手] {run_safely(client, user_input, think=args.think)}")
                    continue
                for w in mgr.remember_from_user_text(user_input):
                    if w["written"]:
                        print(f"[记忆] 已写入（{w['kind']}）：{w['reason']}")
                    else:
                        print(f"[记忆] 未写入：{w['reason']}")
                # 规划模式的 system 换成规划器提示词（业务纪律 + 工具目录 + 计划契约），
                # 记忆区仍由 build_messages 拼接——计划因此能直接引用记忆里的事实（M3 协同）
                base_system = (planner.build_planner_system(toolset, SYSTEM_PROMPT)
                               if args.plan else SYSTEM_PROMPT)
                messages, stats = mgr.build_messages(base_system, user_input,
                                                     summarizer=make_llm_summarizer(client))
                print(f"[记忆] 保留近 {stats['kept_turns']} 轮 | 已折叠 {stats['summarized_turns']} 轮"
                      f" | 召回 {stats['memory_hits']} 条 | 估算 {stats['estimated_tokens']} tokens")
                if args.plan:
                    answer = run_planned(client, messages)
                else:
                    answer = run_safely(client, user_input, think=args.think, messages=messages)
                print(f"[助手] {answer}")
                mgr.record_turn(user_input, answer)
        elif args.question:
            if args.plan:
                messages = [{"role": "system",
                             "content": planner.build_planner_system(toolset, SYSTEM_PROMPT)},
                            {"role": "user", "content": args.question}]
                print(f"[用户] {args.question}")
                print(f"[助手] {run_planned(client, messages)}")
            else:
                print(f"[助手] {run_safely(client, args.question, think=args.think)}")
        else:                                # 预置问题集：M2 验收路径（--demo 时为 M1 复验）
            if args.plan:
                print("[提示] --plan 只作用于单次提问与 --chat；预置问题集保持 ReAct "
                      "原路径（M2 回归锚点，第 12 篇 §实践）")
            for q in PRESET_QUESTIONS:
                print(f"[助手] {run_safely(client, q, think=args.think)}")
    except KeyboardInterrupt:
        print("\n[退出] 用户中断（第 3 篇终止条件之四：用户确认/打断）")
    finally:
        if mcp_toolset is not None:
            mcp_toolset.close()   # 优雅关停 MCP server 子进程（stdin EOF → 进程退出）


if __name__ == "__main__":
    main()
```

### 3.4 直连版 vs MCP 版：代码差异对照

| 维度 | 直连版（第 8-10 篇） | MCP 版（本篇） | 差几行 |
|---|---|---|---|
| 工具发现 | `import admin_tools`（静态，启动即定） | 连接期 `tools/list`（动态，server 加工具后重连即见） | +1 个连接建立调用 |
| 工具调用 | 进程内函数调用 | `tools/call` JSON-RPC 报文 → stdio 管道 → server 进程执行 → 结果文本回来 | 调用方零改动（注册件同形） |
| schema 来源 | 手写 TOOL_SCHEMAS | server 端签名生成，client 翻译 | 维护面从「两份」变「一份」 |
| 错误路径 | 异常 → 兜底链 → 回喂（第 9 篇） | isError + 错误文本 → 回喂（同一终点） | 语义等价 |
| 部署形态 | 工具与 agent 同进程、同语言 | server 独立进程，可换语言、换机器、多 client 共用 | 本篇 stdio 同机；远程形态第 15 篇 |
| loop 改动 | — | **零** | 0 行 |

最后一行是本篇最想让你看见的事实：把工具从进程内搬进独立进程、从函数调用换成协议往返，**驱动循环没有改一行核心逻辑**——因为第 3 篇起 loop 依赖的就是「注册件接口」这个形状，而不是任何具体工具。

## 4｜保护分层：三层保护搬家后的住址

工具搬家，第 9-10 篇攒下的保护怎么办？先给总表，再逐条讲为什么。

### 4.1 分层总表

| 保护 | 直连版住址 | MCP 版住址 | 搬家理由 |
|---|---|---|---|
| schema 形状校验 | `tools.validate_args`（client 进程） | **两侧都有**：server 端 SDK 按 inputSchema 自动做（拒绝走 isError 文本回喂，第 13 篇补充实验二实测）；client 端保留第一段形状校验 | server 端是权威（任何 client 都拦不住绕过自己），client 端让坏参数死在发出之前、错误提示更早 |
| 语义白名单（user_id 正则 + role 枚举） | `admin_tools.EXTRA_VALIDATORS`（client 进程） | **server 端**，工具函数体内（复用同一份 EXTRA_VALIDATORS） | 规则与实现同居一处；任何 client 接入自动获得同一套校验，不用每个 client 重写一遍 |
| 确认闸门（写操作 y/N） | `guard.confirm` 挂 `execute_tool`（client） | **仍在 client**，`execute_tool` 原样生效，server 不知情 | 见 §4.2 |
| 审计日志 | `audit_log.jsonl`（单文件） | **双文件**：server 记到达它的调用（`audit_mcp_server.jsonl`）；client 记本侧决定（含 denied） | 各自如实，见 §4.4 |
| 登录态（token/凭证） | `admin_tools._token`（agent 进程内存） | **server 进程**内存（同一变量，换了进程） | 第 8 篇决定二原样成立：不进 prompt、不进日志、进程内存持有 |

### 4.2 为什么闸门必须在 client

三个理由，一个比一个硬：

1. **stdio server 没有终端**。确认闸门要问人「允许吗 (y/N)」，问话需要一个能跟用户交互的通道。stdio server 的 stdin/stdout 都是 JSON-RPC 专线（§2.3），它根本摸不到用户的键盘。
2. **「会有人确认」是 client 的承诺，不是工具的属性**。同一个 server 可能被 CLI client（终端问 y/N）、Web client（弹窗确认）、无人值守的自动化 client（全部拒绝或走审批流）连接——server 如果在描述里写死「会有人确认」，就是在替素未谋面的 client 做承诺。所以直连版描述里那句「系统会在执行时请求用户在终端确认」从 server 的 docstring 里删掉，由 `mcp_client_tools` 在 client 侧拼装时追加（`GATE_NOTICE`）——文案跟着策略走，策略住在谁家，文案就由谁写。
3. **闸门的位置由挂点决定，不由工具决定**。闸门挂在 `execute_tool` 的必经之路上（第 9 篇），`execute_tool` 在 client 进程里——工具搬去 server，闸门的挂点没搬，保护就不可能丢：模型要触发写操作，必须先经过 client 的 `execute_tool`，那里有 `guard.confirm` 等着。

### 4.3 为什么语义校验与审计在 server

语义白名单（第 10 篇）的核心论点是「规则与实现同居一处」——user_id 格式、role 枚举这些业务语义，跟着工具实现走。工具搬进 server，规则就该一起搬：这样**任何** MCP client 接入这个 server，都自动获得同一套校验，而不是指望每个 client 作者都记得重写一遍。selfcheck_p14 场景 5 验证的就是这条：`user_id="../../x"` 的路径遍历载荷死在 server 的校验层，**请求不会发到后台**，server 审计如实记下 `error:rejected`。

审计在 server 的理由相同：server 是工具的唯一入口，「到达 server 的每一次调用」由 server 记账最完整——不管调用来自哪个 client。

### 4.4 双审计文件：两本账各记各的

保护分层最直观的观察点是两个审计文件。跑一次「用户拒绝写操作」（selfcheck_p14 场景 7）：

- **client 侧 `audit_log.jsonl`**：多一行 `status: denied`——「人说了不」，这是追责时最重要的一类记录（第 9 篇定的纪律）。
- **server 侧 `audit_mcp_server.jsonl`**：**零增量**——`tools/call` 根本没发出，server 无从知晓 client 内部发生过一次拒绝。

这不是缺陷，是分层的本质：**server 只见得到「到达它的调用」，client 的本侧决定无处可去、必须留在本地**。两本账合起来才是完整的事实链：「模型请求了写操作 → 用户拒绝 → 没有任何请求过网」。如果只有 server 的账，这次拒绝就像没发生过——审计跟着保护走，保护住哪层，账就记哪层。

「权限归谁管、server 该不该信 client」是 MCP 选型的核心问题之一，第 15 篇的直连 vs MCP 对比表会回收这条线索。

## 5｜代码落地

| 文件 | 动作 | 行数 | 说明 |
|---|---|---|---|
| `admin_mcp_server.py` | 新增 | 108 | MCP stdio server：四工具声明式包装 + server 端校验/审计 |
| `mcp_client_tools.py` | 新增 | 201 | 连接层（MCPConnection）+ 注册件同形层（MCPToolset） |
| `agent.py` | 改动 | 391 | `--mcp` 开关；`toolset` 改惰性加载（`_default_toolset`）；loop 零改动 |
| `selfcheck_p14.py` | 新增 | 228 | 38 项断言 × 7 场景（零 LLM token；场景 5-7 真实 stdio 往返） |
| `demo_mcp_equiv.py` | 新增 | 233 | 两版等价验证器（桩 LLM × 真实 HTTP，零 LLM token） |
| `requirements.txt` | 改动 | 10 | 新增 `mcp==2.3.0`（2026-10-16 PyPI 核验现行版） |
| `.gitignore` | 改动 | +1 | `audit_mcp_server.jsonl`（测试产物） |
| `selfcheck_p11.py` | 同步 1 行 | — | 场景 8 的回归锚点断言随惰性加载同步（见下） |

**一处测试维护的诚实记录**：`agent.py` 把 `toolset` 改成惰性加载后，`selfcheck_p11.py` 场景 8 里「agent 模块默认工具集仍是 admin_tools」的断言（原来检查 `agent.toolset is admin_tools`）随之失效——模块级变量现在是 None，第一次用时才装。语义保证没变（默认模式拿到的仍是 admin_tools），断言改成经 `_default_toolset()` 验证同一语义。跑全量回归确认：p9（36）+ p10（29）+ p11（59）+ p12（51）+ p14（38）全绿。重构改变结构时，跟着结构走的断言要同步、且要说明同步的理由——这就是那句话的实例。

`selfcheck_p14.py` 全文（七个场景的断言即 §4 分层表的逐条证据）：

```python
# selfcheck_p14.py — 第 14 篇桩测试：MCP 链路的 7 个场景（零 LLM token）
# 覆盖（对照文章小节）：
#   场景 1  进程隔离：--mcp 模式下 agent 进程从头到尾不 import admin_tools（保护分层的
#           物理前提——工具实现只活在 server 子进程里，文章 §4）
#   场景 2  server 启停与握手：MCPConnection 拉起子进程、initialize 握手、优雅关停
#   场景 3  tools/list 动态发现：四工具 + OpenAI 信封 + 写工具的闸门提示由 client 追加
#   场景 4  自动生成 schema vs 第 7 篇手写 schema：同一份信息的两种写法，结构等价
#   场景 5  语义白名单在 server 端：畸形参数死在 server、不触网（请求不会发到后台）
#   场景 6  全链路闭环（真实 HTTP）：login→查人→查角色→分配→复核，MCP 往返 × 真实
#           mock_backend；两份审计各自如实记录
#   场景 7  确认闸门在 client 端：拒绝时 tools/call 根本不发出（server 审计零增量、
#           client 审计记 denied）；放行后 server 审计 +1
# 运行：cd agent && .venv/bin/python selfcheck_p14.py   （零 LLM token；场景 5-7 需要本机
#       起 mock_backend——脚本自己拉起一个进程内 uvicorn，不依赖外部终端）
# 诚实声明：本文件为第 14 篇的验收测试，输出即文章实践节的证据；对真实 DeepSeek 的
#           调用不在本文件范围（那是 agent.py --mcp 的路径，<未实测，以官方文档为准>）。
import json
import os
import warnings
# PyJWT 对教学短密钥（mock_backend 的 teaching-secret，15 字节）会报 InsecureKeyLengthWarning
# ——与本篇被测对象无关，过滤之；生产密钥长度要求（RFC 7518 §3.2，≥32 字节）见 mock_backend 注释。
warnings.filterwarnings("ignore", message=".*HMAC key.*", category=Warning)
import sys
import threading
import time
import types
import urllib.request

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
SERVER_AUDIT = os.path.join(BASE_DIR, "audit_mcp_server.jsonl")
CLIENT_AUDIT = os.path.join(BASE_DIR, "audit_log.jsonl")

_passed, _failed = 0, 0


def check(name: str, cond: bool):
    """计数断言：全部场景跑完统一汇报（与 selfcheck_p9/p10/p11/p12 同一惯例）。"""
    global _passed, _failed
    if cond:
        _passed += 1
        print(f"  PASS {name}")
    else:
        _failed += 1
        print(f"  FAIL {name}")


def truncate(path: str):
    """清空审计文件：给「增量断言」一个确定的起点（两个文件都是 gitignored 测试产物）。"""
    with open(path, "w", encoding="utf-8") as f:
        f.write("")


def audit_lines(path: str) -> list[dict]:
    if not os.path.exists(path):
        return []
    with open(path, encoding="utf-8") as f:
        return [json.loads(line) for line in f if line.strip()]


def start_backend() -> bool:
    """进程内起 mock_backend（uvicorn 线程）。返回是否就绪。
    若 8000 端口已有服务在听（比如读者自己起的），直接复用它——状态由调用方负责重置。"""
    import uvicorn
    from mock_backend import main as mock_main

    def probe() -> bool:
        try:
            body = json.dumps({"username": "admin", "password": "admin123"}).encode()
            req = urllib.request.Request("http://127.0.0.1:8000/api/v1/auth/login",
                                         data=body,
                                         headers={"Content-Type": "application/json"},
                                         method="POST")
            with urllib.request.urlopen(req, timeout=1.0) as resp:
                return resp.status == 200
        except Exception:
            return False

    if probe():          # 已有服务在听：复用（教学脚手架内存态，由场景 6 前的重置保证确定性）
        return True
    server = uvicorn.Server(uvicorn.Config(mock_main.app, host="127.0.0.1",
                                           port=8000, log_level="warning"))
    threading.Thread(target=server.run, daemon=True, name="mock-backend").start()
    for _ in range(50):  # 最多等 5 秒
        if probe():
            return True
        time.sleep(0.1)
    return False


def reset_zhangsan_to_viewer():
    """把张三重置为查看者（覆盖式分配的逆操作），给场景 6 一个确定性的起点。"""
    import admin_tools   # 借直连版做重置——它和 MCP 版打的是同一个后台
    admin_tools.login("admin", "admin123")
    admin_tools.assign_role("u-1001", "r-viewer")


class FakeCall:
    """第 9 篇 selfcheck 同款桩：伪装 SDK 的 tool_call 对象（只用到 execute_tool 读的字段）。"""

    def __init__(self, name: str, args_json: str):
        self.function = types.SimpleNamespace(name=name, arguments=args_json)


print("=== 场景 1：进程隔离——MCP 模式下 agent 进程不 import admin_tools ===")
# 先跑（顺序就是断言的一部分）：此刻本进程还没 import 过 admin_tools；连接 MCP server
# 只拉起子进程，server 代码在子进程里跑——本进程的 sys.modules 不该出现这两个名字。
import mcp_client_tools   # noqa: E402  （只 import client 侧：guard + tools，无 admin_tools）

ts = mcp_client_tools.build()
check("连接后本进程未加载 admin_tools", "admin_tools" not in sys.modules)
check("连接后本进程未加载 admin_mcp_server（server 只活在子进程）",
      "admin_mcp_server" not in sys.modules)

print("=== 场景 2：server 启停与握手 ===")
check("serverInfo 自报家门 admin-tools/0.1",
      ts.server_info == {"name": "admin-tools", "version": "0.1"})
check(f"握手协商出协议版本（实测 {ts.protocol_version}）",
      str(ts.protocol_version).startswith("20"))

print("=== 场景 3：tools/list 动态发现（OpenAI 信封 + client 侧闸门提示） ===")
discovered = sorted(ts.TOOL_REGISTRY)
check("发现四个工具", discovered == ["assign_role", "get_user_roles", "get_users", "login"])
envelope_ok = all(s["type"] == "function" and set(s["function"]) >= {"name", "description", "parameters"}
                  for s in ts.TOOL_SCHEMAS)
check("TOOL_SCHEMAS 已翻译成 OpenAI 函数信封", envelope_ok)
raw = ts._conn.list_tools().tools        # 原始发现结果（client 改写之前的那份）
raw_desc = {t.name: t.description or "" for t in raw}
check("server 原始描述不承诺终端确认（闸门是 client 的策略）",
      "终端确认" not in raw_desc["assign_role"])
check("client 拼装后的写工具描述带闸门提示",
      "终端确认" in ts.SCHEMA_BY_NAME["assign_role"]["description"])
check("读工具描述不带闸门提示（防警报疲劳，第 9 篇同一纪律）",
      all("终端确认" not in ts.SCHEMA_BY_NAME[n]["description"]
          for n in ("login", "get_users", "get_user_roles")))

print("=== 场景 4：@mcp.tool() 自动 schema vs 第 7 篇手写 schema ===")
import admin_tools   # noqa: E402  （从这起才 import——场景 1 的隔离断言已经跑完）
for name in ("login", "get_users", "get_user_roles", "assign_role"):
    hand = admin_tools.SCHEMA_BY_NAME[name]
    auto = ts.SCHEMA_BY_NAME[name]
    same_required = sorted(hand["parameters"].get("required", [])) == \
        sorted(auto["parameters"].get("required", []))
    check(f"{name}：必填参数一致", same_required)
    hand_props = hand["parameters"]["properties"]
    auto_props = auto["parameters"]["properties"]
    same_names = set(hand_props) == set(auto_props)
    check(f"{name}：属性名一致", same_names)
    type_map = {"string": "string", "integer": "integer"}
    types_match = all(
        (auto_props[p].get("type") in (hand_props[p]["type"], "string")) or
        hand_props[p]["type"] == type_map.get(auto_props[p].get("type"), "?")
        for p in hand_props)
    check(f"{name}：属性类型一致（pydantic int→integer / str→string）", types_match)

print("=== 场景 5：语义白名单在 server 端（畸形参数死在 server、不触网） ===")
truncate(SERVER_AUDIT)   # 增量断言的起点：此刻 server 审计应为空
r_bad = json.loads(ts.TOOL_REGISTRY["assign_role"](user_id="../../x", role_id="r-admin"))
check("路径遍历载荷被 server 校验层拒绝（error:rejected）",
      r_bad.get("code") == "rejected" and "u-" in r_bad.get("detail", ""))
r_role = json.loads(ts.TOOL_REGISTRY["assign_role"](user_id="u-1001", role_id="r-hacker"))
check("未知角色被枚举白名单拒绝", r_role.get("code") == "rejected")
server_rows = audit_lines(SERVER_AUDIT)
check("server 审计如实记录被拒调用（2 行 error:rejected）",
      len(server_rows) == 2 and all(r["status"] == "error:rejected" for r in server_rows))

print("=== 场景 6：全链路闭环（MCP 往返 × 真实 HTTP） ===")
ready = start_backend()
check("mock_backend 就绪（进程内 uvicorn）", ready)
if ready:
    reset_zhangsan_to_viewer()
    r_login = json.loads(ts.TOOL_REGISTRY["login"](username="admin", password="admin123"))
    check("MCP login 成功且摘除令牌", r_login.get("status") == "ok" and "access_token" not in r_login)
    r_users = json.loads(ts.TOOL_REGISTRY["get_users"](keyword="张三"))
    check("MCP 搜到张三 u-1001",
          r_users.get("total") == 1 and r_users["users"][0]["user_id"] == "u-1001")
    r_roles0 = json.loads(ts.TOOL_REGISTRY["get_user_roles"](user_id="u-1001"))
    check("改前张三是查看者", [r["role_id"] for r in r_roles0["roles"]] == ["r-viewer"])
    r_assign = json.loads(ts.TOOL_REGISTRY["assign_role"](user_id="u-1001", role_id="r-admin"))
    check("MCP 分配管理员成功（覆盖式）",
          [r["role_id"] for r in r_assign["roles"]] == ["r-admin"])
    r_roles1 = json.loads(ts.TOOL_REGISTRY["get_user_roles"](user_id="u-1001"))
    check("复核：张三已是管理员", [r["role_id"] for r in r_roles1["roles"]] == ["r-admin"])
    ok_rows = [r for r in audit_lines(SERVER_AUDIT) if r["status"] == "ok"]
    check("server 审计记录 5 次真实执行的调用", len(ok_rows) == 5)
    check("server 审计令牌/密码打码（隐私红线不因换进程而放松）",
          all("access_token" not in json.dumps(r, ensure_ascii=False) for r in audit_lines(SERVER_AUDIT)))

print("=== 场景 7：确认闸门在 client 端 + 审计双文件 ===")
import agent   # noqa: E402  （execute_tool 是闸门与审计的挂点；agent 不直接 import admin_tools）
import guard   # noqa: E402

truncate(CLIENT_AUDIT)
server_before = len(audit_lines(SERVER_AUDIT))
agent.toolset = ts                       # 把 agent 的工具集切到 MCP 版（--mcp 的等价动作）
answers = iter(["n", "y"])               # 第一次拒绝、第二次放行——两种命运的对照
real_confirm = guard.confirm
def scripted_confirm(name, args, _input=input):
    return real_confirm(name, args, _input=lambda _prompt: next(answers))
guard.confirm = scripted_confirm

r_denied = agent.execute_tool(FakeCall(
    "assign_role", json.dumps({"user_id": "u-1001", "role_id": "r-editor"},
                              ensure_ascii=False)))
guard.confirm = real_confirm             # 用完立刻还原，不污染后续场景
check("用户拒绝：回喂文本是「最终决定」话术（不是可自愈错误）", "最终决定" in r_denied)
check("拒绝时 server 审计零增量（tools/call 根本没发出）",
      len(audit_lines(SERVER_AUDIT)) == server_before)
client_rows = audit_lines(CLIENT_AUDIT)
check("拒绝记录留在 client 审计（status=denied）",
      len(client_rows) == 1 and client_rows[0]["status"] == "denied")

guard.confirm = scripted_confirm         # 放行路径：y → 真实 tools/call → server 审计 +1
r_allowed = agent.execute_tool(FakeCall(
    "assign_role", json.dumps({"user_id": "u-1001", "role_id": "r-editor"},
                              ensure_ascii=False)))
guard.confirm = real_confirm
r_allowed_json = json.loads(r_allowed)
check("放行后 MCP 分配成功（r-editor）",
      [r["role_id"] for r in r_allowed_json["roles"]] == ["r-editor"])
check("放行使 server 审计 +1（这次真的过网了）",
      len(audit_lines(SERVER_AUDIT)) == server_before + 1)
client_rows = audit_lines(CLIENT_AUDIT)
check("client 审计两行齐备（denied → ok），保护分层各有各的账本",
      [r["status"] for r in client_rows] == ["denied", "ok"])

ts.close()
print(f"\n=== selfcheck_p14：{_passed} 项断言通过，{_failed} 项失败 ===")
sys.exit(1 if _failed else 0)
```

`demo_mcp_equiv.py` 全文（两版等价的三件验收口径写在文件头）：

```python
# demo_mcp_equiv.py — 第 14 篇实践：同一句话，直连版与 MCP 版跑同一条链（零 LLM token）
# 「等价」的验收口径（文章实践节的三件事）：
#   ① 两版触发的工具序列相同（同名工具、同参数——行为等价）；
#   ② 两版的关键结果语义相同（逐轮 JSON 对照：登录态、查人、改前现状、改后结果）；
#   ③ 终局后台状态相同（张三都是 r-admin）。
# 模型用剧本桩（5 轮：login→搜人→查角色→分配→答复）——真实模型只影响措辞与轮次编排，
# 不影响「同一 loop、同一保护层、同一后台」这条被验证的链路；真实 DeepSeek 跑法见文末。
# 运行：cd agent && .venv/bin/python demo_mcp_equiv.py
#       （自管后台：进程内 uvicorn；自管闸门：guard.confirm 剧本注入 y）
# 诚实声明：本文件已在本机实测（输出即文章实践节证据）；真实 DeepSeek 调用
#           <未实测，以官方文档为准>，命令在文末注释。
import json
import os
import warnings
# PyJWT 对教学短密钥（mock_backend 的 teaching-secret，15 字节）会报 InsecureKeyLengthWarning
# ——与本篇被测对象无关，过滤之；生产密钥长度要求（RFC 7518 §3.2，≥32 字节）见 mock_backend 注释。
warnings.filterwarnings("ignore", message=".*HMAC key.*", category=Warning)
import sys
import threading
import time
import types
import urllib.request

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

import admin_tools        # noqa: E402  A 组：直连版（工具与本进程同住）
import agent              # noqa: E402  同一个 loop，喂不同的 toolset
import guard              # noqa: E402  闸门在 client：两版共用，剧本注入 y
import mcp_client_tools   # noqa: E402  B 组：MCP 版（工具在 stdio 子进程）
from mock_backend import main as mock_main   # noqa: E402  进程内后台（状态可直接重置）


# ---------- 剧本桩 LLM：把「把张三提升为管理员」演成 5 轮标准链 ----------
def _call(idx, name, **args):
    return types.SimpleNamespace(
        id=f"call_equiv_{idx}", type="function",
        function=types.SimpleNamespace(name=name,
                                       arguments=json.dumps(args, ensure_ascii=False)))


SCRIPT = [
    ("tool", _call(1, "login", username="admin", password="admin123")),
    ("tool", _call(2, "get_users", page=1, page_size=20, keyword="张三")),
    ("tool", _call(3, "get_user_roles", user_id="u-1001")),
    ("tool", _call(4, "assign_role", user_id="u-1001", role_id="r-admin")),
    ("text", "已把张三（u-1001）的角色设置为管理员（r-admin）。"),
]


class ScriptedLLM:
    """与 selfcheck_p9 同思路的桩 client：chat.completions.create 按剧本出牌。
    只实现 run_agent 会读到的字段（choices[0].message / finish_reason）。"""

    def __init__(self):
        self._i = 0
        outer = self

        class _Completions:
            def create(self, **kwargs):
                return outer._next()

        class _Chat:
            completions = _Completions()
        self.chat = _Chat()

    def _next(self):
        kind, payload = SCRIPT[self._i]
        self._i += 1
        if kind == "text":
            msg = types.SimpleNamespace(content=payload, reasoning_content=None, tool_calls=None)
            return types.SimpleNamespace(choices=[types.SimpleNamespace(
                message=msg, finish_reason="stop")])
        msg = types.SimpleNamespace(content=None, reasoning_content=None, tool_calls=[payload])
        return types.SimpleNamespace(choices=[types.SimpleNamespace(
            message=msg, finish_reason="tool_calls")])


def record_calls(dest: list):
    """给 agent.execute_tool 套一层记录器（调用原实现，只旁听）——两版各挂各的。"""
    real_exec = agent.execute_tool

    def recording_exec(call):
        result = real_exec(call)
        dest.append({"tool": call.function.name,
                     "args": json.loads(call.function.arguments), "result": result})
        return result
    agent.execute_tool = recording_exec
    return real_exec


def auto_yes(confirm_answers: list):
    """闸门剧本注入：把 guard.confirm 的终端输入换成预设应答（这里恒放行）。"""
    real_confirm = guard.confirm

    def scripted(name, args, _input=input):
        return real_confirm(name, args,
                            _input=lambda _prompt: confirm_answers.pop(0))
    guard.confirm = scripted
    return real_confirm


def start_backend():
    """进程内 uvicorn（与 selfcheck_p14 同法）。复用已监听的服务时状态由 reset 保证。"""
    import uvicorn

    def probe() -> bool:
        try:
            body = json.dumps({"username": "admin", "password": "admin123"}).encode()
            req = urllib.request.Request("http://127.0.0.1:8000/api/v1/auth/login",
                                         data=body, headers={"Content-Type": "application/json"},
                                         method="POST")
            with urllib.request.urlopen(req, timeout=1.0) as resp:
                return resp.status == 200
        except Exception:
            return False

    if probe():
        return True
    server = uvicorn.Server(uvicorn.Config(mock_main.app, host="127.0.0.1",
                                           port=8000, log_level="warning"))
    threading.Thread(target=server.run, daemon=True, name="mock-backend").start()
    for _ in range(50):
        if probe():
            return True
        time.sleep(0.1)
    return False


def _admin_token() -> str:
    """裸 HTTP 登录换令牌：重置与终局核对都「不经过任何工具层」，独立于被测对象。"""
    body = json.dumps({"username": "admin", "password": "admin123"}).encode()
    req = urllib.request.Request("http://127.0.0.1:8000/api/v1/auth/login", data=body,
                                 headers={"Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=5.0) as resp:
        return json.loads(resp.read().decode())["access_token"]


def reset_zhangsan_to_viewer():
    """把张三重置为查看者。走 HTTP 而非进程内改字典：后台无论在不在本进程
    （复用已运行的外部实例时，进程内改字典对它无效——这正是首次实测踩出的坑），
    覆盖式分配的逆操作都成立。"""
    req = urllib.request.Request(
        "http://127.0.0.1:8000/api/v1/users/u-1001/roles",
        data=json.dumps({"role_id": "r-viewer"}).encode(),
        headers={"Content-Type": "application/json",
                 "Authorization": f"Bearer {_admin_token()}"},
        method="POST")
    with urllib.request.urlopen(req, timeout=5.0) as resp:
        json.loads(resp.read().decode())


def zhangsan_roles() -> list[str]:
    """从后台读张三的当前角色（终局状态的独立核对，不经过任何工具层）。"""
    req = urllib.request.Request("http://127.0.0.1:8000/api/v1/users/u-1001/roles",
                                 headers={"Authorization": f"Bearer {_admin_token()}"})
    with urllib.request.urlopen(req, timeout=5.0) as resp:
        return [r["role_id"] for r in json.loads(resp.read().decode())["roles"]]


# ---------- 演示开始 ----------
print("=== 演示：同一句话「把张三提升为管理员」，两版各跑一遍（零 LLM token） ===")
assert start_backend(), "mock_backend 未就绪"
print("[后台] mock_backend 已就绪（127.0.0.1:8000；重置与核对走 HTTP，不依赖后台在哪个进程）")

runs = {}

# --- A 组：直连版（admin_tools 进程内 import；工具与 loop 同进程） ---
reset_zhangsan_to_viewer()                                # 确定性起点（HTTP，两种后台都成立）
agent.toolset = admin_tools
calls_a, answers = [], ["y"]
real_exec, real_confirm = record_calls(calls_a), auto_yes(answers)
print("\n--- A. 直连版：python agent.py \"把张三提升为管理员\" ---")
answer_a = agent.run_safely(ScriptedLLM(), "把张三提升为管理员")
agent.execute_tool, guard.confirm = real_exec, real_confirm
print(f"[A 终局] 张三角色 = {zhangsan_roles()}（独立核对，不经过工具层）")
runs["A 直连版"] = {"calls": calls_a, "answer": answer_a}

# --- B 组：MCP 版（stdio 子进程；工具发现在连接期完成） ---
reset_zhangsan_to_viewer()                                # 同一确定性起点（HTTP，两种后台都成立）
ts = mcp_client_tools.build()
print(f"\n--- B. MCP 版：python agent.py --mcp \"把张三提升为管理员\" ---")
print(f"[MCP] 已连接 server {ts.server_info}（协议 {ts.protocol_version}），"
      f"发现 {len(ts.TOOL_SCHEMAS)} 个工具：{sorted(ts.TOOL_REGISTRY)}")
agent.toolset = ts
calls_b, answers = [], ["y"]
real_exec, real_confirm = record_calls(calls_b), auto_yes(answers)
answer_b = agent.run_safely(ScriptedLLM(), "把张三提升为管理员")
agent.execute_tool, guard.confirm = real_exec, real_confirm
ts.close()
print(f"[B 终局] 张三角色 = {zhangsan_roles()}（独立核对，不经过工具层）")
runs["B MCP 版"] = {"calls": calls_b, "answer": answer_b}

# ---------- 等价性对照 ----------
print("\n=== 等价性对照（逐轮：工具序列 + 关键结果语义） ===")
a, b = runs["A 直连版"], runs["B MCP 版"]
passed, failed = 0, 0


def check(name, cond):
    global passed, failed
    passed, failed = passed + (1 if cond else 0), failed + (0 if cond else 1)
    print(f"  {'PASS' if cond else 'FAIL'} {name}")


check("① 工具序列相同（同名工具、同参数）",
      [(c["tool"], c["args"]) for c in a["calls"]] == [(c["tool"], c["args"]) for c in b["calls"]])


def semantic(result_text: str) -> dict:
    """把两版的结果 JSON 归一成可比较的语义键（两版缩进不同，语义相同）。"""
    data = json.loads(result_text)
    if "users" in data:
        return {"total": data["total"],
                "first_user": (data["users"][0]["user_id"], data["users"][0]["username"])}
    if "roles" in data:
        return {"user_id": data.get("user_id"), "roles": sorted(r["role_id"] for r in data["roles"])}
    if "status" in data or "assignable_roles" in data:
        return {"status": data.get("status"),
                "assignable": sorted(r["role_id"] for r in data.get("assignable_roles", []))}
    return data


for i, (ca, cb) in enumerate(zip(a["calls"], b["calls"]), 1):
    check(f"② 第 {i} 轮 {ca['tool']}：结果语义相同", semantic(ca["result"]) == semantic(cb["result"]))
check("③ 终局后台状态相同（张三都是 r-admin）", zhangsan_roles() == ["r-admin"])
check("附：两版最终答复文本相同（剧本桩下必然）", a["answer"] == b["answer"])

print(f"\n=== demo_mcp_equiv：{passed} 项通过，{failed} 项失败 ===")
print("两版差异的正确打开方式：A 的工具在本进程 import；B 的工具在 server 子进程、")
print("经 tools/list 发现、tools/call 调用——loop 与保护层（闸门/审计/校验）完全没变。")
print("真实模型验证（需要 DEEPSEEK_API_KEY，<未实测，以官方文档为准>）：")
print('  python agent.py "把张三提升为管理员"        # 直连版')
print('  python agent.py --mcp "把张三提升为管理员"   # MCP 版')
sys.exit(1 if failed else 0)
```

## 动手实践：一键跑通 + 同一句话两版等价

环境即第 5 篇的 venv（Python 3.12），依赖 `pip install -r requirements.txt`（新增 mcp 2.3.0）。三个实践递进：先机制断言，再两版对照，最后真实模型（可选）。

### 实践①：selfcheck_p14——38 项断言（零 LLM token）

```bash
cd agent
.venv/bin/python selfcheck_p14.py
```

预期输出（2026-10-19 本机实测原文，全 38 项 PASS；场景 5-7 会自动拉起进程内 mock_backend，不需要另开终端）：

```text
=== 场景 1：进程隔离——MCP 模式下 agent 进程不 import admin_tools ===
  PASS 连接后本进程未加载 admin_tools
  PASS 连接后本进程未加载 admin_mcp_server（server 只活在子进程）
=== 场景 2：server 启停与握手 ===
  PASS serverInfo 自报家门 admin-tools/0.1
  PASS 握手协商出协议版本（实测 2025-11-25）
=== 场景 3：tools/list 动态发现（OpenAI 信封 + client 侧闸门提示） ===
  PASS 发现四个工具
  PASS TOOL_SCHEMAS 已翻译成 OpenAI 函数信封
  PASS server 原始描述不承诺终端确认（闸门是 client 的策略）
  PASS client 拼装后的写工具描述带闸门提示
  PASS 读工具描述不带闸门提示（防警报疲劳，第 9 篇同一纪律）
=== 场景 4：@mcp.tool() 自动 schema vs 第 7 篇手写 schema ===
  PASS login：必填参数一致
  PASS login：属性名一致
  PASS login：属性类型一致（pydantic int→integer / str→string）
  PASS get_users：必填参数一致
  PASS get_users：属性名一致
  PASS get_users：属性类型一致（pydantic int→integer / str→string）
  PASS get_user_roles：必填参数一致
  PASS get_user_roles：属性名一致
  PASS get_user_roles：属性类型一致（pydantic int→integer / str→string）
  PASS assign_role：必填参数一致
  PASS assign_role：属性名一致
  PASS assign_role：属性类型一致（pydantic int→integer / str→string）
=== 场景 5：语义白名单在 server 端（畸形参数死在 server、不触网） ===
  PASS 路径遍历载荷被 server 校验层拒绝（error:rejected）
  PASS 未知角色被枚举白名单拒绝
  PASS server 审计如实记录被拒调用（2 行 error:rejected）
=== 场景 6：全链路闭环（MCP 往返 × 真实 HTTP） ===
  PASS mock_backend 就绪（进程内 uvicorn）
  PASS MCP login 成功且摘除令牌
  PASS MCP 搜到张三 u-1001
  PASS 改前张三是查看者
  PASS MCP 分配管理员成功（覆盖式）
  PASS 复核：张三已是管理员
  PASS server 审计记录 5 次真实执行的调用
  PASS server 审计令牌/密码打码（隐私红线不因换进程而放松）
=== 场景 7：确认闸门在 client 端 + 审计双文件 ===
[确认] 用户拒绝
  PASS 用户拒绝：回喂文本是「最终决定」话术（不是可自愈错误）
  PASS 拒绝时 server 审计零增量（tools/call 根本没发出）
  PASS 拒绝记录留在 client 审计（status=denied）
[确认] 用户放行
  PASS 放行后 MCP 分配成功（r-editor）
  PASS 放行使 server 审计 +1（这次真的过网了）
  PASS client 审计两行齐备（denied → ok），保护分层各有各的账本

=== selfcheck_p14：38 项断言通过，0 项失败 ===
```

七个场景对照 §4 的分层表逐条读：场景 1 是物理前提（进程隔离），场景 5-6 是 server 侧住址（校验/审计），场景 7 是 client 侧住址（闸门 + 双审计的 denied/零增量对照）。

### 实践②：demo_mcp_equiv——同一句话，两版各跑一遍

```bash
.venv/bin/python demo_mcp_equiv.py
```

剧本桩 LLM 把「把张三提升为管理员」演成标准 5 轮链（login → 搜人 → 查角色 → 分配 → 答复），直连版与 MCP 版各跑一遍（每版跑前都把张三重置回查看者）。A 段（直连版）输出与第 8 篇 M2 闭环同构，看 B 段（MCP 版）的开头与两版对照（2026-10-19 本机实测原文节选）：

```text
--- B. MCP 版：python agent.py --mcp "把张三提升为管理员" ---
[MCP] 已连接 server {'name': 'admin-tools', 'version': '0.1'}（协议 2025-11-25），发现 4 个工具：['assign_role', 'get_user_roles', 'get_users', 'login']

[用户] 把张三提升为管理员
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] login({"username": "admin", "password": "admin123"})
[工具→] {
  "token_type": "Bearer",
  "expires_in": 3600,
  "assignable_roles": [
    {
      "role_id": "r-admin",
      "role_name": "管理员"
    },
    {
      "role_id": "r-editor",
      "role_name": "编辑者"
    },
    {
      "role_id": "r-viewer",
      "role_name": "查看者"
    }
  ],
  "status": "ok"
}
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_users({"page": 1, "page_size": 20, "keyword": "张三"})
[工具→] {
  "total": 1,
  "page": 1,
  "page_size": 20,
  "users": [
    {
      "user_id": "u-1001",
      "username": "张三",
      "status": "active",
      "role_ids": [
        "r-viewer"
      ]
    }
  ]
}
[loop] 第 3 轮 | 发送 6 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_user_roles({"user_id": "u-1001"})
[工具→] {
  "user_id": "u-1001",
  "username": "张三",
  "roles": [
    {
      "role_id": "r-viewer",
      "role_name": "查看者"
    }
  ]
}
[loop] 第 4 轮 | 发送 8 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] assign_role({"user_id": "u-1001", "role_id": "r-admin"})
[确认] 用户放行
[工具→] {
  "user_id": "u-1001",
  "username": "张三",
  "roles": [
    {
      "role_id": "r-admin",
      "role_name": "管理员"
    }
  ]
}
[loop] 第 5 轮 | 发送 10 条消息
[loop] finish_reason=stop | tool_calls=0 个
[模型] 已把张三（u-1001）的角色设置为管理员（r-admin）。
[loop] 终止：共 5 轮
[B 终局] 张三角色 = ['r-admin']（独立核对，不经过工具层）

=== 等价性对照（逐轮：工具序列 + 关键结果语义） ===
  PASS ① 工具序列相同（同名工具、同参数）
  PASS ② 第 1 轮 login：结果语义相同
  PASS ② 第 2 轮 get_users：结果语义相同
  PASS ② 第 3 轮 get_user_roles：结果语义相同
  PASS ② 第 4 轮 assign_role：结果语义相同
  PASS ③ 终局后台状态相同（张三都是 r-admin）
  PASS 附：两版最终答复文本相同（剧本桩下必然）

=== demo_mcp_equiv：7 项通过，0 项失败 ===
两版差异的正确打开方式：A 的工具在本进程 import；B 的工具在 server 子进程、
经 tools/list 发现、tools/call 调用——loop 与保护层（闸门/审计/校验）完全没变。
真实模型验证（需要 DEEPSEEK_API_KEY，<未实测，以官方文档为准>）：
  python agent.py "把张三提升为管理员"        # 直连版
  python agent.py --mcp "把张三提升为管理员"   # MCP 版
```

等价性判定的 7 项全 PASS：① 工具序列相同；② 逐轮结果语义相同（两版 JSON 缩进不同——直连版 `json.dumps` 单行、MCP 版 server 序列化带缩进——语义键归一后逐项相等）；③ 终局后台状态相同；附：剧本桩下最终答复文本相同。

**一个实测踩出的坑**，值得写进你的备忘录：demo 最初用「进程内改 `mock_main.USERS` 字典」做两版之间的状态重置，第一次跑就 FAIL 在第 3 轮——因为 8000 端口上还挂着一个先前调试留下的**外部** mock_backend 进程，进程内改字典对它无效。教训：凡是要对「被测系统的状态」做确定性重置，重置动作必须走系统自己的接口（HTTP），而不是旁路进程内细节——重置手段要和被测对象在同一层。demo 现在的 `reset_zhangsan_to_viewer()` 走 HTTP 分配接口，无论后台在哪个进程都成立。

### 实践③：真实模型两连（有条件选做，`<未实测，以官方文档为准>`）

```bash
export DEEPSEEK_API_KEY=sk-xxx
.venv/bin/python agent.py "把张三提升为管理员"        # 直连版
.venv/bin/python agent.py --mcp "把张三提升为管理员"   # MCP 版
```

两版对同一句话的行为应与 demo 的剧本链一致（登录 → 搜人 → 查角色 → 确认闸门 → 分配 → 答复）；MCP 版开头会多一行 `[MCP] 已连接 server ...` 的装载日志。差异只在措辞与轮次编排——那是模型的自由度，不是链路的差异。

### 自检清单

1. `selfcheck_p14.py` 38 项全 PASS（场景 5-7 无需手动起后台）；
2. `demo_mcp_equiv.py` 7 项全 PASS，两段终局都是 `['r-admin']`；
3. B 段开头的装载日志能看到 `协议 2025-11-25`（握手期协商结果）与 4 个工具名；
4. 拒绝路径：把 selfcheck 场景 7 的 `answers` 改成 `["n", "n"]`，断言会 FAIL 在「放行后」两项——读 FAIL 信息确认 server 审计确实零增量；
5. `grep print agent/admin_mcp_server.py`：除注释外无输出（stdout 纪律）；
6. 回归：p9/p10/p11/p12 全绿（惰性加载没有破坏任何前序保证）。

## 本篇结论

1. `@mcp.tool()` 把「手写 schema + 注册实现」合成「签名即 schema」——同一份信息从两份变一份，selfcheck 场景 4 证明两种写法结构等价。
2. 注册件同形让 MCP 化的 loop 改动为零；真正的设计工作在保护分层：闸门留 client（server 没终端、承诺归 client），校验与审计进 server（工具唯一入口），双审计各记各账。
3. 同一句话两版等价已被桩测证明；下一步的问题只有一个——这多出来的一个 server 进程和一层协议，到底**值不值**？第 15 篇把这笔账算清，并第一次把工具接上真实 Go 后台。

## 参考资料

1. [MCP 官方 Python SDK（modelcontextprotocol/python-sdk）](https://github.com/modelcontextprotocol/python-sdk) — MCPServer（原 FastMCP）的声明式定义、stdio 运行、client API。2026-10-16 核验（第 13 篇），本篇实测版本 2.3.0。
2. [mcp — PyPI](https://pypi.org/project/mcp/) — 版本与 requires-python。2026-10-16 PyPI JSON API 核验：2.3.0。
3. [MCP 规范仓库（modelcontextprotocol/modelcontextprotocol）](https://github.com/modelcontextprotocol/modelcontextprotocol) — 规范 tag `2026-07-28`（2026-10-15 GitHub tags API 核验仍为最新）；握手期与无状态期的分野见第 13 篇 §5.1。
4. [MCP 规范站（modelcontextprotocol.io）](https://modelcontextprotocol.io/) — tools/list / tools/call 的 schema 定义。2026-10-16 实测抓取（第 13 篇）。
5. [第 13 篇：MCP 是什么，为什么值得学](13-MCP是什么.md) — JSON-RPC 报文逐字段解剖、stdio 传输、双时代口径、FastMCP→MCPServer 更名实测。
6. [第 10 篇：工具全谱系与安全边界](../part3-接上真实系统/10-工具全谱系与安全边界.md) — 语义白名单（EXTRA_VALIDATORS）的设计理由，本篇 server 端原样复用。
7. [第 9 篇：真实世界的工具工程设计](../part3-接上真实系统/09-工具工程设计.md) — 确认闸门 fail-closed 与审计 denied 单独成态的纪律，本篇分层表的上游。

> 核验说明：本篇全部 stdio 往返、真实 HTTP 闭环、38+7 项断言与两版等价对照均在 2026-10-19 本机实测（Python 3.12.15 + mcp 2.3.0 + fastapi 0.143.1）；真实 DeepSeek 调用 `<未实测，以官方文档为准>`。PyJWT 对教学短密钥的 InsecureKeyLengthWarning（RFC 7518 §3.2，≥32 字节建议）与本篇被测对象无关，测试脚本内过滤；生产密钥管理见 P7·20。
