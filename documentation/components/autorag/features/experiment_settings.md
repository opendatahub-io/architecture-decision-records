# Experiment settings (pipeline parameters)

This page documents the **top-level input parameters** for the **Documents RAG optimization** pipeline shipped in [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) under `pipelines/training/autorag/`.

Higher-level architecture and ADR-level parameter groups are in [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md). Chunking, retrieval, and Docling behavior referenced by presets are in [Chunking and retrieval methods](./chunking_and_retrievals_methods.md).

## Table of contents

- [Documents RAG optimization pipeline](#documents-rag-optimization-pipeline)
  - [Current shipping parameters](#current-shipping-parameters)
  - [OpenShift AI 3.5 and later](#openshift-ai-35-and-later)
- [Preset support](#preset-support)

---

## Documents RAG optimization pipeline

These parameters are the public surface of `documents_rag_optimization_pipeline` in [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py).

### Current shipping parameters

Arguments exposed today on the Documents RAG optimization pipeline (verify on your **pipelines-components** tag or branch):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `test_data_secret_name` | `str` | (required) | Kubernetes **Secret** for S3-compatible test-data access. Expected keys: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`; `AWS_DEFAULT_REGION` optional. |
| `test_data_bucket_name` | `str` | (required) | Bucket containing the evaluation benchmark JSON. |
| `test_data_key` | `str` | (required) | Object key of the test data file. |
| `input_data_secret_name` | `str` | (required) | Kubernetes **Secret** for document corpus access (same key convention as test data). |
| `input_data_bucket_name` | `str` | (required) | Bucket containing source documents. |
| `input_data_key` | `str` | `""` | Object key or prefix for input documents. |
| `ogx_secret_name` | `str` | (required) | Secret for OGX / Llama Stack API: `OGX_CLIENT_API_KEY`, `OGX_CLIENT_BASE_URL`. |
| `vector_io_provider_id` | `str` | (required) | Registered vector I/O provider id (for example Milvus). |
| `embedding_models` | `Optional[List[str]]` | `None` | Optional allow-list of embedding model ids for the search space. When omitted, ai4rag uses platform defaults. |
| `generation_models` | `Optional[List[str]]` | `None` | Optional allow-list of generation model ids for the search space. |
| `optimization_metric` | `str` | `faithfulness` | GAM objective. One of `faithfulness`, `answer_correctness`, `context_correctness`. |
| `optimization_max_rag_patterns` | `int` | `8` | Maximum number of RAG patterns to evaluate and retain (`max_number_of_rag_patterns` in ai4rag). |

### OpenShift AI 3.5 and later

Additional KFP pipeline parameter planned for the optimization graph. Confirm name and optionality in **`pipeline.py`** for your build.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preset` | `str` | `speed` | Pipeline-level quality tier. Maps to Docling extraction options, chunking search space, and indexing features before `rag_templates_optimization`. Use only values listed under [Preset support](#preset-support). Recorded in MLflow as `preset` on the parent run. |

---

## Preset support

**OpenShift AI AutoRAG** exposes two pipeline presets. Each preset fixes **ingestion and search-space defaults**; ai4rag still optimizes embedding, retrieval, and generation settings within that envelope. **Both presets use the same pipeline resource tier** (8 vCPU / 32 GiB RAM per workload step).

| Preset | Min resources (workload steps) | Role (summary) |
|--------|-------------------------------|----------------|
| `speed` | 4 vCPU / 16 GiB RAM | Fastest path: recursive chunking on exported text, no table-structure parsing, no LLM contextual enrichment. |
| `balanced` | 8 vCPU / 32 GiB RAM | Higher quality for structured PDFs/DOCX: Docling table layout parser, hybrid chunker, and [contextual retrieval](./chunking_and_retrievals_methods.md#llm-contextual-enrichment-index-time-rhoai-35) at index time. |

### `speed` (default)

| Layer | Setting |
|-------|---------|
| **Docling (`text_extraction`)** | PDF pipeline: `do_table_structure: false` — layout detection only; tables are not reconstructed with TableFormer. |
| **Chunking search space** | `chunking.method: recursive` — Markdown (or flat text) export, then size/overlap splitting. |
| **Contextual enrichment** | `chunking.contextual_enrichment.enabled: false` — not explored in the search space. |
| **Benchmark query concurrency** | ai4rag `query_rag` **`max_threads`**: **10** (default). |

### `balanced`

Enables the three features below relative to `speed`:

| Feature | Where it applies | Setting |
|---------|------------------|---------|
| **Docling table layout parser** | `text_extraction` (PDF) | `do_table_structure: true` — [TableFormer](https://docling-project.github.io/docling/guides/pdf-processing/) reconstructs table rows and columns from detected layout. |
| **Docling hybrid chunker** | Optimization / indexing search space | `chunking.method: hybrid` — structure-aware chunks from persisted `DoclingDocument`, with tokenizer-aware split/merge (see [Hierarchical / hybrid chunking](./chunking_and_retrievals_methods.md#hierarchical-chunking-rhoai-35)). |
| **Contextual retrieval** | Indexing (`chunking`) | `chunking.contextual_enrichment.enabled: true` — LLM-generated situating text prepended before embedding and sparse indexing ([Anthropic contextual retrieval](https://www.anthropic.com/news/contextual-retrieval)). |
| **Benchmark query concurrency** | ai4rag `query_rag` **`max_threads`**: **4** (lower than `speed` because each concurrent request carries more retrieved context). |

**Example `pattern.json` fragment** produced under `balanced` (other fields still optimized by ai4rag):

```json
{
  "settings": {
    "chunking": {
      "method": "hybrid",
      "chunk_size": 1024,
      "chunk_overlap": 128,
      "merge_lists": true,
      "include_context": true,
      "contextual_enrichment": {
        "enabled": true,
        "context_generation_model": "<from search space or platform default>"
      }
    }
  }
}
```

Docling extraction settings are applied in **`text_extraction`** and are not repeated per pattern in `pattern.json`; they are fixed for the pipeline run by the chosen preset.

**Benchmark concurrency:** Each pattern evaluation runs benchmark questions through ai4rag [`query_rag`](https://github.com/IBM/ai4rag/blob/main/ai4rag/core/experiment/utils.py) (`ThreadPoolExecutor`, configurable **`max_threads`**). Each worker performs retrieval plus one generation LLM call with up to `number_of_chunks` strings in the prompt. The pipeline preset maps **`max_threads`** into ai4rag (implementation tracked separately in ai4rag). Indexing embeddings use batched API calls; Unitxt evaluation uses batch `evaluate()`, not this thread pool.

> **Cost note:** `balanced` increases ingestion time and LLM usage (context generation per chunk) compared to `speed`, but both presets target the same **8 vCPU / 32 GiB** step sizing. Prefer `speed` for plain-text corpora or exploratory runs; use `balanced` when documents contain tables, headings, and multi-section structure.

---

When in doubt, prefer the **README and `pipeline.py` on the exact branch or tag** of pipelines-components that ships with your OpenShift AI version.
