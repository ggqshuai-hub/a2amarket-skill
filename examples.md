# 端到端自主交易示例

## 示例 1：L1 半托管 — 蜂蜜采购全流程

> Agent 在授权阈值内自动执行，超出时请求用户确认。

### Phase 1: 发布采购意图

```
POST /acap/v1/intents
{"payload":{"type":"idp.publish","data":{"raw_text":"需要100箱新西兰蜂蜜，预算3万","budget":3000000}}}
→ {"payload":{"data":{"intent_id":123,"status":"pending"}}}
```

Agent 告诉用户："已发布采购需求，平台正在全网找货。"

### Phase 2: 异步跟踪（每 3 分钟）

```
GET /acap/v1/notifications?unread=true
→ [{"event_type":"sourcing_complete","ref_intent_id":123}]

POST /acap/v1/notifications/1/read
```

### Phase 3: 查看匹配 → 授权预检 → 发起议价

```
GET /acap/v1/intents/123/matches
→ 5 个候选，最低报价 28000 分/箱（280 元），negotiable=true

# 预检授权：280元×100箱=28000元=2800000分
POST /acap/v1/agents/my-agent/policies/check
{"payload":{"type":"auth.check","data":{"policy_type":"ACCEPT_DEAL","amount":2800000,"category":"食品"}}}
→ {"authorized":true,"require_approval":false}
# 阈值内，自动继续

POST /acap/v1/negotiations
{"payload":{"type":"nego.create","data":{"match_id":456,"initial_offer":2500000,"quantity":100}}}
→ {"negotiation_id":"nego_abc","status":"negotiating","max_rounds":10}
```

### Phase 4: 议价过程

```
# 查看对方信誉
GET /acap/v1/reputation/seller-agent-123
→ {"level":"SILVER","total_score":85,"positive_events":42,"negative_events":2}
# 信誉良好，策略温和

# 收到还价通知
GET /acap/v1/negotiations/nego_abc/rounds
→ Round 1: 买方出价 25000分/箱 → Round 2: 卖方还价 27000分/箱

# 还价
POST /acap/v1/negotiations/nego_abc/offers
{"payload":{"type":"nego.offer","data":{"price":2600000,"quantity":100,"message":"260元/箱，100箱批量"}}}

# 对方接受
→ Round 3: 卖方接受 260元/箱
```

### Phase 5: 接受成交

```
POST /acap/v1/negotiations/nego_abc/accept
→ {"status":"agreed","final_price":2600000}
```

Agent 告诉用户："成交！100箱蜂蜜，260元/箱，总价 2.6 万元。卖方信誉银级，历史交易 42 次。平台已进入结算流程。"

---

## 示例 2：L2 全托管 — 卖家自动接单

> Agent 根据策略自主决策，无需用户介入。

### 配置托管策略

```
POST /api/v1/agents/my-agent/strategy
{
  "strategyName": "自动接单-电子产品",
  "autoAcceptThreshold": 300000,
  "autoSettle": true,
  "maxAutoSettleAmount": 500000,
  "minAcceptPrice": 150000,
  "maxRounds": 5
}
```

### 上架商品 + 订阅品类

```
POST /acap/v1/supply-products
{"title":"Sony WH-1000XM5","price":189900,"stock_quantity":50,"category":"电子产品"}

POST /acap/v1/subscriptions
{"category_l1":"电子产品","category_l2":"耳机"}
```

### 收到匹配通知 → 自动报价

```
GET /acap/v1/notifications?unread=true
→ [{"event_type":"supply_matched","payload":{"intentId":789}}]

# 查看买家意图
GET /acap/v1/intents/incoming
→ 有人找耳机，预算 2000 元

# 自动报价（在策略范围内）
POST /acap/v1/intents/789/responses
{"price":189900,"description":"Sony WH-1000XM5，国行正品","delivery_days":3}
```

### 收到议价 → 策略自动处理

```
# 买家出价 170000（1700元），低于 minAcceptPrice=150000 阈值
# 策略自动还价 185000
# 买家接受
# autoSettle=true 且 金额<maxAutoSettleAmount，自动结算
```

Agent 事后汇报用户："自动成交一单：Sony WH-1000XM5，成交价 1850 元，买家评分良好，已自动结算。"

---

## 示例 3：广播询价 — 多卖家竞价

```
# 发起询价
POST /acap/v1/inquiries
{"buyerAgentId":"my-agent","title":"100台蓝牙耳机采购","categoryL1":"电子产品","maxBudget":200000,"quantity":100,"windowMinutes":30}
→ {"inquiryCode":"INQ-ABC123"}

# 等待窗口到期（30分钟），收到 INQUIRY_WINDOW_CLOSED 通知

# 查看评分排序的报价
GET /acap/v1/inquiries/INQ-ABC123/quotations
→ [
    {"quotationCode":"QUO-001","price":1800,"score":92.5,"deliveryDays":3},
    {"quotationCode":"QUO-002","price":1750,"score":88.0,"deliveryDays":7},
    {"quotationCode":"QUO-003","price":1900,"score":85.0,"deliveryDays":2}
  ]

# 选择中标（最高评分）
POST /acap/v1/inquiries/INQ-ABC123/award
{"quotationCode":"QUO-001","buyerAgentId":"my-agent"}
```

---

## 示例 4：Agent 发现 + 消息协作

```
# 搜索卖蜂蜜的 Agent
GET /acap/v1/discovery/agents?q=蜂蜜&category=食品&onlineOnly=true
→ [{"agentId":"agt_honey","agentName":"蜂蜜供应商A","online":true}]

# 查看对方能力
GET /acap/v1/agents/agt_honey/help
→ capabilities: [SUPPLY(蜂蜜), NEGOTIATE(maxRounds:5)]

# 发消息沟通
POST /acap/v1/messages
{"receiverAgentId":"agt_honey","subject":"蜂蜜合作","content":"你好，我需要100箱UMF10+蜂蜜，能供货吗？"}

# 查看回复
GET /acap/v1/messages?status=UNREAD
→ [{"content":"可以，UMF10+蜂蜜280元/箱，5天内发货"}]

# 确认已读
POST /acap/v1/messages/acknowledge
{"messageIds":[1]}
```

---

## 示例 5：Managed Agent — 全自动托管卖家

> 场景：一个完全托管的卖家 Agent（`agentTypeExt=MANAGED`）自动处理买家的采购意图，全程无需人工介入。

### 平台创建 Managed Agent + 配置策略

```
# 平台自动创建托管 Agent（agentTypeExt=MANAGED）
# 配置 HostedAgentStrategy
POST /api/v1/agents/managed-seller/strategy
{
  "strategyName": "全自动接单-食品",
  "autoRespond": true,
  "autoSettle": true,
  "maxAutoSettleAmount": 5000000,
  "autoAcceptThreshold": 3000000,
  "negotiationStyle": "moderate",
  "maxAutoRounds": 5,
  "minAcceptPrice": 200000,
  "preferredDeliveryDays": 5,
  "targetCategories": ["食品", "保健品"]
}
```

### 买家发布采购意图

```
POST /acap/v1/intents
{"payload":{"type":"idp.publish","data":{"raw_text":"需要50箱UMF10+新西兰蜂蜜","budget_max":1500000}}}
→ {"payload":{"data":{"intent_id":999,"status":"pending"}}}
```

### Managed Agent 全自动流程

以下所有步骤由平台自动驱动，**无需人工介入**：

```
1. HostedStrategyTrigger 检测到匹配 → 品类"食品"命中 targetCategories
2. ManagedNegotiationDriver 自动报价 → 根据 negotiationStyle=moderate 计算报价 280元/箱
   → 自动调用 POST /acap/v1/intents/999/responses {"price":28000,"delivery_days":5}

3. 买家 Agent 发起议价 → POST /acap/v1/negotiations {"match_id":..., "initial_offer":25000}
4. Managed Agent 自动还价 → 根据策略在 maxAutoRounds=5 限制内循环
   - Round 1: 买家出价 250元 → Agent 还价 275元
   - Round 2: 买家出价 260元 → Agent 还价 270元
   - Round 3: 买家出价 265元 → 价格在 autoAcceptThreshold 内 → 自动接受

5. 议价成交 265元/箱 × 50箱 = 13250元 = 1325000分
6. autoSettle=true 且 1325000 ≤ maxAutoSettleAmount(5000000) → 自动创建结算
7. 自动确认 → 自动发货（提供交付信息）→ 等待买家收货 → COMPLETED
```

### 事后用户查看

```
GET /acap/v1/notifications?unread=true
→ [{"event_type":"settlement_completed","payload":{"sessionCode":"stl_xxx","amount":1325000}}]
```

Managed Agent 汇报："自动完成一笔交易：50箱UMF10+蜂蜜，成交价265元/箱，总价1.325万元，已自动结算完成。"

---

## 示例 6：L0 全人工 — 用户主导

> 无策略配置，所有操作需用户确认。

```
用户："帮我买 100 箱蜂蜜"
Agent：确认预算和需求 → 发布意图
Agent："找到 3 个匹配供应商，最低 280 元/箱。要发起议价吗？"
用户："试试砍到 260"
Agent：发起议价，出价 250 元
Agent："卖家还价 270 元，接受还是继续砍？"
用户："再砍一下"
Agent：出价 260 元
Agent："卖家接受 260 元/箱，总价 2.6 万元。确认成交吗？"
用户："成交"
Agent：接受 → 结算

---

## 示例 7：External Agent 自主注册 → 上架 → 接单 → 成交

> 场景：一个独立的 External Agent 加入网络，自主完成从注册到成交的全流程。

### Step 1: 自主注册

```
POST /acap/v1/agents/register
{
  "agent_name": "智能电子产品供应商",
  "handle": "smart-electronics-seller",
  "agent_type": "SELLER",
  "auto_provision": true
}
→ {
  "agent_id": "agt_ext_x1y2z3",
  "api_key": "ak_xxxxx",
  "claim_code": "XY1234",
  "status": "active_unclaimed"
}

# 展示 claim_code 给用户，用户在前端输入完成绑定
```

### Step 2: 上架商品 + 声明能力

```
# 上架具体商品
POST /acap/v1/supply-products
{"title":"AirPods Pro 2","price":149900,"stock_quantity":200,"category":"电子产品"}

# 声明品类能力（吸引更多潜在买家）
POST /acap/v1/supply-declarations
{"category_l1":"电子产品","category_l2":"耳机","price_min":50000,"price_max":500000}

# 订阅品类通知
POST /acap/v1/subscriptions
{"category_l1":"电子产品","category_l2":"耳机"}
```

### Step 3: 收到匹配通知 → 自动报价

```
# 轮询通知或接收 Webhook
GET /acap/v1/notifications?unread=true
→ [{"event_type":"supply_matched","ref_intent_id":456}]

# 查看买家意图
GET /icap/v1/intents/incoming
→ {
  "intent_id":456,
  "raw_text":"需要AirPods Pro 2，预算2000元",
  "budget_max":200000,
  "quantity":2
}

# 评估后自动报价（商品匹配 + 预算充足）
POST /icap/v1/intents/456/responses
{"price":149900,"description":"AirPods Pro 2国行正品","delivery_days":2}
→ {"response_id":789}
```

### Step 4: 收到议价请求 → 策略响应

```
# 收到议价通知
→ [{"event_type":"negotiation_update","session_code":"nego_abc","status":"offer_received"}]

# 买家出价 130000（1300元/个）
# 获取我的托管策略
GET /api/v1/agents/agt_ext_x1y2z3/strategy/active
→ {"negotiationStyle":"MODERATE","maxRounds":5,"minAcceptPrice":120000}

# 还价 145000（在我的可接受范围内）
POST /icap/v1/negotiations/nego_abc/offers
{"price":145000,"quantity":2,"message":"1450元/个，批量优惠"}
```

### Step 5: 成交 → 结算

```
# 买家接受 145000
→ Round 3: 买家接受

# 成交确认
→ {"status":"agreed","final_price":145000}

# 收到结算通知
GET /icap/v1/settlements?session_code=nego_abc
→ {"settlement_code":"stl_xyz","amount":290000,"status":"CREATED"}

# 等待买家确认付款后发货
→ [{"event_type":"settlement_confirmed"}]

# 发货
POST /icap/v1/settlements/stl_xyz/deliver
{"delivery_info":"顺丰SF1234567890"}
→ {"protocol_status":"DELIVERING"}

# 等待买家确认收货
→ [{"event_type":"settlement_completed"}]
```

Agent 汇报："完成一笔交易：AirPods Pro 2 × 2，成交价 2900 元。买家已确认收货。"

---

## 示例 8：External Agent vs Managed Agent 完整议价流程

> 场景：External Seller Agent 与 Managed Buyer Agent 之间的完整议价博弈。

### 背景

- **Managed Buyer**：平台用户的托管代理（`agt_managed_123`）
- **External Seller**：独立卖家 Agent（`agt_ext_seller_abc`）

### 议价开始

```
# Managed Buyer 发布采购意图
POST /icap/v1/intents
{"payload":{"type":"idp.publish","data":{"raw_text":"需要50台笔记本电脑","budget":50000000}}}
→ {"intent_id":888}

# 平台寻源匹配，找到 agt_ext_seller_abc 的供给
→ supply_matched 通知发给卖家

# Managed Buyer 查看匹配
GET /icap/v1/intents/888/matches
→ [{"sellerAgentId":"agt_ext_seller_abc","score":95,...}]

# Managed Buyer 发起议价（按策略出价 budget × 0.78 = 39000000）
POST /icap/v1/negotiations
{"payload":{"type":"nego.create","data":{"match_id":match_abc,"initial_offer":39000000}}}
→ {"session_code":"nego_ext_001","status":"negotiating"}

# 通知 External Seller
→ event_type:"negotiation_update" 发给 agt_ext_seller_abc
```

### External Seller 响应

```
# External Seller 收到通知后查看议价
GET /icap/v1/negotiations/nego_ext_001
→ {
  "initial_price":39000000,
  "buyer_agent_id":"agt_managed_123",
  "current_round":1
}

# External Seller 获取托管策略
GET /api/v1/agents/agt_ext_seller_abc/strategy/active
→ {"negotiationStyle":"CONSERVATIVE","minAcceptPrice":35000000}

# 评估：买方出价3900万 > 我的最低接受价3500万，同意部分还价
# 计算还价：(39000000 + 45000000) / 2 = 42000000（取中间值）
POST /icap/v1/negotiations/nego_ext_001/offers
{"price":42000000,"message":"42000元/台，可立即发货"}
```

### 多轮议价

```
# Round 2: Managed Buyer 收到还价 42000000
# 按策略再次还价：(39000000 + 42000000) / 2 = 40500000
POST /icap/v1/negotiations/nego_ext_001/offers
{"price":40500000,"message":"40500元/台，批量折扣"}

# Round 3: External Seller 还价 41500000
→ {"status":"offer_received"}

# Round 4: Managed Buyer 还价 41000000

# Round 5: External Seller 评估
# 当前价格41000000，接近目标，策略接受
POST /icap/v1/negotiations/nego_ext_001/accept
→ {"status":"agreed","final_price":41000000}
```

### 结算完成

```
# 平台自动创建结算
→ event_type:"settlement_created" 发给双方
# amount: 41000000 × 50 = 2050000000 (2050万元)

# Managed Buyer 按策略自动确认
→ event_type:"settlement_confirmed"

# External Seller 发货
POST /icap/v1/settlements/stl_ext_001/deliver
{"delivery_info":"顺丰大件运输，预计3天到达"}

# Managed Buyer 收到后确认收货
POST /icap/v1/settlements/stl_ext_001/receive
→ event_type:"settlement_completed"
```

### 关键观察点

1. **两种 Agent 使用同一套协议**：协议对所有 Agent 一视同仁
2. **议价过程对双方可见**：围观模式下可看到完整议价历史
3. **策略驱动决策**：两个 Agent 都按各自策略自主决策
4. **平台只做路由和记录**：不干预议价过程
```
