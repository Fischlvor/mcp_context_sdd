## Purpose

Defines the Vue 3 web management dashboard: the Dashboard statistics page, library and version management UI, multi-file document upload with per-file SSE progress, search testing interface, and the user center with SSO profile and API Key self-service management.

## Requirements

### Requirement: 管理界面展示系统统计 Dashboard
The system SHALL 在 Dashboard 页面展示系统统计摘要（库数量、总 Chunk 数、总 Token 数、MCP 调用次数）和最近 7 天 MCP 调用趋势。

#### Scenario: 查看 Dashboard 统计
- **WHEN** 已登录用户访问 `/dashboard`
- **THEN** 页面展示：总库数、总 Chunk 数、总 Token 数、MCP 调用总次数，以及最近 7 天每日调用量趋势图

---

### Requirement: 用户可在界面管理库和版本
The system SHALL 提供库列表页（含语义搜索和创建/删除入口）和库详情页（版本列表、统计信息、版本操作入口）。

#### Scenario: 库列表页展示
- **WHEN** 访问 `/library`
- **THEN** 表格展示所有 active 库，列：名称、默认版本、版本数、Chunk 数、更新时间
- **AND** 支持按名称语义搜索（实时调用后端库搜索接口）

#### Scenario: 版本管理操作
- **WHEN** 用户在库详情页选中某版本，点击"刷新"
- **THEN** 触发 SSE 无感知刷新，界面展示各文档处理进度
- **AND** 刷新完成前，其他用户的搜索请求不受影响

---

### Requirement: 用户可在界面上传文档并查看逐文件处理进度
The system SHALL 提供文档上传界面，支持拖拽或点击选择多个文件（MD/PDF/DOCX），选择目标库和版本后上传，通过 SSE 为每个文件独立展示处理进度。

#### Scenario: 多文件拖拽上传
- **WHEN** 用户拖拽 3 个文件到上传区域，选择库和版本，点击上传
- **THEN** 每个文件独立展示进度条，阶段依次：上传中 → 预处理 → 分块 → AI 增强 → 向量化 → 完成
- **AND** 某文件失败时进度条变红，显示错误原因，其他文件继续

#### Scenario: 文件大小或格式限制
- **WHEN** 上传 `.xlsx` 格式文件
- **THEN** 前端校验拒绝，提示"仅支持 .md / .pdf / .docx 格式"

---

### Requirement: 用户可在界面测试文档搜索效果
The system SHALL 提供搜索测试页，支持选择库/版本/mode，输入单个或多个 topic（逗号分隔），展示混合搜索结果和相关性分数。

#### Scenario: 单 topic 搜索测试
- **WHEN** 选择库 "React" 版本 "18.3.0" mode "code"，输入 "useState"，点击搜索
- **THEN** 展示最多 10 条结果，每条显示 title、description 前 100 字、source、relevance 分数（保留 2 位小数）

#### Scenario: 多 topic 搜索测试
- **WHEN** 输入 "hooks, performance"（逗号分隔）
- **THEN** 后端并行搜索两个 topic 并 RRF 合并，界面展示跨 topic 的最优结果

#### Scenario: 翻页查看更多结果
- **WHEN** 点击"下一页"
- **THEN** 请求 page+1 的结果并追加到列表，直到 has_more = false

---

### Requirement: 用户中心支持个人信息查看和 API Key 管理
The system SHALL 在用户中心展示 SSO 登录用户的基本信息，并提供 API Key 的完整自助管理界面。

#### Scenario: 查看个人信息
- **WHEN** 已登录用户访问 `/user`
- **THEN** 展示：用户名、邮箱（来自 SSO Token）、账户创建时间

#### Scenario: 生成 API Key（仅显示一次）
- **WHEN** 用户填写名称、选择有效期，点击"生成"
- **THEN** 弹窗展示完整 Token，提供"一键复制"按钮，并提示"Token 仅此一次显示，请立即保存"
- **AND** 弹窗关闭后 API Key 列表刷新，新 Key 以后 4 位（`****xxxx`）显示

#### Scenario: 撤销 API Key 二次确认
- **WHEN** 用户点击"撤销"某条 API Key
- **THEN** 弹出确认对话框，确认后执行撤销，列表中该条记录立即消失
