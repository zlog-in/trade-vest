# Partner Webhooks

> Source: https://docs.tradevest.ai/documentation/partner-webhooks

## 概述

Partner 须准备接收平台发送的异步事件通知消息（Webhook）。平台要求 **Partner 为每种对象类型提供独立的 Webhook 端点**。

## 通信模型

平台与 Partner 之间的 Webhook 交互涉及三方：

```
  P (Client)                   T (API)                    PW (Webhook)
      |                           |                            |
      | Create/Update request     |                            |
      |-------------------------->|                            |
      | OK (with object id)       |                            |
      |<--------------------------|                            |
      |                           |                            |
      |                           |  [ Async Processing ]      |
      |                           |                            |
      |                           | Notification (EVENT_TYPE 1)|
      |                           |--------------------------->|
      |                           | Notification (EVENT_TYPE 2)|
      |                           |--------------------------->|
      |                           |         ...                |
      |                           | Notification (EVENT_TYPE n)|
      |                           |--------------------------->|
      |                           |                            |
      | Get object request        |                            |
      |-------------------------->|                            |
      | OK (with object data)     |                            |
      |<--------------------------|                            |
```

| 角色 | 说明 |
|------|------|
| **P** (Client) | Partner's API Client — 发起创建/更新/查询请求 |
| **T** (API) | Tradevest API — 处理请求，异步触发事件通知 |
| **PW** (Webhook) | Partner's Webhook Server — 接收异步事件通知 |

## 两种对象创建场景

### 1. Partner 发起的创建

Partner 主动调用 API 创建对象（如客户、客户产品等），创建时即获得对象 ID。后续通过 Webhook 接收状态变更通知。

**示例：** 创建客户 → 收到 `id` → 后续收到 Customer Notification（CREATED → PENDING → ACTIVE）

### 2. 平台自动创建

平台基于业务规则自动创建对象（如身份验证过程中自动生成的文档），Partner 收到的 Webhook 通知中包含**此前未知的新 ID**。此时 Partner 应调用对应的 GET 接口获取对象详情。

**示例：** 外部身份验证完成 → 系统自动创建 `IDENTIFICATION_CERTIFICATE` 文档 → Partner 收到 Document Notification（包含新 documentId）→ Partner 调用 `GET /v2/documents/{documentId}` 获取详情

## 关键要求

- **独立端点：** 每种对象类型须有单独的 Webhook 接收端点
- **异步性：** API 调用返回同步 OK 响应后，状态变更通过 Webhook 异步推送
- **主动查询：** 对于平台自动创建的对象，Partner 需通过 GET 接口获取完整数据

## 相关文档

- [Onboarding](./onboarding.md) — 入驻流程中涉及的所有 Webhook 通知类型
- [Offboarding](./offboarding.md) — 离场流程的 Webhook 通知
- [Document Management](./document-management.md) — 文档状态变更通知

### API Reference

- [Webhooks](https://docs.tradevest.ai/api-reference/webhooks) — Webhook 管理接口（注册 / 查询 / 删除）
- [Webhooks Schemas](https://docs.tradevest.ai/api-reference/webhooks/schemas) — Webhook 数据结构定义（含 24 种事件类型）
