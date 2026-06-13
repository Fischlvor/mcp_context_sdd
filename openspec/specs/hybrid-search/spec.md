## Purpose

Implements the hybrid search engine combining pgvector cosine similarity (vector search) and PostgreSQL BM25 full-text search, merged with Reciprocal Rank Fusion (RRF) plus a hot-score boost. Supports multi-topic parallel search, Redis result caching, and asynchronous access-count tracking.

## Requirements

### Requirement: 系统执行向量相似度搜索
The system SHALL 使用 pgvector cosine 相似度对查询文本生成 Embedding 后，在指定库/版本中召回最相关的 Top-50 Chunks，支持 mode 和 library_id 过滤。

#### Scenario: 指定库和版本的向量搜索
- **WHEN** library_id = 1，version = "18.3.0"，mode = "code"
- **THEN** 执行 `embedding <=> query_embedding ORDER BY ASC LIMIT 50` SQL，条件 `chunk_type IN ('code', 'mixed') AND status = 'active' AND deleted_at IS NULL AND library_id = 1 AND version = '18.3.0'`

#### Scenario: 全局向量搜索（library_id = 0）
- **WHEN** library_id = 0
- **THEN** 不加 library_id 过滤，跨所有库执行向量搜索

---

### Requirement: 系统执行 BM25 全文搜索
The system SHALL 使用 PostgreSQL tsvector（simple 配置）对查询词执行全文搜索，按 ts_rank 降序返回 Top-50 Chunks。

#### Scenario: 关键词全文搜索
- **WHEN** topic = "useCallback"
- **THEN** 执行 `chunk_tsvector_simple @@ plainto_tsquery('simple', ?)` SQL，按 ts_rank 降序返回 Top-50

#### Scenario: 查询词在全文索引中无命中
- **WHEN** topic 词在 tsvector 中无匹配
- **THEN** BM25 返回空列表，不报错，混合搜索仅依赖向量搜索结果

---

### Requirement: 系统用 RRF 算法合并向量和 BM25 结果
The system SHALL 将向量搜索和 BM25 搜索结果按排名用 RRF 公式合并（k=60），叠加热度加成，取 Top-10 分页返回。

#### Scenario: 两路搜索结果 RRF 合并
- **WHEN** 同一 Chunk 同时出现在向量搜索（rank=3）和 BM25（rank=5）结果中
- **THEN** 该 Chunk 的 RRF 分数 = `0.7/(3+60) + 0.3/(5+60) + 0.2 * hot_score`，hot_score = access_count / max_access_count_in_result
- **AND** 仅出现在向量搜索的 Chunk 只有向量 RRF 贡献（0.7 权重项），仅 BM25 同理

#### Scenario: mode 过滤
- **WHEN** mode = "code"
- **THEN** 向量搜索和 BM25 均加条件 `chunk_type IN ('code', 'mixed')`
- **WHEN** mode = "info"
- **THEN** 加条件 `chunk_type IN ('info', 'mixed')`

#### Scenario: 分页返回
- **WHEN** page = 1
- **THEN** 返回 RRF 排名前 10 条，has_more = true（若总结果 > 10）
- **WHEN** page = 2
- **THEN** 返回排名第 11-20 条

---

### Requirement: 系统支持多 topic 并行搜索并 RRF 合并
The system SHALL 支持 topic 参数为逗号分隔的多个词，对每个 topic 独立并行执行混合搜索，再次用 RRF 跨 topic 合并结果。

#### Scenario: 多 topic 并行搜索
- **WHEN** topic = "data fetching, caching"
- **THEN** 并行对 "data fetching" 和 "caching" 各执行一次完整的混合搜索（向量 + BM25 + RRF）
- **AND** 两路结果再次 RRF 合并，返回跨 topic 的最优 Top-10

#### Scenario: 单 topic 不走多路合并
- **WHEN** topic = "hooks"（无逗号）
- **THEN** 直接执行单路混合搜索，不走多 topic 逻辑

---

### Requirement: 系统对热门搜索结果进行 Redis 缓存
The system SHALL 将搜索结果缓存到 Redis，key 格式为 `search:topic:{library_id}:{version}:{mode}:{topic_md5}`，TTL 24 小时；文档更新时按 library_id + version 标签批量失效。

#### Scenario: 缓存命中返回
- **WHEN** 相同 library_id + version + mode + topic 组合再次被查询
- **THEN** 直接从 Redis 返回缓存结果，不执行 PostgreSQL 查询，响应时间 < 50ms

#### Scenario: 文档更新触发缓存失效
- **WHEN** 某库某版本的文档上传/删除/刷新完成
- **THEN** 删除所有 `search:topic:{library_id}:*` 前缀的 Redis key，确保下次搜索返回最新结果

---

### Requirement: 系统异步更新被搜索到的 Chunk 访问计数
The system SHALL 在每次搜索成功返回结果后，异步（goroutine）对返回的 Chunk 的 access_count 字段 +1，不阻塞搜索响应。

#### Scenario: 搜索后异步更新 access_count
- **WHEN** 搜索返回 10 条结果
- **THEN** 在后台 goroutine 中对这 10 条 Chunk 的 access_count 各自 +1
- **AND** 该更新不影响搜索接口的响应时间

#### Scenario: access_count 参与热度排序
- **WHEN** 同一 topic 的两个 Chunk 向量分数相近，Chunk A 被历史查询访问 100 次，Chunk B 访问 10 次
- **THEN** Chunk A 的 RRF 热度加成更高，最终排名靠前
