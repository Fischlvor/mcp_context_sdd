## ADDED Requirements

### Requirement: 用户可创建和管理技术库
The system SHALL 支持创建、查询、更新、删除技术库（Library），每个库有唯一名称、描述和来源类型（local/github）。

#### Scenario: 创建 local 类型库
- **WHEN** POST `/api/v1/libraries`，body `{"name": "react", "description": "..."}`
- **THEN** 数据库新增 libraries 记录，status = "active"，异步触发 name+description 的 embedding 生成（用于 search-libraries 语义搜索）
- **AND** 返回新库的完整信息（含 id）

#### Scenario: 库名重复
- **WHEN** POST `/api/v1/libraries`，name 与已有 active 库相同
- **THEN** 返回 HTTP 409，message = "library name already exists"

#### Scenario: 更新库名或描述
- **WHEN** PUT `/api/v1/libraries/:id`，body `{"name": "React", "description": "Updated"}`
- **THEN** 更新 name 和 description 字段，异步重新生成 embedding
- **AND** 不影响该库下的已有 Chunks 和版本

#### Scenario: 删除库及其所有数据
- **WHEN** DELETE `/api/v1/libraries/:id`
- **THEN** 库及关联的 document_uploads、document_chunks 级联软删除（deleted_at 置为当前时间）

---

### Requirement: 系统支持库的语义向量搜索和分页列表
The system SHALL 在获取库列表时，当查询参数包含 name 时优先使用向量语义搜索，无语义匹配时降级为模糊匹配，支持按 MCP 调用次数排序。

#### Scenario: 按名称语义搜索库列表
- **WHEN** GET `/api/v1/libraries?name=状态管理`，该词与 "Pinia" 库的 embedding cosine distance < 0.7
- **THEN** 返回结果包含 "Pinia" 库，按相似度排列

#### Scenario: 按 MCP 调用次数排序
- **WHEN** GET `/api/v1/libraries?sort=mcp_calls`
- **THEN** 返回库列表，按该库的 MCP get-library-docs 调用总次数降序排列

---

### Requirement: 系统支持库的多版本管理（semver 格式校验）
The system SHALL 支持同一库下多个大版本共存，每个大版本只保留最新小版本；版本号须符合语义化版本规范（支持 `v1.0.0`、`1.0.0`、`v1.0.0-alpha` 等格式）。

#### Scenario: 创建有效版本
- **WHEN** POST `/api/v1/libraries/:id/versions`，body `{"version": "v18.3.0"}`
- **THEN** 版本号经 semver 正则校验通过（`^v?(\d+)\.(\d+)\.(\d+)(-[a-zA-Z0-9.]+)?$`），Library.versions 数组添加该版本
- **AND** 若已存在同大版本（如 18.x.x），将旧版本 Chunks 软删除（status='replaced'）

#### Scenario: 版本格式非法
- **WHEN** 版本号为 "latest" 或 "abc"（不符合 semver）
- **THEN** 返回 HTTP 400，message = "invalid version format"

#### Scenario: 删除指定版本
- **WHEN** DELETE `/api/v1/libraries/:id/versions/:version`
- **THEN** 该版本所有 Chunks 软删除，Library.versions 数组移除该版本号

#### Scenario: 获取版本列表（含统计）
- **WHEN** GET `/api/v1/libraries/:id/versions`
- **THEN** 返回 versions 数组，每个版本包含 version、chunk_count、token_count、created_at

---

### Requirement: 系统支持版本文档无感知刷新
The system SHALL 支持对某版本文档重新处理（重新分块、Enrich、Embedding），刷新期间搜索服务不中断，通过 batch_version 机制保证原子切换。

#### Scenario: SSE 无感知刷新（RefreshVersionWithCallback）
- **WHEN** POST `/api/v1/libraries/:id/versions/:version/refresh-sse`
- **THEN** 所有新 Chunk 以 `status=pending` + 新 `batch_version` 写入，不影响已有 `status=active` 的 Chunks
- **AND** 全部处理完成后，在单个数据库事务中：将新 Chunk 切为 active，将旧 Chunk 切为 deleted
- **AND** SSE 流推送进度：start → doc_N/total → activating → complete

#### Scenario: 刷新期间 MCP 搜索不受影响
- **WHEN** 版本刷新正在进行（新 Chunks 尚未激活）
- **THEN** MCP get-library-docs 搜索结果仍为旧版本的完整 Chunks（status=active），不出现空结果

#### Scenario: 刷新处理部分文档失败
- **WHEN** 某文档的 Embedding 生成失败，其他文档正常处理
- **THEN** 仅失败文档不参与 batch_version 切换；成功文档仍正常激活
- **AND** SSE 推送 warning 事件，报告失败文件数

---

### Requirement: 系统支持从 GitHub Release 导入文档
The system SHALL 支持从 GitHub 仓库的 Release tarball 自动下载并导入 Markdown 文档，支持同步和 SSE 实时进度两种模式，LLM 自动生成库名。

#### Scenario: 初始化 GitHub 库（首次导入）
- **WHEN** POST `/api/v1/libraries/github/init-import`，body `{"source_url": "https://github.com/vercel/next.js"}`
- **THEN** 验证仓库连通性（请求 GitHub API），LLM 从仓库 URL + README 描述生成友好库名
- **AND** 创建新 Library 记录（source_type="github"），返回 library_id 和可用 Release 版本列表

#### Scenario: SSE 方式导入 GitHub 文档
- **WHEN** POST `/api/v1/libraries/github/import-sse`，body `{"library_id": 1, "version": "v15.0.0", "tag": "v15.0.0"}`
- **THEN** SSE 流推送进度：start → downloading → extracting → doc_N/total → complete
- **AND** 只导入 tarball 中的 `.md` 文件，跳过其他格式

#### Scenario: GitHub Release 不存在
- **WHEN** 指定的 tag 在 GitHub Releases 中不存在
- **THEN** SSE 推送 error 事件，message = "release not found"，HTTP 连接关闭
