# Memory 模块详细设计

## 1. 模块概述

Memory 模块负责 ZeroClaw 的记忆存储和管理，支持多种后端实现 (SQLite/Markdown/Lucid/Postgres/Qdrant)，提供统一的记忆存取接口，并支持语义搜索和向量嵌入。

## 2. 架构设计

### 2.1 模块结构

```
src/memory/
├── mod.rs           # 模块导出和工厂 (19.9KB)
├── traits.rs        # Memory Trait 定义 (4.4KB)
├── backend.rs       # 后端抽象层 (5.2KB)
├── sqlite.rs        # SQLite 后端 (64.6KB - 最完整)
├── markdown.rs      # Markdown 文件后端 (10.6KB)
├── lucid.rs         # Lucid 云记忆 (19.5KB)
├── postgres.rs      # PostgreSQL 后端 (12.9KB)
├── qdrant.rs        # Qdrant 向量数据库 (18.8KB)
├── none.rs          # 无记忆模式 (2.0KB)
├── embeddings.rs    # 文本嵌入 (10.1KB)
├── vector.rs        # 向量搜索 (12.3KB)
├── chunker.rs       # 文本分块 (12.4KB)
├── hygiene.rs       # 记忆卫生管理 (15.8KB)
├── snapshot.rs      # 记忆快照 (14.7KB)
├── response_cache.rs# 响应缓存 (13.6KB)
└── cli.rs           # CLI 工具 (10.9KB)
```

### 2.2 设计模式

- **策略模式**: 每个后端实现是独立的策略
- **工厂模式**: 根据配置创建对应后端实例
- **装饰器模式**: 在基础后端上增加缓存、向量搜索等增强功能

## 3. 核心 Trait 定义 (traits.rs)

### 3.1 Memory Trait

```rust
#[async_trait]
pub trait Memory: Send + Sync {
    /// 后端名称
    fn name(&self) -> &str;

    /// 存储记忆
    async fn store(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
    ) -> Result<()>;

    /// 召回记忆 (关键词搜索)
    async fn recall(
        &self,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
    ) -> Result<Vec<MemoryEntry>>;

    /// 获取特定记忆
    async fn get(&self, key: &str) -> Result<Option<MemoryEntry>>;

    /// 列出记忆
    async fn list(
        &self,
        category: Option<&MemoryCategory>,
        session_id: Option<&str>,
    ) -> Result<Vec<MemoryEntry>>;

    /// 删除记忆
    async fn forget(&self, key: &str) -> Result<bool>;

    /// 计数
    async fn count(&self) -> Result<usize>;

    /// 健康检查
    async fn health_check(&self) -> bool;
}
```

### 3.2 数据结构

#### MemoryEntry

```rust
pub struct MemoryEntry {
    pub id: String,              // 唯一标识符
    pub key: String,             // 记忆键
    pub content: String,         // 记忆内容
    pub category: MemoryCategory,// 分类
    pub timestamp: String,       // ISO8601 时间戳
    pub session_id: Option<String>, // 会话 ID
    pub score: Option<f64>,      // 相关性评分 (召回时)
}
```

#### MemoryCategory 枚举

```rust
pub enum MemoryCategory {
    Core,           // 长期事实、偏好、决策
    Daily,          // 日常会话日志
    Conversation,   // 对话上下文
    Custom(String), // 用户自定义
}
```

## 4. 主要后端实现

### 4.1 SQLite Backend (sqlite.rs - 64.6KB)

**特点**: 默认后端，功能最完整

#### 4.1.1 表结构

```sql
CREATE TABLE memories (
    id TEXT PRIMARY KEY,
    key TEXT NOT NULL,
    content TEXT NOT NULL,
    category TEXT NOT NULL,
    timestamp TEXT NOT NULL,
    session_id TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    updated_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_memories_key ON memories(key);
CREATE INDEX idx_memories_category ON memories(category);
CREATE INDEX idx_memories_session ON memories(session_id);
```

#### 4.1.2 核心实现

**Store 操作**:
```rust
async fn store(
    &self,
    key: &str,
    content: &str,
    category: MemoryCategory,
    session_id: Option<&str>,
) -> Result<()> {
    let id = generate_id();
    let timestamp = Utc::now().to_rfc3339();
    
    self.pool.execute(
        "INSERT OR REPLACE INTO memories 
         (id, key, content, category, timestamp, session_id, updated_at)
         VALUES (?, ?, ?, ?, ?, ?, ?)",
        params![id, key, content, category.to_string(), timestamp, session_id, timestamp],
    ).await?;
    
    Ok(())
}
```

**Recall 操作 (关键词搜索)**:
```rust
async fn recall(
    &self,
    query: &str,
    limit: usize,
    session_id: Option<&str>,
) -> Result<Vec<MemoryEntry>> {
    let mut sql = String::from(
        "SELECT id, key, content, category, timestamp, session_id
         FROM memories
         WHERE content LIKE ?"
    );
    
    if let Some(_) = session_id {
        sql.push_str(" AND session_id = ?");
    }
    
    sql.push_str(" ORDER BY updated_at DESC LIMIT ?");
    
    let rows = if let Some(sid) = session_id {
        self.pool.fetch_all(
            &sql,
            params![format!("%{}%", query), sid, limit as i64],
        ).await?
    } else {
        self.pool.fetch_all(
            &sql,
            params![format!("%{}%", query), limit as i64],
        ).await?
    };
    
    Ok(rows.into_iter().map(|row| {
        MemoryEntry {
            id: row.get(0),
            key: row.get(1),
            content: row.get(2),
            category: MemoryCategory::from_str(&row.get::<String, _>(3)).unwrap(),
            timestamp: row.get(4),
            session_id: row.get(5),
            score: None,
        }
    }).collect())
}
```

### 4.2 Markdown Backend (markdown.rs)

**特点**: 基于文件的简单实现，适合调试和小规模使用

#### 4.2.1 目录结构

```
~/.zeroclaw/workspace/memory/
├── core/
│   ├── favorite_language.md
│   └── preferences.md
├── daily/
│   └── 2026-02-24.md
└── conversation/
    └── session_abc123.md
```

#### 4.2.2 文件格式

```markdown
---
key: favorite_language
category: core
timestamp: 2026-02-24T00:00:00Z
session_id: null
---

Rust is my favorite programming language.
```

### 4.3 Lucid Backend (lucid.rs)

**特点**: 云记忆服务，支持跨设备同步

#### 4.3.1 API 集成

```rust
pub struct LucidMemory {
    client: reqwest::Client,
    api_key: Secret<String>,
    base_url: String,
}

async fn store(&self, key: &str, content: &str, category: MemoryCategory, session_id: Option<&str>) -> Result<()> {
    let payload = serde_json::json!({
        "key": key,
        "content": content,
        "category": category.to_string(),
        "session_id": session_id,
    });
    
    let response = self.client
        .post(format!("{}/memories", self.base_url))
        .header("Authorization", format!("Bearer {}", self.api_key.expose()))
        .json(&payload)
        .send()
        .await?;
    
    if !response.status().is_success() {
        bail!("Lucid API error: {}", response.text().await?);
    }
    
    Ok(())
}
```

### 4.4 PostgreSQL Backend (postgres.rs)

**特点**: 生产级关系型数据库支持

#### 4.4.1 连接配置

```rust
use sqlx::PgPool;

pub struct PostgresMemory {
    pool: PgPool,
}

impl PostgresMemory {
    pub async fn new(connection_string: &str) -> Result<Self> {
        let pool = PgPool::connect(connection_string).await?;
        
        // 自动迁移
        sqlx::query(
            r#"
            CREATE TABLE IF NOT EXISTS memories (
                id TEXT PRIMARY KEY,
                key TEXT NOT NULL,
                content TEXT NOT NULL,
                category TEXT NOT NULL,
                timestamp TIMESTAMPTZ NOT NULL,
                session_id TEXT,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW()
            )
            "#
        ).execute(&pool).await?;
        
        Ok(Self { pool })
    }
}
```

### 4.5 Qdrant Backend (qdrant.rs)

**特点**: 向量数据库，支持语义搜索

#### 4.5.1 向量集成

```rust
use qdrant_client::{Qdrant, QdrantError};
use qdrant_client::qdrant::{PointStruct, Value, VectorParams};

pub struct QdrantMemory {
    client: Qdrant,
    embedding_model: Arc<dyn EmbeddingModel>,
    collection_name: String,
}

async fn store(&self, key: &str, content: &str, category: MemoryCategory, session_id: Option<&str>) -> Result<()> {
    // 生成嵌入向量
    let embedding = self.embedding_model.embed(content).await?;
    
    // 创建点
    let point = PointStruct {
        id: Some(key.into()),
        vector: embedding,
        payload: hashmap! {
            "content".to_string() => Value::from(content),
            "category".to_string() => Value::from(category.to_string()),
            "session_id".to_string() => Value::from(session_id.unwrap_or_default()),
            "timestamp".to_string() => Value::from(Utc::now().to_rfc3339()),
        },
    };
    
    // 上传到 Qdrant
    self.client
        .upsert_points(&self.collection_name, None, vec![point], None)
        .await?;
    
    Ok(())
}

async fn recall(&self, query: &str, limit: usize, session_id: Option<&str>) -> Result<Vec<MemoryEntry>> {
    // 查询嵌入
    let query_embedding = self.embedding_model.embed(query).await?;
    
    // 向量搜索
    let search_result = self.client
        .search_points(&SearchPoints {
            collection_name: self.collection_name.clone(),
            vector: query_embedding,
            limit: limit as u64,
            filter: session_id.map(|sid| Filter {
                must: vec![Condition {
                    key: "session_id".to_string(),
                    value: Some(ConditionValue::Keyword(sid)),
                }],
                ..Default::default()
            }),
            with_payload: Some(true.into()),
            ..Default::default()
        })
        .await?;
    
    // 转换为 MemoryEntry
    Ok(search_result.result.into_iter().map(|scored| {
        MemoryEntry {
            id: scored.id.unwrap().to_string(),
            key: scored.id.unwrap().to_string(),
            content: scored.payload.get("content").unwrap().as_str().unwrap().to_string(),
            category: MemoryCategory::from_str(scored.payload.get("category").unwrap().as_str().unwrap()).unwrap(),
            timestamp: scored.payload.get("timestamp").unwrap().as_str().unwrap().to_string(),
            session_id: Some(scored.payload.get("session_id").unwrap().as_str().unwrap().to_string()),
            score: Some(scored.score as f64),
        }
    }).collect())
}
```

## 5. 高级功能

### 5.1 嵌入模型 (embeddings.rs)

```rust
pub trait EmbeddingModel: Send + Sync {
    /// 将文本转换为向量
    async fn embed(&self, text: &str) -> Result<Vec<f32>>;
}

// 实现：使用本地模型或云服务
pub struct SentenceTransformersEmbedder {
    model: ort::Session,
}

pub struct OpenAIEmbedder {
    api_key: Secret<String>,
    client: reqwest::Client,
    model: String,  // e.g., "text-embedding-ada-002"
}
```

### 5.2 文本分块 (chunker.rs)

用于长文本处理:

```rust
pub struct TextChunker {
    max_chunk_size: usize,
    overlap: usize,
}

impl TextChunker {
    pub fn chunk(&self, text: &str) -> Vec<String> {
        // 按句子分割
        let sentences = split_sentences(text);
        
        let mut chunks = Vec::new();
        let mut current_chunk = String::new();
        
        for sentence in sentences {
            if current_chunk.len() + sentence.len() > self.max_chunk_size {
                chunks.push(current_chunk.clone());
                current_chunk = sentence.clone();
            } else {
                current_chunk.push_str(&sentence);
            }
        }
        
        if !current_chunk.is_empty() {
            chunks.push(current_chunk);
        }
        
        chunks
    }
}
```

### 5.3 记忆卫生 (hygiene.rs)

自动清理过期记忆:

```rust
pub struct MemoryHygiene {
    memory_backend: Arc<dyn Memory>,
    retention_days: u32,
}

impl MemoryHygiene {
    pub async fn cleanup_old_memories(&self) -> Result<usize> {
        let cutoff = Utc::now() - Duration::days(self.retention_days as i64);
        
        let entries = self.memory_backend.list(None, None).await?;
        let mut deleted_count = 0;
        
        for entry in entries {
            let timestamp = DateTime::parse_from_rfc3339(&entry.timestamp)?;
            if timestamp < cutoff {
                if self.memory_backend.forget(&entry.key).await? {
                    deleted_count += 1;
                }
            }
        }
        
        Ok(deleted_count)
    }
}
```

### 5.4 响应缓存 (response_cache.rs)

缓存常见查询结果:

```rust
pub struct ResponseCache {
    cache: DashMap<String, CacheEntry>,
    ttl: Duration,
}

struct CacheEntry {
    response: String,
    expires_at: SystemTime,
}

impl ResponseCache {
    pub fn get(&self, key: &str) -> Option<String> {
        if let Some(entry) = self.cache.get(key) {
            if entry.expires_at > SystemTime::now() {
                return Some(entry.response.clone());
            }
        }
        None
    }
    
    pub fn set(&self, key: String, response: String) {
        self.cache.insert(key, CacheEntry {
            response,
            expires_at: SystemTime::now() + self.ttl,
        });
    }
}
```

## 6. 记忆加载策略 (memory_loader.rs)

### 6.1 相关性计算

```rust
pub struct MemoryLoader {
    backend: Arc<dyn Memory>,
    relevance_threshold: f64,
    max_memories: usize,
}

impl MemoryLoader {
    pub async fn load_relevant_memories(
        &self,
        query: &str,
        session_id: Option<&str>,
    ) -> Result<Vec<MemoryEntry>> {
        let mut memories = self.backend.recall(query, self.max_memories, session_id).await?;
        
        // 过滤低相关性
        memories.retain(|m| {
            m.score.unwrap_or(1.0) >= self.relevance_threshold
        });
        
        // 按相关性排序
        memories.sort_by(|a, b| {
            b.score.partial_cmp(&a.score).unwrap()
        });
        
        Ok(memories)
    }
}
```

## 7. CLI 命令 (cli.rs)

```bash
# 查看记忆统计
zeroclaw memory stats

# 列出记忆
zeroclaw memory list --category core --limit 10

# 获取特定记忆
zeroclaw memory get favorite_language

# 清除记忆
zeroclaw memory clear --category conversation --yes
```

## 8. 配置选项

```toml
[memory]
backend = "sqlite"  # sqlite, markdown, lucid, postgres, qdrant, none

[memory.sqlite]
path = "~/.zeroclaw/workspace/memory.db"

[memory.lucid]
api_key = "${LUCID_API_KEY}"

[memory.postgres]
connection_string = "${DATABASE_URL}"

[memory.qdrant]
url = "http://localhost:6333"
collection = "zeroclaw_memories"
embedding_model = "sentence-transformers/all-MiniLM-L6-v2"

[memory.hygiene]
retention_days = 30
auto_cleanup = true

[memory.cache]
enabled = true
ttl_seconds = 3600
```

## 9. 性能优化

### 9.1 连接池

SQLite/Postgres 使用连接池提高并发性能。

### 9.2 批量操作

```rust
pub async fn batch_store(&self, entries: &[MemoryEntry]) -> Result<()> {
    let mut tx = self.pool.begin().await?;
    
    for entry in entries {
        sqlx::query("INSERT INTO memories ...")
            .bind(&entry.id)
            .bind(&entry.key)
            // ...
            .execute(&mut *tx)
            .await?;
    }
    
    tx.commit().await?;
    Ok(())
}
```

### 9.3 索引优化

为常用查询字段创建索引:
- key (精确查找)
- category (过滤)
- session_id (会话隔离)
- timestamp (时间排序)

## 10. 错误处理

### 10.1 错误类型

```rust
pub enum MemoryError {
    NotFound(String),
    AlreadyExists(String),
    InvalidCategory(String),
    BackendError(anyhow::Error),
    EmbeddingError(anyhow::Error),
}
```

### 10.2 恢复机制

- 连接失败自动重试
- 事务回滚
- 降级到备份后端

## 11. 测试策略

### 11.1 单元测试

```rust
#[tokio::test]
async fn test_sqlite_memory_store_and_recall() {
    let memory = SqliteMemory::new_in_memory().await.unwrap();
    
    memory.store("test_key", "test content", MemoryCategory::Core, None).await.unwrap();
    
    let entry = memory.get("test_key").await.unwrap().unwrap();
    assert_eq!(entry.content, "test content");
}
```

### 11.2 集成测试

测试不同后端的完整 CRUD 操作。

## 12. 扩展指南

### 12.1 添加新后端

1. 创建新文件 `src/memory/mybackend.rs`
2. 实现 `Memory` trait
3. 在 `mod.rs` 中注册到 factory
4. 添加配置 schema
5. 编写测试

### 12.2 最小实现示例

```rust
use crate::memory::traits::{Memory, MemoryCategory, MemoryEntry};

pub struct MyMemoryBackend {
    // 后端特定字段
}

#[async_trait]
impl Memory for MyMemoryBackend {
    fn name(&self) -> &str { "mybackend" }

    async fn store(&self, key: &str, content: &str, category: MemoryCategory, session_id: Option<&str>) -> Result<()> {
        // 实现存储逻辑
        Ok(())
    }

    async fn recall(&self, query: &str, limit: usize, session_id: Option<&str>) -> Result<Vec<MemoryEntry>> {
        // 实现召回逻辑
        Ok(vec![])
    }

    // ... 其他方法
}
```

## 13. 监控指标

### 13.1 关键指标

- 存储延迟 (p50, p95, p99)
- 召回延迟
- 命中率 (cache hit rate)
- 存储使用量

### 13.2 日志记录

```rust
tracing::info!(
    key = %key,
    category = %category.to_string(),
    "Memory stored successfully"
);

tracing::error!(
    key = %key,
    error = %e,
    "Failed to store memory"
);
```
