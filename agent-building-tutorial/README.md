# 从零构建 Agent（智能体调用后台管理系统接口）

> 目标：教我从 0 构建一个能调用现有后台接口的 Agent。语言=Python，
> 路径=先直连 REST 再升级 MCP，落地场景=后台管理助手。
> 学习为主，真实实现等学会之后另开 phased design doc。

## 这个文件夹是干嘛的

| 文件 | 作用 |
|---|---|
| [00-生成提示词.md](00-生成提示词.md) | 主提示词（**v1.0.6**，2026-10-16 最新）。发给 AI 按 Part 逐篇生成学习文档 |
| [docs/](docs/) | 生成的正文（每 Part 一个目录，如 `docs/part1/01-全景.md`） |

## 关键决策（已定）

1. **Agent 语言**：Python；默认模型供应商 = **DeepSeek**（现售 `deepseek-flash` =
   V4.1-Flash；OpenAI 兼容，base_url `https://api.deepseek.com`；思维模式由
   `thinking` + `reasoning_effort`（low/high/max）控制、默认开启；重型档
   `deepseek-v4-pro` 仅 P7·20 路由表用。✅ 旧名 `deepseek-chat`/`deepseek-reasoner`
   （官方 2026-07-24 停用）全系列已于 2026-10-16 统一迁移为 `deepseek-flash`）
2. **接入路径**：先直连 REST（P3）→ 再升级 MCP（P5）→ 真实落地（P7/毕业）
3. **场景**：后台管理助手 —— 登录→查用户/角色→分配角色，自然语言驱动
4. **产出**：教学学习文档 + 可运行 Python 代码目录 `agent/`；学完再真实落地
5. **原有 Go 后台不用管**：backend/ 还在脚手架，不是教程前置依赖；教程参照
   "行业成熟方案三条路线"（SDK 直连现行 REST / MCP Server 包装 / 企业 Agent 平台）
6. **P3 参考服务**：FastAPI 纯 Python 脚手架（临时验证 agent 用，不落地）；
   真实 Go 后台映射放 P5·15（Go MCP 桥接）与 P7
7. **默认堆栈**：从零手写 loop（学习用）→ 需要封装时 pydantic-ai / openai-agents
   （均可配 DeepSeek）；claude-agent-sdk 仅作拆解/对比参考
8. **记忆落库**（P4·11）：短期/会话 → Redis 或 RDBMS；长期 → Postgres 结构化表 +
   pgvector（项目同栈，优先）；Qdrant/Milvus 规模化再上；**会话/记忆/缓存按租户隔离**
   （Redis key 加租户前缀、记忆表带 tenant_id）

## ▶ 当前指针（每次开工先看这里）

- 下一步动作：**全系列 21 篇已完成**（P8·21 已确认 10-16）；**2026-10-16 完成两件事**：
  ① DeepSeek 命名统一迁移（全系列 → `deepseek-flash`，00 升 v1.0.6）；
  ② 知识缺口补全（reasoning_content 回传规则 / 思考模式采样参数限制 / token 换算比例 /
  tool_calls 数组 / DeepSeek Harness）。后续可选动作：
  ① **M1-M6 复验回执**（读者实跑，见各 Part 收口节）——对应里程碑转「已验证」；
  ② 读者回「**落地**」→ 按 P8·21 §1.4 交接单另开"真实落地分阶段设计文档"。
- **待决事项（已全部关闭）**：① DeepSeek 旧模型名勘误——**已完成**（2026-10-16 全系列
  迁移为 `deepseek-flash`，旧名仅在勘误/迁移说明处作历史引用保留，见 P2·4 篇头与
  P7·20 §2.2）。② 磁盘版 `agent/` 代码目录是否组装——**读者已定：不组装**，代码由读者
  照文档自己一步步实现（教程交付物 = 文档，代码以文章内嵌形式为准）。
- **待验证提醒（五项，均不阻塞）**：M1（P2·7 六条自检）、M2（Part 3 复验四条，见
  10 篇收口节）、M3（Part 4 复验四条，见 12 篇收口节）、M4（Part 5 复验四条，见 15 篇
  收口节）、M5（Part 6 复验四条，见 17 篇收口节）待读者实跑后回一声，对应 Part 分组
  行的里程碑转「已验证」
- 执行依据：00-生成提示词.md【系列结构】（对应 Part 的要点清单）+ 本文件【生成与验收流程】
- 恢复规则：进度表中**第一个状态非「已确认」的篇**即当前工作位——「待确认」= 先做读者
  确认收尾；「未生成」= 从它开始生成。不问「上次做到哪」。

**结构速览（v1.0.6）**：P1 概念理论(1-2) → P2 亲手写最小agent(3-7, **M1**) → P3 直连后台闭环(8-10, **M2**) → P4 记忆与规划(11-12, **M3**) → P5 MCP(13-15, **M4**) → P6 拆解成熟agent+多智能体(16-17, **M5**) → P7 工程化(18-20, **M6**) → P8 Agentic OS+毕业(21)

## 逐篇进度表（状态：未生成 → 待确认 → 已确认；里程碑：未验证 → 待验证 → 已验证）

> 文件列已预填全部 21 篇计划路径；实际落盘即以此为准，若文件名有出入以实际为准并回写。
> 备注列是"生成该篇时的注意事项"，新会话生成到该篇前必读。

| 篇 | 篇名 | 状态 | 文件 | 生成日 | 确认日 | 备注 |
|---|---|---|---|---|---|---|
| **P1（1-2）** | 打开黑盒（概念与理论为主） | 里程碑：无 | docs/part1/ | — | — | 第 1 篇含一段最小验证脚本 |
| P1·1 | 全景与学习路线地图 | ✅ 已确认 | docs/part1/01-全景.md | 2026-10-15 | 2026-10-15 | 环境准备小节并入本篇（uv/pip 国内源 + DeepSeek key/.env）；引用限参考资料汇总；勘误(10-16)：模型名迁移 `deepseek-flash` + 输出说明补 `reasoning_content` 一见 |
| P1·2 | 经典智能体理论 → 现代 LLM Agent 映射 | ✅ 已确认 | docs/part1/02-经典理论映射.md | 2026-10-15 | 2026-10-15 | 动脑练习篇（豁免动手实践：对照 PMTA 指认 Claude Code）；勘误(10-16)：PMTA 表 Tool use 行 "P4·14-15（MCP）"→ "P5·13-15"（篇号笔误） |
| **P2（3-7）** | 亲手写出第一个 Agent | 里程碑 M1：最小 agent 跑通（🔄 待验证：末篇已确认，等读者实跑六条自检）| docs/part2/ | — | — | 工程地基 |
| P2·3 | agent loop（harness）与上下文管理 | ✅ 已确认 | docs/part2/03-agent-loop与上下文.md | 2026-10-15 | 2026-10-15 | 工程地基篇；骨架 15 行主循环不接 LLM（本环境已实测）；勘误(10-16)：预告更新 + 补 token 换算口径（中文≈0.6 token/字，以 API usage 为准） |
| P2·4 | 模型选型与推理能力 | ✅ 已确认 | docs/part2/04-模型选型与推理.md | 2026-10-15 | 2026-10-15 | Claude 腿为有条件选做；基线 DeepSeek + Ollama；**勘误(10-16)：整篇重写为 deepseek-flash 单模型两态**（thinking 关/开 + effort 三档；含 reasoning_content 回传规则[带 tools 必传否则 400]、思考模式采样参数限制、Anthropic 格式端点一句） |
| P2·5 | 2026 年 Python 选型地图 | ✅ 已确认 | docs/part2/05-Python选型地图.md | 2026-10-15 | 2026-10-16 | 必答「LangChain 呢」；实践写「先跑通不求甚解」；已补 LlamaIndex/Semantic Kernel 定位（v1.0.2）；勘误(10-16)：模型名迁移 + claude-agent-sdk 行补 Anthropic 端点一句 |
| P2·6 | 调用 LLM 的三种姿势 | ✅ 已确认 | docs/part2/06-调用LLM三种姿势.md | 2026-10-16 | 2026-10-16 | 可运行两脚本 = 直连 DeepSeek + openai SDK 连 DeepSeek；claude SDK 仅形态对照；勘误(10-16)：模型名迁移 + 补 reasoning_content 字段与回传规则、tool_calls 数组（并行调用）说明、Anthropic 兼容端点桥 |
| P2·7 | 手写一个"看得懂每行代码"的最小 agent | ✅ 已确认 | docs/part2/07-最小agent.md | 2026-10-16 | 2026-10-16 | **里程碑 M1**；b-8 单行 JSON 日志已落（log_step）；`agent/` 目录设计已写入正文（tools.py + main.py 全文）；calc 工具层本环境已实测；勘误(10-16)：模型名迁移 + 关键差异①补「append 整对象天然满足 reasoning_content 回传」 |
| **P3（8-10）** | 接上真实系统：后台管理助手（直连 REST） | 里程碑 M2：一句话闭环「登录→查用户→查角色→分配角色」（🔄 待验证：末篇已确认，等读者跑复验四条）| docs/part3/ | — | — | 主线 demo |
| P3·8 | 把后台 API 变成 Agent 的工具 — 第一个闭环 | ✅ 已确认 | docs/part3/08-第一个闭环.md | 2026-10-16 | 2026-10-16 | **里程碑 M2**（Part 首篇，动机高点）；target_service 5 端点（+GET /roles 补闭环翻译）；服务端+工具层 13 条用例已实测 |
| P3·9 | 真实世界的工具工程设计 | ✅ 已确认 | docs/part3/09-工具工程设计.md | 2026-10-16 | 2026-10-16 | 认证/错误处理/护栏起步；tools 后台段整段替换（_request 401 自动重登 + _shape hint + audit）；main.py 宿主层确认钩子；宿主层 5 组用例本环境实测过 |
| P3·10 | 工具全谱系与安全边界 | ✅ 已确认 | docs/part3/10-工具全谱系与安全.md | 2026-10-16 | 2026-10-16 | 末篇收口：M2 复验四条 + 预告 P4 痛点；eval 注入 RCE 演示/字符白名单/REST 参数校验拦截点均已实测 |
| **P4（11-12）** | 记忆与规划 | 里程碑 M3：记住偏好 + 先计划再执行（🔄 待验证：末篇已确认，等读者跑 M3 复验四条）| docs/part4/ | — | — | 设计深度 |
| P4·11 | 记忆设计：三层记忆、压缩整理与存储后端 | ✅ 已确认 | docs/part4/11-记忆设计.md | 2026-10-16 | 2026-10-16 | Part 4 首篇⭐；实测：裁剪策略/SQLite 跨进程/Redis TTL/向量召回；pgvector 段按官方文档编写并标注未实测（留给读者自验）；勘误(10-16)：模型名迁移 |
| P4·12 | 规划能力设计 — 复杂任务怎么被拆开 | ✅ 已确认 | docs/part4/12-规划能力设计.md | 2026-10-16 | 2026-10-16 | **里程碑 M3**（= 11 记忆 + 本篇规划联合验收，复验四条在收口节）；末篇收口；实测场景 B 全链（失败→重规划×2→降级汇报）；② 轻模式改造代码标注未整跑；勘误(10-16)：模型名迁移 + ReAct 段改「关 thinking 的 deepseek-flash / 开 thinking 的外显要求」 |
| **P5（13-15）** | 标准化：接入 MCP 生态 | 里程碑 M4：MCP 版助手 + Python↔Go 桥接（🔄 待验证：末篇已确认，等读者跑 M4 复验四条，见 15 篇收口节）| docs/part5/ | — | — | 先直连后 MCP |
| P5·13 | MCP 是什么，为什么值得学 | ✅ 已确认 | docs/part5/13-MCP是什么.md | 2026-10-16 | 2026-10-16 | Part 5 首篇；N×M/LSP 类比/三角色/四拍/JSON-RPC vs HTTP/stdio vs Streamable HTTP/生态实测表/挂 example server；命令语法已核验、挂载与观察留给读者；勘误(10-16)：FastMCP→MCPServer 更名口径修正（13 篇 3 处、12 篇 1 处、00 篇已同步） |
| P5·14 | 把管理助手工具升级为 MCP server（MCPServer，原 FastMCP） | ✅ 已确认 | docs/part5/14-FastMCP升级.md | 2026-10-16 | 2026-10-16 | 核心：@mcp.tool() 声明式定义 + 5 工具搬进 stdio server + 宿主同步外壳（后台线程事件循环+run_coroutine_threadsafe）；P3 五设计点体检（confirm 留宿主、audit 拆两处 audit_server.jsonl）；mcp 2.3.0 主类更名 MCPServer；代码按官方文档编写未实跑，六条自检留读者 |
| P5·15 | 直连 vs MCP — 选型与 Go 后台桥接 | ✅ 已确认 | docs/part5/15-直连vsMCP与Go桥接.md | 2026-10-16 | 2026-10-16 | **里程碑 M4**；Part 5 收口；六维选型+生产最终建议（MCP server+只读先行+service account）；官方 go-sdk（单一主包 mcp；v1.8.0，v1.7.0+ 支持规范 2026-07-28）30 行量级 server；bridge_test.py 宿主零改动桥接；M4 复验四条；Go+Python 未实跑（gofmt-clean 与语法级已过） |
| **P6（16-17）** | 协同作战：借鉴成熟 Agent、落地多智能体 | 里程碑 M5：主 agent + 只读 subagent 分工（🔄 待验证：末篇已确认，等读者跑 M5 复验四条，见 17 篇收口节）| docs/part6/ | — | — | 实例先行 |
| P6·16 | 拆解 Claude Code / Cursor / OpenHands / OpenClaw | ✅ 已确认 | docs/part6/16-拆解成熟agent.md | 2026-10-16 | 2026-10-16 | Part 6 首篇 ⭐ 实例先行；四家一手核验；三层架构 + 五个为什么 + 对照表 + HITL；实践 B 钩子/checkpoint 已本地实测；核查补"流式 UI"小节（MessageDisplay/黑屏问题） |
| P6·17 | 多智能体 — 模式、通信与实战 | ✅ 已确认 | docs/part6/17-多智能体.md | 2026-10-16 | 2026-10-16 | **里程碑 M5**；Part 6 收口；五模式/三通信机制/成本账；机制级本地实测（minimodel：总量 1.28×、主路径峰值 46%）；核查补"消息传递/交接"命名 + 五模式选型速问；A/B 实验与六条自检留读者 |
| **P7（18-20）** | 支撑上线的工程底子 | 里程碑 M6：带评测/trace/安全/治理/只读开关的可交付版（🔄 待验证：末篇已确认，等读者跑 M6 复验四条，见 20 篇 §6.5）| docs/part7/ | — | — | demo 与上线的分水岭 |
| P7·18 | 开发方法、评测、可观测与自改进边界 | ✅ 已确认 | docs/part7/18-评测与可观测.md | 2026-10-16 | 2026-10-16 | Part 7 首篇；10 条黄金用例 + 规则 judge + JSONL trace 全套代码；评测闭环机制级本地实测（stub 三版：8/10→10/10→7/10 回归全被抓住）；核查补"失败案例吸收"叙事句；LLM judge 与真 agent 接入标注未实跑 |
| P7·19 | 多轮对话、中断与人工介入 — 工程实战问题集 | ✅ 已确认 | docs/part7/19-中断与人工介入.md | 2026-10-16 | 2026-10-16 | Part 7 第二篇 ⭐ 实战汇；交互侧（操作风险分级表/放行一次 vs 永久/何时停下问）；身份选型架构侧见 20；三场景 interact_demo.py 已实测（Py3.9）；CC interactive-mode/permissions 一手核验（2026-10-16）；核查补操作分级表 + cancellation token 术语 + 答非所问兜底 |
| P7·20 | 生产化、安全与既有系统治理 — 落地清单 | ✅ 已确认 | docs/part7/20-生产化与治理.md | 2026-10-16 | 2026-10-16 | **里程碑 M6**；末篇收口；可靠性五件套/成本三杠杆/部署三形态/安全专项/身份与租户治理；三层壳 guard.py 已实测（Py3.9）；**命名换代说明块**（旧名 2026-07-24 停用；迁移已于 10-16 全系列完成）；Web 骨架按文档编写未实跑；核查补 CLI/嵌入两骨架；勘误(10-16)：事实块补 thinking 默认开启 + 三条配套规则；路由表补「快档需显式 low」「effort 官方映射」两细节 |
| **P8（21）** | 写大了怎么治 + 毕业收口 | 毕业：终版组装 + 交接单（🔄 待验证：末篇已确认，等读者做毕业验收——画 OS 式架构图 + 组件清单 + 交接单）| docs/part8/ | — | — | 视野 + 交接 |
| P8·21 | 毕业项目与 Agentic OS | ✅ 已确认 | docs/part8/21-毕业与AgenticOS.md | 2026-10-16 | 2026-10-16 | 全系列收尾；动脑练习（画 OS 式架构图+清单+交接单）；趋势检索已做（agentgateway 5.4k★/kagent 4k★/Dify 159k★/Coze 22k★/Manus"action engine"，**官方未见 AgentOS 提法**已如实标注并回写 00 v1.0.5）；核查补"多 agent 编排平台"行 + 三段演化路径句；勘误(10-16)：补 **DeepSeek Harness** 趋势行（模型厂 agent 脚手架开发者预览） |

## 知识台账（新会话生成下一篇时，只读本表获得前情，不重读已生成的正文）

| 篇 | 已讲核心知识点 | 新引入术语 | 留给下篇的钩子 |
|---|---|---|---|
| P1·1 | 聊天/工具调用/自主 agent 分界线（三台阶 + 自主三把尺子）；一次 tool call 六步与时序图；为什么需要编排层（模型只声明宿主才执行、agent loop 四阶段）；8 Part×M1-M6 路线图 + 必修选学 + 怎么自测；核心词表；动手前环境（uv/pip 国内源、DeepSeek key+.env、Ollama 可选） | agent、tool use、function calling、tool_calls、tool_result、agent loop、harness、workflow vs agent、subagent（预告） | ①「自主」的界定 → 第 2 篇经典分类映射；②harness 仅点名 → 第 3 篇深讲；③PMTA 雏形 → 第 2 篇正式定义 |
| P1·2 | 五档经典智能体分类（叠加式，各带类比 + 判定标准）；经典→LLM 构件映射（无记忆 tool call≈reflex、loop 状态≈model-based、显式规划≈goal-based、守卫/评测≈utility、eval 闭环≈learning）；PMTA 四要素坐标系（要素×经典类型×哪篇学对照表）；为何回这套模型 + 局限 | percept、simple reflex、model-based reflex、goal-based、utility-based、learning、PMTA（正式定义）、verifier（预告） | ①「上下文=内部状态」→ P2·3 上下文管理展开；②goal-based → P4·12 规划；③utility/verifier → P7·18/19；④指认 Claude Code 的规划模式 → P6·16 拆解 |
| P2·3 | agent loop 五阶段（获取输入→组上下文→调LLM→解析→行动/回填）+ 伪代码；harness 概念落地为"loop 内核 + 调度/工具注册/事件流/权限/持久化/可观测"分层；上下文管理三手段（裁剪/压缩/总结）与/compact 类比；终止条件四类（完成判定/max steps/工具超时/用户打断）；**token 换算口径**（1 中文≈0.6 / 1 英文≈0.3 token，以 API usage 为准，离线 tokenizer 可预估）；动手：15 行 agent_loop 骨架实测 | agent loop（正式）、harness（正式）、tool_calls/tool_result、上下文窗口、token 预算、token 换算、trim、compact、max steps | ①"上下文=内部状态"兑现 → P2·7 手写真 LLM 版；②"调试习惯"只提不写 → P2·7 加单行 JSON 迭代日志（勘误 b-8）；③服务层（harness 组件）→ P6·16/P7 逐个加 |
| P2·4 | **deepseek-flash 单模型两态**（thinking 关 vs 开 + `reasoning_effort` low/high/max 三档决策表，默认开=high）；双态接入写法（`extra_body.thinking` / `reasoning_effort`）；**三个工程要点**（① 带 `tools` 时历史 `reasoning_content` 必须完整回传否则 **400**——官方推荐 append 整 message；② 思考模式下 temperature/presence_penalty/frequency_penalty **不生效**、top_p 仅思考模式生效且钳 0.95–1.0；③ 思维链计入输出 token）；Anthropic 格式端点一句（`/anthropic`，桥接 anthropic SDK）；提示策略"规划前置+反思收尾"基础写法；国内落地选项表（DeepSeek/通义/智谱在线 + Ollama 本地 + vLLM 私有化）；动手：model_compare.py 同题 thinking 关/开对比质量/时延/token | deepseek-flash、thinking（enabled/disabled）、reasoning_effort（low/high/max）、reasoning_content 回传规则、chain of thought、模型路由（预告）、prompt 级隐性规划 | ①"规划前置+反思"prompt 写法 → P4·12 升级为显式计划；②路由/成本 → P7·20；③接模型 → P2·6 三种姿势、P2·7 真 LLM 手写版 |
| P2·5 | 2026-10 Python 选型地图：五路线定位（手写直连/pydantic-ai/openai-agents/claude-agent-sdk/langgraph）；"LangChain 呢"专段（star/PyPI 实测数据表 + 链式 API 退居二线 + 官方转向 langgraph）；五维决策表 + 三段式结论（学习手写 → 封装 pydantic-ai/openai-agents → 复杂图编排 langgraph）；DeepSeek 适配各框架方式；五框架 20 行内最小 agent 并排（示意非实测）；动手：openai-agents 12 行跑通不求甚解 | pydantic-ai、openai-agents、claude-agent-sdk、langgraph、LCEL（链式 API 已退出）、环境变量适配 base_url、LlamaIndex / Semantic Kernel（撞见级） | ①框架"先跑通不求甚解" → P2·6 拆三种姿势、P2·7 手写；②langgraph 只提不展开 → P6·17 多 agent 落地；③claude-agent-sdk → P6·16 拆解；④LlamaIndex → P4·11 RAG 分界线提一句 |
| P2·6 | 三种调用姿势递进（REST/SDK/agent SDK）；真实消息结构四角色（system/user/assistant+tool_calls/tool+tool_call_id 配对）+ 与第 3 篇"tool_result"简化版的对应关系；finish_reason 判据；工程细节（key 红线 / timeout+max_retries 什么该重试 / SSE 流式 vs 非流式选择）；动手：curl 与 SDK 双脚本对比结构等价；**＋reasoning_content 字段与回传规则（带 tools 必传否则 400）、tool_calls 数组（一轮可多调用=并行调用）、Anthropic 兼容端点桥** | role="tool"、tool_call_id、finish_reason、reasoning_content、Anthropic 格式端点、SSE、max_retries、timeout、content 块（claude 形态对照）| ①协议闭环 → P2·7 手写完整最小 agent（M1）；②"工具层面的韧性" → P3·9；③流式 → P7·20 Web 形态 |
| P2·7 | 三原料拼装（第3篇骨架 + 第6篇协议 + 第4篇模型）→ `agent/` 项目（tools.py + main.py 全注释 ≈70 行）；"工具描述变成 prompt"落在 TOOLS_SCHEMA（name/description/parameters 三样决定模型选不选、填不填对）；registry 注册表；四坑兜底（幻觉调用/空参坏参/工具异常/死循环）统一"喂回模型 + MAX_STEPS 宿主刹车"；校验分层（轻量=JSON可解析+TypeError，严格版留 P7·20）；结构化输出 vs function calling 对照表（JSON 模式 response_format）；M1 自检六条（单工具/双工具复合/算对600/幻觉诱导/1÷0兜底/刹车可见）；**＋append 整对象天然满足 reasoning_content 回传**（思考模式默认开启下本 loop 零改动兼容） | ast 白名单安全计算器（工具=攻击面）、TOOLS_SCHEMA、registry、MAX_STEPS、四种兜底、response_format JSON 模式、M1 里程碑 | ①M1 是玩具接不上真实系统 → P3 直连后台（token 过期/业务错误码/危险写操作，P3·9-10）；②工具描述设计规范 → P3·10；③结构化输出 → P4·12 显式计划载体；④单行 JSON 日志 → P7·18 正式 trace |
| P3·8 | 主场景落场（第 8 篇起不再换）；路线① 形态：agent 只通过 HTTP 看后端、语言解耦；临时 FastAPI 目标服务 target_service（5 端点：login/发JWT、users 分页、roles 目录、user_roles 查、assign 幂等，内存数据）；**为什么多 GET /roles**=闭环缺数据给查询工具别让模型猜；API→工具五点映射表；**工具三模式**：登录当工具（失败喂回）、token 进程内存不进模型上下文、_auth_headers 自动注入 Authorization、description 写"必须先登录"；registry.update + TOOLS_SCHEMA += 循环零改动；main.py 两处改（SYSTEM f-string 注入 .env 凭证 + MAX_STEPS 12）；tools_smoke.py 零 token 工具层冒烟（10 条实测）；curl 中文 query 必须 URL 编码（400 坑）；M2 自检六条（含 401 自愈） | JWT、Bearer token、FastAPI、uvicorn、pydantic BaseModel、幂等、Authorization 自动注入、进程内存 token、URL 编码、M2 里程碑 | ①裸奔版三缺：token 过期无重登/错误处理最简/写操作无确认无审计 → P3·9 装件（登录幂等+过期重登、错误链、二次确认钩子、先只读后写、审计落盘）；②敏感信息不进上下文正名 → P3·10；③"每次新进程重新登录"的失忆痛点 → P4·11；④Part 收口复验 M2 → P3·10 |
| P3·9 | 裸奔版三伤口（认证脆弱/错误一锅端/写操作无护栏）；**认证收回宿主**：login 幂等（乐观复用+兜底双保险）、_creds 记凭证、_request 统一入口 401 自动重登一次（对模型透明）、重登失败喂回；**错误文本公式**=状态码+hint+body（hint 表=宿主侧知识）；兜底链四层图（run_tool except ← _shape ← _request ← 工具函数）；"不该重试"决策表（401 重试一次 / 400·404 不重试 / 500·网络留 P7·20）；**护栏三件套**：确认钩子装宿主层三理由（一处名单/工具纯粹/拒绝不进工具）、拒绝也是数据（"不要重试"指令）、先只读后写两层（SYSTEM 规则 5 + 上线初期只放读工具）、审计落盘 audit.jsonl（tool_call/tool_result/relogin/rejected 四事件；给事后的人看 vs log_step 给调试）；实测：破 token 自动重登/404 hint/ConnectionError 兜底/拒绝与放行审计全记录 | transparent retry、_request 统一入口、_shape/hint 表、兜底链（四层）、idempotent login、DANGEROUS_TOOLS、confirm 钩子（HITL 最小版）、审计 vs 调试日志分立 | ①工具全谱系与风险等级/工具即攻击面/设计规范小结 → P3·10（收口复验 M2 加装后仍可跑）；②确认交互粒度与放行策略 → P7·19；③权限分级/审计治理（含审计自身脱敏）→ P7·20；④500/网络重试退避 → P7·20 |
| P3·10 | 工具五家族全谱系（REST/代码执行/文件/数据库/浏览器）+ **风险三级阶梯**（读<写<执行）+ 各家族最小防线；"由点及面"认知=宿主服务模式属 harness 不属 REST；**工具即攻击面**：prompt injection/间接注入思想实验 + 攻击链五步图（②③④在模型黑盒，⑤宿主可掐断）+ 五家族攻击面速查（参数注入/RCE/路径遍历/SQL/浏览器会话）；**设计规范五条小结**（单一职责/自包含/描述可理解/参数可校验/可观测）+ 对照自查表 + 新工具上架检查五问；动手：eval 版被注入**实测执行了系统命令**（RCE 演示）→ 字符白名单版挡住 → 诚实指出指数炸弹边界（字符集≠终点）→ 落回 ast 白名单；REST 参数形状校验 isdigit + 拦截点实测（服务停：C1C2 不发请求 / C3 走网络证明不误伤）；Part 收口：M2 复验四条 + P4 两痛点预告 | prompt injection、indirect prompt injection、RCE、path traversal、沙箱 sandbox、根目录 jail、只读账号、工具集收窄、风险三级阶梯 | ①记忆（每次忘 + token 预算）→ P4·11；②规划（一步干到底）→ P4·12；③注入防御深化 → P6·16；④权限/治理/沙箱资源限制全版 → P7·20 |
| P4·11 | 从 M2"每次都忘"痛点切入（进程/窗口/跨会话三层丢失）；三层记忆模型（in-context/session/persist 表：存哪不花 token、取什么进上下文才花；每层读写两时机）；三问过滤器（能查的不记=P3·8 姊妹原则）+ 该记/不该记表；三个实战问题（串话=`{tenant}:{user}` 隔离、语义漂移=要点式摘要+原文升级 persist、回忆失败=时间戳+工具核实）；读写策略（写三口：显式信号/回合提炼/离线；读三召回器：最近/相关/重要+预算+来源）；压缩（近 N 轮+摘要的三触发时机；单层 vs 层级摘要；遗忘四策略 TTL/LRU/语义去重/降级）；存储选型（记忆类型×存哪×何时用总表；Redis 三职责=会话/token 缓存/并发锁；pgvector 优先三理由=同库同事务/少组件/租户过滤一条 SQL；Chroma 中间档；Qdrant/Milvus"证明顶不住再上"）；RAG 分界线（记忆 vs 知识检索对照表；四坑：检索跑偏=阈值召回/上下文爆炸/时效/召回内容=不可信输入回连 P3·10）；实测：裁剪 2049→268（13%）且 4 事实全保、memory.py 跨进程（压缩 11 次/消息表有界 14 条/新进程引用偏好，含"UPDATE 无行静默丢摘要"坑修复）、纯 stdlib 向量召回含阈值不召回、Redis TTL 到期即忘（pgvector 段未实测已标注）| 三层记忆、in-context/session/persist、滚动摘要、串话 cross-talk、语义漂移、TTL/LRU/语义去重、分层摘要、pgvector、HNSW、embedding/余弦相似度、阈值召回、RAG、Chroma、docker compose | ①"一步干到底"→ P4·12 规划设计（M3=11+12 联合验收）；②token 挪 Redis（P3·9 遗留）文中已点；③注入深化（检索内容=不可信输入）→ P6·16；④RAG 完整管线（文档摄取）→ 真实落地另开 |
| P4·12 | 从 M2 第二痛点"一步干到底"切入（三特征：局部最优/无全局视图/错了不回头）+ PMTA 定位（T 工具提供可能性、P 规划决定路径；M3=11 记忆+12 规划）；三档形态（隐式 → ReAct → Plan-and-Execute）+ ReAct 三拍（Thought 可审计可干预；M2 loop 已是骨架）+ 对比表；**计划表示**："能执行"vs"像话"=tool/args/done_when 三字段各对一机制、done_when 可观察=与愿望清单的分界线、步骤间数据引用、P2·7 结构化输出钩子兑现；**生命周期**：生成两做法 / 宿主校验（计划=不可信输入，错误清单喂回重生成）/ 轻模式 vs 重模式表 / 重规划三要点（带失败观察+限预算+降级目标）/ 回退回滚（先只读后写+检查点 fix-forward+幂等）；反思与验证（critic pass vs step check；verifier 生成者≠检查者、防自我强化；量化留 P7·18，观察点=成功率/步数/重规划次数/瞎试占比）；规划成本账+决策表+动态升级（5 次工具不收敛即改计划）；实测 plan_check.py 三场景（顺利 3 步 / 失败→两次重规划→降级如实汇报[含"空列表≠拿到"判定坑] / 校验器拦幻觉工具名+缺字段+散文步骤）；② 轻模式改造（PLAN_SYSTEM + make_plan[json_object 包一层] + 注入 SYSTEM + 重规划钩子，标注未整跑）；③ 开关对比（audit.jsonl 当量尺）；Part 4 收口=M3 联合验收四条 | ReAct、Plan-and-Execute、Thought/Action/Observation、计划表示、done_when、计划校验、重规划、降级目标、commit point、fix-forward、critic pass、verifier、自我强化、动态升级 | ①工具层标准化 → P5 MCP（13 是什么/14 FastMCP 升级/15 选型与 Go 桥接；M4）；②评测量化（黄金用例+通过率）→ P7·18；③高风险重模式（宿主执行引擎）→ P7·19-20；④多 agent 分工=规划的另一形态 → P6·17 |
| P5·13 | Part 5 开场"N×M 问题"（N 个 agent×M 个系统 → N+M；LSP 类比；意义=P3 的 5 工具升级为 server）；三角色（Host/Client/Server）+ 原语（Tools 主线，Resources/Prompts 提及）；**心智模型对照表**（TOOLS_SCHEMA↔tools/list、registry↔server 进程、run_tool↔tools/call、返回值↔result.content、喂回错误↔isError）——"MCP 没有新概念、只有标准化：工具三件套搬到进程/网络边界"；协议四拍（initialize 版本协商 / tools/list / tools/call / result）；JSON-RPC 2.0 vs HTTP REST 对比表（消息格式 vs 传输两层）；传输选型（stdio=子进程/进程边界即信任边界 vs Streamable HTTP=远程/OAuth）+ **server 即权限边界**（工具集收窄升为架构原则）；2026 生态实测表（规范 2026-07-28 / mcp 2.3.0 / 10 语言 SDK / servers 91.3k★ / MCP Registry / Claude Code 一等公民）；动手=给 Claude Code 挂 memory server（claude mcp add 语法实测核验含 -s scope；npx 包名官方 README 一手；trace 三层次：界面调用块 / server stderr / MCP Inspector），挂载留读者 | Host、Client、Server、tools/list、tools/call、isError、JSON-RPC 2.0、stdio、Streamable HTTP、OAuth、MCP Registry、MCP Inspector、LSP 类比、原语、权限边界 | ①把 5 工具改写成 `@mcp.tool()` 声明式 server → P5·14（mcp 2.3.0）；②直连 vs MCP 选型 + Go 桥接 → P5·15（M4 收口）；③授权深水区（scope/token/审批）→ P5·15 / P7·20 |
| P5·14 | ①声明式：@mcp.tool()+类型注解+docstring 三件套替换手写 TOOLS_SCHEMA（type hints are the contract；默认值=可选参数）；**2.x 事实：FastMCP 更名 MCPServer**（旧导入 ModuleNotFoundError，官方 migration 核验）；②搬家：tools.py 主体**逐字未改**进 admin_mcp_server/server.py（_token/_creds/_request 401 重登/_shape 全部随迁）；子进程三规矩（env 白名单不继承→env= 显式传；stdout 是协议通道；python server.py 静默等 stdin）；token 隔离升级=宿主进程不再持有；③宿主端 mcp_tools.py 同步外壳（后台线程事件循环+长驻协程持 async with+run_coroutine_threadsafe；Client 生命周期=连接=进程寿命；isinstance 收窄 content；input_schema 直通换信封）；main.py 仅两行替换（tool_names/call）+close 收尾，agent_loop 零改动；④**P3 五设计点体检**：token/重登/_shape 随迁 server、**confirm 留宿主**（annotations=read_only_hint/destructive_hint 是 hints not security）、审计拆两处（宿主记决定含 rejected；server 记 relogin；audit_server.jsonl 防同名互踩）；13 篇心智对照表逐行兑现；⑤实践：等价性验证（同句话两版）+N+M（claude mcp add 同一 server 给 Claude Code）+六条自检；**代码按官方 python-sdk 2.3.0 文档编写、未实跑（编写环境 Py3.9）** | MCPServer（原 FastMCP）、@mcp.tool()、ToolAnnotations（read_only_hint/destructive_hint）、StdioServerParameters、Client、run_coroutine_threadsafe、同步外壳（MCPToolbox）、input_schema、is_error/content、子进程环境白名单、hints not security | ①直连 vs MCP 六维选型 + Go SDK 桥接（M4 收口）→ P5·15；②分布式审计聚合 → P7；③annotations 的工程化用法 → P7·20 |
| P5·15 | ①路线全景三图（A 直连=工具在宿主 / B Python server=P5·14 / C Go server=本篇；**宿主零改动、只换启动命令**——协议是唯一接口）；路线①教学脚手架（FastAPI target_service）vs 路线②真实落地（Go 后台加 MCP 入口，教程只演示形态）；②**六维选型表**（学习成本/标准化/调试/部署/多客户端复用/权限模型）+ 直连够的信号（≥3条）与必须 MCP 的信号（≥1条：多客户端/跨团队跨语言/生态复用/权限边界/凭证归属）+ **生产最终建议**=原型直连、生产且多方复用用 MCP、后台管理类=MCP server+只读先行+service account；③官方 Go SDK 一手事实（5,231★；**v1.8.0**（2026-09-14）；v1.7.0+ 支持规范 2026-07-28；**单一主包 mcp**+jsonrpc/auth/oauthex；go get …/mcp）+ Python↔Go API 对照表（NewServer/AddTool、Input 结构体+json/jsonschema 标签=声明式、TextContent 返文本、error→IsError）；④30 行量级 server 全文（list_users；错误=文本同 P3·9 铁律；env 白名单规矩①的 Go 版；先 go build 别 go run 的两个理由）+ go mod init/get/build 命令；⑤bridge_test.py：**MCPToolbox 零改动**连 Go 二进制（env= 显式传 BASE/TOKEN）+ 预期输出；真实落地三差异（登录归 server=service account / mock→真后台 / P3·9 工程件照补）；⑥Part 5 收口：M4 复验四条+五篇回顾+Part 6 引出（问题从能力转向组织） | Go SDK mcp 包、NewServer、AddTool、StdioTransport、Input 结构体 schema 推断、jsonschema 结构体标签、TextContent/CallToolResult、IsError、go build 产物挂载、bridge_test、service account、路线①/② | ①拆解 Claude Code/Cursor/OpenHands/OpenClaw（三层架构/subagent/hooks/checkpoint）→ P6·16；②service account 身份映射 → P7·20；③真实 Go 后台 MCP 入口落地 → 学完后 landing doc |
| P6·16 | ①定位=实例先行（Part 6 开场；四样本卡片：CC 闭源/终端·Cursor 闭源/IDE·OpenHands 开源 91,040★·OpenClaw 开源 392,152★；2026-10-16 curl 一手核验，附"文档站 .md 直读 + llms.txt"取资料技巧）；②**三层架构共性**（模型适配/harness/应用 ASCII 图 + 四家映射表；Cursor 官方三组件原话 Instructions/Tools/Model；三判决=模型是租的/harness 是各家真功夫/应用层接口不变）——兑现 P2·3 harness 定义；③**五个为什么**：subagent=上下文隔离（独立窗口+system prompt+工具+权限原文；内置 Explore/Plan 只读且不载 CLAUDE.md；15,000 token 描述预算警告；"只读就别给派生能力"+嵌套树；hierarchical vs flat 决策表）；hooks=确定性插槽（三节拍事件表约 30 事件；exit 2 阻断[即使 JSON allow 也拦] / exit 1 非阻断警告 / 超时默认不阻断；matcher+if 收窄；官方 block-rm 示例）；权限=分级梯子（Cursor Run Modes 三档表+Auto-review 执行顺序+"not a security boundary"；默认值哲学：读放行/编辑放行但配置文件要批/终端要批/MCP 逐调用批/网络三口）；checkpoint=会话内撤销栈（CC 每回合打点/100 快照/随会话/约 30 天/rewind 六项/四类局限/不是 VCS 替代；Cursor 只回文件不回消息；两条状态线=文件+消息）；事件流（OpenHands 三句原话+状态归属表；CC hooks 即公开事件流；OpenClaw Gateway control plane + trusted gateway untrusted execution）；④**对照表 14 行**（7-15 篇已实现 8 项含审计四事件/confirm 最小版；缺 hooks/checkpoint/subagent/权限分级/trace/沙箱各有处置）；⑤HITL 三类风险（写/花钱/边界指令）+ 四落点表（工具内/hooks/应用层/异步 defer）+ 两结论；⑥实践 A 指认练习（两个只读 subagent 定义 md 全文+五观察点+事件流填空图+settings hooks 片段[按官方文档编写未实跑]）；⑦实践 B **已实测**：hooks 注册表+checkpoint 脚本 149 行纯 stdlib（Py3.9 实跑：run1 两步后中断→session.json 落盘→run2 恢复+delete_user 被 readonly_policy DENY 且未触达执行体；输出全文粘贴；三设计点=恢复点落在模型调用间/原子写 tmp+rename/拒绝也要回填） | 三层架构（模型适配层/harness 层/应用层）、subagent 上下文隔离、描述预算、hierarchical vs flat、PreToolUse/PostToolUse、exit 2 阻断、permissionDecision（allow/deny/ask/defer）、Run Modes、best-effort guardrail、checkpoint/rewind、原子写、defer、事件流（streams events to clients）、control plane、trusted gateway untrusted execution | ①多智能体模式/通信/成本账 + M5 只读 subagent 实战 → P6·17（Part 6 收口）；②权限分级配置化 → P7·19/20；③正式事件流/trace → P7·18；④沙箱 → P7·20；⑤异步审批=挂起落盘恢复 → P7·19 |
| P6·17 | ①从观察者到设计者（16 拆解→本篇抽象：4 动机/5 模式/3 通信机制/1 落地=M5）；②**四动机**（上下文隔离/分工专精/并行/职责边界）+ 两条钉子（"token 分摊=分窗口压力不是分账单"；**复杂度守恒律**=复杂度从上下文搬进协调；Anthropic 早期翻车=50 subagent/重复劳动/互相打扰）+ 决策门槛三条；③**五模式表**（orchestrator-worker[BEA 原话]/planner-executor[P4·12 重模式+Cursor Plan Mode]/verifier[evaluator-optimizer]/debate[arXiv 2305.14325]/handoff[openai-agents]）+ 三要点（模式可叠加/planner 独立的真价值/模式先于框架）；④**三通信机制表**（共享上下文=其实没拆/嵌套调用 agent-as-tool=CC+Anthropic Research+M5/handoff=控制权移交）+ 三工程注意（回传物=委派四要素 objective/output format/tools/boundaries+"game of telephone"写文件传引用；失败也要回传；写操作别外包）；⑤**成本账**：4×/15× 事实 + 三本账表（token↑/延迟[并行↓串行↑]/可维护性↓）+ 规模规则表（1 agent 3-10 调用 / 2-4 subagent×10-15 / >10）+ MAX_STEPS=5 来由；⑥**M5 实践**：设计图（agent 当工具三决定表）+ scout.py 全文（READONLY_TOOLS 过滤+双保险+subagent 事件）+ main.py 三处改动（LOCAL_TOOLS 分支/chat 记账口子 TOKENS）+ A/B 对照实验表与**诚实预期**（B 总量更大可能更慢、主上下文更小；三行结论模板）；⑦**机制级本地实测**（minimodel.py 纯 stdlib，口径=每次调用重读全上下文字符数）：A 4 调用/1082 字符/峰值 443；B 6 调用/1386（1.28×）/主路径峰值 204（46%）/隔离检查 False=干净；结论"买的是隔离不是省钱"；⑧Part 6 收口=M5 复验四条 + Part 7 预告 | orchestrator-worker、planner-executor、verifier/evaluator-optimizer、debate、handoff、agents-as-tools、嵌套调用、共享上下文、委派四要素（objective/output format/tools/boundaries）、game of telephone、传引用、复杂度守恒律、规模规则、15× 成本、MAX_STEPS、LOCAL_TOOLS、TOKENS 记账、A/B 对照 | ①评测/trace/可观测（TOKENS 升级为正式 trace）→ P7·18（Part 7 开场）；②审批交互侧 → P7·19；③生产化/治理/M6 → P7·20；④多租户批量排查的更大编队 → 学完后落地设计 |
| P7·18 | ①Part 7 开场=从"能跑"到"敢上线"（M6=评测报告+trace+安全壳+治理壳+只读开关）；②**开发方法五步环**（需求→可验证子任务→草稿→3-5 用例先行→失败驱动；非确定性 vs 写函数对照表；调试=trace 驱动三重黑盒；反模式三连）+ audit vs trace 读者对照（合规 vs 开发）；③**评测最小版**：用例两来源（真实样本+bug 固化=回归本体）、判定三档（包含/工具断言[agent 独有]/行为规则 no_tools）、judge 分工表（规则先写满；LLM judge 三纪律=不同款 prompt/评分 prompt 进版本/关键用例人抽检；Anthropic "scales when done well" + "start small-scale right away"）；④**可观测**：trace 事件表（turn_start/llm_call/llm_usage/tool_call/tool_done/subagent_*/turn_end）+ JSONL 四理由 + 三种复用（评测数据源/成本报表/失败回放）+ Langfuse（35,903★）与 OTel GenAI（原页面 Moved→独立仓库 semantic-conventions-genai 450★，含 MCP 约定）指针表；⑤**自改进五级阶梯**（失败→用例集/改 prompt/改结构/reflection/微调；收益↓风险↑）+ 微调三问 + 红线=自动改动必过评测集；⑥**实践已实测**：trace.py（Span 计时器）/ evals.py（judge+runner+报告落盘）/ cases.json 10 条（边界#5#9/空结果#7/工具断言#3#8）三文件与文章块 diff=0；stub 三版真实输出 v1 8/10→v2 10/10→v3 7/10（丢 v2 修复+新回归全被抓住）；⑦接真 agent 两处替换（offset 收集 trace tool_call）+ LLM judge 示意[未实跑]；五条自检 | 黄金用例、规则 judge vs LLM judge、工具断言、回归清单、JSONL trace、Span、Langfuse、OTel GenAI SemConv（Moved 至独立仓库）、自改进五级阶梯、微调三问、评测即护栏 | ①多轮/中断/介入 → P7·19；②生产化治理/身份/RBAC → P7·20；③trace→Langfuse/OTel 正式接入 → 学完后落地设计 |
| P7·19 | ①定位=工程层问题集（16 篇审批设计→本篇落地细节）；②**打断三语义表**（喊停/插话/改需求）+ CC 两条设计律（"Claude keeps the work done so far"=中断不丢工、排队消息 "as soon as those tool calls finish, within the same turn"=落点在步边界）+ 取消检查点放步边界（工具=事务边界，内部不设）+ 停止三件事（落盘/汇报 X/Y/把"已停止"当结果返回）+ 暂停（同进程挂起）vs 取消（落盘续跑）；③**介入三方式 y/c/n**（approved/rejected/aborted）+ 修正指令本质=用户的话**作为 tool 结果回填**（对照 CC 的 Tab 附备注）；confirm() 升级 diff +8 行；④**何时停下问**=四信号表（破坏性高/确定性低/信息不足/多选浮现）+ 一条判据（错了修得回吗、成本多大）+ 反向防"盲按 y"；⑤长任务：进度=步骤级（trace 供数）/断点续跑+幂等（只读重查无害/写操作幂等键或只跑未完成段）/超时四层表（LLM/工具/子任务/回合）+ MAX_STEPS（防绕圈）vs 超时（防卡死）；⑥**十个怪问题兜底表**（参数幻觉 P3·9/乱调 P7·18/编结果/上下文漂移 P4·11/结果过大 P6·17/回环[本篇新增 detect_loop 8 行]/烧钱/并发限流/串话/错误滚雪球）；⑦权限交互侧：**操作风险分级表**（只读=白名单 / 可逆写=每类问 / 不可逆·出站付费·敏感数据=每次问）+ "不再问"两种寿命表（文件修改=会话级 / Bash·域名=永久按仓库粒度；原则=**能完整展示范围才给永久**）+ allowlist.json 升级 + 审计对齐（approved/rejected/aborted/rule_added 闭环）+ 身份选型指路 20 篇；⑧实践**已实测**（Py3.9）：interact_demo.py 三场景（优雅停止 2/5+task_ckpt.json 恢复 / 修正指令回填+audit / 步边界注入），文章块与实测 diff=0、输出原文粘贴；接真 agent 三处改造（SIGINT 置位/confirm 三选一/每步末尾 drain 队列）+ 三条验收；⑨篇末预告 P7·20（M6 收口） | 取消语义（cancellation）、中断检查点、步边界、排队消息（queue interjection）、暂停 vs 取消、修正指令（correction as tool result）、四信号判定、幂等键、超时四层、回环检测器、allowlist、rule_added、会话级 vs 永久放行、能展示范围才给永久 | ①生产化/身份选型（用户 token vs service account）/RBAC/租户隔离/审计治理/M6 收口 → P7·20；②交互壳的 Web 化（停止按钮/插话接口）= 落地时；③审批异步化（defer 挂起落盘）已在 16 篇点过 → 落地设计 |
| P7·20 | ①定位=系统侧（19 交互侧→20 系统与系统）+ M6 定义；②**可靠性五件套表**（重试/超时/幂等/熔断/降级→agent 对应物；特殊处="所有处理喂回模型"；两坑=不在回合层重试、熔断治屡败+回环治重复）+ 熔断最小实现；③**成本三杠杆**：prompt caching（DeepSeek 默认开启前缀缓存、hit/miss 计费字段、命中/未命中 50× 价差、前缀稳定设计推论）/ **模型路由**（含"命名换代勘误"块：deepseek-chat·reasoner 2026-07-24 停用→**deepseek-flash**（V4.1-Flash）/ **deepseek-v4-pro**；thinking effort low/high/max 三档路由表；宿主挑模型）/ 并发预算（步数/token/钱/限额四层 + 峰谷价跑批排 off-peak）；路由表两细节=thinking 默认开启「快档需显式 effort=low」+ effort 官方映射（如 medium→high）；④**部署三形态**（CLI/Web/嵌入对比表 + Web 最小骨架 FastAPI+SSE[按官方文档编写未实跑，编写环境 Py3.9 无 fastapi] + sse-starlette 3.5.0[Py≥3.10] + /stop /interject 接口呼应 19 篇 CANCEL/队列）；⑤**安全专项**：注入反制四件套（工具返回=不可信输入打标/高危动作永远确认/只读默认/输出不二次执行）+ OWASP LLM Top 10 2025 对标（LLM01 注入/LLM06 过度自主/LLM02 泄露/LLM05 输出处理/LLM10 无界消耗）+ 沙箱现状评估（无代码执行工具→需求低）+ 密钥三级三铁律 + 审计脱敏 redact() + **国内合规四条**（办法 2023-08-15：第十四条"停止生成/停止传输/消除+报告"、第十一条脱敏法源；标识办法 2025-09-01：显式+元数据隐式标识、第九条留存≥6 个月；备案前提；以官方为准）；⑥**既有系统治理 F1**：身份选型表（用户 token vs service account→默认 service account+高风险写借用户 y/c/n+**"第四种主体"**审计带 agent_id+on_behalf_of）/ RBAC 双闸门（工具注册表收窄+后台 role_permissions）/ **多租户三层防线表**（工具层宿主注入[本篇实测]/应用层 `.Where(...TenantID.Eq)` 惯例/数据层 RLS 可选）/ 会话记忆缓存同租户直套 P4·11 / 审计服务端化+**事件全集 8 事件**（+tenant_override）/ 操作限额三档；⑦实践**已实测 Py3.9**：guard.py（三层壳=只读开关+租户注入覆盖+审计；越权用例=模型传 t2 被覆盖为 t1+tenant_override 留痕+写操作只读拦截回填）；文章块与实测 diff=0、输出原文粘贴；接真 agent 挂载点（run_tool 外包 guard：校验→确认→执行）；⑧**M6 清单打勾收口**（评测报告/trace/安全壳/治理壳/只读开关全 ✅）+ **M6 复验四条**（只读拦截/租户覆盖/cache 命中记录/审计无明文）+ Part 7 三篇回顾 + 最小落地清单 8 项；⑨预告 P8·21 毕业 | prompt caching / prompt_cache_hit_tokens、模型路由、thinking effort（low/high/max，默认开启）、峰谷定价、熔断 cooldown、service account、"第四种主体"、tenant_override、RLS、Excessive Agency（LLM06）、停止生成/停止传输、标识义务（显式/隐式）、留存 6 个月、备案、sse-starlette、StreamingResponse、text/event-stream、Last-Event-ID | ①毕业组装（终版清单+OS 架构图+交接单+"写到多大上什么组件"）→ P8·21；②模型旧名跨篇勘误——**已完成**（2026-10-16 全系列迁移 `deepseek-flash`，v1.0.6）；③真实落地（真 Go 后台对接）= 学完后 landing doc |
| P8·21 | ①定位=毕业两层（收口+升维）；②**毕业收口四件**：能力清单表（PMTA 四块+多体协作→各篇产物）、工程壳五件表（评测/trace/安全/治理/只读开关）、**选型记录模板**（路线/身份/形态：决定+理由+复核日期）、**交接单模板**（10 项含评测集生长纪律/owner 节奏）；③**Agentic OS**：分层 ASCII 图（应用/内核=harness/驱动=MCP/存储=记忆层次/权限层/生态）+ **八类比表**（进程=agent 实例/RAM=上下文/磁盘=长期记忆/设备驱动=MCP/内核调度=harness/fork=subagent/权限弹窗=y·c·n+只读开关/应用生态=注册表+MCP Registry）；④**平台治理五问表**（生命周期/可观测/配额/身份/跨 agent 通信→单 agent 版 vs 平台级，结论="增量几乎全在治理面"）；⑤**趋势表**（agentgateway 5.4k★=网关[LLM/MCP/A2A+Guardrails]/kagent 4k★=Agent 即 CRD/MCP Registry/Dify 159k★·Coze 22k★/**DeepSeek Harness（模型厂 agent 脚手架·开发者预览）**/Manus=action engine[官方未见 AgentOS 提法，以官方为准]/OpenHands·OpenClaw）+ 读趋势法（看接口在标准化什么）+ **落链接**；⑥**演化路径四档表**（1 人→小团队→多租户→平台；心法=组件跟着"多方"长+复杂度守恒律）；⑦**PMTA 闭环表**（四要素→平台构件，收束句："把 PMTA 从一个 agent 的零件清单养到一群 agent 的操作系统"）；⑧**毕业验收（动脑，不写代码）**：画 OS 图/写组件清单/写交接单 + **M1-M6 全景打勾表**（六里程碑+出处）；⑨全系列结语三件事（非确定性用工程接住/标准化处是共识处/组件跟着规模长）+ 出口（回「落地」开设计文档） | 应用层/内核/驱动层/存储层/权限层、agent 注册表、CRUD→CRD、A2A（趋势词，系列未展开）、action engine、平台治理五问、"写到多大上什么组件"、组件跟着多方长 | ——（全系列终篇；下一动作=读者确认后按需开"真实落地设计文档"，见 README 指针） |
|（后续每篇确认后追加一行） | | | |

## 状态流转规则

1. `未生成 → 待确认`：本篇写完 + 自检四问全过，**同一会话内立即落表**（先落表再求确认，防会话中断丢状态）；
2. `待确认 → 已确认`：读者明确确认/回复「继续」，指针推进下一篇；
3. `待确认 → 未生成`（回退）：读者否决，备注写原因后重写；
4. Part 里程碑：该 Part 末篇「已确认」→ 里程碑「待验证」→ 读者实际跑通 →「已验证」→ 才开下一 Part；
5. 临时增篇与提示词勘误一律记入对应行「备注」。知识台账每篇确认后追加一行。

## 生成与验收流程（上下文压缩后照此执行）

1. 开工先读：【当前指针】→【逐篇进度表】→【知识台账】→ [00-生成提示词.md](00-生成提示词.md)
   全篇（唯一事实源），**不要另外探索代码**。
2. **一次只生成一篇**：把提示词正文交给 AI，要求它按该篇在【系列结构】里的要点清单生成
   完整内容，写到【逐篇进度表】里该篇对应的 `docs/` 文件。
3. **每篇生成后立即检查**（不许攒批）：
   - 要点全覆盖？对照该篇 bullet 清单逐条打勾；
   - 引用有出处？网址/版本/参数不是编的，拿不准标 `<以官方 docs 为准>`；
   - 动手实践（若有）能否跑通？至少 mentally 完整；
   - 有缺漏 → 当场补齐再算过。
4. 读者确认 → 进度表该行状态改「已确认」→ 下一篇。
5. 一个 Part 完成 → 验证该 Part 里程碑可运行 → 里程碑行改「已验证」→ 读者确认 → 下一 Part。
6. 复述约定：每篇末尾按提示词要求"等我回复『继续』"——回复即信号。

## 2026 技术事实（2026-10-15 已核验，实测数据）

- **GitHub star**：langchain 14.8w / langgraph 4.3w / openai-agents 3.0w /
  pydantic-ai 2.0w / claude-agent-sdk 0.8w
- **PyPI**：langchain 1.4.6 / langgraph 1.2.15 / openai-agents 0.24.0 /
  claude-agent-sdk 0.2.167 / pydantic-ai-slim 2.56
- **目标服务脚手架（P3·8，2026-10-16 PyPI 实测）**：fastapi 0.143.2 / uvicorn 0.54.0 /
  pyjwt 2.15.1 / requests 2.34.2（本机实测跑通用 py3.9 解析版本 fastapi 0.128.8 等，
  接口行为一致）；官方文档 fastapi.tiangolo.com（实测 200）、pyjwt.readthedocs.io
- **LangChain 定位**：库大 ≠ 用它直接构建 agent；链式 API 已退出该路径，官方转向
  langgraph（选型篇专门回答）
- MCP（2026-10-16 实测）：规范 `2026-07-28`（specification/latest 实测指向）；PyPI
  包 `mcp` **2.3.0**；官方 SDK **10 语言**（TS/Python/Go/C#/Java/Kotlin/PHP/Ruby/
  Rust/Swift）；servers 仓库 **91.3k★**（7 个参考实现：Everything/Fetch/Filesystem/
  Git/Memory/Sequential Thinking/Time），早期示例转归档、改走 **MCP Registry**
  （registry.modelcontextprotocol.io）；python-sdk 24.6k★ / go-sdk 5.2k★
- **DeepSeek 模型换代（2026-10-16 核验，官方 changelog/定价页/Thinking Mode 页）**：
  现售 `deepseek-flash`（V4.1-Flash，2026-09-10 发布）与 `deepseek-v4-pro`（V4-Pro GA
  2026-08-13）；旧名 `deepseek-chat`/`deepseek-reasoner` **2026-07-24 停用**
  （v4-flash 系列旧名暂路由至 V4.1-Flash）；思维控制 = `thinking` +
  `reasoning_effort`（low/high/max 三档，**默认开启、默认 high**；请求值有官方
  映射如 medium→high；Anthropic 格式用 `{"reasoning": {"effort": ...}}`）；**三条
  配套规则**（带 tools 时历史 reasoning_content 必传否则 400 / 思考模式下
  temperature·presence_penalty·frequency_penalty 不生效 / top_p 仅思考模式生效且
  钳 0.95–1.0）；1M 上下文 / 384K 最大输出；并发 2500（flash）/ 500（v4-pro）；
  **峰谷定价**（peak=工作日 UTC 01–04 / 06–10，其余半价）；**缓存默认开启**（前缀
  命中，`prompt_cache_hit_tokens` / `prompt_cache_miss_tokens`；flash 命中
  $0.003/1M vs 未命中 $0.15/1M）；**token 换算口径**（1 英文≈0.3 / 1 中文≈0.6
  token，离线 tokenizer 可预估，精确数以 API usage 为准）；**DeepSeek Harness**
  开发者预览（Agent Integrations，官方列 Claude Code/Codex/OpenCode 等）。
  ✅ 全系列命名迁移已于 2026-10-16 完成（旧名仅存于勘误/迁移说明）
- **国内合规（2026-10-16 核验，cac.gov.cn 一手）**：《生成式人工智能服务管理暂行
  办法》2023-08-15 施行（第十一条：不得非法留存能识别使用者身份的输入/使用记录；
  第十四条：违法内容"停止生成/停止传输/消除"+报告）；《人工智能生成合成内容标识
  办法》2025-09-01 施行（显式+元数据隐式标识；第九条：特定场景日志留存≥6 个月）
- 向量库：pgvector（Postgres 原生，本项目优先）/ Qdrant / Milvus / Chroma 均在线
- harness 概念参考：Claude Code（本机）、OpenHands、OpenClaw、Cursor
- 联网工具说明：**本环境 WebSearch/WebFetch 被拦**，细节核验走 curl 直连官方源，
  或标注"以官方 docs 为准"

## 定稿说明（2026-10-15 已锁定；当日终检勘误 → v1.0.1）

- **规模**：21 篇 / 8 Part，作为基线定稿；生成中发现某知识点还需拆开，允许按
  "按知识点切篇"原则临时增篇（在进度表备注记一笔即可）。
- **前置环境清单**：并入第 1 篇（"动手前的环境准备"小节），不再单开一页。
- **v1.0.1 终检勘误（2026-10-15）**：速查表 memory 正确指 P2·3；里程碑锚点措辞放宽
  （加粗标注落在 Part 内最自然的一篇、末篇复验收口）+ 第 12 篇补 M3 标签；P1「无代码」
  措辞修正；第 4 篇 Claude 改有条件选做；第 19/20 篇权限话题分工（19 交互侧 / 20 架构侧）；
  知识地图 D4/C3 索引补全、第 2 篇标签改 PMTA。正文结构不变、不新增知识点。
- **v1.0.2 补充（2026-10-15）**：技术约束⑤ + P2·5 补 LlamaIndex（RAG/数据平台，5.3w star、
  PyPI 0.14.25）/ Semantic Kernel（微软系 .NET 优先，2.9w star、PyPI 1.45.0）一句话定位
  （已 curl 实测核验）；不进主线选择、不拆新篇，仅防读者撞见时困惑。
- **v1.0.3~v1.0.5（2026-10-16）**：v1.0.3 = MCP 主类更名口径同步（FastMCP→MCPServer，
  13/12 篇与 00 同步）；v1.0.4 = Go SDK 事实勘误（单一主包 mcp、v1.8.0、规范
  2026-07-28）；v1.0.5 = 第 21 篇生成时核验回写（Agentic OS 参考名单 + Manus
  定位勘正）。详见 00 篇尾版记。
- **v1.0.6（2026-10-16）**：**DeepSeek 命名统一迁移 + 知识缺口补全**——① 全系列
  （P1·1、P2·3~7、P4·11~12、P7·20、P8·21、00）旧名 `deepseek-chat`/`deepseek-reasoner`
  → `deepseek-flash`（V4.1-Flash 单模型 + thinking 两态），P2·4 整篇重写；
  ② 补齐五个知识缺口：reasoning_content 回传规则（带 tools 必传否则 400）、思考模式
  采样参数限制（temperature 失效 / top_p 0.95–1.0）、token 换算比例（中文≈0.6/字）、
  tool_calls 数组（并行调用）、DeepSeek Harness（模型厂 agent 脚手架预览）；
  ③ 00 升 v1.0.6，技术栈与模型层事实同步改写。旧名仅在勘误/迁移说明处作历史引用保留。
- 其余开放点已全部关闭（默认模型=DeepSeek、参考服务=FastAPI、SDK 盘点、记忆落库、
  输出规范、动手按需）。

## 外部链接备忘

官方源集中在 00-生成提示词.md 末尾"参考资料汇总"，本文件不放第二份。

**最后更新**：2026-10-16（**全系列 21 篇全部确认完毕** ✅；**DeepSeek 命名统一迁移完成**
（全系列 → `deepseek-flash`，00 升 v1.0.6）+ **知识缺口补全**（reasoning_content 回传
规则 / 思考模式采样参数 / token 换算 / tool_calls 数组 / DeepSeek Harness）；
待决事项已全部关闭——旧名勘误 ✅ 完成；磁盘版 `agent/` 代码不组装，**代码由读者照文档
自己一步步实现**；M1-M6 六项复验待读者实跑回执）
