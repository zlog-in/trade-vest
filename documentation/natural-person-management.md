# Natural Person Management

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/natural_person_management

## 概述

本文档涵盖自然人实体的准备与管理流程，包括成人自然人、未成年人自然人和联名人三种场景，以及身份识别验证和文档签署流程。

## 成人自然人准备流程

```
+----------------------------------------------+
| Step 1: Create Natural Person                |
| POST /entities/natural-persons               |
+----------------------------------------------+
                    |
                    v
       +------------+------------+
       |                         |
   Option A                  Option B
   (Manual Upload)           (External Verifier)
       |                         |
       v                         v
+------------------+  +---------------------------+
| Step 2a: Submit  |  | Step 2b: Create           |
| identification   |  | verification request      |
| POST /entities/  |  | POST /entities/           |
| natural-persons/ |  | identification-           |
| {id}/identifi-   |  | verifications             |
| cations          |  +---------------------------+
+------------------+           |
       |                       v
       v              +---------------------------+
+------------------+  | Retrieve identificationUrl|
| Step 3a: Upload  |  | from response or GET      |
| document         |  | /entities/identification- |
| POST /v2/        |  | verifications/{id}        |
| documents        |  +---------------------------+
| resourceType:    |           |
|  NATURAL_PERSON  |           v
| resourceId:      |  +---------------------------+
|  naturalPersonId |  | On completion, system     |
+------------------+  | auto-creates:             |
       |               | - IDENTIFICATION_         |
       |               |   CERTIFICATE document    |
       |               | - Identity record         |
       |               |   (passport/ID metadata)  |
       |               +---------------------------+
       |                         |
       +------------+------------+
                    |
                    v
+----------------------------------------------+
| Step 4: Sign Documents                       |
| POST /v2/documents/sign                      |
| - partnerDocumentId                          |
| - naturalPersonId                            |
| - customerId (optional, for T&C)             |
+----------------------------------------------+
```

### 身份识别验证（Identification Verification）

每个自然人在入驻过程中都须按照 KYC 要求完成身份识别。

#### Option A: 文档上传方式

1. 通过 `POST /v2/documents` 上传身份证明文件
   - 成人: `IDENTIFICATION_CERTIFICATE`
   - 未成年人: `BIRTH_CERTIFICATE`
2. 通过 `POST /entities/natural-persons/{naturalPersonId}/identifications` 创建身份记录

#### Option B: 外部验证方式

当前支持 **WebID**（识别类型: `VIDEO_IDENT`）。

| 步骤 | 操作 |
|------|------|
| 1. 创建验证请求 | `POST /entities/identification-verifications`（需提供 naturalPersonId、外部验证方类型、识别类型） |
| 2. 获取验证链接 | 从响应的 `identificationUrl` 字段获取，或通过 `GET /entities/identification-verifications/{identificationVerificationId}` 查询 |
| 3. 完成验证 | 用户完成视频认证流程 |
| 4. 系统自动处理 | 自动创建 `IDENTIFICATION_CERTIFICATE` 文档（关联到自然人）+ 身份记录（含证件元数据） |

**状态追踪：**
- 单次查询: `GET /entities/identification-verifications/{identificationVerificationId}`
- 列表查询: `GET /entities/identification-verifications`

## 未成年人自然人准备流程

适用于 18 岁以下的自然人。

```
+----------------------------------------------+
| Step 1: Create Child NP                      |
| POST /entities/natural-persons               |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 2: Submit Identification                |
| POST /entities/natural-persons/              |
|   {naturalPersonId}/identifications          |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 3: Provide Guardian Contact             |
| In child's contact information,              |
| include guardian's contact details           |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 4: Upload Birth Certificate             |
| POST /v2/documents                           |
| documentType: BIRTH_CERTIFICATE              |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 5: Guardian Signs Documents             |
| POST /v2/documents/sign                      |
| - naturalPersonId + partnerDocumentId        |
| - T&C: in customer context (customerId)     |
| - DPP: only if not previously signed        |
+----------------------------------------------+
```

**未成年人签署规则：**
- 监护人（Guardian）代替未成年人执行签署
- **Terms and Conditions (T&C)**: 必须在客户上下文（customer context）中签署，请求体需设置 `customerId`
- **Data Privacy Policy (DPP)**: 仅当该自然人**此前未签署过**（无客户上下文的情况下）时才需签署
- 监护人通过代理关系（Proxy）在角色分配阶段创建并关联

## 文档要求

### 成人自然人

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | **必需** — 所有 18 岁及以上的自然人 |
| `PROOF_OF_RESIDENCE` | 主地址非德国（country ≠ DE）时必需 |
| `TIN_NA_CONFIRMATION` | 税务信息中 `NoTinConfirmation = true` 时必需 |

### 未成年人自然人

| 文档类型 | 条件 |
|----------|------|
| `BIRTH_CERTIFICATE` | **必需** — 所有 18 岁以下的自然人 |
| `PROOF_OF_RESIDENCE` | 主地址非德国（country ≠ DE）时必需 |
| `TIN_NA_CONFIRMATION` | 税务信息中 `NoTinConfirmation = true` 时必需 |

**重要：**
- `IDENTIFICATION_CERTIFICATE` 对未成年人**无效**，仅接受 `BIRTH_CERTIFICATE`
- 系统**自动校验**：未成年人使用出生证明，成人使用身份证明

### 文档签署

| 场景 | 签署方式 |
|------|----------|
| 成人 | `POST /v2/documents/sign`，提供 `partnerDocumentId` + `naturalPersonId`，可选 `customerId`（T&C 场景） |
| 未成年人 | 监护人代签，需 `naturalPersonId` + `partnerDocumentId`，T&C 需在 customer context 下（`customerId` 必填） |

## 联名人准备流程

联名人（Joint Person）由两个自然人组成。

```
+----------------------------------------------+
| Step 1: Create 1st Natural Person            |
| POST /entities/natural-persons               |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 2: Submit 1st Person Identification     |
| POST /entities/natural-persons/              |
|   {naturalPersonId}/identifications          |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 3: Create 2nd Natural Person            |
| POST /entities/natural-persons               |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 4: Submit 2nd Person Identification     |
| POST /entities/natural-persons/              |
|   {naturalPersonId}/identifications          |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 5: Create Joint Person                  |
| POST /entities/joint-persons                 |
| Fields: np1Id, np2Id                         |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 6: Upload Documents for Both NPs        |
| POST /v2/documents                           |
| resourceType: NATURAL_PERSON                 |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
| Step 7: Sign Documents for Both NPs          |
| POST /v2/documents/sign                      |
+----------------------------------------------+
```

### 联名人文档要求

两个自然人**各自**需要以下文档：

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | 成人必需 |
| `BIRTH_CERTIFICATE` | 未成年人必需 |
| `PROOF_OF_RESIDENCE` | 主地址非德国时必需 |
| `TIN_NA_CONFIRMATION` | `NoTinConfirmation = true` 时必需 |

## 涉及的 API 端点汇总

| 端点 | 方法 | 用途 |
|------|------|------|
| `/entities/natural-persons` | `POST` | 创建自然人 |
| `/entities/natural-persons/{id}/identifications` | `POST` | 提交身份识别数据 |
| `/entities/identification-verifications` | `POST` | 创建外部身份验证请求 |
| `/entities/identification-verifications/{id}` | `GET` | 查询验证状态 |
| `/entities/identification-verifications` | `GET` | 列出所有验证记录 |
| `/entities/joint-persons` | `POST` | 创建联名人（np1Id + np2Id） |
| `/v2/documents` | `POST` | 上传文档 |
| `/v2/documents/sign` | `POST` | 签署文档 |

## 相关文档

- [Natural Person Create Process](./natural-person-create-process.md) — 创建流程、校验、重复检测、合规审查
- [Natural Person Update Process](./natural-person-update-process.md) — 更新流程、文档要求、受限字段、Death Day 特殊流程

### API Reference

- [Natural Persons](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/natural-persons) — 自然人 CRUD 接口
- [Identification](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/identification) — 身份识别接口
- [Trading Profile](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/trading-profiles) — 交易档案接口
- [Appropriateness Test](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/appropriateness-test) — 适当性测试接口
- [Joint Persons](https://docs.tradevest.ai/api-reference/entities/joint-persons) — 联名人接口
