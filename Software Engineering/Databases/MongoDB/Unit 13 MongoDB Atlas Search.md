---
tags: [databases, databases/mongodb]
aliases: [Atlas Search, MongoDB Search, $search, Search Index, MongoDB Vector Search, MongoDB Unit 13]
updated: 2026-09-27
---
# Unit 13 MongoDB Atlas Search

> [!tldr] TL;DR
> **Atlas Search** (now also called MongoDB Search) adds Lucene-based **relevance search** (full-text, fuzzy, autocomplete, facets) next to your data, queried with the `$search` / `$searchMeta` aggregation stages. It uses **search indexes**, separate from normal database indexes, defined by **analyzers** and **field mappings** (dynamic or static). `compound` combines clauses (`must`, `mustNot`, `should`, `filter`) to shape the score, and **facets** bucket results. Since 2025 the same engine powers **Vector Search** and **hybrid search** (`$scoreFusion`), and it's available for self-managed Community/Enterprise in preview.

> Course: MongoDB University, Unit 13 · Prev: [[Unit 12 MongoDB Indexes]] · Next: none (end of the path, back to [[MongoDB]])

## Setup used in the lab
Connect to the sample dataset:

```bash
mongosh -u myAtlasDBUser -p myatlas-001 $MY_ATLAS_CONNECTION_STRING/sample_supplies
```

> [!warning] Credentials in this note
> The command above has the lab password in plain text (it also ends up in shell history). Treat it as exposed and **rotate it** in Atlas. Use a placeholder or env var instead, or leave out `-p` so mongosh prompts: `mongosh -u "$ATLAS_DB_USER" "$MY_ATLAS_CONNECTION_STRING/sample_supplies"`.

## Relevance search vs database search
| | Relevance-based search | Database search |
|---|---|---|
| Who | end users typing into a search bar | developers querying by criteria |
| Question | "which records best match these words?" | "which records meet exactly these conditions?" |
| Result | ranked by a **score** | filtered set, ordered by a sort |
| Handles | typos, stemming, synonyms, partial words | exact values, ranges |
| MongoDB tool | `$search` + search index | `find()` / `$match` + B-tree index |

## Search indexes vs database indexes
- **Database indexes** (B-trees, [[Unit 12 MongoDB Indexes]]) make exact-match, range and sort queries efficient.
- **Search indexes** (Lucene inverted indexes, run by the `mongot` process) define how text is tokenized and referenced for relevance-based search. They are updated asynchronously from the change stream, so they are **eventually consistent** with the collection.

### Components of a search index
- **Analyzers**: how text is split and normalized into tokens. An *index analyzer* at index time and a *search analyzer* at query time (e.g. `lucene.standard`, `lucene.english`, `lucene.keyword`).
- **Mapping type**: dynamic or static.
- **Stored source**: optionally store some fields in the search index to return them without a lookup.
- **Field mappings**: the type of each field (`string`, `number`, `date`, `autocomplete`, `token`, `document`, ...).

## Dynamic mapping
With **dynamic mapping**, all fields of supported types are indexed automatically. In the course version, booleans, ObjectIds and timestamps were not indexed dynamically (newer docs allow configuring dynamic type sets; check the current docs).

### Create from the CLI
`search_index.json`:

```json
{
  "name": "sample_supplies-sales-dynamic",
  "searchAnalyzer": "lucene.standard",
  "analyzer": "lucene.standard",
  "collectionName": "sales",
  "database": "sample_supplies",
  "mappings": {
    "dynamic": true
  }
}
```

```bash
atlas clusters search indexes create \
  --clusterName myAtlasClusterEDU \
  -f /app/search_index.json

atlas clusters search indexes list \
  --clusterName myAtlasClusterEDU \
  --db sample_supplies \
  --collection sales
```

You can also create search indexes from `mongosh` or a driver: `db.sales.createSearchIndex("name", { mappings: { dynamic: true } })`.

### Query it
```js
db.sales.aggregate([
  {
    $search: {
      index: "sample_supplies-sales-dynamic",
      text: { query: "notepad", path: { wildcard: "*" } }
    }
  },
  { $set: { score: { $meta: "searchScore" } } }
])
```

### Create from the Atlas UI
Go to the **Search** tab of your cluster (Atlas Search / Search & Vector Search):

![[Screenshot_2023-04-27_at_4.15.34_PM.png]]

![[Screenshot_2023-04-27_at_4.16.06_PM.png]]

![[Screenshot_2023-04-27_at_4.16.38_PM.png]]

And that's how you create a search index.

## Static field mapping
Use **static mapping** when the queried fields are always the same: only listed fields are indexed, which keeps the index small and fast.

```json
{
  "name": "sample_supplies-sales-static",
  "searchAnalyzer": "lucene.standard",
  "analyzer": "lucene.standard",
  "collectionName": "sales",
  "database": "sample_supplies",
  "mappings": {
    "dynamic": false,
    "fields": {
      "storeLocation": { "type": "string" }
    }
  }
}
```

```bash
atlas clusters search indexes create --clusterName myAtlasClusterEDU -f /app/search_index.json
atlas clusters search indexes list --clusterName myAtlasClusterEDU --db sample_supplies --collection sales
```

## `compound`: controlling the score
`compound` combines clauses and decides how each affects results and scoring:

| Clause | Effect |
|---|---|
| `must` | documents **must** match; contributes to the score |
| `mustNot` | documents must **not** match (the negation of `must`) |
| `should` | optional; matching documents get a **higher score** (weight it with `score`) |
| `filter` | documents must match, but it **doesn't affect the score** (eliminates results) |

![[Screenshot_2023-04-27_at_5.19.34_PM.png]]

Birds example: must live in a "field" habitat; boost those with a wingspan of 45 cm or more:

```js
{
  $search: {
    compound: {
      must: [{ text: { query: "field", path: "habitat" } }],
      should: [{
        range: { gte: 45, path: "wingspan_cm", score: { constant: { value: 5 } } }
      }]
    }
  }
}
```

Sales example: only online purchases, boost those containing a notepad:

```js
db.sales.aggregate([
  {
    $search: {
      index: "sample_supplies-sales-dynamic",
      compound: {
        filter: [{ text: { query: "Online", path: "purchaseMethod" } }],
        should: [{
          text: { query: "notepad", path: "items.name", score: { constant: { value: 5 } } }
        }]
      }
    }
  },
  {
    $project: {
      "items.name": 1,
      purchaseMethod: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

## Facets: grouping search results
**Facets** are buckets you group search results into (by string, number or date), like the category counts in a shop's sidebar. Use **`$searchMeta`** to get the facet counts (metadata about the search) rather than the documents.

```js
{
  $searchMeta: {
    facet: {
      operator: { text: { query: ["machine"], path: "author" } },
      facets: {
        weeklyFacet: {
          type: "date",
          path: "date",
          boundaries: [
            ISODate("2022-01-01"),
            ISODate("2022-01-08"),
            ISODate("2022-01-15"),
            ISODate("2022-01-22")
          ],
          default: "other"
        }
      }
    }
  }
}
```
(The original note had invalid dates like `2022-01-015`; fixed.)

Index with a facet field mapping:

```json
{
  "name": "sample_supplies-sales-facets",
  "searchAnalyzer": "lucene.standard",
  "analyzer": "lucene.standard",
  "collectionName": "sales",
  "database": "sample_supplies",
  "mappings": {
    "dynamic": true,
    "fields": {
      "purchaseMethod": [
        { "dynamic": true, "type": "document" },
        { "type": "string" }
      ],
      "storeLocation": [
        { "dynamic": true, "type": "document" },
        { "type": "stringFacet" }
      ]
    }
  }
}
```

Newer docs recommend the `token` type for string facets instead of `stringFacet`; check the current docs.

```bash
atlas clusters search indexes create --clusterName myAtlasClusterEDU -f /app/search_index.json
atlas clusters search indexes list --clusterName myAtlasClusterEDU --db sample_supplies --collection sales
```

```js
db.sales.aggregate([
  {
    $searchMeta: {
      index: "sample_supplies-sales-facets",
      facet: {
        operator: { text: { query: "In store", path: "purchaseMethod" } },
        facets: {
          locationFacet: { type: "string", path: "storeLocation" }
        }
      }
    }
  }
])
// => { count: { lowerBound: ... }, facet: { locationFacet: { buckets: [ { _id: "Denver", count: ... }, ... ] } } }
```

## Beyond the course: Vector Search and hybrid search
**Vector Search** finds documents whose **embedding** (a vector from an ML model) is close to the query's embedding: semantic search, RAG for LLM apps, recommendations. See [[Vector Databases & pgvector]].

```js
// Vector search index definition
{ fields: [ { type: "vector", path: "plot_embedding", numDimensions: 1024, similarity: "cosine" },
            { type: "filter", path: "year" } ] }

// Query
db.movies.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "plot_embedding",
      queryVector: [/* 1024 floats from your embedding model */],
      numCandidates: 200,     // ANN candidates considered
      limit: 10,
      filter: { year: { $gte: 2000 } }
    }
  },
  { $project: { title: 1, score: { $meta: "vectorSearchScore" } } }
])
```

**Hybrid search** combines keyword (`$search`) and semantic (`$vectorSearch`) results. `$scoreFusion` normalizes and combines their scores:

```js
db.movies.aggregate([
  {
    $scoreFusion: {
      input: {
        pipelines: {
          vector: [{ $vectorSearch: { index: "vector_index", path: "plot_embedding", queryVector: qv, numCandidates: 200, limit: 20 } }],
          text:   [{ $search: { index: "default", text: { query: "space heist", path: "plot" } } }]
        },
        normalization: "sigmoid"
      },
      combination: { weights: { vector: 0.7, text: 0.3 } }
    }
  },
  { $limit: 10 }
])
```

> [!info]+ 🆕 2025–2026
> - **2025-02-24**: MongoDB announced the **acquisition of Voyage AI** (embedding and reranking models). ([press release](https://www.mongodb.com/press/mongodb-announces-acquisition-of-voyage-ai))
> - **2025-09-16, MongoDB 8.2**: **Search and Vector Search for Community Edition and Enterprise Server** in public preview (previously Atlas-only), `$scoreFusion` for hybrid search, and new Queryable Encryption query types (prefix, suffix, substring). ([8.2 announcement](https://www.mongodb.com/products/updates/mongodb-8-2-is-now-available/))
> - **May 2026, MongoDB 8.3**: `$scoreFusion` is **GA**. ([8.3 release notes](https://www.mongodb.com/docs/manual/release-notes/8.3/))
> - **Automated Embedding** (public preview, not for production): Vector Search generates and syncs embeddings for you with Voyage AI models (`voyage-4`, `voyage-4-lite`, `voyage-4-large`, `voyage-code-4`) using an `autoEmbed` field type; available on Atlas and self-managed Community 8.2+. ([docs](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/))

## Common pitfalls
- Putting `$search` anywhere but the **first stage** (it must be first).
- Using `$match` after `$search` for filtering: slow; use `compound.filter` inside `$search` instead.
- Expecting search results immediately after a write: search indexes are eventually consistent.
- Dynamic mappings on huge, varied documents create large indexes.

## Senior perspective
- **Architecture**: Atlas Search runs `mongot` alongside (or on dedicated **Search Nodes**, separate from) the database nodes. Dedicated search nodes isolate search load from OLTP.
- **Why not Elasticsearch?** Search inside the database removes a sync pipeline and its consistency bugs, and uses one query API. A separate search cluster may still win for very large, search-first workloads or advanced relevance tuning.
- **Relevance tuning** is product work: choose analyzers per language, add synonyms, boost fields with `should`, test with real queries, measure click-through.
- **Vector search at scale**: memory for vector indexes, quantization (scalar/binary) to cut RAM, `numCandidates` vs recall/latency trade-off, pre-filter fields in the index.
- **Hybrid search** usually beats pure vector or pure keyword for real users; tune weights with offline evaluation.

## Interview questions
> [!question]- How is a search index different from a regular MongoDB index?
> A regular index is a B-tree on field values for exact matches, ranges and sorts. A search index is a Lucene inverted index built with analyzers (tokenizing, stemming, lowercasing) for relevance-ranked full-text queries; it's maintained asynchronously by `mongot` and queried with `$search`.

> [!question]- Dynamic vs static mappings?
> Dynamic indexes all supported fields automatically: easy, but bigger. Static indexes only the listed fields with explicit types and analyzers: smaller, faster and more controlled. Use static in production when query fields are known.

> [!question]- What's the difference between must, should and filter in compound?
> `must` must match and adds to the score; `should` is optional and boosts matching documents; `filter` must match but doesn't affect the score; `mustNot` excludes matches.

> [!question]- What is $searchMeta used for?
> Returning metadata about a search, such as total counts and facet buckets, without the documents themselves.

> [!question]- What is hybrid search and how do you do it in MongoDB?
> Combining lexical (keyword) and semantic (vector) retrieval so you catch exact terms and meaning. In MongoDB, run `$search` and `$vectorSearch` pipelines and merge them with `$scoreFusion` (score-based, GA in 8.3) or `$rankFusion` (rank-based).

## Related
- [[MongoDB]] · [[Databases Index]]
- [[Vector Databases & pgvector]] · [[Database Indexes]] · [[NoSQL Database Types]]

## Sources
- [MongoDB Search (Atlas Search) documentation](https://www.mongodb.com/docs/atlas/atlas-search/)
- [compound operator](https://www.mongodb.com/docs/atlas/atlas-search/compound/)
- [facet collector](https://www.mongodb.com/docs/atlas/atlas-search/facet/)
- [MongoDB Vector Search](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-overview/)
- [$scoreFusion](https://www.mongodb.com/docs/manual/reference/operator/aggregation/scorefusion/)
- [Automated Embedding](https://www.mongodb.com/docs/vector-search/crud-embeddings/automated-embedding/)
- [MongoDB 8.2 is now available](https://www.mongodb.com/products/updates/mongodb-8-2-is-now-available/)
- [MongoDB announces acquisition of Voyage AI](https://www.mongodb.com/press/mongodb-announces-acquisition-of-voyage-ai)
- [MongoDB University: MongoDB Atlas Search](https://learn.mongodb.com/)
