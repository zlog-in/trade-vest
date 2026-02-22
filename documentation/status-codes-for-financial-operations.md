# Status Codes for Financial Operations

> Source: https://docs.tradevest.ai/documentation/status_codes_for_financial_operations

## 概述

金融操作涉及三类对象：**Order（订单）**、**Transfer（转账）**、**Transaction（交易记录）**。每类对象拥有独立的状态码体系，其中 Transaction 的状态由 Order / Transfer 的状态映射而来。

---

## 1. Order Status Codes（订单状态码）

### 状态定义

| 状态码 | 说明 | 可能的前置状态 | 可能的后续状态 |
|--------|------|---------------|---------------|
| **RECEIVED** | 订单请求已通过 API 层 | N/A | N/A |
| **INVALID** | 订单请求未通过校验 | N/A | N/A |
| **PENDING** | 订单已被系统接收，排队等待处理 | N/A | FILLED, CANCELED, REJECTED |
| **CANCELED** | 订单已被用户或系统取消 | PENDING, CORRECTED | N/A |
| **REJECTED** | 订单被系统拒绝 | PENDING | N/A |
| **FILLED** | 订单执行成功，交易完成 | PENDING | SETTLED |
| **CORRECTED** | 订单发生了更正 | FILLED | CANCELED |
| **SETTLED** | 订单已完全结算，所有相关金融交易完成 | FILLED | N/A |

> [!WARNING]
> 原文中 RECEIVED 的后续状态为 N/A，PENDING 的前置状态为 N/A，但从序列图可知流程为 RECEIVED → PENDING。这两个字段应分别填写 PENDING 和 RECEIVED。

### 状态流转图

```
                      [API Request]
                           |
                  +--------+--------+
                  |                 |
             [valid]           [invalid]
                  |                 |
                  v                 v
            +-----------+     +-----------+
            | RECEIVED  |     |  INVALID  |
            +-----+-----+     +-----------+
                  |
                  v
            +-----------+
            |  PENDING  |
            +-----+-----+
                  |
         +--------+--------+
         |        |        |
    [executed] [canceled] [rejected]
         |        |        |
         v        v        v
   +---------+ +----------+ +----------+
   | FILLED  | | CANCELED | | REJECTED |
   +----+----+ +----------+ +----------+
        |            ^
   +----+----+       |
   |         |  [canceled]
   v         v       |
+--------+ +----------+
|SETTLED | |CORRECTED |
+--------+ +----------+
```

**终态：** INVALID, REJECTED, CANCELED, SETTLED

### 序列图 — 订单创建与执行

```
  Partner                              Platform
     |                                    |
     | POST Create MARKET order (10 BTC)  |
     |----------------------------------->|
     | Order-id (status RECEIVED)         |
     |<-----------------------------------|
     |                                    |
     |        [Alt: order executed]        |
     |                                    |--- Order processing
     |                                    |--- Balance and position updated
     | Order status FILLED                |
     |<-----------------------------------|
     |                                    |--- Settlement completed
     | Order status SETTLED               |
     |<-----------------------------------|
     |                                    |
     |   [Opt: insufficient balance/      |
     |         position]                  |
     | Order status REJECTED              |
     |<-----------------------------------|
     |                                    |
```

**说明：**
- `[Alt]` — 订单执行成功：RECEIVED → PENDING → FILLED → SETTLED
- `[Opt]` — 余额/仓位不足：PENDING → REJECTED

> [!NOTE]
> 序列图仅展示了成功（FILLED → SETTLED）和拒绝（REJECTED）路径。CANCELED 和 CORRECTED 路径仅在状态表中描述，未在图中体现。

---

## 2. Transfer Status Codes（转账状态码）

### 状态定义

| 状态码 | 说明 |
|--------|------|
| **RECEIVED** | 转账请求已通过 API 层 |
| **INVALID** | 转账请求未通过校验 |
| **PENDING** | 转账已创建但尚未处理，正在执行操作或检查 |
| **REJECTED** | 转账被拒绝（详见错误码） |
| **PROCESSED** | 转账处理成功 |

> [!NOTE]
> 原文中 Transfer 状态表仅包含状态码和说明两列，不像 Order 状态表那样提供前置/后续状态列。以下流转图根据序列图推导。

### 状态流转图

```
                      [API Request / SEPA]
                           |
                  +--------+--------+
                  |                 |
             [valid]           [invalid]
                  |                 |
                  v                 v
            +-----------+     +-----------+
            | RECEIVED  |     |  INVALID  |
            +-----+-----+     +-----------+
                  |
                  v
            +-----------+
            |  PENDING  |
            +-----+-----+
                  |
            +-----+-----+
            |           |
       [processed]  [rejected]
            |           |
            v           v
      +-----------+ +-----------+
      | PROCESSED | | REJECTED  |
      +-----------+ +-----------+
```

**终态：** INVALID, REJECTED, PROCESSED

### 序列图 — 转账流程

转账分为 **Incoming（入账）** 和 **Outgoing（出账）** 两种场景，涉及三方：

| 角色 | 说明 |
|------|------|
| **Partner** | 合作方 — 发起出账请求、接收通知 |
| **Platform** | Tradevest 平台 — 处理转账、更新余额 |
| **SEPA** | SEPA 支付网络 — 外部银行转账通道 |

```
  Partner                      Platform                       SEPA
     |                            |                             |
     |                      [Incoming transfer]                 |
     |                            | Transfer notification       |
     |                            |<----------------------------|
     |                            |--- Balance updated          |
     | Transfer notif             |                             |
     | (status PROCESSED)         |                             |
     |<---------------------------|                             |
     |                            |                             |
     |                      [Outgoing transfer]                 |
     | Create Transfer            |                             |
     |--------------------------->|                             |
     | Transfer created           |                             |
     | (transfer id,              |                             |
     |  status RECEIVED)          |                             |
     |<---------------------------|                             |
     |                            |--- Balance updated          |
     |                            | Create Transfer             |
     |                            |---------------------------->|
     |                            |            (ack)            |
     |                            |<- - - - - - - - - - - - - - |
     | Transfer notif             |                             |
     | (status PENDING)           |                             |
     |<---------------------------|                             |
     |                            | Transfer notification       |
     |                            |<----------------------------|
     |                            |                             |
     |      [Alt: transfer processed]                           |
     | Transfer notif             |                             |
     | (status PROCESSED)         |                             |
     |<---------------------------|                             |
     |                            |                             |
     |      [Opt: insufficient balance]                         |
     | Transfer notif             |                             |
     | (status REJECTED)          |                             |
     |<---------------------------|                             |
     |                            |                             |
```

**Incoming（入账）流程：**
1. SEPA 向 Platform 发送转账通知
2. Platform 更新余额
3. Platform 向 Partner 发送通知（status = PROCESSED）

**Outgoing（出账）流程：**
1. Partner 调用 API 创建转账 → 获得 transfer id（status = RECEIVED）
2. Platform 更新余额，向 SEPA 发起转账
3. SEPA 返回确认（虚线箭头）→ Platform 通知 Partner（status = PENDING）
4. SEPA 处理完成后通知 Platform
5. `[Alt]` 成功：Platform 通知 Partner（status = PROCESSED）
6. `[Opt]` 余额不足：Platform 通知 Partner（status = REJECTED）

---

## 3. Transaction Status Codes（交易记录状态码）

Transaction 是对 Order 和 Transfer 最终结果的统一视图，状态由源对象状态映射而来。

### 状态定义

| 状态码 | 说明 |
|--------|------|
| **COMPLETED** | 交易已完全处理或（对于订单）执行成功 |
| **MODIFIED** | 交易详情在初始完成后被修改或更新 |

### Order Status → Transaction Status 映射

| Order 状态 | Transaction 状态 |
|------------|-----------------|
| RECEIVED | N/A |
| INVALID | N/A |
| PENDING | N/A |
| CANCELED | N/A |
| REJECTED | N/A |
| **FILLED** | **COMPLETED** |
| **CORRECTED** | **MODIFIED** |
| **SETTLED** | **COMPLETED** |

### Transfer Status → Transaction Status 映射

| Transfer 状态 | Transaction 状态 |
|---------------|-----------------|
| RECEIVED | N/A |
| INVALID | N/A |
| PENDING | N/A |
| REJECTED | N/A |
| **PROCESSED** | **COMPLETED** |

**规律：** 仅成功终态（FILLED / SETTLED / PROCESSED）和更正状态（CORRECTED）会生成 Transaction 记录。中间态和失败态不产生 Transaction。

---

## 相关文档

- [Partner Webhooks](./partner-webhooks.md) — Webhook 通信模型（状态变更通知机制）
- [Overview](./overview.md) — API 全局状态概念（RECEIVED / INVALID / REJECTED）

### API Reference

- [Asset Management](https://docs.tradevest.ai/api-reference/asset-management) — 资产管理 / 交易接口总览
- [Digital Assets — Orders](https://docs.tradevest.ai/api-reference/asset-management/digital-assets/create-order) — 数字资产订单接口
- [Transfers](https://docs.tradevest.ai/api-reference/transfers/transfers) — 转账接口
- [Transfers Schemas](https://docs.tradevest.ai/api-reference/transfers/schemas) — 转账数据结构定义
- [Transaction History](https://docs.tradevest.ai/api-reference/transaction-history) — 交易历史查询接口
- [Transaction History Schemas](https://docs.tradevest.ai/api-reference/transaction-history/schemas) — 交易记录数据结构定义
