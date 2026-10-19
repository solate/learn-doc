# 从零构建 Agent（让智能体能调用后台管理系统的接口）

> 一套 21 篇 / 8 Part 的**教学型学习文档**：先教会原理，再给可复现的 Python 代码。
> 读者定位：懂后端（Go 为主）、对 AI Agent 零基础的工程师。
> 技术栈：Python + DeepSeek（OpenAI 兼容）；从零手写 agent loop 起步，
> 需要封装时用 pydantic-ai / openai-agents，MCP 标准化用 FastMCP + 官方 Go SDK。

**学习主线**：先做出一个「能干活但有缺憾」的后台管理助手（P3 里程碑），
再用记忆/规划升级它（P4），再标准化为 MCP（P5）、多智能体化（P6）、工程化上生产（P7），
最后用 Agentic OS 视角看规模化（P8 毕业收口）。

> ⚠️ 本目录由 `00-生成提示词.md` 驱动生成，该文件是内部 AI 生成提示词，**不是教程正文**。
> 生成进度与续接台账见 [`progress.md`](progress.md)，单篇格式契约见 [`_contract.md`](_contract.md)。

## 目录

### Part 1 打开黑盒：Agent 到底是什么
- [x] [第 1 篇：全景与学习路线地图](docs/part1-打开黑盒/01-全景与学习路线地图.md)
- [x] [第 2 篇：经典智能体理论 → 现代 LLM Agent 映射](docs/part1-打开黑盒/02-经典理论映射.md)

### Part 2 亲手写出第一个 Agent
- [x] [第 3 篇：Agent 的心脏 — agent loop 与上下文管理](docs/part2-亲手写出第一个Agent/03-agent-loop与上下文.md)
- [x] [第 4 篇：模型选型与推理能力](docs/part2-亲手写出第一个Agent/04-模型选型与推理.md)
- [x] [第 5 篇：harness 框架怎么选（读完就有结论）](docs/part2-亲手写出第一个Agent/05-Python选型地图.md)
- [x] [第 6 篇：调用 LLM 的三种姿势](docs/part2-亲手写出第一个Agent/06-调用LLM的三种姿势.md)
- [x] [第 7 篇：手写一个「看得懂每行代码」的最小 agent（**里程碑 M1**）](docs/part2-亲手写出第一个Agent/07-手写最小agent.md)

### Part 3 接上真实系统：后台管理助手
- [x] [第 8 篇：把后台 API 变成 Agent 的工具 — 第一个闭环（**里程碑 M2**）](docs/part3-接上真实系统/08-后台API变成工具.md)
- [x] [第 9 篇：真实世界的工具工程设计](docs/part3-接上真实系统/09-工具工程设计.md)
- [x] [第 10 篇：工具全谱系与安全边界](docs/part3-接上真实系统/10-工具全谱系与安全边界.md)

### Part 4 让它更聪明：记忆与规划
- [x] [第 11 篇：记忆设计 — 三层记忆、压缩整理与存储后端](docs/part4-记忆与规划/11-记忆设计.md)
- [x] [第 12 篇：规划能力设计 — 复杂任务怎么被拆开、反思怎么纠正（**里程碑 M3**）](docs/part4-记忆与规划/12-规划能力设计.md)

### Part 5 标准化：接入 MCP 生态
- [x] [第 13 篇：MCP 是什么，为什么值得学](docs/part5-接入MCP生态/13-MCP是什么.md)
- [x] [第 14 篇：用 FastMCP 把管理助手工具升级为 MCP server](docs/part5-接入MCP生态/14-FastMCP包装.md)
- [x] [第 15 篇：直连 vs MCP — 选型与 Go 后台桥接（**里程碑 M4**）](docs/part5-接入MCP生态/15-直连vsMCP与Go桥接.md)

### Part 6 协同作战：借鉴成熟 Agent、落地多智能体
- [x] [第 16 篇：拆解 Claude Code / Cursor / OpenHands / OpenClaw](docs/part6-多智能体协同/16-拆解成熟Agent.md)
- [x] [第 17 篇：多智能体 — 模式、通信与实战（**里程碑 M5**）](docs/part6-多智能体协同/17-多智能体模式与实战.md)

### Part 7 支撑上线的工程底子
- [x] [第 18 篇：开发方法、评测、可观测与自改进边界](docs/part7-工程化上线/18-评测与可观测.md)
- [x] [第 19 篇：多轮对话、中断与人工介入](docs/part7-工程化上线/19-中断与人工介入.md)
- [x] [第 20 篇：生产化、安全与既有系统治理（**里程碑 M6**）](docs/part7-工程化上线/20-生产化与治理.md)

### Part 8 写大了怎么治 + 毕业收口
- [x] [第 21 篇：毕业项目与 Agentic OS（**毕业收口**）](docs/part8-毕业与AgenticOS/21-毕业项目与AgenticOS.md)

---

进度状态：**21 / 21 篇全部生成**（M1-M6 + 毕业三件交付物全达成，系列收官）（详见 [progress.md](progress.md)）
