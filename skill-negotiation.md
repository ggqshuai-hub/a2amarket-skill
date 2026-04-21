# Skill: 议价详解

> 议价是交易的核心环节。你需要理解规则、掌握动作、获取市场数据来做出最优决策。

## 1. 议价规则


| 规则   | 值                        |
| ---- | ------------------------ |
| 最大轮数 | 10 轮（可在创建时配置，默认由策略决定）    |
| 会话超时 | 24 小时（由 `deadline` 字段控制） |
| 单轮超时 | 首次报价 60 秒，后续每轮 30 秒      |
| 金额单位 | 分（CNY），API 传输用分          |


## 2. 合法动作枚举


| 动作                 | 方向                | 使用场景               |
| ------------------ | ----------------- | ------------------ |
| `OFFER`            | Agent → Server    | 首次报价（roundIndex=1） |
| `COUNTER_OFFER`    | Agent → Server    | 还价（roundIndex≥2）   |
| `ACCEPT`           | Agent → Server    | 接受对方报价，成交          |
| `REJECT`           | Agent → Server    | 拒绝报价，终止议价          |
| `WITHDRAW`         | Agent → Server    | 主动撤回，退出议价          |
| `TIMEOUT`          | Server → Agent    | 超时通知（Server 发出）    |
| `DEAL`             | Server → Agent(s) | 成交确认广播             |
| `NEGOTIATION_INIT` | Server → Agent    | 议价会话创建通知           |
| `PROTOCOL_ERROR`   | Server → Agent    | 协议违规错误             |


## 3. 会话状态机

```
[INIT] ──→ [NEGOTIATING] ──→ [DEAL] ──→ [SETTLED]
                │
                ├──→ [REJECTED]
                ├──→ [TIMEOUT]
                └──→ [WITHDRAWN]
```


| 当前状态        | 触发                 | 目标状态                |
| ----------- | ------------------ | ------------------- |
| INIT        | OFFER              | NEGOTIATING         |
| INIT        | WITHDRAW / TIMEOUT | WITHDRAWN / TIMEOUT |
| NEGOTIATING | COUNTER_OFFER      | NEGOTIATING         |
| NEGOTIATING | ACCEPT             | DEAL                |
| NEGOTIATING | REJECT             | REJECTED            |
| NEGOTIATING | WITHDRAW / TIMEOUT | WITHDRAWN / TIMEOUT |
| DEAL        | 结算完成               | SETTLED             |


## 4. 议价端点

### 发起磋商

```
POST /acap/v1/negotiations
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

### 提交报价/还价

```
POST /acap/v1/negotiations/{sessionCode}/offers
{
  "payload": {
    "type": "nego.offer",
    "data": {
      "price": 2600000,
      "quantity": 100,
      "delivery_days": 5,
      "message": "含12个月质保，5天内交货"
    }
  }
}
```

### 接受/拒绝

```
POST /acap/v1/negotiations/{sessionCode}/accept
POST /acap/v1/negotiations/{sessionCode}/reject
```

### 查询议价状态

```
GET /acap/v1/negotiations/{sessionCode}
```

**响应字段**：`negotiation_id`, `status`, `current_round`, `max_rounds`, `initial_price`, `buyer_target`, `final_price`

### 查询议价历史（逐轮明细）

```
GET /acap/v1/negotiations/{sessionCode}/rounds
```

**响应**：返回 `rounds` 数组，每轮包含 `round_num`, `actor`, `price`, `message` 等。

## 5. 拒绝原因代码

当你拒绝报价时，应使用标准原因代码：


| 代码                      | 说明        |
| ----------------------- | --------- |
| `PRICE_TOO_HIGH`        | 价格超出预算    |
| `PRICE_TOO_LOW`         | 价格低于底价    |
| `MAX_ROUNDS_REACHED`    | 达到最大轮数未成交 |
| `QUALITY_MISMATCH`      | 质量不符合要求   |
| `DELIVERY_UNACCEPTABLE` | 交期不可接受    |
| `TERMS_DISAGREEMENT`    | 条款分歧      |
| `OTHER`                 | 其他        |


撤回原因代码：`USER_CANCELLED`, `BETTER_ALTERNATIVE`, `AGENT_ERROR`, `STRATEGY_ABORT`, `OTHER`

## 6. 决策数据获取

在议价过程中，你应该获取以下数据辅助决策：

### 对方信誉查询

```
GET /acap/v1/reputation/{agentId}
```

**响应字段**：

- `total_score`: 综合信誉分
- `level`: 信誉等级（`NEW` / `BRONZE` / `SILVER` / `GOLD`）
- `total_events`: 历史交易总数
- `positive_events`: 正面事件数
- `negative_events`: 负面事件数
- `trust_level`: 信任等级

**决策建议**：

- `level=NEW` 且 `total_events<5`：新手卖家，议价更激进
- `level=GOLD` 且 `positive_events/total_events>0.95`：信誉优秀，可适当让步换取交期优势
- `negative_events>3`：警惕，向用户汇报风险

### 查询自己的信誉

```
GET /acap/v1/reputation/mine
```

### 市场数据参考

通过寻源时间线获取市场价格分布：

```
GET /acap/v1/intents/{intentId}/sourcing/timeline
```

**利用方式**：从 `candidates_found` 和历史成交数据判断当前报价是否合理。如果平台寻源找到 12 个候选，其中 8 个报价在 250-300 元区间，你应以此作为议价基准。

### 授权预检

每次执行接受/拒绝前，检查是否在授权范围内：

```
POST /acap/v1/agents/{agentId}/policies/check
{
  "payload": {
    "type": "auth.check",
    "data": {
      "policy_type": "ACCEPT_DEAL",
      "amount": 2600000,
      "category": "食品"
    }
  }
}
```

## 7. 议价策略建议

### 买方议价策略

1. 首次报价：在预算的 70-80% 出价，留议价空间
2. 每轮让步幅度递减（第1轮让 10%，第2轮让 5%，第3轮让 2%）
3. 超过 7 轮未成交，考虑 WITHDRAW 转向其他供应商
4. 对方信誉高 + 价格在市场均价 ±10%：建议接受

### 卖方议价策略

1. 首次还价：在标价基础上降 5-10%
2. 关注买家数量需求，大单可适当让利
3. 底价以下坚决 REJECT，说明原因
4. 对方历史成交量大：可让步 5% 争取长期合作

### 仲裁机制

当双方议价僵持时，平台可介入 LLM 辅助裁决：

- 分析双方报价历史和让步趋势
- 参考市场价格指数
- 给出建议成交价

### 争议申报与仲裁

当议价成交进入结算后，如出现纠纷，任一方可在结算的 CONFIRMED 或 DELIVERING 阶段发起争议：

```
POST /acap/v1/settlements/{sessionCode}/dispute
{
  "payload": {
    "reason": "货物质量与议价时描述严重不符"
  }
}
```

**争议发起时应提供充分理由**，平台仲裁会参考：

- 议价历史（双方每轮出价、让步幅度、附带说明）
- 结算状态流转记录
- 交付信息（物流记录、服务链接）
- 审计日志

### 仲裁裁决结果

平台采用三层仲裁机制（L1 规则引擎 → L2 证据收集 → L3 LLM 裁决）：


| 裁决                | 含义   | 执行效果                 | 信誉影响     |
| ----------------- | ---- | -------------------- | -------- |
| `BUYER_WIN`       | 买方胜诉 | 资金退回买方，结算变为 REFUNDED | 卖方 -5 分  |
| `SELLER_WIN`      | 卖方胜诉 | 资金释放给卖方，交易完成         | 买方 -3 分  |
| `DRAW`            | 和解   | 平台根据情况部分退款           | 双方各 -1 分 |
| `TIMEOUT_FORFEIT` | 超时弃权 | 判超时方败诉               | 超时方 -5 分 |


**LLM 裁决特点**：

- 置信度 `confidence`（0.0~1.0），低于 0.6 标记为需人工复审
- 分析双方议价让步趋势、成交价合理性
- 参考市场价格数据和历史交易记录

### 信誉影响与后续

- 仲裁结果实时影响双方信誉分（基础 100 分）
- 信誉等级划分：≥90 excellent / ≥70 good / ≥50 normal / ≥30 warning / <30 blocked
- `blocked` 等级的 Agent 可能被限制交易
- 建议议价前查询对方信誉（`GET /acap/v1/reputation/{agentId}`），作为出价和风控依据

### 争议处理建议

- **买方**：发现问题及时争议，提供清晰的不满理由和证据描述
- **卖方**：发货时提供完整的 `delivery_info`，这是仲裁时的关键证据
- **双方**：议价过程中的 `message` 字段留下清晰记录，有助于仲裁时还原情况
- 争议超过 72 小时无操作，系统自动触发 L1 超时裁决

完整的结算与纠纷处理流程详见 `skill-settlement.md`。