# Multi-User Medical RAG Assistant
# Production-oriented RAG system for secure multi-user medical document querying.

## Workflow Overview


**Telegram · n8n · Supabase (pgvector) · OpenAI**

---

## Overview

This project implements a **multi-user Retrieval-Augmented Generation (RAG) system** for querying medical PDF documents through Telegram.

Users can:

- Upload medical reports (PDF)
- Ask natural language questions about their documents
- Receive structured, safety-constrained summaries

The system is built as a unified workflow in **n8n**, integrating:

- Conditional OCR for scanned PDFs
- PostgreSQL-native vector search (pgvector)
- Metadata-based multi-user isolation
- Tool-based RAG agent with medical safety constraints

---

## Architecture

```
User (Telegram)
      │
      ▼
Telegram Bot
      │
      ▼
n8n Unified Workflow
      │
      ├── Document Ingestion
      │     ├─ PDF Text Extraction
      │     ├─ Conditional OCR (if low text density)
      │     ├─ Chunking (1000 / 200 overlap)
      │     ├─ OpenAI Embeddings
      │     └─ Supabase Vector Store (pgvector)
      │
      └── User Query
            ├─ Embedding generation
            ├─ Vector similarity search (filtered by user_id)
            └─ AI Agent (GPT-4.1-mini)
                    ↓
              Structured medical summary
```

---

## Key Features

- **Multi-user document isolation**
  Each document chunk is stored with `metadata.user_id`, ensuring per-user retrieval filtering.

- **Conditional OCR pipeline**
  If extracted text length is below a threshold, the system automatically routes the document to OCR.

- **PostgreSQL-native vector search (pgvector)**
  No external vector database required.

- **Tool-based RAG integration**
  Supabase vector store is exposed as a tool to the AI Agent rather than hardcoded retrieval.

- **Medical safety constraints**
  The agent:
  - Never provides diagnoses
  - Never recommends treatments
  - Always includes a disclaimer
  - Formats lab values with unit + reference range

---

## Tech Stack

| Layer         | Technology                      |
| ------------- | ------------------------------- |
| Interface     | Telegram Bot API                |
| Orchestration | n8n Cloud                       |
| Database      | Supabase (PostgreSQL)           |
| Vector Engine | pgvector                        |
| Embeddings    | OpenAI `text-embedding-3-small` |
| LLM           | OpenAI `gpt-4.1-mini`           |
| OCR           | Mistral OCR API                 |

---

## Database Schema

### Enable pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Documents Table

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    embedding VECTOR(1536),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

CREATE INDEX idx_documents_user_id
ON documents ((metadata->>'user_id'));
```

### RAG Search Function

```sql
CREATE OR REPLACE FUNCTION match_documents(
    query_embedding VECTOR(1536),
    match_count INT DEFAULT 5,
    filter JSONB DEFAULT '{}'
)
RETURNS TABLE (
    id BIGINT,
    content TEXT,
    metadata JSONB,
    similarity FLOAT
)
LANGUAGE plpgsql
AS $$
DECLARE
    filter_user_id TEXT;
BEGIN
    filter_user_id := filter->>'user_id';

    RETURN QUERY
    SELECT
        d.id,
        d.content,
        d.metadata,
        1 - (d.embedding <=> query_embedding) AS similarity
    FROM documents d
    WHERE
        (filter_user_id IS NULL OR d.metadata->>'user_id' = filter_user_id)
    ORDER BY d.embedding <=> query_embedding
    LIMIT match_count;
END;
$$;
```

---

## Workflow Logic

### 1️⃣ Document Ingestion

- Download PDF from Telegram
- Extract text
- If text length < threshold → OCR
- Chunk (1000 characters, 200 overlap)
- Generate embeddings
- Store in Supabase with `user_id` metadata

---

### 2️⃣ Retrieval Layer

- Convert user question to embedding
- Query pgvector via `match_documents`
- Filter by `user_id`
- Return top-k relevant chunks

---

### 3️⃣ Generation Layer

The AI Agent:

- Uses only retrieved document context
- Formats medical values clearly
- Applies strict non-diagnostic policy
- Provides structured summary for informational purposes only

---

## Design Decisions

### Why pgvector instead of Pinecone?

- Reduced infrastructure complexity
- Lower operational cost
- PostgreSQL-native filtering
- Simpler deployment model

---

### Why metadata-based multi-user isolation?

- Avoids separate tables per user
- Scales cleanly
- Keeps schema simple

---

### Why chunk size 1000 / overlap 200?

- Preserves medical section context
- Improves retrieval precision
- Reduces semantic fragmentation

---

### Why GPT-4.1-mini?

- Cost-efficient
- Sufficient for structured medical summarization
- Lower latency for conversational UX

---

## Safety Notice

This system is designed for informational purposes only.

It does **not** provide medical diagnosis, treatment recommendations, or clinical interpretation.
Final medical decisions must always be made by a licensed physician.

---

## Future Improvements

- Structured lab value extraction (NER-based parsing)
- Voice message support (Whisper integration)
- Encrypted document storage layer
- GDPR/HIPAA compliance hardening
- Advanced trend analysis across reports

---

## Author

Fabio Roggero
AI & Workflow Automation Projects
