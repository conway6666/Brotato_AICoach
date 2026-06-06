# ADR 0001: Deterministic Calculation First

## Status

Accepted

## Context

Brotato AI Coach 的首期目标是为大块头（Chunky）商店阶段提供可信建议。该场景高度依赖数值判断，包括 EHP、DPS、ROI、生命值转伤害以及 `%伤害` 对 Chunky 无效等机制。

如果让 LLM 直接进行关键数值推理，容易出现心算错误、规则遗忘和解释自洽但结论错误的问题。

## Decision

首期采用“确定性计算优先”的架构原则：

- Python 或等价确定性工具负责 EHP、DPS、ROI 和机制规则计算。
- LLM 只消费计算结果、风险标签和候选排序。
- LLM 负责解释、语气、压缩表达和追问回答，不负责关键数值心算。
- 任一推荐结论必须能追溯到计算输出或显式规则。

## Consequences

- 首期需要先定义稳定的数据输入输出结构和测试样例。
- 工程实现可以先用离线 mock 数据验证算法，再接入真实 Mod 数据。
- 后续多角色扩展应优先复用计算工具链，而不是为每个角色写纯 prompt 策略。
