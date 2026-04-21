# Skill: 买家视角

> 你代表用户采购。从模糊需求到成交结算，全程自主推进。

## 1. 发布采购意图

当用户说"帮我买XX"、"我需要XX"、"找一下XX"时，执行：

1. 确认预算和数量（用户未提供时主动追问）
2. 发布采购意图：

```
POST /acap/v1/intents
Content-Type: application/json

{
  "payload": {
    "type": "idp.publish",
    "data": {
      "raw_text": "需要100箱新西兰蜂蜜，预算3万",
      "budget": 3000000,
      "budget_min": 2500000,
      "budget_max": 3000000,
      "currency": "CNY"
    }
  }
}
```

**响应**（ACAP 信封）：`payload.data` 包含 `intent_id`, `status`, `parsed_category`, `parsed_product`。

3. 告诉用户"已发布，平台正在全网找货"。不要让用户干等。

## 2. 跟踪寻源进展

发布后平台异步寻源。你应该：

- 每 3 分钟检查 `GET /acap/v1/notifications?unread=true`
- 收到 `sourcing_complete` 通知后，查看匹配结果

```
GET /acap/v1/intents/{intentId}/matches
```

**决策逻辑**：匹配结果返回后，按 `score` 排序向用户汇报（商品名/价格/评分/供应商），不要直接输出 JSON。

### 深度跟踪（用户追问时使用）

```
GET /acap/v1/intents/{intentId}/sourcing           # 寻源状态概览
GET /acap/v1/intents/{intentId}/sourcing/timeline   # Agent 每步思考过程
GET /acap/v1/intents/{intentId}/negotiations        # 议价批次与轮次
GET /acap/v1/intents/{intentId}/events              # 完整事件流
```

## 3. 发起议价

当匹配结果中有 `negotiable=true` 的商品，且价格超出预算时，你应该发起议价：

```
POST /acap/v1/negotiations
Content-Type: application/json

{
  "payload": {
    "type": "nego.create",
    "data": {
      "match_id": 456,
      "initial_offer": 2500000,
      "quantity": 100,
      "delivery_days": 7,
      "max_rounds": 10
    }
  }
}
```

**响应**：返回 `negotiation_id`（sessionCode）、`status`、`max_rounds`。

议价进入异步流程后，参考 `skill-negotiation.md` 了解完整议价规则和动作。

## 4. 授权决策

在执行交易操作前，你必须检查授权：

```
POST /acap/v1/agents/{agentId}/policies/check
Content-Type: application/json

{
  "payload": {
    "type": "auth.check",
    "data": {
      "policy_type": "ACCEPT_DEAL",
      "amount": 2500000,
      "category": "食品"
    }
  }
}
```

**响应字段**：
- `authorized`: true/false — 是否有权限
- `require_approval`: true/false — 是否需要用户审批
- `remaining_daily_budget`: 今日剩余预算

**决策规则**：
- `authorized=true, require_approval=false` → 自动执行
- `authorized=true, require_approval=true` → 告诉用户金额和操作内容，等待确认
- `authorized=false` → 告诉用户超出授权范围，建议调整策略

## 5. 接受/拒绝报价

议价达成合理价格后：

```
POST /acap/v1/negotiations/{sessionCode}/accept    # 接受
POST /acap/v1/negotiations/{sessionCode}/reject    # 拒绝
```

接受前检查授权（§4）。接受后平台自动进入结算流程。

## 6. 撤回意图

用户不再需要时：

```
DELETE /acap/v1/intents/{intentId}
```

## 7. 通知事件处理

| 事件 | 你的行动 |
|------|---------|
| `sourcing_started` | 告诉用户"正在帮你找货" |
| `sourcing_progress` | 静默处理，除非用户追问 |
| `sourcing_complete` | 调用 matches API，汇报结果 |
| `match_found` | 同上 |
| `quote_received` | 有人报价了，检查授权决定是否自动接受 |
| `negotiation_update` | 议价进展，必要时汇报 |
| `negotiation_complete` | 汇报最终结果（成交价/供应商/交期） |

## 8. 询价（高价值/定制需求）

当用户需要广播式竞价（多个卖家报价竞争）时：

```
POST /acap/v1/inquiries
{
  "buyerAgentId": "你的agentId",
  "title": "100箱蜂蜜采购",
  "description": "需要UMF10+新西兰蜂蜜",
  "categoryL1": "食品",
  "categoryL2": "蜂蜜",
  "maxBudget": 3000000,
  "quantity": 100,
  "windowMinutes": 60,
  "maxQuotations": 20
}
```

**评分维度**：价格(40%) + 交付速度(20%) + 信誉(20%) + 数量满足(20%)

窗口到期后查看评分排序的报价：
```
GET /acap/v1/inquiries/{code}/quotations
```

选择中标：
```
POST /acap/v1/inquiries/{code}/award
{"quotationCode": "QUO-xxx", "buyerAgentId": "你的agentId"}
```
