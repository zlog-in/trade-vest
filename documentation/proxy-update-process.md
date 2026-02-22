# Proxy Update Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/proxy_update_process

## 概述

代理人（Proxy）更新流程，涵盖代理类型变更、作用域/监护权更新、客户产品关联等操作。根据代理类型不同，校验规则和文档要求有所差异。

## 前置条件

- 代理人状态为 `CREATED` 或 `ACTIVE`
- Partner 拥有更新授权
- 部分更新（scopeType、custodyType 变更）需提前准备支撑文档

## API 端点

```
PUT /roles/proxies/{proxyId}
```

## 代理类型一览

| 代理类型 | 适用实体类型 | 说明 |
|----------|-------------|------|
| `SIGNATORY` | `LEGAL_ENTITY` | 签署人 |
| `GUARDIAN` | `NATURAL_PERSON` | 监护人 |
| `GENERAL_POWER_OF_ATTORNEY` | — | 一般授权代理 |
| `INFORMATION_PROXY` | — | 信息代理 |
| `LIQUIDATOR` | — | 清算人 |
| `JOINT_ACCOUNT_HOLDER` | — | 联名账户持有人 |

## 可更新 / 不可更新字段

| 可更新 | 不可更新 |
|--------|----------|
| `proxyType`（仅 CREATED 状态） | `naturalPersonId` |
| `scopeType`（仅 SIGNATORY） | `entityId` |
| `custodyType`（仅 GUARDIAN） | `entityType` |
| `validityType`（仅 GENERAL_POWER_OF_ATTORNEY / INFORMATION_PROXY） | `status` |
| `customerProducts`（仅 GENERAL_POWER_OF_ATTORNEY / INFORMATION_PROXY） | |

## 整体流程

```
              Partner PUT Request
                       |
                       v
          +----------------------------+
          | 1. Retrieve Proxy          |
          |    Verify existence        |
          +----------------------------+
                       |
                       v
          +----------------------------+
          | 2. Synchronous Static      |
          |    Validation              |
          |  - OpenAPI schema check    |
          |  - Business logic rules    |
          |  - Document ID check       |
          +----------------------------+
                 |             |
               FAIL           PASS
                 |             |
                 v             v
           +---------+  +----------------------------+
           | 400/409 |  | 3. Publish Command Event   |
           | (reject)|  +----------------------------+
           +---------+        |
                              v
                  +----------------------------+
                  | 4. Async Business          |
                  |    Validation              |
                  |  - Proxy existence/status  |
                  |  - Proxy type validation   |
                  |  - Document validation     |
                  |  - Scope type validation   |
                  |  - Custody type validation |
                  |  - Customer product valid. |
                  +----------------------------+
                       |             |
                     FAIL           PASS
                       |             |
                       v             v
                 +-----------+  +----------------------------+
                 | Rejected  |  | 5. Update Proxy            |
                 | Webhook   |  +----------------------------+
                 | error     |        |
                 +-----------+        v
                              Webhook success
```

> [!NOTE] 此流程**不涉及人工审核（REVIEW）或 Admin Task**，仅有通过/拒绝两种结果。

## 校验体系

### 1. 同步静态校验（Immediate）

| 校验项 | 规则 |
|--------|------|
| 字段长度/格式/类型 | OpenAPI schema 校验 |
| 枚举值 | `proxyType`, `scopeType`, `custodyType`, `validityType` 枚举校验 |
| Proxy Type 兼容性 | 更新后的 `proxyType` 必须与 `entityType` 兼容 |
| Scope Type 限制 | 仅 `SIGNATORY` 可设置 `scopeType` |
| Custody Type 限制 | 仅 `GUARDIAN` 可设置 `custodyType` |
| Validity Type 兼容性 | 仅 `GENERAL_POWER_OF_ATTORNEY` / `INFORMATION_PROXY` 可更新 |
| Customer Product 限制 | 仅 `GENERAL_POWER_OF_ATTORNEY` / `INFORMATION_PROXY` 可更新 |
| Document ID | scopeType / custodyType 变更时必须提供 `documentId` |

### 2. 异步业务校验（Asynchronous）

```
              Async Validation Pipeline
                         |
                         v
            +-------------------------------+
            | Proxy Existence & Status      |
            | Must be CREATED or ACTIVE     |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Proxy Type Update Validation  |
            | (when proxyType changed)      |
            | - Only when status = CREATED  |
            | - Must be compatible with     |
            |   entityType                  |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Document Validation           |
            | (when documentId provided)    |
            | - Document exists & type      |
            |   matches requirement         |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Scope Type Validation         |
            | (SIGNATORY only)              |
            | - Signature authorization     |
            |   consistency check           |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Custody Type Validation       |
            | (GUARDIAN only)               |
            | - Single custody uniqueness   |
            | - Max 2 guardians per NP      |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Customer Product Validation   |
            | - Product IDs valid           |
            | - Proxy type supports         |
            |   product updates             |
            +-------------------------------+
```

## 各代理类型专属规则

### SIGNATORY（签署人）

仅适用于 `LEGAL_ENTITY` 类型的实体。

**scopeType 更新规则：**

| scopeType | 要求 |
|-----------|------|
| `INDIVIDUAL` | 自然人必须是该法人实体的**活跃法定代表人**，且 `soleSignatureAuthorized = true` |
| `JOINT` | 自然人必须是该法人实体的**活跃法定代表人**，且 `soleSignatureAuthorized = false` |

**文档要求：** scopeType 变更需提供 `documentId` 指向 `CURRENT_REGISTRY_EXTRACT`（resourceType = `LEGAL_ENTITY`）

### GUARDIAN（监护人）

仅适用于 `NATURAL_PERSON` 类型的实体。

| 规则 | 说明 |
|------|------|
| 最大监护人数 | 每个自然人最多 **2 个**监护人 |
| `SINGLE_CUSTODY` 排他性 | 若已有 `SINGLE_CUSTODY` 监护人，不可添加任何监护人，也不可将现有监护人变更为 `SINGLE_CUSTODY` |
| `JOINT_CUSTODY` 冲突 | 若已有 `JOINT_CUSTODY` 监护人，不可变更为 `SINGLE_CUSTODY` |

**custodyType 枚举：** `SINGLE_CUSTODY` / `JOINT_CUSTODY`

**文档要求：** custodyType 变更需提供 `documentId` 指向 `PROOF_OF_SINGLE_CUSTODY`（resourceType = `PROXY`）

> [!NOTE] 创建流程额外校验监护人年满 18 岁（成年要求），更新流程原文未提及此校验。

### GENERAL_POWER_OF_ATTORNEY / INFORMATION_PROXY

| 可更新项 | 说明 |
|----------|------|
| `validityType` | 必须与代理类型兼容 |
| `customerProducts` | 可更新客户产品关联 |

### LIQUIDATOR / JOINT_ACCOUNT_HOLDER

- **不可更新** `customerProducts`
- 原文未提及其他特殊规则

> [!NOTE] 创建流程中**所有代理类型均可分配 `customerProducts`**，但更新流程中仅 `GENERAL_POWER_OF_ATTORNEY` 和 `INFORMATION_PROXY` 可更新此字段。`GUARDIAN`、`SIGNATORY`、`LIQUIDATOR`、`JOINT_ACCOUNT_HOLDER` 创建后无法修改客户产品关联。

### proxyType 变更规则

- **仅当状态为 `CREATED` 时**允许变更 `proxyType`
- 变更后的类型必须与 `entityType` 兼容：
  - `SIGNATORY` → 仅 `LEGAL_ENTITY`
  - `GUARDIAN` → 仅 `NATURAL_PERSON`

> [!NOTE] 原文未说明 `proxyType` 变更时，已有的 `scopeType`、`custodyType`、`customerProducts` 等关联数据如何处理。

## 业务结果（3 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 数据格式错误、类型不兼容
- scopeType/custodyType 变更时缺少 `documentId`
- validityType 与 proxyType 不兼容

**后续操作：** 修正数据，确认类型兼容性，提供必需文档，重新提交

### Path 2: 异步校验失败 — 拒绝

**触发条件：**
- Proxy 不存在或状态不是 CREATED/ACTIVE
- 文档不存在或类型不匹配
- scopeType 与法定代表人签署授权不一致
- custodyType 与现有监护人冲突
- customerProducts ID 无效

**后续操作：** 根据 Webhook 中的错误详情修正问题，重新提交

### Path 3: 更新成功

**触发条件：** 所有校验通过

**影响：** 代理人数据更新成功，Partner 通过 Webhook 收到成功通知

## 已知状态值

| 状态 | 含义 |
|------|------|
| `CREATED` | 初始状态，允许变更 proxyType |
| `ACTIVE` | 活跃状态，不允许变更 proxyType |

> [!NOTE] 原文仅列出 CREATED 和 ACTIVE 两种状态，未提及其他可能的状态值。

## API Reference

- [Proxies](https://docs.tradevest.ai/api-reference/roles/proxies) — 代理人接口
- [Roles Schemas](https://docs.tradevest.ai/api-reference/roles/schemas) — 角色数据结构定义
