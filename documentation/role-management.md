# Role Management

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/roles_management

## 概述

平台基于**实体-角色分离架构**实现角色管理，角色分为两类：**客户（Customer）** 和 **代理人（Proxy）**。实体准备完成后即可分配角色。

## 角色类型

### Customer（客户）

- 可分配给：**自然人（NP）、法人实体（LE）、联名人（JP）**
- 是主要的银行业务关系
- 须经过完整的入驻（Onboarding）和验证流程

### Proxy（代理人）

- 仅可分配给：**自然人**
- 授权代表另一实体行事
- 6 种代理类型：

| 代理类型 | 说明 |
|----------|------|
| `GUARDIAN` | 监护人（未成年人） |
| `SIGNATORY` | 签署人（法人实体） |
| `JOINT_ACCOUNT_HOLDER` | 联名账户持有人 |
| `GENERAL_POWER_OF_ATTORNEY` | 一般授权代理 |
| `LIQUIDATOR` | 清算人 |
| `INFORMATION_PROXY` | 信息代理 |

## 角色分配流程

### Customer 角色分配

```
+---------------------------------------+
| 1. Assign Customer Role               |
| POST /roles/customers                 |
| - entityId                            |
| - entityType                          |
+---------------------------------------+
                 |
                 v
+---------------------------------------+
| 2. Upload Customer Documents          |
| POST /v2/documents                    |
| resourceType: CUSTOMER                |
+---------------------------------------+
```

### Proxy 角色分配

```
+---------------------------------------+
| Prerequisites:                        |
| - NP (proxy) exists                   |
| - Entity (to be represented) exists   |
+---------------------------------------+
                 |
                 v
+---------------------------------------+
| 1. Assign Proxy Role                  |
| POST /roles/proxies                   |
| - naturalPersonId                     |
| - entityId                            |
| - entityType                          |
+---------------------------------------+
                 |
                 v
+---------------------------------------+
| 2. Upload Proxy-Specific Documents    |
| POST /v2/documents                    |
| resourceType: PROXY                   |
+---------------------------------------+
```

### Proxy 文档要求（按代理类型）

| 代理类型 | 所需文档类型 |
|----------|-------------|
| `GUARDIAN` | `PROOF_OF_SINGLE_CUSTODY`, `PROOF_OF_CUSTODY` |
| `LIQUIDATOR` | `INSOLVENCY_ORDER` |
| `GENERAL_POWER_OF_ATTORNEY`（death validity） | `INHERITANCE_LEGITIMATION` |
| 其他代理类型 | `GENERAL`, `CONTRACTS` 等通用文档 |

## 角色分配后

角色分配完成后，需执行**入驻（Onboarding）** 流程。详见 [Onboarding](./onboarding.md)。

## 涉及的 API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/roles/customers` | `POST` | 分配客户角色 |
| `/roles/proxies` | `POST` | 分配代理人角色 |
| `/v2/documents` | `POST` | 上传角色相关文档 |

## 相关文档

- [Entity And Role Management](./entity-and-role-management.md) — 实体类型与关系总览
- [Natural Person Management](./natural-person-management.md) — 自然人/联名人实体准备
- [Legal Entity Management](./legal-entity-management.md) — 法人实体准备
- [Proxy Create Process](./proxy-create-process.md) — 代理人创建校验与规则
- [Proxy Update Process](./proxy-update-process.md) — 代理人更新流程
- [Onboarding](./onboarding.md) — 入驻流程、验证、KYC
