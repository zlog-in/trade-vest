# Proxy Create Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/proxy_create_process

## 概述

代理人（Proxy）是代表其他实体行事的自然人。创建流程通过校验确保代理类型与实体类型兼容、满足各类型专属业务规则。

关键要点：
- 代理人的自然人和被代理实体**不可为同一人**（`naturalPersonId ≠ entityId`）
- 流程**不涉及人工审核（REVIEW）**，仅有通过/拒绝两种结果

## 前置条件

- 代理人对应的自然人已存在
- 被代理的实体（自然人、法人实体或联名人）已存在
- 已收集代理所需信息（类型、作用域/监护权、有效性类型等）
- Partner 拥有创建授权

## API 端点

```
POST /roles/proxies
```

## 必填字段

| 字段 | 说明 | 条件 |
|------|------|------|
| `naturalPersonId` | 担任代理人的自然人 ID | 必填 |
| `entityId` | 被代理的实体 ID | 必填，且不可等于 `naturalPersonId` |
| `proxyType` | 代理类型 | 必填 |
| `validityType` | 有效性类型 | 必填，取值受 proxyType 约束 |
| `scopeType` | 作用域类型 | `SIGNATORY` 时必填 |
| `custodyType` | 监护权类型 | `GUARDIAN` 时必填 |
| `customerProducts` | 客户产品 ID 列表 | 可选，所有代理类型均支持 |

> [!NOTE] `entityType` 不在请求体中提交，系统从已有实体记录中获取。

## 代理类型与实体类型兼容性

| 代理类型 | 适用实体类型 | 说明 |
|----------|-------------|------|
| `SIGNATORY` | 仅 `LEGAL_ENTITY` | 签署人 |
| `GUARDIAN` | 仅 `NATURAL_PERSON` | 监护人 |
| `GENERAL_POWER_OF_ATTORNEY` | 无显式限制 | 一般授权代理 |
| `INFORMATION_PROXY` | 无显式限制 | 信息代理 |
| `LIQUIDATOR` | 无显式限制 | 清算人 |
| `JOINT_ACCOUNT_HOLDER` | 无显式限制 | 联名账户持有人 |

## validityType 兼容性矩阵

| 代理类型 | 允许的 validityType |
|----------|-------------------|
| `GUARDIAN` | `UNTIL_LEGAL_AGE` |
| `SIGNATORY` | `UNLIMITED` |
| `GENERAL_POWER_OF_ATTORNEY` | `UNLIMITED`, `IN_CASE_OF_DEATH`, `UNTIL_CASE_OF_DEATH` |
| `INFORMATION_PROXY` | `UNLIMITED`, `UNTIL_CASE_OF_DEATH` |
| `LIQUIDATOR` | `UNLIMITED` |
| `JOINT_ACCOUNT_HOLDER` | `UNLIMITED` |

## 整体流程

```
              Partner POST Request
                       |
                       v
          +----------------------------+
          | 1. Synchronous Static      |
          |    Validation              |
          |  - OpenAPI schema check    |
          |  - Business logic rules    |
          +----------------------------+
                 |             |
               FAIL           PASS
                 |             |
                 v             v
           +---------+  +----------------------------+
           | 400/409 |  | 2. Publish Command Event   |
           | (reject)|  +----------------------------+
           +---------+        |
                              v
                  +----------------------------+
                  | 3. Store Data              |
                  |    status = RECEIVED       |
                  +----------------------------+
                              |
                              v
                  +----------------------------+
                  | 4. Async Business          |
                  |    Validation              |
                  |  - NP status check         |
                  |  - Entity status check     |
                  |  - Type-specific rules     |
                  |  - Customer product check  |
                  +----------------------------+
                       |             |
                     FAIL           PASS
                       |             |
                       v             v
                 +-----------+  +----------------------------+
                 | Rejected  |  | 5. Create Proxy            |
                 | Webhook   |  |    status = CREATED        |
                 | error     |  +----------------------------+
                 +-----------+        |
                                      v
                              Webhook success
```

## 校验体系

### 1. 同步静态校验（Immediate）

| 校验项 | 规则 |
|--------|------|
| 字段长度/格式/类型 | OpenAPI schema 校验 |
| 枚举值 | `proxyType`, `scopeType`, `custodyType`, `validityType` 枚举约束 |
| Proxy Type 兼容性 | `proxyType` 必须与实体 `entityType` 兼容 |
| Scope Type | `SIGNATORY` 时必填 |
| Custody Type | `GUARDIAN` 时必填 |
| Validity Type 兼容性 | 必须与 `proxyType` 匹配（见上方矩阵） |
| 自代理防止 | `naturalPersonId` 和 `entityId` 必须不同 |

### 2. 异步业务校验（Asynchronous）

```
              Async Validation Pipeline
                         |
                         v
            +-------------------------------+
            | Natural Person Status         |
            | Must be ACTIVE or CREATED     |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Related Entity Status         |
            | Must be ACTIVE or CREATED     |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Proxy Type-Specific           |
            | Validation                    |
            | (see rules per type below)    |
            +-------------------------------+
                         |
                         v
            +-------------------------------+
            | Customer Product Validation   |
            | All product IDs must exist    |
            +-------------------------------+
```

## 各代理类型专属规则

### SIGNATORY（签署人）

仅适用于 `LEGAL_ENTITY`。

| 规则 | 说明 |
|------|------|
| 法定代表人身份 | 自然人必须是该法人实体的**活跃法定代表人** |
| `INDIVIDUAL` scope | 要求法定代表人 `soleSignatureAuthorized = true` |
| `JOINT` scope | 要求法定代表人 `soleSignatureAuthorized = false` |

### GUARDIAN（监护人）

仅适用于 `NATURAL_PERSON`。

| 规则 | 说明 |
|------|------|
| 成年要求 | 监护人自然人必须年满 **18 岁** |
| 最大监护人数 | 每个自然人最多 **2 个**监护人 |
| `SINGLE_CUSTODY` 排他性 | 若已有 `SINGLE_CUSTODY` 监护人，**不可再添加任何监护人** |
| `JOINT_CUSTODY` 冲突 | 若已有 `JOINT_CUSTODY` 监护人，**不可添加** `SINGLE_CUSTODY` 监护人 |
| `JOINT_CUSTODY` 要求 | 需恰好 **2 个**监护人 |

### GENERAL_POWER_OF_ATTORNEY / INFORMATION_PROXY

- 支持多种 `validityType`（见兼容性矩阵）
- 可分配 `customerProducts`

### LIQUIDATOR / JOINT_ACCOUNT_HOLDER

- `validityType` 仅支持 `UNLIMITED`
- 可分配 `customerProducts`
- 原文未提及其他特殊规则

## 业务结果（3 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 必填字段缺失
- proxyType 与 entityType 不兼容
- SIGNATORY 缺少 scopeType / GUARDIAN 缺少 custodyType
- `naturalPersonId` 与 `entityId` 相同

**后续操作：** 修正数据，确认类型兼容性，重新提交

### Path 2: 异步校验失败 — 拒绝

**触发条件：**
- 自然人或实体状态非 ACTIVE/CREATED
- GUARDIAN 未成年、监护权冲突、超过监护人上限
- SIGNATORY 非法定代表人、签署授权不匹配
- customerProducts ID 无效

**后续操作：** 根据 Webhook 错误详情修正问题，重新提交

### Path 3: 创建成功 — CREATED

**触发条件：** 所有校验通过

**影响：** 代理人创建成功，状态 `CREATED`，Partner 通过 Webhook 收到成功通知

## 已知状态值

| 状态 | 含义 |
|------|------|
| `RECEIVED` | 初始状态，数据已接收待异步处理 |
| `CREATED` | 创建成功 |

## 与更新流程的主要差异

| 对比项 | Create | Update |
|--------|--------|--------|
| HTTP 方法 | `POST` | `PUT` |
| `proxyType` 变更 | 创建时指定 | 仅 CREATED 状态可变更 |
| `validityType` 变更 | 创建时指定，所有类型必填 | 仅 GPOA / INFO_PROXY 可更新 |
| `customerProducts` | 所有代理类型均可分配 | 仅 GPOA / INFO_PROXY 可更新 |
| scopeType/custodyType 文档 | 无文档要求 | 变更时需提供 `documentId` |
| GUARDIAN 成年校验 | 有（18+） | 未提及 |
| 自代理防止 | 校验 `NP ID ≠ Entity ID` | 不适用（字段不可更新） |

## API Reference

- [Proxies](https://docs.tradevest.ai/api-reference/roles/proxies) — 代理人接口
- [Roles Schemas](https://docs.tradevest.ai/api-reference/roles/schemas) — 角色数据结构定义
