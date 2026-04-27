# RAG Pattern Inference

This page documents **AutoRAG pattern deployment** from optimization completion through production inference.

**RHOAI 3.4 (current)**

- [Pattern artifacts](#pattern-artifacts)
  - [`rag_patterns` artifact](#1-rag_patterns-artifact-rag_templates_optimization)
  - [`responses_api_artifacts` artifact](#2-responses_api_artifacts-artifact-prepare_responses_api_requests)
- [Pattern.json](#patternjson)
- [GenAI Studio integration](#genai-studio-integration)
- [Deployment today](#deployment-today)

**RHOAI 3.5 and later (planned)**

- [AutoRAG Backend (Optimization Pipeline)](#autorag-backend-optimization-pipeline)
- [Generated artifacts per pattern](#generated-artifacts-per-pattern)
- [Target payloads (examples)](#target-payloads-examples)
  - [Example pattern.json](#example-patternjson)
- [LLama-Stack /v1/responses API](#llama-stack-v1responses-api)
  - [Request payload](#request-payload)
  - [Response format](#response-format)
- [AutoRAG Dashboard (UI)](#autorag-dashboard-ui)
  - [Dashboard capabilities](#dashboard-capabilities)
  - [Code snippet generation](#code-snippet-generation)
- [Configuration Persistence for Gen AI Studio](#configuration-persistence-for-gen-ai-studio)
  - [End-to-end integration workflow](#end-to-end-integration-workflow)

---

## Current state (RHOAI 3.4)

### Pattern artifacts

The **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** writes **three** KFP-relevant outputs for “what to ship per pattern,” plus a **fourth** pipeline-wide HTML file. Paths below are **logical** names under each **per-pattern subdirectory** (one folder per optimized pattern); the exact artifact store URI layout follows Data Science Pipelines backend.

#### 1. `rag_patterns` artifact ([`rag_templates_optimization`](https://github.com/red-hat-data-services/pipelines-components/blob/main/components/training/autorag/rag_templates_optimization/component.py))

Each **`<pattern_subdir>/`** contains:

| File | Type | Purpose |
|------|------|---------|
| `pattern.json` | JSON | **Authoritative pattern record** from ai4rag: name, iteration, **`settings`** (chunking, embedding, retrieval, generation, vector store / collection, …), aggregate **`scores`**, **`final_score`**, timing. Additional **Responses / binding** fields are [planned](#target-payloads-examples); validate against the schema your build emits. |
| `indexing_notebook.ipynb` | Jupyter Notebook | **Indexing** notebook instantiated from templates (for example **`ls_indexing_template.ipynb`**), parameterized for this pattern. |
| `inference_notebook.ipynb` | Jupyter Notebook | **Inference** notebook instantiated from templates (for example **`ls_inference_template.ipynb`**), parameterized for this pattern. |
| `evaluation_results.json` | JSON | **Per-pattern evaluation payload** (per-question scores, traces, metadata) written when the pattern is created—**this is the dedicated evaluation file**; aggregate numbers also appear inside **`pattern.json`** for leaderboard use. |

#### 2. `responses_api_artifacts` artifact ([`prepare_responses_api_requests`](https://github.com/red-hat-data-services/pipelines-components/blob/main/components/deployment/autorag/build_responses_request_bodies/component.py))

A **separate** output artifact **mirrors** the same `<pattern_subdir>/` names. Under each folder:

| File | Type | Purpose |
|------|------|---------|
| `v1_responses_body.json` | JSON | Frozen **`POST /v1/responses`** JSON body derived from that folder’s **`pattern.json`** (today: **`model`**, **`input`**, **`tools`** / **`file_search`**, **`instructions`**, **`metadata`**, … per the component). Used by **AutoRAG Dashboard**, **Gen AI Studio**, and copy/paste snippets—not chat-completions **`messages`**. |
| `create_model_response.py` | Python | Small **interactive** client script (embeds Llama Stack base URL from env, prompts for API key, loops on questions). |
| `README.md` | Markdown | How to run the script, TLS / CA bundle notes, and URL overrides. |


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

This is **not** the long-term integrated flow; it proves connectivity while [Planned work](#planned-work-rhoai-35-and-later) lands Dashboard → Studio handoff.

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

| Artifact | Type | Purpose |
|----------|------|---------|
| **`pattern.json`** | JSON | **Consolidated pattern record** containing: chunking, embedding, retrieval, generation settings; **Responses API request template** (`input`, `tools`, `instructions`, `metadata`); vector store binding; aggregate scores and timing. This single file provides everything needed for registration, deployment, and code generation. |
| **`indexing_notebook.ipynb`** | Jupyter Notebook | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| **`inference_notebook.ipynb`** | Jupyter Notebook | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| **`evaluation_results.json`** | JSON | Per-question evaluation metrics (context precision, answer relevance, faithfulness), traces, metadata—the dedicated evaluation file |
| **`create_model_response.py`** | Python | Interactive client script for testing patterns (embeds Llama Stack base URL from env, reads Responses config from `pattern.json`) |
| **`README.md`** | Markdown | How to run the test script, TLS / CA bundle notes, URL overrides |

**Key change from 3.4:** 
- The separate `v1_responses_body.json` artifact is eliminated
- The Responses API request template is embedded in `pattern.json` under the `settings.responses_template` field, providing a single source of truth
- `settings.vector_store_binding` introduced in place of out-dated `settings.vector_store` (properties aligned with LLS config)

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

**Key fields (3.5 additions to 3.4 schema):**
- **`settings.responses_template`:** Contains the complete **`POST /v1/responses`** request template that matches what the optimization engine used (ai4rag + Llama Stack). The `<user_query_placeholder>` in the `input` array is replaced with actual questions during inference. Dashboard and GenAI Studio extract this field to generate code snippets and enable "try me out."
  - **`model`**: generation model identifier (e.g., "gpt-4.1-mini")
  - **`stream`**: whether to stream responses (boolean)
  - **`store`**: whether to persist the interaction (boolean)
  - **`input`**: structured message array with placeholder for user query
  - **`metadata`**: run tracking metadata (`autorag_run_id`, `rag_pattern_name`)
  - **`instructions`**: system-level instructions for the RAG task
  - **`tools`**: file_search configuration with vector_store_ids and optional ranking_options for hybrid search (search_mode, ranker_strategy, ranker_k, ranker_alpha)
  - **`tool_choice`**: specifies which tool to invoke
  - **`include`**: controls what to include in the response (e.g., "file_search_call.results")

- **`settings.vector_store_binding`:** Llama Stack vector store registration metadata that aligns with the ConfigMap and REST API schema (see [Vector store ConfigMaps](#vector-store-configmaps)). This field is nested within the `settings` block:
  - **`provider_id`**: Llama Stack provider identifier (e.g., "milvus-provider") registered in ConfigMap `providers.vector_io[].provider_id`
  - **`provider_type`**: Provider implementation type matching ConfigMap `provider_type` (e.g., "inline::milvus", "remote::milvus")
  - **`vector_store_id`**: Unique Llama Stack vector store identifier (e.g., "vs_coll_pattern_01") that matches `registered_resources.vector_stores[].vector_store_id` in ConfigMap and is referenced in `responses_template.tools[].vector_store_ids`
  - **`vector_store_name`**: Human-readable name matching ConfigMap `vector_store_name`

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

---

## AutoRAG Dashboard (UI)

The **OpenShift AI AutoRAG Dashboard** provides data scientists with visibility into optimization runs and enables pattern export.

### Dashboard capabilities

The Dashboard aggregates KFP pipeline results and pattern artifacts for RHOAI users:

| Capability | Description                                                                                                                                                                           |
|------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Responses preview** | Renders **`pattern.json`** `settings.responses_template` field (read-only)—including structured `input` and `file_search` when present—so users see the exact Responses API template. |
| **Vector binding** | Shows **`settings.vector_store_binding`** in place of `vector_store`.                                                                                                                 |
| **Hand-off to GenAI Studio** | Deep link or button to GenAI Studio “try me out” (requires configuration persistence described in https://redhat.atlassian.net/browse/RHAIRFE-912).                                   |

### Code snippet generation

The Dashboard provides one-click export of Python/curl examples that call **`POST /v1/responses`**, ready for use in production applications. Users can either:

- **Download the ready-to-use script:** Export the **`create_model_response.py`** script directly from the pattern artifacts (see [Generated artifacts per pattern](#generated-artifacts-per-pattern)). This interactive client script embeds the Llama Stack base URL from environment variables, reads the Responses configuration from **`pattern.json`**, and provides a command-line interface for testing patterns.
  
- **Copy inline code snippets:** Generate Python or curl examples on-demand for integration into existing applications.

- **Run the code snippet:** Execute REST API example on-demand and render the results.

**Python example:**

```python
import requests

# LLama-Stack inference endpoint (RHOAI 3.5)
LLAMA_STACK_URL = “https://llama-stack.apps.cluster.example.com/v1/responses”

# Pattern ID from AutoRAG optimization
PATTERN_ID = “autorag-pattern-12345-v1”

def query_rag_pattern(user_query: str) -> dict:
    “””
    Query a deployed RAG pattern via Llama Stack Responses API.
    “””
    payload = {
        “model”: “granite-3.1-8b-instruct”,
        “input”: user_query,
        “tools”: [
            {
                “type”: “retrieval”,
                “retrieval”: {
                    “pattern_id”: PATTERN_ID,
                    “top_k”: 5,
                    “rerank”: {“enabled”: True, “top_n”: 3},
                },
            }
        ],
        “temperature”: 0.7,
        “max_output_tokens”: 512,
    }

    response = requests.post(LLAMA_STACK_URL, json=payload)
    response.raise_for_status()
    result = response.json()

    # Parsing is build-specific: many stacks expose `output_text` via SDK;
    # raw JSON often includes an `output` list (assistant message + tool payloads).
    answer = result.get(“output_text”) or “”
    chunks: list = []
    for item in result.get(“output”, []):
        if item.get(“type”) == “retrieval”:
            chunks.extend(item.get(“chunks”, []))

    return {“answer”: answer, “chunks”: chunks, “raw”: result}

# Example usage
if __name__ == “__main__”:
    result = query_rag_pattern(“What is the refund policy?”)
    print(f”Answer: {result['answer']}”)
    print(f”Retrieved {len(result['chunks'])} chunks”)
```

**curl equivalent:**

```bash
curl -X POST https://llama-stack.apps.cluster.example.com/v1/responses \
  -H “Content-Type: application/json” \
  -d '{
    “model”: “granite-3.1-8b-instruct”,
    “input”: “What is the refund policy?”,
    “tools”: [{
      “type”: “retrieval”,
      “retrieval”: {
        “pattern_id”: “autorag-pattern-12345-v1”,
        “top_k”: 5,
        “rerank”: {“enabled”: true, “top_n”: 3}
      }
    }],
    “temperature”: 0.7,
    “max_output_tokens”: 512
  }'
```

**Portability:** Examples use plain **`requests`** / **`curl`** for portability. 

---

---

## Configuration Persistence for Gen AI Studio

Configuration persistence for Gen AI Studio, including pattern registration via Config, vector store configuration, and “Try me out” workflow integration, to be discussed in [RHAIRFE-912](https://issues.redhat.com/browse/RHAIRFE-912).

**Key topics to be covered:**
- Pattern config registration for discovery
- GenAI Studio “Try me out” for AutoRAG Patterns

---

### End-to-end integration workflow

The following diagram shows how all components could integrate together:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. AutoRAG Backend (Optimization Pipeline)                      │
│    └─> Evaluates candidates                                     │
│    └─> Generates pattern.json (with Responses API template)     │
│    └─> Artifacts written to object storage                      │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. AutoRAG Dashboard                                            │
│    └─> Lists optimization runs and patterns                     │
│    └─> Displays pattern.json (including Responses API template) │
│    └─> Provides code snippets (Python/curl)                     │
│    └─> Links to GenAI Studio for try-out (saves configs)        │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. GenAI Studio / Playground (Optional)                         │                    │
│    └─> Loads persisted AutoRAG config                           │
│    └─> Injects test query into template                         │
│    └─> Sends POST /v1/responses to Llama Stack                  │
│    └─> Displays results and retrieved chunks                    │
└─────────────────────────────────────────────────────────────────┘
```

---
