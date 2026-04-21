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
