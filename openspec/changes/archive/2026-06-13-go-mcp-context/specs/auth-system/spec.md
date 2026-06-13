## ADDED Requirements

### Requirement: 用户通过 SSO 登录管理界面
The system SHALL 支持通过企业 SSO OAuth 2.0 完成登录，颁发 AccessToken（2h）和 RefreshToken（7d），用于管理界面的 API 鉴权。

#### Scenario: SSO 登录流程
- **WHEN** 用户访问 GET `/api/auth/sso_login_url`
- **THEN** 返回企业 SSO 的 OAuth 授权 URL

#### Scenario: OAuth 回调处理
- **WHEN** 用户完成 SSO 授权后浏览器重定向到 GET `/api/auth/callback?code=xxx`
- **THEN** 服务端用 code 换取 SSO Token，用 PEM 公钥本地验证 JWT，生成本地 AccessToken（2h）+ RefreshToken（7d）
- **AND** 前端收到两个 Token，存储于 localStorage

#### Scenario: AccessToken 过期刷新
- **WHEN** 前端检测到 401，携带 RefreshToken 调用 POST `/api/auth/refresh`
- **THEN** 验证 RefreshToken 有效，返回新的 AccessToken（2h）

#### Scenario: 用户主动登出
- **WHEN** POST `/api/auth/logout`（携带 AccessToken）
- **THEN** AccessToken 写入 Redis 黑名单（TTL = 剩余有效期），前端收到 200 后清除本地 Token

#### Scenario: RefreshToken 过期
- **WHEN** RefreshToken 已超过 7d
- **THEN** 返回 HTTP 401，前端跳转 SSO 重新登录

---

### Requirement: 用户可生成和管理长期 API Key 用于 MCP 调用
The system SHALL 支持已登录用户创建长期 API Key（30 天/90 天/永久），查看列表（Token 脱敏显示），以及撤销 Key（Redis 黑名单即时生效）。

#### Scenario: 生成 API Key
- **WHEN** POST `/api/v1/api-keys/create`，body `{"name": "cursor-work", "expires_in": "90d"}`
- **THEN** 生成随机 Token，明文仅在此响应中返回一次
- **AND** 数据库存储 SHA256(token)、token 后 4 位、name、user_uuid、expires_at

#### Scenario: 查看 API Key 列表
- **WHEN** GET `/api/v1/api-keys/list`
- **THEN** 返回当前用户所有 Key，每条显示 name、`****xxxx`（后 4 位）、created_at、expires_at、usage_count，不返回完整 Token

#### Scenario: 撤销 API Key
- **WHEN** DELETE `/api/v1/api-keys/:id`
- **THEN** 数据库软删除（deleted_at 置为当前时间），Redis 写入永久黑名单 `api_token:blacklist:{token_id}`
- **AND** 被撤销 Token 在下次 MCP 请求时立即被拒绝（< 50ms 生效）

---

### Requirement: MCP 调用通过 API Key 鉴权
The system SHALL 对所有 MCP 请求验证 `MCP_API_KEY` Header 中的 API Key，依次检查格式、SHA256 匹配、Redis 黑名单、过期时间。

#### Scenario: 有效 API Key
- **WHEN** Header `MCP_API_KEY: Bearer {valid_token}`
- **THEN** SHA256 验证通过，Redis 黑名单无记录，expires_at 未到期，请求正常处理

#### Scenario: API Key 缺失或格式错误
- **WHEN** Header 缺少 `MCP_API_KEY` 或无法解析
- **THEN** 返回 JSON-RPC error，code = -32001，message = "missing or invalid api key"

#### Scenario: API Key 已撤销（Redis 黑名单）
- **WHEN** Token 对应的 token_id 在 Redis 黑名单中存在
- **THEN** 返回 JSON-RPC error，code = -32001，message = "api key revoked"

#### Scenario: API Key 已过期
- **WHEN** Token 的 expires_at < 当前时间
- **THEN** 返回 JSON-RPC error，code = -32001，message = "api key expired"
