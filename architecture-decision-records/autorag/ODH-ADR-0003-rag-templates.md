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

This ADR defines the reusable AutoRAG templates selected during optimization: Simple RAG and Neo4j Graph RAG.

## Why

The template determines the retrieval composition, database backend, optimization dimensions, and deployment artifact. A single template contract lets AutoRAG compare patterns while preserving the infrastructure each retrieval approach needs.

## Goals

* Define Simple RAG and Neo4j Graph RAG.
* Specify the database and retrieval contracts for each template.
* Map templates to deployment artifacts.

## Non-Goals

* Alternate Graph RAG engines or dual-engine comparisons.
* Detailed pipeline parameters and Simple RAG search-space values ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).
* Full `pattern.json`, indexing, or serving contracts ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md)).

## How

A template is the retrieve-and-generate blueprint that AutoRAG parameterizes during optimization. GAM explores values within that blueprint; it does not change the template's database or retrieval architecture.

## Templates

| Template | Composition | Database |
|----------|-------------|----------|
| **Simple RAG** | Single retrieve → generate hop over document chunks. | Milvus or PGVector. |
| **Neo4j Graph RAG** | Knowledge-graph construction followed by Neo4j vector, hybrid, or Cypher retrieval. | Neo4j graph, vector, and full-text indexes. |

Each optimized instance is emitted as a RAG pattern. The shared artifact envelope and deployment contract are in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md).

### Simple RAG

Simple RAG retrieves document chunks, then grounds one generation call in those chunks.

| Concern | Contract |
|---------|----------|
| Indexing | Chunk → MaaS embeddings → vector store. |
| Retrieval | `number_of_chunks`, `search_mode` (`vector`, `keyword`, `hybrid`), and optional ranker. |
| Generation | MaaS model and prompt settings from `settings.generation`. |
| Pattern | `settings.store_binding` identifies the selected Milvus or PGVector collection. |
| Optimization | Chunking, embedding, retrieval, and generation dimensions from [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md). |

### Neo4j Graph RAG

Neo4j Graph RAG uses `neo4j-graphrag` for knowledge-graph construction and Neo4j-native retrieval. `db_secret_name` selects the Neo4j connection; it is the same generic database parameter used by Simple RAG.

| Concern | Contract |
|---------|----------|
| Indexing | Extract entities and relations from the document corpus into Neo4j graph, vector, and full-text indexes. |
| Retrieval | `VectorRetriever`, `HybridRetriever`, `VectorCypherRetriever`, `HybridCypherRetriever`, or `Text2CypherRetriever`. |
| Generation | MaaS-backed generation grounded in graph or hybrid context. |
| Pattern | `settings.store_binding.provider_type: neo4j`; `settings.retrieval.method` records the selected retriever. |
| Optimization | Query-time retriever type, top-k, Cypher depth or limit, and generation settings. Index-changing settings use constrained allow-lists. |

Graph RAG does not share Simple RAG chunk collections. LangGraph may orchestrate multi-step or tool-based flows around Neo4j retrieval; it is not a separate retrieval engine.

## Template-to-deployment mapping

Serving behavior is defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation).

| Template | Deployment artifact |
|----------|---------------------|
| **Simple RAG** | [`agentic_rag`](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) via the run-level `starter_kit.zip`. |
| **Neo4j Graph RAG** | Graph RAG starter-kit template using `neo4j-graphrag` retrievers and LangGraph orchestration. |

## Related

- [AutoRAG architecture](./ODH-ADR-0001-autorag.md)
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md)
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md)
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [Neo4j GraphRAG user guide](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
