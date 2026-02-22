# Prices

> Source: https://docs.tradevest.ai/documentation/prices

## 概述

Tradevest API 提供基于 **WebSocket** 的实时数字资产价格订阅端点，允许 Partner 订阅并接收数字资产的买卖盘价格流。

**端点：** `GET /websockets/prices`

---

## 连接流程

1. 客户端通过 HTTPS 连接到 WebSocket Prices 端点
2. 升级为 WebSocket 协议（大多数 WebSocket 客户端库自动处理）
3. 客户端发送一条 `PRICE_SUBSCRIPTION` 消息，指定需订阅的法币/数字资产交易对
4. 服务端处理订阅后，持续推送 `PRICES` 消息
5. 流持续至出错或任一方断开连接

---

## 消息类型

通过 `type` 字段区分三种消息类型：

| type 值 | 方向 | 说明 |
|---------|------|------|
| `PRICE_SUBSCRIPTION` | Client → Server | 订阅请求 |
| `PRICES` | Server → Client | 价格数据推送 |
| `ERROR` | Server → Client | 错误通知 |

### PRICE_SUBSCRIPTION（订阅请求）

```json
{
  "type": "PRICE_SUBSCRIPTION",
  "priceSubscription": {
    "markets": [
      { "currency": "EUR", "cryptoSymbol": "BTC" },
      { "currency": "EUR", "cryptoSymbol": "ETH" }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 固定值 `PRICE_SUBSCRIPTION` |
| `priceSubscription.markets` | array | 否 | 订阅的交易对列表。**省略时订阅平台支持的所有 EUR / cryptoSymbol 交易对** |
| `markets[].currency` | string | 是 | 法币代码（如 `EUR`） |
| `markets[].cryptoSymbol` | string | 是 | 数字资产代码（如 `BTC`, `ETH`） |

### PRICES（价格推送）

```json
{
  "type": "PRICES",
  "prices": [
    {
      "market": { "currency": "EUR", "cryptoSymbol": "BTC" },
      "time": "2024-07-10T08:01:22.783+02:00",
      "asks": [
        { "price": "54577.54", "quantity": "0.01832255" },
        { "price": "54578.84", "quantity": "0.07328847" }
      ],
      "bids": [
        { "price": "54564.14", "quantity": "0.01832705" },
        { "price": "54559.06", "quantity": "0.07331504" }
      ]
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 固定值 `PRICES` |
| `prices` | array | 各交易对的价格数据 |
| `prices[].market` | object | 交易对标识（currency + cryptoSymbol） |
| `prices[].time` | string | ISO 8601 时间戳 |
| `prices[].asks` | array | 卖盘列表（price + quantity） |
| `prices[].bids` | array | 买盘列表（price + quantity） |
| `asks/bids[].price` | string | 价格（decimal） |
| `asks/bids[].quantity` | string | 数量（decimal） |

### ERROR（错误通知）

```json
{
  "type": "ERROR",
  "error": "Error message",
  "connectionId": "0116ec3f-6237-4c13-97b0-c3de46c5f6f1"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 固定值 `ERROR` |
| `error` | string | 错误描述 |
| `connectionId` | string (UUID) | 连接标识符，用于向 Tradevest 报告问题时的追踪参考 |

---

## 通信序列图

```
  API Client                           Our API
     |                                    |
  1. | HTTP connect /websockets/prices    |
     |----------------------------------->|
  2. | Upgrade to WebSocket               |
     |----------------------------------->|
     | OK                                 |
     |<-----------------------------------|
     |                                    |
  3. | PRICE_SUBSCRIPTION                 |
     | (EUR/ETH, EUR/BTC)                 |
     |----------------------------------->|
     |                                    |
     |  [Loop: long running subscription] |
     |                                    |
  4. | PRICES (EUR/ETH)                   |
     |<-----------------------------------|
  5. | PRICES (EUR/BTC)                   |
     |<-----------------------------------|
  6. | PRICES (EUR/ETH)                   |
     |<-----------------------------------|
     |           ...                      |
     |                                    |
     |  [Opt: error encountered]          |
     |                                    |
  7. | ERROR                              |
     |<-----------------------------------|
  8. | HTTP disconnect                    |
     |<-----------------------------------|
     |                                    |
```

---

## 行为规则

| 规则 | 说明 |
|------|------|
| **会话保持** | 无价格变动时，服务端每秒发送 `asks` 和 `bids` 为空数组的 `PRICES` 消息以维持会话 |
| **单向通信** | 订阅处理完成后，服务端不再期望接收任何客户端消息 |
| **修改订阅** | 不支持在已有连接上修改订阅内容。需断开连接，重新建立新连接并发送新的 `PRICE_SUBSCRIPTION` |
| **连接稳定性** | 客户端应假设服务端可能随时关闭 WebSocket 连接，需实现重连机制 |

---

## OpenAPI Models

消息结构定义参见 WebSocket Prices schemas：

| Schema | 说明 |
|--------|------|
| `WebSocketRequest` | 请求消息包装器（`PRICE_SUBSCRIPTION`） |
| `WebSocketResponse` | 响应消息包装器（`PRICES` / `ERROR`），通过 `type` 字段区分 |

---

## 相关文档

- [Status Codes for Financial Operations](./status-codes-for-financial-operations.md) — Order / Transfer / Transaction 状态码
- [Partner Webhooks](./partner-webhooks.md) — Webhook 异步通知机制（与 WebSocket 实时推送互补）

### API Reference

- [WebSocket Prices](https://docs.tradevest.ai/websockets/websocket-prices/prices) — WebSocket 价格订阅接口
- [WebSocket Prices Schemas](https://docs.tradevest.ai/websockets/websocket-prices/schemas) — WebSocket 消息数据结构定义
