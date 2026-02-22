# Onboarding

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/onboarding

## 概述

入驻（Onboarding）是实体准备和角色分配完成后的最终激活流程。所有入驻过程均为**异步**执行，涵盖实体/角色校验、文档校验、参考账户 IBAN 校验（客户适用）及 KYC 合规验证。

### 三阶段架构

```
+------------------------------------------------+
| Stage 1: Entity Preparation                    |
| - Create entities (NP / LE / JP / BO / LR)    |
| - Prepare data and documents                   |
| - Complete identification verification         |
+------------------------------------------------+
                       |
                       v
+------------------------------------------------+
| Stage 2: Role Assignment                       |
| - POST /roles/customers  (assign Customer)     |
| - POST /roles/proxies    (assign Proxy)        |
| - Upload role-specific documents               |
+------------------------------------------------+
                       |
                       v
+------------------------------------------------+
| Stage 3: Onboarding Initiation                |
| - POST /roles/onboardings  (Customer / Proxy) |
| - POST /entities/onboardings  (BO)            |
| - Async validation, KYC, activation           |
+------------------------------------------------+
```

## 三种入驻类型

| 类型 | 端点 | 说明 |
|------|------|------|
| **Customer Onboarding** | `POST /roles/onboardings` | 综合性入驻：处理客户角色及所有关联实体（代理人、受益所有人）|
| **Proxy Onboarding** | `POST /roles/onboardings` | 专项入驻：为已激活客户添加新代理人 |
| **Beneficial Owner Onboarding** | `POST /entities/onboardings` | 专项入驻：为已激活法人实体客户添加新受益所有人 |

> [!NOTE] 原文未提供 `POST /roles/onboardings` 和 `POST /entities/onboardings` 的请求体字段定义。Customer Onboarding 以 Customer ID 发起，Proxy Onboarding 通过指定 onboarding type 区分，BO Onboarding 的请求参数均未详细说明。

## Onboarding 状态流转

Onboarding 自身与关联实体/角色在入驻过程中各自独立管理状态。

**Onboarding 状态：**

```
  CREATED ---------> PENDING ---------> APPROVED
  (initiated)       (processing)        (all passed)
                        |
                        +-------> REJECTED
                                  (validation failed)
```

**关联实体/角色状态（入驻期间）：**

```
  CREATED ---------> PENDING ---------> ACTIVE
                        |               (activated)
                        |
                        +-------> REVIEW -------> ACTIVE
                        |         (admin task)    (admin accepts)
                        |              |
                        |              +-------> REJECTED
                        |                        (admin rejects)
                        |
                        +-------> REJECTED
                                  (critical failure)
```

## Webhook 通知

入驻过程中 Partner 通过以下 Webhook 接收所有状态变更通知：

| 通知类型 | 说明 |
|----------|------|
| Onboarding Notification | 入驻流程状态 |
| Customer Notification | 客户角色状态 |
| Natural Person Notification | 自然人实体状态 |
| Joint Person Notification | 联名人实体状态 |
| Legal Entity Notification | 法人实体状态 |
| Beneficial Owner Notification | 受益所有人状态 |
| Proxy Notification | 代理人角色状态 |
| Document Notification | 文档状态 |

---

## Customer Onboarding（客户入驻）

Customer Onboarding 是最全面的入驻类型，处理客户角色及其所有关联实体（代理人、受益所有人）的验证与激活。

**关键特性：** Customer Onboarding **不执行任何同步 API 级校验**。所有校验、验证和处理均在入驻发起后异步执行。

### 异步处理流程

```
  POST /roles/onboardings (Customer ID)
                       |
                       v
  +------------------------------------------------+
  | 1. Create Onboarding Record (status: CREATED)  |
  +------------------------------------------------+
                       |
                       v
  +------------------------------------------------+
  | 2. Fetch Customer Compound Data                |
  |    (Customer + all related entities)           |
  +------------------------------------------------+
                       |
                       v
  +------------------------------------------------+
  | 3. Fetch Relations                             |
  |    - Proxies in CREATED status                 |
  |    - Beneficial Owners in CREATED status       |
  +------------------------------------------------+
                       |
                       v
  +------------------------------------------------+
  | 4. Fetch Required Documents                    |
  +------------------------------------------------+
                       |
                       v
  +------------------------------------------------+
  | 5. Execute All Validations                     |
  |    - Entity & Customer status                  |
  |    - Document presence & signatures            |
  |    - Proxy & relationship rules                |
  |    - Business logic (KYC, Vendor)              |
  |    - Reference account IBAN                    |
  +------------------------------------------------+
              |              |             |
              v              v             v
     +---------------+ +-----------+ +------------+
     |   Immediate   | | Onboarding| |  Pass /    |
     |   Rejection   | | Only      | |  Manual    |
     |               | | Rejection | |  Review    |
     +---------------+ +-----------+ +------------+
           |               |              |
           v               v              v
      ALL entities    Onboarding     PENDING →
      and roles →     → REJECTED;    KYC auto-
      REJECTED        entities stay  triggered
                      CREATED
```

### 校验失败结果分类

| 类别 | 触发条件 | Onboarding | Customer / 实体 |
|------|----------|------------|-----------------|
| **Immediate Rejection** | 严重错误：无效状态、已设置 deathDay 等 | REJECTED | **全部 REJECTED** |
| **Onboarding Only Rejection** | 非严重错误：缺少文档、缺少代理人等 | REJECTED | **保持 CREATED**（可修复） |
| **Manual Review** | Vendor 数据差异、未知 legalForm | — | REVIEW（创建 Admin Task） |

> **重新入驻：** Onboarding Only Rejection 后，Partner 修正问题（补充文档/代理人等），然后发起**新的入驻请求**（非重试机制）。

### Natural Person Customer 校验

**实体状态校验：**

| 校验项 | 要求 |
|--------|------|
| Natural Person 状态 | `CREATED` 或 `ACTIVE` |
| Customer 状态 | `CREATED` |
| Death Day | 入驻时**不可已设置** |

**文档要求：**

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | **必需** |
| `PROOF_OF_RESIDENCE` | 主地址非德国（country ≠ DE）时必需 |
| `BIRTH_CERTIFICATE` | 未成年人（< 18 岁）必需 |

**其他校验：**
- 身份识别（Identification）必须已完成且有效
- 必需文档须有签署记录（Document Signatures）

### Legal Entity Customer 校验

**实体状态校验：**

| 校验项 | 要求 |
|--------|------|
| Legal Entity 状态 | `CREATED` 或 `ACTIVE` |
| Customer 状态 | `CREATED` |

**文档要求（按 legalForm 不同）：**

| 文档类型 | 适用场景 |
|----------|----------|
| `CURRENT_REGISTRY_EXTRACT` | 大多数法律形式 |
| `SHAREHOLDER_LIST` | 特定法律形式 |
| `PARTNERSHIP_AGREEMENT` | 合伙企业 |
| `TRANSPARENCY_REGISTER_EXTRACT` | 特定场景 |
| `BUSINESS_REGISTRATION` | 特定法律形式 |
| `STATUTE` | 特定法律形式 |

**Vendor 数据验证：**
- 与外部法人实体数据供应商比对提交数据
- 检测差异 → 创建 Admin Task 进行人工审核
- 未知 legalForm → 触发手动验证（Manual Verification）

### Joint Person Customer 校验

**实体状态校验：**

| 校验项 | 要求 |
|--------|------|
| 两个 NP 状态 | `CREATED` 或 `ACTIVE` |
| 两个 NP ID | **必须不同** |
| Customer 状态 | `CREATED` |
| 两个 NP Death Day | 均不可已设置 |
| 年龄 | 两人均须年满 **18 岁** |

**文档要求（两个 NP 各自需要）：**

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | **必需** |
| `PROOF_OF_RESIDENCE` | 主地址非德国（country ≠ DE）时必需 |

**其他校验：**
- 双方身份识别均须已完成且有效
- 双方必需文档均须有签署记录

### Proxy 校验（Customer Onboarding 阶段）

Customer Onboarding 同时校验所有 CREATED 状态的关联代理人。

#### Guardian

| 校验项 | 要求 |
|--------|------|
| 适用场景 | 客户为未成年人（< 18 岁）时**至少需 1 个** |
| 监护人年龄 | ≥ 18 岁 |
| 与客户关系 | 不可为同一人 |
| 监护权类型 | `JOINT_CUSTODY` 需 2 名；`SINGLE_CUSTODY` 允许 1 名 |
| NP 状态 | `CREATED` 或 `ACTIVE` |
| 身份识别 | 已完成且有效 |
| 文档 | `IDENTIFICATION_CERTIFICATE`（必需），Partner Documents 已签署 |
| Proxy 状态 | `CREATED` 或 `ACTIVE` |
| Proxy 文档 | `PROOF_OF_CUSTODY`（始终必需），`PROOF_OF_SINGLE_CUSTODY`（SINGLE_CUSTODY 额外需要） |

#### Signatory

| 校验项 | 要求 |
|--------|------|
| NP 状态 | `CREATED` 或 `ACTIVE` |
| 身份识别 | 已完成且有效 |
| 文档 | `IDENTIFICATION_CERTIFICATE`（必需），Partner Documents 已签署 |
| Proxy 状态 | `CREATED` 或 `ACTIVE` |
| 法定代表人匹配 | 必须是该法人实体的法定代表人 |
| scopeType | `soleSignatureAuthorized = true` → `INDIVIDUAL`；`= false` → `JOINT` |

#### General Power of Attorney

| 校验项 | 要求 |
|--------|------|
| NP 状态 | `CREATED` 或 `ACTIVE` |
| 身份识别 | 已完成且有效 |
| 文档 | `IDENTIFICATION_CERTIFICATE`（必需），Partner Documents 已签署 |
| Proxy 状态 | `CREATED` 或 `ACTIVE` |
| IN_CASE_OF_DEATH 文档 | `INHERITANCE_LEGITIMATION`（必需） |

#### Joint Account Holder

- 在 Joint Person Customer Onboarding 中**由系统自动创建**，状态直接为 `ACTIVE`
- 为联名人的两个自然人各创建一个 Joint Account Holder Proxy
- **不可手动创建**，纯系统生成
- 校验遵循标准自然人代理人规则

### KYC 与合规集成

所有校验通过后，系统自动触发 KYC 验证——将客户及关联实体发送至自动化合规筛查系统。

```
  All Validations Passed
            |
            v
  +-----------------------------------+
  | KYC Verification                  |
  | (automated compliance screening) |
  +-----------------------------------+
       |          |         |         |
       v          v         v         v
  +---------+ +--------+ +------+ +--------+
  |  Valid  | | Manual | |Repeat| |Rejected|
  |         | | Review | | KYC  | |        |
  +---------+ +--------+ +------+ +--------+
       |          |         |         |
       v          v         v         v
  Auto-       Admin     New round  Entities
  activate    Task      triggered  may be
  all         created              REJECTED
  entities
```

---

## Proxy Onboarding（代理人入驻）

专门用于为**已激活客户**添加新代理人。仅处理代理人角色，不涉及完整客户档案。

### 前置条件

| 条件 | 要求 |
|------|------|
| 大多数代理类型 | Customer 状态须为 `ACTIVE` |
| GPoA（`IN_CASE_OF_DEATH`）| NP 状态须为 `DECEASED`；Customer 状态须为 `ESTATE` |

> [!NOTE] 原文未解释客户何时或如何转变为 `ESTATE` 状态。

### 校验规则

**实体状态校验：**

| 校验项 | 通常要求 | GPoA IN_CASE_OF_DEATH |
|--------|---------|----------------------|
| NP 状态 | `CREATED` 或 `ACTIVE` | `DECEASED` |
| Customer 状态 | `ACTIVE` | `ESTATE` |
| Death Day | 不可已设置 | 不适用（已 DECEASED） |
| Proxy 状态 | `CREATED` 或 `ACTIVE` | `CREATED` 或 `ACTIVE` |

**文档校验：**

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | NP 必需 |
| 身份识别 | 已完成且有效 |
| Partner Documents | 已签署 |
| `PROOF_OF_CUSTODY` / `PROOF_OF_SINGLE_CUSTODY` | Guardian 必需 |
| `INHERITANCE_LEGITIMATION` | GPoA（IN_CASE_OF_DEATH）必需 |

**业务逻辑校验：**
- Guardian：监护权类型逻辑（与 Customer Onboarding 一致）
- Signatory：法定代表人匹配、scopeType 验证
- KYC 自动触发

---

## Beneficial Owner Onboarding（受益所有人入驻）

专门用于为**已激活法人实体客户**添加新受益所有人。仅处理受益所有人实体。

### 前置条件

- Legal Entity 状态须为 `ACTIVE`
- 该法人实体已有活跃的 Customer 角色

### 校验规则

**实体状态校验：**

| 校验项 | 要求 |
|--------|------|
| BO 状态 | `CREATED` |
| Legal Entity 状态 | `ACTIVE` |
| Death Day | 不可已设置 |

**业务逻辑校验：**

| 校验项 | 规则 |
|--------|------|
| BO 类型 | 需与持股/投票权一致 |
| Share > 0 | 类型必须为 `REAL_UBO_25` |
| Voting Rights > 0 | 类型必须为 `REAL_UBO_25` |
| `uboRelationship` | `REAL_UBO_25` 时必填 |
| FATCA | controlling person 与 LE 的 FATCA 分类一致 |
| 所有权结构 | 法人实体所有权结构校验 |

**文档校验：**

| 文档类型 | 条件 |
|----------|------|
| `IDENTIFICATION_CERTIFICATE` | 成人必需 |
| `BIRTH_CERTIFICATE` | 未成年人（< 18 岁）必需 |
| `PROOF_OF_RESIDENCE` | 主地址非德国（country ≠ DE）时必需 |
| `TIN_NA_CONFIRMATION` | `NoTinConfirmation = true` 时必需 |
| Partner Documents | 已签署 |
| 地址白名单 | 地址须在白名单国家内 |

---

## 入驻序列流程

以下序列图展示三方交互：

| 角色 | 缩写 | 说明 |
|------|------|------|
| **P** (Client) | Partner's API Client | 发起 API 调用，接收同步 OK 响应 |
| **T** (API) | Tradevest API | 处理请求，返回同步响应，触发异步通知 |
| **PW** (Webhook) | Partner's Webhook | 接收异步 Webhook 状态变更通知 |

> **约定：** `P → T` 为同步请求/响应（OK）；`T → PW` 为异步 Webhook 推送。P 与 PW 之间无直接交互。序列图使用 `[Alt]`/`[Opt]`/`[Loop]` 标注条件分支和循环。

### Natural Person Customer Onboarding 序列

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      | POST /entities/natural-persons                         |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      |  [Alt: Document Upload]                                |
      | POST /entities/NP/{id}/identifications                 |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: UPDATED          |
      |                           |--------------------------->|
      | POST /v2/documents [loop]                              |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |  [Or: External Verifier]                               |
      | POST /entities/identification-verifications            |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |  (Partner redirects user to identificationUrl)         |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |                           | NP Notif: UPDATED          |
      |                           |--------------------------->|
      |  [End Alt]                |                            |
      |                           |                            |
      | POST /v2/documents/sign                                |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: UPDATED          |
      |                           |--------------------------->|
      |                           |                            |
      | POST /roles/customers                                  |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Customer Notif: CREATED    |
      |                           |--------------------------->|
      |                           |                            |
      |  [Opt: Guardian — loop per guardian]                    |
      |  POST /entities/natural-persons (Guardian)             |
      |  ----------------------->|                             |
      |  OK                      |                             |
      |  <-----------------------|                             |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |  (Identification + Documents for Guardian NP)          |
      |  POST /roles/proxies                                   |
      |  ----------------------->|                             |
      |  OK                      |                             |
      |  <-----------------------|                             |
      |                           | Proxy Notif: CREATED       |
      |                           |--------------------------->|
      |    [Opt: SINGLE_CUSTODY proxy docs]                    |
      |    POST /v2/documents [loop]                           |
      |    --------------------->|                             |
      |    OK                    |                             |
      |    <---------------------|                             |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |    [End Opt]              |                            |
      |  [End Opt: Guardian]      |                            |
      |                           |                            |
      | POST /roles/onboardings (Customer)                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Onboarding: CREATED        |
      |                           |--------------------------->|
      |                           |                            |
      |         [ Async Processing ]                           |
      |                           |                            |
      |                           | Onboarding: PENDING        |
      |                           |--------------------------->|
      |                           | NP Notif: PENDING          |
      |                           |--------------------------->|
      |                           | Customer: PENDING          |
      |                           |--------------------------->|
      |  [Opt: Child Customer]    |                            |
      |                           | Proxy: PENDING [loop]      |
      |                           |--------------------------->|
      |                           | NP (Guardian): PENDING     |
      |                           |--------------------------->|
      |                           | Doc Notifs: PENDING [loop] |
      |                           |--------------------------->|
      |  [End Opt]                |                            |
      |                           | Doc Notifs: PENDING [loop] |
      |                           |--------------------------->|
      |                           |                            |
      |         [ On Success ]                                 |
      |                           |                            |
      |                           | Onboarding: APPROVED       |
      |                           |--------------------------->|
      |                           | NP Notif: ACTIVE           |
      |                           |--------------------------->|
      |                           | Customer: ACTIVE           |
      |                           |--------------------------->|
      |  [Opt: Child Customer]    |                            |
      |                           | Proxy: ACTIVE [loop]       |
      |                           |--------------------------->|
      |                           | NP (Guardian): ACTIVE      |
      |                           |--------------------------->|
      |                           | Doc Notifs: APPROVED [loop]|
      |                           |--------------------------->|
      |  [End Opt]                |                            |
      |                           | Doc Notifs: APPROVED [loop]|
      |                           |--------------------------->|
```

**失败场景：**

| 失败类型 | 状态变更 |
|----------|----------|
| Manual Review Required | NP → `REVIEW`（等待管理员审核） |
| NP Verification Failed | NP → `REJECTED` |
| Customer Verification Failed | Customer → `REJECTED` |
| Onboarding Validation Failed | Onboarding / NP / Customer / Documents → `REJECTED` |
| KYC Verification Failed | → `REVIEW`（等待管理员决定） |

### Legal Entity Customer Onboarding 序列

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      |  [Opt: Vendor Data Preparation]                        |
      | GET /entities/vendor/legal-entities                    |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      | POST /entities/vendor/legal-entities                   |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | LE Search Notif: COMPLETED |
      |                           |--------------------------->|
      |  [End Opt]                |                            |
      |                           |                            |
      | POST /entities/legal-entities                          |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | LE Notif: CREATED          |
      |                           |--------------------------->|
      |  [Opt: auto-created documents from vendor data]        |
      |                           | Doc Notifs: CREATED [loop] |
      |                           |--------------------------->|
      |  [End Opt]                |                            |
      |                           |                            |
      |  [Loop: Legal Representatives]                         |
      | POST /entities/{leId}/legal-representatives            |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | LR Notif: CREATED          |
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           |                            |
      |  [Opt Loop: Beneficial Owners]                         |
      | POST /entities/{leId}/beneficial-owners                |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | BO Notif: CREATED          |
      |                           |--------------------------->|
      |  [End Opt Loop]           |                            |
      |                           |                            |
      |  [Opt Loop: LE Documents]                              |
      | POST /v2/documents                                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |  [End Opt Loop]           |                            |
      |                           |                            |
      | POST /roles/customers                                  |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Customer Notif: CREATED    |
      |                           |--------------------------->|
      |                           |                            |
      |  [Loop: Proxies (e.g. Signatories)]                    |
      |    [Opt: Create NP for proxy]                          |
      |    POST /entities/natural-persons                      |
      |    --------------------->|                             |
      |    OK                    |                             |
      |    <---------------------|                             |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |    (Identification for proxy NP)                       |
      |    --------------------->|                             |
      |    [End Opt]              |                            |
      |    POST /v2/documents [NP docs, loop]                  |
      |    --------------------->|                             |
      |    OK                    |                             |
      |    <---------------------|                             |
      |                           | Doc Notifs: CREATED        |
      |                           |--------------------------->|
      |    POST /roles/proxies                                 |
      |    --------------------->|                             |
      |    OK                    |                             |
      |    <---------------------|                             |
      |                           | Proxy Notif: CREATED       |
      |                           |--------------------------->|
      |    POST /v2/documents [Proxy docs, loop]               |
      |    --------------------->|                             |
      |    OK                    |                             |
      |    <---------------------|                             |
      |                           | Doc Notifs: CREATED        |
      |                           |--------------------------->|
      |  [End Loop: Proxies]      |                            |
      |                           |                            |
      | POST /roles/onboardings (Customer)                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Onboarding: CREATED        |
      |                           |--------------------------->|
      |                           |                            |
      |         [ Async Processing ]                           |
      |                           |                            |
      |                           | Onboarding: PENDING        |
      |                           |--------------------------->|
      |                           | LE Notif: PENDING          |
      |                           |--------------------------->|
      |                           | Customer: PENDING          |
      |                           |--------------------------->|
      |                           | BO Notifs: PENDING [loop]  |
      |                           |--------------------------->|
      |  [Loop: Proxies]          |                            |
      |                           | NP Notif: PENDING          |
      |                           |--------------------------->|
      |                           | Proxy Notif: PENDING       |
      |                           |--------------------------->|
      |                           | Doc Notifs: PENDING [loop] |
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           | Doc Notifs: PENDING [loop] |
      |                           |--------------------------->|
      |                           |                            |
      |         [ On Success ]                                 |
      |                           |                            |
      |                           | Onboarding: APPROVED       |
      |                           |--------------------------->|
      |                           | LE Notif: ACTIVE           |
      |                           |--------------------------->|
      |                           | Customer: ACTIVE           |
      |                           |--------------------------->|
      |                           | BO Notifs: ACTIVE [loop]   |
      |                           |--------------------------->|
      |  [Loop: Proxies]          |                            |
      |                           | Proxy Notif: ACTIVE        |
      |                           |--------------------------->|
      |                           | NP Notif: ACTIVE           |
      |                           |--------------------------->|
      |                           | Doc Notifs: APPROVED [loop]|
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           | Doc Notifs: APPROVED [loop]|
      |                           |--------------------------->|
```

**失败场景：**

| 失败类型 | 影响范围 | 可修复 |
|----------|----------|--------|
| Manual Review | LE → `REVIEW`；如管理员拒绝 → Onboarding / LE / Customer / BO / Proxy / NP 全部 `REJECTED` | — |
| Fixable Issues | Onboarding → `REJECTED`；LE / Customer / BO / Proxy / NP **保持 `CREATED`** | 是 |
| Critical Issues | Onboarding / LE / Customer / BO / Proxy / NP → **全部 `REJECTED`** | 否 |
| KYC (admin accepts) | 所有实体/角色 → `ACTIVE` | — |
| KYC (admin rejects) | Onboarding → `REJECTED` + 全局拒绝（LE / Customer / Proxy / Proxy Customers / BO） | 否 |
| Resource Validation | 单个资源 → `INVALID`（Partner 收到具体错误，修正后重试） | 是 |

> **Fixable vs Critical：** Fixable Issues（如缺少文档或代理人）允许 Partner 修正后发起新入驻请求。Critical Issues（如敏感数据不合规或实体不应被入驻）导致所有关联实体被拒绝，不可恢复。

### Joint Person Customer Onboarding 序列

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      | POST /entities/natural-persons (NP1)                   |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      |  [Alt: Document Upload / External Verifier for NP1]    |
      |  (same as NP Customer flow above)                      |
      |  [End Alt]                |                            |
      |                           |                            |
      | POST /entities/natural-persons (NP2)                   |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      |  [Alt: Document Upload / External Verifier for NP2]    |
      |  (same as NP Customer flow above)                      |
      |  [End Alt]                |                            |
      |                           |                            |
      | POST /entities/joint-persons                           |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | JP Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      |  [Loop: Documents for both NPs (Document Upload only)] |
      | POST /v2/documents                                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           |                            |
      | POST /roles/customers                                  |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Customer Notif: CREATED    |
      |                           |--------------------------->|
      |                           |                            |
      | POST /roles/onboardings (Customer)                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Onboarding: CREATED        |
      |                           |--------------------------->|
      |                           |                            |
      |         [ Async Processing ]                           |
      |                           |                            |
      |                           | Onboarding: PENDING        |
      |                           |--------------------------->|
      |                           | JP Notif: PENDING          |
      |                           |--------------------------->|
      |                           | NP1 Notif: PENDING         |
      |                           |--------------------------->|
      |                           | NP2 Notif: PENDING         |
      |                           |--------------------------->|
      |                           | Customer: PENDING          |
      |                           |--------------------------->|
      |                           |                            |
      |         [ On Success ]                                 |
      |                           |                            |
      |                           | Onboarding: APPROVED       |
      |                           |--------------------------->|
      |                           | JP Notif: ACTIVE           |
      |                           |--------------------------->|
      |                           | NP1 Notif: ACTIVE          |
      |                           |--------------------------->|
      |                           | NP2 Notif: ACTIVE          |
      |                           |--------------------------->|
      |                           | Customer: ACTIVE           |
      |                           |--------------------------->|
```

**自动行为：** 系统为联名人的两个自然人各自动创建一个 **Joint Account Holder** Proxy（状态 `ACTIVE`）。此类代理人不可手动创建。

**失败场景：**

| 失败类型 | 状态变更 |
|----------|----------|
| Manual Review | NP（任一）→ `REVIEW` |
| NP Verification Failed | NP（任一）→ `REJECTED` |
| Customer Verification Failed | Customer → `REJECTED` |
| Onboarding Validation Failed | Onboarding / JP / Customer / Documents → `REJECTED` |
| KYC (admin accepts) | Onboarding / Customer / NP1 / NP2 / Documents / Proxy / Proxy Documents → `ACTIVE`/`APPROVED` |
| KYC (admin rejects) | 全局拒绝：NP1 / NP2 / JP / Customer / Proxies → `REJECTED` |

> [!WARNING] **JP 全局拒绝范围：** KYC 拒绝时，原文提及会级联拒绝 "Proxies customers (when proxy type is other than General\_Power\_of\_Attorney / Liquidator / Information\_Proxy)"。即部分代理类型关联的客户实体也会被拒绝，但具体规则不够清晰。

### Proxy Onboarding 序列

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      | POST /entities/natural-persons                         |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | NP Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      |  [Alt: Document Upload / External Verifier]            |
      |  (same as NP Customer flow)                            |
      |  [End Alt]                |                            |
      |                           |                            |
      |  [Loop: NP Documents]                                  |
      | POST /v2/documents                                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           |                            |
      | POST /roles/proxies                                    |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Proxy Notif: CREATED       |
      |                           |--------------------------->|
      |                           |                            |
      |  [Loop: Proxy Documents]                               |
      | POST /v2/documents                                     |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Doc Notif: CREATED         |
      |                           |--------------------------->|
      |  [End Loop]               |                            |
      |                           |                            |
      | POST /roles/onboardings (Proxy)                        |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Onboarding: CREATED        |
      |                           |--------------------------->|
      |                           |                            |
      |         [ Async Processing ]                           |
      |                           |                            |
      |                           | Onboarding: PENDING        |
      |                           |--------------------------->|
      |                           | NP Notif: PENDING          |
      |                           |--------------------------->|
      |                           | Proxy Notif: PENDING       |
      |                           |--------------------------->|
      |                           |                            |
      |         [ On Success ]                                 |
      |                           |                            |
      |                           | Onboarding: APPROVED       |
      |                           |--------------------------->|
      |                           | NP Notif: ACTIVE           |
      |                           |--------------------------->|
      |                           | Proxy Notif: ACTIVE        |
      |                           |--------------------------->|
```

**失败场景：**

| 失败类型 | 状态变更 |
|----------|----------|
| NP Verification Failed | NP → `REJECTED` |
| Proxy Creation Failed | Proxy → `REJECTED` |
| Onboarding Validation Failed | Onboarding / Proxy / NP / Documents → `REJECTED` |
| KYC Verification Failed | → `REVIEW`（等待管理员决定） |

### Beneficial Owner Onboarding 序列

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      | POST /entities/{leId}/beneficial-owners                |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | BO Notif: CREATED          |
      |                           |--------------------------->|
      |                           |                            |
      | POST /entities/onboardings (BO)                        |
      |-------------------------->|                            |
      | OK                        |                            |
      |<--------------------------|                            |
      |                           | Onboarding: CREATED        |
      |                           |--------------------------->|
      |                           |                            |
      |         [ Async Processing ]                           |
      |                           |                            |
      |                           | Onboarding: PENDING        |
      |                           |--------------------------->|
      |                           | BO Notif: PENDING          |
      |                           |--------------------------->|
      |                           |                            |
      |         [ On Success ]                                 |
      |                           |                            |
      |                           | Onboarding: APPROVED       |
      |                           |--------------------------->|
      |                           | BO Notif: ACTIVE           |
      |                           |--------------------------->|
```

**失败场景：**

| 失败类型 | 状态变更 |
|----------|----------|
| BO Creation Failed | BO → `REJECTED` |
| Onboarding Validation Failed | Onboarding / BO → `REJECTED` |
| KYC Verification Failed | → `REVIEW`（等待管理员决定） |

---

## 注意事项与内容缺口

> [!NOTE] **Reference Account IBAN 校验：** 原文提及 Customer Onboarding 包含参考账户 IBAN 校验，但未提供具体字段名或校验规则。

> [!NOTE] **ESTATE 状态：** Proxy Onboarding 中 GPoA（IN\_CASE\_OF\_DEATH）需要 Customer 处于 `ESTATE` 状态，但原文**未解释客户何时或如何转变为 ESTATE 状态**。

> [!NOTE] **Resource Validation Failed（INVALID）：** 此类失败发生在准备阶段（入驻前），单个资源（NP / LE / BO / Document 等）校验不通过时状态设为 `INVALID`，Partner 收到具体错误通知后可修正重试。

## 相关文档

- [Entity And Role Management](./entity-and-role-management.md) — 实体类型与关系总览
- [Natural Person Management](./natural-person-management.md) — 自然人准备流程、身份识别、文档签署
- [Legal Entity Management](./legal-entity-management.md) — 法人实体准备流程、Vendor 数据
- [Role Management](./role-management.md) — 角色类型与分配流程
- [Beneficial Owner Create Process](./beneficial-owner-create-process.md) — 受益所有人创建流程（含 FICTIVE_UBO）
- [Proxy Create Process](./proxy-create-process.md) — 代理人创建校验与规则
- [Document Management](./document-management.md) — 文档类型、上传、状态生命周期

### API Reference

- [Entities Onboardings](https://docs.tradevest.ai/api-reference/entities/onboardings) — 实体入驻接口（BO Onboarding）
- [Roles Onboardings](https://docs.tradevest.ai/api-reference/roles/onboardings) — 角色入驻接口（Customer / Proxy Onboarding）
