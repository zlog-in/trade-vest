# Offboarding

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/offboarding

## 概述

离场（Offboarding）是将客户、代理人或受益所有人从平台移除的流程。与入驻类似，离场遵循**实体-角色分离**架构，通过两个独立的 API 路径处理：

- **角色离场（Role Offboarding）** — 处理客户（Customer）和代理人（Proxy）
- **实体离场（Entity Offboarding）** — 处理受益所有人（Beneficial Owner）

## API 端点

| 端点 | 方法 | 适用类型 | 请求体字段 |
|------|------|----------|-----------|
| `/roles/offboardings` | `POST` | Customer / Proxy | `roleType`, `roleId`, `reason` |
| `/entities/offboardings` | `POST` | Beneficial Owner | `entityType`, `entityId`, `reason` |

**`POST /roles/offboardings` 请求体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `roleType` | string | 枚举：`CUSTOMER` 或 `PROXY` |
| `roleId` | UUID | 目标角色 ID |
| `reason` | string | 离场原因（见下方枚举） |

**`POST /entities/offboardings` 请求体：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `entityType` | string | 枚举：`BENEFICIAL_OWNER` |
| `entityId` | UUID | 目标实体 ID |
| `reason` | string | 离场原因（见下方枚举） |

## 三种离场类型

| 类型 | 端点 | 适用实体 |
|------|------|----------|
| **Customer Offboarding** | `POST /roles/offboardings` | NP / LE / JP 的客户角色 |
| **Proxy Offboarding** | `POST /roles/offboardings` | 代理人角色（非 LIQUIDATOR） |
| **Beneficial Owner Offboarding** | `POST /entities/offboardings` | 受益所有人实体 |

## 关闭类别与离场原因

### 两种关闭类别

| 类别 | 说明 | 取消日期 |
|------|------|----------|
| **Ordinary**（常规） | 计划性离场 | 发起日期 + **2 个月** |
| **Immediate**（即时） | 紧急离场 | 离场执行**当日** |

### 离场原因枚举

| 原因 | 类别 | 说明 |
|------|------|------|
| `CANCELLATION_BY_PARTNER_ORDINARY` | Ordinary | Partner 发起的常规取消 |
| `CUSTOMER_REQUEST_IMMEDIATE` | Immediate | 客户主动请求即时取消 |
| `CANCELLATION_BY_PARTNER_IMMEDIATE` | Immediate | Partner 发起的即时取消 |
| `DEATH_IMMEDIATE` | Immediate | 死亡 |
| `DISSOLUTION_IMMEDIATE` | Immediate | 公司解散 |
| `INSOLVENCY_IMMEDIATE` | Immediate | 破产 |
| `COMPANY_STRUCTURE_CHANGES_IMMEDIATE` | Immediate | 公司结构变更 |

### 原因适用性矩阵

| 原因 | NP Customer | JP Customer | LE Customer | Proxy | BO |
|------|:-----------:|:-----------:|:-----------:|:-----:|:--:|
| `CANCELLATION_BY_PARTNER_ORDINARY` | ✓ | ✓ | ✓ | ✗ | ✗ |
| `CUSTOMER_REQUEST_IMMEDIATE` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `CANCELLATION_BY_PARTNER_IMMEDIATE` | ✓ | ✓ | ✓ | ✓ | ✗ |
| `DEATH_IMMEDIATE` | ✓ | ✗ | ✗ | ✓ | ✗ |
| `DISSOLUTION_IMMEDIATE` | ✗ | ✗ | ✓ | ✗ | ✗ |
| `INSOLVENCY_IMMEDIATE` | ✗ | ✗ | ✓ | ✗ | ✗ |
| `COMPANY_STRUCTURE_CHANGES_IMMEDIATE` | ✗ | ✗ | ✓ | ✓ * | ✓ |

> \* `COMPANY_STRUCTURE_CHANGES_IMMEDIATE` 对 Proxy 仅适用于 **Signatory** 类型。

> [!NOTE] Beneficial Owner 仅支持一种离场原因：`COMPANY_STRUCTURE_CHANGES_IMMEDIATE`。

## 离场状态流转

```
  Request
     |
     v
  INVALID -----> (validation failed at initiation, process does not start)

  Request
     |
     v
  CREATED ---------> PENDING ---------> APPROVED
  (initiated)       (processing)        (completed,
                        |                entity/role
                        |                offboarded)
                        |
                        +-------> REJECTED
                        |         (validation failed
                        |          during execution)
                        |
                        +-------> CANCELED
                                  (Ordinary only)
```

| 状态 | 含义 |
|------|------|
| `INVALID` | 初始数据无效，流程未启动 |
| `CREATED` | 离场流程已创建 |
| `PENDING` | 处理中（Ordinary 等待调度器触发） |
| `APPROVED` | 离场完成，实体/角色已移除 |
| `REJECTED` | 执行阶段校验失败 |
| `CANCELED` | 仅适用于 Ordinary 离场 |

## 处理流程

### Immediate 离场流程

```
  POST offboarding (Immediate reason)
                |
                v
  +------------------------------------------+
  | 1. All Validations Execute Immediately   |
  |    - Status eligibility                  |
  |    - Existence verification              |
  |    - Reason validity                     |
  |    - Product closure check               |
  |    - Custody / proxy type checks         |
  +------------------------------------------+
           |                |
         FAIL             PASS
           |                |
           v                v
     INVALID /        +------------------+
     REJECTED         | 2. Execute       |
                      |    Offboarding   |
                      | cancellationDate |
                      | = today          |
                      +------------------+
                              |
                              v
                      +------------------+
                      | 3. Cascade       |
                      |    Effects       |
                      | (auto-offboard   |
                      |  related roles)  |
                      +------------------+
                              |
                              v
                         APPROVED
                      (Webhook notifications)
```

### Ordinary 离场流程

```
  POST offboarding (CANCELLATION_BY_PARTNER_ORDINARY)
                |
                v
  +------------------------------------------+
  | 1. Initial Validations                   |
  |    - Status eligibility                  |
  |    - Existence verification              |
  |    - Reason validity                     |
  |    (product closure NOT checked yet)     |
  +------------------------------------------+
           |                |
         FAIL             PASS
           |                |
           v                v
     INVALID /        +------------------+
     REJECTED         | CREATED          |
                      | cancellationDate |
                      | = today + 2 mo   |
                      +------------------+
                              |
                              v
                      +------------------+
                      | PENDING          |
                      | Wait for         |
                      | scheduler        |
                      | (2 months)       |
                      +------------------+
                              |
                     (scheduler triggers)
                              |
                              v
                      +------------------+
                      | 2. Re-validate   |
                      |    ALL rules     |
                      |    including     |
                      |    product       |
                      |    closure check |
                      +------------------+
                           |        |
                         FAIL     PASS
                           |        |
                           v        v
                      REJECTED   APPROVED
                                 (Webhook
                                  notifications)
```

> [!WARNING] **Ordinary 离场的产品未关闭问题：** 若调度器触发时客户产品仍未全部关闭（CANCELLED），流程保持 PENDING 状态。原文提及目前**不支持恢复（resumption）机制**，即无法自动重试。

## 前置条件

Partner 在发起离场前须确保：
- 所有客户产品（Customer Products）已处于 `CANCELLED` 状态

## 校验规则

### 可离场状态

| 类型 | 允许的状态 |
|------|-----------|
| **Customer** | `ACTIVE`, `ESTATE`, `SUSPENDED`, `SUSPENDED_COMPLIANCE` |
| **Proxy** | `ACTIVE`, `SUSPENDED`, `SUSPENDED_COMPLIANCE` |
| **Beneficial Owner** | `ACTIVE`, `SUSPENDED_COMPLIANCE` |

### Customer Offboarding 校验

| 校验项 | 规则 |
|--------|------|
| 状态 | 须为 `ACTIVE` / `ESTATE` / `SUSPENDED` / `SUSPENDED_COMPLIANCE` |
| 存在性 | 数据库中须存在该记录 |
| 离场原因 | 须匹配客户实体类型（NP / LE / JP）的有效原因 |
| 客户产品 | 所有产品须处于 `CANCELLED` 状态 |
| 监护权校验 | 当离场的是 Guardian 的客户时，剩余 Guardian 的 `custodyType` 须为 `SINGLE_CUSTODY` |

### Proxy Offboarding 校验

| 校验项 | 规则 |
|--------|------|
| 状态 | 须为 `ACTIVE` / `SUSPENDED` / `SUSPENDED_COMPLIANCE` |
| 存在性 | 数据库中须存在该记录 |
| 离场原因 | 须匹配 Proxy 类型的有效原因 |
| 代理类型 | **不可为 `LIQUIDATOR`**（清算人不可通过此流程离场） |
| 监护权校验 | 离场 Guardian 时，其关联客户的剩余 Guardian 须为 `SINGLE_CUSTODY` |

### Beneficial Owner Offboarding 校验

| 校验项 | 规则 |
|--------|------|
| 状态 | 须为 `ACTIVE` / `SUSPENDED_COMPLIANCE` |
| 存在性 | 数据库中须存在该记录 |
| 离场原因 | 仅支持 `COMPANY_STRUCTURE_CHANGES_IMMEDIATE` |

## 级联效应（Cascading Effects）

Customer Offboarding 触发自动级联：

```
  Customer Offboarding
          |
          v
  +------------------------------------------+
  | 1. Offboard Customer Role                |
  +------------------------------------------+
          |
          v
  +------------------------------------------+
  | 2. Auto-offboard ALL associated:         |
  |    - Proxies (all types)                 |
  |    - Beneficial Owners (if LE customer)  |
  +------------------------------------------+
          |
          v
  +------------------------------------------+
  | 3. Entity Offboarding Check              |
  |    For each offboarded role's entity:    |
  |    - Has other active roles? → STAY      |
  |    - No other active roles? → OFFBOARD   |
  +------------------------------------------+
```

**关键规则：**
- Customer 离场时，所有关联的 Proxy 和 Beneficial Owner **自动离场**
- 被离场的 Proxy / BO 的实体（自然人等）仅在**无其他活跃角色**时才会同步离场
- 若该实体在其他客户下仍有活跃角色，则实体保持活跃

## Webhook 通知

离场完成后，Partner 通过以下 Webhook 接收通知：

| 通知类型 | 说明 |
|----------|------|
| Offboarding Notification | 离场流程状态 |
| Customer Notification | 客户角色状态变更 |
| Natural Person Notification | 自然人实体状态变更 |
| Joint Person Notification | 联名人实体状态变更 |
| Legal Entity Notification | 法人实体状态变更 |
| Beneficial Owner Notification | 受益所有人状态变更 |
| Proxy Notification | 代理人角色状态变更 |

## 注意事项与内容缺口

> [!NOTE] **LIQUIDATOR 限制：** Proxy 离场明确排除 `LIQUIDATOR` 类型，但原文未说明清算人通过何种途径离场。

> [!NOTE] **CANCELED 状态：** 仅适用于 Ordinary 离场，但触发条件（何时从 PENDING 转为 CANCELED）在原文中**未明确定义**。

> [!NOTE] **Guardian 离场影响：** 原文提及 Guardian 离场时需校验剩余 Guardian 的 `custodyType`，但**未详述离场后对未成年客户的具体影响**（如是否需要重新指定监护人）。

> [!NOTE] **ESTATE 状态：** 作为 Customer 可离场状态之一出现，但其转换条件在离场文档中未进一步说明（Onboarding 文档中同样缺失）。

## 相关文档

- [Onboarding](./onboarding.md) — 入驻流程、三种入驻类型、异步校验、KYC
- [Role Management](./role-management.md) — 角色类型与分配流程
- [Proxy Create Process](./proxy-create-process.md) — 代理人创建校验与规则（含 Guardian 监护权规则）
- [Proxy Update Process](./proxy-update-process.md) — 代理人更新流程
- [Document Management](./document-management.md) — 文档类型与状态生命周期

### API Reference

- [Entities Offboarding](https://docs.tradevest.ai/api-reference/entities/offboarding) — 实体离场接口（BO Offboarding）
- [Roles Offboarding](https://docs.tradevest.ai/api-reference/roles/offboarding) — 角色离场接口（Customer / Proxy Offboarding）
