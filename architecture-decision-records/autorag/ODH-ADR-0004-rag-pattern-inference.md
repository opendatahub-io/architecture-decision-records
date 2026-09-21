# Open Data Hub - AutoRAG Pattern Inference

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1846](https://redhat.atlassian.net/browse/RHAISTRAT-1846) · [RHAISTRAT-1731](https://redhat.atlassian.net/browse/RHAISTRAT-1731) · [RHAISTRAT-1724](https://redhat.atlassian.net/browse/RHAISTRAT-1724) · [RHAISTRAT-1424](https://redhat.atlassian.net/browse/RHAISTRAT-1424) · [RHAISTRAT-2623](https://redhat.atlassian.net/browse/RHAISTRAT-2623) · [RHOAIENG-88692](https://redhat.atlassian.net/browse/RHOAIENG-88692) · [RHOAIENG-90017](https://redhat.atlassian.net/browse/RHOAIENG-90017) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) |

## What

This ADR documents AutoRAG pattern artifacts after optimization: the pattern.json schema, production retrieve-and-generate via MaaS through a Responses-compatible API, three consumers of that contract (inference notebook, starter-kit zip with Helm, one-click Agent Sandbox), the AutoRAG BFF test endpoint, and full-corpus index building through the managed documents indexing pipeline.

## Why

Optimized configurations must be portable across optimization, indexing, and inference. A durable pattern contract lets Dashboard and APIs select a winning pattern, rebuild the production index, and reconstruct retrieve-and-generate from `settings` (the same chunking, retrieval, and generation fields used during benchmarking). There is no frozen request-body template in `pattern.json`.

## Goals

* Define the target pattern.json schema (`settings`, `indexing`, `inference`, `evaluation`)
* Document how request-level generation and retrieval settings drive MaaS and LangChain retrieval
* Document the inference notebook, parameterized starter-kit zip, Helm / BuildConfig deploy, and one-click Agent Sandbox
* Document the AutoRAG BFF test endpoint (retrieve-and-generate from `pattern.json`; not the agent API)
* Document indexing.pipeline_spec for the managed documents-indexing-pipeline

## Non-Goals

* Metric catalog and score computation (see ODH-ADR-0005)
* Graph RAG pattern storage profiles beyond sketches in ODH-ADR-0003

## How

The sections below define the artifact, inference, and indexing contracts for an optimized pattern.

## Table of contents

- [Optimization pipeline](#optimization-pipeline)
- [Pattern artifacts](#pattern-artifacts)
- [pattern.json](#patternjson)
  - [Example pattern.json](#example-patternjson)
- [Retrieve and generation](#retrieve-and-generation)
  - [Inference notebook](#inference-notebook)
  - [Agentic Starter-kit](#agentic-starter-kit)
  - [One-click Deployment](#one-click-deployment)
  - [Test endpoint](#test-endpoint)
- [Index building](#index-building)
- [Related](#related)

---

## Optimization pipeline

The **[`documents_rag_optimization_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** runs **`rag_templates_optimization`** to search RAG configurations and score each candidate on a benchmark (up to 1 GB document sample). Per-pattern outputs land under **`rag_patterns/<pattern_subdir>/`**. Run-level artifacts, including **`starter_kit.zip`** and the pipeline-wide HTML leaderboard, land at `<bucket>/<pipeline-name>/<run-id>/` in DSPA storage.

Each **`pattern.json`** captures optimized **`settings`**, **`indexing`** (pipeline spec), **`inference`** (`runtime_spec`), and **`evaluation`** results. The optimization run generates one **`starter_kit.zip`**, seeded with the best pattern's defaults (see [Agentic Starter-kit](#agentic-starter-kit)). Index building processes the **full document corpus** (union of `input_data_keys` locations) into the store the pattern queries at inference time. Responses-compatible requests use `settings` as defaults. **`inference.runtime_spec`** defines the deployment image and credential Connections.

---

## Pattern artifacts

Canonical pattern and run artifact inventory. Sibling ADRs link here instead of repeating this table. Row-level score schema: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#evaluation_resultsjson). Zip Helm and one-click serving: [Retrieve and generation](#retrieve-and-generation). MLflow pointers: [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md).

| Artifact | Purpose |
|----------|---------|
| `pattern.json` | Authoritative record: `name`, `settings`, `indexing`, `inference`, `evaluation`, `iteration`, `max_combinations`, `duration_seconds` |
| `starter_kit.zip` | One per optimization run. The [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) uses the best pattern as defaults and accepts Responses-compatible request overrides. |
| `indexing_notebook.ipynb`, `inference_notebook.ipynb` | Parameterized notebooks: full-corpus index vs retrieve-and-generate with a sample query |
| `evaluation_results.json` | Per-question detail ([`evaluation_results.json`](./ODH-ADR-0005-rag-pattern-evaluation.md#evaluation_resultsjson)) |

---

## pattern.json

```text
pattern.json
├── name, iteration, max_combinations, duration_seconds
├── settings
│   ├── store_binding (provider_type, collection_name)
│   ├── chunking (method, chunk_size, chunk_overlap, include_metadata)
│   ├── embedding (model_id, embedding_params)
│   ├── retrieval (method, number_of_chunks, search_mode, ranker_strategy, ranker_alpha)
│   └── generation (model_id, temperature, max_completion_tokens,
│                   context_template_text, user_message_text,
│                   system_message_text, language)
├── indexing
│   └── pipeline_spec
│       ├── pipeline_name
│       ├── parameters
│       └── overrides_allowed
├── inference
│   └── runtime_spec
│       ├── framework
│       ├── protocol
│       ├── image
│       └── connections (maas_secret_name, db_secret_name)
└── evaluation
    └── metrics[]
        ├── evaluator (unitxt | ragas | custom), name, description, scores (mean, ci_low, ci_high)
        ├── model_id (ragas entries when recorded)
        └── optimization_metric: true (exactly one entry — GAM objective)
```

| Field                                                       | Description                                                                                                                                                                                                                                                                                                                                       |
|-------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`, `iteration`, `max_combinations`, `duration_seconds` | Pattern identity, GAM iteration, search-space size, wall time                                                                                                                                                                                                                                                                                     |
| `settings`                                                  | Optimized RAG config: `store_binding` (`provider_type`, `collection_name`), `chunking` (incl. `include_metadata`), `embedding`, `retrieval` (`method`, `number_of_chunks`, `search_mode`, ranker fields), `generation` (model, sampling, `context_template_text` / `user_message_text` / `system_message_text`, `language` `{code, name}`) |
| `indexing.pipeline_spec`                                    | Managed indexing pipeline inputs — [Index building](#index-building)                                                                                                                                                                                                                                                                              |
| `inference.runtime_spec`                                    | Agent deployment: `framework`, `protocol`, `image`, and credential `connections` — [One-click Deployment](#one-click-deployment)                                                                                                                                                                                                                  |
| `evaluation`                                                | `metrics[]` aggregates. Catalog, evaluators, and GAM flag: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md)                                                                                                                                                                                                                               |

GAM ranks patterns by the evaluator-qualified pipeline [`optimization_metric`](./ODH-ADR-0002-experiment-settings.md) ID. The matching `evaluation.metrics[]` entry is marked `optimization_metric: true`; its `scores.mean` is the pattern objective score ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#optimization_metric)).

---

### Example pattern.json

```json
{
  "name": "Pattern1",
  "max_combinations": 90,
  "evaluation": {
    "metrics": [
      {
        "name": "faithfulness",
        "evaluator": "unitxt",
        "scores": {
          "mean": 0.9063,
          "ci_low": 0.8806,
          "ci_high": 0.9292
        }
      },
      {
        "name": "overall_score",
        "evaluator": "custom",
        "description": "Equal-weight mean of active evaluator metrics.",
        "scores": {
          "mean": 0.8806,
          "ci_low": 0.7844,
          "ci_high": 0.936
        },
        "optimization_metric": true
      }
    ]
  },
  "duration_seconds": 48,
  "settings": {
    "store_binding": {
      "provider_type": "milvus",
      "collection_name": "ai4rag_20260824161307_t2pxa4s3"
    },
    "chunking": {
      "method": "hybrid",
      "chunk_size": 512,
      "chunk_overlap": 0,
      "include_metadata": true
    },
    "embedding": {
      "model_id": "publishers/ai-eng-cracow/models/redhataibge-m3",
      "embedding_params": {
        "embedding_dimension": 1024,
        "context_length": 1015
      }
    },
    "retrieval": {
      "method": "simple",
      "number_of_chunks": 5,
      "search_mode": "hybrid",
      "ranker_strategy": "weighted",
      "ranker_alpha": 0.5
    },
    "generation": {
      "model_id": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic",
      "temperature": 0.2,
      "max_completion_tokens": 2048,
      "context_template_text": "Document {doc_number}:\n{document}",
      "user_message_text": "\nContext:\n{reference_documents}\n\nQuestion: {question}\nRespond exclusively in English, regardless of any other language used in the provided context. You MUST respond in English.",
      "system_message_text": "Please answer the user's question based solely on the provided context documents. If the question cannot be answered from the provided context, say you cannot answer. Your answer should be concise.",
      "language": {
        "code": "en",
        "name": "English"
      }
    }
  },
  "iteration": 0,
  "indexing": {
    "pipeline_spec": {
      "pipeline_name": "documents-indexing-pipeline",
      "parameters": {
        "maas_secret_name": "maas",
        "db_secret_name": "milvus",
        "input_data_secret_name": "minio",
        "input_data_bucket_name": "jwalaszc-bucket",
        "input_data_keys": [
          "rh_summit_2026/documents",
          "rh_summit_2026/policies"
        ],
        "batch_size": 20,
        "provider_type": "milvus",
        "collection_name": "ai4rag_20260824161307_t2pxa4s3",
        "embedding_model_id": "publishers/ai-eng-cracow/models/redhataibge-m3",
        "embedding_params": {
          "embedding_dimension": 1024,
          "context_length": 1015
        },
        "chunking_method": "hybrid",
        "chunk_size": 512,
        "chunk_overlap": 0
      },
      "overrides_allowed": [
        "input_data_secret_name",
        "input_data_bucket_name",
        "input_data_keys",
        "collection_name",
        "batch_size"
      ]
    }
  },
  "inference": {
    "runtime_spec": {
      "framework": "langgraph",
      "protocol": "responses",
      "image": "quay.io/opendatahub/odh-autorag-inference:odh-stable",
      "connections": {
        "maas_secret_name": "maas",
        "db_secret_name": "milvus"
      }
    }
  }
}
```

---

## Retrieve and generation

Optimization and production use **MaaS** for Responses-compatible generation and embeddings (indexing and query-time vector search). For simple RAG, retrieval uses the LangChain vector-store adapter for `store_binding.provider_type` / `collection_name` (Milvus or PGVector credentials from `db_secret_name`).

`settings` supplies defaults. A Responses-compatible request carries nonsecret generation, store, and retrieval values; it may override allowed defaults for that request. `connections` are deployment-only and are never accepted in the request.

```json
{
  "model": "publishers/ai-eng-cracow/models/qwen3-8b-fp8-dynamic",
  "input": "What is the policy for refunds?",
  "instructions": "Answer only from the retrieved documents.",
  "temperature": 0.2,
  "max_output_tokens": 2048,
  "tools": [{
    "type": "autorag_retrieval",
    "store_binding": {
      "provider_type": "milvus",
      "collection_name": "ai4rag_20260824161307_t2pxa4s3"
    },
    "embedding_model": "publishers/ai-eng-cracow/models/redhataibge-m3",
    "retrieval": {
      "method": "simple",
      "number_of_chunks": 5,
      "search_mode": "hybrid",
      "ranker_strategy": "weighted",
      "ranker_alpha": 0.5
    }
  }]
}
```

`autorag_retrieval` is an AutoRAG extension to the Responses API shape; it is not an OpenAI built-in tool. It retrieves chunks, then provides them to generation. The full-corpus index must exist first ([Index building](#index-building)). All consumers use this request contract. The [Test endpoint](#test-endpoint) runs it **inside the AutoRAG BFF**; it is not a caller of the agent API.

| Path | Audience | Intent |
|------|----------|--------|
| Inference notebook | Data scientist | Inspect retrieve → generate in Jupyter with a sample query |
| Agentic starter kit | Developer | Download and customize the generated deployment artifact |
| One-click deployment | Operator / Dashboard | Deploy the default application from `inference.runtime_spec` |
| BFF test endpoint | Dashboard user | Test a pattern interactively; not production serving |

### Inference notebook

`inference_notebook.ipynb` sits next to `pattern.json` under `rag_patterns/<pattern_name>/`. It is a generated workbench artifact for inspecting retrieval and generation with the pattern's MaaS and database Connections; it is not a server. `indexing_notebook.ipynb` builds the full-corpus index.

### Agentic Starter-kit

`starter_kit.zip` is one generated, editable deployment artifact per optimization run. It is seeded with the best pattern's settings as defaults; Responses-compatible requests can override the permitted nonsecret generation, store, and retrieval values. It provides `POST /responses`, `GET /health`, local-run assets, and Helm deployment assets. Its operational instructions are maintained with the [agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag).

### One-click Deployment

One-click deployment starts the default RAG application from `inference.runtime_spec`, using the prebuilt inference image and project Connections. It injects only the MaaS and database Connections; the request supplies nonsecret model, store, and retrieval settings. The Dashboard deployment guide owns runtime, Sandbox, and lifecycle details.

### Test endpoint

The AutoRAG BFF runs interactive Responses-compatible retrieve-and-generate tests against a selected pattern's store and MaaS connection. It is a Dashboard test path, not the production agent API; request and response details are owned by the [AutoRAG BFF](https://github.com/opendatahub-io/odh-dashboard/tree/main/packages/autorag/bff).

---

## Index building

Index building populates the production vector store via the managed **`documents-indexing-pipeline`** ([`documents_indexing_pipeline`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/data_processing/autorag/documents_indexing_pipeline/pipeline.py)), registered in the AI Pipelines catalog. One pipeline definition serves all patterns; per-pattern values come from **`indexing.pipeline_spec`**.

| `pipeline_spec` field | Role |
|-----------------------|------|
| `pipeline_name` | Managed catalog name (e.g. `documents-indexing-pipeline`) |
| `parameters` | Pre-filled from optimization run + pattern `settings` |
| `overrides_allowed` | Keys the UI may expose for user override at submit time |

**Parameter sources:** optimization run → `maas_secret_name`, `db_secret_name`, `input_data_*`; pattern `settings` → embedding (`embedding_model_id`, `embedding_params`), chunking, `collection_name` / `provider_type`. Secret fields are **names only** (Kubernetes Secret references).

`parameters.input_data_keys` is the same **`list[str]`** as the optimization run (1–10 object keys or prefixes, same Connection/bucket). Production indexing uses that full list so the production corpus matches optimization. A one-element list is a single location. Corpus contract: [ODH-ADR-0002 — Corpus locations](./ODH-ADR-0002-experiment-settings.md#corpus-locations).

**Workflow:** optimization completes → user selects pattern → read `pipeline_spec` → resolve managed pipeline → pre-fill run form → user confirms/overrides → submit → full corpus indexed → [retrieve and generation](#retrieve-and-generation) ready.

**Pipeline steps:** load inputs → document discovery/extraction → chunking → embedding (MaaS) → vector store write → validation/logging. Observable via KFP; re-runnable when documents or overrides change.

---

## Related

- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
- [RAG templates](./ODH-ADR-0003-rag-templates.md) — simple vs Neo4j Graph RAG templates; which kit path each maps to
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — pipeline parameters, corpus list, benchmark JSON, presets, chunking, retrieval, HPO MaaS / database Connections
- [MLflow integration](./ODH-ADR-0006-mlflow-integration.md) — tracking pointers to pattern artifacts
- [RHOAIENG-90017](https://redhat.atlassian.net/browse/RHOAIENG-90017) — AutoRAG BFF pattern test
- [opendatahub-io/odh-dashboard AutoRAG BFF](https://github.com/opendatahub-io/odh-dashboard/tree/main/packages/autorag/bff)
- [Agentic RAG starter-kit](https://github.com/red-hat-data-services/agentic-starter-kits/tree/main/agents/langgraph/templates/agentic_rag) — zip generation template
- [Agent Sandbox](https://agent-sandbox.sigs.k8s.io/docs/) — one-click `Sandbox` / `SandboxTemplate` / `SandboxClaim` / `SandboxWarmPool`
- [Agent Sandbox API](https://agent-sandbox.sigs.k8s.io/docs/api/) — `SandboxClaim.spec.env` cold-start behavior
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
