# Azure AI Search — DevOps Notes

> Formerly **Azure Cognitive Search** / Azure Search. The retrieval engine behind most RAG apps on Azure.
> Prerequisite: [`01_ai_terms_for_beginners.md`](01_ai_terms_for_beginners.md) §5–6 (embeddings, RAG).

---

## 1. Mental model

```
     DATA SOURCES                  INDEXING PIPELINE (pull)                     QUERY
 ┌────────────────┐   ┌──────────┐   ┌───────────────────────┐   ┌────────┐
 │ Blob / ADLS    │──►│ Data     │──►│ Indexer (scheduled)   │──►│ INDEX  │◄── app: keyword /
 │ Azure SQL      │   │ source   │   │  ├─ document cracking │   │ fields │    vector / hybrid
 │ Cosmos DB      │   │ (conn +  │   │  ├─ skillset (AI):    │   │ vectors│    + semantic ranker
 │ SharePoint*    │   │ container)│  │  │   split → embed →  │   └────────┘
 │ OneLake*       │   └──────────┘   │  │   OCR, entities    │
 └────────────────┘                  │  └─ field mappings    │
                                     └───────────────────────┘
        OR  your code PUSHes JSON docs directly into the index (push model)
```

- **Pull model**: indexer crawls a supported source on a schedule, tracks changes.
- **Push model**: your app/pipeline sends documents via REST/SDK — any source, full control, near-real-time.
- **Index** = like a DB table optimized for search; each document has fields; a key field is required.

---

## 2. Key terms

| Term | Meaning |
|------|---------|
| **Search service** | The Azure resource (`Microsoft.Search/searchServices`). Endpoint `https://<name>.search.windows.net`. |
| **Index** | Schema + data. Fields have attributes: `searchable`, `filterable`, `sortable`, `facetable`, `retrievable`, `key`. |
| **Document** | One record in the index (often one *chunk* of a file in RAG). |
| **Data source** | Connection definition for an indexer. |
| **Indexer** | Crawler that pulls from data source into index on a schedule (min every 5 minutes). |
| **Skillset** | AI enrichment steps during indexing: text split, embedding, OCR, language detection, entity recognition, custom Web API skill. |
| **Integrated vectorization** | Search itself chunks + embeds (via Azure OpenAI/Foundry embedding skill) at index time and **vectorizes queries** at query time (a *vectorizer*). No custom code. |
| **Vector field** | `Collection(Edm.Single)` with `dimensions` matching your embedding model. |
| **Vector profile / algorithm** | HNSW (approximate, fast) or exhaustive kNN (exact, slow). |
| **Vector compression / quantization** | Scalar/binary quantization to cut vector storage (cheaper, small quality loss). |
| **Full-text search (BM25)** | Keyword relevance scoring. |
| **Hybrid search** | Keyword + vector in one query, merged with **RRF** (Reciprocal Rank Fusion). |
| **Semantic ranker** | Microsoft re-ranking model applied to top results; also gives captions/answers. Billed per query beyond free quota. |
| **Scoring profile** | Boost fields/freshness/geo in keyword ranking. |
| **Analyzer** | How text is tokenized (language analyzers, e.g. `en.microsoft`). |
| **Synonym map** | "k8s ⇔ kubernetes". |
| **Filter** | OData `$filter` (e.g. `department eq 'HR'`) — used for **security trimming**. |
| **Index projection** | One source document → many chunk documents in the index (parent/child). |
| **Knowledge store** | Save enriched output to Storage for analytics. |
| **Alias** | Stable name pointing to an index → swap indexes with zero downtime. |
| **Agentic retrieval / knowledge base** | Newer feature: an LLM plans multiple sub-queries over one or more "knowledge sources", runs them in parallel, merges results for agents. |
| **Replica** | Copy of the index for **query throughput + HA**. |
| **Partition** | Slice of storage for **index size + indexing throughput**. |
| **Search Unit (SU)** | Billing unit = replicas × partitions. |

---

## 3. Tiers & capacity

| Tier | Use | Notes |
|------|-----|-------|
| **Free** | Learning | Shared, 1 per subscription, small limits, no SLA |
| **Basic** | Small prod/dev | Up to 3 replicas; limited partitions |
| **Standard S1 / S2 / S3** | Most production | More storage per partition, up to 12 replicas × 12 partitions (≤ 36 SU) |
| **S3 HD** | Many small indexes (multi-tenant SaaS) | High index count |
| **Storage Optimized L1 / L2** | Very large, mostly-static data | Cheaper per GB, higher latency |

Exact storage/limits change (Microsoft raised limits for services created after 2024) — check current "service limits" page before sizing.

```
 SLA:
   1 replica   → no SLA
   2 replicas  → 99.9% read (queries)
   3+ replicas → 99.9% read + write (indexing)

 Scaling:
   queries slow / throttled (503) → add REPLICAS
   index full / indexing slow     → add PARTITIONS
   cost = SU × hourly tier price  (billed even when idle!)
```

Gotcha: **you can't change tier in place** for most moves (e.g. Basic → S1 historically required a new service + reindex; newer services support some upgrades). Pick tier deliberately.

---

## 4. RAG index design (typical)

```json
{
  "name": "docs-v2",
  "fields": [
    { "name": "chunk_id",   "type": "Edm.String", "key": true, "analyzer": "keyword" },
    { "name": "parent_id",  "type": "Edm.String", "filterable": true },
    { "name": "title",      "type": "Edm.String", "searchable": true },
    { "name": "chunk",      "type": "Edm.String", "searchable": true },
    { "name": "content_vector", "type": "Collection(Edm.Single)",
      "dimensions": 1536, "vectorSearchProfile": "hnsw-profile", "searchable": true },
    { "name": "allowed_groups", "type": "Collection(Edm.String)", "filterable": true },
    { "name": "source_url", "type": "Edm.String", "retrievable": true },
    { "name": "last_modified", "type": "Edm.DateTimeOffset", "filterable": true, "sortable": true }
  ]
}
```

Query (hybrid + semantic + security trimming):

```http
POST https://<svc>.search.windows.net/indexes/docs-v2/docs/search?api-version=2024-07-01
Authorization: Bearer <Entra token>

{
  "search": "how to rotate key vault secrets",
  "vectorQueries": [{ "kind": "text", "text": "how to rotate key vault secrets",
                      "fields": "content_vector", "k": 50 }],
  "queryType": "semantic",
  "semanticConfiguration": "default",
  "filter": "allowed_groups/any(g: search.in(g, 'grp-devops,grp-all'))",
  "top": 5,
  "select": "title,chunk,source_url"
}
```

(`"kind": "text"` works when the index has a **vectorizer**; otherwise send `"kind": "vector"` with your own embedding.)

---

## 5. Security & networking

```
 App (managed identity) ──► Search  (role: Search Index Data Reader)
 Indexer pipeline       ──► Search  (role: Search Index Data Contributor)
 Search MI ──► Storage  (Storage Blob Data Reader)        ← indexer reads docs
 Search MI ──► Foundry  (Cognitive Services OpenAI User)  ← embedding skill / vectorizer
 Admins                ──► Search  (Search Service Contributor = manage indexes, NOT query data)
```

- Enable **RBAC** (`authOptions: aadOrApiKey`) and eventually `disableLocalAuth: true`. Admin keys = full control; query keys = read-only — avoid both in prod.
- **Private endpoint** for inbound (`privatelink.search.windows.net`) + `publicNetworkAccess: disabled`.
- **Outbound** to private data sources: **shared private links** (managed private endpoints from Search to Storage/SQL/Cosmos/OpenAI) — must be **approved** on the target resource.
- Or "trusted service" exception on Storage firewall + Search managed identity.
- Search has no per-document ACL by default → implement **security trimming** with a filter field (groups) — or use newer native ACL/permission features for ADLS Gen2/SharePoint where available.
- CMK encryption for indexes if required (costs more, set at creation of index/synonym map).

---

## 6. Gotchas

1. **Billed per hour per SU even with zero traffic.** Dev/test services left running = waste. Free tier for learning.
2. **Index schema changes**: you can add fields, but can't change type/attributes of existing fields → create new index, reindex, swap alias.
3. **Embedding dimension mismatch** (`3072` vs `1536`) → indexing errors. Field `dimensions` must equal the model output.
4. **Indexer timeouts / partial failures** — check `az search ...`/REST `indexers/<name>/status`; failed docs are listed with errors. Large PDFs + OCR are slow; set `maxFailedItems` deliberately.
5. **Indexer + embedding model throttling** — skillset calls hit OpenAI TPM limits (429) during big reindexes. Raise TPM temporarily or slow batch size.
6. **Deleted source files remain in index** unless a deletion detection policy (soft delete column / blob soft delete / metadata flag) is configured.
7. **Shared private link not approved** → indexer fails with "403" or network errors.
8. **Semantic ranker** must be enabled on the service (`semanticSearch: free|standard`); free plan has a monthly query cap.
9. **Region choice** — semantic ranker, some vector features and agentic retrieval are not in every region; put Search near your Foundry resource to reduce latency.
10. **Replica count 1 = no SLA**; also index updates can briefly affect query latency on small services.
11. **API versions** — preview APIs carry new features (agentic retrieval) but change; pin GA versions in prod code.

---

## 7. Production scenario FAQ

**Q1. Queries started returning 503 during a sales event.**
Throttling — not enough replicas. Scale replicas (takes minutes, no downtime). Add autoscale logic (Logic App / Function on metrics) since Search has no native autoscale. Cache frequent queries at the app/APIM layer.

**Q2. HR documents appeared in answers for engineers.**
No security trimming. Add `allowed_groups` field populated at indexing time from source ACLs, filter by user's group claims at query time (server-side, never from client input). Re-index all docs. Audit logs via diagnostic settings.

**Q3. Need to change chunk size / embedding model on a live index with 2M docs.**
Blue/green: create `docs-v3` index + new indexer/skillset → reindex in background (watch OpenAI TPM) → validate quality with eval set → point alias `docs` to v3 → delete v2. App always queries the alias.

**Q4. Indexer runs successfully but new blobs aren't searchable.**
Check: indexer schedule & last run, change detection (blob `LastModified` — copying with preserved timestamps can skip), `maxFailedItems` hiding errors, field mappings, file type not supported, blob size over tier limit.

**Q5. How to size Search for a new RAG app?**
Estimate index size = docs × chunks/doc × (text size + vector size × dimensions × 4 bytes, before compression) + overhead. Pick tier by storage per partition; estimate QPS → replicas (start 2–3 for SLA). Load test with real queries including semantic ranker.

**Q6. Disaster recovery for Search?**
No built-in geo-replication. Options: second service in paired region with the same IaC + indexers pointed at geo-replicated sources (active-active), or rebuild from source on failover (RTO = reindex time). Store index definitions in git (as JSON) — data comes from sources.
