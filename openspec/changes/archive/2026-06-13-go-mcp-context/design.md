## Context

go-mcp-context 是对标 Context7 的私有化文档检索服务。Context7 运行在公网云端，企业内网无法访问；同时企业内部存在大量私有技术文档（PDF、DOCX、内部 Markdown）需要被 AI IDE 检索。本系统在企业内网完整自部署，提供与 Context7 等价的 MCP 工具接口。

**技术约束：**
- 必须兼容标准 MCP 协议（JSON-RPC 2.0 + Streamable HTTP），与 Cursor/Windsurf/Claude Code 等 IDE 无缝集成
- Embedding 和 LLM 调用通过企业内部 OpenAI 代理出口，不直连公网
- 文档文件存储于七牛云，Go 侧通过 Storage 接口访问（支持后期扩展 S3/MinIO）
- SSO 认证对接企业现有 OAuth 2.0 体系，通过 JWT 公钥（PEM）本地验证

## Goals / Non-Goals

**Goals:**
- 实现完整 MCP Server（search-libraries + get-library-docs），兼容主流 AI IDE
- 支持 Markdown/PDF/DOCX 多格式文档上传、语义分块、向量化、混合检索全流程
- 提供 Vue 3 管理界面，支持库/版本/文档管理和 API Key 自助申请
- 版本无感知刷新：刷新期间 MCP 搜索请求不中断、不降质
- 性能目标：文档处理 < 1 分钟/库，搜索延迟 < 1 秒

**Non-Goals:**
- 不支持自动爬取外部网站（只支持手动上传和 GitHub tarball 导入）
- 不支持 Swagger/JavaDoc/HTML 格式文档解析（预留接口，暂不实现）
- 不支持 PostgreSQL 主从复制和 Redis Cluster（单机部署满足初期需求）
- 不支持本地 Embedding 模型（bge-m3）接入（依赖 OpenAI API 代理）

## Decisions

### D1：两层架构（Library → Chunk），不引入中间 Document 层

**选择**：Library 直接关联 DocumentChunk，Chunk 携带 `version` 字段。DocumentUpload 仅记录上传元数据，不参与检索路径。

**理由**：对标 Context7 的 `/org/project` + Snippet 设计；三层架构（Library → Document → Chunk）在多版本共存时需要跨层聚合，增加查询复杂度。两层架构检索 SQL 只需扫描一张主表，pgvector 索引直接命中。

**替代方案**：三层架构 — 放弃，MVP 阶段过度工程化，Context7 本身也是扁平化设计。

---

### D2：混合搜索用 RRF 算法合并，而非线性加权

**选择**：向量搜索（Top-50）+ BM25 全文搜索（Top-50）各自排名后用 RRF 公式合并，权重 Vector 70% + BM25 30%，再叠加热度加成 20%。

**理由**：向量分数（cosine distance 0~1）与 BM25 分数（ts_rank，依赖文档频率）量纲不同，直接线性加权不可比。RRF 只依赖排名，天然解决量纲问题，k=60 来自 Elasticsearch 生产验证。

**替代方案**：Min-Max 归一化后线性加权 — 放弃，极值样本少时归一化失真严重。

---

### D3：PostgreSQL + pgvector 而非独立向量数据库

**选择**：pgvector HNSW 索引（m=16, ef_construction=64）存储所有向量，与关系数据统一存储。

**理由**：统一存储减少运维复杂度；pgvector 在 99% 召回率下吞吐量 471 QPS（为 Qdrant 的 11.4 倍）；企业内网已有 PostgreSQL 运维能力；ACID 事务保证版本切换原子性。

**替代方案**：Milvus — 预留 `embedding_model` 字段，后期数据量超千万向量时可迁移。

---

### D4：Markdown 语义分块，以标题层级为切割边界

**选择**：按 Markdown 标题（h1-h6）分割成带上下文标题的 section，每个 section 若超过 512 tokens（tiktoken cl100k_base），再按段落和代码块原子单元切分，代码块不可跨块切分。

**理由**：标题边界天然对应文档的语义单元（功能说明、API 参考、示例代码），切分更完整；代码块保护确保一个函数示例不被截断成两段；tiktoken 精确计数避免向量截断问题。

**替代方案**：固定 token 窗口 + 50 token 重叠 — 放弃，容易在代码块中间截断，且重叠区占用额外 Embedding 费用。

---

### D5：LLM Enrich 仅对 code 类型 Chunk 调用，info 类型用标题路径

**选择**：Chunk 分类为 code（含 ``` 代码块）和 info（纯文字）。code Chunk 调用 gpt-4o-mini 生成 title（英文动词短语）和 description（1-3句）；info Chunk 以标题层级路径（h1 > h2 > h3）作为 title，不调用 LLM。Enrich 用 5 个并发 worker 加速。

**理由**：info 类型的结构化语义已由标题层级充分表达，无需额外 LLM 调用；code 类型的意图需要 LLM 理解代码语义才能生成高质量描述，显著提升向量搜索质量。分类处理降低约 50% 的 LLM 调用成本。

**替代方案**：所有 Chunk 都调用 LLM — 放弃，成本加倍但 info 类型收益边际极低。

---

### D6：版本刷新使用 batch_version 无感知更新

**选择**：刷新某版本文档时，所有新 Chunk 以 `status=pending` 写入并打上新的 `batch_version` 时间戳；全部处理完成后，在一个数据库事务中将新 Chunk 切换为 `active`，将旧 Chunk 软删除（`status=deleted`）。

**理由**：如果刷新期间直接替换旧 Chunk，会出现部分文档已更新、部分尚未更新的中间态，MCP 搜索结果不一致。batch_version 机制保证切换是原子的，用户在刷新完成前始终搜到完整的旧版本文档。

**替代方案**：先删除旧版本再写新版本 — 放弃，删除到写入之间存在空窗期，搜索结果为空。

---

### D7：双 Token 认证（SSO JWT + 长期 API Key）

**选择**：管理界面用 SSO OAuth 2.0 回调生成短期 AccessToken（2h）+ RefreshToken（7d）；MCP 调用用用户手动生成的长期 API Key（30 天/90 天/永久），存 SHA256 Hash，撤销通过 Redis 黑名单即时生效（O(1) 查询）。

**理由**：MCP 调用来自 IDE 后台进程，不适合浏览器 OAuth 流程；长期 API Key 符合 IDE 集成场景；Redis 黑名单无需额外数据库查询，撤销生效延迟 < 50ms。

## Risks / Trade-offs

- **[风险] OpenAI API 不可用时文档上传失败** → 缓解：记录 `status='failed'` + `error_message`，支持重新触发；LLM Enrich 失败时有 fallback（用标题路径和文本片段），不阻塞 Embedding 生成
- **[风险] 七牛云 DeleteByPrefix 操作失败导致旧文件残留** → 缓解：数据库软删除优先于文件删除；旧文件残留不影响检索（Chunks 已标记 deleted），但会产生存储浪费，通过定期清理脚本补偿
- **[风险] pgvector HNSW 索引构建内存消耗大** → 缓解：`ef_construction=64`（较保守），批量写入后自动触发增量索引更新
- **[权衡] RRF 热度加成（20%）无时间衰减** → 老旧但高访问文档会持续排名靠前；可后期加时间衰减因子
- **[权衡] 同一大版本只保留最新小版本** → 用户无法对比两个小版本 API 差异；通过版本号显示告知用户当前版本

## Deployment

1. 将 `configs/config.yaml.example` 复制为 `configs/config.yaml`，填写 PostgreSQL/Redis/OpenAI 代理/七牛云/SSO 参数
2. 执行 `docker-compose -f docker-compose.prod.yml up -d`，GORM AutoMigrate 自动建表和创建索引
3. 访问管理界面，通过 SSO 登录后在用户中心生成第一个 API Key
4. 在 IDE 中配置 MCP Server URL 和 API Key，验证 `search-libraries` 工具可正常调用

## Open Questions

- 搜索延迟 P99 在 Chunk 数量超过 100 万时是否仍能 < 1s？（需压测验证）
- GitHub tarball 导入时如遇 rate limit，当前无重试退避，是否需要加队列？
- 多租户隔离（不同部门的文档是否需要隔离访问权限）？`created_by` 字段预留但未实现权限过滤
