# Authentication

> Source: https://docs.tradevest.ai/documentation/2_authentication

## 概述

API 使用 OAuth `client_credentials` 授权类型进行服务器到服务器（server-to-server）的认证。**不应用于终端用户认证。**

> [!WARNING] 原文写作 "**OAuth 3.0**"，但 `client_credentials` grant type 属于 **OAuth 2.0** 规范。OAuth 3.0 标准不存在，疑为原文笔误。

## 认证流程

```
  Partner                        Tradevest Auth Service
     |                                    |
     |  1. POST /oauth2/token             |
     |    Content-Type: x-www-form-urlencoded
     |    - grant_type=client_credentials  |
     |    - client_id=xxx                  |
     |    - client_secret=xxx             |
     |----------------------------------->|
     |                                    |
     |  2. JSON Response                  |
     |    - access_token                  |
     |    - token_type                    |
     |    - expires_in (seconds)          |
     |<-----------------------------------|
     |                                    |
     |  3. API Request                    |
     |    Authorization: Bearer <token>   |
     |    Requestor-ID: <natural_person_id>
     |----------------------------------->|
```

## Step 1: 获取凭证

由 Tradevest 提供以下凭证：

| 参数 | 说明 |
|------|------|
| `client_id` | 客户端（应用）在系统中的唯一标识符 |
| `client_secret` | 用于认证的密钥，**必须保密** |

## Step 2: 请求 Access Token

向 token 端点发送 HTTP POST 请求：

- **Content-Type**: `application/x-www-form-urlencoded`

| 参数 | 值 |
|------|-----|
| `grant_type` | `client_credentials` |
| `client_id` | Tradevest 提供的 client ID |
| `client_secret` | Tradevest 提供的 client secret |

请求成功后返回 JSON，包含：`access_token`、`token_type`、`expires_in`（秒）。

## Step 3: 使用 Token 调用 API

在请求头中携带：

```
Authorization: Bearer <your_access_token>
```

## Token 生命周期

- Access Token 有**有限的生命周期**
- 过期后需重新请求新 Token
- 建议**缓存 Token**，在接近过期时刷新，避免不必要的 API 调用

## Requestor-ID Header

**除以下接口外**，所有请求都必须携带 `Requestor-ID` 请求头，值为发起请求的自然人 ID（natural person id）：

不需要 `Requestor-ID` 的接口：
- Create natural person / natural persons wizards
- Create legal entity customer / prepare legal entity

> [!NOTE] 原文提到："The logic with the validation of permissions to perform a specific command will be added in the future."（权限校验逻辑将在未来添加）——说明当前 `Requestor-ID` 仅用于标识请求发起者，尚未实际做权限校验。

## API Reference

- [Authorization — Get Token](https://docs.tradevest.ai/api-reference/authorization) — OAuth Token 端点
