# 第 8 篇：把后台 API 变成 Agent 的工具 — 第一个闭环

> **知识地图【B1/F1】· Part 3**
> Part 2 的 agent 只会玩两个演示工具；从本篇起主线落地——工具换成「登录 → 查用户 → 查角色 → 分配角色」的真实 REST 接口，里程碑 **M2** 在本篇动手实践收口。本篇节奏：先看它跑起来什么样（§1 起一个 100 行的假后台，实践节一句话闭环），再拆怎么做的（§2 工具设计模式、§3 loop 复用）。前情：agent loop 五阶段（[第 3 篇](../part2-亲手写出第一个Agent/03-agent-loop与上下文.md)）、HTTP 裸写法与 key 管理（[第 6 篇](../part2-亲手写出第一个Agent/06-调用LLM的三种姿势.md)）、`agent/` 项目最小版（[第 7 篇](../part2-亲手写出第一个Agent/07-手写最小agent.md)）。

## 本篇要回答的问题

1. 「最小目标服务」是什么？为什么是 100 行的 Python 脚手架，而不是现在就去连真实 Go 后台？
2. REST 接口怎么变成工具？与第 7 篇的本地函数工具相比，什么变了，什么必须不变？
3. 认证怎么设计？登录为什么本身就该是一个工具、token 存在哪、谁负责把 Authorization 头带上？
4. 模型怎么「自己学会先登录」？用户一句话里没提登录，agent 为什么知道第一轮先调 login？
5. 动手实践（**里程碑 M2**）：一句话「把张三提升为管理员」，agent 自动完成 登录→查用户→查角色→分配→答复 的完整闭环，结果可核对。

## 1｜最小目标服务：给 agent 一个能打的真实 HTTP 后台

读完本节你能理解：这个 100 行的脚手架是什么、不是什么，四个接口为什么长那样，以及它和真实 Go 后台的关系。

### 1.1 定位：它是练手靶子，不是落地成品

大白话先行：**最小目标服务就是一个"假后台"——一个真的 HTTP 服务器，但数据在内存里、代码只有 100 行，专门给 agent 练手用**。

为什么不直接连真实 Go 后台？三个理由：

1. **真实后台还在建**（本项目 Go 后台的业务 REST 在建），教学要求"每步可跑"，等不起；
2. **可重置可复现**：内存数据重启即清空，张三永远从"查看者"开始——错误场景随时重演，这在共享的真后台上做不到；
3. **更重要的是：接谁都不影响 agent**。成熟方案里 agent 只通过 HTTP 看后端——模型从不感知 HTTP，感知它的是工具层（[第 1 篇](../part1-打开黑盒/01-全景与学习路线地图.md)的三方分工），至于后端是 Go 还是 Python 写的，agent 一无所知也不必知道。所以用 Python 脚手架验证 agent，将来接 Go 后台时**理论上只改 `admin_tools.py` 的 `BASE_URL` 一行**——这是本篇工具设计最重要的性质，§2 展开。

对照全系列的"成熟方案三条路线"（行业把 agent 接入既有系统的通用做法归纳为 SDK 直连现有 REST / MCP Server 包装 / 企业平台三条，2026 现状基线，第 15/21 篇详述）：本脚手架对应**路线 ①（SDK 直连现有 REST）的前置验证**——工具层包一层 HTTP client，不动老系统，最快验证 agent 本身。

### 1.2 四个接口：恰好够一个管理闭环

接口清单（路径前缀 `/api/v1`，与真实后台惯例一致）：

| 接口 | 方法 | 作用 | 给闭环提供的原料 |
|---|---|---|---|
| `/api/v1/auth/login` | POST | 用户名+密码换 JWT | 令牌 + 可分配角色清单 |
| `/api/v1/users` | GET | 分页查用户，支持 keyword 模糊搜 | user_id（下一步的钥匙） |
| `/api/v1/users/{user_id}/roles` | GET | 查用户当前角色 | 变更前的现状 |
| `/api/v1/users/{user_id}/roles` | POST | 设置用户角色（覆盖式） | 闭环的"动作" |

两个非显然的设计决定，值得停下来想：

**一、登录响应里带了 `assignable_roles`（系统里可分配的角色清单）。** 这不是随手加的字段：想闭环"把张三提升为管理员"，agent 必须知道"管理员"对应的 `role_id` 是什么——但全系统只有 4 个工具，没有"列角色"工具。把角色清单挂在登录响应里既真实（不少系统的登录响应本来就带用户可用的权限/角色信息）又省一个工具——四个工具恰好够闭环。

**二、分配角色是覆盖式（把用户角色整体替换为指定的这一个），不是追加。** 一方面它让"先查现状再动手"变得必要——GET 与 POST 两个接口因此形成"读-改-写"链；另一方面它是个真正有破坏性的写操作，§2.3 的风险分级表和第 9 篇的确认闸门都要靠它当教材。

另外两处小设计直接为 agent 服务：`keyword` 参数让 agent 能用名字搜人（模型的强项是语义匹配，不是记 ID）；401 的错误文案写成"缺少访问令牌：请先调 POST /api/v1/auth/login 登录"——**错误文本是写给模型读的**，第 9 篇的自动重登就吃这口饭。

### 1.3 完整代码：mock_backend/main.py

```python
# mock_backend/main.py — 第 8 篇：最小目标服务（FastAPI 脚手架，约 100 行）
# 定位：给 agent 练手的"假后台"——数据在内存、重启即重置，只是验证 agent 用的临时目标，
#       真实 Go 后台落地在 P5·15 / P7。四接口：登录(JWT) / 分页查用户 / 查角色 / 分配角色。
# 依赖+启动：pip install fastapi uvicorn pyjwt；cd agent && python -m uvicorn mock_backend.main:app --port 8000
# 诚实声明：本文件已在本机实测（起服务 + 四接口 curl 全过，输出见文章实践节）。
import time

import jwt
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel

app = FastAPI(title="最小目标服务（教学脚手架）")

# ---------- 内存数据：重启即重置（教学友好）。示例数据全部用通用名 ----------

ROLES = {  # 系统里的三个角色
    "r-admin": {"role_id": "r-admin", "role_name": "管理员"},
    "r-editor": {"role_id": "r-editor", "role_name": "编辑者"},
    "r-viewer": {"role_id": "r-viewer", "role_name": "查看者"},
}
USERS = {  # 三个用户；张三初始是查看者——给 M2 的"提升为管理员"留出可核对的变化
    "u-1001": {"user_id": "u-1001", "username": "张三", "status": "active", "role_ids": ["r-viewer"]},
    "u-1002": {"user_id": "u-1002", "username": "李四", "status": "active", "role_ids": ["r-viewer"]},
    "u-1003": {"user_id": "u-1003", "username": "王五", "status": "disabled", "role_ids": []},
}
ACCOUNTS = {"admin": "admin123"}  # 登录账号（演示专用；生产严禁明文写死，见 P7·20 密钥管理）

SECRET_KEY = "teaching-secret"  # 签名密钥（教学写死；生产必须走环境变量/密钥管理，P7·20）


class LoginBody(BaseModel):
    username: str
    password: str


class AssignRoleBody(BaseModel):
    role_id: str


def sign_token(username: str) -> str:
    """签发 JWT：RFC 7519 的三段式 header.payload.signature，HS256 签名，有效期 1 小时。"""
    return jwt.encode({"sub": username, "exp": int(time.time()) + 3600},
                      SECRET_KEY, algorithm="HS256")


def check_auth(authorization: str) -> None:
    """校验 Authorization 头，不合格一律 401。agent 侧把 401 文本喂回模型；
    token 过期自动重登是第 9 篇的主题，本篇只把错误如实报出去。"""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="缺少访问令牌：请先调 POST /api/v1/auth/login 登录")
    try:
        jwt.decode(authorization[7:], SECRET_KEY, algorithms=["HS256"])  # PyJWT 自动校验 exp
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="令牌已过期，请重新登录")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="令牌无效，请重新登录")


@app.post("/api/v1/auth/login")
def login(body: LoginBody):
    """登录：用户名+密码换 JWT。响应带 assignable_roles——agent 分配角色时要从这里拿 role_id。"""
    if ACCOUNTS.get(body.username) != body.password:
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    return {"access_token": sign_token(body.username), "token_type": "Bearer",
            "expires_in": 3600, "assignable_roles": list(ROLES.values())}


@app.get("/api/v1/users")
def list_users(page: int = 1, page_size: int = 20, keyword: str = "",
               authorization: str = Header(default="")):
    """分页查用户；keyword 按用户名模糊过滤——agent 找"张三"的 user_id 就靠它。"""
    check_auth(authorization)
    hit = [u for u in USERS.values() if keyword in u["username"]]
    start = (page - 1) * page_size
    return {"total": len(hit), "page": page, "page_size": page_size,
            "users": hit[start:start + page_size]}


@app.get("/api/v1/users/{user_id}/roles")
def get_user_roles(user_id: str, authorization: str = Header(default="")):
    check_auth(authorization)
    user = USERS.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail=f"用户不存在：{user_id}")
    return {"user_id": user_id, "username": user["username"],
            "roles": [ROLES[r] for r in user["role_ids"]]}


@app.post("/api/v1/users/{user_id}/roles")
def assign_role(user_id: str, body: AssignRoleBody, authorization: str = Header(default="")):
    """分配角色：覆盖式——把该用户角色设置为 role_id 这一个。本系统唯一的写接口。"""
    check_auth(authorization)
    user, role = USERS.get(user_id), ROLES.get(body.role_id)
    if user is None:
        raise HTTPException(status_code=404, detail=f"用户不存在：{user_id}")
    if role is None:
        raise HTTPException(status_code=404, detail=f"角色不存在：{body.role_id}")
    user["role_ids"] = [body.role_id]
    return {"user_id": user_id, "username": user["username"],
            "roles": [ROLES[r] for r in user["role_ids"]]}
```

代码只有 4 个概念来自 FastAPI：`@app.get/@app.post` 路由装饰器、`BaseModel` 请求体、`Header` 读请求头（参数名按请求头名匹配、大小写不敏感，`authorization` 即 `Authorization` 头；含下划线的参数名会自动转连字符，如 `user_agent`→`user-agent`——官方 header-params 文档的约定）、`HTTPException` 抛错误响应。PyJWT 只用两个函数：`jwt.encode(payload, key, algorithm="HS256")` 签发、`jwt.decode(token, key, algorithms=[...])` 校验——`exp` 过期时间是它**自动**校验的，过期抛 `ExpiredSignatureError`（官方 usage 文档原文，签名逐字段核对过）。

### 1.4 不花 token 的接口自测

起服务后用 curl 把四个接口打一遍（零模型调用，本节输出全部为实测原文）：

```bash
# 密码错误：401 + 可读的 detail（这行文本马上会被 agent 当提示读）
curl -s -X POST http://127.0.0.1:8000/api/v1/auth/login \
  -H "Content-Type: application/json" -d '{"username":"admin","password":"wrong"}'
# {"detail":"用户名或密码错误"}

# 登录成功
curl -s -X POST http://127.0.0.1:8000/api/v1/auth/login \
  -H "Content-Type: application/json" -d '{"username":"admin","password":"admin123"}'
```

登录成功响应（实测；`access_token` 为本机实测值，每次运行都不同）：

```json
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsImV4cCI6MTc5MjA3Mjc2Nn0.ALlhHU5yW5LjKeQL2bwGFfasUFN-cCKEWOfR52LMNEY","token_type":"Bearer","expires_in":3600,"assignable_roles":[{"role_id":"r-admin","role_name":"管理员"},{"role_id":"r-editor","role_name":"编辑者"},{"role_id":"r-viewer","role_name":"查看者"}]}
```

顺手看看 JWT 长什么样（RFC 7519 的三段式 `header.payload.signature`，前两段是 Base64URL 编码的 JSON，可直接解开；第三段是 HS256 签名，改任何一个字符都会校验失败）：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9        → {"alg":"HS256","typ":"JWT"}            ← header
.eyJzdWIiOiJhZG1pbiIsImV4cCI6MTc5MjA3Mjc2Nn0 → {"sub":"admin","exp":1792072766}      ← payload
.ALlhHU5yW5LjKeQL2bwGFfasUFN-cCKEWOfR52LMNEY                                          ← signature
```

`exp` 是 1 小时后的时间戳——第 9 篇"token 过期自动重登"讲的就是它到期之后的事。

受保护接口的行为（实测；`$TOK` 换成上一步拿到的 access_token，`%E5%BC%A0%E4%B8%89` 是"张三"的 URL 编码）：

```bash
# 不带令牌：401，detail 告诉你该做什么
curl -s http://127.0.0.1:8000/api/v1/users
# {"detail":"缺少访问令牌：请先调 POST /api/v1/auth/login 登录"}

# 带令牌搜人
curl -s "http://127.0.0.1:8000/api/v1/users?keyword=%E5%BC%A0%E4%B8%89" -H "Authorization: Bearer $TOK"
# {"total":1,"page":1,"page_size":20,"users":[{"user_id":"u-1001","username":"张三","status":"active","role_ids":["r-viewer"]}]}

# 查角色 → 分配 → 复查
curl -s http://127.0.0.1:8000/api/v1/users/u-1001/roles -H "Authorization: Bearer $TOK"
# {"user_id":"u-1001","username":"张三","roles":[{"role_id":"r-viewer","role_name":"查看者"}]}

curl -s -X POST http://127.0.0.1:8000/api/v1/users/u-1001/roles \
  -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" -d '{"role_id":"r-admin"}'
# {"user_id":"u-1001","username":"张三","roles":[{"role_id":"r-admin","role_name":"管理员"}]}

# 给不存在的角色赋权：404
curl -s -w " [%{http_code}]" -X POST http://127.0.0.1:8000/api/v1/users/u-1001/roles \
  -H "Authorization: Bearer $TOK" -H "Content-Type: application/json" -d '{"role_id":"r-god"}'
# {"detail":"角色不存在：r-god"} [404]
```

接口层验证完毕。接下来是本篇的主菜：agent 这边怎么把它变成模型能用的工具。

## 2｜工具设计模式：REST 接口怎么变成模型能用的工具

读完本节你能理解：从本地函数到远程 HTTP 变了什么、认证的四个设计决定为什么这么定，以及一张可以直接抄走的工具设计模式表。

### 2.1 变了什么，没变什么

第 7 篇的工具是纯本地函数（`get_time` 读时钟、`calculate` 算四则）；本篇的工具函数体变成了一次 HTTP 请求的封装。变化清单：

| | 第 7 篇演示工具 | 第 8 篇后台工具 |
|---|---|---|
| 函数体 | 纯计算 / 本地读数 | 拼 URL + 发请求 + 解析响应 |
| 失败来源 | Python 异常（除零等） | HTTP 状态码 + 错误响应体（401/404/…） |
| 状态 | 无状态 | 有认证态（token 持有与注入） |
| 对 loop 的呈现 | 名字 + schema + 返回 dict 的函数 | **完全相同** |

最后一行是关键：**对 agent loop 而言工具的形状没变**——还是"名字 + JSON Schema + 一个返回 dict 的普通函数"。后端类比：这层工具函数就是你的 Go 后台里的 client 层——把远端 API 包成方法签名，调用方（loop）不感知 HTTP 细节。循环眼里没有"远程"，只有函数。

### 2.2 认证设计的四个决定

后台有登录态，工具就得管认证。四个决定，每个都有"为什么不那样"：

**决定一：登录本身就是一个工具**，与 get_users 平起平坐进 schema 清单，而不是在循环外由框架预先登录。理由：让模型自己管理"先登录"，登录失败它会读到错误文本、重试或向你确认账号（错误回喂自愈，第 7 篇 §2.4）；token 过期它下轮重登（第 9 篇的自动重登就是这个机制的完善）；整个认证过程都躺在 loop 日志里可观察。框架预登录则相反：出错时 agent 一头雾水，你还得另开一条通道把"登录挂了"喂给它——多一套机制，少一份可观察。

**决定二：token 只活在进程内存里**——模块级变量 `_token`，不进 prompt、不进日志。为什么不给模型？三个理由：它**不需要**（工具自动注入，见决定三）；它会**泄露**（模型拿到令牌后可能复述进回复、进终端、进对话记录）；它会**过时**（token 是易变状态，让模型搬运等于把状态同步责任推给最不可靠的一环）。这与第 6 篇 §2.2 的 key 纪律同源：凭证只放在代码够得着、模型看不见的地方。

**决定三：其余工具自动注入 Authorization 头**。四个工具共用一个 `_call()` 统一出口：进程里有 token 就带上 `Bearer` 头。业务工具函数体内没有一行认证代码——`get_users` 压根不知道 Authorization 的存在。关注点分离：认证是水管的事，不是每个水龙头的事。

**决定四：工具描述里写清"必须先登录"**。工具描述就是 prompt（第 7 篇 §1.4——模型看到的只有 schema 文本），本篇把这条认识用在行为编排上：

- login 的描述：**"在调用任何其他后台工具之前必须先成功登录一次"** + 附上演示账号（用户名 admin / 密码 admin123）——模型自己读账号自己登录，用户一句话里不需要出现"登录"两个字；
- assign_role 的描述：写明"覆盖式：该用户原有的角色会被整体替换"、role_id 从 login 返回的 `assignable_roles` 里选、**"执行前先调 get_user_roles 确认现状，并向用户复述一遍将发生的变更"**——"读现状→复述→再写"的行为链直接由描述教出来。

一句话总结：**描述即流程**。你期望模型执行的顺序（登录→搜人→查现状→分配），就写成描述里的顺序。顺带说明一个边界：演示账号写死在描述里是教学取舍（agent 才能自给自足）；生产的凭证来自配置/密钥管理，不会进任何 prompt——P7·20 密钥管理篇展开。

还有一个细节藏在 login 的实现里，值得单独拎出来——**"token 不进日志"是怎么做到的**。agent.py 会把每个工具结果打印到终端（`[工具→]` 行），如果 login 把 access_token 原样返回，令牌就上了终端。所以 login 在返回前把它从结果里摘走：

```python
if "access_token" in result:             # 成功：摘走 token 再返回（不进 prompt / 不进日志）
    _token = result.pop("access_token")
    result["status"] = "ok"
return result
```

模型和终端看到的是 `{"token_type": ..., "expires_in": 3600, "assignable_roles": [...], "status": "ok"}`——"登录成功"这个**事实**回喂给模型，令牌**本体**留在进程内存。这是本篇最值得抄进你自己项目的一个细节。

### 2.3 工具设计模式表

把四个工具的设计决策收进一张表（风险分级为第 9 篇的确认闸门做铺垫）：

| 工具 | 描述要点（写给模型看） | 认证处理 | 风险级 |
|---|---|---|---|
| `login` | "调任何其他后台工具前必须先登录"；附演示账号；令牌自动保管、不必向用户提及 | 自己调登录接口；token 存进程内存并**从返回值摘除** | 低（演示凭证；生产凭证管理见 P7·20） |
| `get_users` | 返回 users 含 user_id；keyword 按用户名模糊搜人 | `_call` 自动注入 Bearer | 低（只读） |
| `get_user_roles` | user_id 从 get_users 来；"改角色前先看一眼现状" | 同上 | 低（只读） |
| `assign_role` | 覆盖式语义写明；role_id 从 login 的 assignable_roles 选；执行前先查现状+复述变更 | 同上 | **中高（写操作+覆盖语义；第 9 篇上二次确认闸门）** |

### 2.4 统一 HTTP 出口：把错误变成信息

`_call()` 是四个工具共用的一根水管（完整代码见 §2.5），它要兜三类失败：

**一、4xx/5xx。** urllib 把非 2xx 响应当异常抛（`urllib.error.HTTPError`），但异常对象里带着完整响应体——读出来、拆掉 FastAPI 的 `{"detail": ...}` 信封、装成结构化 dict 返回：

```python
except urllib.error.HTTPError as e:      # 4xx/5xx：urllib 把非 2xx 当异常，正文里有 detail
    raw = e.read().decode("utf-8", "replace")
    try:                                 # FastAPI 的错误信封是 {"detail": "..."}——拆出人话再回喂
        detail = json.loads(raw).get("detail", raw)
    except json.JSONDecodeError:
        detail = raw
    return {"error": True, "status": e.code, "detail": detail}
```

返回形态是 `{"error": true, "status": 401, "detail": "用户名或密码错误"}`。为什么装成结构化 dict 而不是一段文字？状态码是程序性信息：模型判断"该重试（如 401 过期→重登）还是该改参数（如 404→换 ID）"，主要看 status 与 detail 的组合。这与第 7 篇 §2.4 的原则同源——**错误不抛出，回喂给模型**；只是错误源从 Python 异常换成了 HTTP 层。

**二、连接层失败**（后台没起、端口不对）：`urllib.error.URLError`，装成一句可操作的人话：`"无法连接后台（…）——确认 mock_backend 已启动"`。实践节有个选做实验专门复现它。

**三、环境坑：系统代理劫持本地回环。** 如果你的 shell 设过 `http_proxy`，urllib 默认会把 `http://127.0.0.1` 的请求也交给代理——本地回环地址必须直连。一行修复：`_opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))`，之后用 `_opener.open(...)` 代替 `urllib.request.urlopen(...)`。这是实测踩过的坑（代理开着时表现为"连接失败"，一关就好）。

至于 401 里的"令牌已过期"——本篇只把错误如实报出去，**token 过期自动重登是第 9 篇的第一课**。这里留个钩子：重登需要的所有零件（登录工具、凭证来源、错误文本）本篇已经备齐，第 9 篇只需教 agent"看到 401 → 自动再 login 一次"。

### 2.5 完整代码：admin_tools.py

```python
# admin_tools.py — 第 8 篇：后台管理工具集（login / get_users / get_user_roles / assign_role）
# 与第 7 篇 tools.py 的本质区别：工具不再是本地函数，而是对远程 REST 接口的 HTTP 调用——
# agent 从此"通过 HTTP 看后端"，后端是什么语言写的它一无所知也不必知道（本篇 §2.1）。
# 与 tools.py 暴露同一组注册件接口（TOOL_SCHEMAS / TOOL_REGISTRY / SCHEMA_BY_NAME /
# validate_args），所以 agent.py 的 loop 代码一行核心逻辑都不用改（本篇 §3.1）。
# 诚实声明：本模块已在本机对 mock_backend 实测跑通 M2 闭环与错误路径（见文章实践节）。
import json
import urllib.error
import urllib.parse
import urllib.request

import tools  # 复用第 7 篇的最小 schema 校验：工具换了，参数闸门不换

BASE_URL = "http://127.0.0.1:8000"  # 本地脚手架基址；接真实后台时只改这一行（P5·15 / P7）
TIMEOUT = 10.0  # 工具自己的超时，与 LLM 调用的 60 秒（config.TIMEOUT）各管各的

# 不走系统代理：若 shell 里设了 http_proxy，urllib 会把 127.0.0.1 的请求也交给代理——本地
# 回环地址必须直连。这是实测踩过的坑（代理开着时表现为"连接失败"，关代理或用本 opener 即好）。
_opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))

# ---------- 认证状态：token 只活在进程内存里 ----------
# 设计决定（本篇 §2.2）：token 不进 messages（模型不需要看到它）、不进日志（终端会打印
# 工具结果，所以 login 工具在返回前把 token 从结果里摘走）；进程重启即失效，重登一次就好。
_token: str | None = None


# ---------- 统一 HTTP 出口：四个工具共用的一根水管 ----------
def _call(method: str, path: str, body: dict | None = None) -> dict:
    """拼 URL、发请求、自动注入 Authorization；任何失败都不抛出——
    把状态码+错误文本装进 dict 返回，由 loop 当作普通工具结果回喂模型（第 7 篇 §2.4 同一哲学）。"""
    headers = {"Content-Type": "application/json"}
    if _token:  # 已登录则自动带令牌——其余工具从此"感觉不到"认证的存在
        headers["Authorization"] = f"Bearer {_token}"
    data = json.dumps(body, ensure_ascii=False).encode("utf-8") if body is not None else None
    req = urllib.request.Request(BASE_URL + path, data=data, headers=headers, method=method)
    try:
        with _opener.open(req, timeout=TIMEOUT) as resp:
            return json.loads(resp.read().decode("utf-8"))
    except urllib.error.HTTPError as e:      # 4xx/5xx：urllib 把非 2xx 当异常，正文里有 detail
        raw = e.read().decode("utf-8", "replace")
        try:                                 # FastAPI 的错误信封是 {"detail": "..."}——拆出人话再回喂
            detail = json.loads(raw).get("detail", raw)
        except json.JSONDecodeError:
            detail = raw
        return {"error": True, "status": e.code, "detail": detail}
    except urllib.error.URLError as e:       # 连接层失败：典型是后台服务没起
        return {"error": True, "status": None,
                "detail": f"无法连接后台（{e.reason}）——确认 mock_backend 已启动"}


# ---------- 四个工具：普通 Python 函数，返回 dict（会经 json.dumps 变成 tool 结果） ----------

def login(username: str, password: str) -> dict:
    """登录工具：成功后把 token 存进进程内存；返回给模型/终端的只有"登录成功"的元信息。"""
    global _token
    result = _call("POST", "/api/v1/auth/login", {"username": username, "password": password})
    if "access_token" in result:             # 成功：摘走 token 再返回（不进 prompt / 不进日志）
        _token = result.pop("access_token")
        result["status"] = "ok"
    return result


def get_users(page: int = 1, page_size: int = 20, keyword: str = "") -> dict:
    qs = urllib.parse.urlencode({"page": page, "page_size": page_size, "keyword": keyword})
    return _call("GET", f"/api/v1/users?{qs}")


def get_user_roles(user_id: str) -> dict:
    return _call("GET", f"/api/v1/users/{user_id}/roles")


def assign_role(user_id: str, role_id: str) -> dict:
    return _call("POST", f"/api/v1/users/{user_id}/roles", {"role_id": role_id})


# ---------- 工具清单：随请求发给模型的 JSON Schema（与第 7 篇同一 wire 格式） ----------
# 描述就是 prompt（第 7 篇 §1.4）：谁先谁后、参数从哪来、哪个是写操作，全写在描述里——
# 模型照着描述自己排步骤，用户一句话里不需要出现"登录"两个字。
TOOL_SCHEMAS = [
    {
        "type": "function",
        "function": {
            "name": "login",
            "description": (
                "登录后台管理系统，获取访问令牌。在调用任何其他后台工具之前必须先成功登录一次。"
                "演示账号：用户名 admin，密码 admin123。登录成功后令牌由系统自动保管并在后续"
                "调用中自动携带，你无需向用户提及令牌，也不要询问密码以外的任何凭证信息。"
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "username": {"type": "string", "description": "登录用户名"},
                    "password": {"type": "string", "description": "登录密码"},
                },
                "required": ["username", "password"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_users",
            "description": (
                "分页查询后台用户列表，返回 users（每人含 user_id、username、status）与 total。"
                "keyword 按用户名模糊搜索——不知道 user_id 时，先用它把人搜出来。"
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "page": {"type": "integer", "description": "页码，从 1 开始，默认 1"},
                    "page_size": {"type": "integer", "description": "每页条数，默认 20"},
                    "keyword": {"type": "string", "description": "按用户名模糊过滤，可为空串"},
                },
                "required": [],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_user_roles",
            "description": (
                "查询指定用户当前拥有的角色列表。user_id 从 get_users 的结果里来。"
                "在给用户改角色之前，先调它看一眼现状。"
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "user_id": {"type": "string", "description": "用户 ID，来自 get_users"},
                },
                "required": ["user_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "assign_role",
            "description": (
                "把指定用户的角色设置为 role_id 对应的角色。这是写操作，且是覆盖式：该用户"
                "原有的角色会被整体替换，role_id 从 login 返回的 assignable_roles 里选；"
                "执行前先调 get_user_roles 确认现状，并向用户复述一遍将发生的变更。"
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "user_id": {"type": "string", "description": "用户 ID，来自 get_users"},
                    "role_id": {"type": "string", "description": "角色 ID，来自 login 返回的 assignable_roles"},
                },
                "required": ["user_id", "role_id"],
            },
        },
    },
]

TOOL_REGISTRY = {"login": login, "get_users": get_users,
                 "get_user_roles": get_user_roles, "assign_role": assign_role}

SCHEMA_BY_NAME = {s["function"]["name"]: s["function"] for s in TOOL_SCHEMAS}

validate_args = tools.validate_args  # 校验闸门原样复用（第 7 篇 §3.1），接口同形
```

## 3｜loop 的复用：换工具集，不换循环

读完本节你能理解：agent.py 这次改了哪几处、为什么四条错误兜底路径一行没改就能继续工作。

### 3.1 改动清单与完整代码

第 7 篇的循环结构原封不动，改动全部在"外围配置"：

| 改动 | 内容 |
|---|---|
| 导入与选择 | `import admin_tools`；模块级 `toolset = admin_tools`（主线切换到后台工具集） |
| 提示词 | 新增后台助手的 SYSTEM_PROMPT（含"先 login""写操作先复述"两条行为规则）；原提示词保留为 DEMO_SYSTEM_PROMPT |
| 预置问题 | 换成 3 条后台问题（--demo 时换回原 4 条） |
| 命令行 | 新增 `--demo` 旗标：工具集、系统提示词、预置问题三者整组切回演示工具集 |

为什么循环能零改动？因为两个工具集模块暴露**同一组注册件接口**（`TOOL_SCHEMAS` / `TOOL_REGISTRY` / `SCHEMA_BY_NAME` / `validate_args`）——第 3 篇 §2.3 说"工具注册件是形状"，现在是这句话兑现的时刻：换的是注册表的内容，不是循环的结构。

`requirements.txt` 也更新了（完整内容如下）：新增 `fastapi==0.143.1` / `uvicorn==0.54.0` / `pyjwt==2.15.1`（2026-10-15 PyPI 核验的现行版，只服务脚手架）——**agent 侧零新增依赖**，HTTP 客户端用的是第 6 篇教过的标准库 urllib。为什么选 urllib 不选 requests？零新依赖、与第 6 篇姿势一的写法一脉相承、拿到的是未经包装的响应体；requests 的便利（自动重定向、Session）在这个 4 工具的教学规模里用不上。若你的项目已在用 requests，把 `_call` 换成 `requests.request(...)` 约十行，两类兜底逻辑不变。

```text
# agent/ 项目依赖（Python 3.11+，系列约定见第 5 篇）
openai==3.31.0
# 第 8 篇新增：只服务于 mock_backend 脚手架（agent 侧工具走标准库 urllib，零新增依赖）。
# 版本为 2026-10-15 PyPI 核验的现行版。
fastapi==0.143.1
uvicorn==0.54.0
pyjwt==2.15.1
```

```python
# agent.py — 手写最小 agent（第 7 篇 M1 建立；第 8 篇起主线切后台管理工具集，M2）
# 依赖：pip install -r requirements.txt（openai 3.31.0；Python 3.11+，第 5 篇）
# 运行：
#   终端 1：python -m uvicorn mock_backend.main:app --port 8000   # 先起后台脚手架（第 8 篇）
#   终端 2：
#   export DEEPSEEK_API_KEY=sk-xxx && python agent.py "把张三提升为管理员"   # M2 主线路径
#   python agent.py                            # 跑后台预置问题集
#   python agent.py --demo                     # 切回第 7 篇演示工具集（M1 复验）
#   python agent.py --chat                     # 交互模式
#   python agent.py --think                    # 思考档（第 4 篇 §1.5）
# 诚实声明：loop 逻辑（含四条错误兜底路径与熔断）已用桩模型本地实测；
#           M2 闭环已对本地 mock_backend 实测（桩 LLM + 真实 HTTP，见第 8 篇实践节）；
#           对真实 DeepSeek 的调用 <未实测，以官方文档为准>
import argparse
import json

from openai import OpenAI

import admin_tools
import config
import tools as demo_tools   # 演示工具集：--demo 复验 M1 时使用

# 第 8 篇主线切换：工具集从演示工具（tools.py）换成后台管理工具（admin_tools.py）。
# 两个模块暴露同一组注册件接口（TOOL_SCHEMAS / TOOL_REGISTRY / SCHEMA_BY_NAME /
# validate_args），所以下面的 loop 代码一行核心逻辑都不用改——这就是第 3 篇 §2.3
# 说"工具注册件是形状"的红利：换的是注册表内容，不是循环结构。
toolset = admin_tools

# 系统提示词随工具集绑定切换（main 里 --demo 时三者一起换）——它们与工具描述同属
# "给模型看的说明书"（第 7 篇 §1.4），改工具不改说明，模型立刻按旧世界观干活。
# 第 4 篇 §2.2 的"数值原样引用"护栏升级成"ID/角色名原样引用"；第 4 条让 §2.4 的
# 错误自愈通道与写操作前的复述成为行为预期（真正的确认闸门在第 9 篇实现）。
SYSTEM_PROMPT = (
    "你是一个后台管理助手，通过工具操作用户管理系统。规则：\n"
    "1. 涉及用户、角色的事实必须调用工具获取，禁止编造 user_id / role_id；\n"
    "2. 调用其他后台工具前必须先 login（演示账号写在工具说明里）；令牌由系统自动保管，"
    "不必向用户提及，也不要在回复里复述任何令牌；\n"
    "3. 回答里引用的 ID、角色名必须原样来自工具结果，不要改写或补全；\n"
    "4. 工具返回错误时读懂错误、调整后重试；assign_role 是写操作，执行前先把要做的事"
    "向用户复述一遍。"
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
        # 工具已是远程 HTTP（第 8 篇），教学版仍顺序执行——同消息多工具并行发送属于
        # 时延优化项，本系列不做（M2 流程每轮通常只来一个调用）。
        for call in msg.tool_calls:
            print(f"[工具] {call.function.name}({call.function.arguments})")
            result = execute_tool(call)
            print(f"[工具→] {result}")
            messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
    # 终止条件二：最大迭代数熔断——熔断后必须有明确行为，而不是静默返回空（第 3 篇 §4.2）
    raise RuntimeError(f"[熔断] 达到最大迭代数 {config.MAX_ITERATIONS}，强制终止")


def run_safely(client, question: str, think: bool = False) -> str:
    """熔断等 RuntimeError 转成可见的答复文本——明确报错，不静默、不崩终端。"""
    try:
        return run_agent(client, question, think=think)
    except RuntimeError as e:
        return str(e)


def main():
    global toolset, SYSTEM_PROMPT, PRESET_QUESTIONS   # --demo 时在函数内给模块级变量重新赋值
    parser = argparse.ArgumentParser(description="手写最小 agent（第 7 篇 M1 → 第 8 篇 M2）")
    parser.add_argument("question", nargs="?", help="单次提问；不给则跑预置问题集")
    parser.add_argument("--chat", action="store_true", help="交互模式；每行输入一次全新会话")
    parser.add_argument("--think", action="store_true", help="开思考档（第 4 篇 §1.5）")
    parser.add_argument("--demo", action="store_true",
                        help="切回第 7 篇演示工具集 get_time/calculate（M1 复验用）")
    args = parser.parse_args()
    if args.demo:   # 工具集、系统提示词、预置问题三者整组切换
        toolset, SYSTEM_PROMPT, PRESET_QUESTIONS = demo_tools, DEMO_SYSTEM_PROMPT, DEMO_PRESET_QUESTIONS
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
        else:                                # 预置问题集：M2 验收路径（--demo 时为 M1 复验）
            for q in PRESET_QUESTIONS:
                print(f"[助手] {run_safely(client, q, think=args.think)}")
    except KeyboardInterrupt:
        print("\n[退出] 用户中断（第 3 篇终止条件之四：用户确认/打断）")


if __name__ == "__main__":
    main()
```

### 3.2 四条兜底路径在新场景下的表现

工具从本地换到远程，第 7 篇的四条防护一条没动，逐条对账：

| 坑（第 7 篇 §3.2） | 本篇场景下的表现 | 防护位置 |
|---|---|---|
| 幻觉调用 | 模型调了不存在的工具 → 注册表查不到 → 错误文本回喂 | `execute_tool`（未改） |
| 坏 JSON/空参 | 同左，与工具远近无关 | `execute_tool`（未改） |
| 参数不合规 | schema 校验原样复用（`admin_tools.validate_args = tools.validate_args`）；实测里它真拦下过一次 `user_id: null` 的坏参数 | `validate_args`（复用） |
| 运行时异常 | HTTP 层的失败在 `_call` 内就被装成 error dict 返回，根本到不了 `execute_tool` 的 except——两层防线 | `_call`（新）+ `execute_tool`（未改） |

熔断依然在：M2 主路径 5 轮收尾，`MAX_ITERATIONS=10` 余量充足。系统提示词的规则 4（"assign_role 是写操作，执行前先复述"）是给确认闸门铺的行为预期——真正的闸门（代码级强制，不靠模型自觉）在第 9 篇。

## 动手实践：里程碑 M2——一句话完成「查用户、分配角色」

目标：用户只说一句话「把张三提升为管理员」，agent 自动完成 登录→查用户→查角色→分配→答复，**全程无需人工中途接手，结果可核对**。做完你会亲眼验证本篇的四个设计决策：描述即流程（没提"登录"它自己登录了）、token 不出进程内存、错误即信息（401/404 回喂不崩）、loop 零改动。

诚实声明：本实践的 `[loop]`/`[工具]`/`[工具→]` 日志行已用"**桩 LLM + 真实 mock_backend（真实 HTTP）**"实测并逐字记录（方法同第 7 篇：只把模型响应换成照剧本出牌的桩，loop、工具层、HTTP 全部走真实代码）；`[模型]`/`[助手]` 的自然语言文本随模型浮动，为结构示意——**对真实 DeepSeek 的调用 `<未实测，以官方文档为准>`**。判断验收看**结构**（工具序列、轮数、结果字段），不看措辞。

### 第 0 步：准备

```bash
cd agent
# 环境沿用第 5/7 篇的 venv（Python 3.11+）；没有就重建
python3.11 -m venv .venv && source .venv/bin/activate
# requirements.txt 新增 fastapi/uvicorn/pyjwt（只服务脚手架；agent 侧零新增依赖）
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
export DEEPSEEK_API_KEY=sk-xxxxxxxx
```

预算：脚手架自测（curl）**零 token**；agent 闭环一次约 5 轮调用、4 个工具，量级与第 7 篇单条预置问题相当（约 0.01-0.05 元）；选做错误路径每次再花一两分钱。

### 第 1 步：终端 1——起脚手架并自测接口（不花 token）

```bash
python -m uvicorn mock_backend.main:app --port 8000
```

预期输出（实测）：

```text
INFO:     Started server process [56990]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

另开一个终端做接口自测——§1.4 的 curl 全套打一遍（零模型调用），确认四个接口都是预期行为再上 agent。

### 第 2 步：终端 2——一句话跑闭环（M2 验收路径）

```bash
python agent.py "把张三提升为管理员"
```

预期输出（实测转写；诚实声明见节首）：

```text
[用户] 把张三提升为管理员
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] login({"username": "admin", "password": "admin123"})
[工具→] {"token_type": "Bearer", "expires_in": 3600, "assignable_roles": [{"role_id": "r-admin", "role_name": "管理员"}, {"role_id": "r-editor", "role_name": "编辑者"}, {"role_id": "r-viewer", "role_name": "查看者"}], "status": "ok"}
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_users({"keyword": "张三"})
[工具→] {"total": 1, "page": 1, "page_size": 20, "users": [{"user_id": "u-1001", "username": "张三", "status": "active", "role_ids": ["r-viewer"]}]}
[loop] 第 3 轮 | 发送 6 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] get_user_roles({"user_id": "u-1001"})
[工具→] {"user_id": "u-1001", "username": "张三", "roles": [{"role_id": "r-viewer", "role_name": "查看者"}]}
[loop] 第 4 轮 | 发送 8 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] assign_role({"role_id": "r-admin", "user_id": "u-1001"})
[工具→] {"user_id": "u-1001", "username": "张三", "roles": [{"role_id": "r-admin", "role_name": "管理员"}]}
[loop] 第 5 轮 | 发送 10 条消息
[loop] finish_reason=stop | tool_calls=0 个

[模型] 已完成：张三（u-1001）的角色已从 查看者 变更为 管理员。
[loop] 终止：共 5 轮

[助手] 已完成：张三（u-1001）的角色已从 查看者 变更为 管理员。
```

### 看懂输出：五轮逐轮拆 + 闭环时序图

1. **第 1 轮 login**：用户消息里没有"登录"两个字，第一个工具却是 login——描述教的（§2.2 决定四）。`[工具→]` 的结果里**没有 access_token**：token 被摘进进程内存（§2.2 的细节），但 `assignable_roles` 进了上下文——第 4 轮能填对 role_id 全靠它。
2. **第 2 轮搜人**：keyword=张三 是模型自己想到的（描述写了"不知道 user_id 时先用它搜人"），user_id=u-1001 从这一步的结果里来。
3. **第 3 轮查现状**：不是多余动作——"改角色前先看一眼现状"写在 assign_role 的描述里（读-改-写），也让第 5 轮的答复能说出"**从 查看者** 变更为 管理员"。
4. **第 4 轮写操作**：覆盖式分配后返回新角色清单，闭环的"动作"落地。
5. **第 5 轮收尾**：无 tool_calls + finish_reason=stop——loop 的结构化终止判定与工具远近无关（第 7 篇 §2.3 原样生效）。

时序全景（agent 的 5 轮与后台的 4 次 HTTP 交互）：

```text
用户         agent loop（终端 2）                        mock_backend（终端 1）
 │                │                                          │
 │ "把张三提升为    │ 第 1 轮：调 LLM（随请求发 4 个工具 schema）  │
 │  管理员"        │ ───────────> DeepSeek                     │
 ├───────────────>│ <───────────── login(admin/admin123)      │
 │                │   execute_tool → POST /api/v1/auth/login  │
 │                │ ─────────────────────────────────────────>│ 校验账号、签发 JWT
 │                │ <───────────────── {access_token, roles} ─│
 │                │   token 存进程内存；结果摘掉 token 再回填     │
 │                │ 第 2 轮：调 LLM（带上 login 结果）            │
 │                │ <───────────── get_users(keyword=张三)     │
 │                │   GET /api/v1/users?keyword=…（自动带 Bearer）│
 │                │ ─────────────────────────────────────────>│ 分页 + 模糊过滤
 │                │ <──────────── {users:[{user_id:u-1001}]} ─│
 │                │ 第 3 轮：… get_user_roles(u-1001)          │
 │                │ ─────────────────────────────────────────>│
 │                │ <──────────────────────── {roles:[查看者]} ─│
 │                │ 第 4 轮：… assign_role(u-1001, r-admin)     │
 │                │ ─────────────────────────────────────────>│ 覆盖式写入
 │                │ <──────────────────────── {roles:[管理员]} ─│
 │                │ 第 5 轮：调 LLM → 无 tool_calls，stop       │
 │ "已完成：张三…   │                                           │
 │  管理员"        │                                           │
 │<───────────────┤                                           │
```

### 第 3 步：自检清单（M2 验收标准）

前三条是 M2 的验收线（一句话触发全链路、无需人工接手、结果可核对），全部通过即里程碑达成：

- [ ] 一条命令 `python agent.py "把张三提升为管理员"` 跑完，中途不需要你输入任何额外信息（**无需人工接手**）；
- [ ] `[工具]` 行出现 4 次，顺序覆盖 login → get_users → get_user_roles → assign_role（**一句话触发全链路**）；
- [ ] `[助手]` 答复与第 4 轮 `[工具→]` 的角色一致；在数据重置前再问一句"张三现在是什么角色？"，答复为 管理员（**结果可核对**）；
- [ ] 用户消息里没有"登录"两个字，第 1 个工具是 login（描述即流程的亲证）；
- [ ] 在全部终端输出里搜不到 `eyJ` 开头的字符串（token 不进 prompt / 不进日志）；
- [ ] 每轮 `[loop]` 日志可见、`发送 N 条消息` 逐轮 +2、以 `[loop] 终止：共 5 轮` 收尾；
- [ ] （选做·401 回喂）把 `admin_tools.py` 中 login 描述里的演示密码 admin123 改成 admin999 再跑：模型会照描述用错密码，看到 `"status": 401` 的回喂与 agent 的登录失败报告，循环未崩（跑完改回来）；
- [ ] （选做·404 回喂）说一句"把 user_id 是 u-9999 的用户提升为管理员"：模型若跳过搜人直接 assign_role，会收到 `用户不存在：u-9999` 的回喂并向你报告——模型行为可能不同，看到 404 回喂即为复现；
- [ ] （选做·后台没起）Ctrl+C 停掉终端 1 再问一句：agent 报告"无法连接后台"且正常收尾（§2.4 的 URLError 人话回喂）；
- [ ] （选做·内存数据）重启脚手架（数据重置）后问"张三现在是什么角色？"——回到 查看者；
- [ ] （选做·M1 回归）`python agent.py --demo "现在几点了？"`——演示工具集与原提示词原样可用。

### 选做：错误路径的实测转写（401 回喂）

把 login 描述里的演示密码改成 admin999 后重跑（实测转写）：

```text
[用户] 把张三提升为管理员
[loop] 第 1 轮 | 发送 2 条消息
[loop] finish_reason=tool_calls | tool_calls=1 个
[工具] login({"username": "admin", "password": "bad-password"})
[工具→] {"error": true, "status": 401, "detail": "用户名或密码错误"}
[loop] 第 2 轮 | 发送 4 条消息
[loop] finish_reason=stop | tool_calls=0 个
[模型] 登录失败了：后台返回「用户名或密码错误」。请确认账号信息后我再试一次。
[loop] 终止：共 2 轮

[助手] 登录失败了：后台返回「用户名或密码错误」。请确认账号信息后我再试一次。
```

对照看三个点：`[工具→]` 是 §2.4 的结构化错误 dict（status + detail）；循环**没有**崩、也没有原样重试（第 7 篇系统提示词的"不要原样重复同样的调用"在起作用）；agent 把失败如实报告而不是编一个"已提升"——**错误回喂 + 提示词纪律**共同守住了诚实底线。404（不存在的角色/用户）与 URLError（后台没起）同走这一条通道，错误源不同、回喂结构相同。

---

到这里 Part 3 的主线正式立起来：一句话进、结果出、四个真实 REST 工具在中间自己排步骤。但也要诚实说清这份成品的粗糙面：token 一小时后过期，本篇只会把 401 如实报给你；写操作只有提示词里的"复述"约束，没有代码级的确认闸门；4xx/5xx 都混在一个 error dict 里，没有分级重试。**这些正是第 9 篇的课：token 过期自动重登、4xx/5xx 分级处理、危险写操作的二次确认。**

## 本篇结论

- 最小目标服务 = 100 行 FastAPI 脚手架（内存数据、登录/查用户/查角色/分配角色四接口），是成熟方案路线 ① 的验证靶子；agent 只通过 HTTP 看后端、与后端语言解耦——真实 Go 后台落地在 P5·15 / P7，届时理论上只改 `BASE_URL`。
- 工具设计模式：登录即工具、token 只活进程内存（login 返回前摘除——不进 prompt 不进日志）、其余工具由统一 HTTP 出口 `_call` 自动注入 Authorization、描述即流程（"必须先登录""覆盖式写操作"写在描述里，模型照着自己排步骤）。
- loop 一行核心逻辑未改（注册件同形接口），四条错误兜底原样生效；**里程碑 M2 达成**——下一站第 9 篇：token 过期自动重登、4xx/5xx 分级、写操作二次确认。

## 参考资料

1. [FastAPI 官方文档 — First Steps / Body / Header Parameters](https://fastapi.tiangolo.com/tutorial/first-steps/) — `FastAPI()`、`@app.get/@app.post` 路由装饰器、`BaseModel` 请求体、`Header` 参数声明（下划线自动转连字符）与非 Annotated 写法（`str | None = Header(default=None)`）；Body 见 https://fastapi.tiangolo.com/tutorial/body/ ，Header 见 https://fastapi.tiangolo.com/tutorial/header-params/ 。2026-10-15 实测。
2. [FastAPI Tutorial — Security First Steps](https://fastapi.tiangolo.com/tutorial/security/first-steps/) — 生产认证方向（OAuth2PasswordBearer / Security 依赖）；本篇脚手架用简化 Header 方案换取概念密度最低，二者取舍在 §1.3。
3. [PyJWT 官方文档 — Usage Examples](https://pyjwt.readthedocs.io/en/stable/usage.html) — `jwt.encode(payload, key, algorithm="HS256")` / `jwt.decode(token, key, algorithms=[...])` 官方签名、`exp` 过期自动校验并抛 `jwt.ExpiredSignatureError` 的原文示例。2026-10-15 实测。
4. [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) — 三段式 `header.payload.signature` 结构、HS256、`exp`/`sub` 等 registered claims。
5. [Python 标准库 urllib.request](https://docs.python.org/3/library/urllib.request.html) — `Request`/`urlopen`、`HTTPError` 读响应体、`ProxyHandler` 与 opener 机制（§2.4 代理坑的修复依据）。
6. [uvicorn（PyPI）](https://pypi.org/project/uvicorn/) — `uvicorn main:app` 启动格式另见 FastAPI 部署文档 https://fastapi.tiangolo.com/deployment/manually/ （`main:app` = main.py 的 app 对象）。版本 0.54.0，2026-10-15 PyPI 核验。
7. 系列前序：[第 1 篇](../part1-打开黑盒/01-全景与学习路线地图.md)（三方分工）、[第 3 篇](../part2-亲手写出第一个Agent/03-agent-loop与上下文.md)（五阶段、工具注册件、终止条件）、[第 6 篇](../part2-亲手写出第一个Agent/06-调用LLM的三种姿势.md)（urllib 裸 HTTP 写法、key 管理、超时重试）、[第 7 篇](../part2-亲手写出第一个Agent/07-手写最小agent.md)（`agent/` 项目、schema 即 prompt、四条错误兜底）。

> 核验说明：fastapi 0.143.1 / uvicorn 0.54.0 / pyjwt 2.15.1 为 2026-10-15 PyPI JSON API 核验的现行版，并实际装入 venv（Python 3.12）跑通；FastAPI 的路由/BaseModel/Header/HTTPException 用法逐页对照官方 docs 核验；PyJWT encode/decode 签名与 exp 自动校验对照官方 usage 页原文核验；`mock_backend/main.py` 起服务后四接口全部 curl 实测（本文 §1.4 输出为实测原文）；M2 闭环与错误路径（401/404/连接失败）以"桩 LLM + 真实 HTTP"实测，6 组场景 15 项断言全过，`[loop]`/`[工具]`/`[工具→]` 日志与实测逐字一致；`agent/` 全部 Python 文件通过 `python3 -m py_compile`；对真实 DeepSeek 的调用未实测并已显式标注。

