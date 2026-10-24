# Agent 设计文档

**产品名称**: Globex 跨境选购 Agent  
**文档版本**: v1.0  
**日期**: 2026-10-25

---

## 1. 多 Agent 架构总览

Globex 采用「主 Agent + 专家子 Agent」的编排模式，由一个超级框总调度（MainAgent）根据任务性质决定自己直接执行，还是派发给专家子 Agent。

```
┌─────────────────────────────────────────────────────────────────┐
│                     用户输入（自然语言）                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              MainAgent（commerce_concierge）                     │
│                                                                 │
│  职责：理解意图 → 制定任务计划 → 单干或调度子 Agent → 整合回复   │
│                                                                 │
│  工具集 4 类：                                                   │
│  ① 业务工具（直接可调：搜索/下单/查单/取消/品类知识/联网）       │
│  ② 任务计划（TaskCreate/Update/List/Get）                        │
│  ③ 调度工具（task_dispatch → 派发子 Agent）                     │
│  ④ 记忆工具（remember/update/forget_preference）                │
└───────┬────────────────────────────────────────┬────────────────┘
        │  派发：满足"可并行/上下文隔离/调用链深"条件    │
        ▼                                        ▼
┌───────────────────┐                  ┌──────────────────────┐
│  SearchAgent       │                  │  TradeAgent           │
│  catalog_search    │                  │  order_trade_agent    │
│                   │                  │                       │
│  专注：商品检索    │                  │  专注：订单操作        │
│  工具：           │                  │  工具：               │
│  · product_search │                  │  · create_order_tool  │
│  · category_insight│                 │  · query_order_tool   │
│  · web_search      │                 │  · cancel_order_tool  │
│                   │                  │                       │
│  max_iters: 6     │                  │  max_iters: 6         │
└───────────────────┘                  └──────────────────────┘
```

---

## 2. 三个 Agent 角色详解

### 2.1 MainAgent（commerce_concierge）

**角色定位**：用户的唯一对话入口，跨境购物总指挥

**核心能力**：
- 听懂复合意图（"买个不含塑料、预算 800 发到上海的行李箱"）
- 判断是自己直接做，还是派发给专家子 Agent
- 管理多步任务计划（TaskCreate/Update）
- 维护长期记忆（偏好的写入/修改/撤回）
- 整合多个子 Agent 结果给出最终回复

**派发判断原则**（默认单干，只有满足以下条件才派发）：

| 触发条件 | 示例场景 |
|---------|---------|
| 可并行：多个彼此独立的子任务 | "比较行李箱和背包" → 同时派两个检索子任务 |
| 上下文隔离：子任务会产生大量中间数据 | 多轮迭代检索不污染主对话 |
| 调用链深：子任务内需多轮思考-调工具 | 复杂检索 + 品类知识组合查询 |

> 一次简单检索、一次下单/查单/取消 → 不派发，主 Agent 直接单干

**工具调用安全边界**：
- 记忆操作须经 AgentScope 原生审批（前端弹窗确认）
- 不能伪造商品信息（product_id 必须来自工具返回）
- 对话中的"确认"≠ 页面确认按钮，不执行订单写入

---

### 2.2 SearchAgent（catalog_search_agent）

**角色定位**：商品检索专家，接受 MainAgent 派发的检索子任务

**工作步骤**：

```
Step 1: （按需）查品类知识库
  → category_insight_tool
  → 获取：选购口径、属性判断方式、价格区间参考

Step 2: 改写查询词
  normalized_query = 标准化商品描述
  + 抽取槽位：
    - category（品类）
    - ship_to（收货国）
    - price_max_major（价格上限）
    - excluded_material_tags（排除材质标签）

Step 3: 召回商品
  → product_search_tool
  → Embedding 向量检索 + 可选 BM25 混合 + 硬约束过滤 + 精排

Step 4: 补召回（如果候选 < 3）
  → 改写扩展 query
  → 再调一次 product_search_tool

Step 5: 按偏好筛选
  dislike → 硬排除（命中的候选不进 hits）
  like    → 排序参考（加权但不强制排除）

Step 6: 返回 JSON
  {
    "hits": [...],         // 最终候选（含到手价）
    "filtered_out": [...], // 被约束挡掉的候选
    "notes": "..."         // 筛选说明
  }
```

**关键约束**：
- product_id / sku_id 必须来自工具真实返回，不得伪造
- dislike 命中的候选必须出现在 filtered_out（或 notes 说明），不能消失不说
- 价格/库存数字原样透传，不加工

---

### 2.3 TradeAgent（order_trade_agent）

**角色定位**：下单交易专家，接受 MainAgent 派发的交易子任务

**工作步骤**：

```
Step 1: （取消订单时）先查询
  → query_order_tool(order_id)
  → 确认状态为 CONFIRMED 才能继续取消

Step 2: 准备订单/取消确认单
  → create_order_tool / cancel_order_tool
  → 透传 product_id/sku_id，不替换商品
  → 金额来自工具返回，不自行计算

Step 3: 返回确认单
  → confirmation_required=true
  → 告知用户到页面完成确认

Step 4: 等待页面操作
  → 不代替用户执行
  → 页面按钮是唯一有效的确认路径
```

**安全护栏**：
- 模型文字同意 ≠ 确认按钮，在 notes 里明确告知
- 不承诺支付、物流、退款（本产品范围外）
- 错误信息原样透传，让 MainAgent 决策是否重试

---

## 3. 记忆系统设计

### 3.1 记忆分层

```
短期记忆（会话内）
└── AgentState（AgentScope 内置）
    - 对话历史、任务状态、中间工具结果
    - 服务重启后从 SQLite 恢复
    - 超长时自动压缩摘要，保留近期轮次

长期记忆（跨会话）
└── BuyerPreference（独立持久化）
    - 用户偏好条目（kind: like/dislike + statement）
    - LLM 语义提炼 + 向量化存储（SemanticPreferenceStore）
    - 每位用户最多 200 条
    - 每条有 memory_id + version（并发安全）
```

### 3.2 偏好注入机制

**核心设计决策：偏好不进 System Prompt**

```
原因：
System Prompt 是 Prompt Cache 的稳定前缀
如果把每个用户的个性化偏好写进 System Prompt
→ 每个用户的前缀都不同
→ Cache 命中率趋向于 0
→ Token 成本大幅上升
```

**实际方案**：偏好作为独立的 hint 消息注入

```
会话上下文结构：
[System Prompt]            ← 稳定，所有用户共用，Cache 友好
[<buyer-preferences> hint] ← 因人而异，作为独立消息
[对话历史]
[本轮用户输入]
```

hint 消息格式：
```xml
<buyer-preferences>
以下是该买家的长期偏好（来自历史会话，
like=喜好，dislike=忌口/黑名单）：
- [dislike] 不要塑料材质 (memory_id=abc123, version=2)
- [like] 喜欢日系简约风格 (memory_id=def456, version=1)
</buyer-preferences>
```

### 3.3 偏好注入策略

| 偏好类型 | 注入策略 | 原因 |
|---------|---------|------|
| **dislike（忌口/黑名单）** | 全量注入，不截断 | 漏掉一条 = 推了用户明确拒绝的东西 = 安全问题 |
| **like（正向偏好）** | 按本轮语义相关性排序，取 top_k（默认5）| 漏掉一条顶多推荐不够贴，可接受 |

**仅当偏好内容发生变化时才重新注入**，避免每轮重复填充上下文。

### 3.4 偏好写操作工具

| 工具 | 触发时机 | 关键约束 |
|-----|---------|---------|
| `remember_preference_tool` | 用户明确要记住长期偏好 | 需原生审批；一次性临时要求不记 |
| `update_preference_tool` | 用户明确要修改已有偏好 | 须原样引用 memory_id + expected_version |
| `forget_preference_tool` | 用户明确说某偏好不再适用 | 须原样引用 memory_id，不按语义相似猜测目标 |

---

## 4. 上下文管理设计

### 4.1 问题背景

跨境购物对话复杂度高：一个会话可能包含 20 次商品询问、5 次价格比较、多次偏好讨论。不加管理会导致：
- 模型"忘记"用户早期的核心需求
- Token 成本飙升
- 回复质量因窗口挤满而下降

### 4.2 上下文预算参数

```
CONTEXT_SIZE = 128,000 tokens      ← 模型窗口声明上限
TOOL_RESULT_LIMIT = 20,000 tokens  ← 单次工具返回截断上限
CONTEXT_TARGET_TOKENS = 48,000     ← 目标维持范围（不超过这个就不压缩）
CONTEXT_PRODUCT_TOKENS = 6,000     ← 商品候选预留空间
```

### 4.3 压缩策略（layered 模式）

```
触发条件：上下文超过 TARGET_TOKENS（pressure 时机）

压缩逻辑：
1. 识别"旧轮次"（距当前较远的完整交互）
2. 生成该轮次摘要（保留关键决策、商品选择、偏好信息）
3. 摘要替换原始消息，释放 token 空间
4. 近期 N 轮始终保留完整内容

不压缩的内容：
- dislike 偏好（安全底线，永不截断）
- 最近 N 轮完整消息
```

### 4.4 上下文变化通知

压缩发生后推送 `context.compressed` 事件给前端，让用户了解部分历史已摘要。

---

## 5. 安全护栏设计（Harness）

### 5.1 护栏层次

```
第一层：工具调用前（HarnessToolMiddleware）
  → SequencingTracker：确保调用顺序正确
    - 创建订单前必须有检索证据
    - 取消订单前必须先查询
  → LoopDetector：检测同参数重复调用
    - 超过阈值时拦截，防止死循环

第二层：工具调用层（ToolResilienceMiddleware）
  → CircuitBreaker：工具连续失败时熔断
  → Timeout：工具调用超时保护

第三层：输出层（output_guard - L4）
  → 最终回复推给用户前脱敏
  → 移除内部 ID、调试信息等敏感内容
  → 命中只脱敏不阻断（避免误判导致整轮失败）
```

### 5.2 6 条核心安全契约

以下由 Harness 自动化测试强制保证（0 误放行）：

```
契约 1: dislike 偏好命中商品 → 不出现在 hits，出现在 filtered_out
契约 2: 超价格约束商品 → 只在 filtered_out，附原因说明
契约 3: 工具返回 confirmation_required=true → Agent 不能说"已下单"
契约 4: 偏好写/改/删 → 必须等待原生审批，拒绝后不重试
契约 5: 工具未返回 [error] → 不能声称工具不可用
契约 6: product_id / sku_id → 必须来自工具返回，不凭空生成
```

---

## 6. 提示词工程设计

### 6.1 提示词架构原则

| 原则 | 说明 |
|------|------|
| System Prompt 最稳定 | 各用户共用，保护 Prompt Cache |
| 偏好作为 hint 消息 | 因人而异的内容不进 System Prompt |
| YAML 版本化管理 | 提示词变更可 diff、可回滚 |
| 修改触发评测 | 任何 Prompt 变更必须跑 Harness 评测 |

### 6.2 三个 Agent 的提示词分层

**MainAgent System Prompt 结构**：
```
[角色定位] 1段
[工具分类说明] 4类
  ├── ① 业务工具（各工具的使用条件和注意事项）
  ├── ② 计划工具（何时建，何时不建）
  ├── ③ 调度工具（单干 vs 派发的判断原则）
  └── ④ 记忆工具（触发条件 + 审批要求）
[工作约定] 10条行为规范
```

**SearchAgent / TradeAgent System Prompt 结构**：
```
[角色定位] 1段
[工作步骤] 5-6步（顺序执行）
[约束条件] 3-5条（安全边界）
```

### 6.3 提示词质量标准

| 维度 | 好的写法 | 坏的写法 |
|------|---------|---------|
| 工具使用条件 | "仅当候选 < 3 条时改写扩展再召回" | "尽量搜索更多商品" |
| 边界情况 | "工具未返回 [error] 时不得声称工具不可用" | "如果工具有问题就告知用户" |
| 格式要求 | "返回 JSON，hits 必须含 product_id/price" | "返回结构化结果" |
| 安全规则 | "再生尼龙仍是合成聚合物，不能标为无塑料" | "注意材质分类" |

---

## 7. 模型选型设计

### 7.1 不同组件的模型需求

| 组件 | 推荐模型类型 | 需要的能力 |
|-----|------------|----------|
| MainAgent / 子 Agent | 强推理模型（qwen3-max 量级） | 工具调用、多轮 ReAct 推理、JSON 输出 |
| Fallback 模型 | 轻量但可靠的模型（qwen-plus 量级） | 基本工具调用，超时/限流时快速兜底 |
| Embedding | 专用 Embedding 模型（text-embedding-v4） | 语义相似度、偏好召回 |
| Reranker | 专用精排模型（qwen3-rerank） | 二阶段排序，提升相关性 |

### 7.2 模型目录（Model Registry）

- 支持在页面配置模型档位（profile）
- 每个 profile 定义：base_url、api_key（加密存储）、model_name、context_size
- 用户选择模型后，下次对话生效
- 切换模型后，已配置的自动化任务须重新确认

### 7.3 Fallback 逻辑

```
主模型调用 → 超时 / 限流 → 切换 Fallback 模型重试
Fallback 也失败 → 整轮失败，告知用户稍后重试
```

---

## 8. 评测体系（Harness）

### 8.1 评测档位

| 档位 | 指令 | 使用场景 |
|-----|------|---------|
| dev | `make eval-harness` | 日常开发，每次修改相关组件后 |
| release | `make eval-harness-release` | 上线前，完整质量/安全/成本门禁 |
| CI | `.github/workflows/harness-eval.yml` | 每次 Push/PR 自动触发 |

### 8.2 评测指标

| 指标类型 | 具体指标 | 门禁要求 |
|---------|---------|---------|
| **安全** | dislike 误放行率 | 0 |
| **安全** | 价格约束违反率 | 0 |
| **安全** | 下单安全违规 | 0 |
| **效果** | Recall@10（召回率） | 待定基准 |
| **效果** | MRR（平均倒数排名） | 待定基准 |
| **成本** | 平均 Token 消耗 | 不超基线 × 1.1 |
| **延迟** | P95 端到端延迟 | 待定 |

### 8.3 评测要求

以下修改类型必须重新评测才能交付：

```
✓ 上下文治理策略修改
✓ System Prompt 内容修改
✓ 工具参数合同修改
✓ Skill / 记忆注入逻辑修改
✓ 循环保护参数调整
✓ 模型缓存策略调整
✓ Token 预算规则修改
```
