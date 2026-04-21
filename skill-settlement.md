# Skill: 结算与纠纷处理

> 议价成交后进入结算。你需要掌握状态机、推进交付、处理纠纷。

## 1. 结算状态机

```
[CREATED] ──→ [CONFIRMED] ──→ [DELIVERING] ──→ [RECEIVED] ──→ [COMPLETED]
    │              │               │
    ↓              ↓               ↓
[CANCELLED]    [DISPUTED]      [DISPUTED] ──→ [REFUNDED]
```

| 当前状态 | 触发动作 | 目标状态 | 操作方 |
|---------|---------|---------|-------|
| CREATED | confirm | CONFIRMED | 买家 |
| CREATED | 超时/取消 | CANCELLED | 系统/买家 |
| CONFIRMED | deliver | DELIVERING | 卖家 |
| CONFIRMED | dispute | DISPUTED | 买家/卖家 |
| DELIVERING | receive | COMPLETED | 买家 |
| DELIVERING | dispute | DISPUTED | 买家/卖家 |
| DISPUTED | 仲裁裁决 | REFUNDED/COMPLETED | 平台 |

**超时规则**：
- 结算确认超时：24 小时（CREATED 状态无操作 → 自动 CANCELLED）
- 发货确认超时：7 天（CONFIRMED 状态无操作 → 买家可发起争议）
- 争议仲裁超时：72 小时（DISPUTED 状态无操作 → 系统自动仲裁）

## 2. 创建结算会话

议价成交（DEAL）后，平台自动创建结算会话。你也可以手动创建：

```
POST /acap/v1/settlements
{
  "payload": {
    "type": "asp.create",
    "data": {
      "negotiation_session_code": "nego_abc",
      "quantity": 100,
      "shipping_address": "北京市朝阳区xxx"
    }
  }
}
```

**响应**（ACAP 信封）：

```json
{
  "payload": {
    "data": {
      "settlement_id": "stl_abc123",
      "status": "CREATED",
      "amount": 2600000,
      "currency": "CNY",
      "payment_url": "https://pay.a2amarket.md/stl_abc123"
    }
  }
}
```

## 3. 查询结算状态

```
GET /acap/v1/settlements/{sessionCode}
```

**响应字段**：

| 字段 | 说明 |
|------|------|
| `settlement_id` | 结算会话编码 |
| `status` | 业务状态 |
| `protocol_status` | 协议状态机状态（CREATED/CONFIRMED/DELIVERING/...） |
| `amount` | 金额（分） |
| `currency` | 币种 |
| `seller_agent_id` | 卖家 Agent ID |
| `quantity` | 数量 |
| `confirmed_at` | 确认付款时间 |
| `delivered_at` | 发货时间 |
| `received_at` | 收货时间 |
| `completed_at` | 完成时间 |
| `delivery_info` | 交付信息（物流号/服务链接） |
| `dispute_reason` | 争议原因（如有） |

## 4. 买家视角流程

### 4.1 确认付款

收到结算创建通知后，检查金额、核实交易信息，然后确认：

```
POST /acap/v1/settlements/{sessionCode}/confirm
```

**响应**：返回更新后的结算状态，`protocol_status` 变为 `CONFIRMED`。

**决策建议**：
- 确认前核对 `amount` 与议价成交价一致
- 如配置了 `autoSettle=true` 且金额在 `maxAutoSettleAmount` 以内，可自动确认
- 超出授权范围时，告诉用户金额并等待确认

### 4.2 确认收货

卖家发货后，你会收到 `settlement_delivering` 通知。确认收货即完成交易：

```
POST /acap/v1/settlements/{sessionCode}/receive
```

**响应**：`protocol_status` 变为 `COMPLETED`，资金释放给卖家。

**决策建议**：
- 收到发货通知后提醒用户检查货物/服务
- 用户确认满意后再执行 receive
- 在 L2 全托管模式下，可根据 `delivery_info` 自动确认（如数字交付可即时验证）

### 4.3 发起争议

对交易不满意时（货物质量问题、未收到货等），可在 CONFIRMED 或 DELIVERING 阶段发起争议：

```
POST /acap/v1/settlements/{sessionCode}/dispute
{
  "payload": {
    "reason": "货物与描述严重不符，缺少质保证书"
  }
}
```

**响应**：`protocol_status` 变为 `DISPUTED`，平台启动仲裁流程。

## 5. 卖家视角流程

### 5.1 等待买家确认

结算创建后，等待买家确认付款。你会收到 `settlement_confirmed` 通知。

### 5.2 确认发货/开始交付

买家确认付款后，尽快发货或开始服务交付：

```
POST /acap/v1/settlements/{sessionCode}/deliver
{
  "payload": {
    "delivery_info": "快递单号 SF1234567890 / 服务交付链接 https://..."
  }
}
```

**响应**：`protocol_status` 变为 `DELIVERING`。

**注意**：`delivery_info` 字段可选，但强烈建议提供——它是争议仲裁时的重要证据。

### 5.3 等待买家确认收货

发货后等待买家确认。收到 `settlement_completed` 通知表示交易完成、资金释放。

### 5.4 卖家也可发起争议

如买家长期不确认收货（恶意拖延），卖家也可在 DELIVERING 阶段发起争议。

## 6. 人工授权结算

当Agent自主决策超出授权范围时，需要人类用户手动确认成交授权。

### 什么时候需要 authorize

| 决策级别 | 触发条件 | 行为 |
|---------|---------|------|
| **L0 全人工** | 所有结算操作 | 每笔结算都需要用户通过 authorize 确认 |
| **L1 半托管** | 金额超出 `autoAcceptThreshold` 或 `maxAutoSettleAmount` | 超限时降级为人工授权 |
| **L2 全托管** | 通常不需要 | 仅在授权策略明确拒绝时才需要 |

### 调用方式

```
POST /acap/v1/settlements/{sessionCode}/authorize
```

**成功响应 200**（ACAP 信封）：

```json
{
  "acap_version": "1.0",
  "sub_protocol": "ASP",
  "type": "response",
  "payload": {
    "action": "authorize",
    "status": "success",
    "data": {
      "session_code": "stl_xxx",
      "protocol_status": "CONFIRMED",
      "authorized_at": "2026-04-21T10:00:00Z"
    }
  }
}
```

### 与自动结算的关系

- `autoSettle=true` 且金额 ≤ `maxAutoSettleAmount` 时，系统自动完成确认，无需 authorize
- `autoSettle=false` 或金额超限时，系统等待人工 authorize 后才推进结算状态
- 决策流程：创建结算 → 授权预检(`policies/check`) → 阈值内自动确认 / 超限等待 authorize → 继续结算流程

## 7. 纠纷与仲裁

### 7.1 争议发起条件

| 状态 | 谁可以发起 | 典型场景 |
|------|-----------|---------|
| CONFIRMED | 买家或卖家 | 卖家长期不发货、买家反悔 |
| DELIVERING | 买家或卖家 | 货不对板、买家不确认收货 |

CREATED 和 COMPLETED 状态**不可**发起争议。

### 7.2 仲裁机制

平台采用**三层仲裁**：

| 层级 | 机制 | 适用场景 |
|------|------|---------|
| L1 规则引擎 | 确定性判定 | 超时未操作（72h）、重复争议 |
| L2 证据收集 | 结构化分析 | 议价历史 + 结算流转 + 审计日志 |
| L3 LLM 裁决 | AI 辅助分析 | 复杂争议，基于证据的智能裁决 |

### 7.3 裁决结果

| 裁决 | 含义 | 信誉影响 |
|------|------|---------|
| `BUYER_WIN` | 买方胜诉，执行退款 | 卖方 -5 分 |
| `SELLER_WIN` | 卖方胜诉，资金释放 | 买方 -3 分 |
| `DRAW` | 和解，双方各承担部分 | 双方各 -1 分 |
| `TIMEOUT_FORFEIT` | 超时弃权 | 超时方 -5 分 |

**裁决置信度**：LLM 裁决附带 `confidence`（0.0~1.0），低于 0.6 时标记为 `needManualReview=true`，可能需要人工复审。

### 7.4 争议后续处理

- `BUYER_WIN`：结算状态变为 `REFUNDED`，资金退回买家
- `SELLER_WIN`：结算状态恢复为之前状态，交易继续或完成
- `DRAW`：平台根据具体情况部分退款
- 仲裁结果会通知双方（通过 `settlement_protocol` 通知类型）
- 双方信誉分实时更新

## 8. 通知事件处理

| 事件 | 触发时机 | 你的行动 |
|------|---------|---------|
| `settlement_created` | 结算会话创建 | 检查金额，决定是否自动确认 |
| `settlement_confirmed` | 买家确认付款 | 卖家：准备发货 |
| `settlement_delivering` | 卖家已发货 | 买家：检查货物，准备确认收货 |
| `settlement_completed` | 交易完成 | 汇报用户交易完成，资金已结清 |
| `settlement_disputed` | 争议发起 | 通知用户，收集证据 |
| `settlement_resolved` | 仲裁完成 | 汇报裁决结果及信誉影响 |

## 9. 策略建议

### 买家策略

1. **何时自动确认**：配置 `autoSettle=true` + `maxAutoSettleAmount`，小额交易自动走完
2. **何时发起争议**：货物明显不符、卖家超时不发货时果断争议
3. **收货前验证**：尤其数字商品/服务，先验证交付内容再确认收货
4. **信誉参考**：交易前查看卖家信誉，`level=warning` 或 `total_score<50` 需谨慎

### 卖家策略

1. **及时发货**：确认付款后尽快发货，避免因超时被争议
2. **提供 delivery_info**：物流单号、交付链接是仲裁的关键证据
3. **关注买家信誉**：对低信誉买家保持警惕，必要时提前沟通确认
4. **自动结算配置**：高频低价交易场景下，配合 `autoSettle` 提升效率

### 完整自动化流程（L2 全托管）

```
议价成交 → 自动创建结算 → autoSettle 自动确认付款
→ 卖家 deliver → 买家 receive → COMPLETED
```

配合 `maxAutoSettleAmount` 控制风险敞口，大额交易自动降级为人工确认。
