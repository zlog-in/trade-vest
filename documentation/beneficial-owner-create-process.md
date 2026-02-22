# Beneficial Owner Create Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/beneficial_owner_create_process

## 概述

受益所有人（Beneficial Owner）是指持有法人实体股份或控制权的个人。

创建流程通过全面的校验和审核程序来确保数据完整性与合规性，包括防重复创建、数据质量保障及监管合规。

关键要点：
- 通过此流程创建的受益所有人类型均为 **`REAL_UBO_25`**
- AML/KYC 合规验证在 **Onboarding 阶段**单独进行，不在创建流程中

## 前置条件

- 已收集完整的受益所有人信息
- 关联的法人实体已存在于系统中

## API 端点

```
POST /entities/{legalEntityId}/beneficial-owners
```

## 整体流程

```
                     Partner Request
                           |
                           v
               +---------------------------+
               | 1. Synchronous Static     |
               |    Validation             |
               |    - OpenAPI schema check |
               |    - Business logic rules |
               +---------------------------+
                     |             |
                   FAIL           PASS
                     |             |
                     v             v
                +---------+  +---------------------------+
                | 400/409 |  | 2. Publish Command Event  |
                | (reject)|  +---------------------------+
                +---------+        |
                                   v
                           +---------------------------+
                           | 3. Store Data             |
                           |    status = RECEIVED      |
                           +---------------------------+
                                   |
                                   v
                           +---------------------------+
                           | 4. Async Business         |
                           |    Validation             |
                           |  - Legal entity status    |
                           |  - Data validation        |
                           |  - Geographic compliance  |
                           |  - FATCA validation       |
                           |  - Duplicate detection    |
                           |  - Restricted data check  |
                           +---------------------------+
                            /          |          \
                         FAIL     Need Review     PASS
                          /            |            \
                         v             v             v
                   +---------+   +----------+   +---------+
                   | INVALID |   |  REVIEW  |   | CREATED |
                   +---------+   +----------+   +---------+
                                       |
                                       v
                                 Admin Decision
                                  /         \
                             APPROVE       REJECT
                               /               \
                              v                 v
                        +---------+       +----------+
                        | CREATED |       | REJECTED |
                        +---------+       +----------+
```

## 校验体系

### 1. 同步静态校验（Immediate）

请求到达时立即执行，失败直接返回错误。

#### OpenAPI Schema 层校验

| 校验项 | 规则 |
|--------|------|
| 必填字段 | `firstName`, `lastName`, `birthDay`, `birthPlace`, `birthCountry`, `taxDetails`, `isUsNationality`, `nationalities`, `uboRelationship`, `share`, `votingRights` |
| 字段长度 | `firstName` ≤ 255, `lastName` ≤ 255, `birthPlace` ≤ 255 |
| 格式校验 | `firstName` 正则: `^\S+( \S+)*$` |
| `share` | 0-100%, 精度 0.01, 必须 > 0 |
| `votingRights` | 0-100%, 精度 0.01, 必须 > 0 |

#### 业务逻辑校验

| 校验项 | 规则 |
|--------|------|
| `uboRelationship` | 必填，枚举值见下方 |
| Share/Voting vs UBO 关系一致性 | `DIRECTLY_HOLDING_25` 或 `INDIRECTLY_HOLDING_25`: share 或 votingRights 至少一项 ≥ 25% |
| | `DOMINANT_INFLUENCE_OVER_SHARE_CAPITAL`: 无最低要求 |
| 地址 | 主地址（main address）必填 |
| US 公民关系 | 校验 US Citizenship 相关信息 |
| 税务信息 | 校验 Tax Details 完整性 |

#### `uboRelationship` 枚举值

| 值 | 含义 |
|----|------|
| `DIRECTLY_HOLDING_25` | 直接持有公司 ≥25% 股份或投票权 |
| `INDIRECTLY_HOLDING_25` | 通过中间实体或信托安排间接持有 ≥25% |
| `DOMINANT_INFLUENCE_OVER_SHARE_CAPITAL` | 通过投票协议、特殊权利或信托关系以其他方式控制公司 |

### 2. 异步业务校验（Asynchronous）

端点返回成功响应后异步执行，结果通过 **Webhook** 通知。

```
                Async Validation Pipeline
                           |
                           v
              +----------------------------+
              | Legal Entity Status Check  |
              | must be ACTIVE or CREATED  |
              +----------------------------+
                           |
                           v
              +----------------------------+
              | Data Validation            |
              | business rules recheck     |
              +----------------------------+
                           |
                           v
              +----------------------------+
              | Geographic Compliance      |
              | addr vs whitelisted countries
              +----------------------------+
                           |
                           v
              +----------------------------+
              | FATCA Validation           |
              | controlling person vs      |
              | LE's FATCA classification  |
              +----------------------------+
                           |
                           v
              +----------------------------+
              | Global Duplicate Detection |
              | Search by: firstName,      |
              | lastName, birthDay,        |
              | birthPlace, birthCountry,  |
              | taxDetails                 |
              +----------------------------+
                /          |           \
               /           |            \
              v            v             v
     No match &      Similar records   Exact match
     no similar         found            found
         |                |                |
         v                v                v
   Auto Approve    Admin Review     +------------------+
                   (MATCHING_       | Restricted Data  |
                    SIMILARITIES)   | Changes Check    |
                                    +------------------+
                                       /           \
                                      v             v
                                  No diff       Diff found
                                    |               |
                                    v               v
                               Auto link       Admin Review
                               to existing     (BENEFICIAL_
                               record          OWNER_CREATE)
```

## 业务结果（4 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：** 数据格式错误、必填字段缺失、UBO 关系与 share/votingRights 不一致等

**影响：** 请求被立即拒绝，Partner 收到校验错误，需修正后重新提交

### Path 2: 异步校验失败 — INVALID

**触发条件：**
- 法人实体状态非 `ACTIVE` 或 `CREATED`
- 地址不在白名单国家
- FATCA 校验失败
- 其他业务规则违反

**影响：** 状态变为 `INVALID`，不创建受益所有人记录，Partner 通过 Webhook 收到详细错误

### Path 3: 需人工审核 — REVIEW

**触发条件：**
- 全局数据库中发现匹配或相似的人员记录
- 关键个人信息存在差异（restricted data changes）

**影响：** 状态设为 `REVIEW`，创建流程暂停，通知管理员处理

**Admin Task 类型与决策：**

| Task 类型 | 触发场景 | 可选决策 |
|-----------|----------|----------|
| `MATCHING_SIMILARITIES` | 未找到精确匹配，但发现相似记录 | **MATCH** — 确认匹配现有全局记录（需提供 `global_id`）|
| | | **NOT_MATCH** — 确认为新人员，创建新记录 |
| `BENEFICIAL_OWNER_CREATE` | 找到精确匹配，且关键字段存在差异 | **APPROVE** — 批准数据变更（更新全局记录，创建成功）|
| | | **REJECT** — 拒绝数据变更（创建被拒绝，全局记录不变）|

### Path 4: 自动通过 — CREATED

**触发条件：** 所有校验通过，无匹配/相似记录，无数据差异

**影响：** 受益所有人成功创建，类型 `REAL_UBO_25`，状态 `CREATED`，Partner 通过 Webhook 收到成功通知

## 受益所有人状态值

| 状态 | 含义 |
|------|------|
| `RECEIVED` | 初始状态，数据已接收待处理 |
| `INVALID` | 异步校验失败 |
| `REVIEW` | 人工审核中 |
| `REJECTED` | 创建被拒绝 |
| `PENDING` | 等待进一步处理 |
| `CREATED` | 创建成功，已通过校验 |
| `ACTIVE` | 在系统中激活 |
| `INACTIVE` | 已停用 |
| `SUSPENDED_COMPLIANCE` | 因合规问题被暂停 |

## 业务规则汇总

### 法人实体要求
- 受益所有人只能为**法人实体**创建
- 法人实体状态必须为 `ACTIVE` 或 `CREATED`
- Partner 必须有权访问该法人实体

### 受益所有人类型
- 手动创建的受益所有人类型始终为 `REAL_UBO_25`（`boType` 参数已弃用）
- `uboRelationship` 为必填字段

### Share 与 Voting Rights
- 范围: 0% < value ≤ 100%，精度 0.01
- `votingRights` 对所有受益所有人必填
- `DIRECTLY_HOLDING_25` / `INDIRECTLY_HOLDING_25`: share 或 votingRights 至少一项 ≥ 25%
- `DOMINANT_INFLUENCE_OVER_SHARE_CAPITAL`: 无最低值要求

### 地域合规
- 地址必须在白名单国家内

### FATCA
- FATCA controlling person 状态必须与法人实体的 FATCA 分类一致

### 人员基础数据管理
- 全局已存在的 person basic data 会被复用
- 不存在时创建新记录
- 关联到已有全局人员时执行数据更新

## 与法人实体客户入驻的联动：自动创建 FICTIVE_UBO

当法人实体客户（LEC）进行入驻流程时，若系统检测到该法人实体下**不存在任何状态为 `CREATED` 的 `REAL_UBO_25` 受益所有人**，将自动创建 `FICTIVE_UBO`：

```
              LEC Onboarding
                    |
                    v
    +-------------------------------+
    | Any REAL_UBO_25 with          |
    | status CREATED?               |
    +-------------------------------+
             |              |
            YES             NO
             |              |
             v              v
        Continue   +-------------------------------+
        Onboarding | 1. Remove all existing BOs    |
                   | 2. Retrieve ACTIVE legal reps |
                   | 3. For each legal rep, create:|
                   |    - type: FICTIVE_UBO        |
                   |    - share: 0%                |
                   |    - votingRights: 0%         |
                   |    - status: PENDING          |
                   |    - Copy all personal data   |
                   +-------------------------------+
                               |
                               v
                      Continue Onboarding
```

**要点：**
- 此自动流程**仅在入驻流程中触发**，不影响手动创建
- 确保即使无人持有 ≥25% 股份，也能满足监管对受益所有权记录的要求
- FICTIVE_UBO 基于法定代表人自动生成，无需人工干预

## API Reference

- [Beneficial Owners](https://docs.tradevest.ai/api-reference/entities/beneficial-owners) — 受益所有人接口
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
