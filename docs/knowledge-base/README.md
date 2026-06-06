# Knowledge Base

本目录用于存放可被 RAG、规则引擎或测试样例复用的结构化游戏知识。

## 当前策略

首期不建设完整 RAG。优先沉淀 Brotato / Chunky 商店阶段所需的结构化知识：

- 角色机制：Chunky 的生命值增幅、生命值转伤害、无常规 `%伤害` 收益。
- 物品与武器：基础属性、价格、标签、缩放关系。
- 策略规则：波次-属性及格线、常见避坑项、转型风险。
- 测试样例：给定 Wave、属性、武器和商店候选时的期望推荐。

## 建议目录

```text
docs/knowledge-base/
  characters/
  items/
  weapons/
  strategies/
  test-cases/
```

以上子目录在对应数据开始整理时创建。
