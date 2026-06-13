## ADDED Requirements

### Requirement: 系统对上传文档执行预处理后再分块
The system SHALL 在文本提取完成后，先执行 Markdown 预处理（去除徽章、HTML 注释、独立图片、连续空行、水平分隔线），再进入分块阶段。

#### Scenario: 移除无效 Markdown 元素
- **WHEN** 上传含有 `[![CI](badge-url)](link)` 徽章、`<!-- comment -->` HTML 注释、`---` 分隔线的 Markdown 文件
- **THEN** 预处理后这些元素被移除，不出现在任何 Chunk 中
- **AND** 正文内容、代码块、标题层级完整保留

---

### Requirement: 系统按 Markdown 语义边界（标题层级）进行文档分块
The system SHALL 以 Markdown 标题（h1-h6）为主要切割边界，每个 section 携带其所有上级标题路径；section 超过 512 tokens（tiktoken cl100k_base）时，按段落和代码块原子单元继续切分，代码块保持完整不跨块。

#### Scenario: 标题层级正常切分
- **WHEN** Markdown 文档包含 h2 "Installation"（500 tokens）和 h2 "Usage"（300 tokens）两个 section
- **THEN** 生成 2 个 Chunk，每个 Chunk 的 title 包含对应的标题路径（如 "Installation" 或 "Getting Started > Usage"）

#### Scenario: Section 超过 512 tokens 时继续切分
- **WHEN** 某 h2 section 总计 1200 tokens
- **THEN** 按段落和代码块边界切分为多个子 Chunk，每个 Chunk ≤ 562 tokens（512 + 少量宽余）
- **AND** 代码块（``` 包裹）作为原子单元，不在代码块内部切断

#### Scenario: 代码块内标题不触发切割
- **WHEN** Markdown 代码块内含有以 `#` 开头的注释行
- **THEN** 该行不被识别为标题，不触发 section 切割

#### Scenario: 文档无标题结构
- **WHEN** 文档为纯文本，不含任何 Markdown 标题
- **THEN** 整个文档作为单一 section 处理，再按段落原子切分到 512 tokens

#### Scenario: 极短文档（< 50 tokens）
- **WHEN** 文档提取文本总量 < 50 tokens
- **THEN** 生成 1 个 Chunk，包含全部内容，不进行分割

---

### Requirement: 系统对每个 Chunk 进行类型检测并分别处理
The system SHALL 根据 Chunk 文本是否包含 ``` 代码块，将其分类为 `code` 或 `info` 类型，并据此决定 LLM Enrich 策略。

#### Scenario: code 类型检测
- **WHEN** Chunk 文本包含 ``` 包裹的代码块
- **THEN** chunk_type = "code"，language 字段从代码块语言标识（如 `js`、`go`）提取，code 字段存储第一个代码块的内容

#### Scenario: info 类型 Chunk 保留标题路径
- **WHEN** Chunk 文本无任何代码块
- **THEN** chunk_type = "info"，title = 标题层级路径（如 "API Reference > Hooks"），不调用 LLM

---

### Requirement: 系统对 code 类型 Chunk 执行 LLM Enrich（并发 5 个 worker）
The system SHALL 对所有 code 类型 Chunk 调用 gpt-4o-mini，生成英文 title（动词短语）和 description（1-3 句），使用 5 个并发 worker 加速，失败时使用 fallback 值。

#### Scenario: code Chunk 正常 Enrich
- **WHEN** code Chunk 进入 Enrich 阶段
- **THEN** gpt-4o-mini 返回 JSON：`{"title": "Create a Custom React Hook", "description": "This example demonstrates..."}`
- **AND** Chunk 的 title 和 description 字段更新为 LLM 输出

#### Scenario: LLM 调用失败的 fallback
- **WHEN** gpt-4o-mini 调用超时或返回错误
- **THEN** title 保留 Chunk 的标题层级路径，description 保留为空，Chunk 仍正常进入 Embedding 阶段

#### Scenario: info Chunk 不进入 LLM Enrich
- **WHEN** info 类型 Chunk 进入处理流水线
- **THEN** 直接跳过 LLM 调用，保留标题路径作为 title，进入 Embedding 阶段

---

### Requirement: 系统对每个 Chunk 生成 Embedding 向量并存入 pgvector
The system SHALL 使用 OpenAI text-embedding-3-small（1536 维）对 Chunk 文本批量生成向量，支持 Redis 缓存和指数退避重试。

#### Scenario: 批量 Embedding 生成
- **WHEN** 分块和 Enrich 完成后进入 Embedding 阶段
- **THEN** 以批次（最多 100 条/批）调用 OpenAI Embedding API，生成 1536 维 float32 向量，存入 document_chunks.embedding

#### Scenario: Embedding 结果 Redis 缓存命中
- **WHEN** 相同文本在 Redis 中已有缓存向量（key = text 的 hash）
- **THEN** 直接使用缓存，不重复调用 OpenAI API

#### Scenario: API 限流（429）指数退避重试
- **WHEN** OpenAI API 返回 429
- **THEN** 指数退避重试最多 3 次（间隔 1s/2s/4s）
- **AND** 超出重试次数后该批次 Chunks status = "failed"，记录 error_message

---

### Requirement: 系统支持文档上传内容 Hash 去重和状态跟踪
The system SHALL 在上传时计算文件 SHA256 Hash，若与已有记录相同则跳过处理；文档处理全程状态记录在 document_uploads 表（pending → processing → completed / failed）。

#### Scenario: 相同内容文档跳过处理
- **WHEN** 上传文件的 SHA256 Hash 与 document_uploads.content_hash 已有记录匹配
- **THEN** 跳过解析/分块/Embed 流程，SSE 推送 skip 事件，message = "document unchanged"

#### Scenario: 部分文档处理失败不阻塞其他文档
- **WHEN** 批量上传 3 个文件，其中 1 个 PDF 损坏
- **THEN** 损坏文件 status = "failed"，error_message 记录原因；其余 2 个文件继续处理，SSE 分别展示进度

---

### Requirement: 系统支持 SSE 实时推送文档处理进度
The system SHALL 在文档上传处理期间，通过 SSE 流实时推送每个处理阶段的进度事件。

#### Scenario: SSE 进度事件序列
- **WHEN** 单文档 SSE 上传（POST `/api/v1/documents/upload-sse`）
- **THEN** SSE 流依次推送：upload（5%）→ preprocessing（10%）→ chunking（20%）→ enriching（35%）→ embedding（60%）→ saving（85%）→ completed（100%）
- **AND** 每个事件包含 stage、progress（百分比）、message、status 字段
