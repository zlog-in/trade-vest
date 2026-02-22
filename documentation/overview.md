# API Overview

> Source: https://docs.tradevest.ai/documentation/readme-1

## API Guide

API 遵循 **OpenAPI 3.0** 规范，采用 RESTful 风格，按业务域拆分为多个独立的 OpenAPI 文件。

> [!NOTE] 原文提到 "refer to the corresponding OpenAPI file"，但文档中**未提供任何 OpenAPI 文件的下载链接或访问地址**。

Schema 命名约定如下：

| 操作 | Schema 名称 | 说明 |
|------|-------------|------|
| **Create** | `ObjectData` | 用于 POST 请求，schema 中指定必填字段 |
| | `ObjectResult` | 封装 POST 操作结果，通常仅包含新创建对象的 ID（封装为对象以提高复用性） |
| **Read** | `Object` | 完整实体 schema，包含 `ObjectData` 全部字段 + 元数据（如实体标识符） |
| | `Objects` | `Object` 数组 + 分页 |
| | `Array of Object` | 不需要分页时使用的对象数组 |
| **Update** | `ObjectUpdate` | 更新操作 schema，对象标识符不在 schema 内，而是作为 path parameter 传递 |

> [!NOTE] 原文仅涵盖 Create / Read / Update 三种操作的命名约定，**未提及 Delete 操作**。

## Concepts

### External IDs

许多对象包含 `externalId` 属性，用途：
- 允许合作方提供自己的标识符
- 跨自治系统进行**重复检测**，防止网络故障或操作重试时产生重复实体

### Allowed Data

此环境可用于实时测试，但**禁止使用真实客户数据或任何敏感信息**。

### RECEIVED / INVALID / REJECTED 状态

此逻辑**仅适用于创建实体（Create）**，不适用于更新操作。

#### 创建实体的状态流转

```
                          Partner POST Request
                                  |
                                  v
                    +----------------------------+
                    |  Synchronous Validation     |
                    |  - Static rules (OpenAPI)   |
                    |  - Duplicate detection       |
                    |    (via externalId)          |
                    +----------------------------+
                          |              |
                       FAILED           PASS
                          |              |
                          v              v
                  +------------+   +-----------+
                  | 400 / 409  |   | RECEIVED  |--- entity created, awaiting
                  | (rejected) |   +-----------+    async processing
                  +------------+         |
                                         v
                              +---------------------+
                              | Async Full Validation|
                              | - Customer ID valid? |
                              | - Customer active?   |
                              | - Sufficient balance?|
                              +---------------------+
                               |         |         |
                            FAILED      PASS    FAILED
                          (validation) (stage1) (later stages)
                               |         |         |
                               v         v         v
                         +---------+ +--------+ +----------+
                         | INVALID | | CREATED| | REJECTED |
                         +---------+ | (etc.) | +----------+
                               |     +--------+       |
                               |         |             |
                               v         v             v
                          +---------+ Webhook      Webhook
                          | Cleanup | notifies     notifies
                          +---------+ partner      partner
                               |
                               v
                      Entity removed,
                      externalId retained
                      (cannot be reused)
```

**各状态含义：**

| 状态 | 含义 |
|------|------|
| **RECEIVED** | 平台已接收请求的初始状态。在 webhook 通知处理结果前持续保持。用于追踪合作方请求的接收情况 |
| **INVALID** | 异步完整校验失败。对应同步处理中 4xx 客户端错误的场景 |
| **REJECTED** | 异步处理后续阶段失败。部分实体的异步处理分为多个阶段，通过第一阶段后仍可能在后续阶段被拒绝 |
| **CREATED** 等 | 校验通过，实体创建成功，分配 UUID 格式的 ID 和对应业务状态 |

> [!NOTE] 原文对 REJECTED 的描述仅一句："部分实体的异步处理分为多个阶段，状态可变为 REJECTED"，**缺少明确的触发条件和具体场景说明**。

#### 更新操作的处理

更新操作不走上述状态流转，规则如下：
- 请求正确且通过完整校验 → 修改实体的状态和数据
- 校验失败 → 实体**不会被修改**，通过 webhook 通知合作方拒绝原因（该 webhook **不包含 `status` 和 `modifiedOn` 字段**）

#### 清理机制

- RECEIVED 和 INVALID 状态的实体最终会被平台**自动移除**
- 但这些实体使用过的 `externalId` **不会被删除，也不可被合作方复用**

## Domains

原文明确列出以下 6 个域，每个域对应一个独立的 OpenAPI 文件：

| 域 | 说明 |
|----|------|
| **Authorization** | 安全 API 访问，Access Token 生成 |
| **Customers** | 客户入驻、管理、文档处理 |
| **Webhooks** | HTTP 回调，实时事件通知 |
| **Trading** | 订单下单与市场数据获取 |
| **Products** | 产品定义与客户产品管理 |
| **Transactions** | 金融操作历史记录 |

> [!NOTE] 原文 Overview 页仅列出上述 6 个域，但 API Reference 侧边栏中实际还包含 **Transfers**（转账）、**Documents**（文档）、**Partner Documents**（合作方文档）等模块，与此处列表不完全一致。

## API Reference

- [Authorization](https://docs.tradevest.ai/api-reference/authorization) — 认证授权
- [User Management](https://docs.tradevest.ai/api-reference/user-management/users) — 用户管理
- [Entities](https://docs.tradevest.ai/api-reference/entities) — 实体管理
- [Roles](https://docs.tradevest.ai/api-reference/roles) — 角色管理
- [Documents v2](https://docs.tradevest.ai/api-reference/documents/new-documents) — 文档管理
- [Products](https://docs.tradevest.ai/api-reference/products/product-definitions-and-fees) — 产品定义与费用
- [Customer Products](https://docs.tradevest.ai/api-reference/products/customer-products) — 客户产品
- [Taxes](https://docs.tradevest.ai/api-reference/taxes/information) — 税务信息
- [Asset Management](https://docs.tradevest.ai/api-reference/asset-management) — 资产管理 / 交易
- [Transaction History](https://docs.tradevest.ai/api-reference/transaction-history) — 交易历史
- [Transfers](https://docs.tradevest.ai/api-reference/transfers/transfers) — 转账
- [Partner Documents](https://docs.tradevest.ai/api-reference/partner-documents) — 合作方文档
- [Webhooks](https://docs.tradevest.ai/api-reference/webhooks) — Webhook 管理
- [Changelogs v2](https://docs.tradevest.ai/api-reference/changelogs/new-changelogs) — API 变更日志
