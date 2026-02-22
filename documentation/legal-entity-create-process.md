# Legal Entity Create Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/legal_entity_create_process

## 概述

法人实体（Legal Entity）代表公司、基金会、协会等组织。创建流程通过校验确保数据完整性、NACE 行业合规以及防止重复创建。

关键要点：
- 创建流程**不涉及人工审核（REVIEW）环节**
- AML/KYC 合规验证在 Onboarding 阶段单独进行

## 前置条件

- 已收集完整的法人实体信息
- Partner 拥有创建法人实体的授权
- 已准备法人实体的必要身份识别数据

## API 端点

```
POST /entities/legal-entities
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
                  |    Validation              |
                  |  - externalId uniqueness   |
                  |  - Global existence check  |
                  |  - NACE sectors validation |
                  |  - Local existence check   |
                  |  - Schema validation       |
                  +----------------------------+
                       |             |
                     FAIL           PASS
                       |             |
                       v             v
                 +-----------+  +----------------------------+
                 | INVALID   |  | 5. Create / Link Entity    |
                 | Webhook   |  +----------------------------+
                 | error     |       |              |
                 +-----------+       v              v
                              Global record    Global record
                              exists           not exists
                                 |                 |
                                 v                 v
                           Use existing      Create new
                           globalId &        global record
                           update global
                           data
                                 \               /
                                  \             /
                                   v           v
                              +-------------------+
                              | Create local      |
                              | partner-specific  |
                              | record            |
                              | status = CREATED  |
                              +-------------------+
                                       |
                                       v
                              Webhook success
                              notification
```

## 校验体系

### 1. 同步静态校验（Immediate）

请求到达时立即执行，失败直接返回错误。

#### OpenAPI Schema 层校验

| 校验项 | 规则 |
|--------|------|
| 必填字段 | 包括但不限于 `legalName`, `legalForm`, `registerCountry` 等 |
| 字段长度 | 各字段最小/最大长度限制 |
| 格式校验 | 正则模式匹配 |
| 数据类型 | 各字段类型校验 |
| 枚举约束 | 枚举字段取值范围校验 |

#### 业务逻辑校验

| 校验项 | 规则 |
|--------|------|
| FATCA CRS 声明 | 必须提供且正确 |
| `isSanctionedCountries` | 必须为 `false`（表示法人实体未与制裁国家或高风险国家的相关方开展业务） |
| `activeNfeType` 条件必填 | 当 `fatcaClassification` = `ACTIVE_NFE` 时，`activeNfeType` 不可为空 |
| NACE 行业 | 列表不可为空，不可包含重复项 |
| `externalId` 唯一性 | 在 Partner 范围内必须唯一 |

> [!NOTE] 原文 FATCA 规则处有两处笔误：
> - "if **filed** fatcaClassification" 应为 "if **field** fatcaClassification"
> - "activeNfeType **fields is** not empty" 应为 "activeNfeType **field is** not empty"

### 2. 异步业务校验（Asynchronous）

端点返回成功响应后异步执行，结果通过 **Webhook** 通知。

```
         Async Validation Pipeline
                    |
                    v
     +-------------------------------+
     | externalId Uniqueness Check   |
     | (cross-partner)               |
     +-------------------------------+
                    |
                    v
     +-------------------------------+
     | Global Existence Validation   |
     | Same basic data exists        |
     | globally?                     |
     +-------------------------------+
                    |
                    v
     +-------------------------------+
     | NACE Sectors Validation       |
     | - Valid sectors?              |
     | - Banned sectors?             |
     | - Normalize before saving     |
     +-------------------------------+
                    |
                    v
     +-------------------------------+
     | Local Existence Validation    |
     | Already exists for this       |
     | partner?                      |
     +-------------------------------+
                    |
                    v
     +-------------------------------+
     | Schema Validation             |
     +-------------------------------+
                    |
                    v
           Data Creation / Update
```

## 业务结果（3 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 必填字段缺失或格式错误
- `externalId` 在 Partner 范围内已存在
- `isSanctionedCountries` 不为 `false`
- FATCA / NACE 基础校验失败

**影响：** 请求被立即拒绝，Partner 收到校验错误

**后续操作：** 修正数据后重新提交

### Path 2: 异步校验失败 — INVALID

**触发条件：**
- NACE 行业被禁止或无效
- 法人实体在该 Partner 下**本地已存在**
- Schema 校验失败

**影响：** 状态变为 `INVALID`，Partner 通过 Webhook 收到详细错误

**后续操作：** 检查 NACE 行业是否合规、确认法人实体未重复、修正问题后重新提交

### Path 3: 创建成功 — CREATED

**触发条件：** 所有校验通过，NACE 行业有效且未被禁止，本地无重复

**影响：** 法人实体创建成功，状态为 `CREATED`，Partner 通过 Webhook 收到成功通知

## 全局/本地存在性逻辑

```
     Global Existence Check
            |
     +------+------+
     |             |
  Exists       Not Exists
     |             |
     v             v
 Use existing   Create new
 globalId       global record
 Update global
 data
     |             |
     +------+------+
            |
            v
     Local Existence Check
            |
     +------+------+
     |             |
  Exists       Not Exists
     |             |
     v             v
  REJECT     Create local
  (error)    partner-specific
             record
             status = CREATED
```

| 场景 | 全局记录 | 本地记录 | 结果 |
|------|----------|----------|------|
| 全局已存在 + 本地不存在 | 复用 `globalId`，更新全局数据 | 创建 Partner 专属记录 | **CREATED** |
| 全局不存在 + 本地不存在 | 新建全局记录 | 创建 Partner 专属记录 | **CREATED** |
| 本地已存在 | — | — | **REJECT**（返回已存在的法人实体错误） |

## 已知字段列表

| 字段 | 说明 |
|------|------|
| `legalName` | 法人名称 |
| `legalForm` | 法律形式 |
| `registerCountry` | 注册国家 |
| `externalId` | Partner 外部标识符（Partner 范围内唯一） |
| `globalId` | 系统全局标识符 |
| `isSanctionedCountries` | 是否与制裁/高风险国家有业务往来（必须为 `false`） |
| `fatcaClassification` | FATCA 分类（枚举，已知值: `ACTIVE_NFE`） |
| `activeNfeType` | 活跃 NFE 类型（当 `fatcaClassification` = `ACTIVE_NFE` 时必填） |

## 状态值

| 状态 | 含义 |
|------|------|
| `RECEIVED` | 初始状态，数据已存储待异步处理 |
| `CREATED` | 法人实体创建成功 |
| `INVALID` | 异步校验失败 |

> [!NOTE] 原文仅列出以上 3 种状态。相比受益所有人的 9 种状态（含 ACTIVE、INACTIVE、REVIEW、REJECTED 等），此处可能未完整列出法人实体的所有可能状态。

## 业务规则汇总

### 数据要求
- 所有 schema 要求的必填字段必须完整
- NACE 行业必须有效且未被禁止

### NACE 行业规则
- 列表不可为空，不可包含重复项
- 必须属于允许的行业列表
- 被禁止的行业不可使用
- 保存前会进行标准化处理（normalization）

### 全局存在性规则
- 全局已有相同基础数据的法人实体 → 复用 `globalId`，更新全局数据，创建 Partner 专属记录
- 全局不存在 → 新建全局记录 + Partner 专属记录

### 本地存在性规则
- Partner 下已存在该法人实体 → 拒绝创建，返回错误

### 状态规则
- 新法人实体以 `CREATED` 状态创建

## API Reference

- [Legal Entities](https://docs.tradevest.ai/api-reference/entities/legal-entities) — 法人实体接口
- [Search NACE Sectors](https://docs.tradevest.ai/api-reference/entities/search-nace-sectors) — NACE 行业分类查询接口
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
