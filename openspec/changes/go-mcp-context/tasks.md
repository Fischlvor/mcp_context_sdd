## 1. 项目初始化与基础设施

- [x] 1.1 初始化 `server-mcp` Go 模块（`go.mod`），添加 gin、gorm、pgvector-go、go-redis、mcp-golang、go-openai、docconv、zap、tiktoken-go、swaggo/swag 依赖
- [x] 1.2 初始化 `web-mcp` Vue 3 + TypeScript 项目（vite），添加 element-plus、pinia、axios、vue-router 依赖
- [x] 1.3 创建 `configs/config.yaml`（含 PostgreSQL、Redis、OpenAI、七牛云、SSO 配置项），实现 `pkg/config` 读取逻辑
- [x] 1.4 实现 `pkg/core/zap.go`：结构化日志初始化（开发/生产两种格式）
- [x] 1.5 实现 `internal/initialize/gorm.go`：连接 PostgreSQL，启用 pgvector 扩展，GORM AutoMigrate 8 张表，创建所有索引（HNSW m=16 ef_construction=64、GIN tsvector、复合索引）
- [x] 1.6 实现 `internal/initialize/redis.go`：Redis 客户端初始化和健康检查
- [x] 1.7 创建 `docker-compose.prod.yml`（PostgreSQL 15 + pgvector、Redis 7、Go Server、Vue Nginx）和 `deploy.sh` 部署脚本

## 2. 数据模型

- [x] 2.1 实现 `internal/model/database/` 下 8 个 GORM 模型：Library（含 embedding vector(1536)、batch_version、access_count）、DocumentUpload（含 content_hash、status、batch_version）、DocumentChunk（含 chunk_type、language、code、embedding、batch_version、access_count、status）、SearchCache、APIKey（含 token_sha256、token_last4、expires_at）、Statistics、ActivityLog、MCPCallLog
- [x] 2.2 实现 `internal/model/request/` 下所有请求 DTO（库管理、文档上传、搜索、API Key、GitHub 导入）
- [x] 2.3 实现 `internal/model/response/` 下所有响应 DTO（库详情、文档片段、搜索结果、统计数据）
- [x] 2.4 实现 `pkg/global/` 全局变量（DB、Redis、Config、Logger、EmbeddingService、LLMService、Storage）

## 3. 认证系统

- [x] 3.1 实现 `internal/middleware/sso_jwt.go`：解析 SSO 公钥（PEM），验证 JWT AccessToken，Redis 黑名单检查，提取 user_uuid 写入 Gin context
- [x] 3.2 实现 `internal/middleware/apikey.go`：从 `MCP_API_KEY` Header 提取 Token，SHA256 查库，Redis 黑名单验证，过期检查（依次按格式→SHA256→黑名单→过期顺序）
- [x] 3.3 实现 `internal/api/auth.go` + `internal/service/auth.go`：SSO 登录 URL 生成（`GET /api/auth/sso_login_url`）、OAuth 回调处理（`GET /api/auth/callback`）、AccessToken（2h）/RefreshToken（7d）颁发、Token 刷新（`POST /api/auth/refresh`）
- [x] 3.4 实现登出端点（`POST /api/auth/logout`）：将 AccessToken 写入 Redis 黑名单（TTL = 剩余有效期），返回 200
- [x] 3.5 实现 `internal/api/apikey.go` + `internal/service/apikey.go`：API Key 生成（随机 Token + SHA256 存储，明文仅返回一次）、列表查询（`****xxxx` 脱敏显示）、撤销（软删除 + Redis 永久黑名单 `api_token:blacklist:{token_id}`）
- [x] 3.6 实现 `internal/api/user.go`：获取当前登录用户信息（从 SSO Token 解析 name/email/avatar）

## 4. 文档处理流水线

- [x] 4.1 实现 `pkg/parser/markdown.go`：Markdown → 纯文本提取，保留代码块标识
- [x] 4.2 实现 `pkg/parser/pdf.go`：基于 docconv 的 PDF → 文本提取，支持多页合并
- [x] 4.3 实现 `pkg/parser/docx.go`：基于 docconv 的 DOCX → 文本提取
- [x] 4.4 实现 `pkg/preprocessor/markdown.go`（`preProcessMarkdown`）：移除 `[![...](...)` 徽章、`<!-- -->` HTML 注释、HTML `<img>` 标签、连续空行（压缩为单空行）、`---` 水平分隔线；正文内容、代码块、标题层级完整保留
- [x] 4.5 实现 `pkg/tokenizer/tiktoken.go`：封装 tiktoken cl100k_base，提供 `CountTokens(text string) int` 接口，用于精确 token 计数（替代字符/4 估算）
- [x] 4.6 实现 `pkg/chunker/markdown_chunker.go`（`splitMarkdownWithMetadata`）：以 h1-h6 标题为主切割边界，每个 section 携带完整标题层级路径（如 "Getting Started > Installation"）；section 超过 512 tokens 时按段落和代码块原子单元继续切分；代码块（``` 包裹）不跨块切断；代码块内的 `#` 注释行不触发 section 分割
- [x] 4.7 实现 chunk 类型检测：Chunk 含代码块时 chunk_type = "code"（提取 language 和 code 字段），否则 chunk_type = "info"（title = 标题层级路径，不调用 LLM）
- [x] 4.8 实现 `pkg/llm/openai.go`：LLMService 封装（gpt-4o-mini），实现 `EnrichChunk` 方法，输入 code 类型 Chunk 文本，输出 `{"title": ..., "description": ...}` JSON；调用失败时 fallback 保留标题路径
- [x] 4.9 实现 `pkg/llm/enrich_worker.go`：5 个并发 worker goroutine 消费 Enrich 任务队列，仅处理 code 类型 Chunk，info 类型 Chunk 直接跳过进入 Embedding 阶段
- [x] 4.10 实现 `pkg/embedding/openai.go`：EmbeddingService 封装（text-embedding-3-small，1536 维），批量生成（最多 100 条/批），Redis 缓存（key = text SHA256 hash），429 限流时指数退避重试（最多 3 次，间隔 1s/2s/4s）
- [x] 4.11 实现 `pkg/storage/qiniu.go`：QiniuStorage 实现 Storage 接口（Upload/Delete/DeleteByPrefix/ListByPrefix/GetPublicURL）
- [x] 4.12 实现 `internal/service/document.go`：文档处理主流程（上传 → SHA256 去重 → 预处理 → 分块 → 类型检测 → Enrich → Embedding → 批量写入 Chunks），document_uploads 状态跟踪（pending → processing → completed / failed）
- [x] 4.13 实现文档删除：`DELETE /api/v1/documents/:id` 软删除（deleted_at）关联 Chunks，触发 Redis 搜索缓存按 library_id 前缀失效
- [x] 4.14 实现 `internal/api/document.go`：文档上传 API（同步 + SSE），文档列表，文档详情，Chunks 查询，文档删除
- [x] 4.15 实现 `internal/middleware/mcplog.go`：MCP 调用日志中间件，记录 func_name/params/latency_ms/result_count/status/error_msg 到 mcp_call_logs

## 5. 库管理和 GitHub 集成

- [x] 5.1 实现 semver 版本校验函数（`ValidateVersion`）：正则 `^v?(\d+)\.(\d+)\.(\d+)(-[a-zA-Z0-9.]+)?$`，不符合时返回 "invalid version format" 错误
- [x] 5.2 实现 `internal/service/library.go`：库 CRUD，版本创建（semver 校验 + 同大版本旧 Chunks 软删除 + 七牛云文件清理）、版本删除、版本列表查询（含 chunk_count/token_count）
- [x] 5.3 实现库 embedding 异步生成：库创建或更新 name/description 时，异步 goroutine 调用 EmbeddingService 生成 `name + description` 的 1536 维向量，存入 libraries.embedding
- [x] 5.4 实现无感知版本刷新（`RefreshVersionWithCallback`）：新 Chunks 以 `status=pending` + 新 `batch_version` 写入；全部完成后在单个 DB 事务中将新 Chunks 切为 active、旧 Chunks 切为 deleted；SSE 推送 start → doc_N/total → activating → complete；失败文档推送 warning 事件
- [x] 5.5 实现 `internal/api/library.go`：库管理 REST API（CRUD + 版本管理端点 + `POST .../refresh-sse`），路由注册
- [x] 5.6 实现 `pkg/github/client.go`：GitHub API 客户端（Release 列表获取，tarball 流式下载）
- [x] 5.7 实现 `internal/service/github.go`：GitHub 文档导入服务（init-import：LLM 从 URL+README 生成库名；import-sse：tarball 解压过滤 .md 文件，调用文档处理流水线）
- [x] 5.8 实现 GitHub API 端点：`POST /api/v1/libraries/github/init-import`、`POST /api/v1/libraries/github/import-sse`、`GET /api/v1/libraries/github/releases`

## 6. 混合搜索引擎

- [x] 6.1 实现向量搜索：生成查询 Embedding，执行 pgvector cosine `<=>` 搜索 Top-50，支持 library_id（0 表示全局）/version/chunk_type/status 过滤
- [x] 6.2 实现 BM25 全文搜索：`chunk_tsvector_simple @@ plainto_tsquery('simple', ?)` + ts_rank 降序，Top-50，相同过滤条件
- [x] 6.3 实现 RRF 合并算法（`hybridRRF`）：合并两路结果，计算 `0.7/(vector_rank+60) + 0.3/(bm25_rank+60) + 0.2*(access_count/max_access_count)`，去重，分页返回 Top-10
- [x] 6.4 实现多 topic 并行搜索（`searchMultiTopicsWithRRF`）：按逗号分隔 topic 参数，对每个 sub-topic 并行执行完整混合搜索，结果再次 RRF 合并取 Top-10；单 topic 时直接走单路搜索
- [x] 6.5 实现异步 access_count 更新：搜索返回后，后台 goroutine 对命中 Chunks 的 access_count 各自 +1，不阻塞搜索响应
- [x] 6.6 实现 Redis 搜索缓存：key `search:topic:{library_id}:{version}:{mode}:{topic_md5}`，24h TTL，命中直接返回；文档更新/删除/刷新后按 `search:topic:{library_id}:*` 前缀批量失效
- [x] 6.7 实现 `internal/api/search.go`：搜索 API，支持 library_id/version/mode/topic/page 参数
- [x] 6.8 实现 `pkg/vectorstore/store.go`：向量存储访问层，封装 pgvector 批量写入和查询操作

## 7. MCP Server

- [x] 7.1 实现 `internal/transport/` 传输层：HTTP/SSE/Streamable HTTP 协议检测和分发（`detector.go`、`factory.go`）
- [x] 7.2 实现 MCP search-libraries 工具：向量语义搜索（cosine distance < 0.7 阈值）优先，无结果时降级为前缀匹配（score=0.9）→ 包含匹配（score=0.8）→ Levenshtein 相似度（最高 0.7 分），返回 libraries 数组（含 library_id、name、description、versions、default_version、snippets、score）
- [x] 7.3 实现 MCP get-library-docs 工具：接受 library_id/version/mode/topic/page 参数，调用混合搜索（含多 topic 并行），格式化返回 documents 数组；topic 支持逗号分隔多值
- [x] 7.4 实现 `internal/service/mcp_handler.go`：统一 MCP 请求处理器（initialize/initialized/tools/list/tools/call），记录 mcp_call_logs
- [x] 7.5 实现 `internal/api/mcp.go`：MCP 协议端点（`POST /mcp`），挂载 apikey 中间件和 mcplog 中间件
- [x] 7.6 实现基础端点：`GET /api/base/health`（返回 `{"status":"ok","version":"..."}`)、`GET /swagger/index.html`（Swagger UI，`swag init` 生成 docs/）
- [x] 7.7 实现 `internal/api/stats.go`：统计数据 API（`GET /api/v1/stats`），`pkg/bufferedwriter/` 缓冲写入统计计数器（异步写 statistics 表）
- [x] 7.8 实现 `internal/api/activity_log.go`：活动日志 API，支持按 library_id/task_id 查询，事件类型覆盖：EventDocParse、EventDocChunk、EventDocEnrich、EventDocEmbed、EventDocComplete、EventDocFailed
- [x] 7.9 实现 `internal/router/enter.go`：路由总装配，注册所有 API 分组（auth、v1、base、mcp），挂载全局中间件（CORS、logger）
- [x] 7.10 实现 `main.go`：应用启动入口，依次初始化 config/zap/gorm/redis/embedding/llm/storage/router，启动 HTTP Server

## 8. Vue 3 前端

- [x] 8.1 实现 `src/utils/request.ts`：axios 封装（baseURL、JWT 拦截器、401 自动刷新 Token、统一错误处理）
- [x] 8.2 实现 `src/api/` 下所有接口模块：auth.ts（含 logout）、library.ts、document.ts（含 delete）、search.ts、apikey.ts
- [x] 8.3 实现 `src/stores/user.ts`：Pinia store（用户信息、Token 存储、登录/登出）
- [x] 8.4 实现 `src/router/index.ts`：路由配置（Dashboard/Library/Search/User），路由守卫（未登录跳转 SSO）
- [x] 8.5 实现 `src/views/dashboard/index.vue`：统计卡片（总库数、总 Chunk 数、总 Token 数、MCP 调用次数）+ 最近 7 天 MCP 调用趋势图
- [x] 8.6 实现 `src/views/library/index.vue`：库列表（语义搜索/创建/删除）和 `admin.vue`（版本管理 + GitHub 导入）
- [x] 8.7 实现 `src/views/library/detail.vue`：库详情（版本列表、文档列表、文档删除按钮）+ `src/components/AddDocsModal.vue`（文件拖拽上传 + SSE 逐文件进度条）
- [x] 8.8 实现 `src/views/search/index.vue`：搜索测试页（库/版本/mode 选择器、多 topic 输入、结果列表 + relevance 分数 + 翻页）
- [x] 8.9 实现用户中心（`src/views/home/index.vue`）：个人信息展示 + API Key 管理表格（生成弹窗含一次性明文展示与复制 + 撤销二次确认）
- [x] 8.10 实现 `src/views/SSOCallback.vue`：SSO 回调页面，解析 code 参数，换取 Token，跳转 Dashboard

## 9. 测试

- [x] 9.1 编写 `test/unit/library_test.go`：库 CRUD、semver 校验、版本管理、embedding 触发逻辑的单元测试（mock DB）
- [x] 9.2 编写 `test/unit/processor_test.go`：Markdown 语义分块器单元测试，覆盖：正常标题切分、超 512 tokens 继续切分、代码块保持原子、代码块内 `#` 不触发切分、无标题文档、极短文档
- [x] 9.3 编写 `test/unit/processor_test.go`：preProcessMarkdown 单元测试，覆盖：移除徽章、HTML 注释、独立图片、分隔线、压缩空行
- [x] 9.4 编写 `test/unit/search_test.go`：RRF 合并算法单元测试（向量只命中/BM25 只命中/两路都命中/多 topic 合并）；access_count 热度加成验证
- [x] 9.5 编写 `test/unit/mcp_test.go`：MCP 工具调用单元测试（search-libraries 向量优先+降级路径；get-library-docs 单/多 topic）
- [x] 9.6 编写 `test/unit/document_test.go`：文档处理流水线单元测试（SHA256 去重、Enrich fallback、批量 Embedding 重试逻辑）
- [x] 9.7 编写 `test/integration/mcp_handler_integration_test.go`：MCP 集成测试（需真实 DB + Redis）
- [x] 9.8 编写 `test/unit/apikey_test.go`：API Key 生成/SHA256 存储/脱敏显示/撤销/黑名单验证测试
- [x] 9.9 运行所有测试，确保覆盖率 ≥ 80%（核心服务层）

## 10. 部署与文档

- [ ] 10.1 编写各服务 `Dockerfile`（server-mcp 多阶段构建，web-mcp nginx 静态托管）
- [ ] 10.2 配置 `web-mcp/nginx.conf`：前端静态资源服务 + `/api/` 和 `/mcp` 反向代理到 Go Server
- [ ] 10.3 运行 `swag init` 生成 Swagger 文档（`docs/swagger.json`），验证 `GET /swagger/index.html` 可访问
- [ ] 10.4 编写 `docs/MCP.md`：MCP 工具文档（search-libraries 和 get-library-docs 输入/输出格式、多 topic 用法、IDE 配置示例）
- [ ] 10.5 编写 `docs/DEPLOYMENT.md`：部署指南（环境变量说明、docker-compose 启动步骤、常见问题）
- [ ] 10.6 验收测试：部署到测试环境，配置 Claude Code MCP，验证 search-libraries 和 get-library-docs 工具可正常调用，多 topic 并行搜索正确，检索延迟 < 1s
