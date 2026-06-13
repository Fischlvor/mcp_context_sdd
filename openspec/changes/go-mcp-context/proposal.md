## Why

企业内网 AI IDE（Cursor、Windsurf、Claude Code）在 AI 辅助编程时无法访问公网 Context7 服务，导致 LLM 引用过时文档或产生幻觉代码。go-mcp-context 是一个可在企业内网完整自部署的文档检索服务，通过标准 MCP 协议为 AI IDE 提供私有技术文档的智能检索能力，同时支持企业内部私有文档（PDF、DOCX）的接入，弥补公网服务无法覆盖的内部知识库场景。

## What Changes

- MCP Server（JSON-RPC 2.0 + Streamable HTTP），实现 `search-libraries` 和 `get-library-docs` 两个标准 MCP 工具
- 多格式文档处理流水线：Markdown / PDF / DOCX 上传 → 语义分块 → LLM Enrich → Embedding → 存储
- 混合搜索引擎：pgvector 向量搜索 + BM25 全文搜索 + RRF 重排序，支持多 topic 并行搜索
- GitHub 文档自动导入：tarball 流式下载 + SSE 实时进度推送
- 版本无感知刷新：`batch_version` 机制保证刷新期间搜索不中断
- 双重认证体系：SSO JWT（管理界面）+ API Key（MCP 调用）+ Redis 黑名单撤销
- Vue 3 管理界面：库管理、文档上传（SSE 进度）、搜索测试、统计分析、用户中心
- Redis 多层缓存：搜索结果缓存（24h TTL + 版本标签失效）+ Embedding 结果缓存
- Docker Compose 部署方案：PostgreSQL 15 + pgvector + Redis 7 + Go Server + Vue/Nginx

## Capabilities

### New Capabilities

- `mcp-server`: MCP 协议端点，实现 search-libraries（向量搜索优先 + 模糊降级）和 get-library-docs（混合搜索 + 多 topic + 分页）两个工具，支持 JSON-RPC 2.0 和 Streamable HTTP 传输
- `library-management`: 库的增删改查（含 semver 版本校验）、版本管理（创建/删除/同步刷新/SSE 无感知刷新）、GitHub Release 导入（同步/SSE 异步）
- `document-processing`: 多格式文档上传（MD/PDF/DOCX）、Markdown 语义分块（按标题层级切割 + 代码块保护 + tiktoken 计数）、LLM Enrich（code 块生成 title/description，info 块保留标题路径）、Embedding 批量生成（带 Redis 缓存和指数退避重试）
- `hybrid-search`: 向量搜索（pgvector cosine Top-50）+ BM25 全文搜索（Top-50）+ RRF 合并重排 + 热度加成 + 多 topic 并行 + Redis 搜索缓存 + access_count 异步更新
- `auth-system`: SSO 单点登录（OAuth 2.0 + JWT AccessToken 2h / RefreshToken 7d）+ API Key 管理（生成/查看/撤销 + Redis 黑名单即时生效）
- `web-dashboard`: Vue 3 管理界面，含库列表（语义搜索）、版本管理、文档上传（SSE 进度逐文件展示）、搜索测试（多 topic / 分页）、统计 Dashboard、用户中心（API Key 自助管理）

### Modified Capabilities

## Impact

- **后端**：Go 1.23，`server-mcp/` 目录，核心依赖 mcp-golang、Gin、GORM、pgvector-go、go-redis、go-openai、tiktoken-go、docconv、zap、levenshtein
- **前端**：Vue 3 + TypeScript + Vite，`web-mcp/` 目录，核心依赖 element-plus、pinia、axios、vue-router
- **数据库**：PostgreSQL 15 + pgvector 扩展，8 张表（libraries、document_uploads、document_chunks、search_cache、api_keys、statistics、activity_logs、mcp_call_logs），15+ 个索引（HNSW、GIN、复合条件索引）
- **外部服务**：OpenAI API（text-embedding-3-small + gpt-4o-mini，支持内网代理）、七牛云对象存储（支持 Storage 接口扩展 S3/MinIO）、企业 SSO（OAuth 2.0）
- **运行时配置**：`configs/config.yaml`，含数据库连接、Redis、OpenAI 代理、七牛云密钥、SSO 参数、分块大小等
