# User Management

> Source: https://docs.tradevest.ai/documentation/11_user_management

## 创建用户

完整创建一个用户账户需要以下步骤：

```
1. Create User          POST /users
        |
        |  * email 必须唯一
        |  * email 域名必须与 Partner 一致
        v
2. Upload Document      POST /v2/documents
        |
        |  * 上传该用户的 IDENTIFICATION_CERTIFICATE 文档
        v
3. Start Onboarding     POST /user-onboardings
        |
        v
4. Receive Email        收到包含一次性密码 (OTP) 的邮件
        |
        v
5. Login & Set Password 登录 Partner Panel，将一次性密码改为永久密码
        |
        v
6. Configure MFA        配置 authenticator 应用
```

> [!NOTE] 原文第 3 句 "In order to have fully functioning user account **a couple of steps needs** to be accomplished" 语法有误，应为 "a couple of steps **need** to be accomplished"。

## 停用用户

调用 [`Update User`](https://docs.tradevest.ai/api-reference/user-management/users#patch-users-userid) (`PATCH /users/{userId}`)，将 `status` 设为 `INACTIVE`。

> [!NOTE] 原文第 12 句 "**I order** to deactivate a user" 应为 "**In order** to deactivate a user"，疑为拼写错误。

## 重置 MFA

当用户无法访问 authenticator 应用时，需要重置 MFA：

1. 调用 [`Update User`](https://docs.tradevest.ai/api-reference/user-management/users#patch-users-userid) (`PATCH /users/{userId}`)，将 `mfaType` 设为 `SMS`
2. 用户下次登录时，系统会向其绑定的手机号发送短信验证码
3. 登录成功后，系统会要求用户重新配置 authenticator 应用，`mfaType` 将自动恢复

> [!NOTE] 原文第 16 句 "SMS will be **send** to telephone number" 语法有误，应为 "SMS will be **sent**"。

## API Reference

- [Users](https://docs.tradevest.ai/api-reference/user-management/users) — 用户 CRUD 接口
- [User Management Schemas](https://docs.tradevest.ai/api-reference/user-management/schemas) — 数据结构定义
