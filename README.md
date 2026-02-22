# Tradevest API Documentation

> Source: https://docs.tradevest.ai/

Tradevest 开发者文档，涵盖用户管理、实体管理、角色管理、文档、产品、税务、资产交易、转账、Webhooks 等 API 接口。

---

## Documentation

### 0. Overview (总览)
- [API Overview](./documentation/overview.md) — API 规范、命名约定、核心概念、域划分

### 1. Authentication (认证)
- [Authentication](./documentation/authentication.md) — OAuth 认证流程、Token 管理、Requestor-ID

### 2. User Management (用户管理)
- [User Management](./documentation/user-management.md) — 用户创建流程、停用、MFA 重置

### 3. Entity And Role Management (实体与角色管理)
- [Entity And Role Management](./documentation/entity-and-role-management.md) — 实体类型、实体-角色分离架构、关系、统一搜索
  - [Beneficial Owner Create Process](./documentation/beneficial-owner-create-process.md) — 受益所有人创建流程、校验体系、审核机制、FICTIVE_UBO
  - [Beneficial Owner Update Process](./documentation/beneficial-owner-update-process.md) — 受益所有人更新流程、校验体系、与创建流程差异
  - [Legal Entity Create Process](./documentation/legal-entity-create-process.md) — 法人实体创建流程、NACE 行业校验、全局/本地存在性逻辑
  - [Legal Entity Management](./documentation/legal-entity-management.md) — 法人实体准备流程、Vendor 数据、6 步工作流、文档上传
  - [Legal Entity Update Process](./documentation/legal-entity-update-process.md) — 法人实体更新流程、CREATED/ACTIVE 差异、审核机制、KYC、文档要求
  - [Legal Representative Create Process](./documentation/legal-representative-create-process.md) — 法定代表人创建流程、校验体系、审核机制、FICTIVE_UBO 联动
  - [Natural Person Management](./documentation/natural-person-management.md) — 自然人准备流程（成人/未成年/联名人）、身份验证、文档签署
  - [Natural Person Create Process](./documentation/natural-person-create-process.md) — 自然人创建流程、Partner 去重、NACE 行业、地域合规
  - [Natural Person Update Process](./documentation/natural-person-update-process.md) — 自然人更新流程、Death Day 特殊流程、KYC、文档要求、审核触发
  - [Proxy Create Process](./documentation/proxy-create-process.md) — 代理人创建流程、6 种代理类型、validityType 矩阵、类型专属规则
  - [Proxy Update Process](./documentation/proxy-update-process.md) — 代理人更新流程、scopeType/custodyType 规则、文档要求
  - [Role Management](./documentation/role-management.md) — 角色类型（Customer/Proxy）、角色分配流程、文档要求
  - [Document Management](./documentation/document-management.md) — 文档上传/签署/查询、35+ 文档类型、resourceType、状态生命周期
  - [Onboarding](./documentation/onboarding.md) — 三种入驻类型（Customer/Proxy/BO）、异步校验、KYC、5 种序列流程
  - [Offboarding](./documentation/offboarding.md) — 三种离场类型、Ordinary/Immediate 流程、级联效应、原因矩阵

### 4. Partner Webhooks (合作方 Webhook)
- [Partner Webhooks](./documentation/partner-webhooks.md) — Webhook 通信模型、三方交互、两种对象创建场景

### 5. Status Codes for Financial Operations (金融操作状态码)
- [Status Codes for Financial Operations](./documentation/status-codes-for-financial-operations.md) — Order/Transfer/Transaction 状态码、状态流转、映射关系

### 6. Prices (实时价格)
- [Prices](./documentation/prices.md) — WebSocket 价格订阅、消息类型、通信序列、行为规则

---

## API Reference

### 1. Authorization (授权)
- [Authorization](https://docs.tradevest.ai/api-reference/authorization) — 认证与授权机制

### 2. User Management (用户管理)
- [Users](https://docs.tradevest.ai/api-reference/user-management/users) — 用户 CRUD 操作
- [Schemas](https://docs.tradevest.ai/api-reference/user-management/schemas)

### 3. Entities (实体管理)
- [Natural Persons](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/natural-persons) — 自然人管理
  - [Identification](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/identification) — 身份识别
  - [Trading Profile](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/trading-profiles) — 交易档案
  - [Appropriateness Test](https://docs.tradevest.ai/api-reference/entities/natural-persons-directory/appropriateness-test) — 适当性测试
- [Joint Persons](https://docs.tradevest.ai/api-reference/entities/joint-persons) — 联名人
- [Legal Entities](https://docs.tradevest.ai/api-reference/entities/legal-entities) — 法人实体
- [Beneficial Owners](https://docs.tradevest.ai/api-reference/entities/beneficial-owners) — 受益所有人
- [Legal Representatives](https://docs.tradevest.ai/api-reference/entities/legal-representatives) — 法定代表人
- [Onboardings](https://docs.tradevest.ai/api-reference/entities/onboardings) — 实体入驻
- [General](https://docs.tradevest.ai/api-reference/entities/general) — 通用接口
- [Search NACE Sectors](https://docs.tradevest.ai/api-reference/entities/search-nace-sectors) — NACE 行业分类查询
- [Offboarding](https://docs.tradevest.ai/api-reference/entities/offboarding) — 实体离场
- [Schemas](https://docs.tradevest.ai/api-reference/entities/schemas)

### 4. Roles (角色管理)
- [Customers](https://docs.tradevest.ai/api-reference/roles/customers) — 客户角色
- [Customer Labels](https://docs.tradevest.ai/api-reference/roles/customer-labels) — 客户标签
- [Proxies](https://docs.tradevest.ai/api-reference/roles/proxies) — 代理人
- [Onboardings](https://docs.tradevest.ai/api-reference/roles/onboardings) — 角色入驻
- [Offboarding](https://docs.tradevest.ai/api-reference/roles/offboarding) — 角色离场
- [Schemas](https://docs.tradevest.ai/api-reference/roles/schemas)

### 5. Documents (文档管理)
- [Documents v2](https://docs.tradevest.ai/api-reference/documents/new-documents) — 文档上传与管理
- [Schemas v2](https://docs.tradevest.ai/api-reference/documents/new-schemas)

### 6. Products (产品)
- [Product Definitions & Fees](https://docs.tradevest.ai/api-reference/products/product-definitions-and-fees) — 产品定义与费用
- [Customer Products](https://docs.tradevest.ai/api-reference/products/customer-products) — 客户产品
- [Schemas](https://docs.tradevest.ai/api-reference/products/schemas)

### 7. Taxes (税务)
- [Tax Information](https://docs.tradevest.ai/api-reference/taxes/information) — 税务信息
- [Tax Exemption](https://docs.tradevest.ai/api-reference/taxes/exemption) — 免税
- [Schemas](https://docs.tradevest.ai/api-reference/taxes/schemas)

### 8. Asset Management (资产管理/交易)
- [WAWEX / Tokenized Assets](https://docs.tradevest.ai/api-reference/asset-management/wawex-tokenized-assets) — 代币化资产
  - [Orders](https://docs.tradevest.ai/api-reference/asset-management/wawex-tokenized-assets/orders) — 订单
  - [Tokenized Assets](https://docs.tradevest.ai/api-reference/asset-management/wawex-tokenized-assets/tokenized-assets) — 资产
- [Traditional Assets (Trading)](https://docs.tradevest.ai/api-reference/asset-management/trading) — 传统资产交易
  - [Assets](https://docs.tradevest.ai/api-reference/asset-management/trading/assets) — 资产
- [Digital Assets](https://docs.tradevest.ai/api-reference/asset-management/digital-assets) — 数字资产
  - [Orders](https://docs.tradevest.ai/api-reference/asset-management/digital-assets/create-order) — 订单创建
  - [Assets](https://docs.tradevest.ai/api-reference/asset-management/digital-assets/assets) — 资产
  - [Schemas](https://docs.tradevest.ai/api-reference/asset-management/digital-assets/schemas)

### 9. Transaction History (交易历史)
- [Transaction History](https://docs.tradevest.ai/api-reference/transaction-history) — 交易记录查询
- [Schemas](https://docs.tradevest.ai/api-reference/transaction-history/schemas)

### 10. Transfers (转账)
- [Transfers](https://docs.tradevest.ai/api-reference/transfers/transfers) — 转账接口
- [IBAN Validator](https://docs.tradevest.ai/api-reference/transfers/general) — IBAN 验证
- [Create Incoming Test Transfer](https://docs.tradevest.ai/api-reference/transfers/create-incoming-test-transfer) — 测试入账
- [Schemas](https://docs.tradevest.ai/api-reference/transfers/schemas)

### 11. Partner Documents (合作方文档)
- [Partner Documents](https://docs.tradevest.ai/api-reference/partner-documents) — 合作方文档管理
- [Schemas](https://docs.tradevest.ai/api-reference/partner-documents/schemas)

### 12. Webhooks
- [Webhooks](https://docs.tradevest.ai/api-reference/webhooks) — Webhook 事件通知
- [Schemas](https://docs.tradevest.ai/api-reference/webhooks/schemas)

### 13. Changelogs (变更日志)
- [Changelogs v2](https://docs.tradevest.ai/api-reference/changelogs/new-changelogs) — API 变更记录
- [Schemas](https://docs.tradevest.ai/api-reference/changelogs/schemas)

### WebSocket
- [Prices](https://docs.tradevest.ai/websockets/websocket-prices/prices) — WebSocket 价格订阅
- [Schemas](https://docs.tradevest.ai/websockets/websocket-prices/schemas)

---

## API Guidelines

### Onboarding (入驻指南)
- [NP Onboarding — Tradevest Ident](./api-guidelines/np-onboarding-tradevest-ident.md) — 自然人入驻完整流程（WebID 身份识别）、5 步骤、请求/响应示例
- [LE Onboarding — Partner Ident](./api-guidelines/le-onboarding-partner-ident.md) — 法人实体入驻完整流程（Partner 外部身份识别）、13 步骤、请求/响应示例

---

*整理进度: Documentation 章节 0-6（22 个文件）+ API Guidelines 2 个文件，共 ~5120 行。API Reference 章节（13 个模块）尚未开始。*
