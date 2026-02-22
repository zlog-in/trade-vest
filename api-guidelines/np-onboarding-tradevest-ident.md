# Natural Person Onboarding — Tradevest Ident Services

> Source: https://docs.tradevest.ai/api-guidelines/onboarding/natural-person-tradevest-ident-services

## 概述

使用 Tradevest 集成的 WebID 身份识别服务完成自然人客户入驻的完整流程指南。整个流程为**异步设计**，每个 API 调用立即返回 UUID，后续状态通过 Webhook 或 GET 轮询获取。

### 必要步骤总览

```
  1. Create Natural Person          POST /entities/natural-persons
                |
                v
  2. Create Identification          POST /entities/identification-verifications
     Verification (WebID)
                |
                v
     [Customer completes video identification]
                |
                v
  3. Create Customer                POST /roles/customers
                |
                v
  4. Sign Terms & Conditions        POST /v2/documents/sign
     + Data Privacy Policy
                |
                v
  5. Start Onboarding               POST /roles/onboardings
                |
                v
     [Async background checks & validation]
                |
                v
     APPROVED → Customer fully activated
```

---

## Step 0: Webhook 配置

入驻流程开始前须注册以下 5 种 Webhook 事件类型：

| 事件类型 | 说明 |
|----------|------|
| `NATURAL_PERSON_NOTIFICATION` | 自然人状态变更 |
| `CUSTOMER_NOTIFICATION` | 客户状态变更 |
| `IDENTIFICATION_VERIFICATION_NOTIFICATION` | 身份验证状态变更 |
| `DOCUMENT_NOTIFICATION` | 文档状态变更 |
| `ONBOARDING_NOTIFICATION` | 入驻状态变更 |

> [!NOTE] Webhook 为强烈推荐。若未配置，Partner 须主动轮询对应 GET 端点获取状态更新。**详细的错误原因仅通过 Webhook 通知返回**，轮询响应中不包含。

---

## Step 1: 创建自然人

### 端点

```
POST /entities/natural-persons
```

### 业务规则

| 规则 | 说明 |
|------|------|
| 德国税务 | `taxDetails.taxResidency = DE` 时，`taxId` **不需要**（Tradevest 从本地税务机关获取） |
| 其他国家税务 | 非 DE 税务居住地时，`taxId` **必须提供** |
| 加拿大地址 | `mainAddress.country = CA` 时，`mainAddress.state` **必填** |

### 请求示例

```json
{
  "gender": "MALE",
  "title": "PROFESSOR",
  "firstName": "John",
  "lastName": "Doe",
  "birthDay": "2005-11-19",
  "birthPlace": "Berlin",
  "birthCountry": "DE",
  "nationalities": [
    { "countryCode": "DE" }
  ],
  "taxDetails": [
    { "taxResidency": "DE" }
  ],
  "isUsNationality": false,
  "profession": "text",
  "professionGroup": "EMPLOYED",
  "mainAddress": {
    "street": "Doe Street",
    "streetNumber": "24",
    "city": "Berlin",
    "zip": "12345",
    "country": "DE"
  },
  "contact": {
    "phone": "123456789",
    "email": "tvdtester90+209461494@gmail.com"
  },
  "externalId": "0e03cbb8-91ea-484f-ae3c-bc665e05b5b6"
}
```

### 响应

```json
200 OK
{
  "naturalPersonId": "2ef8762d-5128-44f3-86ef-10fc2312b7e9"
}
```

### 查询自然人

```
GET /entities/natural-persons/{naturalPersonId}
```

关键返回字段：

| 字段 | 说明 |
|------|------|
| `naturalPersonId` | UUID 标识符 |
| `globalId` | 12 位全局标识符（系统根据匹配数据自动分配） |
| `naturalPersonStatus` | 创建后为 `CREATED` |

---

## Step 2: 身份识别验证（WebID）

### 创建验证请求

```
POST /entities/identification-verifications
Header: Requestor-ID: {naturalPersonId}
```

| 字段 | 值 | 说明 |
|------|----|------|
| `externalVerifier` | `WEB_ID` | 当前唯一支持的外部验证服务 |
| `identificationType` | `VIDEO_IDENT` | 当前唯一支持的识别类型 |
| `redirectUrl` | URL | 客户完成识别后的跳转地址（成功或失败均跳转） |

**前提条件：** 自然人须年满 **18 岁**。

```json
{
  "naturalPersonId": "fb2fab01-48cc-408b-8fec-56522cd75c91",
  "externalVerifier": "WEB_ID",
  "identificationType": "VIDEO_IDENT",
  "redirectUrl": "https://www.tradevest.ai"
}
```

```json
200 OK
{
  "identificationVerificationId": "b4414702-1a12-4554-b2b9-3e53a7e95f9d"
}
```

### 查询验证详情

```
GET /entities/identification-verifications/{identificationVerificationId}
```

```json
200 OK
{
  "naturalPersonId": "9b29cd58-3105-4e18-ab4d-f4970ea29d4a",
  "identificationVerificationId": "4d18fa87-b233-4990-88cc-5a288828b8e8",
  "externalVerifier": "WEB_ID",
  "identificationType": "VIDEO_IDENT",
  "identificationUrl": "https://test.webid-solutions.de/service/index/cn/000792/aid/145588703",
  "createdOn": "2025-11-19T14:06:35.008247Z",
  "modifiedOn": "2025-11-19T14:06:35.008247Z",
  "status": "PENDING"
}
```

**关键字段：** `identificationUrl` — 将客户重定向到此 URL 完成视频身份识别。

> [!NOTE] Sandbox 环境支持手动点击完成 WebID 流程。

### Webhook 通知 — 验证状态变更

```json
POST http://client-api-address/identification-verification-notification

{
  "identificationVerificationId": "4d18fa87-b233-4990-88cc-5a288828b8e8",
  "naturalPersonId": "9b29cd58-3105-4e18-ab4d-f4970ea29d4a",
  "modifiedOn": "2025-11-19T14:06:35.008247Z",
  "status": "APPROVED",
  "notificationType": "UPDATED"
}
```

### 验证通过后查询

验证通过后 GET 响应新增两个字段：

| 新增字段 | 说明 |
|----------|------|
| `documentId` | 系统自动生成的身份证明文档 ID |
| `naturalPersonIdentificationId` | 身份识别记录 ID |

```json
200 OK
{
  "naturalPersonId": "9b29cd58-3105-4e18-ab4d-f4970ea29d4a",
  "identificationVerificationId": "4d18fa87-b233-4990-88cc-5a288828b8e8",
  "externalVerifier": "WEB_ID",
  "identificationType": "VIDEO_IDENT",
  "identificationUrl": "https://test.webid-solutions.de/...",
  "documentId": "5395550f-9856-4d9f-8607-6cc2434fc4fe",
  "naturalPersonIdentificationId": "1ac56561-6aa4-4ad9-84cb-d25f238ce6bf",
  "createdOn": "2025-11-19T14:06:35.008247Z",
  "modifiedOn": "2025-11-19T14:06:35.008247Z",
  "status": "APPROVED"
}
```

---

## Step 3: 创建客户角色

> **创建客户角色不会启动入驻流程。** 此步骤仅为自然人准备入驻所需的客户数据。

### 端点

```
POST /roles/customers
```

### 请求

须提供参考银行账户（IBAN），用于验证入账和触发出账。

```json
{
  "entityId": "9b29cd58-3105-4e18-ab4d-f4970ea29d4a",
  "entityType": "NATURAL_PERSON",
  "refAccounts": [
    {
      "bankName": "Test Bank",
      "iban": "DE75512108001245126199",
      "bic": "SOGEDEFFXXX",
      "type": "PRIMARY",
      "ownerName": "Test TesterqRwXkkwAs"
    }
  ],
  "labels": []
}
```

### 响应

```json
200 OK
{
  "customerId": "5b5c7296-4782-4669-83c9-125b98b9ed4a"
}
```

### 查询客户

```
GET /roles/customers/{customerId}
```

初始状态 `customerStatus: CREATED`，表示可以进入入驻流程。

---

## Step 4: 签署条款与隐私政策

Tradevest 要求客户签署平台的 **Terms & Conditions** 和 **Data Privacy Policy**。这些文档由 Tradevest 团队上传，可通过 `GET /partner-documents` 获取。

### 签署 Terms & Conditions

```
POST /v2/documents/sign
```

```json
{
  "partnerDocumentId": "{{documentId_TERMS_AND_CONDITIONS}}",
  "naturalPersonId": "{{naturalPersonId}}",
  "customerId": "{{customerId}}"
}
```

```
202 Accepted
```

> 须同时提供 `naturalPersonId`（签署人）和 `customerId`（条款适用的客户）。

### 签署 Data Privacy Policy

```
POST /v2/documents/sign
```

```json
{
  "partnerDocumentId": "{{documentId_DATA_PRIVACY_POLICY}}",
  "naturalPersonId": "{{naturalPersonId}}"
}
```

```
202 Accepted
```

> Data Privacy Policy 签署**不需要 `customerId`**，因为隐私政策记录在自然人层面，而非客户层面。

---

## Step 5: 启动入驻

### 端点

```
POST /roles/onboardings
```

```json
{
  "roleType": "CUSTOMER",
  "roleId": "5b5c7296-4782-4669-83c9-125b98b9ed4a"
}
```

```json
200 OK
{
  "onboardingId": "9473f691-de72-486a-ae6b-8e46c3771c19"
}
```

### 入驻状态监控

通过 Webhook（`ONBOARDING_NOTIFICATION`）或轮询 `GET /roles/onboardings/{onboardingId}` 监控。

| 状态 | 含义 | 是否需要操作 |
|------|------|-------------|
| `CREATED` | 入驻已发起 | 无需操作 |
| `PENDING` | 背景检查进行中 | 无需操作 |
| `APPROVED` | 客户已激活，可创建产品 | 入驻完成 |
| `REJECTED` | 可修复的校验问题 | 查看 Webhook 错误原因 |
| `INVALID` | 阻塞性数据问题 | 查看 Webhook 错误原因 |

### 查询入驻状态

```
GET /roles/onboardings/{onboardingId}
```

```json
200 OK
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

## 入驻完成后

入驻成功（`status: APPROVED`）后，客户已完全激活，可请求创建：
- 余额账户（Balance Account）
- 托管账户（Depository Account）
- 钱包（Wallet）

---

## 相关文档

- [LE Onboarding — Partner Ident](./le-onboarding-partner-ident.md) — 法人实体入驻流程（Partner 外部身份识别）
- [Onboarding](../documentation/onboarding.md) — 三种入驻类型、异步校验、KYC、序列图
- [Natural Person Management](../documentation/natural-person-management.md) — 自然人准备流程、身份验证、文档签署
- [Natural Person Create Process](../documentation/natural-person-create-process.md) — 自然人创建校验规则
- [Role Management](../documentation/role-management.md) — 角色类型与分配流程
- [Document Management](../documentation/document-management.md) — 文档类型、上传、状态生命周期
- [Partner Webhooks](../documentation/partner-webhooks.md) — Webhook 通信模型

### API Reference

- [Natural Persons](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/natural-persons) — 自然人 CRUD 接口
- [Identification](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/identification) — 身份识别接口
- [Customers](https://docs.tradevest.ai/api-reference/roles/customers) — 客户角色接口
- [Documents v2](https://docs.tradevest.ai/api-reference/documents/new-documents) — 文档管理接口
- [Roles Onboardings](https://docs.tradevest.ai/api-reference/roles/onboardings) — 角色入驻接口
- [Webhooks](https://docs.tradevest.ai/api-reference/webhooks) — Webhook 管理接口
- [Partner Documents](https://docs.tradevest.ai/api-reference/partner-documents) — 合作方文档接口（获取 T&C / Privacy Policy）
