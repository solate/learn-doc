# 《从零构建 Agent》系列 生成进度

> 本文件是**跨会话唯一事实来源**。上下文压缩后只读本文件即可续接。
> 最后更新：2026-10-19

## ⚠️ 系列级技术事实（全系列 writer 必读，2026-10-15 双重核验）

- **DeepSeek 模型名已换血**：官方 News（api-docs.deepseek.com/news/news250702）与 pricing 页
  2026-10-15 两次独立核验——旧名 `deepseek-chat` / `deepseek-reasoner` 已停用（2026-07-24 起）；
  现行 `deepseek-flash`（DeepSeek-V4.1-Flash，1M 上下文）与 `deepseek-v4-pro`。
  「通用 vs 推理」不再对应两个模型名，而是**单模型 `thinking` 开关 +
  `reasoning_effort`（none/low/high/max，默认 high）**。
  带 tools 的请求须完整回传历史 reasoning_content 否则 400；思维链按输出价计费。
  考据全文见第 4 篇 §1.2。**后续所有篇的 DeepSeek 示例一律用 deepseek-flash。**
  （00-生成提示词.md 技术约束里缓存的是旧口径，以本条与官方 docs 为准。）
- **框架版本/star（2026-10-15 PyPI JSON API + GitHub API 实测）**：langchain 1.4.6
  （star 147,857）/ langgraph 1.2.15（43,305）/ openai-agents 0.24.0（30,052）/
  pydantic-ai-slim 2.56.0（20,632）/ claude-agent-sdk 0.2.167（8,283）。
  langchain 1.x 的 agent 入口是 create_agent（官方自述 built on top of LangGraph），
  旧链式 API 在 langchain-classic 包（Legacy）。
- **2026-10-19 harness 版图补充（第 5 篇修订时重新核验）**：新增 DeepSeek Harness
  253,527 star（npm 0.2.0-rc.2 / PyPI deepseek-harness-sdk 0.2.1rc2，developer
  preview）、OpenClaw 392,423、Pi 116,236、OpenHarness 16,015；库/框架：deepagents
  30,396（0.7.28）、Google ADK 21,907（2.12.0）、Microsoft Agent Framework 14,199
  （1.22.0）、Strands 8,861（strands-agents 1.61.1 / strands-harness 0.3.1）、
  smolagents 29,918（1.26.0）、CrewAI 59,853（1.15.28）；pydantic-ai-slim 2.57.0、
  langgraph 1.2.16、claude-agent-sdk 0.2.168。**第 5 篇已改为"决策篇"：首选 deepagents
  （多模型/subagents/skills 开箱即有），备选 Microsoft Agent Framework（企业治理/多语言），
  DeepSeek Harness 因 developer preview 暂列预研。** 后续篇引用选型结论以第 5 篇 §3 决策表为准。
- **全系列 Python 版本要求 3.11+**（openai SDK requires-python>=3.10、pydantic-ai-slim
  >=3.11；Python 3.9 会静默降级 openai 3.x→2.x，第 5 篇实践实测踩坑）。
- **OWASP LLM Top 10 现行版 = 2026 版**（2026-08-04 发布，仓库 GenAI-Security-Project/
  GenAI-LLM-Top10 的 2026/final；LLM01:2026 Prompt Injection 居首；Excessive Agency 升至
  第 3、System Prompt Leakage 更名 Hidden Context Exposure）。第 10 篇已按 2026 版写，
  后续 P7·20 安全篇沿用此版本口径（2026-10-16 官方 repo 逐句核验）。
- **MCP 版本双时代口径（2026-10-16 核验）**：规范 tag `2026-07-28` 仍为最新，但该版已删
  initialize 握手、走向无状态（每请求 `_meta` 信封 + `server/discover`，删 Mcp-Session-Id
  与 GET 流端点）；官方 python-sdk `mcp 2.3.0` 双时代并存（HANDSHAKE ≤2025-11-25 /
  MODERN 2026-07-28），部署现实仍以握手期为主——**后续篇按「学协议看最新版、写代码查
  当下生态」双线处理**。另：`mcp 2.x` 已把 FastMCP 更名为 `MCPServer`
  （`from mcp.server.mcpserver import MCPServer`）。详见第 13 篇 §5.1。


## 状态：21/21 全部完成（系列收官：M1-M6 + 毕业三件交付物全交付）

## 契约

- 规格来源（唯一）：`00-生成提示词.md`
- 单篇契约：`_contract.md`（本系列专用，覆盖 skill 通用 6 节契约）
- 终审脚本：`_audit-v2.sh`

---

## 进度表

status 取值：`pending` | `writing` | `done`

### Part 1 打开黑盒（2/2 ✅）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 01 | 全景与学习路线地图 | done | 350 | `docs/part1-打开黑盒/01-全景与学习路线地图.md` |
| 02 | 经典智能体理论 → 现代 LLM Agent 映射 | done | 232 | `docs/part1-打开黑盒/02-经典理论映射.md` |

### Part 2 亲手写出第一个 Agent（5/5 ✅）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 03 | Agent 的心脏 — agent loop 与上下文管理 | done | 449 | `docs/part2-亲手写出第一个Agent/03-agent-loop与上下文.md` |
| 04 | 模型选型与推理能力 | done | 515 | `docs/part2-亲手写出第一个Agent/04-模型选型与推理.md` |
| 05 | harness 框架怎么选（读完就有结论） | done | 138 | `docs/part2-亲手写出第一个Agent/05-Python选型地图.md` |
| 06 | 调用 LLM 的三种姿势 | done | 653 | `docs/part2-亲手写出第一个Agent/06-调用LLM的三种姿势.md` |
| 07 | 手写最小 agent（里程碑 M1） | done | 671 | `docs/part2-亲手写出第一个Agent/07-手写最小agent.md` |

### Part 3 接上真实系统（3/3 ✅）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 08 | 把后台 API 变成 Agent 的工具（里程碑 M2） | done | 870 | `docs/part3-接上真实系统/08-后台API变成工具.md` |
| 09 | 真实世界的工具工程设计 | done | 1577 | `docs/part3-接上真实系统/09-工具工程设计.md` |
| 10 | 工具全谱系与安全边界 | done | 1355 | `docs/part3-接上真实系统/10-工具全谱系与安全边界.md` |

### Part 4 记忆与规划（2/2 ✅）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 11 | 记忆设计 — 三层记忆、压缩整理与存储后端 | done | 2133 | `docs/part4-记忆与规划/11-记忆设计.md` |
| 12 | 规划能力设计（里程碑 M3） | done | 2165 | `docs/part4-记忆与规划/12-规划能力设计.md` |

### Part 5 接入 MCP 生态（0/3）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 13 | MCP 是什么，为什么值得学 | done | 571 | `docs/part5-接入MCP生态/13-MCP是什么.md` |
| 14 | 用 FastMCP 把工具升级为 MCP server | pending | - | `docs/part5-接入MCP生态/14-FastMCP包装.md` |
| 15 | 直连 vs MCP 选型与 Go 后台桥接（里程碑 M4） | done | 297 | `docs/part5-接入MCP生态/15-直连vsMCP与Go桥接.md` |

### Part 6 多智能体协同（2/2 ✅）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 16 | 拆解 Claude Code / Cursor / OpenHands / OpenClaw | done | 283 | `docs/part6-多智能体协同/16-拆解成熟Agent.md` |
| 17 | 多智能体 — 模式、通信与实战（里程碑 M5） | done | 476 | `docs/part6-多智能体协同/17-多智能体模式与实战.md` |

### Part 7 工程化上线（3/3，已收口）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 18 | 开发方法、评测、可观测与自改进边界 | done | 764 | `docs/part7-工程化上线/18-评测与可观测.md` |
| 19 | 多轮对话、中断与人工介入 | done | 600 | `docs/part7-工程化上线/19-中断与人工介入.md` |
| 20 | 生产化、安全与既有系统治理（里程碑 M6） | done | 535 | `docs/part7-工程化上线/20-生产化与治理.md` |

### Part 8 毕业与 AgenticOS（1/1，已收口）

| 篇 | 标题 | status | 行数 | 文件 |
|---|---|---|---|---|
| 21 | 毕业项目与 Agentic OS | done | 218 | `docs/part8-毕业与AgenticOS/21-毕业项目与AgenticOS.md` |

---

## 知识台账（Ledger）

> writer 子 agent 读这里获得前情，**不读前几篇正文**。每篇 PASS 后追加一行。

| 篇 | 已讲的核心知识点 | 新引入术语 | 给下篇留的钩子 |
|---|---|---|---|
| 01 | 分界线（聊天→tool call→自主 agent，控制流判定 workflow vs agent）；tool call 全流程（三方分工+ASCII 时序图+消息结构 tool_calls/tool_result/finish_reason）；agent=编排层；核心词表 6 词（LLM/Prompt/Function Calling/Tool Use/Workflow vs Agent/Harness）；21 篇/8 Part 路线图+三门槛自测 | autonomous agent / control flow / tool call / function calling / tool use / workflow / orchestration layer / harness；agent loop、PMTA 仅埋词 | PMTA 与 agent loop 两个词已埋待展开——第 2 篇定义为全系列坐标系 |
| 02 | 五档经典分类（simple reflex/model-based/goal-based/utility-based/learning，每档大白话+后端类比+判定标准，出处 AIMA §2.4）；五档→现代映射（reflex≈无记忆单次 tool call、model-based≈有状态循环、goal-based≈拆计划、utility-based≈模型路由/护栏/LLM judge、learning≈eval 闭环）；PMTA 正式定义（CoALA arXiv 2309.02427）+「要素×经典类型×你在哪篇学它」对照表；为什么回到经典模型 | simple reflex / model-based reflex / goal-based / utility-based / learning / PMTA / CoALA / condition-action rule / critic / learning element / actuator（理解性映射） | 本篇只指认 PMTA 构件未实现——第 3 篇把 M 与 A 落成代码（agent loop 五阶段+上下文管理），可回收 §2.2「回填 messages=内部状态」说法 |
| 03 | agent loop 五阶段（输入→调 LLM→解析→执行工具→追加→循环）+15 行伪代码+三镜头对照（Claude Code 三相位/OpenClaw 六环节/本篇五阶段）；harness 三层分层图+六件套（调度/工具注册/事件流/权限/持久化/可观测）；上下文管理（window/token/budget、忘事两机制、裁剪/压缩/隔离对比、/compact 实测行为）；四类终止条件（模型自判/最大迭代/工具超时/用户确认） | agent loop / harness / context window / token / token budget / compaction / termination condition / tool registry / event stream / permission gate / persistence / observability / stub | 循环里 LLM() 该填什么模型——第 4 篇讲通用 vs 推理模型与国产接入 |
| 04 | 通用 vs 推理模型（现行=单模型 thinking 开关+reasoning_effort none/low/high/max；决策表+loop 三处改动：回传 reasoning_content/终止判定/预算）；提示策略基础版（规划指令+反思指令可抄片段，P4·12 伏笔）；国内落地（DeepSeek/通义/智谱 OpenAI 兼容对照表+Ollama 本地+vLLM 私有化+选型表）；model_compare.py 对比实验（质量/时延/成本四判读点） | reasoning model / thinking mode / chain-of-thought / reasoning_content / reasoning_effort / OpenAI-compatible / base_url / model routing | LLM() 填好了但 loop 散在脚本里——自己写还是用框架？第 5 篇给 2026 Python 选型地图 |
| 05 | 决策篇（按用户要求大幅精简）：harness≠图运行时≠随便一个库（LangGraph 是运行时不是 harness）；成熟 harness 八件套（loop/工具注册/权限HITL/会话持久化/上下文管理/子agent/MCP/可观测）三家齐；按多模型/多agent/多skill/多模式四能力轴打分：**首选 deepagents**（create_deep_agent 一个函数，subagents=/skills= 开箱即有，换模型换 chat model 一处，LangGraph 底座+LangSmith），备选 Microsoft Agent Framework（企业治理/多语言/teams流程），DeepSeek Harness 多模式最全但 developer preview 暂列预研；不选 LangGraph裸框架/claude-agent-sdk(绑Claude)/openai-agents(OpenAI优先)/ADK·Strands(绑云)/CrewAI(偏剧本)/手写(生产)；deepagents+DeepSeek 最小跑通 14 行 | harness 八件套 / multi-agent(subagents) / skills / 运行模式 / developer preview / create_deep_agent / ChatDeepSeek / interrupt_on / permissions | 第 6 篇讲三种姿势（curl 直连/官方 SDK/agent SDK）+key 管理+流式+消息结构；第 7 篇手写最小 agent 仍是"看懂 deepagents 在替你做什么"的前提 |
| 06 | 三种姿势=同一端点三个观察高度（裸 TCP/HTTP client 封装/Web 框架类比，发出同一份 wire JSON）；姿势一协议全貌（官方 curl 逐字段解剖+响应结构+finish_reason 全集 stop/length/content_filter/tool_calls/insufficient_system_resource/aborted）；四件必修事（① 消息结构：四快照演变图+tool_call_id 成对/配对裁剪/中途插入不支持三条硬规则 ② key 管理：环境变量/.env+python-dotenv/.gitignore 三层级 ③ 超时重试：openai+anthropic 包默认 timeout 10min/max_retries 2（_constants.py 双源核验）、重试条件（连接错误/408/409/429/≥500）、流式不自动重试原文 ④ 流式：SSE/chunk/delta、stream_options 必须随 stream:true 否则 400、include_usage 只在末 chunk、思考模式 delta.reasoning_content 先行）；姿势二 extra_body 逃生舱真相（thinking 并进顶层、reasoning_effort 原生参数）+anthropic 对照（/v1/messages、max_tokens 必填、DeepSeek Anthropic 兼容 base_url）；姿势三=调用者换人（messages/loop/Schema 生成被 Runner 接管，发出的仍是同一 wire JSON）；对比表（看得见/看不见/写多少代码/适合谁/何时不选）+选型三句话；动手实践两脚本实测：01 urllib 裸 HTTP + 02 openai SDK with_raw_response 抓 wire JSON 对比——请求体唯一差异 stream 字段、响应结构零差异，本机模拟端点全流程跑通 | endpoint / wire JSON / SSE (Server-Sent Events) / chunk / delta / Bearer token / .env (python-dotenv) / extra_body / with_raw_response / stream_options.include_usage | 零件备齐：第 7 篇把消息四角色/tool_call_id/finish_reason 判定/key·超时·流式配置装进第 3 篇循环骨架，手写最小 agent 跑里程碑 M1（姿势选型=姿势一进眼睛、姿势二进代码） |
| 07 | 骨架补全为真 LLM agent（openai SDK + deepseek-flash；2 演示工具 get_time/calculate；循环+四类终止）；逐行注释（schema 即 prompt/validate_args 校验+错误回喂自愈/finish_reason 判定+非 stop 警告/熔断——官方示例无熔断作教学对照）；踩坑表（幻觉/空参坏JSON/不合规/异常/死循环，各含复现方式）；结构化输出（response_format json_object，prompt 必含 json 字样；strict Beta）与 function call 异同+口诀；M1 动手实践（4 条预置问题+10 条自检清单含熔断/幻觉复现实验）；agent/ 项目目录本篇建立（config/tools/agent/json_mode_demo 共 317 行，桩测试 16 断言过） | JSON mode / response_format / strict 模式（Beta）/ 幻觉调用 / 错误回喂自愈 / 次级校验 | Part 3 接真实系统：工具换成登录→查用户→分配角色的 REST 闭环（M2），暴露 token 过期、4xx/5xx 喂回、写操作确认 |
| 08 | 最小目标服务（FastAPI 脚手架 100 行：JWT 登录/分页查用户/查角色/覆盖式分配，内存数据，四接口 curl 实测；定位=路线①前置验证，真实 Go 后台在 P5·15/P7）；工具设计模式（登录即工具/token 进程内存持有/统一 HTTP 出口自动注入 Authorization/描述即流程教模型先登录/设计模式表/风险分级）；loop 复用（toolset 模块级切换，loop 核心零改动）；M2 闭环（5 轮转写+时序图+11 条自检清单+401/404 错误回喂实测） | 最小目标服务 / 登录即工具 / 描述即流程 / 统一 HTTP 出口 / 覆盖式写操作与风险分级 / 错误信封拆包 | 401 只回喂未自动重登（重登零件已备）；4xx/5xx 混在一处；写操作只有提示词级约束无代码级闸门——第 9 篇补齐 |
| 09 | 认证与会话（幂等登录三性质/401 静默重登+重放一次 _replayed 防风暴/重登也失败的降级路径，类比连接池透明重连）；错误处理（错误分级表 8 行——重试与否判幂等性；状态码+body 结构化回喂；四层兜底链 连接错→HTTP错→业务错→未知错）；护栏起步（确认闸门 fail-closed 挂 execute_tool 必经之路/先只读后写三层落地/审计 JSONL 脱敏打码+denied 单独成态）；代码落地 8 文件（guard.py 38 行/audit.py 61 行/selfcheck_p9.py 262 行 36 断言实测全过/文章与磁盘逐字节一致）；MDN 状态码语义当日核验 | 静默重登 / 重放一次 / 降级路径 / 幂等登录 / 错误分级表 / 兜底链 / 确认闸门 / fail-closed / 警报疲劳 / 先只读后写 / 审计 JSONL | 闸门护住执行时机但挡不住不可信输入进参数——第 10 篇由 REST 由点及面到全工具谱系+工具即攻击面+设计规范小结 |
| 10 | 工具全谱系（五类：REST/代码执行/文件/DB/浏览器，风险等级表 7 行×5 维，锚点 OWASP LLM03 三根因）；工具即攻击面（三条注入路径思想实验：工具结果藏指令=存储型 XSS agent 版/用户输入带指令含无意/文件内容带指令+可信表面投放；闸门挡不住的根因=行为层 vs 输入层；lethal trifecta/Rule of Two）；设计规范五条清单（单一职责/自包含/描述可理解/参数可校验/可观测，带回链）；最小安全清单落地（assign_role 两段式校验=user_id 正则+role 枚举白名单；eval_expr 字符+AST 双白名单，拦截 ** 乘方与 200 字上界；error:rejected 审计成态）；M2 回归复验（36+29 断言+桩闭环+M1 回归）；**OWASP LLM Top 10 现行版=2026 版（2026-08-04 发布，LLM01 Prompt Injection 居首）** | 风险等级表 / 暴露面盘点 / indirect·direct prompt injection / trusted-surface injection / lethal trifecta / Rule of Two / 语义校验白名单 / 两段式校验 / AST 白名单 / error:rejected | M2 助手每次会话从零开始（记不住）→P4·11 记忆篇；不拆任务一步到底→P4·12 规划篇；「记忆是注入的持久化载体」（OWASP 放大器）在 P4·11 记忆写入设计时回收 |
| 11 | 三层记忆模型（in-context/session/long-term=寄存器/内存/磁盘；CoALA 工作/情景/语义/程序记忆；每层 token/成本/时效表）；实战问题（什么该记该忘/串话事故与键设计/语义漂移/回忆失败三档代价）；读写策略（写三档：显式>启发式>模型自判+写入校验两层防线回收 OWASP 持久化载体钩子；读拼接+记忆区/资料区声明）；压缩整理（近 N 轮+滚动摘要 summarized_through 游标/层级摘要/离线压缩/两层 TTL+近似 LRU+语义去重；token 129→323 稳态 vs 423 线性涨实测）；存储选型表（会话→Redis/会话表；长期→Postgres+pgvector 同栈优先，Qdrant/Milvus 规模化再考虑）；实现链条（memory.py 479 行 JSONL→memory_redis_pg.py 144 行 Redis+pgvector 接口同形，HashingEmbedder→fastembed 替换点，<=>/HNSW/IVFFlat）；RAG 分界线（记忆=关于你的检索 vs RAG=关于世界；三问判定；三坑）；selfcheck_p11 59 断言 | 滚动摘要 / 层级摘要 / summarized_through / 两层 TTL / 语义去重 / cross-talk / 语义漂移 / 写策略三档 / 能力探测 / 特征哈希嵌入 / <=> / HNSW·IVFFlat / RAG 分界线 | 助手记得住过去但复杂任务仍一步干到底→第 12 篇规划；M3=记忆+规划联合验收 |
| 12 | Planning 在 PMTA 的位置（工具=可能性/规划=路径；三祸：抢跑/空转/胡编）；ReAct vs Plan-and-Execute 对比表含「什么时候不选它」+计划表示四决定（JSON 五字段 intent-tool-args-success-check-rollback + $引用 + json_object 不带 tools）；生命周期状态机（DRAFT/RUNNING/REGENERATE/ABORTING/DONE）+计划校验四层+重规划三纪律（预算有界 REGEN_BUDGET/journal-writes 跨轮累积/0 步收场）+回滚协议（逆操作过同闸门）；self-correct 两写法+verifier（程序级判据优先，P7·18 评测预告）；帮倒忙三笔开销+token 成本账（规划未必更贵的反直觉结论）；M3 三幕联合验收（抢跑被闸门拦/查无此人重规划 0 步收场/记忆×规划协同 3 步 vs 4 步）+开关对比表；planner.py 484 行 | ReAct / Plan-and-Execute / Plan-and-Solve / 结构化计划 / $引用 / 计划校验四层 / REGEN_BUDGET / 0 步收场 / rollback 逆操作 / MAX_PLAN_STEPS / verifier / --plan 开关 | M3 助手的工具仍是 python import 私有协议——TOOL_SCHEMAS/REGISTRY 只活在进程里，换语言要重写、外部客户端无从接入→P5 MCP 标准化 |
| 13 | 为什么需要 MCP（N×M→N+M 双图+私有协议 vs MCP 对照表，对位第 7 篇 TOOL_SCHEMAS）；协议白话版（JSON-RPC 2.0 与 REST 对位表；initialize/tools/list/tools/call 报文逐字段白话注释；inputSchema/content/isError）；传输（stdio vs Streamable HTTP 对比表+SSE 历史一句）；OAuth 2.1 四角色对位+授权场景表；2026 现状（规范 tag 2026-07-28 仍最新·GitHub tags API 核验；**该版删 initialize 握手走向无状态：每请求 _meta 信封+server/discover**；python-sdk 2.3.0 双时代并存 HANDSHAKE≤2025-11-25 / MODERN 2026-07-28；**mcp 2.x 已把 FastMCP 更名 MCPServer**）；实践三件套本机跑通（weather_server 12 行+裸 JSON-RPC client+两实验，报文实测捕获） | MCP / JSON-RPC 2.0 / notification / method 命名空间 / stdio / Streamable HTTP / 能力协商 / inputSchema / isError / OAuth 2.1 四角色 / _meta 信封 / server/discover / MCP Registry / Inspector / FastMCP→MCPServer 更名 | ①第 14 篇定版本口径（mcp 2.x 新名 MCPServer 或 pin mcp<2，标题含 FastMCP 需处理）②@mcp.tool() 类型注解+docstring 自动生成 schema，四工具包装路径已铺 ③握手期与无状态期两套语义的落点 |
| 14 | 更名与口径（FastMCP→MCPServer in mcp 2.x；锁 2.3.0 跑握手期协议 2025-11-25）；@mcp.tool() 签名即 schema（vs 第 7 篇手写对照表：两份变一份、漂移风险、pydantic int→integer；场景 4 十二断言证等价）；admin_mcp_server.py 108 行（委托原实现/token 搬 server 进程/语义白名单随实现/描述删闸门句/stdout 纪律）；client 端 mcp_client_tools.py 201 行（MCPConnection 后台线程包 async/常驻连接保登录态；MCPToolset 注册件同形三差异：tools/list 发现+OpenAI 信封翻译三件事/isError 回喂/validate_args 只剩第一段）；agent.py --mcp+惰性加载（进程隔离物理前提，loop 零改动第二次兑现）；**保护分层表**（闸门留 client 三理由：无终端/承诺归 client/挂点在 execute_tool；校验审计进 server：唯一入口；双审计文件：server 零增量 vs client denied）；p11 一行同步（测试维护实例）；M4 前置：selfcheck_p14 38 断言+demo_mcp_equiv 7 项（外部后台进程踩坑→HTTP 重置教训） | MCPServer（原 FastMCP）/ 签名即 schema / 保护分层 / GATE_NOTICE / 双审计文件 / 惰性加载 _default_toolset / isError 回喂 / stdout 纪律 | 直连 vs MCP 的账（多一个进程一层协议值不值）+ 真实 Go 后台桥接（官方 go-sdk）——第 15 篇 M4 里程碑 |
| 15 | 六维对比表（学习成本/标准化/调试/部署/多客户端复用/权限模型，每维「什么时候不选它」；论断全部回链第 8-14 篇实证）；生产选型（条件化判定四问+本项目答案：起步直连→第二消费方出现上 MCP→治理平台化留给规模化；胜负手=是否出现第二个消费方）；两种架构图（B=agent 直连 REST 现状；A=Go 进程当 MCP server：工具实现跟后台走+任何 MCP client 复用）；go-sdk 用法片段（45 行 Go server，AddTool 对位 @mcp.tool()、NewToolResultError 对位 isError 回喂、Run(stdin,stdout) 对位 mcp.run()；<未实测，以官方文档为准>）；与 FastAPI 脚手架解耦声明；M4 实践（go build→bridge_check→完整闭环→补齐工具集时保护分层照搬 14 篇表） | go-sdk / 架构 A·B / 语言随系统走 / 第二消费方即回本点 / go mod init·GOPROXY | 单 agent 上下文/记忆/规划/工具全挤一个循环——第 16 篇拆解 Claude Code/Cursor/OpenHands，第 17 篇落地主 agent+subagent |
| 16 | 三层架构共性图（模型适配/harness/应用；我们的六件套对上号+逐件位置表）；四系统逐个为什么（Claude Code：subagent=上下文隔离/hooks=策略代码化/白名单三档/checkpoint JSONL//compact/auto memory/事件流；Cursor：编辑器形态重塑权限粒度（diff 级确认）；OpenHands：sandbox 优先的世界模型（代码执行=万能胶水）；OpenClaw：harness 即插件+六环节（streaming/persistence 升格为一等环节））；对照总表（九构件：五个已有/两个本篇补/subagent 17 篇/沙箱不做）；hierarchical vs flat 决策信号（层级=上下文预算的压力反应）；HITL 风险分级表（粒度/档位/审批流三观察）；实践：指认 subagent+事件流图+hooks.py（43 行，guard 泛化为注册式）+checkpoint/--resume 接线（py_compile 过+回归 213 断言全绿，行为未实测标注） | subagent / hooks（before·after）/ 权限白名单三档 / checkpoint / 事件流 / hierarchical vs flat / sandbox 优先 / harness 即插件 / 审批流 | 第 17 篇把 subagent 落到助手：主 agent+只读查询 subagent 分工跑通 M5，量化 token/延迟对比 |
| 17 | 四动机（上下文隔离/token 分摊/并行/职责边界，各对位前篇痛点）+复杂度守恒警告（多 agent 搬运不消灭复杂度，接口即成本）；五模式选型表（orchestrator-worker/planner-executor/verifier/debate/对称协作，各对位实例与不适合场景，口诀=九成落 orchestrator-worker）；通信三路线（共享上下文=零损耗无隔离/嵌套调用=agent 当工具（工具=确定性函数，子 agent=自带 loop 的工具）/消息传递=平台化预告 P8）；成本账（派发开销/汇总损耗/延迟串行化；P≫C 才值）；M5 实测：subagent.py（ReadOnlyToolset 物理只读/_run_subagent 嵌套换装/HierarchicalToolset；全局状态是嵌套天敌的毛边诚实记录）+demo_m5_compare（主上下文峰值 14→6 条、总调用 7→9 次、保护零丢失、终局等价 PASS、三行结论）；hooks 默认空=回归 213 断言全绿 | orchestrator-worker / planner-executor / verifier / debate / 对称协作 / 嵌套调用 / 派发开销 / 汇总损耗 / ReadOnlyToolset / agent 当工具 | demo≠产品：没评测集/没结构化 trace/没成本限额——P7 三篇工程底子（18 评测可观测/19 中断介入/20 生产治理） |
| 18 | 开发方法（用例集与草稿同时出生=自然语言的类型检查；拆解即验收：三可观察面=工具序列/终局状态/答复内容，与评测断言三类同源）；trace 驱动调试（print 三缺陷：散/无结构/无基线；三问=想什么/调了什么/结果如何；包装法 install/uninstall 不改主路径一行）；可观测（llm 事件含 n_messages=上下文曲线+latency+tokens；tool 事件含被拒调用；audit vs trace 生命周期分家=审计不可覆盖 vs 评测可重置，合并必出事故；生产指针 Langfuse（trace 可视化/prompt 版本/线上评测）与 OTel GenAI SemConv（gen_ai.* 进既有 OTel 管道），选型先问「观测什么、谁消费」）；评测最小可行版（=行为的版本管理/CI：10 条黄金用例按三维度配平——正常读含措辞照妖镜 E09/E10、正常写含写前必查 E06、边界拒绝 E04/E05/E08；四类规则断言 tools_include/exclude/final_roles/answer_contains；规则 vs LLM judge 表：零 token 可复现可进 CI vs 开放质量但抖动，新手从规则开始；run_eval 五设计决定：自管后台+每用例 reset（第 14 篇教训复用）、旁听工具序列、评测期 guard 自动放行=测试边界示范、trace 顺带装上、exit code 进 CI；--stub 评测器自验收/--weak 坏剧本造失败）；自改进边界（回路：trace 复盘→归因三类→修复+失败案例进评测集→回归护住，「失败案例不进评测集=白交学费」；与 Reflexion 划界：运行时反思改当次，评测集改每次，长期改进靠制度；值得/不值得闭环六情形表，大重构期评测作用最大；新手别微调四理由：prompt 边际收益未吃干/评测集太小一调就过拟合（E09/E10 照妖镜）/行为控制粒度粗/成本不可逆，类比=讲规矩 vs 重新招人培训；微调三正当场景均前置万级评测集）；实测：--stub E02 PASS 1/1、--weak E06 FAIL「✗应调用 get_user_roles」（结果对但行为错=只验结果发现不了的核心教学点）+exit=1、trace.jsonl 证物（login 与 assign_role 之间无 get_user_roles 肉眼可见）、eval_report.jsonl | trace 驱动 / 黄金用例 / 规则断言 / LLM judge / 评测集 / 回归护栏 / install/uninstall 包装法 / n_messages / audit-trace 分家 / SemConv / eval 闭环 / 过拟合 / eval_report.jsonl | 助手还是「一条道跑到黑」：用户插不进话/喊不停/改不了需求——19 篇处理多轮对话、中断与人工介入（取消语义/风险分级/停下来问的时机/长任务进度/兜底链） |
| 19 | 打断三形态（插话=吸收不停止/喊停=取消且交代已做/改需求=剩余替换不追溯）；取消语义两派（抢占式 task.cancel：取消不了服务端已执行的请求 vs 协作式=检查点定粒度；检查点位置表：每工具前太细→每轮中→条与条之间=批量天然粒度→任务完太粗）；任务级 checkpoint{指令,remaining,done,skipped} 与 16 篇会章快照互补（对话级 vs 任务级），每条完成即存+原子写（tmp+os.replace，POSIX rename 原子——防半文件）；人工介入分级（L0 只读不问/确认疲劳=无脑 y 是事故源→L1 可逆写 y-N/L2 不可逆+参数复述+影响面摘录/L3 批量=计划先审+逐条可拒；原则=问本身有成本，破坏性越大问得越重）；介入三方式（确认后继续/修正指令/彻底中止）对位打断三态；两级控制（guard 管单条 y-N/介入点管全局 q-改向——拒绝一条≠中止批量）；停下来问四信号（确定性低/破坏性高/信息不足/多选浮现——多选闷头选=替用户做未授权选择）；长任务三件套（进度 k/n+每条摘要/断点续跑/超时四层各有之后怎么办：工具 60s→SDK 重试→熔断→任务层超预算中止+报告）；怪问题速查表（参数幻觉/乱调/编结果/漂移/结果过大/答非所问已兜底收拢；回环检测=指纹窗口事中拦截思路，无失败模式支撑不实现；并发限流→20 篇）；交互侧权限搭配（批量计划审一次+执行中逐条可拒/放行一次→会话放行 hooks 可做→永久白名单=权限蔓延需过期+理由+审计成态/审计对齐表：y=ok,N=denied,s=skipped,q=remaining,改需求=task 变更/身份选型用户 token vs service account→20 篇）；实践：run_batch 介入点设计+三场景（A 打断+续跑/B 拒绝一条批量不断+审计 denied 可查/C 改需求旧条不追溯）——文章内完整代码，未实测（系列工作方式：读者实现验证） | 打断三形态 / 协作式取消 / 中断检查点 / 确认疲劳 / 风险分级 / 两级控制 / 四信号 / 任务 checkpoint / 原子写 / 权限蔓延 / batch_state.json | 交互关已过，最后一道：以什么身份/形态/边界挂上生产——20 篇 M6：可靠性重试熔断降级、成本控制、部署形态 CLI/Web-SSE、prompt injection 反制、租户隔离治理、只读开关+越权用例验收 |
| 20 | 可靠性五件套对应物表（重试=SDK max_retries/超时=四层 19 篇/幂等=agent 特殊点：覆盖式天然幂等、创建式重发=事故→写操作超时要先查证不是重发——先只读后写同时是安全与可靠性纪律/熔断=循环+任务层/降级=摘要折叠+模型链）；成本（prompt caching：agent 每轮全量重发但前缀不变=缓存最佳客户，稳定内容放前易变放后，警惕动态 system；模型路由从模式路由起步别上内容路由——路由层是新故障面；并发 RPM/用户配额/全局预算熔断）；部署三形态（CLI 现状/Web=FastAPI+SSE 流式骨架 40 行未实测留三生产差异点：鉴权/取消/确认/嵌入后台=终局 /agent 入口）；安全专项（injection 三入口×四防线纵深：隔离包裹+最小权限+闸门+终局对拍——9→18 篇每篇一层；沙箱=先问执行什么，HTTP 工具 agent 的对应物是网络出口最小化；密钥 env 不落码不落日志+后台 JWT 系统保管；脱敏三面日志/工具结果/记忆——暴露面与生命周期成正比；国内合规=生成式 AI 暂行办法+数据三法+备案，内部工具 PII/出境红线仍在）；治理 F1（身份选型：用户 token vs service account 表——管理后台选用户 token 权限审计双归零，agent 侧重建权限体系=最常见反模式；RBAC/多租户：复用不绕过——第 8 篇起工具即 API 薄封装零 SQL 直连，服务端 Where(tenant_id)=最后防线+客户端壳=纵深，旁路是治理的癌症；会话/记忆/缓存同按租户隔离：隔离键 user→(tenant,user)、记忆表带 tenant_id、Redis 键租户前缀、自查=每样持久化数据问租户键在哪——记忆串租比操作串租更隐蔽；审计三级=工具行为+服务端双审计+治理决策带 reason；操作限额=介入点的规则化）；M6 实践：governance.py（configure 热切换+governance_gate 挂 16 篇 hooks before：READONLY 拒一切写/TENANT_OF_USER 跨租户拒/_log_denial 记 reason；顺序问题：guard 在 hooks 前故只读仍先弹确认——生产应策略先判人后扰，重排思考题）+demo_m6.py 三用例（①只读拦下后台零改动②跨租户拦李四③同租户不误伤）——文章内完整代码未实测（读者实现验证）；落地清单 12 项（对接/身份/多租户/只读/审计三级/key/限流配额/trace/评测/安装/直连 MCP 最终决定=直连（15 篇触发条件：第二消费方出现即切）/部署形态）；M6 复验口径如实：18 篇实测、19/20 篇读者实现（桩结构与 run_eval 同构） | 幂等 / 先查证不重发 / prompt caching / 模式路由 / SSE / 纵深防御 / 用户 token / service account / 旁路是癌症 / 租户键自查 / 治理决策审计 / 操作限额 / governance.py / 落地清单 | 零件全齐但还是 21 个文件——21 篇终篇：组装终版助手（组装清单+OS 式架构图+落地交接单）+Agentic OS 视野（agent=进程/上下文=内存/MCP=设备驱动/harness=内核/subagent=fork/权限请求=弹窗/注册表=生态） |
| 21 | 上半篇毕业收口（能力清单六件 8-17 篇对位+工程壳四件 18-20 篇对位+选型记录八岔路口含直连最终决定+终版目录树+总架构图 ASCII 含 OS 概念标注）；下半篇 Agentic OS 八条类比（agent=进程有生命周期耗 token/上下文=RAM 易失有限贵且滚动摘要=换页 swap/长期记忆=磁盘持久廉价检索 embedding=索引/MCP server=设备驱动统一系统调用层 N×M→N+M/harness=内核调度 loop=调度器熔断=OOM killer 预算=ulimit/subagent=fork 独立地址空间派发开销=fork 代价/权限请求=系统弹窗能力声明 schema=manifest+运行时请求+授权+审计/agent 注册表=应用生态）；类比价值（OS 五十年治理智慧白拿：配额审计沙箱调度存储层次）与边界三条失效（无抢占式调度硬件基础——取消是协作式/上下文非真 RAM——不能随机访问不能写保护/进程隔离靠硬件 agent 隔离靠约定——全局变量可穿透 17 篇毛边）；视角解决什么（松散产物→平台治理五件：生命周期=进程管理/可观测=syslog·proc/配额=ulimit·cgroup/身份=uid·capabilities/跨 agent 通信=IPC——与第 2 篇呼应：个体生理学 vs 城市规划）；趋势三股（AgentOS 定位 Manus 代表/agent 网关·server=harness 产品化引第 5 篇 2026-10-19 版图/多 agent 编排平台引 17 篇——诚实标注未新检索）；演化三段（裸 agent+工具 7-8→harness+MCP+治理 9-20 当前→agent 平台未做）+「写到多大上什么组件」八档触发式清单（规模信号→组件→不上会怎样，与 15 篇触发条件同精神）；动脑练习四题（画自己的架构图/给自己场景定档/写落地交接单=20 篇 §6.6 加负责人截止日/找类比第四条漏洞=IPC 背压流控）；Part 8 收口：毕业验收三件全交付+M1-M6 全里程碑表 | Agentic OS / 换页 / 设备驱动 / 内核调度 / fork / 权限弹窗 / 注册表 / 平台治理五件 / 写到多大上什么组件 / 交接单 | 系列收官——真实落地另开设计文档，交接单已标入口 |

---

## 待修复项

> 终审（阶段 3）发现的问题记在这里，修完打勾。

- （暂无）

---

## 会话记录

- 2026-10-15：建立脚手架（目录树 + 00-生成提示词.md + _contract.md + progress.md + README 骨架），准备生成第 1 篇
- 2026-10-15：第 1 篇完成（350 行，契约校验全过：问题清单/5 分节/结论/6 条参考资料/零残留标记；知识点关键词全覆盖；无敏感信息）
- 2026-10-15：第 2 篇完成（232 行，契约全过：4 分节/7 条参考资料 curl 实测核验/动脑练习可操作/PMTA 对照表在 §3.4）。Part 1 收口，README 链接已回填
- 2026-10-15：第 4 篇完成（515 行，契约全过：4 分节/11 条参考资料 curl 实测/对比脚本 py_compile 通过）。**发现并双重核验系列级事实：deepseek-chat/deepseek-reasoner 已停用，现行 deepseek-flash + thinking 开关**，已记入上方「系列级技术事实」，第 1 篇实践代码模型名已同步修正
- 2026-10-15：第 5 篇完成（461 行，契约全过：5 分节/8 条参考当日实测/实践 venv+安装本机跑通+代码 diff 一致）。框架版本/star 全部重核并登记系列级事实；确立全系列 Python 3.11+ 要求
- 2026-10-15：第 3 篇完成（449 行，契约全过：4 分节/6 条参考资料当日 curl 实测/实践代码本机 Python 3.9.6 实跑且与文中逐字一致）
- 2026-10-15：第 6 篇完成（653 行，契约全过：4 分节/问题清单 5 条/10 条参考资料当日 curl+源码核验/残留标记 0）。实践两脚本（01 urllib 裸 HTTP + 02 openai SDK with_raw_response 抓 wire）对本地模拟端点端到端实测跑通，文中代码与实测脚本逐字一致，预期输出为实测记录；真实 DeepSeek 调用未实测已标注。SDK 默认值（timeout 10min/max_retries 2）以 README+两包 _constants.py 双源核验
- 2026-10-15：第 7 篇完成（671 行，M1 里程碑达成：契约全过；agent/ 目录 6 文件 py_compile 全过；loop 桩测试 10 场景 16 断言过；文章与磁盘代码 diff 一致）。Part 2 收口（5/5），README 链接回填
- 2026-10-15：第 8 篇完成（870 行，M2 里程碑达成：mock_backend 四接口 curl 实测过；M2 闭环+错误路径桩 LLM×真实 HTTP 6 场景 15 断言过；文章与磁盘 diff 一致；fastapi 0.143.1/uvicorn 0.54.0/pyjwt 2.15.1 PyPI 核验实装）
- 2026-10-15：第 9 篇完成（1577 行，契约全过；selfcheck_p9 36 断言 venv 实测全过；三场景真实 HTTP 桩 LLM 实测；文章与磁盘 byte-exact 校验 ALL OK）。注意：系统 python3 是 3.9 跑不了（str|None 语法），一律用 agent/.venv（3.12.15）
- 2026-10-16：第 10 篇完成（1355 行；断点恢复后续写成功——昨晚限流中断时只有调研无落盘）。selfcheck_p10 29 断言+p9 回归 36 断言 venv 实测全过；文章与磁盘 byte-exact；发现并登记 OWASP LLM Top 10 现行版=2026 版。Part 3 收口（3/3），M2 复验通过，README 链接回填
- 2026-10-16：第 11 篇完成（2133 行，全系列最重知识点篇：8 条全覆盖；selfcheck_p11 59 断言+p9/p10 回归全过；demo_memory 实测 token 稳态与重启召回；redis_pg 运行时未实测已标注，契约桩测过；文章与磁盘 byte-exact）
- 2026-10-16：第 12 篇完成（2165 行，M3 里程碑达成：三幕 demo 6/6 PASS；selfcheck_p12 51 断言新增，全套 175 断言回归全绿；论文出处 arXiv metadata 逐字核验；LangChain 死链诚实规避）。Part 4 收口（2/2）。writer 已自行回填两处 README 链接，根 README 进度计数由主流程补齐
- 2026-10-16：第 13 篇完成（571 行，协议篇：method 名/字段/传输/OAuth 全部官方源核验；实践三件套 mcp 2.3.0 本机跑通，报文实测捕获；Claude Code 挂载命令双源核验）。发现并登记 MCP 双时代口径与 FastMCP→MCPServer 更名两条系列级事实
- 2026-10-19：第 14 篇完成（1561 行；两个 writer 先后被限流打断留下半成品代码，主流程接管补齐 selfcheck_p14（38 断言）+demo_mcp_equiv（7 项）并撰写正文；代码块与磁盘 byte-exact 5 文件全过；p11 一行同步（惰性加载）；清理 writer 残留的外部 uvicorn 进程）
- 2026-10-19：第 15 篇完成（297 行，主流程直接撰写：六维对比/选型结论/双架构图/go-sdk 片段全部标注未实测边界；M4 验收口径明确）。Part 5 收口（3/3），README 链接回填
- 2026-10-19：第 16 篇完成（283 行，主流程直接撰写；hooks.py 落盘+agent.py 接线（hooks/checkpoint/--resume），py_compile 过、全套回归 213 断言全绿（钩子默认空=行为零改变）；行为级验证按未实测标注）
- 2026-10-19：第 17 篇完成（476 行，M5 里程碑达成：demo_m5_compare 实测主上下文 14→6 条消息/总调用 7→9 次的成本账/保护零丢失；subagent.py+demo 落盘实测；代码块注入 byte-exact）。Part 6 收口（2/2），README 链接回填
- 2026-10-19：第 5 篇两轮重写定稿（461→814→**138 行决策篇**）：第一轮补全 2026 下半年 harness 版图后再因"太长/不适合读"应要求精简为决策篇——harness 八件套 + 多模型/多agent/多skill/多模式四能力轴打分，**首选 deepagents、备选 MAF、DeepSeek Harness 列预研**，其余候选一句"为什么不选"；删掉全部冗余表格与 10 个示例代码，只留 deepagents+DeepSeek 最小跑通 14 行；契约校验全过（5 问题/4 分节/7 参考/0 残留），py_compile 过，_audit-v2.sh 全绿
- 2026-10-19：第 18 篇完成（764 行，P7 开篇：trace.py/evalset.jsonl/run_eval.py 三件落盘并 --stub/--weak 两模式实测（E02 PASS 1/1；E06 FAIL ✗应调用 get_user_roles + exit=1=回归护栏现场）；代码块注入 byte-exact 3/3；契约全过（4 分节/7 参考/0 残留）；知识点 grep 全覆盖；真实模式标注未实测）
- 2026-10-19：第 19 篇完成（600 行，实战汇：打断三形态/协作式取消/风险分级/两级控制/四信号/长任务三件套/怪问题速查表/交互侧权限搭配全落；实践 demo_interrupt.py 完整列于文中——按系列工作方式未在本机运行，标注未实测，验证由用户执行；契约全过 5 分节/8 参考/0 残留；知识点 grep 全覆盖）
- 2026-10-19：第 20 篇完成（535 行，M6 里程碑达成+Part 7 收口：可靠性五件套/成本三杠杆/部署三形态含 SSE 骨架/安全纵深/治理 F1 全落；governance.py+demo_m6.py 完整列于文中未实测（读者实现验证）；落地清单 12 项含直连最终决定；契约全过 5 分节/9 参考/0 残留；知识点 grep 全覆盖）
- 2026-10-19：第 21 篇完成（218 行，终篇纯理论：毕业三件交付物（组装清单/OS 式架构图/交接单模板）全交付；八条 OS 类比+三条失效边界+平台治理五件+演化三段+八档触发式组件清单+动脑练习四题；趋势节按工作方式未新检索、引第 5 篇已核验版图并标注；契约全过 5 分节/7 参考/0 残留）。【系列收官】21/21 全完成，M1-M6 全达成，Part 8 README 与根 README 已回填终态
