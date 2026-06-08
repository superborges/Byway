# Phase 3.5 本地家庭 PIN / Session 设计

## 目标

Byway 第一阶段不做复杂账号体系，但正式使用不能让本机 Proxy API 完全裸露。Phase 3.5 增加轻量家庭 PIN 和 Bearer session token：

- 未配置 PIN：保持开发和自动化测试的无鉴权体验。
- 配置 `BYWAY_AUTH_PIN`：除 `/health` 和 `/api/auth/*` 外，所有 `/api/*` 需要 `Authorization: Bearer <token>`。
- 前端检测 auth 状态，需要时展示 PIN 输入；成功后把 token 存到 localStorage，并为后续请求带上 Authorization。

## API

### `GET /api/auth/status`

返回：

```json
{
  "enabled": true,
  "authenticated": false
}
```

若未启用 PIN，`enabled=false` 且 `authenticated=true`。

### `POST /api/auth/session`

请求：

```json
{
  "pin": "123456"
}
```

成功返回：

```json
{
  "ok": true,
  "token": "..."
}
```

失败返回 401，不透露正确 PIN。

## Token 策略

第一阶段使用本地 HMAC token：

- `BYWAY_AUTH_SECRET` 配置时使用该 secret。
- 未配置 secret 时使用 PIN 派生，适合单机自用。
- token 只代表当前本地家庭用户 `local_family`，不引入多用户模型。

## 边界

- 这不是公网多租户鉴权方案。
- 不在前端保存 Hermes、高德或 LLM key。
- 不做密码找回、账号注册、短信登录。
