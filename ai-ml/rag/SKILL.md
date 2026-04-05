---
name: rag
description: >
  Use this skill for RAG (Retrieval Augmented Generation) pipeline tasks: embedding
  generation, vector store integration (Pinecone/Weaviate/pgvector/Chroma), chunking
  strategies, retrieval ranking, hybrid search (BM25 + dense), re-ranking, context
  window management, RAG evaluation, and production RAG architecture.
  Triggers: "RAG", "retrieval", "vector store", "embeddings", "Pinecone", "pgvector",
  "Weaviate", "Chroma", "Qdrant", "semantic search", "hybrid search", "chunking",
  "re-rank", "knowledge base", "document Q&A", "LlamaIndex", "LangChain retrieval".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# RAG Pipeline Skill

## Why this skill exists

RAG systems built naively — naive chunking, no re-ranking, no evaluation —
produce poor retrieval quality that doesn't improve no matter how good the LLM is.
Retrieval quality is almost always the bottleneck, not the generator. This skill
provides **production RAG architecture** with proper chunking, hybrid search,
re-ranking, and systematic evaluation.

---

## 0. RAG stack audit

```bash
# Detect vector store and embedding libraries
cat requirements.txt package.json 2>/dev/null | python3 - <<'EOF'
import sys
text = sys.stdin.read()
libs = {
    "Vector stores": ["pinecone","weaviate","qdrant","chroma","pgvector","milvus","lancedb"],
    "Embeddings":    ["sentence-transformers","openai","cohere","voyageai","fastembed"],
    "Frameworks":    ["langchain","llama-index","llamaindex","haystack"],
    "Reranking":     ["cohere","flashrank","cross-encoder","rerankers"],
}
for category, items in libs.items():
    found = [i for i in items if i in text.lower()]
    if found: print(f"{category}: {found}")
EOF
```

---

## 1. Document ingestion and chunking

### Chunking strategies
```python
# rag/chunking.py
from dataclasses import dataclass
from typing import Iterator
import re

@dataclass
class Chunk:
    content:    str
    doc_id:     str
    chunk_index: int
    metadata:   dict
    token_count: int = 0

class ChunkingStrategy:
    """
    Strategy selection guide:
    - RecursiveCharacter: General purpose (default)
    - Semantic:           When logical coherence matters (dense technical docs)
    - Sentence:           FAQ / short Q&A content
    - Fixed (overlap):    Tabular / structured data
    - ParentDocument:     When small chunks retrieve but big chunks generate
    """

    @staticmethod
    def recursive_character(text: str, chunk_size: int = 512, overlap: int = 64) -> list[str]:
        """Default — splits on paragraphs, sentences, words in order."""
        separators = ["\n\n", "\n", ". ", " ", ""]
        chunks = []

        def split(text: str, seps: list[str]) -> list[str]:
            if not seps or len(text) <= chunk_size:
                return [text] if text.strip() else []
            sep = seps[0]
            parts = text.split(sep)
            current, result = "", []
            for part in parts:
                if len(current) + len(part) + len(sep) <= chunk_size:
                    current += (sep if current else "") + part
                else:
                    if current: result.append(current)
                    current = part
            if current: result.append(current)
            # Re-split oversized chunks with next separator
            final = []
            for chunk in result:
                if len(chunk) > chunk_size:
                    final.extend(split(chunk, seps[1:]))
                else:
                    final.append(chunk)
            return final

        raw = split(text, separators)
        # Apply overlap
        for i, chunk in enumerate(raw):
            if i == 0 or not overlap:
                chunks.append(chunk)
            else:
                prev = raw[i-1]
                overlap_text = prev[-overlap:] if len(prev) > overlap else prev
                chunks.append(overlap_text + "\n" + chunk)
        return chunks

    @staticmethod
    def by_markdown_header(text: str, max_chunk_tokens: int = 400) -> list[dict]:
        """Preserve document structure — split at headings, keep heading as metadata."""
        sections = []
        current_section = {"heading": "Introduction", "content": [], "level": 0}

        for line in text.splitlines():
            heading_match = re.match(r"^(#{1,6})\s+(.+)", line)
            if heading_match:
                if current_section["content"]:
                    sections.append(current_section)
                current_section = {
                    "heading": heading_match.group(2),
                    "content": [],
                    "level":   len(heading_match.group(1)),
                }
            else:
                current_section["content"].append(line)

        if current_section["content"]:
            sections.append(current_section)

        return [{
            "text":    s["heading"] + "\n" + "\n".join(s["content"]),
            "heading": s["heading"],
            "level":   s["level"],
        } for s in sections if "\n".join(s["content"]).strip()]
```

### Document loader
```python
# rag/loaders.py
from pathlib import Path
import fitz   # PyMuPDF
import docx
import boto3

class DocumentLoader:
    def load(self, source: str | Path) -> list[dict]:
        path = Path(source)
        if not path.exists():
            raise FileNotFoundError(f"Document not found: {source}")

        loaders = {
            ".pdf":  self._load_pdf,
            ".docx": self._load_docx,
            ".txt":  self._load_text,
            ".md":   self._load_text,
        }
        loader = loaders.get(path.suffix.lower())
        if not loader:
            raise ValueError(f"Unsupported file type: {path.suffix}")

        pages = loader(path)
        return [{"content": p["content"], "page": p.get("page", 0),
                 "source": str(path), "file_type": path.suffix[1:]} for p in pages]

    def _load_pdf(self, path: Path) -> list[dict]:
        doc = fitz.open(path)
        return [{"content": page.get_text("text").strip(), "page": i+1}
                for i, page in enumerate(doc) if page.get_text("text").strip()]

    def _load_docx(self, path: Path) -> list[dict]:
        doc = docx.Document(path)
        content = "\n".join(p.text for p in doc.paragraphs if p.text.strip())
        return [{"content": content}]

    def _load_text(self, path: Path) -> list[dict]:
        return [{"content": path.read_text(encoding="utf-8")}]
```

---

## 2. Embeddings

```python
# rag/embeddings.py
import anthropic
import numpy as np
from typing import Optional

class EmbeddingService:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model  = "voyage-3"   # Voyage is Anthropic's embedding model
        self.cache: dict[str, list[float]] = {}

    def embed(self, texts: list[str], input_type: str = "document") -> np.ndarray:
        """
        input_type: "document" for indexing, "query" for search queries
        Voyage differentiates these for better retrieval quality.
        """
        # Batch (Voyage supports up to 128 texts per request)
        batch_size = 96
        all_embeddings = []

        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            # Check cache
            cached = [self.cache.get(t) for t in batch]
            to_embed = [t for t, c in zip(batch, cached) if c is None]

            if to_embed:
                response = self.client.beta.embeddings.create(
                    model=self.model,
                    input=to_embed,
                    input_type=input_type,
                )
                new_embeddings = [e.embedding for e in response.data]
                for text, emb in zip(to_embed, new_embeddings):
                    self.cache[text] = emb

            embeddings = [self.cache.get(t) or next(iter([e.embedding for e in response.data])) for t in batch]
            all_embeddings.extend(embeddings)

        return np.array(all_embeddings)

    def similarity(self, q: list[float], d: list[float]) -> float:
        q_arr, d_arr = np.array(q), np.array(d)
        return float(np.dot(q_arr, d_arr) / (np.linalg.norm(q_arr) * np.linalg.norm(d_arr)))
```

---

## 3. Vector store integrations

### pgvector (PostgreSQL — recommended for most teams)
```sql
-- Setup
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
    id           SERIAL PRIMARY KEY,
    doc_id       TEXT NOT NULL,
    content      TEXT NOT NULL,
    embedding    vector(1024),            -- Voyage-3 dimension
    metadata     JSONB DEFAULT '{}',
    chunk_index  INTEGER,
    created_at   TIMESTAMP DEFAULT NOW()
);

-- HNSW index for fast ANN search
CREATE INDEX ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

```python
# rag/stores/pgvector_store.py
import psycopg2, json
import numpy as np

class PGVectorStore:
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)

    def upsert(self, chunks: list[Chunk], embeddings: np.ndarray):
        with self.conn.cursor() as cur:
            for chunk, emb in zip(chunks, embeddings):
                cur.execute("""
                    INSERT INTO document_chunks (doc_id, content, embedding, metadata, chunk_index)
                    VALUES (%s, %s, %s::vector, %s, %s)
                    ON CONFLICT (doc_id, chunk_index) DO UPDATE
                    SET content=EXCLUDED.content, embedding=EXCLUDED.embedding
                """, (chunk.doc_id, chunk.content, emb.tolist(), json.dumps(chunk.metadata), chunk.chunk_index))
        self.conn.commit()

    def search(self, query_embedding: list[float], k: int = 10, filter: dict = None) -> list[dict]:
        where_clause = ""
        params = [query_embedding, k]
        if filter:
            conditions = [f"metadata->>{repr(k)} = %s" for k in filter]
            where_clause = "WHERE " + " AND ".join(conditions)
            params = [query_embedding] + list(filter.values()) + [k]

        with self.conn.cursor() as cur:
            cur.execute(f"""
                SELECT id, doc_id, content, metadata, chunk_index,
                       1 - (embedding <=> %s::vector) AS similarity
                FROM document_chunks
                {where_clause}
                ORDER BY embedding <=> %s::vector
                LIMIT %s
            """, [query_embedding] + (list(filter.values()) if filter else []) + [k])

            cols = [d[0] for d in cur.description]
            return [dict(zip(cols, row)) for row in cur.fetchall()]
```

### Pinecone
```python
# rag/stores/pinecone_store.py
from pinecone import Pinecone, ServerlessSpec

class PineconeStore:
    def __init__(self, api_key: str, index_name: str, dimension: int = 1024):
        pc = Pinecone(api_key=api_key)
        if index_name not in pc.list_indexes().names():
            pc.create_index(name=index_name, dimension=dimension, metric="cosine",
                            spec=ServerlessSpec(cloud="aws", region="us-east-1"))
        self.index = pc.Index(index_name)

    def upsert(self, chunks: list[Chunk], embeddings: np.ndarray, batch_size: int = 100):
        vectors = [
            {"id": f"{c.doc_id}_{c.chunk_index}", "values": e.tolist(),
             "metadata": {"content": c.content, "doc_id": c.doc_id, **c.metadata}}
            for c, e in zip(chunks, embeddings)
        ]
        for i in range(0, len(vectors), batch_size):
            self.index.upsert(vectors=vectors[i:i+batch_size])

    def search(self, query_embedding: list[float], k: int = 10, filter: dict = None) -> list[dict]:
        results = self.index.query(vector=query_embedding, top_k=k,
                                   filter=filter, include_metadata=True)
        return [{"content": m.metadata["content"], "score": m.score, "doc_id": m.metadata["doc_id"],
                 "id": m.id} for m in results.matches]
```

---

## 4. Hybrid search (BM25 + dense)

```python
# rag/retrieval/hybrid.py
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    def __init__(self, vector_store, alpha: float = 0.7):
        """
        alpha: weight for dense search (0.0=BM25 only, 1.0=dense only)
        Recommended: 0.7 for general, 0.5 for technical docs with jargon
        """
        self.store = vector_store
        self.alpha = alpha
        self.bm25 = None
        self.corpus: list[dict] = []

    def index_bm25(self, chunks: list[dict]):
        self.corpus = chunks
        tokenised = [c["content"].lower().split() for c in chunks]
        self.bm25 = BM25Okapi(tokenised)

    def retrieve(self, query: str, query_embedding: list[float], k: int = 20) -> list[dict]:
        # Dense retrieval
        dense_results = self.store.search(query_embedding, k=k*2)
        dense_scores = {r["id"]: r["score"] for r in dense_results}

        # BM25 retrieval
        bm25_scores_raw = self.bm25.get_scores(query.lower().split())
        top_bm25_indices = np.argsort(bm25_scores_raw)[::-1][:k*2]

        # Normalise scores
        max_bm25 = bm25_scores_raw[top_bm25_indices[0]] if top_bm25_indices.any() else 1.0
        bm25_scores = {
            self.corpus[i]["id"]: bm25_scores_raw[i] / (max_bm25 + 1e-8)
            for i in top_bm25_indices
        }

        # Combine with RRF (Reciprocal Rank Fusion)
        all_ids = set(dense_scores) | set(bm25_scores)
        combined = {
            id_: (self.alpha * dense_scores.get(id_, 0.0) +
                  (1 - self.alpha) * bm25_scores.get(id_, 0.0))
            for id_ in all_ids
        }

        sorted_ids = sorted(combined.items(), key=lambda x: -x[1])[:k]
        id_to_chunk = {c["id"]: c for c in self.corpus if c["id"] in combined}
        id_to_chunk.update({r["id"]: r for r in dense_results})

        return [{"score": score, **id_to_chunk[id_]} for id_, score in sorted_ids if id_ in id_to_chunk]
```

---

## 5. Re-ranking

```python
# rag/retrieval/reranker.py
# Cross-encoder re-ranker — significantly improves precision at top-k
from sentence_transformers import CrossEncoder

class Reranker:
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = CrossEncoder(model_name)

    def rerank(self, query: str, chunks: list[dict], top_k: int = 5) -> list[dict]:
        if not chunks: return []
        pairs = [[query, c["content"]] for c in chunks]
        scores = self.model.predict(pairs)
        ranked = sorted(zip(chunks, scores), key=lambda x: -x[1])
        return [{"rerank_score": float(score), **chunk} for chunk, score in ranked[:top_k]]

# Cohere reranker (cloud — higher quality)
import cohere

class CohereReranker:
    def __init__(self, api_key: str):
        self.client = cohere.Client(api_key)

    def rerank(self, query: str, chunks: list[dict], top_k: int = 5) -> list[dict]:
        docs = [c["content"] for c in chunks]
        results = self.client.rerank(query=query, documents=docs,
                                     top_n=top_k, model="rerank-english-v3.0")
        return [{"rerank_score": r.relevance_score, **chunks[r.index]}
                for r in results.results]
```

---

## 6. Production RAG pipeline

```python
# rag/pipeline.py
class RAGPipeline:
    def __init__(self, vector_store, embedding_svc, reranker, llm_client, langfuse=None):
        self.store    = vector_store
        self.embedder = embedding_svc
        self.reranker = reranker
        self.llm      = llm_client
        self.langfuse = langfuse

    async def query(self, question: str, user_id: str = None,
                    top_k: int = 20, rerank_top: int = 5,
                    metadata_filter: dict = None) -> RAGResponse:

        trace = self.langfuse.trace(name="rag_query", user_id=user_id) if self.langfuse else None

        # 1. Embed query
        query_emb = self.embedder.embed([question], input_type="query")[0]

        # 2. Retrieve (hybrid)
        retrieved = self.store.search(query_emb.tolist(), k=top_k, filter=metadata_filter)

        # 3. Re-rank
        reranked = self.reranker.rerank(question, retrieved, top_k=rerank_top)

        # 4. Build context (with token budget)
        context, token_budget = "", 12_000  # leave room for question + output
        sources = []
        for chunk in reranked:
            chunk_tokens = len(chunk["content"]) // 4  # rough estimate
            if token_budget - chunk_tokens < 2000: break
            context += f"\n---\nSource: {chunk.get('doc_id','unknown')}\n{chunk['content']}\n"
            token_budget -= chunk_tokens
            sources.append({"doc_id": chunk.get("doc_id"), "score": chunk.get("rerank_score")})

        if not context:
            return RAGResponse(answer="I couldn't find relevant information.", sources=[], retrieved=retrieved)

        # 5. Generate
        response = await self.llm.complete(
            system="""Answer the question using ONLY the provided context.
If the answer is not in the context, say "I don't have information about that."
Cite the source (doc_id) for each fact you state. Be concise and precise.""",
            user=f"Context:\n{context}\n\nQuestion: {question}",
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            temperature=0.0,
        )

        if trace:
            trace.span(name="rag_query", input={"question": question},
                output={"answer": response.content, "sources": sources},
                metadata={"retrieved": len(retrieved), "reranked": len(reranked)})

        return RAGResponse(answer=response.content, sources=sources, retrieved=retrieved, context=context)
```

---

## 7. RAG evaluation

```python
# evals/rag_eval.py
# Four key RAG metrics (RAGAS framework)

class RAGEvaluator:
    def __init__(self, llm_client):
        self.llm = llm_client

    async def evaluate(self, question: str, answer: str,
                       context: str, ground_truth: str = None) -> RAGMetrics:
        # Faithfulness: is the answer grounded in context?
        faithfulness = await self._score_faithfulness(answer, context)

        # Answer relevance: does the answer address the question?
        relevance = await self._score_answer_relevance(question, answer)

        # Context precision: how much of retrieved context is actually needed?
        precision = await self._score_context_precision(question, context)

        # Correctness (requires ground truth)
        correctness = await self._score_correctness(answer, ground_truth) if ground_truth else None

        return RAGMetrics(faithfulness=faithfulness, answer_relevance=relevance,
                          context_precision=precision, correctness=correctness)

    async def _score_faithfulness(self, answer: str, context: str) -> float:
        resp = await self.llm.complete(
            system="Score from 0.0-1.0: Is every claim in the answer directly supported by the context? Return JSON: {\"score\": 0.0-1.0}",
            user=f"Context:\n{context}\n\nAnswer:\n{answer}",
            temperature=0.0, max_tokens=64,
        )
        return json.loads(resp.content)["score"]

    async def _score_answer_relevance(self, question: str, answer: str) -> float:
        resp = await self.llm.complete(
            system="Score 0.0-1.0: How well does the answer address the question? Return JSON: {\"score\": 0.0-1.0}",
            user=f"Question: {question}\n\nAnswer: {answer}",
            temperature=0.0, max_tokens=64,
        )
        return json.loads(resp.content)["score"]
```

---

## 8. Ingestion pipeline

```python
# rag/ingest.py — full document ingestion pipeline
import asyncio
from tqdm import tqdm

async def ingest_documents(paths: list[str], vector_store, embedding_svc,
                           chunk_size: int = 512, batch_size: int = 32):
    loader   = DocumentLoader()
    chunker  = ChunkingStrategy()
    all_chunks, skipped = [], 0

    for path in tqdm(paths, desc="Loading documents"):
        try:
            pages  = loader.load(path)
            for page in pages:
                raw_chunks = chunker.recursive_character(page["content"], chunk_size=chunk_size, overlap=64)
                for i, text in enumerate(raw_chunks):
                    if len(text.strip()) < 50: continue  # skip too-short chunks
                    all_chunks.append(Chunk(
                        content=text, doc_id=path, chunk_index=i,
                        metadata={"page": page.get("page", 0), "source": path},
                    ))
        except Exception as e:
            print(f"Error loading {path}: {e}")
            skipped += 1

    print(f"Chunks: {len(all_chunks)}  Skipped: {skipped}")

    # Embed in batches
    for i in tqdm(range(0, len(all_chunks), batch_size), desc="Embedding"):
        batch = all_chunks[i:i+batch_size]
        texts = [c.content for c in batch]
        embeddings = embedding_svc.embed(texts, input_type="document")
        vector_store.upsert(batch, embeddings)
        await asyncio.sleep(0.1)  # rate limit

    print(f"Ingestion complete: {len(all_chunks)} chunks indexed")
```

---

## 9. Common RAG failure modes

```
Failure                 | Diagnosis                        | Fix
------------------------|----------------------------------|---------------------------
Poor retrieval quality  | Relevant chunks not returned     | Hybrid search + rerank
Hallucinations          | Answer not in context            | Check faithfulness score
Context too long        | LLM ignores middle of context    | Better reranking + trim
Repetitive context      | Same chunk retrieved multiple    | Deduplicate by doc_id
Wrong granularity       | Chunk too large/small            | Tune chunk_size via eval
No cross-chunk answers  | Answer spans multiple chunks     | Increase overlap, parent-doc
Stale data              | Index not updated after source   | Incremental ingestion pipeline
Query-doc mismatch      | Different vocabularies           | Voyage voyage-3 input_type
```

```bash
# Quick RAG health check
python3 - <<'EOF'
# Run a set of known Q&A pairs through the pipeline and score
import asyncio

test_cases = [
    {"q": "What is our refund policy?",  "expected_keywords": ["30 days", "receipt"]},
    {"q": "How do I reset my password?", "expected_keywords": ["email", "link", "minutes"]},
]

async def spot_check():
    for tc in test_cases:
        # Replace with your pipeline
        result = await pipeline.query(tc["q"])
        hits = sum(1 for kw in tc["expected_keywords"] if kw.lower() in result.answer.lower())
        score = hits / len(tc["expected_keywords"])
        print(f"{'✓' if score >= 0.5 else '✗'} ({score:.0%}) {tc['q'][:50]}")
        if score < 0.5:
            print(f"  Got: {result.answer[:100]}")

asyncio.run(spot_check())
EOF
```
