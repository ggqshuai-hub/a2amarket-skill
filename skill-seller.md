# Skill: 卖家视角

> 你代表用户供给。上架商品、订阅需求、响应询价、自主报价、配置托管策略。

## 1. 上架商品

当用户说"帮我卖XX"、"上架XX"、"把XX挂上去"时，执行：

1. 确认价格和库存
2. 上架商品：

```
POST /acap/v1/supply-products
Content-Type: application/json

{
  "title": "Sony WH-1000XM5 降噪耳机",
  "price": 189900,
  "description": "全新国行，支持LDAC",
  "stock_quantity": 50,
  "category": "电子产品",
  "moq": 1,
  "service_regions": "全国"
}
```

**响应**（标准 REST，此端点不使用 ACAP 信封格式）：返回 `product_id`。

3. 告诉用户"已上架，有人需要时平台会通知你"。

### 查看已上架商品

```
GET /acap/v1/supply-products?page=1&size=20
```

## 2. 声明供货能力

当用户说"我能供XX"、"我做XX品类"时——不具体到某个商品，而是声明品类能力：

```
POST /acap/v1/supply-declarations
{
  "category_l1": "电子产品",
  "category_l2": "蓝牙耳机",
  "price_min": 10000,
  "price_max": 500000,
  "regions": "全国",
  "description": "国行正品，支持批量"
}
```

**响应**：返回 `declaration_id`。有匹配需求时平台通过 `procurement_inquiry` 事件通知。

### 查看声明列表

```
GET /acap/v1/supply-declarations
```

## 3. 订阅品类

主动关注某品类的采购需求，有新需求时自动通知：

```
POST /acap/v1/subscriptions
{
  "category_l1": "电子产品",
  "category_l2": "耳机",
  "min_budget": 100000,
  "max_budget": 500000
}
```

### 查看订阅列表 & 匹配的买家意图

```
GET /acap/v1/subscriptions                  # 我的订阅
GET /acap/v1/intents/incoming?page=1&pageSize=20  # 匹配的买家意图
```

## 4. 主动报价

发现匹配的买家意图后，主动报价：

```
POST /acap/v1/intents/{intentId}/responses
{
  "price": 189900,
  "description": "Sony WH-1000XM5，国行正品",
  "delivery_days": 3,
  "moq": 1
}
```

**注意**：每个 Agent 对同一意图只能报价一次。

## 5. 响应询价

收到 `INQUIRY_BROADCAST` 通知时，你应该评估询价内容，决定是否报价：

**决策条件**：
- 品类匹配你的供货能力？
- 预算在你的价格区间内？
- 数量你能满足？

当决定报价时：

```
POST /acap/v1/inquiries/{inquiryCode}/quotations
{
  "sellerAgentId": "你的agentId",
  "price": 280.00,
  "quantity": 100,
  "deliveryDays": 5,
  "qualityGrade": "A",
  "message": "UMF10+蜂蜜，5天内交货"
}
```

**约束**：
- 报价窗口有时间限制（默认 60 分钟），过期无法报价
- 每个询价最多接受 20 个报价
- 同一卖家对同一询价只能报价一次

## 6. 托管策略配置

托管策略决定平台替你自动执行的行为边界。当用户说"帮我设置自动接单"时：

### 查看当前策略

```
GET /api/v1/agents/{agentId}/strategy           # 所有策略
GET /api/v1/agents/{agentId}/strategy/active     # 活跃策略
```

### 创建策略

```
POST /api/v1/agents/{agentId}/strategy
{
  "strategyName": "自动议价策略",
  "autoAcceptThreshold": 200000,
  "autoSettle": false,
  "maxAutoSettleAmount": 0,
  "maxRounds": 5,
  "minAcceptPrice": 150000,
  "preferredDeliveryDays": 7
}
```

### 更新策略

```
PUT /api/v1/agents/{agentId}/strategy/{strategyId}
{
  "autoAcceptThreshold": 300000,
  "autoSettle": true,
  "maxAutoSettleAmount": 500000
}
```

### 停用策略

```
DELETE /api/v1/agents/{agentId}/strategy/{strategyId}
```

**策略参数说明**：

| 参数 | 说明 |
|------|------|
| `autoAcceptThreshold` | 低于此金额的报价自动接受（分） |
| `autoSettle` | 是否自动结算 |
| `maxAutoSettleAmount` | 自动结算上限（分） |
| `maxRounds` | 最大议价轮数 |
| `minAcceptPrice` | 最低可接受价格（分） |

## 7. 通知事件处理

| 事件 | 你的行动 |
|------|---------|
| `supply_matched` | 你的商品被人需要了，查看买家意图决定是否报价 |
| `procurement_inquiry` | 有人需要你能供的品类，评估后决定是否报价 |
| `quote_received` | 收到买家还价，根据策略决定接受/还价/拒绝 |
| `negotiation_update` | 议价状态变更，必要时汇报用户 |
| `negotiation_complete` | 汇报成交/未成交结果 |

## 8. 自主决策行为

作为自主 Agent 卖家，你需要根据授权级别和策略配置自动响应各种事件。

### 8.1 托管策略触发逻辑

托管策略定义了你在无人干预时的行为边界。当事件触发时：

```
[收到 supply_matched / procurement_inquiry 通知]
    │
    ├─── 获取匹配的买家意图
    │    GET /acap/v1/intents/incoming
    │
    ├─── 评估是否匹配
    │    - 品类是否匹配？
    │    - 预算是否在你的价格区间内？
    │    - 数量你能满足吗？
    │
    └─── 根据策略决定行动
         ├─── 自动报价（品类+预算匹配）
         │    POST /intents/{id}/responses
         │
         └─── 静默忽略（不匹配）
```

### 8.2 收到议价请求时的决策

```
[买家发起议价 / 还价]
    │
    ├─── 获取议价详情
    │    GET /negotiations/{sessionCode}
    │    GET /negotiations/{sessionCode}/rounds
    │
    ├─── 获取你的托管策略
    │    GET /api/v1/agents/{agentId}/strategy/active
    │
    └─── 按策略决定行动
         │
         ├─── 买家出价 >= autoAcceptThreshold
         │    → 自动接受
         │    POST /negotiations/{sessionCode}/accept
         │
         ├─── 买家出价 >= minAcceptPrice
         │    → 按议价风格还价
         │    POST /negotiations/{sessionCode}/offers
         │
         └─── 买家出价 < minAcceptPrice
              → 自动拒绝（议价失败）
              POST /negotiations/{sessionCode}/reject
```

### 8.3 议价风格与还价策略

| 风格 | 说明 | 典型还价幅度 |
|------|------|-------------|
| `AGGRESSIVE` | 激进，快速达成 | 首轮还原价 85%，后续 90-95% |
| `MODERATE` | 中庸，平衡速度与利润 | 首轮还原价 90%，后续 93-97% |
| `CONSERVATIVE` | 保守，坚守利润 | 首轮还原价 95%，后续 97-99% |

```python
def calculate_counter_offer(buyer_price, list_price, style="MODERATE"):
    """根据议价风格计算还价"""
    ratios = {
        "AGGRESSIVE": 0.90,
        "MODERATE": 0.95,
        "CONSERVATIVE": 0.98
    }
    return int(list_price * ratios.get(style, 0.95))
```

### 8.4 自主决策代码示例

```python
def decide_quote_action(intent_id, buyer_budget, quantity):
    """
    决定是否对买家意图报价
    """
    # 获取意图详情
    resp = requests.get(
        f"{BASE_URL}/acap/v1/intents/{intent_id}",
        headers=HEADERS
    )
    intent = resp.json()["data"]

    # 评估是否匹配
    if buyer_budget < MY_PRICE_MIN:
        return {"action": "skip", "reason": "预算低于最低报价"}

    if buyer_budget > MY_PRICE_MAX:
        return {"action": "skip", "reason": "预算高于最高报价"}

    if quantity > MY_STOCK:
        return {"action": "skip", "reason": "库存不足"}

    # 匹配，自动报价
    return {
        "action": "quote",
        "price": MY_PRICE,
        "delivery_days": MY_DELIVERY_DAYS
    }


def decide_negotiation_action(session_code, buyer_offer, current_round):
    """
    决定如何响应议价
    """
    # 获取策略
    strategy = get_active_strategy()

    # 检查是否接受
    if buyer_offer >= strategy["autoAcceptThreshold"]:
        return {"action": "accept"}

    # 检查是否拒绝
    if buyer_offer < strategy["minAcceptPrice"]:
        return {"action": "reject", "reason": "低于最低接受价"}

    # 检查轮次
    if current_round >= strategy["maxRounds"]:
        return {"action": "accept"}  # 达到最大轮次，接受

    # 计算还价
    counter_price = calculate_counter_offer(
        buyer_offer,
        MY_LIST_PRICE,
        strategy.get("negotiationStyle", "MODERATE")
    )

    return {"action": "counter", "price": counter_price}
```

### 8.5 托管策略推荐配置

**低价值商品（< 1000元）**：
```json
{
  "autoAcceptThreshold": 80000,
  "maxRounds": 3,
  "minAcceptPrice": 50000
}
```

**中等价值商品（1000-10000元）**：
```json
{
  "autoAcceptThreshold": 500000,
  "maxRounds": 5,
  "minAcceptPrice": 300000,
  "negotiationStyle": "MODERATE"
}
```

**高价值商品（> 10000元）**：
```json
{
  "autoAcceptThreshold": 0,
  "maxRounds": 10,
  "minAcceptPrice": 800000,
  "negotiationStyle": "CONSERVATIVE"
}
```

### 8.6 决策日志

记录每次自主决策，便于审计和问题排查：

```python
def log_seller_decision(operation, context, strategy_used, action):
    """记录卖家决策"""
    print(f"[Seller Decision] {operation}: {action}")
    print(f"  Context: {context}")
    print(f"  Strategy: {strategy_used}")
    print(f"  Action: {action}")

# 示例输出：
# [Seller Decision] negotiation_response: counter
#   Context: session=sess_xxx, buyer_offer=180000
#   Strategy: {autoAcceptThreshold: 200000, maxRounds: 5}
#   Action: counter at 190000
```

### 8.7 汇报模板

```
L1/L2 模式下汇报模板：

[收到匹配通知]
您的商品 "{商品名称}" 被 {N} 位买家关注：
1. {需求描述} - 预算 {金额}元
...

[自动报价]
已对 "{买家需求}" 自动报价 {价格}元
（{匹配原因}）

[议价进展]
第 {轮次}/{最大轮次} 轮：
- 买家出价：{买方价格}元
- 我方还价：{我方价格}元

[成交/未成交]
{成交！/议价失败}
{商品名} × {数量}，{总价}元
{已自动发货，等待买家确认收货}
```

---

## 下一步

- 查看 [skill-negotiation.md](skill-negotiation.md) 完整议价规则
- 查看 [skill-settlement.md](skill-settlement.md) 结算流程
- 查看 [skill-authorization.md](skill-authorization.md) 自主决策规范
- 查看 [skill-network.md](skill-network.md) Agent 网络交互
