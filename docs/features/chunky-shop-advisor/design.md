# Feature Design

## 1. 输出文件

`docs/features/chunky-shop-advisor/design.md`

## 2. 设计目标

| 项目 | 内容 |
| --- | --- |
| feature-slug | `chunky-shop-advisor` |
| 目标类型 | feature |
| 需求来源 | `docs/features/chunky-shop-advisor/requirements.md` |
| 来源 Roadmap Phase | Phase 0-1：验证 Chunky 数值决策，并完成单角色商店阶段 MVP |
| 来源 PRD 章节 | MVP 范围、核心用户场景、产品原则、数据与知识需求 |
| design-depth | `ai-rag` |

`design-depth = ai-rag` 的含义是：本 feature 涉及 LLM provider、Prompt contract 和降级策略；但首期不建设完整 RAG，不让 LLM 负责排序或数值计算。

## 3. 需求摘要

- 进入商店或升级属性选择界面时，默认自动生成非阻塞建议。
- 对当前可见商品和升级属性输出排序、简短理由、购买/跳过/选择倾向和保守刷新建议。
- 建议必须基于最新 Wave、金币、角色属性、武器、道具、候选项和玩家配置重新计算。
- Chunky 的生命值收益、生命值转伤害和常规 `%伤害` 低收益规则必须稳定命中。
- 默认输出控制在 3 行以内；LLM 超时或不可用时必须有确定性降级输出。
- 不自动购买、自动加点、自动刷新，不输出唯一最优构筑路线。

## 4. 确定性标注

| 类型 | 内容 | 来源/状态 |
| --- | --- | --- |
| confirmed | 采用 offline-first：先用离线 JSON、确定性计算器和 golden cases 验证排序，再接 Brotato Mod 数据。 | 用户确认 |
| confirmed | 首批数据策略为本地游戏内容优先，社区资料交叉验证。当前机器无法访问游戏安装目录，抽取后续另开会话验证。 | 用户确认 |
| confirmed | DeepSeek/LLM 是可选解释和追问层，不参与核心排序。 | 用户确认 |
| confirmed | 默认排序目标采用 Wave 动态目标，经济、生命值、EHP、DPS 和风险缓解权重随局势切换。 | 用户确认 |
| from requirements | 自动触发、非阻塞 UI、3 行以内建议、缺失数据降级、低置信标注。 | `requirements.md` |
| from requirements | 需要覆盖商店商品、升级属性、刷新建议和动态修正。 | `requirements.md` |
| from requirements | 不做自动操作、多角色、战斗实时提示、完整 Wiki/RAG。 | `requirements.md` |
| from requirements | 关键判断必须来自确定性计算或显式规则，LLM 不负责心算。 | `docs/decisions/0001-deterministic-calculation-first.md` |
| assumed default | Python 作为 Phase 0 计算器、数据校验、测试和 LLM 编排语言。 | 安全默认 |
| assumed default | DeepSeek 使用 OpenAI-compatible SDK，通过 `base_url`、`model`、`api_key` 配置。 | 安全默认 |
| needs validation | Brotato Mod Hook、UI 注入、安装目录数据结构、打包方式。 | 需要能访问游戏安装和 Mod 环境 |
| needs validation | Chunky 精确公式、物品加成叠加顺序、武器缩放和价格数据。 | 需要游戏文件或可信资料核对 |
| blocked | 无阻塞项；未验证内容先进入 assumptions/open questions，不阻塞设计文档。 | 当前状态 |

## 5. 模块与文件影响

| 模块/文件 | 操作 | 职责 | 不负责 |
| --- | --- | --- | --- |
| `docs/features/chunky-shop-advisor/design.md` | create | 当前 feature 技术设计、契约、流程、未决项。 | 业务实现代码。 |
| `docs/knowledge-base/characters/chunky.json` | future create | Chunky 机制、修正系数、规则标签、来源元数据。 | 运行时爬虫。 |
| `docs/knowledge-base/items/*.json` | future create | 物品属性变化、价格、标签、限制、来源元数据。 | 商店实时状态采集。 |
| `docs/knowledge-base/weapons/*.json` | future create | 武器基础伤害、攻速、缩放属性、来源元数据。 | UI 展示。 |
| `docs/knowledge-base/strategies/chunky-shop-rules.json` | future create | Wave 动态权重、刷新阈值、避坑规则。 | LLM Prompt 话术。 |
| `docs/knowledge-base/test-cases/*.json` | future create | golden cases：输入局面、期望排序、期望标签。 | 替代真实游戏验证。 |
| `src/calculator/` 或等价包 | future create | EHP/DPS/ROI、候选评分、刷新建议、置信度和模板文案。 | 调用 LLM、展示 UI。 |
| `src/advice/` 或等价包 | future create | 组装最终建议、调用可选 LLM、处理超时和 fallback。 | 修改计算排序。 |
| `src/service/` 或等价包 | future create | Phase 1 本地 HTTP/IPC 服务，接收 Mod 状态并返回建议。 | 直接读取游戏内存。 |
| Brotato Mod 文件 | future create | Phase 1 触发、采集当前可见状态、展示非阻塞 UI。 | 阻塞主线程、做复杂排序、调用未降级的外部 API。 |
| `docs/prd.md` / `docs/roadmap.md` | future update | 同步“默认自动触发，可关闭后按钮触发”。 | 当前 feature 详细契约。 |

## 6. 数据契约

所有跨模块契约必须包含 `schema_version`。Phase 0 使用离线 JSON；Phase 1 由 Mod 或本地服务传入同构结构。

### 6.1 Input Contract

```json
{
  "schema_version": "1.0",
  "request_id": "shop-2026-0001",
  "trigger": "shop_entered",
  "game": {
    "name": "Brotato",
    "version": "needs_validation"
  },
  "context": {
    "wave": 8,
    "scene": "shop",
    "gold": 65,
    "reroll_cost": 12
  },
  "player": {
    "character": "chunky",
    "stats": {
      "max_hp": 70,
      "armor": 5,
      "dodge": 10,
      "damage_percent": 0,
      "attack_speed": -10,
      "range": 0,
      "crit_chance": 0,
      "life_steal": 0,
      "hp_regeneration": 3,
      "harvesting": 20
    },
    "weapons": [
      {
        "id": "needs_validation",
        "name": "needs_validation",
        "tier": 1,
        "count": 1
      }
    ],
    "items": [
      {
        "id": "needs_validation",
        "name": "needs_validation",
        "count": 1
      }
    ]
  },
  "candidates": {
    "shop_items": [
      {
        "candidate_id": "slot-1",
        "item_id": "needs_validation",
        "name": "+HP Item",
        "price": 25,
        "locked": false,
        "stat_delta": {
          "max_hp": 6
        },
        "tags": ["hp"]
      }
    ],
    "level_up_options": [
      {
        "candidate_id": "level-1",
        "stat": "max_hp",
        "value": 6,
        "rarity": "needs_validation"
      }
    ]
  },
  "config": {
    "auto_prompt_enabled": true,
    "advice_depth": "L2",
    "llm_enabled": false,
    "locale": "zh-CN"
  }
}
```

`trigger` allowed values:

| Value | 含义 |
| --- | --- |
| `shop_entered` | 进入商店界面。 |
| `shop_changed` | 商店刷新、锁定或候选变化。 |
| `level_up_entered` | 进入升级属性选择界面。 |
| `manual_request` | 自动提示关闭后，玩家按钮触发。 |

`scene` allowed values: `shop`, `level_up`。

### 6.2 Output Contract

```json
{
  "schema_version": "1.0",
  "request_id": "shop-2026-0001",
  "status": "ok",
  "confidence": "medium",
  "rankings": [
    {
      "candidate_id": "slot-1",
      "rank": 1,
      "label": "strong_buy",
      "score": 82.5,
      "score_breakdown": {
        "ehp_gain": 18.0,
        "dps_gain": 14.0,
        "economy_value": 0.0,
        "risk_mitigation": 12.0,
        "chunky_synergy": 30.0,
        "price_efficiency": 8.5
      },
      "reason_codes": ["chunky_hp_double_value", "ehp_gain", "acceptable_price"],
      "short_reason": "+HP 同时提升坦度和 Chunky 隐性伤害。",
      "source": "deterministic"
    }
  ],
  "reroll_advice": {
    "label": "do_not_reroll",
    "reason": "当前已有高价值候选，刷新会降低保留金币空间。"
  },
  "ui_message": {
    "lines": [
      "优先：+HP 道具；跳过纯 %伤害。",
      "+HP 同时提升坦度和 Chunky 隐性伤害。",
      "不建议刷新：当前已有可买核心项。"
    ],
    "expand_available": true
  },
  "warnings": [],
  "llm": {
    "used": false,
    "fallback_reason": "disabled"
  }
}
```

`status` allowed values:

| Value | 行为 |
| --- | --- |
| `ok` | 输出排序和 UI 建议。 |
| `partial` | 部分属性缺失；只输出低置信建议和 warning。 |
| `insufficient_data` | 缺少候选项或关键状态；不输出确定排序。 |
| `error` | 输入结构非法或计算失败；返回错误信息，不阻塞游戏。 |

`label` allowed values:

| Value | 含义 |
| --- | --- |
| `strong_buy` / `strong_pick` | 强推荐购买或选择。 |
| `optional` | 可选，取决于构筑方向或金币空间。 |
| `skip` | 当前低价值或负收益，建议跳过。 |
| `avoid` | 对 Chunky 明显低收益或风险较高。 |
| `unknown` | 规则无法覆盖，不编造结论。 |

## 7. 关键流程

```text
Trigger -> Collect/Load State -> Validate Contract -> Deterministic Scoring
  -> Rank Candidates -> Conservative Reroll Decision -> Template Advice
  -> Optional LLM Rewrite/Follow-up -> Non-blocking UI
```

### 7.1 正常路径

1. Phase 0 从 golden case JSON 读取输入；Phase 1 由 Mod 在商店或升级界面触发状态采集。
2. 校验 `schema_version`、`scene`、候选项、金币、Wave、角色和关键属性。
3. 根据 `character = chunky` 加载 Chunky 机制规则和候选项结构化数据。
4. 确定性计算器计算每个候选的 EHP、DPS、ROI、Chunky 协同、价格效率和风险缓解。
5. Wave 动态排序器调整权重并生成 `rankings`、`reason_codes`、`confidence`。
6. 刷新策略只在候选整体价值明显偏低、金币足够且不会牺牲关键购买空间时输出“考虑刷新”。
7. 无 LLM 时用模板生成 3 行以内建议；有 LLM 且未超时时，只让 LLM 改写和回答追问。
8. Mod UI 展示非阻塞气泡，不遮挡商店商品、升级选项、金币和关键属性。

### 7.2 错误/降级路径

| 场景 | 行为 |
| --- | --- |
| 缺少候选商品或升级选项 | 返回 `insufficient_data`，UI 提示“当前候选项读取不完整”，不输出排序。 |
| 缺少部分角色属性 | 返回 `partial`，只使用可用字段，降低 `confidence` 并列出 `warnings`。 |
| 物品或武器无结构化数据 | 候选项标为 `unknown`，不编造 EHP/DPS/ROI。 |
| LLM 超时、无 API Key 或解析失败 | 使用确定性模板文案；`llm.used = false` 并记录 `fallback_reason`。 |
| 本地服务不可用 | Mod 不阻塞游戏，展示“建议服务暂不可用，可继续游戏”。 |
| 商店价值不明确 | 默认不建议刷新，返回“当前没有明显刷新理由”。 |

## 8. 技术决策与 Trade-off

| 决策点 | 选项 | 推荐方案 | 状态 | 原因 |
| --- | --- | --- | --- | --- |
| MVP 接入顺序 | 直接接 Mod / offline-first | offline-first 后接 Mod | confirmed | 先验证公式和排序，隔离 Mod Hook/UI 风险。 |
| 数据来源 | 游戏文件 / 社区网站 / 运行时爬虫 | 游戏文件优先，社区交叉验证，最终结构化 JSON | confirmed | 可复核、可测试、避免运行时依赖网站。 |
| 社区资料 | 用户大量收集 / 少量核对 / 不使用 | 少量可信资料用于交叉验证 | confirmed | 不把资料收集变成前置重负担。 |
| LLM 职责 | 排序 / 解释 / 不接入 | 可选解释和追问层 | confirmed | 遵守 ADR 0001，避免 LLM 数值幻觉。 |
| DeepSeek 调用 | 专用 SDK / OpenAI-compatible SDK | OpenAI-compatible SDK + provider config | assumed default | 降低替换成本，便于配置 base_url/model/api_key。 |
| 排序目标 | 固定生存 / 固定输出 / Wave 动态 | Wave 动态 | confirmed | Chunky 商店价值受波次、金币、短板和候选质量影响。 |
| Mod 职责 | 采集+计算+LLM / 轻量采集展示 | 轻量采集和展示 | assumed default | 避免阻塞游戏主线程，降低崩溃风险。 |
| 打包方式 | 现在固定 / 后续验证 | 后续验证后写实 | needs validation | 当前机器无游戏目录，不能把未知 Brotato Mod 打包方式写成事实。 |

## 9. 可执行实现说明

### 9.1 伪代码/算法

```text
function advise(input):
  validation = validate_input_schema(input)
  if validation.missing_candidates:
    return insufficient_data("当前候选项读取不完整")

  rules = load_character_rules("chunky")
  knowledge = load_structured_knowledge(input.candidates)

  scored = []
  for candidate in all_candidates(input):
    if knowledge.missing(candidate):
      scored.append(unknown_candidate(candidate))
      continue

    deltas = apply_candidate_delta(input.player, candidate)
    metrics = {
      ehp_gain: calculate_ehp_delta(input.player.stats, deltas),
      dps_gain: calculate_dps_delta(input.player, deltas, rules),
      roi: calculate_roi(candidate.price, deltas),
      chunky_synergy: calculate_chunky_synergy(candidate, deltas, rules),
      risk_mitigation: calculate_wave_risk_delta(input.context.wave, deltas)
    }
    weights = wave_dynamic_weights(input.context, input.player, candidate)
    score = weighted_sum(metrics, weights)
    scored.append(build_ranking_entry(candidate, score, metrics))

  rankings = sort_desc(scored)
  reroll = conservative_reroll(rankings, input.context.gold, input.context.reroll_cost)
  template = render_template(rankings, reroll, max_lines=3)

  if input.config.llm_enabled:
    llm_result = rewrite_with_llm(template, rankings, timeout_ms=config.llm.timeout_ms)
    if llm_result.valid:
      return attach_llm_message(rankings, reroll, llm_result)

  return deterministic_output(rankings, reroll, template)
```

### 9.2 Wave 动态排序原则

| 因素 | 排序影响 | 状态 |
| --- | --- | --- |
| Wave 较早且经济不足 | 提高收获、价格效率、核心 HP 的权重。 | assumed default |
| 当前 EHP 低于及格线 | 提高 HP、护甲、闪避、回复、风险缓解权重。 | assumed default, needs validation |
| 当前清怪能力不足 | 提高有效 DPS、武器协同、攻速/范围等实际输出权重。 | assumed default, needs validation |
| 候选包含纯 `%伤害` | 对 Chunky 降低优先级或标记低价值。 | from requirements |
| 候选存在生命值收益 | 提高 Chunky 协同分，因为同时影响坦度和隐性伤害。 | from requirements, formula needs validation |
| 金币不足或有高价值候选 | 默认不建议刷新。 | from requirements |
| 候选整体低价值且金币仍安全 | 才输出“考虑刷新”。 | from requirements, threshold needs validation |

### 9.3 配置项

| 配置 | 默认值 | 状态 | 说明 |
| --- | --- | --- | --- |
| `advisor.mode` | `offline` | assumed default | Phase 0 使用离线 JSON；Phase 1 可切换 `service`。 |
| `advisor.max_lines` | `3` | from requirements | 默认建议不超过 3 行。 |
| `advisor.auto_prompt_enabled` | `true` | from requirements | 默认自动触发，可关闭后按钮触发。 |
| `advisor.confidence_threshold` | `medium` | assumed default | 低于阈值时展示低置信或 unknown。 |
| `advisor.reroll_policy` | `conservative` | from requirements | 不追逐未知最优解。 |
| `llm.enabled` | `false` | confirmed | DeepSeek 是可选增强。 |
| `llm.provider` | `deepseek_openai_compatible` | assumed default | 使用 OpenAI-compatible SDK。 |
| `llm.base_url` | env/config | assumed default | 不写死，避免泄漏 API Key 或 provider 绑定。 |
| `llm.model` | env/config | assumed default | 由用户 DeepSeek Pro 可用模型决定。 |
| `llm.temperature` | `0.2` | assumed default | 建议解释偏确定，避免创作性扩写。 |
| `llm.top_p` | `0.9` | assumed default | 保持表达自然但不过度发散。 |
| `llm.timeout_ms` | `1500` | assumed default | 超时走模板 fallback，避免阻塞。 |

## 10. 条件章节：LLM/RAG 设计

当前 feature 需要 LLM 设计，但不建设完整 RAG。检索层只使用本地结构化 JSON 和已计算结果；不接向量数据库，不运行时爬网页。

### 10.1 Prompt Contract

| 项目 | 内容 |
| --- | --- |
| system/developer 指令 | 你是 Brotato Chunky 商店建议的表达层。只能基于给定 deterministic_result 改写，不得新增排序、不得心算、不得声称唯一最优。 |
| 输入变量 | `locale`、`scene`、`wave`、`gold`、`rankings`、`reroll_advice`、`warnings`、`advice_depth`。 |
| 禁止内容 | 不自动操作；不输出完整构筑路线；不覆盖 deterministic rank；不编造未知物品效果；不输出超过 3 行默认建议。 |
| 输出 JSON schema | `{ "lines": ["string"], "follow_up_suggestions": ["string"], "notes": ["string"] }` |
| 失败输出 | 解析失败、超时或违反 schema 时丢弃 LLM 输出，使用 deterministic template。 |

LLM 输入示例：

```json
{
  "schema_version": "1.0",
  "locale": "zh-CN",
  "scene": "shop",
  "wave": 8,
  "gold": 65,
  "deterministic_result": {
    "top_rankings": [
      {
        "name": "+HP Item",
        "label": "strong_buy",
        "reason_codes": ["chunky_hp_double_value", "ehp_gain"],
        "short_reason": "+HP 同时提升坦度和 Chunky 隐性伤害。"
      }
    ],
    "reroll_advice": {
      "label": "do_not_reroll",
      "reason": "当前已有高价值候选。"
    },
    "warnings": []
  }
}
```

LLM 输出 schema：

```json
{
  "schema_version": "1.0",
  "lines": [
    "优先买 +HP；跳过纯 %伤害。",
    "+HP 同时提升坦度和 Chunky 隐性伤害。",
    "不建议刷新：当前已有可买核心项。"
  ],
  "follow_up_suggestions": [
    "为什么纯 %伤害低收益？",
    "我想转攻速可以吗？"
  ],
  "notes": []
}
```

### 10.2 Model Parameters

| 参数 | 默认值 | 状态 | 原因 |
| --- | --- | --- | --- |
| model | config/env | assumed default | 由 DeepSeek Pro 当前可用模型决定，不能写死在代码中。 |
| temperature | `0.2` | assumed default | 低创造性，避免改变结论。 |
| top_p | `0.9` | assumed default | 允许自然表达但限制发散。 |
| max_output_tokens | `300` | assumed default | 默认 3 行建议和少量追问。 |
| timeout_ms | `1500` | assumed default | 超时 fallback，不阻塞游戏体验。 |

### 10.3 Retrieval and Evaluation

| 项目 | 设计 |
| --- | --- |
| 检索来源 | 本地结构化 JSON：characters、items、weapons、strategies、test-cases。 |
| top_k | 不适用：首期按候选 ID 和规则直接查表，不做向量检索。 |
| 过滤 | 只允许当前可见候选、当前角色、当前 Wave 相关规则进入 LLM 上下文。 |
| 重排/去重 | 排序已由确定性计算器完成；LLM 不重排。 |
| 无结果行为 | 候选标为 `unknown` 或低置信，不编造。 |
| 引用策略 | `reason_codes` 必须可追溯到公式、规则或结构化数据来源。 |
| 评估样例 | 每个 golden case 验证 top ranking、avoid 标签、reroll 标签和 3 行输出。 |
| 质量门槛 | LLM 输出不得改变 deterministic label/rank；违反则丢弃。 |

## 11. 性能、稳定性与降级

| 场景 | 目标/限制 | 降级策略 | 验证方式 |
| --- | --- | --- | --- |
| Phase 0 离线计算 | 单个 JSON case 可快速返回排序。 | 失败时输出结构化错误。 | 单元测试和 golden case。 |
| Phase 1 Mod 触发 | 不阻塞游戏主线程。 | 异步请求；服务不可用时显示轻提示。 | 手动商店触发测试。 |
| LLM 调用 | 不影响排序输出。 | 超时或失败使用模板文案。 | 模拟无 API Key、超时和非法 JSON。 |
| 数据缺失 | 不输出确定排序。 | `insufficient_data` 或 `partial`。 | 缺字段 fixtures。 |
| 物品规则未知 | 不编造收益。 | `unknown`、低置信、提示需人工确认。 | 未知 item fixture。 |
| 刷新建议 | 保守，不鼓励追逐未知最优。 | 默认 `do_not_reroll`。 | AC-3/AC-4 golden cases。 |

## 12. Feature 实现切片

| Slice | 目标 | 修改模块 | 验证方式 |
| --- | --- | --- | --- |
| 0 | 文档和契约闭环 | `design.md`、后续知识库 schema | 文档检查：契约含 `schema_version`，未决项明确标注。 |
| 1 | 离线数据和 golden cases | `docs/knowledge-base/**` | 样例覆盖 Chunky HP、纯 `%伤害`、刷新/不刷新。 |
| 2 | 确定性计算器 | `src/calculator/` | 单元测试 EHP/DPS/ROI、排序标签、低置信。 |
| 3 | 模板建议输出 | `src/advice/` | 无 LLM 情况下返回 3 行以内建议。 |
| 4 | 可选 DeepSeek 解释层 | `src/advice/llm_*` | 无 API Key、超时、非法 JSON 均 fallback。 |
| 5 | 本地服务契约 | `src/service/` | HTTP/IPC contract test。 |
| 6 | Brotato Mod 接入 | Brotato Mod 文件 | 在真实商店/升级界面触发，不遮挡 UI，不阻塞游戏。 |

## 13. Assumptions / Open Questions / Validation Needed

| 类型 | 内容 | 影响 | 处理方式 |
| --- | --- | --- | --- |
| assumption | Phase 0 使用 Python、JSON fixtures 和测试命令验证。 | 决定首批依赖和打包形态。 | 后续 `game-engineer` 按 Python 项目初始化。 |
| assumption | DeepSeek 通过 OpenAI-compatible SDK 调用。 | 降低 provider 绑定。 | 在配置中暴露 `base_url`、`model`、`api_key`。 |
| assumption | 社区资料只做交叉验证，不作为运行时依赖。 | 避免实时爬虫和不稳定依赖。 | 把数据落入结构化 JSON。 |
| open question | Brotato Mod 具体 Hook、UI 注入和打包方式。 | 影响 Phase 1 工具链和发布包结构。 | 等可访问游戏安装目录或 Mod Loader 信息时验证。 |
| open question | Chunky 精确公式、加成叠加顺序、武器缩放和商店价格数据。 | 影响计算准确性。 | 本地游戏文件优先，社区资料和截图交叉验证。 |
| open question | Wave 动态权重和刷新阈值的具体数值。 | 影响排序口味和 AC-3/AC-4。 | 先用保守默认，再用 golden cases 调参。 |
| validation needed | 当前机器无 Brotato 安装目录。 | 不能现在验证游戏文件抽取。 | 后续另开会话在有游戏文件环境中验证。 |
| validation needed | PRD/Roadmap 与 requirements 的触发方式不一致。 | 上游文档可能过时。 | 交给 `game-doc-helper` 同步。 |
| validation needed | 数据来源版权和复用边界。 | 影响是否可提交完整数据。 | 记录 source metadata，避免复制不可复用内容。 |

## 14. 交给 game-engineer 的实现约束

- 排序、购买/跳过倾向、刷新建议必须由确定性计算器输出；不能让 LLM 重排或心算。
- 任一跨模块输入/输出必须包含 `schema_version`。
- 无 DeepSeek API Key、LLM 超时或 LLM 输出非法时，仍必须返回模板建议或结构化降级状态。
- 不得在运行时爬网站作为主数据源；外部资料必须先转为可复核 JSON 和测试样例。
- Mod 层不得阻塞游戏主线程，不得自动购买、自动加点或自动刷新。
- 缺少候选数据时不得输出确定排序；未知物品不得编造收益。
- 每个 golden case 至少断言：top candidate、avoid/skip 标签、reroll label、warnings、默认 UI 行数。
- DeepSeek API Key 只能从环境变量或本地配置读取，不得提交到仓库。
- Phase 1 Brotato Mod 打包方式在验证前不能写死；实现前必须确认游戏版本、Mod Loader 和安装目录结构。

## 15. 需要同步到上游文档或 ADR 的结论

- `docs/prd.md`：核心用户场景仍偏“玩家按下快捷键”，需同步为“默认自动触发，关闭自动提示后按钮触发”。
- `docs/roadmap.md`：Phase 1 验证方式需同步同上。
- `docs/changelog.md`：记录新增 `chunky-shop-advisor` design。
- 可选 ADR：如果后续确认 offline-first 是项目级原则，可新增“Offline Golden Case Before Mod Hook” ADR。
- `docs/knowledge-base/README.md`：后续开始整理数据时，补充 source metadata、golden case 和数据验证规范。
