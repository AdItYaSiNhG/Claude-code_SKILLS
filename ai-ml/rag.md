---
name: ai-ml-rag
description: Retrieval-Augmented Generation (RAG) pipeline design, implementation, and tuning. Covers chunking strategies, embedding models, vector stores (pgvector, Pinecone, Weaviate, Qdrant), hybrid search, re-ranking, and evaluation.
---

The user is building or improving a RAG system. Apply the pipeline framework below.

## RAG Architecture Tiers

```
Basic RAG        Embed → Store → Retrieve → Generate
Advanced RAG     + Query rewriting + Hybrid search + Re-ranking
Agentic RAG      + Multi-step retrieval + Tool use + Self-correction
```

Start with Basic, move to Advanced when recall < 80% or precision < 70%.

## Document Processing

### Chunking Strategy

| Content type | Strategy | Chunk size |
|---|---|---|
| Prose / documentation | Recursive character split | 512 tokens, 50 overlap |
| Code | AST-based (split at function boundaries) | Full function |
| Tables | Row-level, preserve headers in every chunk | Variable |
| PDFs with mixed content | Layout-aware (pypdf2 + pymupdf) | Section-level |
| Long legal / contracts | Semantic splitting (sentence embeddings) | Paragraph |

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " ", ""],
)
chunks = splitter.split_text(document.text)
```

### Metadata enrichment (critical for filtering)
```python
chunk_metadata = {
    "source": doc.url,
    "section": doc.section_title,
    "document_type": "policy",         # enables filtered retrieval
    "department": "legal",
    "last_updated": doc.updated_at,
    "chunk_index": i,
    "total_chunks": len(chunks),
}
```

## Embedding Models

| Model | Dims | Best for |
|---|---|---|
| `text-embedding-3-small` | 1536 | General, cost-efficient |
| `text-embedding-3-large` | 3072 | Higher accuracy |
| `voyage-3` | 1024 | Code and technical docs |
| `nomic-embed-text-v1` | 768 | Self-hosted, open-source |
| `cohere-embed-v3-multilingual` | 1024 | Multi-language corpora |

Always normalise vectors before storage (`L2 normalisation` if cosine similarity).

## Vector Store Selection

| Store | Best for |
|---|---|
| pgvector | Already on Postgres, <10M vectors |
| Qdrant | High performance, self-hosted, filtering |
| Pinecone | Managed, serverless, rapid prototyping |
| Weaviate | Multi-modal, graph-linked retrieval |
| Chroma | Local development, small datasets |

### pgvector setup
```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE document_chunks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content     TEXT NOT NULL,
    embedding   vector(1536),
    metadata    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
-- HNSW index (ANN, fast query)
CREATE INDEX ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

## Retrieval

### Hybrid Search (BM25 + Dense — recommended default)
```python
# Dense retrieval (semantic)
dense_results = vector_store.similarity_search(query, k=20)

# Sparse retrieval (keyword, BM25)
sparse_results = bm25_index.search(query, k=20)

# Reciprocal Rank Fusion (merge without score normalisation)
from rank_bm25 import BM25Okapi
def rrf(rankings: list[list], k=60):
    scores = defaultdict(float)
    for ranking in rankings:
        for rank, doc in enumerate(ranking):
            scores[doc.id] += 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

### Query rewriting
```python
REWRITE_PROMPT = """
Given a user question, generate 3 alternative phrasings that would help
retrieve relevant documents. Return as a JSON array of strings.

Question: {question}
"""
# Retrieve for all 3 → merge → re-rank
```

### Re-ranking (always apply before final context window)
```python
from cohere import Client
co = Client(api_key=...)

reranked = co.rerank(
    model="rerank-v3.5",
    query=user_query,
    documents=[c.content for c in candidates],
    top_n=5,
)
final_context = [candidates[r.index] for r in reranked.results]
```

## Context Assembly

```python
SYSTEM_PROMPT = """
You are a helpful assistant. Answer based solely on the provided context.
If the context does not contain the answer, say "I don't have that information."
Never speculate or use knowledge outside the context.
"""

context_block = "\n\n---\n\n".join([
    f"Source: {c.metadata['source']}\n{c.content}"
    for c in final_context
])

messages = [
    {"role": "user", "content": f"Context:\n{context_block}\n\nQuestion: {user_query}"}
]
```

Always include source attribution in the response for auditability.

## RAG Evaluation

### Metrics (run with RAGAS or custom)
```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,           # Is the answer grounded in retrieved context?
    answer_relevancy,       # Is the answer relevant to the question?
    context_recall,         # Did retrieval find the right chunks?
    context_precision,      # Are retrieved chunks actually useful?
)

results = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall, context_precision])
```

### Target thresholds (enterprise production)
| Metric | Minimum | Target |
|---|---|---|
| Faithfulness | 0.85 | >0.95 |
| Answer relevancy | 0.80 | >0.90 |
| Context recall | 0.75 | >0.85 |
| Context precision | 0.70 | >0.80 |

### Common failure modes and fixes
| Symptom | Likely cause | Fix |
|---|---|---|
| Hallucinated facts | Low faithfulness | Stronger system prompt + output guardrail |
| "I don't know" on valid questions | Low context recall | Increase k, add hybrid search |
| Wrong document retrieved | Poor chunking | Switch to semantic chunking |
| Slow retrieval | No HNSW index | Add `hnsw` or `ivfflat` index |
| Too much noise in context | No re-ranking | Add Cohere/Jina re-ranker |

## Output Format

1. Start with the pipeline diagram (Basic/Advanced/Agentic)
2. Show the chunking and embedding config for the specific content type
3. Show the retrieval + re-ranking code
4. Provide the eval script with expected thresholds
