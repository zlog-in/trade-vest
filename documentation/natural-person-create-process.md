# Natural Person Create Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/natural_person_create_process

## 概述

自然人（Natural Person）代表系统中的个人实体。创建流程通过两阶段校验（同步 + 异步）确保数据完整性、合规性，包括重复检测和人工审核机制。

## 前置条件

- 已收集完整的个人信息

## API 端点

```
POST /entities/natural-persons
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
           | 400/409 |  | 2. Store Data              |
           | (reject)|  |    status = initial        |
           +---------+  +----------------------------+
                              |
                              v
                  +----------------------------+
                  | 3. Async Business          |
                  |    Validation & Review     |
                  |  - NACE sectors            |
                  |  - NP data validation      |
                  |  - Global duplicate detect |
                  |  - Restricted data check   |
                  +----------------------------+
                    /         |          \
                 FAIL    Need Review     PASS
                  /           |            \
                 v            v             v
           +---------+  +----------+  +---------+
           | Rejected|  |  REVIEW  |  | CREATED |
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

> [!NOTE] 原文仅列出 3 种业务结果：立即拒绝、人工审核、自动通过。**未像受益所有人/法定代表人那样明确提及异步校验失败后的 `INVALID` 状态**，但流程中异步校验失败仍然会导致拒绝。

## 校验体系

### 1. 同步静态校验（Immediate）

请求到达时立即执行，失败直接返回错误。

#### OpenAPI Schema 层校验

| 校验项 | 规则 |
|--------|------|
| 必填字段 | 包括但不限于 `firstName`, `lastName` 等（完整列表参见 `reference/entities.yaml`） |
| 字段长度 | `firstName` ≤ 255; `phone` 3-20 字符 |
| 格式校验 | 正则模式（如 email、phone 格式） |
| 数据类型 | 各字段类型校验 |

> [!NOTE] 原文未列出完整必填字段列表和正则模式，指向 `reference/entities.yaml` 获取详情。相比受益所有人/法定代表人文档，此处字段说明明显简略。

#### 业务逻辑校验

| 校验项 | 规则 |
|--------|------|
| US Citizenship | US 国籍声明与国籍列表一致性校验 |
| US Nationality 声明 | 验证 US 国籍标识正确性 |
| Tax Details | 税务居住信息格式校验 |
| 地址 | 主地址（main address）必填，格式校验 |
| NACE 行业 | 行业代码有效性校验 |
| 地域合规 | 主地址国家对比白名单 |
| Partner 内去重 | 同一 Partner 下不可创建重复的自然人，**重复则立即拒绝** |

### 2. 异步业务校验（Asynchronous）

端点返回成功响应后异步执行，结果通过 **Webhook** 通知。

```
              Async Validation Pipeline
                         |
                         v
            +----------------------------+
            | NACE Sectors Validation    |
            | - Check against valid list |
            | - Banned list is empty     |
            |   for natural persons      |
            +----------------------------+
                         |
                         v
            +----------------------------+
            | NP Data Validation         |
            | - Business rules recheck   |
            +----------------------------+
                         |
                         v
            +----------------------------+
            | Global Duplicate Detection |
            | Search by: firstName,      |
            | lastName, birthDate,       |
            | birthPlace, birthCountry,  |
            | taxDetails                 |
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
                             to existing     (NATURAL_
                             record          PERSON_
                                             CREATE)
```

## 业务结果（3 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 个人信息不完整或格式错误
- NACE 行业无效
- 同一 Partner 下已存在该自然人（重复创建）

**影响：** 请求被立即拒绝，Partner 收到校验错误

**后续操作：** 修正数据，确认 NACE 行业有效，重新提交

### Path 2: 需人工审核 — REVIEW

**触发条件：**
- 全局数据库中发现精确匹配或相似的人员记录
- 关键个人信息存在差异（restricted data changes）

**影响：** 状态设为 `REVIEW`，创建流程暂停，通知管理员处理

**Admin Task 类型与决策：**

| Task 类型 | 触发场景 | 可选决策 |
|-----------|----------|----------|
| `MATCHING_SIMILARITIES` | 未找到精确匹配，但发现相似记录 | **MATCH** — 匹配现有全局记录（需提供 `global_id`） |
| | | **NOT_MATCH** — 确认为新人员，创建新记录 |
| `NATURAL_PERSON_CREATE` | 找到精确匹配，且关键字段存在差异 | **APPROVE** — 批准数据变更（更新全局记录，创建成功） |
| | | **REJECT** — 拒绝数据变更（创建被拒绝，全局记录不变） |

### Path 3: 自动通过 — CREATED

**触发条件：** 所有校验通过，无匹配/相似记录，无数据差异

**影响：** 自然人创建成功，状态 `CREATED`，Partner 通过 Webhook 收到成功通知

## 已知字段列表

| 字段 | 说明 |
|------|------|
| `firstName` | 名（≤ 255 字符） |
| `lastName` | 姓（≤ 255 字符） |
| `email` | 邮箱 |
| `phone` | 电话（3-20 字符） |
| `birthDate` / `birthDay` | 出生日期 |
| `birthPlace` | 出生地 |
| `birthCountry` | 出生国家 |
| `nationalities` | 国籍列表 |
| `taxDetails` | 税务信息（含 tax residency、tax identifiers） |
| `mainAddress` | 主地址（必填） |
| `naceSectors` | NACE 行业代码 |
| US nationality flags | US 国籍相关标识 |
| `global_id` | 全局标识符 |

## 业务规则汇总

### Partner 内去重
- 同一 Partner 下不可创建重复的自然人
- 重复创建尝试会被**立即拒绝**

### NACE 行业规则
- 仅接受有效的自然人 NACE 行业代码
- 自然人的禁止行业列表为**空**（即无禁止项）

### 地域合规
- 主地址国家对比白名单国家
- 白名单国家允许直接处理
- 非白名单国家可能触发审核
- 特定国家触发审核

### 人员基础数据
- 全局已存在 → 复用现有记录
- 不存在 → 创建新全局记录
- 精确匹配时 → 关联到已有全局人员

## 与受益所有人/法定代表人创建流程的差异

| 对比项 | Natural Person | Beneficial Owner | Legal Representative |
|--------|---------------|------------------|---------------------|
| 端点 | `POST /entities/natural-persons` | `POST /entities/{leId}/beneficial-owners` | `POST /entities/{leId}/legal-representatives` |
| 关联法人实体 | 不需要 | 必须 | 必须 |
| 法人实体状态校验 | 无 | ACTIVE / CREATED | ACTIVE / CREATED |
| FATCA 校验 | 无 | 有 | 有（条件性） |
| NACE 行业 | 有（禁止列表为空） | 无 | 无 |
| Partner 内去重 | **立即拒绝** | 无此规则 | 无此规则 |
| 特有字段 | `email`, `phone`, `naceSectors` | `uboRelationship`, `share`, `votingRights` | `function`, `soleSignatureAuthorized` |
| Admin Task (精确匹配) | `NATURAL_PERSON_CREATE` | `BENEFICIAL_OWNER_CREATE` | `LEGAL_REPRESENTATIVE_CREATE` |
| 地域合规 | 非白名单可能触发审核 | 非白名单直接拒绝 | 非白名单直接拒绝 |
