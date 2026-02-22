# Natural Person Update Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/natural_person_update_process

## 概述

自然人更新流程分为两种类型：**标准数据更新**和**死亡日期更新**（特殊流程）。根据实体状态（`CREATED` / `ACTIVE`）采用不同的处理路径，`ACTIVE` 状态下涉及文档要求、KYC 审查和受限字段审核。

## 前置条件

- 自然人状态为 `CREATED` 或 `ACTIVE`
- `ACTIVE` 状态下，部分字段更新需提前上传支撑文档并提供 `documentId`

## API 端点

```
PATCH /entities/natural-persons/{naturalPersonId}
```

**请求体结构：**

```json
{
  "naturalPersonUpdateData": {
    /* 需要更新的字段 */
  },
  "documentId": "uuid-of-supporting-document"
}
```

## CREATED vs ACTIVE 状态差异总览

| 对比项 | CREATED | ACTIVE |
|--------|---------|--------|
| 文档要求 | 无 | 部分字段需提供支撑文档 |
| KYC 审查 | 无 | 更新前触发 KYC screening |
| 审核触发 | 全局匹配冲突时触发 | 受限字段变更 / 地域合规 / 税务变更触发 |
| Death Day 更新 | 不允许 | 允许（特殊流程） |

## 整体流程

### 标准数据更新流程

```
              Partner PATCH Request
                       |
                       v
          +----------------------------+
          | 1. Synchronous Static      |
          |    Validation              |
          |  - OpenAPI schema check    |
          |  - Business logic rules    |
          |  - Document ID check       |
          |    (ACTIVE only)           |
          +----------------------------+
                 |             |
               FAIL           PASS
                 |             |
                 v             v
           +---------+  +----------------------------+
           | 400/409 |  | 2. Retrieve Natural Person |
           | (reject)|  +----------------------------+
           +---------+        |
                              v
                  +----------------------------+
                  | 3. Async Business          |
                  |    Validation & Review     |
                  |  - NP status validation    |
                  |  - Document validation     |
                  |  - NACE sectors            |
                  |  - Geographic compliance   |
                  |  - Restricted data detect  |
                  +----------------------------+
                    /         |          \
                 FAIL    Need Review     PASS
                  /           |            \
                 v            v             v
           +---------+  +----------+       |
           | Rejected|  |  REVIEW  |       |
           +---------+  +----------+       |
                              |            |
                         Admin Decision    |
                          /        \       |
                     APPROVE    REJECT     |
                        |         |        |
                        v         v        |
                    Continue    STOP       |
                        |                  |
                        +--------+---------+
                                 |
                    +------------+------------+
                    |    ACTIVE status only   |
                    v                         v
            +---------------+          +-----------+
            | 4. KYC Check  |          | 5. Update |
            +---------------+          |    (for   |
              |    |    |    |         |  CREATED) |
              v    v    v    v         +-----------+
           Valid Review Repeat Reject
             |    |     KYC    |
             |    v      |     v
             | KYC_      |   STOP
             | SUSPICIONS|
             |    |      |
             +----+------+
                  |
                  v
          +---------------+
          | 5. Update     |
          |    Natural    |
          |    Person     |
          +---------------+
                  |
                  v
          Webhook success
```

### 死亡日期更新流程（仅 ACTIVE 状态）

```
      Partner PATCH Request
      (only deathDay field)
               |
               v
    +-------------------------+
    | Validate:               |
    | - deathDay > birthDay   |
    | - No other fields       |
    |   updated simultaneously|
    | - DEATH_CERTIFICATE     |
    |   document provided     |
    +-------------------------+
           |          |
         FAIL        PASS
           |          |
           v          v
       400/409  +-------------------------+
       (reject) | Check: NP is only       |
                | active proxy for any    |
                | entity?                 |
                +-------------------------+
                     |            |
                    YES          NO
                     |            |
                     v            v
              +------------+  +------------------+
              | Create     |  | Update deathDay  |
              | admin task |  | Update related   |
              | for proxy  |  | statuses         |
              | reassign   |  +------------------+
              +------------+          |
                     |                v
                     v          Webhook success
              Update deathDay
              Update related
              statuses
                     |
                     v
              Webhook success
```

## 文档要求（仅 ACTIVE 状态）

| 更新字段 | 所需文档类型 |
|----------|-------------|
| `firstName` | `KYC` |
| `lastName` | `KYC` |
| `gender` | `KYC` |
| `title` | `KYC` |
| `mainAddress`（非德国地址） | `PROOF_OF_RESIDENCE` |
| `deathDay` | `DEATH_CERTIFICATE` |

## 校验体系

### 1. 同步静态校验（Immediate）

| 校验项 | 规则 |
|--------|------|
| 字段长度/格式/类型 | OpenAPI schema 校验 |
| 文档 ID | `ACTIVE` 状态下校验 `documentId` 是否满足字段要求 |
| Death Day 排他性 | `deathDay` 更新时不可同时更新其他字段；仅 `ACTIVE` 状态允许 |
| 联系信息 | 格式校验 |
| NACE 行业 | 行业代码格式校验 |
| 状态确认 | 自然人状态必须为 `CREATED` 或 `ACTIVE` |

### 2. 异步业务校验（Asynchronous）

| 步骤 | 说明 |
|------|------|
| NP 状态校验 | 确认 `CREATED` 或 `ACTIVE` |
| 文档校验 | `documentId` 引用的文档存在且类型正确（仅 ACTIVE） |
| NACE 行业 | 标准化处理、校验有效性/禁止列表 |
| 地域合规 | 地址变更对比白名单国家，主地址国家变更可能触发审核 |
| 受限数据变更检测 | 检测关键字段变更（详见下方） |
| KYC Screening | 外部供应商验证（仅 ACTIVE） |
| Death Day 专项校验 | Death Day 特殊流程校验 |

## 审核触发条件

### CREATED 状态

| 触发条件 | 说明 |
|----------|------|
| 全局匹配冲突 | 更新的关键个人数据在系统中已被其他实体使用，需人工核实 |

### ACTIVE 状态

| 触发条件 | 说明 |
|----------|------|
| 核心身份字段变更 | `firstName`, `lastName`, `gender`, `title` |
| 税务信息变更 | 德国 Tax ID 变更（新旧 ID 不同）；添加 US 税务居住地 |
| 地域合规问题 | 地址变更至受限国家或非白名单国家 |

## 业务结果（5 种路径）

### Path 1: 同步校验失败 — 立即拒绝

**触发条件：**
- 格式/结构错误
- `ACTIVE` 状态下缺少 `documentId`
- `deathDay` 与其他字段同时更新
- NACE 行业代码无效
- 文档类型不匹配

**影响：** 更新被立即拒绝，Partner 收到校验错误

### Path 2: 需人工审核 — REVIEW

**触发条件：**
- CREATED 状态：全局匹配冲突
- ACTIVE 状态：受限字段变更、地域合规问题、税务变更

**影响：** 状态设为 REVIEW，暂停更新，创建 Admin Task，Partner 通过 Webhook 收到通知，管理员可批准或拒绝

### Path 3: 自动通过 — 更新成功

**触发条件：** 所有校验通过，CREATED 状态无全局冲突，ACTIVE 状态无受限字段变更

**影响：** 自然人数据更新成功，Partner 通过 Webhook 收到成功通知

### Path 4: Death Day 特殊流程

**触发条件：** 仅更新 `deathDay` 字段，提供有效的 `DEATH_CERTIFICATE`

**处理：**
1. 校验 `deathDay` 晚于 `birthDay`
2. 检查该自然人是否为任何实体的**唯一活跃代理人（only active proxy）**
3. 若是 → 为受影响的实体创建 Admin Task（代理人重新分配）
4. 更新自然人的 `deathDay`
5. 更新关联的代理角色、客户角色和实体状态

### Path 5: KYC 审查失败（仅 ACTIVE）

**触发条件：** KYC screening 发现合规问题或数据与监管数据库冲突

| KYC 结果 | 后续动作 |
|----------|----------|
| **Valid** | 继续执行更新 |
| **Manual Review Required** | 创建 `KYC_SUSPICIONS` Admin Task |
| **Repeat KYC** | 重新执行 KYC 验证 |
| **Rejected** | 更新流程**终止**，可能全局标记或拒绝 |

## 业务规则汇总

### 状态要求
- 仅 `CREATED` 或 `ACTIVE` 状态的自然人可被更新

### Death Day 规则
- `deathDay` 必须晚于 `birthDay`
- 更新 `deathDay` 时**不可同时更新其他字段**
- 仅 `ACTIVE` 状态允许更新 `deathDay`
- 必须提供 `DEATH_CERTIFICATE` 文档

### NACE 行业规则
- 行业代码格式校验
- 异步校验有效性和禁止列表，并标准化处理

### 地域合规
- 地址变更对比白名单国家
- 主地址国家变更可能触发审核

### 税务变更限制
- 德国 Tax ID 变更（新旧不同）→ 触发审核
- 添加 US 税务居住地 → 触发审核

> [!NOTE] 原文未提及与受益所有人更新流程类似的 "person basic data 全局更新" 机制。在 BO 更新中，个人信息变更会通过 global ID 影响所有引用该人员的记录，但此文档未说明自然人更新是否有相同的全局传播效果。

> [!NOTE] 原文未明确列出 Admin Task 的具体类型名称（如是否有 `NATURAL_PERSON_UPDATE` 类型）和完整的决策选项，仅描述了管理员可以批准或拒绝。

## API Reference

- [Natural Persons](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/natural-persons) — 自然人 CRUD 接口
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
