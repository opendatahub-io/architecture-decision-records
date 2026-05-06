# RAG Pattern Inference

This page documents **AutoRAG pattern inference** from optimization completion through production inference.

**RHOAI 3.4**

- [Pattern artifacts](#pattern-artifacts)
  - [`rag_patterns` artifact](#1-rag_patterns-artifact-rag_templates_optimization)
  - [`responses_api_artifacts` artifact](#2-responses_api_artifacts-artifact-prepare_responses_api_requests)
- [Pattern.json](#patternjson)
- [GenAI Studio integration](#genai-studio-integration)
- [Deployment today](#deployment-today)

**RHOAI 3.5 and later**

- [AutoRAG Backend (Optimization Pipeline)](#autorag-backend-optimization-pipeline)
- [Generated artifacts per pattern](#generated-artifacts-per-pattern)
- [Target payloads (examples)](#target-payloads-examples)
  - [Example pattern.json](#example-patternjson)
- [LLama-Stack /v1/responses API](#llama-stack-v1responses-api)
  - [Request payload](#request-payload)
  - [Response format](#response-format)
- [Code snippet generation](#code-snippet-generation)

---

## RHOAI 3.4

### Pattern artifacts

The **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** produces pattern artifacts organized into **two main KFP output artifacts** (`rag_patterns` and `responses_api_artifacts`), plus a **pipeline-wide HTML leaderboard**. Paths below are **logical** names under each **per-pattern subdirectory** (one folder per optimized pattern); the exact artifact store URI layout follows Data Science Pipelines backend.

#### 1. `rag_patterns` artifact ([`rag_templates_optimization`](https://github.com/red-hat-data-services/pipelines-components/blob/main/components/training/autorag/rag_templates_optimization/component.py))

Each **`<pattern_subdir>/`** contains:

| File | Purpose |
|------|---------|
| `pattern.json` | Authoritative pattern record from ai4rag: name, iteration, `settings` (chunking, embedding, retrieval, generation, vector store), aggregate `scores`, `final_score`, timing |
| `indexing_notebook.ipynb` | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| `inference_notebook.ipynb` | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| `evaluation_results.json` | Per-pattern evaluation payload (per-question scores, traces, metadata)—the dedicated evaluation file; aggregate numbers also appear in `pattern.json` for leaderboard use |

#### 2. `responses_api_artifacts` artifact ([`prepare_responses_api_requests`](https://github.com/red-hat-data-services/pipelines-components/blob/main/components/deployment/autorag/build_responses_request_bodies/component.py))

A **separate** output artifact **mirrors** the same `<pattern_subdir>/` names. Under each folder:

| File | Purpose |
|------|---------|
| `v1_responses_body.json` | Frozen `POST /v1/responses` JSON payload derived from `pattern.json` (`model`, `input`, `tools`/`file_search`, `instructions`, `metadata`). Used by AutoRAG Dashboard, Gen AI Studio, and copy/paste snippets |
| `create_model_response.py` | Interactive client script for testing patterns (embeds Llama Stack base URL from env, prompts for API key, loops on questions) |
| `README.md` | Instructions for running the client script, TLS/CA bundle notes, and URL overrides |


### Pattern.json

```json
{
  "name": "pattern_01",
  "iteration": 0,
  "max_combinations": 24,
  "duration_seconds": 120.5,
  "settings": {
    "vector_store": {
      "datasource_type": "milvus",
      "collection_name": "coll_pattern_01"
    },
    "chunking": {
      "method": "recursive",
      "chunk_size": 2048,
      "chunk_overlap": 256
    },
    "embedding": {
      "model_id": "text-embedding-3-small",
      "distance_metric": "cosine",
      "embedding_params": {
        "embedding_dimension": 768
      }
    },
    "retrieval": {
      "method": "simple",
      "number_of_chunks": 5,
      "search_mode": "hybrid",
      "ranker_strategy": "rrf",
      "ranker_k": 60,
      "ranker_alpha": 0.5
    },
    "generation": {
      "model_id": "gpt-4.1-mini",
      "context_template_text": "{document}",
      "user_message_text": "\n\nContext:\n{reference_documents}:\n\nQuestion: {question}. ...",
      "system_message_text": "Please answer the question ..."
    }
  },
  "scores": {
    "faithfulness": {
      "mean": 0.91,
      "ci_low": 0.88,
      "ci_high": 0.94
    },
    "answer_correctness": {
      "mean": 0.82,
      "ci_low": 0.78,
      "ci_high": 0.86
    }
  },
  "final_score": 0.87
}
```

### GenAI Studio integration

For **short demos** before automated registration exists, teams typically:

- Register the **vector store** AutoRAG created (for example on a remote **Milvus** instance) as a **Gen AI Studio AI Asset**, often backed by a **ConfigMap** that matches how Llama Stack / Studio expect provider config.
- Add the **embedding** model as an AI Asset via **custom endpoints** (and similarly the **chat / completion** model).
- Combine **vector store + embedding + chat** assets in the **Gen AI Studio playground** to approximate “try me out.”

This is **not** the long-term integrated flow; it proves connectivity while the RHOAI 3.5 improvements land Dashboard → Studio handoff.

### Deployment today

Operators and data scientists follow a **manual** path:

1. **Pipeline completes** → artifacts in the run’s store (**`rag_patterns`**, **`responses_api_artifacts`**, leaderboard HTML), as above.
2. **Download** from the pipeline / object-store UI.
3. **Wire Kubernetes by hand:** build ConfigMaps from **`pattern.json`**, apply with **`kubectl`**, point **Llama Stack** deployments at them, restart services.
4. **Smoke-test** by copying **`responses_api_artifacts/<pattern>/v1_responses_body.json`** and calling **`POST /v1/responses`** (for example **`curl`** or Postman).


---

## RHOAI 3.5 and later

### AutoRAG Backend (Optimization Pipeline)

The 3.5 target architecture achieves **end-to-end Responses API consistency**: AutoRAG optimization evaluates candidates using **Llama Stack `POST /v1/responses`** (not legacy chat completions), ensuring optimization and production use identical API surfaces to prevent drift. The Responses API request body is **embedded directly in `pattern.json`**, eliminating the need for separate `v1_responses_body.json` files.

See [Llama Stack Responses flow](https://llamastack.github.io/docs/api-openai/responses-flow) for upstream API behavior.

### Generated artifacts per pattern

The pipeline produces artifacts under each **`<pattern_subdir>/`** (one directory per optimized pattern):

| Artifact | Purpose |
|----------|---------|
| `pattern.json` | Consolidated pattern record containing: chunking, embedding, retrieval, generation settings; Responses API request template (`input`, `tools`, `instructions`, `metadata`); vector store binding; aggregate scores and timing. Single source of truth for registration, deployment, and code generation |
| `indexing_notebook.ipynb` | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| `inference_notebook.ipynb` | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| `evaluation_results.json` | Per-pattern evaluation payload (per-question scores, traces, metadata)—the dedicated evaluation file; aggregate numbers also appear in `pattern.json` for leaderboard use |
| `create_model_response.py` | Interactive client script for testing patterns (embeds Llama Stack base URL from env, reads Responses config from `pattern.json`) |


**Key changes from 3.4:**

The separate `v1_responses_body.json` artifact is **eliminated**. The Responses API request template is now **embedded in `pattern.json`** under `settings.responses_template`, providing a single source of truth.

**New fields added to `pattern.json`:**

- **`settings.responses_template`** (object): The `POST /v1/responses` request payload template that matches what the optimization engine used (ai4rag + Llama Stack). The `<user_query_placeholder>` in the `input` array is replaced with actual questions during inference. Dashboard and GenAI Studio extract this field to generate code snippets and enable "try me out."
  - **`model`** (string): Generation model identifier (e.g., "gpt-4.1-mini")
  - **`stream`** (boolean): Whether to stream responses
  - **`store`** (boolean): Whether to persist the interaction
  - **`input`** (array): Structured message array with placeholder for user query
  - **`metadata`** (object): Run tracking metadata (`autorag_run_id`, `rag_pattern_name`)
  - **`instructions`** (string): System prompt that directs the model's behavior and response style (similar to [LangChain system messages](https://docs.langchain.com/oss/python/langchain/messages#system-message))
  - **`tools`** (array): Tool configurations; for RAG patterns typically contains a file_search tool with vector store references and retrieval settings
  - **`tool_choice`** (object): Specifies which tool to invoke
  - **`include`** (array): Controls what to include in the response (e.g., "file_search_call.results")

- **`settings.vector_store_binding`** (object): Llama Stack vector store registration metadata that aligns with the ConfigMap and REST API schema. Replaces the 3.4 `settings.vector_store` field:
  - **3.4 schema:** Used `settings.vector_store.collection_name` to reference the Milvus collection
  - **3.5 schema:** Uses `settings.vector_store_binding.vector_store_id` aligned with Llama Stack provider configuration
  - **`provider_id`** (string): Llama Stack provider identifier (e.g., "milvus-provider") registered in ConfigMap `providers.vector_io[].provider_id`
  - **`provider_type`** (string): Provider implementation type matching ConfigMap `provider_type` (e.g., "inline::milvus", "remote::milvus")
  - **`vector_store_id`** (string): Unique Llama Stack vector store identifier (e.g., "vs_coll_pattern_01") that matches `registered_resources.vector_stores[].vector_store_id` in ConfigMap and is referenced in `responses_template.tools[].vector_store_ids`
  - **`vector_store_name`** (string): Human-readable name matching ConfigMap `vector_store_name`

### Target payloads (examples)

Illustrative shape for **`pattern.json`** with **Responses API template** and **`vector_store_binding`** embedded in the schema.

#### Example pattern.json

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
         "vector_store_id":"vs_coll_pattern_01",
         "vector_store_name":"AutoRAG pattern_01 vector store"
      },
      "chunking":{
         "method":"recursive",
         "chunk_size":2048,
         "chunk_overlap":256
      },
      "embedding":{
         "model_id":"text-embedding-3-small",
         "distance_metric":"cosine",
         "embedding_params":{
            "embedding_dimension":768
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
         "user_message_text":"\n\nContext:\n{reference_documents}:\n\nQuestion: {question}. ...",
         "system_message_text":"Please answer the question ..."
      },
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
         "instructions":"Please answer the question using only information found in file_search results. If the question is unanswerable, say you cannot answer. Respond in the same language as the user question.",
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
   "scores":{
      "faithfulness":{
         "mean":0.91,
         "ci_low":0.88,
         "ci_high":0.94
      },
      "answer_correctness":{
         "mean":0.82,
         "ci_low":0.78,
         "ci_high":0.86
      }
   },
   "final_score":0.87
}
```

### LLama-Stack /v1/responses API

The backend uses **`POST /v1/responses`** for both optimization and inference, following the Responses contract (see [Llama Stack Responses flow](https://llamastack.github.io/docs/api-openai/responses-flow)).

#### Request payload

**`POST /v1/responses`** uses **`input`** as a plain string or structured blocks (e.g., **`input_text`** inside a **`message`**) and **`tools`** such as **`file_search`** with **`vector_store_ids`**—not a legacy chat-completions **`messages`** top-level array. The concrete request body is embedded in the `settings.responses_template` field of [Example pattern.json](#example-patternjson).

**Key fields:**
- **`model`**: generation model id (aligns with `pattern.json` `generation.model_id`)
- **`input`**: user question as a string or structured message blocks
- **`tools`**: often **`file_search`** with **`vector_store_ids`** for AutoRAG + Milvus
- **`instructions`**, **`metadata`**, **`tool_choice`**, **`include`**: common in frozen recipes; optional depending on template
- **`max_output_tokens` / `temperature` / `stream` / `store`**: generation and tracing controls

#### Response format

**Illustrative non-streaming Responses object** (field names vary by Llama Stack release—validate against your server):

```json
{
  "id": "resp-abc123",
  "object": "response",
  "model": "granite-3.1-8b-instruct",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "Refunds are processed within 14 days of request. Contact support@example.com with your order number."
        }
      ]
    },
    {
      "type": "retrieval",
      "pattern_id": "autorag-pattern-12345-v1",
      "chunks": [
        {
          "text": "Our refund policy allows customers to request refunds within 30 days...",
          "score": 0.92,
          "metadata": {"source": "policy.pdf", "page": 12}
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 245,
    "output_tokens": 89,
    "total_tokens": 334
  }
}
```

**Consumers (Dashboard, GenAI Studio, tests):** parse **`output`** (and any convenience fields such as **`output_text`** when using the official SDK) instead of **`choices[].message`** from chat completions.

### Code snippet generation

After optimization, you can call **`POST /v1/responses`** in two straightforward ways:

1. **Download the Python helper** — Take **`create_model_response.py`** from the pattern output (see [Generated artifacts per pattern](#generated-artifacts-per-pattern)) or export it from the AutoRAG Dashboard. The script reads **`pattern.json`**, applies your Llama Stack base URL and credentials from the environment, and drives **`/v1/responses`** interactively so you do not have to assemble the JSON by hand.

2. **Build the request from `settings.responses_template`** — In RHOAI 3.5+, **`pattern.json`** already contains the same **`POST /v1/responses`** body the optimizer used, under **`settings.responses_template`** (see [Example pattern.json](#example-patternjson)). Treat that object as your request body: point your HTTP client at **`/v1/responses`**, send it as JSON, and only substitute the user question—typically by replacing the **`<user_query_placeholder>`** token in the structured **`input`** (or whatever placeholder your build emits). No need to re-specify **`tools`**, **`instructions`**, or **`metadata`** unless you intentionally change the recipe.

The Dashboard can still offer copy/export actions; whether you paste from the UI or read **`pattern.json`** from disk, the source of truth for the request shape is **`settings.responses_template`**.

**Python (load template, swap placeholder, POST):**

```python
import copy
import json
import os
from pathlib import Path

import requests

LLAMA_STACK_RESPONSES_URL = "https://llama-stack.apps.cluster.example.com/v1/responses"
PATTERN_JSON = Path("pattern.json")


def build_responses_body(pattern: dict, user_query: str) -> dict:
    """Return POST /v1/responses JSON from pattern.settings.responses_template."""
    body = copy.deepcopy(pattern["settings"]["responses_template"])
    for block in body.get("input", []):
        if block.get("type") != "message":
            continue
        for part in block.get("content", []):
            if part.get("type") == "input_text" and part.get("text") == "<user_query_placeholder>":
                part["text"] = user_query
    return body


def query_pattern(user_query: str) -> dict:
    pattern = json.loads(PATTERN_JSON.read_text())
    payload = build_responses_body(pattern, user_query)
    r = requests.post(
        LLAMA_STACK_RESPONSES_URL,
        json=payload,
        headers={"Authorization": f"Bearer {os.environ['LLAMA_STACK_API_KEY']}"},
        timeout=120,
    )
    r.raise_for_status()
    return r.json()


if __name__ == "__main__":
    out = query_pattern("What is the refund policy?")
    # Parsing is server-specific; often inspect `output` or SDK helpers like `output_text`.
    print(out)
```

**curl:** Same payload shape as **`settings.responses_template`** from **`pattern.json`**, with the user question inlined in **`input`** instead of **`<user_query_placeholder>`**. Adjust **`model`**, **`vector_store_ids`**, and **`metadata`** to match your pattern and deployment.

```bash
curl -X POST "https://llama-stack.apps.cluster.example.com/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LLAMA_STACK_API_KEY}" \
  -d '{
  "model": "gpt-4.1-mini",
  "stream": false,
  "store": true,
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "What is the refund policy?"
        }
      ]
    }
  ],
  "metadata": {
    "autorag_run_id": "012345678",
    "rag_pattern_name": "pattern_01"
  },
  "instructions": "Please answer the question using only information found in file_search results. If the question is unanswerable, say you cannot answer. Respond in the same language as the user question.",
  "tools": [
    {
      "type": "file_search",
      "vector_store_ids": ["vs_coll_pattern_01"],
      "max_num_results": 5,
      "ranking_options": {
        "search_mode": "hybrid",
        "ranker_strategy": "rrf",
        "ranker_k": 60,
        "ranker_alpha": 0.5
      }
    }
  ],
  "tool_choice": {
    "type": "file_search"
  },
  "include": ["file_search_call.results"]
}'
```

**Portability:** The examples use plain **`requests`** and **`curl`**; add TLS options or a service mesh sidecar as required by your cluster.


---
