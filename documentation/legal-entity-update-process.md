# Legal Entity Update Process

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/legal_entity_update_process

## 概述

法人实体更新流程，根据实体当前状态（`CREATED` / `ACTIVE`）采用不同的处理路径，`ACTIVE` 状态下涉及文档要求、KYC 审查和人工审核机制。

## 前置条件

- 法人实体状态为 `CREATED` 或 `ACTIVE`
- `ACTIVE` 状态下，部分字段修改需提供支撑文档（`documentIds`）
- 若法人实体通过 Vendor 创建，部分字段**不可更新**

## API 端点

```
PUT /entities/legal-entities/{legalEntityId}
```

**请求体结构：**

```json
{
  "legalEntityUpdate": { /* 需要更新的字段 */ },
  "documentIds": ["uuid-of-document-1", "uuid-of-document-2"]
}
```

## CREATED vs ACTIVE 状态差异总览

| 对比项 | CREATED | ACTIVE |
|--------|---------|--------|
| 文档要求 | 无 | 部分字段需提供支撑文档 |
| KYC 审查 | 无 | 更新前触发 KYC screening |
| 人工审核 | 无（直接更新） | 敏感字段变更可能触发 Review |
| US 地址 | **直接拒绝** | 允许，但触发合规审核 |
| `isSanctionedCountries=true` | **直接拒绝** | 允许，但触发合规审核 |
| `activeNfeType=NON_PROFIT_ORGANISATION` | **直接拒绝** | 允许，但触发合规审核 |
| 被禁止的 NACE 行业 | **直接拒绝** | 允许，但触发合规审核 |

## 整体流程

### CREATED 状态 — 简化流程

```
         Partner PUT Request
                |
                v
     +---------------------+
     | Synchronous Static  |
     |   Validation        |
     +---------------------+
           |          |
         FAIL        PASS
           |          |
           v          v
       400/409  +---------------------+
       (reject) | Async Business      |
                |   Validation        |
                +---------------------+
                      |          |
                    FAIL        PASS
                      |          |
                      v          v
                  INVALID   +---------------------+
                            | Update Legal Entity |
                            +---------------------+
                                     |
                                     v
                            Webhook success
```

### ACTIVE 状态 — 完整流程

```
              Partner PUT Request
                       |
                       v
          +---------------------------+
          | Synchronous Static        |
          |   Validation              |
          | + Document ID validation  |
          +---------------------------+
                 |            |
               FAIL          PASS
                 |            |
                 v            v
            400/409   +---------------------------+
            (reject)  | Async Business Validation |
                      | + Document type check     |
                      | + LEI vendor validation   |
                      +---------------------------+
                            |            |
                          FAIL          PASS
                            |            |
                            v            v
                        INVALID   +---------------------------+
                                  | Review Detection          |
                                  | Triggers review?          |
                                  +---------------------------+
                                     |               |
                                    YES              NO
                                     |               |
                                     v               v
                              +-----------+   +-----------+
                              |  REVIEW   |   | KYC Check |
                              +-----------+   +-----------+
                                     |               |
                                     v               |
                              Admin Decision         |
                             /     |       \         |
                            /      |        \        |
                           v       v         v       |
                      APPROVED  REJECTED  APPROVE &  |
                         |      _ENTITY   SUSPEND    |
                         |        |       GLOBALLY   |
                         |        v         |        |
                         |      STOP    Suspend &    |
                         |              continue     |
                         |                 |         |
                         +--------+--------+         |
                                  |                  |
                                  v                  |
                           +-----------+             |
                           | KYC Check |<------------+
                           +-----------+
                                  |
                     +------+-----+------+-------+
                     |      |            |       |
                     v      v            v       v
                  Valid  ManualReview  Repeat  Rejected
                    |    (KYC_        KYC       |
                    |     SUSPICIONS)  |        v
                    |       |          |      STOP
                    v       v          v
               +--------------------------+
               | Update Legal Entity      |
               +--------------------------+
                          |
                          v
                  Webhook success
```

## 可更新字段

### ACTIVE 状态可更新字段

| 类别 | 字段 |
|------|------|
| 基本信息 | `legalName`, `legalForm`, `purpose` |
| 行业 | `naceSectors` |
| 税务 | `taxNumber`, `vatId` |
| 上市信息 | `isListed`, `listedAt` |
| LEI | `legalEntityIdentifier` |
| FATCA/CRS | `fatcaCrsDeclaration` |
| 地址 | `mainAddress`, `correspondenceAddress`, `invoiceAddress` |
| 财务 | `averageProfit`, `averageRevenue`, `liquidAssets`, `lastAnnualFinancialStatement` |
| 外部 ID | `externalId` |

### Vendor 字段限制

若法人实体通过 Vendor 创建：

| 不可更新（Vendor 提供） | 仍可更新 |
|--------------------------|----------|
| `legalName`, `legalForm`, `naceSectors` | `mainAddress`, `correspondenceAddress`, `invoiceAddress` |
| `taxNumber`, `vatId`, `isListed`, `listedAt` | `averageProfit`, `averageRevenue`, `liquidAssets` |
| `legalEntityIdentifier`, `purpose`, `fatcaCrsDeclaration` | `lastAnnualFinancialStatement`, `externalId` |

## 文档要求（仅 ACTIVE 状态）

更新 `ACTIVE` 状态法人实体的部分字段时，必须在请求中通过 `documentIds` 提供支撑文档。

### legalName 更新所需文档

`CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST`, `TRANSPARENCY_REGISTER_EXTRACT`

### 地址字段更新所需文档

`mainAddress` / `correspondenceAddress` / `invoiceAddress` → `CURRENT_REGISTRY_EXTRACT`

### legalForm 更新所需文档（按法律形式）

| 法律形式 | 所需文档 |
|----------|----------|
| `LIMITED_LIABILITY_COMPANY` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST` |
| `PUBLIC_LIMITED_COMPANY` | `CURRENT_REGISTRY_EXTRACT`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `FOUNDATION` | `CURRENT_REGISTRY_EXTRACT`, `STATUTE` |
| `ASSOCIATION` | `STATUTE` |
| `REGISTERED_BUSINESSMAN` | `CURRENT_REGISTRY_EXTRACT` |
| `LIMITED_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `GENERAL_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `LIMITED_LIABILITY_COMPANY_AND_LIMITED_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `PARTNERSHIP` | `PARTNERSHIP_AGREEMENT` |

## 校验体系

### 1. 同步静态校验（Immediate）

| 校验项 | 规则 |
|--------|------|
| 字段长度/格式/类型 | OpenAPI schema 校验 |
| 文档 ID | `ACTIVE` 状态下校验 `documentIds` 是否满足字段要求 |
| 地址 | zip code 3-10 字符；国家限制 |
| FATCA/CRS 结构 | `ACTIVE_NFE` 时 `activeNfeType` 不可为空 |
| NACE 行业结构 | code 非空、section 格式合法、无重复项 |

### 2. 异步业务校验（Asynchronous）

| 校验项 | 说明 |
|--------|------|
| 法人实体状态 | 必须为 `CREATED` 或 `ACTIVE` |
| 文档校验 | `documentIds` 引用的文档存在且类型匹配（仅 ACTIVE） |
| LEI 校验 | 通过 Vendor API 验证 LEI 格式、状态、有效期 |
| NACE 行业 | 标准化处理、校验有效性、检查是否被禁止 |
| 地域合规 | 地址国家合规检查 |
| FATCA/CRS | 合规校验 |
| `externalId` | Partner 范围内唯一性校验 |

## 审核检测机制（仅 ACTIVE 状态）

CREATED 状态不触发任何审核，直接更新。

### Admin Task 类型

| Task 类型 | 触发条件 | 说明 |
|-----------|----------|------|
| `LEGAL_ENTITY_UPDATE` | 更新 `legalName` 或 `legalForm` | 常规审核 |
| `COMPLIANCE_REVIEW` | 地址变更至 US 或受限国家 | 合规审核 |
| | NACE 行业包含被禁止项 | |
| | `sanctionedCountries` 变为 `TRUE` | |
| | `activeNfeType` 变为 `NON_PROFIT_ORGANISATION` | |

### Admin 决策选项

| 决策 | 后续动作 |
|------|----------|
| **APPROVED** | 状态恢复，继续进入 KYC Check |
| **REJECTED_ENTITY** | 更新流程**终止** |
| **APPROVE_AND_SUSPEND_GLOBALLY** | 状态恢复，实体**全局挂起**，继续进入 KYC Check |

## KYC 验证（仅 ACTIVE 状态）

在更新应用前触发，由外部供应商评估数据真实性、监管数据库冲突和监控名单匹配。

| KYC 结果 | 后续动作 |
|----------|----------|
| **Valid** | 继续执行更新 |
| **Manual Review Required** | 创建 `KYC_SUSPICIONS` Admin Task |
| **Repeat KYC** | 重新执行 KYC 验证 |
| **Rejected** | 更新流程**终止** |

## 地址校验规则

| 规则 | CREATED | ACTIVE |
|------|---------|--------|
| US 地址 | 直接拒绝（validation error） | 允许，触发 `COMPLIANCE_REVIEW` |
| Zip code | 3-10 字符 | 3-10 字符 |
| 非 DE 主地址 | — | 要求 `fatcaCrsDeclaration.isForeignTaxResidency = TRUE` |

## FATCA/CRS 校验规则

| 规则 | CREATED | ACTIVE |
|------|---------|--------|
| `ACTIVE_NFE` 分类 | 必须提供 `activeNfeType` | 必须提供 `activeNfeType` |
| `PASSIVE_NFE` 分类 | 所有代表人和受益所有人需完整税务信息 | 同左 |
| `isSanctionedCountries = true` | **直接拒绝** | 允许，触发 `COMPLIANCE_REVIEW` |
| `activeNfeType = NON_PROFIT_ORGANISATION` | **直接拒绝** | 允许，触发 `COMPLIANCE_REVIEW` |
| 非 DE 主地址 | — | 要求 `isForeignTaxResidency = TRUE` |

## 业务结果（4 种路径）

### Path 1: 校验失败 — 立即拒绝

**触发条件：** 格式错误、文档缺失、CREATED 状态下的合规硬拒绝

### Path 2: 异步校验失败 — INVALID

**触发条件：** LEI 无效、NACE 非法、文档类型不匹配、地域合规失败等

### Path 3: 需人工审核 — REVIEW

**触发条件：** ACTIVE 状态下 legalName/legalForm 变更或合规敏感项变更

**影响：** 状态变为 `REVIEW`，暂停更新，创建 Admin Task，等待决策

### Path 4: 自动通过 — 更新成功

**触发条件：** 所有校验通过，无审核触发条件（CREATED 状态下始终走此路径）

**影响：** 法人实体数据更新成功，Partner 通过 Webhook 收到成功通知
