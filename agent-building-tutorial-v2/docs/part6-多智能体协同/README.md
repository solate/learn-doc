# Part 6 协同作战：借鉴成熟 Agent、落地多智能体（第 16-17 篇）

**目标**：先看真实系统怎么设计协作，再抽象模式落到自己的 demo。
**里程碑 M5**：主 agent + 只读查询 subagent 分工跑通，能量化 token/延迟对比。✅

| 篇 | 标题 | 状态 |
|---|---|---|
| [16](16-拆解成熟Agent.md) | 拆解 Claude Code / Cursor / OpenHands / OpenClaw ⭐ 实例先行 | ✅ |
| [17](17-多智能体模式与实战.md) | 多智能体 — 模式、通信与实战 ⭐ 里程碑 M5 | ✅ |

**Part 6 里程碑 M5 自查**：`demo_m5_compare.py` 跑通——同一任务两版终局一致，
主上下文峰值 14→6 条消息，总 LLM 调用 7→9 次（成本账），写操作闸门零丢失。

配套代码：[`agent/subagent.py`](../../agent/subagent.py)、[`agent/hooks.py`](../../agent/hooks.py)（checkpoint/--resume 见 agent.py）

[← 返回系列目录](../../README.md) · [上一 Part：接入 MCP 生态](../part5-接入MCP生态/README.md) · [下一 Part：工程化上线 →](../part7-工程化上线/README.md)
