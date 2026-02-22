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

---

## API Reference

### 1. Authorization (授权)
- [Authorization](./api-reference/authorization.md) — 认证与授权机制、Schemas

### 2. User Management (用户管理)
- [User Management](./api-reference/user-management.md) — 用户 CRUD 操作、Schemas

### 3. Entities (实体管理)
- [Entities Overview](./api-reference/entities.md) — 实体管理总览
  - Natural Persons — 自然人管理、身份识别、交易档案、适当性测试
  - Legal Entities — 法人实体
  - Beneficial Owners — 受益所有人
  - Legal Representatives — 法定代表人
  - Joint Persons — 联名人
  - Onboardings — 实体入驻
  - General — 通用接口
  - NACE Sectors — NACE 行业分类查询
  - Offboarding — 实体离场
  - Schemas

### 4. Roles (角色管理)
- [Roles](./api-reference/roles.md) — 角色管理总览
  - Customers — 客户角色
  - Customer Labels — 客户标签
  - Proxies — 代理人
  - Onboardings / Offboarding — 角色入驻与离场
  - Schemas

### 5. Documents (文档管理)
- [Documents](./api-reference/documents.md) — 文档上传与管理 (v2)
  - Schemas

### 6. Products (产品)
- [Products](./api-reference/products.md) — 产品定义、费用、客户产品
  - Product Definitions & Fees
  - Customer Products
  - Schemas

### 7. Taxes (税务)
- [Taxes](./api-reference/taxes.md) — 税务信息与免税
  - Information / Exemption
  - Schemas

### 8. Asset Management (资产管理/交易)
- [Asset Management](./api-reference/asset-management.md) — 资产交易总览
  - Tokenized Assets (WAWEX) — 代币化资产、订单
  - Traditional Assets (Trading) — 传统资产交易
  - Digital Assets — 数字资产、订单创建
  - Schemas

### 9. Transaction History (交易历史)
- [Transaction History](./api-reference/transaction-history.md) — 交易记录查询、Schemas

### 10. Transfers (转账)
- [Transfers](./api-reference/transfers.md) — 转账操作
  - Transfers — 转账接口
  - General — IBAN 验证等通用接口
  - Test Transfers — 测试入账
  - Schemas

### 11. Partner Documents (合作方文档)
- [Partner Documents](./api-reference/partner-documents.md) — 合作方文档管理、Schemas

### 12. Webhooks
- [Webhooks](./api-reference/webhooks.md) — Webhook 事件通知、Schemas

### 13. Changelogs (变更日志)
- [Changelogs](./api-reference/changelogs.md) — API 变更记录 (v2)
  - Schemas

---

*整理进度: Documentation 章节 0-3 已完成（17 个文件，~3400 行）。Entity & Role Management 子页中 Onboarding 等待整理。API Reference 章节（13 个模块）尚未开始。*
