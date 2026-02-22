# Legal Entity Onboarding — Partner Ident Service

> Source: https://docs.tradevest.ai/api-guidelines/onboarding/legal-entity-partner-ident-service

## 概述

使用 Partner 自行完成的外部身份识别（Partner Ident）完成法人实体客户入驻的完整流程指南。与 [NP Tradevest Ident](./np-onboarding-tradevest-ident.md) 流程不同，此流程中 Partner 负责对自然人进行身份识别并将结果传递给 Tradevest。

整个流程为**异步设计**，每个 API 调用立即返回 UUID 或 HTTP 状态码，后续状态通过 Webhook 或 GET 轮询获取。

### 必要步骤总览（13 步）

```
   0. Subscribe to Webhooks              配置 8 种事件类型
                 |
                 v
   1. Create Legal Entity                POST /entities/legal-entities
                 |
          +------+------+
          |             |
          v             v
   2. Create Legal      3. Create Beneficial
      Representatives      Owners
      POST /entities/      POST /entities/
      {leId}/legal-        {leId}/beneficial-
      representatives      owners
          |             |
          +------+------+
                 |
                 v
   4. Create Customer Role               POST /roles/customers
                 |
                 v
   5. Upload LE Documents                POST /v2/documents
                 |
                 v
   6. Create Natural Person (Proxy)      POST /entities/natural-persons
                 |
                 v
   7. Create NP Identification           POST /entities/natural-persons/{npId}/identification
                 |
                 v
   8. Upload Identification Certificate  POST /v2/documents
                 |
                 v
   9. Create Proxy (SIGNATORY)           POST /roles/proxies
                 |
                 v
  10. Sign T&C + Data Privacy Policy     POST /v2/documents/sign
                 |
                 v
  11. Start Onboarding                   POST /roles/onboardings
                 |
                 v
  12. Monitor Onboarding Status          GET /roles/onboardings/{onboardingId}
                 |
                 v
      APPROVED → Customer fully activated
```

---

## Step 0: Webhook 配置

须注册以下 **8 种** Webhook 事件类型：

| 事件类型 | 说明 |
|----------|------|
| `LEGAL_ENTITY_NOTIFICATION` | 法人实体状态变更 |
| `LEGAL_REPRESENTATIVE_NOTIFICATION` | 法定代表人状态变更 |
| `BENEFICIAL_OWNER_NOTIFICATION` | 受益所有人状态变更 |
| `PROXY_NOTIFICATION` | 代理人状态变更 |
| `CUSTOMER_NOTIFICATION` | 客户状态变更 |
| `NATURAL_PERSON_NOTIFICATION` | 自然人状态变更 |
| `DOCUMENT_NOTIFICATION` | 文档状态变更 |
| `ONBOARDING_NOTIFICATION` | 入驻状态变更 |

> [!NOTE] **详细的错误原因仅通过 Webhook 通知返回。** 若未配置 Webhook，Partner 须主动轮询对应 GET 端点。

---

## Step 1: 创建法人实体

### 端点

```
POST /entities/legal-entities
```

### 业务规则

- `legalEntityIdentifier`：若需创建 `productCode: DEPOSITORY_ACCOUNT` 的产品，则**必须提供**
- 创建后状态为 `CREATED`，直到入驻流程显式触发

### 请求示例

```json
{
  "legalName": "TechNova Solutions GmbH",
  "legalForm": "LIMITED_LIABILITY_COMPANY",
  "naceSectors": [
    {
      "section": "C",
      "code": "26.40",
      "description": "Manufacture of consumer electronics"
    }
  ],
  "registerCountry": "DE",
  "registerNumber": "HRB123456",
  "registerCourt": "Amtsgericht München",
  "foundedOn": "2015-03-15",
  "mainAddress": {
    "street": "Maximilianstraße",
    "streetNumber": "12",
    "city": "Munich",
    "zip": "80333",
    "state": "Bavaria",
    "country": "DE"
  },
  "contact": {
    "phone": "+49 89 12345678",
    "email": "info@technovasolutions.de"
  },
  "taxNumber": "DE123456789",
  "fatcaCrsDeclaration": {
    "isForeignTaxResidency": false,
    "isSanctionedCountries": false,
    "fatcaClassification": "ACTIVE_NFE",
    "activeNfeType": "LE_BY_INCOME_ASSETS"
  }
}
```

### 响应

```json
{
  "legalEntityId": "90d741f0-f8ba-4f9b-9b87-18d4bff1a941"
}
```

查询 `GET /entities/legal-entities/{legalEntityId}` 返回系统生成字段：`globalId`、`createdOn`、`modifiedOn`、`legalEntityStatus`。

---

## Step 2: 创建法定代表人

每个法人实体**至少需要一个**法定代表人。法定代表人后续可被分配为 Proxy（Signatory）角色，但在 API 中是两个独立资源。

### 端点

```
POST /entities/{legalEntityId}/legal-representatives
```

### 字段要求对比（LR vs Proxy NP）

| 字段 | 法定代表人 | 自然人（作为 Proxy） |
|------|-----------|---------------------|
| `gender` | — | 必填 |
| `firstName` | 必填 | 必填 |
| `lastName` | 必填 | 必填 |
| `birthDay` | 必填 | 必填 |
| `birthPlace` | 必填 | 必填 |
| `birthCountry` | 必填 | 必填 |
| `isUsNationality` | 必填 | 必填 |
| `nationalities` | 必填 | 必填 |
| `function` | 必填 | — |
| `soleSignatureAuthorized` | 必填 | — |
| `fatcaControllingPerson` | 必填 | — |
| `taxDetails.taxId` | 仅 `fatcaControllingPerson: true` 时必填 | 非 DE 税务居住地时必填 |
| `taxDetails.taxResidency` | 仅 `fatcaControllingPerson: true` 时必填 | 必填 |
| `mainAddress` | — | 必填 |
| `contact.phone` | — | 必填 |
| `contact.email` | — | 必填 |

> [!NOTE] 法定代表人的 `nationalities` 为字符串数组 `["DE"]`，而自然人（Proxy）的 `nationalities` 为对象数组 `[{"countryCode": "DE"}]`，格式不同。

### 请求示例

```json
{
  "firstName": "Alexandra",
  "lastName": "Mueller",
  "birthDay": "1985-07-14",
  "birthPlace": "Munich",
  "birthCountry": "DE",
  "isUsNationality": false,
  "nationalities": ["DE"],
  "function": "MANAGING_DIRECTOR",
  "soleSignatureAuthorized": true,
  "fatcaControllingPerson": false
}
```

### 响应

```json
{
  "legalRepresentativeId": "515e5964-b245-431b-9537-a56cf9bce899"
}
```

---

## Step 3: 创建受益所有人

直接或间接持有法人实体 **≥25%** 股权或投票权的自然人须声明为受益所有人（`REAL_UBO_25`）。

若无人达到 25% 阈值，系统根据法定代表人**自动生成** `FICTIVE_UBO`。

### 端点

```
POST /entities/{legalEntityId}/beneficial-owners
```

### 请求示例

```json
{
  "firstName": "Elena",
  "lastName": "Schneider",
  "birthDay": "1979-11-02",
  "birthPlace": "Hamburg",
  "birthCountry": "DE",
  "taxDetails": [
    {
      "taxId": "DE987654321",
      "taxResidency": "DE",
      "noTinConfirmation": false
    }
  ],
  "isUsNationality": false,
  "nationalities": ["DE"],
  "uboRelationship": "DIRECTLY_HOLDING_25",
  "share": 35,
  "votingRights": 35
}
```

### 响应

```json
{
  "beneficialOwnerId": "b2e7cfa9-a912-472a-bee2-9ca7cd971b97"
}
```

---

## Step 4: 创建客户角色

为法人实体分配客户角色并指定参考银行账户。此步骤**不会启动入驻**，仅分配角色。

### 端点

```
POST /roles/customers
```

### 请求示例

```json
{
  "entityId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "entityType": "LEGAL_ENTITY",
  "refAccounts": [
    {
      "bankName": "Deutsche Bank AG",
      "iban": "DE89370400440532013000",
      "bic": "DEUTDEFF",
      "type": "PRIMARY",
      "ownerName": "TechNova Solutions GmbH"
    }
  ]
}
```

### 响应

```json
{
  "customerId": "18320faf-b998-4fab-9912-a14eda099a26"
}
```

---

## Step 5: 上传法人实体文档

KYC/KYB 合规要求。Partner 须执行 KYC/KYB 检查并上传所需文档。

### 端点

```
POST /v2/documents
Header: Requestor-Id: {naturalPersonId}
```

### 技术要求

- **文件格式：** 仅 PDF
- **最大文件大小：** 6 MB

### 按法律形式的必需文档

| 法律形式 | 必需文档类型 |
|----------|------------|
| `LIMITED_LIABILITY_COMPANY` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST` |
| `PUBLIC_LIMITED_COMPANY` | `CURRENT_REGISTRY_EXTRACT`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `FOUNDATION` | `CURRENT_REGISTRY_EXTRACT`, `STATUTE` |
| `ASSOCIATION` | `STATUTE` |
| `REGISTERED_BUSINESSMAN` | `CURRENT_REGISTRY_EXTRACT` |
| `LIMITED_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `GENERAL_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `LIMITED_LIABILITY_COMPANY_AND_LIMITED_PARTNERSHIP` | `CURRENT_REGISTRY_EXTRACT`, `SHAREHOLDER_LIST`, `TRANSPARENCY_REGISTER_EXTRACT` |
| `PARTNERSHIP` | `PARTNERSHIP_AGREEMENT` |

> [!WARNING] **未上传全部必需文档将导致入驻失败。**

### 请求示例

```json
{
  "document": {
    "name": "Registry Extract",
    "resourceType": "LEGAL_ENTITY",
    "type": "CURRENT_REGISTRY_EXTRACT",
    "resourceId": "90d741f0-f8ba-4f9b-9b87-18d4bff1a941"
  },
  "file": "<binary>"
}
```

### 响应

```json
{
  "documentId": "1a1e3a46-2b99-48b8-a8c8-15b747af2095"
}
```

---

## Step 6: 创建自然人（Proxy 候选人）

创建将作为法人实体签署人（Signatory）的自然人。

### 端点

```
POST /entities/natural-persons
```

### 请求示例

```json
{
  "gender": "MALE",
  "firstName": "Lukas",
  "lastName": "Weber",
  "birthDay": "1997-10-20",
  "birthPlace": "Berlin",
  "birthCountry": "DE",
  "nationalities": [
    { "countryCode": "DE" }
  ],
  "taxDetails": [
    {
      "taxId": "4243882385647",
      "taxResidency": "DE"
    }
  ],
  "isUsNationality": false,
  "mainAddress": {
    "street": "Bakalarska",
    "streetNumber": "34",
    "city": "Berlin",
    "zip": "22-333",
    "country": "DE"
  },
  "contact": {
    "phone": "123456789",
    "email": "tvdtester90+707845139@gmail.com"
  }
}
```

### 响应

```json
{
  "naturalPersonId": "cc2a0ccd-734c-4ad9-bbbe-c27eab2fdd7a"
}
```

---

## Step 7: 创建自然人身份识别（Partner Ident）

Partner 自行完成外部身份识别后，将识别结果传递给 Tradevest。

### 端点

```
POST /entities/natural-persons/{naturalPersonId}/identification
```

> [!NOTE] 此端点与 [NP Tradevest Ident](./np-onboarding-tradevest-ident.md) 流程使用的 `POST /entities/identification-verifications`（WebID 集成）不同。Partner Ident 由 Partner 自行完成身份识别并提交结果。

### 请求示例

```json
{
  "externalVerifier": "ID_NOW",
  "identificationType": "VIDEO_IDENT",
  "identificationDate": "2026-01-22",
  "identificationDocumentType": "ID",
  "documentNumber": "3859937928234",
  "documentCountry": "DE",
  "documentIssuer": "DE",
  "documentIssueDate": "2019-08-24",
  "documentExpiryDate": "2050-08-24"
}
```

### 响应

```
202 Accepted
```

---

## Step 8: 上传身份证明文档

为每个 Proxy 自然人上传身份识别证书（总结外部身份识别过程的文档）。

### 端点

```
POST /v2/documents
```

### 请求示例

```json
{
  "document": {
    "name": "Identification Certificate",
    "type": "IDENTIFICATION_CERTIFICATE",
    "resourceType": "NATURAL_PERSON",
    "resourceId": "cc2a0ccd-734c-4ad9-bbbe-c27eab2fdd7a"
  },
  "file": "<binary>"
}
```

### 响应

```json
{
  "documentId": "c4ea16d1-322c-444d-9f72-f32b543ad7ed"
}
```

---

## Step 9: 创建代理人（Signatory）

### 业务规则

| 规则 | 说明 |
|------|------|
| 代理类型 | 此流程仅支持 `proxyType: SIGNATORY` |
| 代表范围 | 当前仅支持 `scopeType: INDIVIDUAL` |
| 数量 | 可创建多个代理人 |
| 入驻后 | 入驻完成后仍可添加或停用代理人 |

### 端点

```
POST /roles/proxies
```

### 请求示例

```json
{
  "proxyType": "SIGNATORY",
  "naturalPersonId": "cc2a0ccd-734c-4ad9-bbbe-c27eab2fdd7a",
  "entityId": "90d741f0-f8ba-4f9b-9b87-18d4bff1a941",
  "entityType": "LEGAL_ENTITY",
  "validityType": "UNLIMITED",
  "scopeType": "INDIVIDUAL"
}
```

### 响应

```json
{
  "proxyId": "16195271-d3da-4507-acc2-797b0236b37b"
}
```

---

## Step 10: 签署 Tradevest 法律文档

仅 `proxyType: SIGNATORY` 的代理人有权签署。

### 签署流程

```
  1. GET /partner-documents?isValid=true&documentType=...   获取有效文档
                  |
                  v
  2. POST /partner-documents/{documentId}/file              下载文档文件
                  |
                  v
  3. 展示文档给用户 → 用户确认接受
                  |
                  v
  4. POST /v2/documents/sign                                提交签署确认
```

### 10.1 获取有效文档

```
GET /partner-documents?isValid=true&documentType=DATA_PRIVACY_POLICY
GET /partner-documents?isValid=true&documentType=TERMS_AND_CONDITIONS
```

响应示例（DATA_PRIVACY_POLICY）：

```json
{
  "data": [
    {
      "documentId": "09f7e302-be3b-4d4a-a654-95a872356dc7",
      "size": 136948,
      "version": 6,
      "validFrom": "2025-12-17",
      "name": "DPP",
      "type": "DATA_PRIVACY_POLICY",
      "createdOn": "2025-12-17T08:38:31.104169414Z",
      "modifiedOn": "2025-12-17T08:38:31.104169538Z"
    }
  ],
  "pagination": {
    "cursor": "",
    "limit": 20
  }
}
```

### 10.2 下载文档文件

```
POST /partner-documents/{documentId}/file
```

返回 Base64 编码的文件内容。

### 10.3 提交签署确认

```
POST /v2/documents/sign
```

```json
{
  "partnerDocumentId": "a23097ae-214b-4083-8f85-f74aaa9993ef",
  "naturalPersonId": "8696b558-d043-4cc1-a872-8b27ea3272f4",
  "customerId": "18320faf-b998-4fab-9912-a14eda099a26"
}
```

```
202 Accepted
```

> 须同时签署 **Terms & Conditions** 和 **Data Privacy Policy**。

---

## Step 11: 启动入驻

确保前述所有步骤正确完成（实体状态有效、文档已上传、必填字段已提供）后，启动入驻。

### 端点

```
POST /roles/onboardings
```

```json
{
  "roleType": "CUSTOMER",
  "roleId": "18320faf-b998-4fab-9912-a14eda099a26"
}
```

```json
{
  "onboardingId": "c1e98eed-c7db-4453-8964-51819a8be6cd"
}
```

### 失败场景

| 场景 | 说明 |
|------|------|
| **可修复问题** | 入驻状态 → `REJECTED`，相关实体回到 `CREATED`。Partner 收到具体校验错误，修正后须**重新发起入驻请求** |
| **严重合规问题** | 所有入驻要素被拒绝，涉及敏感数据或合规违规 |

---

## Step 12: 监控入驻状态

### 端点

```
GET /roles/onboardings/{onboardingId}
```

### 状态说明

| 状态 | 含义 |
|------|------|
| `INVALID` | 创建的实体状态不正确 |
| `PENDING` | 实体校验通过，背景检查进行中 |
| `APPROVED` | 所有背景检查完成，客户已激活 |
| `REVIEW` | 非德国注册公司（自动检查不完整）或部分德国公司需 Tradevest 运营团队人工审核 |
| `REJECTED` | 合规和风险规则未通过 |

> [!NOTE] **非德国注册公司**的入驻状态将**始终为 `REVIEW`**，因为自动背景检查未完全覆盖。

### 响应示例

```json
{
  "onboardingId": "905050b4-e19c-4d0b-9d47-94611d1728d2",
  "status": "APPROVED",
  "roleType": "CUSTOMER",
  "roleId": "65f4dca7-b03e-423c-a32f-c04a7a7323d0",
  "createdOn": "2026-01-23T09:42:39.593204Z",
  "modifiedOn": "2026-01-23T10:00:08.256311Z"
}
```

---

## 与 NP Tradevest Ident 流程的关键差异

| 对比项 | NP Tradevest Ident | LE Partner Ident |
|--------|-------------------|-----------------|
| 入驻主体 | 自然人 | 法人实体 |
| 身份识别方 | Tradevest（WebID 集成） | Partner（外部完成后提交结果） |
| 识别端点 | `POST /entities/identification-verifications` | `POST /entities/natural-persons/{npId}/identification` |
| 步骤数 | 5 | 13 |
| Webhook 事件 | 5 种 | 8 种 |
| 额外实体 | — | Legal Representative、Beneficial Owner |
| 代理类型 | — | SIGNATORY（必须） |
| LE 文档要求 | — | 按 legalForm 差异化 |
| REVIEW 状态 | 不适用 | 非德国公司始终进入 |

---

## 相关文档

- [NP Onboarding — Tradevest Ident](./np-onboarding-tradevest-ident.md) — 自然人入驻流程（WebID 身份识别）
- [Onboarding](../documentation/onboarding.md) — 三种入驻类型、异步校验、KYC、序列图
- [Entity And Role Management](../documentation/entity-and-role-management.md) — 实体类型与关系总览
- [Legal Entity Management](../documentation/legal-entity-management.md) — 法人实体准备流程
- [Legal Entity Create Process](../documentation/legal-entity-create-process.md) — 法人实体创建校验规则
- [Legal Representative Create Process](../documentation/legal-representative-create-process.md) — 法定代表人创建流程
- [Beneficial Owner Create Process](../documentation/beneficial-owner-create-process.md) — 受益所有人创建流程
- [Proxy Create Process](../documentation/proxy-create-process.md) — 代理人创建流程
- [Document Management](../documentation/document-management.md) — 文档类型、上传、状态生命周期
- [Partner Webhooks](../documentation/partner-webhooks.md) — Webhook 通信模型

### API Reference

- [Legal Entities](https://docs.tradevest.ai/api-reference/entities/legal-entities) — 法人实体接口
- [Legal Representatives](https://docs.tradevest.ai/api-reference/entities/legal-representatives) — 法定代表人接口
- [Beneficial Owners](https://docs.tradevest.ai/api-reference/entities/beneficial-owners) — 受益所有人接口
- [Natural Persons](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/natural-persons) — 自然人接口
- [Identification](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/identification) — 身份识别接口
- [Customers](https://docs.tradevest.ai/api-reference/roles/customers) — 客户角色接口
- [Proxies](https://docs.tradevest.ai/api-reference/roles/proxies) — 代理人接口
- [Documents v2](https://docs.tradevest.ai/api-reference/documents/new-documents) — 文档管理接口
- [Roles Onboardings](https://docs.tradevest.ai/api-reference/roles/onboardings) — 角色入驻接口
- [Webhooks](https://docs.tradevest.ai/api-reference/webhooks) — Webhook 管理接口
- [Partner Documents](https://docs.tradevest.ai/api-reference/partner-documents) — 合作方文档接口
