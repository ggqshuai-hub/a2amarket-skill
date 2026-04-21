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

## 示例 5：L0 全人工 — 用户主导

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
```
