# Document Management

> Source: https://docs.tradevest.ai/documentation/entity_role_management_after_refactor/document_management

## 概述

文档管理系统与实体-角色分离架构对齐，根据关联的实体或角色类型对文档进行精确分类。上传或管理文档时，须指定正确的 `resourceType` 以匹配文档所属的实体或角色。

## API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v2/documents/` | `POST` | 上传文档，需指定 `resourceType` 和 `resourceId` |
| `/v2/documents/sign` | `POST` | 自然人签署文档 |
| `/v2/documents/{documentId}` | `GET` | 获取文档元数据 |
| `/v2/documents/{documentId}/file` | `GET` | 下载文档文件内容 |

## 文档类型总览

### 公司文档（Company Documents）

| 文档类型 | 适用 resourceType | 说明 |
|----------|-------------------|------|
| `CURRENT_REGISTRY_EXTRACT` | `LEGAL_ENTITY` | 当前登记摘要 |
| `CHRONOLOGICAL_REGISTRY_EXTRACT` | `LEGAL_ENTITY` | 历史登记摘要 |
| `SHAREHOLDER_LIST` | `LEGAL_ENTITY` | 股东名册 |
| `PARTNERSHIP_AGREEMENT` | `LEGAL_ENTITY` | 合伙协议 |
| `TRANSPARENCY_REGISTER_EXTRACT` | `LEGAL_ENTITY` | 透明度登记摘要 |
| `STATUTE` | `LEGAL_ENTITY` | 公司章程 |
| `BUSINESS_REGISTRATION` | `NATURAL_PERSON`, `LEGAL_ENTITY` | 商业注册文件 |

### KYC 文档

| 文档类型 | 适用 resourceType | 说明 |
|----------|-------------------|------|
| `IDENTIFICATION_CERTIFICATE` | `NATURAL_PERSON`, `PARTNER_USER` | 身份证明（仅成人） |
| `PROOF_OF_RESIDENCE` | `NATURAL_PERSON`, `PARTNER_USER` | 居住证明 |
| `KYC` | `NATURAL_PERSON`, `PARTNER_USER` | KYC 文档 |
| `BIRTH_CERTIFICATE` | `NATURAL_PERSON`（未成年人） | 出生证明（仅未成年人） |
| `DEATH_CERTIFICATE` | `NATURAL_PERSON` | 死亡证明 |

### 税务文档（Tax Documents）

| 文档类型 | 适用 resourceType | 说明 |
|----------|-------------------|------|
| `TIN_NA_CONFIRMATION` | `NATURAL_PERSON` | TIN 不可用确认 |
| `W_8BEN_E` | `LEGAL_ENTITY` | W-8BEN-E 表格（FATCA 相关） |

### 财务状况文档（Financial Situation）

| 文档类型 | 适用 resourceType | 说明 |
|----------|-------------------|------|
| `SOURCE_OF_INCOME` | `CUSTOMER` | 收入来源证明 |
| `SOURCE_OF_FUNDS` | `CUSTOMER` | 资金来源证明 |

### 代理人文档（Proxy Documents）

| 文档类型 | 适用 resourceType | 适用代理类型 |
|----------|-------------------|-------------|
| `PROOF_OF_SINGLE_CUSTODY` | `PROXY` | Guardian |
| `PROOF_OF_CUSTODY` | `PROXY` | Guardian |
| `INSOLVENCY_ORDER` | `PROXY` | Liquidator |
| `INHERITANCE_LEGITIMATION` | `PROXY` | General Power of Attorney（in case of death） |

### 通用文档（General Documents）

| 文档类型 | 适用 resourceType |
|----------|-------------------|
| `OTHER` | `NATURAL_PERSON`, `JOINT_PERSON`, `LEGAL_ENTITY`, `PARTNER_USER`, `CUSTOMER`, `PROXY` |

## resourceType 汇总

| resourceType | 说明 |
|--------------|------|
| `NATURAL_PERSON` | 自然人实体 |
| `LEGAL_ENTITY` | 法人实体 |
| `JOINT_PERSON` | 联名人实体 |
| `CUSTOMER` | 客户角色 |
| `PROXY` | 代理人角色 |
| `PARTNER_USER` | 合作方用户 |

## 文档状态与生命周期

文档在入驻流程中经历以下状态变更：

```
  CREATED -------> PENDING -------> APPROVED
  (uploaded)       (processing      (validated)
                    during
                    onboarding)
                       |
                       +----------> REJECTED
                                    (failed
                                     validation)
```

| 状态 | 含义 |
|------|------|
| `CREATED` | 文档已上传 |
| `PENDING` | 入驻过程中处理中 |
| `APPROVED` | 已审批通过 |
| `REJECTED` | 验证失败 |

Partner 通过 **Document Notification** Webhook 接收所有状态变更通知。

## API Reference

- [Documents v2](https://docs.tradevest.ai/api-reference/documents/new-documents) — 文档管理接口 (v2)
- [Documents Schemas v2](https://docs.tradevest.ai/api-reference/documents/new-schemas) — 文档数据结构定义 (v2)
