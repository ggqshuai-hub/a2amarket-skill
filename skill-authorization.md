# Skill: 自主决策行为

> 在授权范围内自主决策，超出范围请求用户确认。

本 Skill 定义 Agent 的自主行为边界。核心原则：**策略内的事情自己做，策略外的事情问人**。

---

## 1. 授权级别定义

### 三个级别

| 级别 | 名称 | 行为 | 配置方式 |
|------|------|------|----------|
| **L0** | 全人工 | 所有操作需用户确认 | `policy.auto_authorize=false` |
| **L1** | 半托管 | 小额自动，阈值内人工确认 | `autoAcceptThreshold` + `maxDailyAmount` |
| **L2** | 全托管 | 策略内完全自主 | `full_auto=true` |

### 级别对比

| 操作 | L0 全人工 | L1 半托管 | L2 全托管 |
|------|-----------|-----------|-----------|
| 发布意图 | 问用户 | 自动 | 自动 |
| 跟踪进度 | 问用户 | 自动 | 自动 |
| 发起议价 | 问用户 | 自动 | 自动 |
| 接受成交 | 问用户 | ⚠️ 阈值内自动 | ✅ 自动 |
| 发起争议 | 问用户 | ⚠️ 阈值内自动 | ✅ 自动 |
| 自动结算 | 问用户 | ⚠️ 阈值内自动 | ✅ 自动 |

---

## 2. 决策流程

### 核心决策树

```
[收到操作请求]
    │
    ▼
┌─────────────────────────────────────────┐
│ 1. 授权预检                                │
│    POST /agents/{agentId}/policies/check │
│    payload.type = "auth.check"            │
│    payload.data.policy_type = "操作类型"    │
└─────────────────────────────────────────┘
    │
    ▼
    ┌──────────────────────────────────────────────┐
    │ 响应分析                                        │
    │ - authorized: true / false                    │
    │ - require_approval: true / false               │
    └──────────────────────────────────────────────┘
    │
    ├─── authorized=true, require_approval=false ───→ ✅ 自动执行
    │
    ├─── authorized=true, require_approval=true ────→ ⚠️ 汇报用户，等确认
    │
    └─── authorized=false ───────────────────────────→ ❌ 告知用户，拒绝执行
```

### 授权预检示例

```python
def check_authorization(agent_id, policy_type, amount=None, category=None):
    """授权预检"""
    resp = requests.post(
        f"{BASE_URL}/acap/v1/agents/{agent_id}/policies/check",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={
            "payload": {
                "type": "auth.check",
                "data": {
                    "policy_type": policy_type,
                    "amount": amount,
                    "category": category,
                    "action_time": datetime.utcnow().isoformat()
                }
            }
        }
    )
    return resp.json()["payload"]["data"]

# 使用示例
auth = check_authorization("agt_xxx", "ACCEPT_DEAL", amount=2500000)
if auth["authorized"] and not auth.get("require_approval"):
    # 自动执行
    accept_deal("nego_abc")
else:
    # 汇报用户
    notify_user(f"需要您确认：成交金额 {amount}元")
```

---

## 3. 操作授权检查点

### 检查矩阵

| 操作 | 必须预检 | policy_type | 关键参数 |
|------|---------|-------------|----------|
| 发布意图 | ❌ 否 | - | 用户发起的，不需要检查 |
| 跟踪进度 | ❌ 否 | - | 只读操作，不需要检查 |
| 发起议价 | ❌ 否 | - | 用户发起的，不需要检查 |
| **接受成交** | ✅ **是** | `ACCEPT_DEAL` | amount, category |
| **拒绝成交** | ✅ **是** | `REJECT_DEAL` | amount, reason |
| **自动结算** | ✅ **是** | `AUTO_SETTLE` | amount, currency |
| **发起争议** | ✅ **是** | `DISPUTE` | amount, reason |
| 卖家报价 | ❌ 否 | - | 系统触发，按策略执行 |
| 卖家还价 | ❌ 否 | - | 按策略执行 |

### 接受成交（ACCEPT_DEAL）

```python
def decide_accept_deal(session_code, amount, category):
    """决定是否接受成交"""
    auth = check_authorization(MY_AGENT_ID, "ACCEPT_DEAL", amount, category)

    if auth["authorized"] and not auth.get("require_approval"):
        # 自动接受
        requests.post(
            f"{BASE_URL}/acap/v1/negotiations/{session_code}/accept",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return {"action": "auto_accept", "reason": "within_authorization"}
    elif auth.get("require_approval"):
        return {
            "action": "need_approval",
            "message": f"成交金额 {amount}元 需要您确认",
            "remaining_budget": auth.get("remaining_daily_budget")
        }
    else:
        return {
            "action": "reject",
            "reason": auth.get("reject_reason", "outside_authorization")
        }
```

### 自动结算（AUTO_SETTLE）

```python
def decide_auto_settle(settlement_code, amount):
    """决定是否自动结算"""
    auth = check_authorization(MY_AGENT_ID, "AUTO_SETTLE", amount)

    if auth["authorized"] and not auth.get("require_approval"):
        # 自动确认
        requests.post(
            f"{BASE_URL}/acap/v1/settlements/{settlement_code}/confirm",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return {"action": "auto_confirm", "reason": "within_authorization"}
    else:
        return {
            "action": "need_approval",
            "message": f"结算金额 {amount}元 需要您确认"
        }
```

---

## 4. 授权策略配置

### 查看当前策略

```python
# 查看所有策略
resp = requests.get(
    f"{BASE_URL}/acap/v1/agents/{MY_AGENT_ID}/policies",
    headers={"Authorization": f"Bearer {API_KEY}"}
)
policies = resp.json()["data"]

for p in policies:
    print(f"{p['policy_type']}: enabled={p['enabled']}")
```

### 授权响应字段说明

```json
{
  "authorized": true,           // 是否有权限执行
  "require_approval": false,    // 是否需要用户审批
  "reject_reason": null,        // 拒绝原因（当 authorized=false）
  "remaining_daily_budget": 5000000,  // 今日剩余预算（分）
  "remaining_daily_count": 8,   // 今日剩余操作次数
  "approval_required_reason": null  // 需要审批的原因
}
```

### 授权拒绝场景

| 拒绝原因 | 含义 | Agent 行为 |
|---------|------|-----------|
| `CATEGORY_NOT_ALLOWED` | 品类不在允许范围内 | 告知用户，建议调整品类 |
| `AMOUNT_EXCEEDS_LIMIT` | 金额超出单笔限额 | 告知用户，建议分拆订单 |
| `DAILY_BUDGET_EXHAUSTED` | 今日预算已用完 | 告知用户，建议明天再试 |
| `POLICY_DISABLED` | 授权策略已禁用 | 告知用户，需要重新配置 |
| `TIME_WINDOW_EXPIRED` | 不在有效时间窗口内 | 静默跳过 |

---

## 5. 降级规则与兜底

### 服务不可用时的降级策略

```python
def check_authorization_safe(agent_id, policy_type, amount):
    """
    安全的授权检查：服务不可用时的降级处理
    """
    try:
        auth = check_authorization(agent_id, policy_type, amount)
        return auth
    except requests.exceptions.RequestException as e:
        # 服务不可用时的降级策略

        # 保守策略：默认拒绝，要求人工确认
        return {
            "authorized": True,  # 不阻止交易
            "require_approval": True,  # 但要求人工确认
            "fallback": True,
            "message": "授权服务暂时不可用，需要您确认"
        }

        # 或者激进策略：默认允许
        # return {
        #     "authorized": True,
        #     "require_approval": False,
        #     "fallback": True
        # }
```

### 异常情况处理

| 异常 | 保守策略 | 激进策略 |
|------|---------|---------|
| 授权服务超时 | 要求人工确认 | 自动执行 |
| 授权服务 500 | 要求人工确认 | 自动执行 |
| 授权服务 404 | 要求人工确认 | 自动执行 |
| 网络不可达 | 要求人工确认 | 自动执行（次日记录） |

---

## 6. 自主决策示例

### 场景 1：买家自动接受报价

```python
def handle_quote_received(event):
    """收到卖家报价，决定是否接受"""
    session_code = event["ref_session_code"]
    offered_price = event.get("amount", 0)

    # 查询议价详情
    resp = requests.get(
        f"{BASE_URL}/acap/v1/negotiations/{session_code}",
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    negotiation = resp.json()["data"]

    # 获取品类信息
    category = negotiation.get("category", "通用")

    # 授权检查
    auth = check_authorization(MY_AGENT_ID, "ACCEPT_DEAL", offered_price, category)

    if not auth["authorized"]:
        return {
            "action": "reject",
            "message": f"报价 {offered_price}元超出授权范围",
            "reason": auth.get("reject_reason")
        }

    if auth.get("require_approval"):
        return {
            "action": "wait_user",
            "message": f"卖家报价 {offered_price}元，需要您确认是否接受"
        }

    # 授权范围内，自动接受
    requests.post(
        f"{BASE_URL}/acap/v1/negotiations/{session_code}/accept",
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    return {
        "action": "auto_accept",
        "message": f"已自动接受报价 {offered_price}元"
    }
```

### 场景 2：卖家自动响应议价

```python
def handle_negotiation_request(event):
    """收到议价请求，决定如何响应"""
    session_code = event["ref_session_code"]
    buyer_offer = event.get("offered_price", 0)

    # 查询议价详情
    resp = requests.get(
        f"{BASE_URL}/acap/v1/negotiations/{session_code}",
        headers={"Authorization": f"Bearer {API_KEY}"}
    )
    negotiation = resp.json()["data"]

    # 获取策略
    strategy = get_active_strategy()

    # 计算是否在可接受范围内
    if buyer_offer >= strategy["autoAcceptThreshold"]:
        # 自动接受
        requests.post(
            f"{BASE_URL}/acap/v1/negotiations/{session_code}/accept",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return {"action": "auto_accept", "price": buyer_offer}

    if negotiation["current_round"] >= negotiation["max_rounds"]:
        # 达到最大轮次，自动接受最新报价
        requests.post(
            f"{BASE_URL}/acap/v1/negotiations/{session_code}/accept",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return {"action": "timeout_accept", "price": buyer_offer}

    # 在可接受范围内，还价
    counter_price = calculate_counter(buyer_offer, strategy)
    requests.post(
        f"{BASE_URL}/acap/v1/negotiations/{session_code}/offers",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"price": counter_price, "message": "还价"}
    )
    return {"action": "counter", "price": counter_price}
```

---

## 7. 决策日志

记录每次决策，便于审计和问题排查：

```python
def log_decision(agent_id, operation, auth_result, action_taken):
    """记录决策日志"""
    log_entry = {
        "agent_id": agent_id,
        "operation": operation,
        "auth_result": auth_result,
        "action_taken": action_taken,
        "timestamp": datetime.utcnow().isoformat()
    }

    # 可以发送到审计服务或本地记录
    print(f"[Decision Log] {json.dumps(log_entry)}")

    # 示例输出：
    # [Decision Log] {
    #   "agent_id": "agt_xxx",
    #   "operation": "ACCEPT_DEAL",
    #   "auth_result": {"authorized": true, "require_approval": false},
    #   "action_taken": "auto_accept",
    #   "timestamp": "2026-04-29T10:00:00"
    # }
```

---

## 8. 策略建议

### L1 半托管推荐配置

```json
{
  "policy_type": "NEGOTIATE",
  "auto_authorize": true,
  "autoAcceptThreshold": 1000000,
  "maxSingleAmount": 5000000,
  "maxDailyAmount": 10000000,
  "maxDailyCount": 20,
  "allowedCategories": ["食品", "电子产品", "服装"],
  "timeWindowStart": "09:00",
  "timeWindowEnd": "21:00"
}
```

### 风险控制建议

1. **金额分层**：不同金额段设置不同策略
   - < 1 万：全托管，自动执行
   - 1-10 万：半托管，阈值内自动
   - > 10 万：全人工，用户确认

2. **品类分层**：高风险品类设置更严格
   - 低风险品类：宽松策略
   - 高风险品类：严格策略

3. **时段控制**：工作时间更宽松，非工作时间更保守

---

## 9. 下一步

- 查看 [skill-buyer.md](skill-buyer.md) 买家完整行为规范
- 查看 [skill-seller.md](skill-seller.md) 卖家完整行为规范
- 查看 [skill-account.md](skill-account.md) 授权策略配置方法
