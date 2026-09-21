# Open Data Hub - AutoRAG RAG Templates

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-26 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-2357](https://redhat.atlassian.net/browse/RHAISTRAT-2357) · [RHAIRFE-2709](https://redhat.atlassian.net/browse/RHAIRFE-2709) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) |

## What

This ADR defines AutoRAG RAG templates: the reusable retrieve-and-generate blueprints parameterized during optimization. It covers simple RAG, Neo4j Graph RAG, and which template maps to which starter-kit path. Serving (Helm, one-click, notebooks, BFF test): [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation).

## Why

Template choice determines composition (retrieve/generate), infrastructure (vector store vs graph DB), optimization knobs, and deployment contracts. Graph RAG is **Neo4j-native** (`neo4j-graphrag`).

## Goals

* Document the simple retrieve-then-generate template and its optimization / inference contract
* Document Neo4j Graph RAG as the graph template (single engine)
* Record which template maps to which starter-kit path (serving details: ODH-ADR-0004)

## Non-Goals

* Relationship-Enriched RAG (vector-store-only entity/relation chunk enrichment) — removed from scope
* Alternate Graph RAG engines or dual-engine bake-offs
* Detailed pipeline parameter tables for simple-RAG presets (see ODH-ADR-0002)
* Full pattern.json field reference (see ODH-ADR-0004)
* Helm, one-click Agent Sandbox, and env mapping for serving a pattern (see ODH-ADR-0004)

## How

A **RAG template** is the reusable workflow blueprint AutoRAG parameterizes during optimization. GAM explores values inside a template (chunking, retrieval, generation, prompts); the template itself defines **how** retrieve and generate are composed.

## Table of contents

- [Overview](#overview)
- [Current RAG template](#current-rag-template)
- [Graph RAG template](#graph-rag-template)
  - [Prerequisites](#prerequisites)
  - [Neo4j GraphRAG](#neo4j-graphrag)
  - [LangChain / LangGraph (optional orchestration)](#langchain--langgraph-optional-orchestration)
  - [Pipeline integration](#pipeline-integration)
  - [Pattern artifacts](#pattern-artifacts)
  - [Optimization parameters by template](#optimization-parameters-by-template)
- [Template-to-deployment mapping](#template-to-deployment-mapping)
- [Related](#related)

---

## Overview

| Template | Composition | Infrastructure |
|----------|-------------|----------------|
| **Simple RAG** | Single retrieve → generate hop (`SimpleRAG`) | Milvus or PGVector |
| **Graph RAG** | Neo4j-native KG + hybrid/Cypher retrieval (`neo4j-graphrag`; optional LangGraph orchestration) | Neo4j (graph + vector + full-text) |

Optimized instances of a template are emitted as **RAG patterns** (`pattern.json`). See [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md).

Pipeline stage: **`rag_templates_optimization`** in the documents RAG optimization managed pipeline. Template selection is expected via `rag_template_type` (`simple` | `neo4j_graphrag`).

---

## Current RAG template

The AutoRAG simple template is a **single-turn retrieve-then-generate** pipeline (often called simple / standard RAG):

```text
question
   │
   ▼
┌──────────────┐     top-k chunks      ┌──────────────┐
│  Retriever   │ ───────────────────▶  │  Generator   │ ──▶ answer
│  (vector /   │                       │  (LLM +      │
│   hybrid)    │                       │   prompts)   │
└──────────────┘                       └──────────────┘
```

| Concern | Behavior |
|---------|----------|
| **Indexing** | Chunk → MaaS embed → write vector store ([experiment settings](./ODH-ADR-0002-experiment-settings.md)) |
| **Retrieval** | One query; `number_of_chunks`, `search_mode` (`vector` / `keyword` / `hybrid`), optional ranker |
| **Generation** | One LLM call with `system_message_text` / `user_message_text` / `context_template_text` over retrieved chunks |
| **Inference export** | `pattern.json` `settings` — retrieve-and-generate in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation) |
| **Optimization** | GAM over chunking × embedding × retrieval × generation |

**Naming note:** AutoRAG `hybrid` means dense + BM25 fused with RRF (or weighted fusion) over chunks. That is **not** Neo4j `HybridRetriever` / `HybridCypherRetriever` (vector + full-text, optionally plus Cypher).

What this template does **not** include today: multi-hop retrieval, tool-calling loops, entity/knowledge-graph traversal, or agent planners. Evaluation uses the metrics in [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md).

---

## Graph RAG template

**Graph RAG** is a template for multi-hop and relational questions where one-shot vector retrieval underperforms. It sits beside simple RAG inside the same Kubeflow DAG (Path A: ai4rag templates on `BaseRAGTemplate`), not as a chunk-only `BaseVectorStore` path.

The engine is **Neo4j GraphRAG** (`neo4j-graphrag`): schema-guided KG construction, Neo4j-native vector/hybrid/Cypher retrievers, and `GraphRAG.search`. LangChain/LangGraph is **optional** orchestration on top, not a peer engine.

```text
question
   │
   ▼
┌────────────────────────────────────────────────────────────┐
│  Graph RAG (Neo4j GraphRAG)                                │
│                                                            │
│   SimpleKGPipeline / GraphSchema                           │
│              ▼                                             │
│   Neo4j (graph + vector + full-text)                       │
│              ▼                                             │
│   HybridCypherRetriever (or VectorCypher / Text2Cypher)    │
│              ▼                                             │
│   GraphRAG.search                                          │
└────────────────────────────────────────────────────────────┘
   │
   ▼
 answer (+ graph / hybrid context)
```

| Concern | Direction |
|---------|-----------|
| **Store** | Neo4j only (graph + vector + full-text); profile `neo4j_hybrid_only` |
| **Indexing** | `extracted_text` → entity/relation extraction → Neo4j write; indexes are **not** shared with simple-RAG chunk collections |
| **Retrieval** | Neo4j-native vector / hybrid / Cypher retrievers — not a single vector hop |
| **Generation** | LLM grounded in graph/hybrid context; MaaS-backed embed/LLM adapters |
| **Optimization** | Initially fixed configs; later GAMOpt over retriever type / `top_k` / Cypher depth / schema constraints |
| **Export** | Durable `pattern.json` in the [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) envelope; Neo4j knobs live under `settings.retrieval` / `settings.store_binding` — reconnect after pod exit |

### Prerequisites

| Item | Notes |
|------|--------|
| **Graph store** | Neo4j (`db_secret_name`) |
| **MaaS** | Embedding, generation, and extraction model ids via `maas_secret_name` — no hardcoded vendor SDKs |
| **Corpus** | Same Docling → `extracted_text` and pipeline `test_data` as simple RAG |
| **vs simple RAG** | Graph RAG adds a template type; it does not replace the current LangChain vector store + MaaS embed/generate path |

Graph RAG requires a reachable Neo4j `db_secret_name`; Neo4j holds graph, vector, and full-text indexes. Simple RAG uses the same parameter for a Milvus or PGVector connection: [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#connections).

### Neo4j GraphRAG

Official [`neo4j-graphrag`](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html) for KG construction and Neo4j-native retrieval, then `GraphRAG(retriever=..., llm=...).search`. This path can stand alone for indexing + retrieval; LangChain/LangGraph is **optional** on top (see below).

| Layer | Role |
|-------|------|
| **KG build** | `SimpleKGPipeline` / `GraphSchema` from `extracted_text` (+ vector and full-text indexes in Neo4j) |
| **Retrievers** | See table below |
| **Generation** | `neo4j_graphrag.generation.GraphRAG` |
| **Neo4j** | Single DB for graph + vector + full-text — profile `neo4j_hybrid_only` |
| **MaaS** | Adapters implementing Neo4j `LLM` / `Embedder` interfaces |

| Retriever | Role |
|-----------|------|
| `VectorRetriever` | Vector ANN |
| `VectorCypherRetriever` | Vector hit → Cypher neighborhood |
| `HybridRetriever` | Vector + full-text |
| `HybridCypherRetriever` | Hybrid + Cypher traversal (strong default candidate) |
| `Text2CypherRetriever` | NL → Cypher |
| `ToolsRetriever` | Route across retrievers |

```text
docs ──▶ SimpleKGPipeline / GraphSchema ──▶ Neo4j (graph + vector + full-text)
question ──▶ HybridCypherRetriever (or VectorCypher / Text2Cypher / …)
                └── GraphRAG.search ──▶ answer
```

| Concern | Detail |
|---------|--------|
| **Storage** | Neo4j-only (`neo4j_hybrid_only`) |
| **Initial config** | Fixed retriever + `top_k` / retrieval Cypher; later GAMOpt: retriever type, Cypher depth/LIMIT, schema constraints |
| **ai4rag type** | `Neo4jGraphRAGTemplate` |
| **pattern.json** | Same envelope as [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md); `settings.retrieval.method` is a Neo4j retriever (`hybrid_cypher` \| `vector_cypher` \| `text2cypher`); `settings.store_binding.provider_type` is `neo4j`; embedding model locked at index time |
| **Deps** | `neo4j-graphrag`, `neo4j`; optional `langgraph` for agentic orchestration |

Prefer native Neo4j retrievers over reimplementing the same patterns with LangChain `LLMGraphTransformer` + `Neo4jVector` + custom structured retrievers.

### LangChain / LangGraph (optional orchestration)

LangChain chat models are an optional LLM adapter into Neo4j `GraphRAG` — retrieval itself does not require LangChain.

| Need | Use |
|------|-----|
| Neo4j indexing + retrieval | **`neo4j-graphrag` only** |
| Multi-step agents, HITL, tools outside Neo4j | **LangGraph** calling Neo4j retrievers |
| Org-wide LangChain / LangSmith standard | Thin LangChain LLM wrapper into Neo4j `GraphRAG` |

```text
Track: Neo4jGraphRAGTemplate + neo4j-graphrag retrievers
         │
         └── optional LangGraph router / multi-tool agents
```

### Pipeline integration

```text
documents_discovery → search_space_preparation → text_extraction
                                   ↓
             models_pre_selector
                                   ↓
             rag_templates_optimization
             (SimpleRAG | Neo4jGraphRAGTemplate)
                                   ↓
                        leaderboard (built in rag_templates_optimization)
                   (pattern_type: simple | neo4j_graphrag)
```

| Layer | Change |
|-------|--------|
| **ai4rag** | Templates on `BaseRAGTemplate`; `AI4RAGExperiment.rag_template_type`; MaaS bridges; extra `neo4j-graphrag` |
| **KFP** | `rag_template_type`, `db_secret_name`, graph search-space dims, leaderboard `pattern_type` columns |
| **Path A** | Preferred — template inside the existing documents RAG optimization DAG |
| **Template contract** | `build_index` / `generate` / `generate_stream` — not chunk-only vector store |
| **Key params** | `rag_template_type`, `storage_profile`, `maas_secret_name`, `embedding_model_id`, `generation_model_id`, `extraction_model_id`, `db_secret_name`, `retriever_type` |
| **Acceptance** | Reconnect via `pattern.json` after pod exit; scores on pipeline `test_data` |
| **Out of scope (v1)** | Sharing simple-RAG chunk collections as Neo4j indexes; LangChain-first Neo4j retrieval (use `neo4j-graphrag` retrievers) |

### Pattern artifacts

Neo4j GraphRAG pattern sketch (same top-level envelope as simple RAG; extra knobs under `settings`):

```json
{
  "name": "PatternGraph1",
  "settings": {
    "store_binding": {
      "provider_type": "neo4j",
      "collection_name": "<neo4j-index>"
    },
    "retrieval": {
      "method": "hybrid_cypher",
      "number_of_chunks": 10
    },
    "generation": {
      "model_id": "<maas>",
      "temperature": 0.2,
      "max_completion_tokens": 2048,
      "system_message_text": "<system>",
      "user_message_text": "<user>",
      "context_template_text": "<context>"
    }
  },
  "evaluation": { "metrics": [] },
  "indexing": { "pipeline_spec": {} }
}
```

Graph RAG patterns use the same `evaluation` / `indexing` contract as simple RAG for leaderboard comparison. Query-time retriever choice is `settings.retrieval.method` (for example `hybrid_cypher`).

### Optimization parameters by template

AutoRAG / GAM should explore knobs that change **retrieval or generation quality** without exploding index-rebuild cost. Prefer a **phased** search space: fix heavy index settings first, then optimize query-time dims; widen later once baselines exist. Full simple-RAG dims: [experiment settings](./ODH-ADR-0002-experiment-settings.md).

| Template | Optimize (high value) | Optimize later / conditional | Usually **fix** (not GAM dims) |
|----------|----------------------|------------------------------|----------------------------------|
| **Current (simple) RAG** | Search space in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md) | Ranker / embedding allow-list | `db_secret_name`, Docling preset, corpus / `test_data`, MaaS Connection |
| **Neo4j Graph RAG** | **Query-time:** `retriever_type` (`hybrid_cypher` / `vector_cypher` / `text2cypher` / …), `top_k`, Cypher hop depth / `LIMIT`; **gen:** model + temperature | **Index-time:** `GraphSchema` strictness / entity types; embedding model; full-text vs vector index params | `storage_profile: neo4j_hybrid_only`, `db_secret_name`, LangGraph on/off (orchestration, not core RAG score) |

**Shared rules**

| Rule | Why |
|------|-----|
| Index-time dims (chunk size, embed model, extraction model, KG schema) **rebuild** the store — sample sparsely or as an outer loop | Dominates wall-clock and $ |
| Query-time dims (`retriever_type`, `top_k`, rerank, gen temp) are cheap to sweep on a frozen index | Best GAM ROI |
| Embedding / extraction model ids are categorical and expensive — small allow-lists | Avoid combinatorial blow-up |
| Do not GAM-optimize storage backend choice in the same trial as quality knobs | Confounds quality with infra |

**Suggested v1 Graph RAG search space (frozen index, then query+gen):**

```text
Neo4j GraphRAG: retriever_type ∈ {hybrid_cypher, vector_cypher}  ×  top_k  ×  cypher_depth  ×  gen temp
(+ optional outer: extraction_model_id or schema — few values only)
```

Start Graph RAG with **fixed** `retriever_type=hybrid_cypher` for smoke tests; turn on GAM once reconnect + leaderboard work.

---

## Template-to-deployment mapping

How to **serve** a pattern (notebook, zip + Helm, one-click Agent Sandbox, BFF test endpoint): [ODH-ADR-0004 — Retrieve and generation](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation). Zip/Helm: [Agentic Starter-kit](./ODH-ADR-0004-rag-pattern-inference.md#agentic-starter-kit). One-click: [One-click Deployment](./ODH-ADR-0004-rag-pattern-inference.md#one-click-deployment). Test: [Test endpoint](./ODH-ADR-0004-rag-pattern-inference.md#test-endpoint).

This ADR only records **which template** maps to which starter-kit path.

| Template | Starter-kit path | Notes |
|----------|-------------------|-------|
| **Simple RAG** | [`agentic_rag`](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) via **`starter_kit.zip`** | Runtime contract in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation) |
| **Neo4j Graph RAG** | Starter-kit template with `neo4j-graphrag` retrievers + LangGraph | Requires Neo4j (`db_secret_name`); distinct from the simple-RAG template |

**Graph RAG** uses an agent template that replaces vector-only retrieval with Neo4j Cypher retrievers.

---

## Related

- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — search-space dimensions for the current template
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) — artifacts, `pattern.json`, retrieve / generate, Helm, one-click Agent Sandbox, BFF test endpoint
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — benchmark metrics
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
- [neo4j-graphrag — User Guide: RAG](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
- [neo4j-graphrag — KG Builder](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_kg_builder.html)
- Optional orchestration: [LangGraph + Neo4j](https://neo4j.com/blog/developer/neo4j-graphrag-workflow-langchain-langgraph/)
- [Agentic starter-kits](https://github.com/red-hat-data-services/agentic-starter-kits) — deployment templates for RAG agents
- [Agentic RAG template](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) — LangGraph-based RAG agent (simple-RAG zip)
