# Legal Entity Management

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/legal_entity_management

## 概述

本文档描述法人实体的准备（Preparation）流程，包括通过外部数据供应商（Vendor/Provider）获取数据、创建法人实体、添加受益所有人和法定代表人、上传文档的完整工作流。

## 法人实体准备流程

共 6 步，其中前 2 步为可选步骤：

```
+---------------------------------------------------+
| Step 1 (Optional): Search                         |
| GET /entities/vendor/legal-entities                |
| Query by country + company name                   |
+---------------------------------------------------+
                       |
                       v
+---------------------------------------------------+
| Step 2 (Optional): Prepare                        |
| POST /entities/vendor/legal-entities               |
| Order data from external provider                 |
| Returns: searchId                                 |
+---------------------------------------------------+
                       |
                       v
            Poll status until COMPLETED:
            GET /entities/vendor/legal-entities/{searchId}
                       |
                       v
+---------------------------------------------------+
| Step 3: Create Legal Entity                       |
| POST /entities/legal-entities                      |
| (include searchId to link provider data)          |
+---------------------------------------------------+
                       |
                       v
+---------------------------------------------------+
| Step 4: Add Beneficial Owners                     |
| POST /entities/{legalEntityId}/beneficial-owners   |
| (can use provider data)                           |
+---------------------------------------------------+
                       |
                       v
+---------------------------------------------------+
| Step 5: Add Legal Representatives                 |
| POST /entities/{legalEntityId}/legal-representatives
| (can use provider data)                           |
+---------------------------------------------------+
                       |
                       v
+---------------------------------------------------+
| Step 6: Upload Documents                          |
| POST /v2/documents                                 |
| resourceType = LEGAL_ENTITY                       |
+---------------------------------------------------+
```

## 各步骤详解

### Step 1: Search（可选）

```
GET /entities/vendor/legal-entities
```

基于**国家**和**公司名称**从外部数据供应商检索已有法人实体数据。

### Step 2: Prepare（可选）

```
POST /entities/vendor/legal-entities
```

向外部供应商订购数据，返回唯一的 **`searchId`**。Partner 需通过轮询等待数据就绪后再进入下一步。

**查询准备状态：**

```
GET /entities/vendor/legal-entities/{searchId}
```

#### 准备状态流转

```
  PREPARING
      |
      v
  WAITING_FOR_DATA
      |
      v
  READY_DATA
      |
      v
  WAITING_FOR_DOCUMENTS
      |
      v
  READY_DOCUMENTS
      |
      v
  COMPLETED  -------> Proceed to Step 3
```

**异常状态：**

| 状态 | 含义 |
|------|------|
| `PREPARING` | 正在准备中 |
| `WAITING_FOR_DATA` | 等待供应商返回数据 |
| `READY_DATA` | 数据已就绪 |
| `WAITING_FOR_DOCUMENTS` | 等待供应商返回文档 |
| `READY_DOCUMENTS` | 文档已就绪 |
| `COMPLETED` | 全部就绪，可进入下一步 |
| `FAILED` | 供应商无法检索到请求的数据或文档 |
| `ERROR` | 数据收集过程中发生错误 |

**当供应商数据不可用时，Partner 有两种选择：**
- 使用供应商返回的数据（**推荐**）
- 不使用供应商，**手动录入数据**

### Step 3: Create Legal Entity

```
POST /entities/legal-entities
```

若使用了 Step 2 的供应商数据，创建请求中须包含 **`searchId`**。该 searchId 使系统能够：
- 自动从供应商获取文档（如存在）
- 辅助数据校验
- 将法人实体关联到供应商数据源

> 详细创建流程参见 [Legal Entity Create Process](./legal-entity-create-process.md)

### Step 4: Add Beneficial Owners

```
POST /entities/{legalEntityId}/beneficial-owners
```

受益所有人字段可使用供应商返回的数据填充，以确保与法人实体数据一致。

**重要：** 若法人实体下未添加任何 `REAL_UBO_25` 类型的受益所有人，系统将在 Onboarding 阶段**自动基于法定代表人数据创建 `FICTIVE_UBO`**。

> 详细创建/更新流程参见 [Beneficial Owner Create Process](./beneficial-owner-create-process.md) / [Update Process](./beneficial-owner-update-process.md)

### Step 5: Add Legal Representatives

```
POST /entities/{legalEntityId}/legal-representatives
```

法定代表人字段同样可使用供应商返回的数据填充。

### Step 6: Upload Documents

```
POST /v2/documents
```

- `resourceType` = `LEGAL_ENTITY`
- 供应商可能已自动提供部分文档，Partner 应**核实并补充上传**缺失的必需文档

#### 可能需要的文档类型

| 文档类型 | 说明 |
|----------|------|
| `CURRENT_REGISTRY_EXTRACT` | 当前登记摘要 |
| `SHAREHOLDER_LIST` | 股东名册 |
| `PARTNERSHIP_AGREEMENT` | 合伙协议 |
| `TRANSPARENCY_REGISTER_EXTRACT` | 透明度登记摘要 |
| `BUSINESS_REGISTRATION` | 商业注册文件 |
| `STATUTE` | 公司章程 |

## 相关文档

- [Legal Entity Create Process](./legal-entity-create-process.md) — 法人实体创建流程详情
- Legal Entity Update Process — 法人实体更新流程
- [Beneficial Owner Create Process](./beneficial-owner-create-process.md) — 受益所有人创建流程
- [Beneficial Owner Update Process](./beneficial-owner-update-process.md) — 受益所有人更新流程
