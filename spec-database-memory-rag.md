# Spec: Hybrid Memory & RAG Database Architecture

## Overview

This spec describes a hybrid persistence strategy for AI-powered applications that combines:
- **Redis** — fast, ephemeral session context cache (short-term memory)
- **PostgreSQL** — durable conversation and session storage (long-term memory)
- **pgvector** — semantic vector search for Retrieval-Augmented Generation (RAG)

This architecture separates concerns between speed (Redis), durability (PostgreSQL), and semantic retrieval (pgvector), while maintaining graceful degradation if any backend is unavailable.

---

## Architecture Diagram

```
User Request
     │
     ▼
┌────────────────┐     fast read (< 5ms)     ┌──────────────┐
│  Application   │ ─────────────────────────► │  Redis       │
│  (AI Pipeline) │ ◄─────────────────────────  │  (context)   │
└───────┬────────┘     recent N turns          └──────────────┘
        │
        │ async write (non-blocking)
        ▼
┌────────────────────┐
│  PostgreSQL        │
│  - sessions        │  ◄── permanent history
│  - conversations   │
│  - docs (RAG meta) │
│  - chunks + embed  │  ◄── vector search (pgvector)
└────────────────────┘
```

---

## PostgreSQL Schema

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE SCHEMA IF NOT EXISTS app;
```

### Core Tables

```sql
-- Active sessions
CREATE TABLE app.sessions (
    session_id   TEXT PRIMARY KEY,
    domain       TEXT,
    language     TEXT DEFAULT 'en',
    provider     TEXT,
    model        TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    last_active  TIMESTAMPTZ DEFAULT NOW()
);

-- Full conversation history
CREATE TABLE app.conversations (
    id              BIGSERIAL PRIMARY KEY,
    session_id      TEXT REFERENCES app.sessions(session_id),
    question        TEXT NOT NULL,
    answer          TEXT NOT NULL,
    domain          TEXT,
    category        TEXT,
    provider        TEXT,
    model           TEXT,
    score           FLOAT,          -- Assessor score
    refine_rounds   INT DEFAULT 0,
    language        TEXT,
    duration_s      FLOAT,          -- Total pipeline duration
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- RAG: document metadata
CREATE TABLE app.docs (
    id          BIGSERIAL PRIMARY KEY,
    doc_key     TEXT UNIQUE NOT NULL,   -- Stable identifier (slug or path)
    category    TEXT,
    title       TEXT,
    doc_type    TEXT,                   -- "article", "pdf", "web", "manual"
    tags        TEXT[],
    file_path   TEXT,
    source_url  TEXT,
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- RAG: text chunks with embeddings
CREATE TABLE app.chunks (
    id          BIGSERIAL PRIMARY KEY,
    doc_key     TEXT REFERENCES app.docs(doc_key) ON DELETE CASCADE,
    chunk_id    TEXT NOT NULL,
    content     TEXT NOT NULL,
    embedding   vector(1024),           -- Adjust to your embedding model dimension
    UNIQUE(doc_key, chunk_id)
);

-- RAG: embedding control (incremental updates)
CREATE TABLE app.embed_control (
    doc_key      TEXT PRIMARY KEY,
    content_hash TEXT,                  -- MD5 of doc content at embed time
    embedded_at  TIMESTAMPTZ DEFAULT NOW(),
    model_name   TEXT,
    embed_dim    INT
);
```

### Indices

```sql
-- Session lookups
CREATE INDEX idx_sessions_last_active ON app.sessions(last_active DESC);

-- Conversation queries
CREATE INDEX idx_conversations_session ON app.conversations(session_id, created_at DESC);
CREATE INDEX idx_conversations_domain  ON app.conversations(domain, created_at DESC);

-- RAG: vector similarity search (IVFFlat for large datasets)
CREATE INDEX idx_chunks_embedding ON app.chunks
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 200);   -- lists ≈ sqrt(total_rows); tune after ingestion

-- RAG: doc category filter
CREATE INDEX idx_chunks_doc_key ON app.chunks(doc_key);
CREATE INDEX idx_docs_category  ON app.docs(category);
```

**Note on IVFFlat**: Build the index only after inserting a representative dataset. Running `ANALYZE app.chunks;` after bulk ingestion helps the planner use the index correctly.

---

## Redis Schema

Redis stores per-session data as lightweight JSON with a TTL.

### Key Patterns

| Key | Type | TTL | Content |
|-----|------|-----|---------|
| `session:{session_id}` | Hash or JSON string | 30 min | Session metadata (domain, language, provider) |
| `history:{session_id}` | List (LIFO) | 30 min | Last N conversation turns |

### Turn Format (stored per list item)

```json
{
  "role": "user",
  "content": "What is a binary tree?",
  "ts": "2024-01-15T10:23:45Z"
}
```

Store as pairs: push user turn, then assistant turn. Keep list trimmed to `2 * MAX_TURNS`.

---

## Memory Service Interface

### Core Methods

```python
class MemoryService:

    async def create_session(
        self,
        session_id: str,
        domain: str,
        language: str,
        provider: str,
        model: str
    ) -> None:
        # Write to Redis (JSON with TTL) + PostgreSQL (permanent)

    async def save_turn(
        self,
        session_id: str,
        question: str,
        answer: str,
        metadata: dict
    ) -> None:
        # Push to Redis list (trim to MAX_TURNS)
        # Insert into app.conversations (async, non-blocking)

    async def get_context(
        self,
        session_id: str,
        max_turns: int = 5
    ) -> list[dict]:
        # Try Redis first (fast)
        # Fall back to PostgreSQL query if Redis miss
        # Return list of {role, content} dicts

    async def build_context_prompt(
        self,
        session_id: str
    ) -> str:
        # Get recent turns, format as:
        # User: ...
        # Assistant: ...
        # (ready to inject into LLM system prompt)
```

### Graceful Degradation

```python
async def get_context(self, session_id, max_turns=5):
    try:
        # Primary: Redis
        turns = await self.redis.lrange(f"history:{session_id}", 0, max_turns * 2)
        if turns:
            return [json.loads(t) for t in turns]
    except RedisConnectionError:
        pass  # Redis unavailable

    try:
        # Fallback: PostgreSQL
        rows = await self.db.fetch(
            "SELECT question, answer FROM app.conversations "
            "WHERE session_id = $1 ORDER BY created_at DESC LIMIT $2",
            session_id, max_turns
        )
        return [{"role": "user", "content": r.question} for r in rows]
    except DatabaseError:
        pass  # DB unavailable too

    return []  # Graceful degradation: no context, pipeline still works
```

---

## RAG: Retrieval-Augmented Generation

### Document Ingestion Pipeline

```
Source File (PDF / text / web)
        │
        ▼
    Chunker
    (split by paragraph, N tokens, or semantic boundary)
        │
        ▼
   Embedder
   (call embedding model API or local model)
        │
        ▼
   Upsert into app.chunks
   Update app.embed_control with content_hash
```

### Chunking Strategy

```python
def chunk_document(text: str, chunk_size: int = 512, overlap: int = 64) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks
```

Adjust chunk size based on embedding model's context window. Typical: 256–1024 tokens.

### Incremental Embedding (skip unchanged docs)

```python
import hashlib

async def should_re_embed(doc_key: str, content: str) -> bool:
    new_hash = hashlib.md5(content.encode()).hexdigest()
    row = await db.fetchrow(
        "SELECT content_hash FROM app.embed_control WHERE doc_key = $1", doc_key
    )
    return row is None or row["content_hash"] != new_hash
```

### Semantic Search Query

```python
async def semantic_search(
    query: str,
    category: str | None = None,
    top_k: int = 5,
    min_similarity: float = 0.7
) -> list[dict]:
    query_embedding = await embed(query)   # vector from embedding model

    sql = """
        SELECT
            c.content,
            d.title,
            d.doc_key,
            1 - (c.embedding <=> $1::vector) AS similarity
        FROM app.chunks c
        JOIN app.docs d ON c.doc_key = d.doc_key
        {where_clause}
        ORDER BY c.embedding <=> $1::vector
        LIMIT $2
    """

    where = "WHERE d.category = $3 AND" if category else "WHERE"
    where += f" 1 - (c.embedding <=> $1::vector) >= {min_similarity}"

    params = [query_embedding, top_k]
    if category:
        params.append(category)

    rows = await db.fetch(sql.format(where_clause=where), *params)
    return [{"content": r["content"], "title": r["title"], "score": r["similarity"]}
            for r in rows]
```

### Injecting RAG Context into Agent Prompt

```python
async def build_rag_prompt(question: str, category: str) -> str:
    results = await semantic_search(question, category=category, top_k=5)

    if not results:
        return ""

    context_parts = []
    for r in results:
        context_parts.append(f"[Source: {r['title']} | Similarity: {r['score']:.2f}]\n{r['content']}")

    return "Relevant context from knowledge base:\n\n" + "\n\n---\n\n".join(context_parts)
```

---

## Docker Compose Setup

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./pgvector/init:/docker-entrypoint-initdb.d  # runs init.sql on first start
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

---

## Environment Variables

```env
# PostgreSQL
DATABASE_URL=postgresql://appuser:apppass@localhost:5432/appdb
DB_POOL_MIN=2
DB_POOL_MAX=10

# Redis
REDIS_URL=redis://localhost:6379/0
REDIS_SESSION_TTL=1800       # seconds (30 min)
REDIS_MAX_TURNS=5            # conversation turns to cache

# RAG
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIM=1024
RAG_TOP_K=5
RAG_MIN_SIMILARITY=0.70
CHUNK_SIZE=512
CHUNK_OVERLAP=64
```

---

## Connection Lifecycle

### FastAPI / Async Application

```python
from contextlib import asynccontextmanager
import asyncpg, redis.asyncio as aioredis

@asynccontextmanager
async def lifespan(app):
    # Startup
    app.state.db_pool = await asyncpg.create_pool(DATABASE_URL, min_size=2, max_size=10)
    app.state.redis   = aioredis.from_url(REDIS_URL, decode_responses=True)

    yield

    # Shutdown
    await app.state.db_pool.close()
    await app.state.redis.close()
```

---

## Operational Notes

### pgvector Index Maintenance

- After bulk ingestion, run `VACUUM ANALYZE app.chunks;`
- IVFFlat index accuracy degrades if data grows significantly past `lists * 40` rows — rebuild index
- For smaller datasets (< 100k rows), use `hnsw` instead of `ivfflat` (better recall, slower build):
  ```sql
  CREATE INDEX idx_chunks_hnsw ON app.chunks
      USING hnsw (embedding vector_cosine_ops)
      WITH (m = 16, ef_construction = 64);
  ```

### Redis TTL Strategy

- Set TTL on `session:*` and `history:*` keys to the expected session idle timeout (e.g., 30 min)
- Refresh TTL on every request so active sessions don't expire mid-conversation
- Use `allkeys-lru` eviction so Redis never fills up silently

### Session Recovery

If a session's Redis cache expires, the application should:
1. Detect cache miss
2. Re-fetch last N turns from PostgreSQL
3. Repopulate Redis with those turns + fresh TTL
4. Continue normally (user never notices)

---

## Implementation Checklist

- [ ] Enable `pgvector` extension in PostgreSQL init script
- [ ] Create `app` schema with sessions, conversations, docs, chunks, embed_control tables
- [ ] Build indices: session lookups, conversation queries, IVFFlat or HNSW for vectors
- [ ] Implement `MemoryService` with Redis-first, PostgreSQL-fallback pattern
- [ ] Implement graceful degradation in `get_context()` (catch connection errors)
- [ ] Build document ingestion pipeline: chunker → embedder → upsert
- [ ] Implement `should_re_embed()` to skip unchanged documents
- [ ] Implement `semantic_search()` with cosine similarity threshold
- [ ] Build `build_rag_prompt()` to format retrieved chunks for LLM injection
- [ ] Wire RAG context into Generator Agent prompt (when relevant)
- [ ] Configure Docker Compose with healthchecks for both services
- [ ] Set `REDIS_MAX_TURNS`, `RAG_TOP_K`, `CHUNK_SIZE` as environment variables
- [ ] Implement session recovery from PostgreSQL on Redis cache miss
