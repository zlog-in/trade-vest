# Entity And Role Management

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor

## 概述

平台采用**实体-角色分离（Entity-Role Separation）** 架构：

- **实体（Entity）** — 核心身份，如自然人、法人、联名人、受益所有人
- **角色（Role）** — 实体承担的功能，如客户（Customer）、代理人（Proxy）

这种分离使得：
- 单个实体可拥有**多个角色**
- 实体和角色各自独立管理状态
- 入驻（Onboarding）和离场（Offboarding）流程彼此独立

## 实体类型

### Natural Person（自然人）

代表系统中的个人。可承担的角色/关系：
- 直接作为客户（Customer）
- 加入联名人（Joint Person）
- 作为法人的受益所有人（Beneficial Owner）
- 作为其他实体的代理人（Proxy）
- 作为法人的法定代表人（Legal Representative）

### Legal Entity（法人实体）

代表公司、组织或其他非自然人实体。可承担的角色/关系：
- 作为客户（Customer）
- 拥有受益所有人（Beneficial Owner）
- 拥有法定代表人（Legal Representative）

### Joint Person（联名人）

由**两个自然人**组成、作为单一实体行事（典型场景：联名账户、共同持有）。可承担的角色：
- 作为客户（Customer）

### Beneficial Owner（受益所有人）

最终拥有或控制法人实体的实体。特征：
- **始终关联到某个法人实体**
- 类型枚举：`REAL_UBO_25`、`FICTIVE_UBO`
- 可以是自然人

### Legal Representative（法定代表人）

被授权代表法人实体行事的实体。特征：
- **始终关联到某个法人实体**
- 可以是自然人

## 实体关系图

```
+------------------+        +------------------+
|  Natural Person  |        |   Legal Entity   |
| (individual)     |        |  (company/org)   |
+------------------+        +------------------+
   |   |   |   |               ^    ^    ^
   |   |   |   |               |    |    |
   |   |   |   +-- Proxy ----->|    |    |   (can proxy for any
   |   |   |       for any     |    |    |    customer entity)
   |   |   |       customer    |    |    |
   |   |   |                   |    |    |
   |   |   +-- Legal      ----+    |    |
   |   |       Representative      |    |
   |   |                           |    |
   |   +------ Beneficial   ------+     |
   |           Owner                    |
   |                                    |
   +--- forms ---> +---------------+    |
   |               | Joint Person  |    |
   +--- forms ---> | (2 NPs = 1 JP)|    |
                   +---------------+    |
                                        |
                                        |
     === Roles (Customer / Proxy) ======+====
                                        |
   Any of NP / LE / JP can be ---> [Customer]
   NP can be ----------------------> [Proxy] --> for any Customer
```

### 关系总结表

| 关系 | 说明 |
|------|------|
| Natural Person → Joint Person | 两个自然人组成一个联名人 |
| Natural Person → Proxy → Customer | 自然人可作为任意客户（NP/JP/LE）的代理人 |
| Legal Representative → Legal Entity | 法定代表人关联到法人实体 |
| Beneficial Owner → Legal Entity | 受益所有人关联到法人实体 |

## 实体-角色分离架构

```
+------------------------------------------+
|              Entity Layer                |
|  (core identity & status)               |
|                                          |
|  Natural Person / Legal Entity /         |
|  Joint Person / Beneficial Owner /       |
|  Legal Representative                    |
+--------------------+---------------------+
                     |
                     | one entity, multiple roles
                     v
+------------------------------------------+
|               Role Layer                 |
|  (function & status)                     |
|                                          |
|  Customer / Proxy                        |
|                                          |
|  - Independent onboarding/offboarding    |
|  - Independent status management         |
+------------------------------------------+
```

## 统一实体视图与搜索

### 端点

```
GET /entities
```

### 公共返回字段

| 字段 | 说明 |
|------|------|
| `globalId` | 12 位字符标识符（如 `ABC123DEF456`） |
| `entityId` | UUID 格式标识符（如 `550e8400-e29b-41d4-a716-446655440000`） |
| `entityName` | 实体名称 |
| `entityType` | 实体类型 |
| `entityStatus` | 实体状态 |
| role info | 附带角色信息 |

### 搜索参数

| 参数 | 说明 |
|------|------|
| `searchText` | 全文搜索（匹配 entityName、entityId、globalId） |
| `entityType` | 按实体类型过滤（如 `NATURAL_PERSON`） |
| `entityStatus` | 按状态过滤（如 `ACTIVE`） |
| `role` | 按角色过滤（如 `CUSTOMER`） |
| `globalId` | 按 Global ID 精确过滤 |
| `entityId` | 按 Entity ID (UUID) 精确过滤 |
| `limit` | 分页大小 |
| `cursor` | 游标分页 token |

### 查询示例

```
GET /entities?searchText=Smith&entityType=NATURAL_PERSON&entityStatus=ACTIVE
GET /entities?globalId=ABC123DEF456
GET /entities?entityId=550e8400-e29b-41d4-a716-446655440000&role=CUSTOMER
GET /entities?limit=50&cursor=next_page_token
```

## 相关文档

- [Natural Person Management](./natural-person-management.md) — 自然人管理与准备
- [Legal Entity Management](./legal-entity-management.md) — 法人实体管理
- [Role Management](./role-management.md) — 角色管理与分配
- [Onboarding](./onboarding.md) — 入驻流程、异步校验、KYC
- [Offboarding](./offboarding.md) — 离场流程、级联效应、Ordinary/Immediate
- [Document Management](./document-management.md) — 文档管理

### API Reference

- [Entities](https://docs.tradevest.ai/api-reference/entities) — 实体管理接口总览
- [Entities Schemas](https://docs.tradevest.ai/api-reference/entities/schemas) — 实体数据结构定义
- [Roles](https://docs.tradevest.ai/api-reference/roles) — 角色管理接口总览
- [Roles Schemas](https://docs.tradevest.ai/api-reference/roles/schemas) — 角色数据结构定义
