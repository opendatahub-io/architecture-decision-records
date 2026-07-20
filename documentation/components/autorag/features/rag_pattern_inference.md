# RAG pattern inference

This page describes **AutoRAG patterns** after optimization: the **`pattern.json`** schema, **retrieve and generation** via **`POST /v1/responses`**, and **index building** into the production vector store.

## Table of contents

- [Optimization pipeline](#optimization-pipeline)
- [pattern.json schema](#patternjson-schema)
  - [Target schema](#target-schema)
- [Example pattern.json](#example-patternjson)
- [Retrieve and generation](#retrieve-and-generation)
- [Index building](#index-building)
- [Related](#related)

---

## Optimization pipeline

The **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** runs **`rag_templates_optimization`** to search RAG configurations and score each candidate on a benchmark (up to 1 GB document sample). Outputs land under **`rag_patterns/<pattern_subdir>/`** in DSPA storage (`<bucket>/<pipeline-name>/<run-id>/…`) plus a pipeline-wide HTML leaderboard.

Each **`pattern.json`** captures optimized **settings**, **`inference`** (responses template), **`indexing`** (pipeline spec), and **`evaluation`** results. Index building processes the **full document corpus** into the vector store the pattern queries at inference time.

| Artifact | Purpose |
|----------|---------|
| `pattern.json` | Authoritative record: `settings`, `inference`, `indexing`, `evaluation`, timing |
| `indexing_notebook.ipynb`, `inference_notebook.ipynb` | Parameterized notebooks for the pattern |
| `evaluation_results.json` | Per-question detail ([`evaluation_results.json`](./rag_pattern_evaluation.md#evaluation_resultsjson)) |

---

## pattern.json schema

### Target schema

```text
pattern.json
├── name, iteration, max_combinations, duration_seconds
├── settings
│   ├── vector_store_binding
│   ├── chunking, embedding, retrieval, generation
│   └── generation.language (optional, {code, name})
├── inference
│   └── responses_template
├── indexing
│   └── pipeline_spec
│       ├── pipeline_name
│       ├── parameters
│       └── overrides_allowed
└── evaluation
    └── metrics[]
        ├── evaluator, name, description, scores (mean, ci_low, ci_high)
        ├── model_id (judge entries only)
        └── optimization_metric: true (exactly one entry — GAM objective)
```

| Field | Description |
|-------|-------------|
| `name`, `iteration`, `max_combinations`, `duration_seconds` | Pattern identity, GAM iteration, search-space size, wall time |
| `settings` | Optimized RAG config: `vector_store_binding`, `chunking`, `embedding`, `retrieval`, `generation` (incl. `language` — benchmark language from search-space preparation, `{code, name}`) |
| `inference.responses_template` | Frozen `POST /v1/responses` body — [Retrieve and generation](#retrieve-and-generation) |
| `indexing.pipeline_spec` | Managed indexing pipeline inputs — [Index building](#index-building) |
| `evaluation` | `metrics[]` — per-metric `evaluator`, `name`, `scores` (`mean`, `ci_low`, `ci_high`); exactly one entry has `optimization_metric: true` (GAM objective). See [RAG pattern evaluation](./rag_pattern_evaluation.md) |

GAM ranks patterns by the pipeline [`optimization_metric`](./experiment_settings.md) parameter (default `overall_score`). The matching `evaluation.metrics[]` entry is marked `optimization_metric: true`; its `scores.mean` is the pattern objective score.

---

## Example pattern.json

```json
{
   "name":"pattern_01",
   "iteration":0,
   "max_combinations":24,
   "duration_seconds":120.5,
   "settings":{
      "vector_store_binding":{
         "provider_id":"milvus-provider",
         "provider_type":"remote::milvus",
         "vector_store_id":"vs_coll_pattern_01"
      },
      "chunking":{
         "method":"recursive",
         "chunk_size":2048,
         "chunk_overlap":256
      },
      "embedding":{
         "model_id":"text-embedding-3-small",
         "embedding_params":{
            "embedding_dimension":768,
            "context_length":2048
         }
      },
      "retrieval":{
         "method":"simple",
         "number_of_chunks":5,
         "search_mode":"hybrid",
         "ranker_strategy":"rrf",
         "ranker_k":60,
         "ranker_alpha":0.5
      },
      "generation":{
         "model_id":"gpt-4.1-mini",
         "context_template_text":"{document}",
         "user_message_text":"…",
         "system_message_text":"…",
         "language":{
            "code":"ja",
            "name":"Japanese"
         }
      }
   },
   "inference":{
      "responses_template":{
         "model":"gpt-4.1-mini",
         "stream":false,
         "store":true,
         "input":[
            {
               "type":"message",
               "role":"user",
               "content":[
                  {
                     "type":"input_text",
                     "text":"<user_query_placeholder>"
                  }
               ]
            }
         ],
         "metadata":{
            "autorag_run_id":"012345678",
            "rag_pattern_name":"pattern_01"
         },
         "instructions":"Please answer the question using only information found in file_search results. …",
         "tools":[
            {
               "type":"file_search",
               "vector_store_ids":[
                  "vs_coll_pattern_01"
               ],
               "max_num_results":5,
               "ranking_options":{
                  "search_mode":"hybrid",
                  "ranker_strategy":"rrf",
                  "ranker_k":60,
                  "ranker_alpha":0.5
               }
            }
         ],
         "tool_choice":{
            "type":"file_search"
         },
         "include":[
            "file_search_call.results"
         ]
      }
   },
   "indexing":{
      "pipeline_spec":{
         "pipeline_name":"documents-indexing-pipeline",
         "parameters":{
            "ogx_secret_name":"ogx-connection",
            "vector_io_provider_id":"milvus-provider-1",
            "vector_store_id":"coll_pattern_01",
            "input_data_secret_name":"s3-input-connection",
            "input_data_bucket_name":"customer-docs",
            "input_data_key":"corpus/v1/",
            "embedding_model_id":"nomic-embed-text-v1.5",
            "embedding_params":{
               "embedding_dimension":768,
               "context_length":2048
            },
            "chunking_method":"recursive",
            "chunk_size":1024,
            "chunk_overlap":128,
            "batch_size":20
         },
         "overrides_allowed":[
            "input_data_secret_name",
            "input_data_bucket_name",
            "input_data_key",
            "vector_store_id",
            "batch_size"
         ]
      }
   },
   "evaluation":{
      "metrics":[
         {
            "evaluator":"unitxt",
            "name":"faithfulness",
            "description":"Whether the generated answer is supported by the retrieved context.",
            "scores":{
               "mean":0.91,
               "ci_low":0.88,
               "ci_high":0.94
            }
         },
         {
            "evaluator":"unitxt",
            "name":"answer_correctness",
            "description":"How well the generated answer matches ground-truth benchmark answers.",
            "scores":{
               "mean":0.82,
               "ci_low":0.78,
               "ci_high":0.86
            }
         },
         {
            "evaluator":"unitxt",
            "name":"context_correctness",
            "description":"Whether retrieval returned sufficient context to answer the question.",
            "scores":{
               "mean":0.80,
               "ci_low":0.70,
               "ci_high":0.90
            }
         },
         {
            "evaluator":"judge",
            "model_id":"gpt-4.1-mini",
            "name":"answer_relevance",
            "description":"Whether the generated answer addresses the benchmark question.",
            "scores":{
               "mean":0.91,
               "ci_low":0.88,
               "ci_high":0.94
            }
         },
         {
            "evaluator":"custom",
            "name":"overall_score",
            "optimization_metric": true,
            "description":"Equal-weight mean of faithfulness, answer_correctness, context_correctness, and answer_relevance. GAM objective for this pattern.",
            "scores":{
               "mean":0.84,
               "ci_low":0.79,
               "ci_high":0.89
            }
         }
      ]
   }
}
```

---

## Retrieve and generation

Optimization and production inference both use **Llama Stack `POST /v1/responses`**. The request body is in **`inference.responses_template`** — production calls the same API surface the benchmark used. See [Llama Stack Responses flow](https://llamastack.github.io/docs/api-openai/responses-flow).

Substitute **`<user_query_placeholder>`** in `input`, then POST. Key fields: `model`, `input`, `tools` (`file_search` + `vector_store_ids`), `instructions`, `metadata`, `tool_choice`, `include`. Parse **`output`** (or SDK **`output_text`**) in consumers.

**Python:**

```python
import copy, json, os
from pathlib import Path
import requests

def query_pattern(pattern_path: Path, user_query: str) -> dict:
    pattern = json.loads(pattern_path.read_text())
    body = copy.deepcopy(pattern["inference"]["responses_template"])
    for block in body.get("input", []):
        for part in block.get("content", []):
            if part.get("text") == "<user_query_placeholder>":
                part["text"] = user_query
    r = requests.post(
        os.environ["LLAMA_STACK_RESPONSES_URL"],
        json=body,
        headers={"Authorization": f"Bearer {os.environ['LLAMA_STACK_API_KEY']}"},
        timeout=120,
    )
    r.raise_for_status()
    return r.json()
```

---

## Index building

Index building populates the production vector store via the managed **`documents-indexing-pipeline`** ([`documents_indexing_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline)), registered in the AI Pipelines catalog. One pipeline definition serves all patterns; per-pattern values come from **`indexing.pipeline_spec`**.

| `pipeline_spec` field | Role |
|-----------------------|------|
| `pipeline_name` | Managed catalog name (e.g. `documents-indexing-pipeline`) |
| `parameters` | Pre-filled from optimization run + pattern `settings` |
| `overrides_allowed` | Keys the UI may expose for user override at submit time |

**Parameter sources:** optimization run → `ogx_secret_name`, `input_data_*`, `vector_io_provider_id`; pattern `settings` → embedding (`embedding_model_id`, `embedding_params`), chunking, `vector_store_id`. Secret fields are **names only** (Kubernetes Secret references).

**Workflow:** optimization completes → user selects pattern → read `pipeline_spec` → resolve managed pipeline → pre-fill run form → user confirms/overrides → submit → full corpus indexed → [retrieve and generation](#retrieve-and-generation) ready.

**Pipeline steps:** load inputs → document discovery/extraction → chunking → embedding → vector store write → validation/logging. Observable via KFP; re-runnable when documents or overrides change.

---

## Related

- [RAG pattern evaluation](./rag_pattern_evaluation.md)
- [AutoRAG optimization settings](./experiment_settings.md) — pipeline parameters, presets, chunking, retrieval
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
