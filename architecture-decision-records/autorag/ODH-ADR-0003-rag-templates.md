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

This ADR defines the reusable AutoRAG templates selected during optimization: Agentic RAG on either a Vector or Graph store.

## Why

The template identifies the RAG store profile and the settings that are valid for an optimized pattern. A stable template ID lets AutoRAG preserve that distinction across optimization, indexing, and inference.

## Goals

* Define the first-release Agentic RAG composition and Vector vs Graph store profiles.
* Specify `template_id` as the store-profile discriminator for the Agentic RAG composition.
* Keep Vector and Graph pattern settings distinct.

## Non-Goals

* Detailed pipeline parameters and Vector RAG search-space values ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).
* Full `pattern.json`, indexing, or serving contracts ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md)).

## How

A template is the retrieve-and-generate composition that AutoRAG parameterizes during optimization. GAM explores allowed settings within that composition; it does not change the template or store profile.

## Templates

`template_id` is `agentic_rag` for the Vector profile and `agentic_graph_rag` for the Graph profile. It is derived from the selected store profile.

| `template_id` | Store profile | Compatible `provider_type` |
|---------------|---------------|----------------------------|
| `agentic_rag` | Vector | Vector-store provider |
| `agentic_graph_rag` | Graph | Graph-store provider |

Each optimized instance is a RAG pattern. `template_id` is required by Dashboard, indexing, and serving. The shared artifact envelope is in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md).

### Vector RAG

Vector patterns use `template_id: agentic_rag` and a Vector store binding. Their allowed chunking, retrieval, and generation settings are defined in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md).

### Graph RAG

Graph patterns use `template_id: agentic_graph_rag`, a Graph store binding, and Graph-only extraction and retrieval fields. Vector-only ranking fields do not apply. The graph-specific `pattern.json` contract is defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#graph-only-fields).

## Related

- [AutoRAG architecture](./ODH-ADR-0001-autorag.md)
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md)
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md)
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
