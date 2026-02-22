# Legal Representative Create Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/legal_representative_create_process

## 概述

法定代表人（Legal Representative）是被授权代表法人实体处理法律和商业事务的个人。

创建流程通过全面的校验和审核程序确保数据完整性与合规性，包括重复检测、数据质量保障及监管合规。流程结构与受益所有人创建流程高度相似。

## 前置条件

- 已收集完整的法定代表人信息
- 关联的法人实体已存在于系统中

## API 端点

```
POST /entities/{legalEntityId}/legal-representatives
```

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
                  |    Validation & Review     |
                  |  - Legal entity status     |
                  |  - Data validation         |
                  |  - Geographic compliance   |
                  |  - FATCA validation        |
                  |  - Duplicate detection     |
                  |  - Restricted data check   |
                  +----------------------------+
                    /         |          \
                 FAIL    Need Review     PASS
                  /           |            \
                 v            v             v
           +---------+  +----------+  +---------+
           | INVALID |  |  REVIEW  |  | CREATED |
           +---------+  +----------+  +---------+
                              |
                              v
                        Admin Decision
                         /         \
                    APPROVE       REJECT
                      /               \
                     v                 v
               +---------+      +----------+
               | CREATED |      | REJECTED |
               +---------+      +----------+
```

## 校验体系

### 1. 同步静态校验（Immediate）

请求到达时立即执行，失败直接返回错误。

#### OpenAPI Schema 层校验

| 校验项 | 规则 |
|--------|------|
| 必填字段 | `firstName`, `lastName`, `birthDay`, `birthPlace`, `birthCountry`, `taxDetails`, `isUsNationality`, `nationalities`, `function`, `soleSignatureAuthorized`, `fatcaControllingPerson` |
| 字段长度 | `firstName` ≤ 255, `lastName` ≤ 255, `birthPlace` ≤ 255 |
| 格式校验 | `firstName` 正则: `^\S+( \S+)*$` |
| 数据类型 | 各字段类型校验 |
| `function` | 必须为 `LegalRepresentativeFunctionEnum` 的有效值 |

> [!NOTE] 原文多次引用 `LegalRepresentativeFunctionEnum`，但**未列出具体枚举值**，指向 `reference/entities.yaml` 获取详情。

#### 业务逻辑校验

| 校验项 | 规则 |
|--------|------|
| Person Basic Data | 个人信息格式与结构校验 |
| FATCA Controlling Person | 当 `fatcaControllingPerson = TRUE` 时，`taxDetails` 必须提供 |
| `function` | 确认为允许的枚举值 |
| US Citizenship | US 国籍标识与国籍列表一致性校验 |
| Tax Details | 税务居住信息格式校验 |
| 地址 | 主地址（main address）必填 |
| `soleSignatureAuthorized` | 布尔值校验 |

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
            | - Business rules recheck   |
            | - FATCA requirements       |
            | - Function enum check      |
            +----------------------------+
                         |
                         v
            +----------------------------+
            | Geographic Compliance      |
            | addr vs whitelisted        |
            | countries                  |
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
            | birthPlace, birthCountry   |
            | or taxDetails              |
            +----------------------------+
              /          |           \
             /           |            \
            v            v             v
   No match &     Similar records    Exact match
   no similar        found             found
       |                |                |
       v                v                v
  Auto Approve   Admin Review     +------------------+
                 (MATCHING_       | Restricted Data  |
                  SIMILARITIES)   | Changes Check    |
                                  +------------------+
                                     /           \
                                    v             v
                                No diff       Diff found
                                  |               |
                                  v               v
                             Auto link       Admin Review
                             to existing     (LEGAL_REPRE-
                             record          SENTATIVE_
                                             CREATE)
```

## 业务结果（4 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 个人信息/联系信息格式错误或不完整
- `function` 缺失或值无效
- `fatcaControllingPerson = TRUE` 但未提供 `taxDetails`
- `soleSignatureAuthorized` 或 `fatcaControllingPerson` 布尔值无效

**影响：** 请求被立即拒绝，Partner 收到校验错误，需修正后重新提交

### Path 2: 异步校验失败 — INVALID

**触发条件：**
- 法人实体状态非 `ACTIVE` 或 `CREATED`
- 地址不在白名单国家
- FATCA 校验失败
- 其他业务规则违反

**影响：** 创建被拒绝，无法定代表人记录生成，Partner 通过 Webhook 收到详细错误

### Path 3: 需人工审核 — REVIEW

**触发条件：**
- 全局数据库中发现匹配或相似的人员记录
- 关键个人信息存在差异（restricted data changes）

**影响：** 状态设为 `REVIEW`，创建流程暂停，通知管理员处理

**Admin Task 类型与决策：**

| Task 类型 | 触发场景 | 可选决策 |
|-----------|----------|----------|
| `MATCHING_SIMILARITIES` | 未找到精确匹配，但发现相似记录 | **MATCH** — 匹配现有全局记录（需提供 `global_id`） |
| | | **NOT_MATCH** — 确认为新人员，创建新记录 |
| `LEGAL_REPRESENTATIVE_CREATE` | 找到精确匹配，且关键字段存在差异 | **APPROVE** — 批准数据变更（更新全局记录，创建成功） |
| | | **REJECT** — 拒绝数据变更（创建被拒绝，全局记录不变） |

### Path 4: 自动通过 — CREATED

**触发条件：** 所有校验通过，无匹配/相似记录，无数据差异

**影响：** 法定代表人创建成功，状态 `CREATED`，Partner 通过 Webhook 收到成功通知

## 已知状态值

| 状态 | 含义 |
|------|------|
| `RECEIVED` | 初始状态，数据已接收待处理 |
| `REVIEW` | 人工审核中 |
| `CREATED` | 创建成功 |

> [!NOTE] 原文仅提及以上 3 种状态。相比受益所有人的 9 种状态（含 ACTIVE、INACTIVE、REJECTED、PENDING、SUSPENDED_COMPLIANCE 等），此处可能未完整列出。此外原文中 "Under Review" 与 "REVIEW" 混用描述同一状态。

## 特有字段（与受益所有人对比）

法定代表人相比受益所有人，有以下**特有字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `function` | Enum (`LegalRepresentativeFunctionEnum`) | 法定代表人职能，决定法律权限与职责 |
| `soleSignatureAuthorized` | Boolean | 是否被授权独立代表公司（独立签署权） |
| `fatcaControllingPerson` | Boolean | 是否为 FATCA 控制人 |

**无以下受益所有人字段：** `uboRelationship`、`share`、`votingRights`

## 业务规则汇总

### 法人实体要求
- 法定代表人只能为**法人实体**创建
- 法人实体状态必须为 `ACTIVE` 或 `CREATED`
- Partner 必须有权访问该法人实体
- 每个法人实体可拥有多个不同 `function` 的法定代表人

### FATCA 规则
- `fatcaControllingPerson = TRUE` 时，必须提供 `taxDetails`
- FATCA controlling person 状态必须与法人实体 FATCA 分类一致

### 签署授权
- `soleSignatureAuthorized` 决定法定代表人是否有权独立代表法人实体行事

### 地域合规
- 地址必须在白名单国家内

### 人员基础数据管理
- 全局已存在的 person basic data 会被复用
- 不存在时创建新记录
- 关联到已有全局人员时执行数据更新

## 与 LEC 入驻的联动：FICTIVE_UBO 自动创建

当法人实体客户入驻时，若不存在状态为 `CREATED` 的 `REAL_UBO_25` 受益所有人，系统自动将 ACTIVE 法定代表人转换为 `FICTIVE_UBO`：

- type: `FICTIVE_UBO`
- share: 0%
- status: `PENDING`
- 所有个人数据从法定代表人记录复制

> [!NOTE] 原文此处 FICTIVE_UBO 字段仅列出 `share: 0%`，**未提及 `votingRights`**。而在受益所有人创建流程文档中，同一机制明确列出了 `votingRights: 0%`。两处描述不一致。

此自动流程**仅在入驻流程中触发**，不影响手动创建。

## 与受益所有人创建流程的差异

| 对比项 | Beneficial Owner Create | Legal Representative Create |
|--------|------------------------|----------------------------|
| 端点 | `POST /entities/{leId}/beneficial-owners` | `POST /entities/{leId}/legal-representatives` |
| 特有字段 | `uboRelationship`, `share`, `votingRights` | `function`, `soleSignatureAuthorized`, `fatcaControllingPerson` |
| Admin Task (精确匹配) | `BENEFICIAL_OWNER_CREATE` | `LEGAL_REPRESENTATIVE_CREATE` |
| FATCA 条件校验 | 始终校验 | 仅当 `fatcaControllingPerson = TRUE` 时需 `taxDetails` |
| 类型 | `REAL_UBO_25` (固定) | 由 `function` 枚举决定 |

## API Reference

- [Legal Representatives](https://docs.tradevest.ai/api-reference/entities/legal-representatives) — 法定代表人接口
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
