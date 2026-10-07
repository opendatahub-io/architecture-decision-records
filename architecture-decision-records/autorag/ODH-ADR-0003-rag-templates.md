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
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004-rag-pattern-inference](./ODH-ADR-0004-rag-pattern-inference.md) |

## What

This ADR defines the reusable AutoRAG templates selected during optimization: Simple RAG and Agentic RAG, each on a Vector or Graph store.

## Why

The template determines the retrieval composition, database backend, optimization dimensions, and deployment artifact. A single template contract lets AutoRAG compare patterns while preserving the infrastructure each retrieval approach needs.

## Goals

* Define Simple RAG and Agentic RAG compositions, and Vector vs Graph store profiles.
* Specify `template_id` as the unique composition × store discriminator.
* Map templates to deployment artifacts.

## Non-Goals

* Alternate Graph RAG engines or dual-engine comparisons.
* Detailed pipeline parameters and Vector RAG search-space values ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).
* Full `pattern.json`, indexing, or serving contracts ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md)).

## How

A template is the retrieve-and-generate composition that AutoRAG parameterizes during optimization. GAM explores chunking, retrieval, and generation values within that composition; it does not change the template class or the store architecture.

ai4rag implements two `BaseRAGTemplate` subclasses: [`SimpleRAG`](https://ibm.github.io/ai4rag/architecture/rag-components/#simplerag) (one retrieve-then-generate pass) and [`AgenticRAG`](https://ibm.github.io/ai4rag/architecture/rag-components/#agenticrag) (LangChain agent that may rewrite the query and retrieve again). `AI4RAGExperiment` defaults to `AgenticRAG`. Graph RAG is not a third class: it is the same subclass on Neo4j with `search_mode: graph` plus Graph expansion fields. `settings.retrieval.method` (`simple` vs window) is the retriever strategy, not the template class.

## Templates

`template_id` is `{simple|agentic}_rag` on a vector store, or `{simple|agentic}_graph_rag` on Neo4j. Emit it from the constructed `rag_template` class plus `store_binding.provider_type`.

| `template_id` | ai4rag class | Store | Compatible `provider_type` |
|---------------|--------------|-------|----------------------------|
| `simple_rag` | `SimpleRAG` | Vector | `milvus`, `pgvector` |
| `agentic_rag` | `AgenticRAG` | Vector | `milvus`, `pgvector` |
| `simple_graph_rag` | `SimpleRAG` | Graph | `neo4j` |
| `agentic_graph_rag` | `AgenticRAG` | Graph | `neo4j` |

Each optimized instance is a RAG pattern. `template_id` is required and is the discriminator for Dashboard, indexing, and (when present) serving. Do not infer composition from `provider_type` (both Simple and Agentic run on Vector or Graph). Do not infer it from `retrieval.method`. The shared artifact envelope is in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md).

### Vector RAG

Vector RAG retrieves document chunks, then grounds one generation call in those chunks.

| Concern | Contract |
|---------|----------|
| Indexing | Chunk → MaaS embeddings → vector store. |
| Retrieval | `number_of_chunks`, `search_mode` (`vector`, `keyword`, `hybrid`), and optional ranker. |
| Generation | MaaS model and prompt settings from `settings.generation`. |
| Pattern | `template_id` is `simple_rag` or `agentic_rag`. `settings.store_binding` identifies the selected Milvus or PGVector collection. |
| Optimization | Chunking, embedding, retrieval, and generation dimensions from [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md). |

### Graph RAG

Graph RAG currently uses Neo4j, through `neo4j-graphrag`, for knowledge-graph construction and retrieval. `db_secret_name` selects the Neo4j connection; it is the same generic database parameter used by Vector RAG.

| Concern | Contract |
|---------|----------|
| Indexing | Extract entities and relations from the document corpus into Neo4j graph, vector, and full-text indexes. |
| Retrieval | `search_mode: graph`. Neighbor expansion fields apply only in this mode ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#graph-only-fields)). Ranker fields do not apply. |
| Generation | MaaS-backed generation grounded in graph or hybrid context. |
| Pattern | `template_id` is `simple_graph_rag` or `agentic_graph_rag`; `settings.store_binding.provider_type: neo4j`. Extra `pattern.json` keys: [ODH-ADR-0004 Graph-only fields](./ODH-ADR-0004-rag-pattern-inference.md#graph-only-fields). |
| Optimization | Query-time retriever type, top-k, Cypher depth or limit, and generation settings. Index-changing settings use constrained allow-lists. |

Graph RAG does not share Vector RAG chunk collections. LangGraph may orchestrate multi-step or tool-based flows around Neo4j retrieval; it is not a separate retrieval engine.

## Template-to-deployment mapping

Serving behavior is defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation).

| `template_id` | Deployment artifact |
|---------------|---------------------|
| `agentic_rag` | [`agentic_rag`](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) via the run-level `starter_kit.zip`. |
| `simple_rag` | Same Responses retrieve-then-generate contract without the rewrite loop. |
| `agentic_graph_rag`, `simple_graph_rag` | Graph starter-kit using `neo4j-graphrag` retrievers. |

## Related

- [AutoRAG architecture](./ODH-ADR-0001-autorag.md)
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md)
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md)
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [Neo4j GraphRAG user guide](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
