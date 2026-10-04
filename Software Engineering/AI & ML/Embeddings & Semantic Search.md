---
title: "Embeddings & Semantic Search"
tags: [ai-ml, ai-ml/llm, ai-ml/retrieval]
aliases: [Embeddings, Semantic Search, Vector Search, Cosine Similarity, ANN Search, HNSW]
updated: 2026-09-27
---
# Embeddings & Semantic Search

<Note>
**TL;DR**

An **embedding model** turns text (or images, code, audio) into a fixed-length vector so that **similar meaning ≈ nearby vectors**. **Semantic search** embeds documents once, embeds each query, and returns the nearest vectors by **cosine similarity** or dot product, usually via an **approximate nearest neighbour (ANN)** index such as **HNSW**. It finds "forgot my login" when the doc says "reset your password", which keyword search misses; keyword search (BM25) still wins on exact terms, so production systems often combine both (**hybrid search**).
</Note>

## The idea

Keyword search matches **words**. Semantic search matches **meaning**.

| Query | Keyword (BM25) finds | Semantic search finds |
|---|---|---|
| "forgot my login" | docs containing "forgot" or "login" | "How to reset your password" |
| "cheap flights to the Big Apple" | "cheap", "flights" | "Budget airfare to New York" |
| "error ERR_CERT_DATE_INVALID" | the exact error code ✓ | possibly something vaguely about certificates ✗ |

Background on vectors and embeddings: [ML Core Concepts](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/ML%20Core%20Concepts).

## How embeddings are made
An embedding model is a neural network (usually a transformer encoder) trained with **contrastive learning**: pull together pairs that belong together (question and its answer, a caption and its image, a query and the clicked result), push apart random pairs. The model learns a space where distance reflects relatedness.

- Output: a vector of fixed size, e.g. 384, 768, 1,024, 1,536 or 3,072 dimensions.
- Many models support **Matryoshka** representations: you can truncate the vector (e.g. 3,072 → 256 dims) with modest quality loss, saving storage and speeding search.
- **Multimodal** embedding models place images and text in the same space (text-to-image search).
- An embedding model is **not** the chat LLM; it's a separate, much cheaper model.

## Measuring similarity

```text
cosine(a, b) = (a · b) / (|a| · |b|)     1 = same direction, 0 = unrelated, −1 = opposite
dot(a, b)    = Σ ai·bi                   equals cosine when vectors are normalised to length 1
euclidean    = √Σ (ai − bi)²             smaller = closer
```

```python
import numpy as np
def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

q  = embed("forgot my login")
d1 = embed("How to reset your password")
d2 = embed("Our office opening hours")
cosine(q, d1)  # e.g. 0.78  -> relevant
cosine(q, d2)  # e.g. 0.12  -> not relevant
```

Use the metric the model was trained for (most text models: cosine, or dot product on normalised vectors). Absolute scores aren't comparable across models; only rank within one model.

## The semantic search pipeline

{/* diagram source: /diagrams/diagram-007.mmd */}
![diagram](/diagrams/diagram-007.svg)

1. **Chunk** documents into passages (a few hundred tokens), keeping titles and metadata. Details in [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)).
2. **Embed** each chunk and store the vector with its text, source and metadata (tenant, date, permissions).
3. At query time **embed the query with the same model** (some models need a different prefix or input type for queries vs documents).
4. **Search** for the top-k nearest vectors, applying metadata **filters** (e.g. `tenant_id = 42`).
5. Optionally **rerank** and combine with keyword search.

### Postgres example (pgvector)

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE doc_chunks (
  id bigserial PRIMARY KEY,
  tenant_id int NOT NULL,
  content text NOT NULL,
  embedding vector(1536) NOT NULL
);
CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);

-- $1 = query embedding computed in the app
SELECT id, content, 1 - (embedding <=> $1) AS similarity
FROM doc_chunks
WHERE tenant_id = 42
ORDER BY embedding <=> $1          -- <=> is cosine distance
LIMIT 5;
```

More on storage and indexing: [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector).

## Approximate nearest neighbour (ANN) search
Comparing a query with every vector (exact k-NN) is fine for tens of thousands of vectors, too slow for millions. ANN indexes trade a little recall for huge speed-ups.

| Index | Idea | Trade-offs |
|---|---|---|
| **Flat / brute force** | compare with everything | exact, slow at scale |
| **HNSW** | multi-layer graph; greedy walk from coarse to fine layers | fast, high recall, memory-hungry, slower builds; the default in most vector DBs |
| **IVF** (inverted file) | cluster vectors, search only nearby clusters | less memory, needs training, recall depends on `nprobe` |
| **PQ / quantisation** | compress vectors (int8, binary, product quantisation) | much less memory, some accuracy loss; often combined with rescoring |

Tuning knobs (e.g. HNSW `ef_search`) trade **recall vs latency**. Measure recall@k against exact search on a sample.

## Hybrid search
Combine lexical (BM25 / full-text) and semantic results, commonly with **Reciprocal Rank Fusion (RRF)**:

```text
score(doc) = Σ over result lists  1 / (k + rank_in_list)      (k ≈ 60)
```

Hybrid handles both "meaning" queries and exact identifiers (error codes, SKUs, names). A **reranker** (cross-encoder that reads query and document together) can then reorder the top 50–100 for precision.

## Other uses of embeddings
- **Recommendations** ("more like this"), **deduplication** and near-duplicate detection.
- **Clustering and topic discovery** (a modern version of [Example 2 Micro-genre Book Genre](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/Case%20Studies%20that%20use%20ML/Example%202%20Micro-genre%20Book%20Genre)).
- **Classification** with a small model on top of embeddings, or zero-shot by comparing with label descriptions.
- **Anomaly detection**, **semantic caching** of LLM answers, **routing** queries to the right handler.

## Pitfalls
- **Mixing models**: vectors from different models (or versions) are not comparable. Changing the model means **re-embedding everything**.
- **Bad chunking**: chunks too big (diluted meaning) or too small (no context) hurt more than model choice.
- **Exact-match blindness**: IDs, codes and rare names need keyword search.
- **Negation and nuance**: "flights not via London" may match "flights via London"; embeddings capture topic better than logic.
- **Filtering after ANN** can return too few results; use the database's filtered search or pre-filtering.
- **Security**: embeddings can leak information about the source text (inversion attacks) and multi-tenant indexes need strict filters. See OWASP LLM08 in [AI Safety, Security & Guardrails for Apps](/Software%20Engineering/AI%20%26%20ML/AI%20Safety%2C%20Security%20%26%20Guardrails%20for%20Apps).

## Senior perspective
- **Choose the model with your own eval**: build a set of real queries with known relevant documents and measure recall@k / nDCG@k. Public leaderboards (MTEB) are a starting point, not a decision.
- **Cost model**: embedding is cheap per call, but re-embedding a large corpus on model change is a migration. Store the model name and version with every vector.
- **Dimensions × vectors × bytes** = memory. 10M × 1,536 × 4 bytes ≈ 61 GB before index overhead; quantisation or shorter (Matryoshka) vectors cut this.
- **Freshness**: plan incremental re-indexing when documents change or are deleted (and deletion for privacy requests).
- **Start in your existing database** (pgvector) until scale or features demand a dedicated vector DB.

## Interview questions

<Accordion title="What is an embedding and why does semantic search work?">
A learned vector representation where distance reflects meaning. Because the model was trained to place related texts near each other, a query and a relevant document land close together even without shared words, so nearest-neighbour search finds them.
</Accordion>

<Accordion title="Cosine similarity vs dot product vs Euclidean distance?">
Cosine compares direction only; dot product also reflects magnitude; Euclidean measures straight-line distance. For normalised vectors cosine and dot product give identical rankings and Euclidean is monotonically related. Use what the embedding model was trained with.
</Accordion>

<Accordion title="Why use an ANN index and what's the trade-off?">
Exact search is O(n) per query and too slow for millions of vectors. ANN indexes like HNSW or IVF return approximately the nearest neighbours much faster, trading some recall (tunable) and extra memory or build time.
</Accordion>

<Accordion title="When does keyword search beat semantic search?">
Exact identifiers, error codes, product SKUs, names, rare jargon and precise phrases. That's why production systems use hybrid search with rank fusion.
</Accordion>

<Accordion title="You want to switch embedding models. What's involved?">
Re-embed the whole corpus with the new model (vectors aren't compatible), build a new index side by side, evaluate retrieval quality on a labelled query set, switch reads over, then retire the old index. Store model/version per vector to make this manageable.
</Accordion>

## Related
- [AI & ML Index](/Software%20Engineering/AI%20%26%20ML/AI%20%26%20ML%20Index) · [ML Core Concepts](/Software%20Engineering/AI%20%26%20ML/Machine%20Learning/ML%20Core%20Concepts)
- [Retrieval-Augmented Generation (RAG)](/Software%20Engineering/AI%20%26%20ML/Retrieval-Augmented%20Generation%20(RAG)) · [Vector Databases & pgvector](/Software%20Engineering/Databases/Core%20Concepts/Vector%20Databases%20%26%20pgvector)
- [Transformers & How LLMs Work](/Software%20Engineering/AI%20%26%20ML/Transformers%20%26%20How%20LLMs%20Work) · [LLM Evaluation & Observability](/Software%20Engineering/AI%20%26%20ML/LLM%20Evaluation%20%26%20Observability)

## Sources
- OpenAI, [Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- Google, [Text embeddings (Gemini API)](https://ai.google.dev/gemini-api/docs/embeddings)
- Malkov & Yashunin (2016), [Efficient and robust approximate nearest neighbor search using HNSW graphs](https://arxiv.org/abs/1603.09320)
- Reimers & Gurevych (2019), [Sentence-BERT](https://arxiv.org/abs/1908.10084)
- Kusupati et al. (2022), [Matryoshka Representation Learning](https://arxiv.org/abs/2205.13147)
- Cormack, Clarke & Büttcher (2009), [Reciprocal Rank Fusion outperforms Condorcet and individual rank learning methods](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)
- pgvector: [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)
- MTEB leaderboard: [huggingface.co/spaces/mteb/leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
