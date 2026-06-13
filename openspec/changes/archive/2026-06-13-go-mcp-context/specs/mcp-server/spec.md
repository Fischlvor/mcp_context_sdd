## ADDED Requirements

### Requirement: MCP Server 支持标准协议握手
The system SHALL 响应 MCP 客户端的 initialize 握手请求，返回服务端能力声明，并接受 initialized 通知。

#### Scenario: IDE 发起初始化握手
- **WHEN** IDE 发送 `initialize` 请求（携带 clientInfo 和 protocolVersion）
- **THEN** 服务端返回包含 serverInfo、capabilities（tools.listChanged = false）的标准 MCP 响应

#### Scenario: 客户端发送 initialized 通知
- **WHEN** 客户端发送 `notifications/initialized` 通知
- **THEN** 服务端记录 MCP 调用日志，不返回响应体

---

### Requirement: MCP Server 通过 tools/list 暴露工具清单
The system SHALL 响应 `tools/list` 请求，返回 `search-libraries` 和 `get-library-docs` 两个工具的名称和参数 schema。

#### Scenario: 列出工具
- **WHEN** 客户端发送 `tools/list` 请求
- **THEN** 返回包含两个工具定义的数组，每个工具包含 name、description 和 inputSchema（JSON Schema 格式）

---

### Requirement: search-libraries 工具按库名搜索并返回版本信息
The system SHALL 通过 `search-libraries` 工具，对 libraryName 参数执行语义向量搜索（优先）和模糊名称匹配（降级），返回匹配的库列表及其版本信息和文档片段数。

#### Scenario: 向量语义搜索命中
- **WHEN** 调用 `search-libraries`，参数 `{"libraryName": "状态管理"}`，库的语义 embedding 与查询 cosine distance < 0.7
- **THEN** 返回语义匹配的库，score 反映向量相似度，不超过 10 条

#### Scenario: 精确名称匹配
- **WHEN** 参数 `{"libraryName": "react"}`，库名为 "React"（大小写不敏感）
- **THEN** 返回该库，score = 1.0，排在结果第一位

#### Scenario: 向量搜索无结果时降级模糊匹配
- **WHEN** 向量搜索返回空或全部 distance ≥ 0.7
- **THEN** 依次执行前缀匹配（score=0.9）和包含匹配（score=0.8）作为补充
- **AND** 若有名称相近但不完全匹配的库，Levenshtein 相似度最高贡献 0.7 分

#### Scenario: 无任何匹配
- **WHEN** libraryName 与所有库均不匹配
- **THEN** 返回 `{"libraries": []}`，HTTP 200

#### Scenario: 返回结果结构
- **WHEN** 搜索命中任意库
- **THEN** 每条记录包含 library_id（uint）、name、description、versions（数组）、default_version、snippets（active chunk 总数）、score（0-1）

#### Scenario: 缺少或无效 API Key
- **WHEN** 请求 Header 中无 `MCP_API_KEY` 或 Token 已撤销/过期
- **THEN** 返回 JSON-RPC error，code = -32001，message = "unauthorized"，HTTP 200

---

### Requirement: get-library-docs 工具执行混合搜索返回文档片段
The system SHALL 通过 `get-library-docs` 工具，对指定库（或全局，library_id=0）执行混合搜索，支持多 topic 并行搜索，返回分页的文档片段列表。

#### Scenario: 在指定库中搜索 code 类型片段
- **WHEN** 参数 `{"library_id": 1, "mode": "code", "topic": "hooks", "version": "18.3.0", "page": 1}`
- **THEN** 返回最多 10 条 chunk_type IN ('code', 'mixed') 的片段，每条包含 title、description、source、version、mode、language、code、tokens、relevance

#### Scenario: 全局搜索（library_id = 0）
- **WHEN** 参数 `{"library_id": 0, "mode": "info", "topic": "routing"}`
- **THEN** 跨所有库执行混合搜索，返回最相关的 10 条 info 类型片段

#### Scenario: 多 topic 并行搜索
- **WHEN** topic 参数为逗号分隔的多个词，如 `"hooks, performance"`
- **THEN** 对每个 topic 独立执行混合搜索，结果再次用 RRF 合并，返回 Top-10

#### Scenario: 翻页获取更多
- **WHEN** 参数 page = 2
- **THEN** 返回第 11-20 条结果，has_more 指示是否还有后续页

#### Scenario: info 模式返回 content 字段
- **WHEN** mode = "info" 且命中 info 类型片段
- **THEN** 响应中每条包含 content 字段（chunk_text 原文），code/language/description 字段为空

#### Scenario: topic 无命中
- **WHEN** topic 在指定库中向量搜索和全文搜索均无结果
- **THEN** 返回 `{"documents": [], "page": 1, "has_more": false}`

---

### Requirement: MCP Server 记录每次工具调用日志
The system SHALL 将每次 MCP 工具调用写入 `mcp_call_logs` 表，记录调用者、工具名、参数、耗时、结果数量和状态。

#### Scenario: 成功调用记录
- **WHEN** `get-library-docs` 调用成功
- **THEN** mcp_call_logs 新增一行，func_name = "get_library_docs"，status = "success"，latency_ms 为实际耗时毫秒数

#### Scenario: 调用异常记录
- **WHEN** Embedding 服务不可用导致工具调用失败
- **THEN** mcp_call_logs 新增一行，status = "error"，error_msg 记录错误详情

---

### Requirement: 系统提供健康检查和 API 文档端点
The system SHALL 提供健康检查端点和 Swagger 交互文档，供运维监控和开发调试使用。

#### Scenario: 健康检查
- **WHEN** GET `/api/base/health`
- **THEN** 返回 HTTP 200，body 包含 status = "ok" 和服务版本号

#### Scenario: Swagger 文档
- **WHEN** GET `/swagger/index.html`
- **THEN** 返回可交互的 Swagger UI 页面，列出所有 REST API 端点（不含 MCP 端点）
