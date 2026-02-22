# Beneficial Owner Update Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/beneficial_owner_update_process

## 概述

受益所有人更新流程，用于修改已有受益所有人的数据，通过校验确保数据完整性与合规性。

关键要点：
- 所有受益所有人类型均为 **`REAL_UBO_25`**，**类型不可更新**
- AML/KYC 合规验证在 **Onboarding 阶段**单独进行，不在更新流程中
- 与创建流程不同，更新流程**不涉及重复检测、相似人员匹配或人工审核（REVIEW）环节**

## 前置条件

- 受益所有人已存在且状态适合更新
- 已收集需要更新的信息
- Partner 拥有更新该受益所有人的授权

> [!NOTE] 原文未明确列出受益所有人需处于何种状态才允许更新（如 CREATED、ACTIVE 等），仅笼统提及 "appropriate status"。

## API 端点

```
PATCH /entities/beneficial-owners/{beneficialOwnerId}
```

## 可更新字段

| 类别 | 字段 |
|------|------|
| 个人信息 | `firstName`, `lastName`, `birthDay`, `birthPlace`, `birthCountry` |
| 联系信息 | `mainAddress`, `nationalities` |
| 税务信息 | `taxDetails`, `isUsNationality` |
| 持股信息 | `uboRelationship`, `share`, `votingRights` |
| FATCA 信息 | `fatcaControllingPerson` |

## 整体流程

```
              Partner PATCH Request
                       |
                       v
          +---------------------------+
          | 1. Retrieve Beneficial    |
          |    Owner by UUID          |
          +---------------------------+
                       |
                       v
          +---------------------------+
          | 2. Synchronous Static     |
          |    Validation             |
          |    - OpenAPI schema check |
          |    - Business logic rules |
          +---------------------------+
                 |             |
               FAIL           PASS
                 |             |
                 v             v
           +---------+  +---------------------------+
           | 400/409 |  | 3. Publish Command Event  |
           | (reject)|  +---------------------------+
           +---------+        |
                              v
                  +---------------------------+
                  | 4. Async Business         |
                  |    Validation             |
                  |  - BO existence & access  |
                  |  - Legal entity status    |
                  |  - Data validation        |
                  |  - Geographic compliance  |
                  |  - FATCA validation       |
                  +---------------------------+
                       |             |
                     FAIL           PASS
                       |             |
                       v             v
                  +---------+  +---------------------------+
                  | Update  |  | 5. Update Person Basic    |
                  | halted  |  |    Data (if provided)     |
                  | Webhook |  +---------------------------+
                  | error   |        |
                  +---------+        v
                              +---------------------------+
                              | 6. Update Beneficial      |
                              |    Owner record           |
                              +---------------------------+
                                     |
                                     v
                              Webhook success notification
```

## 校验体系

### 1. 同步静态校验（Immediate）

请求到达时立即执行，失败直接返回错误。

#### OpenAPI Schema 层校验

| 校验项 | 规则 |
|--------|------|
| 字段长度 | `firstName` ≤ 255, `lastName` ≤ 255 |
| 格式校验 | `firstName` 正则: `^\S+( \S+)*$` |
| 数据类型 | 各字段类型校验 |
| `share` | 0-100%, 精度 0.01, > 0 |
| `votingRights` | 0-100%, 精度 0.01, > 0 |
| `nationalities` | 至少 1 项 (minItems: 1) |

#### 业务逻辑校验

| 校验项 | 规则 |
|--------|------|
| Person Basic Data | 个人信息格式与结构校验 |
| `uboRelationship` | 枚举: `DIRECTLY_HOLDING_25` / `INDIRECTLY_HOLDING_25` / `DOMINANT_INFLUENCE_OVER_SHARE_CAPITAL` |
| Share/Voting vs UBO 一致性 | 综合**当前值与更新值**进行校验（详见下方规则） |
| 地址 | 地址格式校验 |

### 2. 异步业务校验（Asynchronous）

端点返回成功响应后异步执行，结果通过 **Webhook** 通知。

| 步骤 | 校验内容 |
|------|----------|
| BO 存在性校验 | 通过 UUID 检索，确认归属正确的 Partner |
| 法人实体状态校验 | 关联法人实体状态必须为 `ACTIVE` 或 `CREATED` |
| BO 更新数据校验 | 格式检查、UBO 关系与 share/votingRights 一致性（综合当前值与更新值） |
| 地域合规校验 | 更新后的地址必须在白名单国家内 |
| FATCA 校验 | 更新后的 FATCA controlling person 状态与法人实体 FATCA 分类一致 |
| Person Basic Data 更新 | 若提供了个人信息字段，通过当前 BO 的 global ID 更新全局人员基础数据 |

## 业务结果（3 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 受益所有人不存在或不可访问
- 个人信息/联系信息格式错误
- UBO 关系与 share/votingRights 组合无效
- share 或 votingRights 数值越界

**影响：** 更新被立即拒绝，Partner 收到校验错误

### Path 2: 异步校验失败 — 更新中止

**触发条件：**
- 法人实体状态非 `ACTIVE` 或 `CREATED`
- 更新后的地址不在白名单国家
- FATCA 分类不一致
- UBO 关系与 share/votingRights 不一致
- Person Basic Data 校验失败

**影响：** 更新中止，BO 数据不变，Partner 通过 Webhook 收到错误通知

### Path 3: 更新成功

**触发条件：** 所有校验通过（同步 + 异步），法人实体状态正确，无地域合规问题

**影响：**
- 受益所有人记录更新成功
- 若提供了个人信息字段，通过 global ID 更新 person basic data（**全局生效，影响所有引用该人员的记录**）
- Partner 通过 Webhook 收到成功通知

## 业务规则汇总

### 法人实体要求
- 受益所有人只能在**法人实体**下更新
- 法人实体状态必须为 `ACTIVE` 或 `CREATED`
- Partner 必须有权访问该法人实体

### 受益所有人类型
- 类型始终为 `REAL_UBO_25`，**不可更新**
- `uboRelationship` 可更新，但必须遵循校验规则

### Share 与 Voting Rights

更新时校验会综合考虑**当前值与更新值**：

| UBO 关系 | 规则 |
|----------|------|
| `DIRECTLY_HOLDING_25` / `INDIRECTLY_HOLDING_25` | **当前或更新后的** share 或 votingRights 至少一项 ≥ 25% |
| `DOMINANT_INFLUENCE_OVER_SHARE_CAPITAL` | 无最低值要求 |

- share (若提供): 0% < value ≤ 100%, 精度 0.01
- votingRights (若提供): 0% < value ≤ 100%, 精度 0.01

### 地域合规
- 更新后的地址（当提供 `mainAddress` 时）必须在白名单国家内

### FATCA
- 更新后的 FATCA controlling person 状态必须与法人实体 FATCA 分类一致

### Person Basic Data 更新机制
- **提供了**个人信息字段 → 通过当前 BO 的 global ID 更新全局 person basic data
- **未提供**个人信息字段 → person basic data 保持不变
- person basic data 的更新**影响全局**，所有引用该人员的记录都会受到影响

## 与创建流程的主要差异

| 对比项 | Create | Update |
|--------|--------|--------|
| HTTP 方法 | `POST` | `PATCH` |
| 重复检测 | 全局 duplicate detection | 无 |
| 相似人员匹配 | 有，触发 Admin Review | 无 |
| Restricted Data Check | 有，触发 Admin Review | 无 |
| REVIEW 状态 / 人工审核 | 有 | 无 |
| 业务结果路径 | 4 种 | 3 种 |
| Share/Voting 校验基准 | 仅校验提交值 | 综合当前值与更新值 |
| Person Basic Data | 新建或复用全局记录 | 通过 global ID 更新现有记录 |

## API Reference

- [Beneficial Owners](https://docs.tradevest.ai/api-reference/entities/beneficial-owners) — 受益所有人接口
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
