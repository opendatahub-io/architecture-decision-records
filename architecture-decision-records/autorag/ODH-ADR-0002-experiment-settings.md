# Open Data Hub - AutoRAG Optimization Settings

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1673](https://redhat.atlassian.net/browse/RHAISTRAT-1673) · [RHAISTRAT-2040](https://redhat.atlassian.net/browse/RHAISTRAT-2040) · [RHAISTRAT-2440](https://redhat.atlassian.net/browse/RHAISTRAT-2440) · [RHAISTRAT-2623](https://redhat.atlassian.net/browse/RHAISTRAT-2623) · [RHOAIENG-79225](https://redhat.atlassian.net/browse/RHOAIENG-79225) · [RHOAIENG-88692](https://redhat.atlassian.net/browse/RHOAIENG-88692) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR documents the AutoRAG documents RAG optimization pipeline public parameters, HPO-time MaaS and vector-store wiring, quality presets, and the chunking / retrieval search-space dimensions explored by ai4rag during GAM optimization.

## Why

Pipeline operators and Dashboard integrations need a stable contract for how optimization runs are configured (data sources, MaaS, vector-DB, and graph-DB Connections, required model lists, presets) and which chunking and retrieval knobs form the searchable configuration space. Capturing this as an ADR keeps the contract versioned alongside the AutoRAG architecture.

## Goals

* Define the public parameter surface of `documents_rag_optimization_pipeline`
* Document multi-location corpus ingest (`input_data_keys` as a list of up to 10 keys/prefixes) and benchmark document identity by object key
* Document speed and balanced presets (Docling behavior, chunking envelope, inference concurrency)
* Specify chunking methods (recursive, hybrid) and retrieval modes (vector, keyword, hybrid) used in the search space
* Specify HPO-time MaaS vs LangChain split (catalog discovery, embeddings, chat, Ragas vs vector-store upsert/search)

## Non-Goals

* RAG template composition beyond the current simple-RAG search space (see ODH-ADR-0003)
* pattern.json schema, production retrieve-and-generate, and serving (see ODH-ADR-0004)
* Evaluation metric backends and catalog (see ODH-ADR-0005)

## How

Pipeline parameters, presets, **Connections** (MaaS / vector DB / graph DB), and the **chunking / retrieval** search space for **`documents_rag_optimization_pipeline`** ([opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)). ADR context: [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md). Production retrieve-and-generate: [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md).

Search-space preparation runs **before** text extraction so unresponsive or misconfigured MaaS models fail the experiment before Docling work.

## Table of contents

- [Pipeline parameters](#pipeline-parameters)
- [Corpus locations](#corpus-locations)
- [Benchmark JSON](#benchmark-json)
- [Connections](#connections)
- [Presets](#presets)
- [Chunking methods](#chunking-methods)
- [Retrieval methods](#retrieval-methods)
- [Related](#related)

---

## Pipeline parameters

Public surface of [`pipeline.py`](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py) (verify on your **pipelines-components** tag):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `test_data_secret_name` | `str` | (required) | Secret for S3-compatible test-data access (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`; `AWS_DEFAULT_REGION` optional) |
| `test_data_bucket_name` | `str` | (required) | Bucket containing the evaluation benchmark JSON |
| `test_data_key` | `str` | (required) | Object key of the test data file |
| `input_data_secret_name` | `str` | (required) | Secret for document corpus access (same key convention) |
| `input_data_bucket_name` | `str` | (required) | Bucket containing source documents |
| `input_data_keys` | `list[str]` | `[]` | Object keys or prefixes for the document corpus (1–10). See [Corpus locations](#corpus-locations). |
| `maas_secret_name` | `str` | (required) | MaaS Connection (`MAAS_BASE_URL`, `MAAS_API_KEY`). Used for model validation, embeddings, generation, and Ragas evaluation. |
| `db_secret_name` | `str` | (required) | Database Connection selected by the active template: `MILVUS_*` or `PGVECTOR_*` for simple RAG, `NEO4J_*` for Graph RAG ([ODH-ADR-0003](./ODH-ADR-0003-rag-templates.md)). |
| `embedding_models` | `list[str]` | (required) | Non-empty list of embedding model ids for the search space. MaaS does not expose type metadata, so models cannot be inferred. |
| `generation_models` | `list[str]` | (required) | Non-empty list of generation / foundation model ids for the search space. Same reason as `embedding_models`. |
| `optimization_metric` | `str` | `custom:overall_score` | Evaluator-qualified GAM objective ID (for example, `unitxt:faithfulness` or `ragas:context_precision`). Valid IDs depend on the preset: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#optimization_metric). |
| `optimization_max_rag_patterns` | `int` | `5` | Max patterns to evaluate and retain (`max_number_of_rag_patterns`) |
| `preset` | `str` | `speed` | Quality tier — maps to Docling extraction, chunking search space, contextual enrichment, and inference concurrency ([Presets](#presets)) |

`db_secret_name` is required. Its keys must match the selected template: simple RAG uses a vector backend; Graph RAG uses Neo4j, which provides graph, vector, and full-text storage.

---

## Corpus locations

`input_data_keys` is a **list of object keys or prefixes** in `input_data_bucket_name`.

| Rule | Detail |
|------|--------|
| Cardinality | **1–10** locations per experiment. More than 10 is out of scope. |
| Scope | All locations are in the **same** Connection (`input_data_secret_name`) and **same** bucket. Multi-Connection / multi-bucket selection is out of scope. |
| Union | Document discovery, the 1 GB optimization sample, and production indexing consume the **union** of those locations. The sample cap is unchanged. |
| Single location | A one-element list is valid. |
| Indexing | The same list is copied onto `pattern.json` → `indexing.pipeline_spec.parameters.input_data_keys` so full-corpus indexing uses those locations ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#index-building)). |

Empty list is invalid at run time: at least one key or prefix is required.

---

## Benchmark JSON

`test_data_key` points at one JSON file (array of rows) used as the evaluation benchmark. Ground-truth **document identity is the S3 object key**, not a filename.

| Field | Type | Role |
|-------|------|------|
| `question` | `str` | Benchmark question |
| `correct_answers` | `list[str]` | Ground-truth answers |
| `correct_answer_document_keys` | `list[str]` | Object keys of the documents the answers were drawn from |

Each `correct_answer_document_keys` value is a full object key (path), unique across the union of selected corpus locations: the same basename under two prefixes is two keys. Pipeline validation **fails** if `correct_answer_document_keys` is missing, empty, or any key does not match an ingested object.

```json
[
  {
    "question": "What warranty period applies to the XR-200 controller?",
    "correct_answers": [
      "The XR-200 controller has a 24-month warranty."
    ],
    "correct_answer_document_keys": [
      "product-manuals/xr-200-manual.pdf",
      "policies/warranty-policy.pdf"
    ]
  },
  {
    "question": "Which firmware versions support remote diagnostics?",
    "correct_answers": [
      "Firmware 3.2 and later supports remote diagnostics."
    ],
    "correct_answer_document_keys": [
      "release-notes/release-notes-3.2.pdf"
    ]
  }
]
```

Row-level evaluation output (`evaluation_results.json`) does **not** repeat `correct_answer_document_keys`. Retrieved context uses `document_key`: [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md).

---

## Connections

Secrets are Kubernetes Secret **names** (RHOAI Connections). Values are injected as environment variables; they are not pipeline-parameter payloads.

HPO uses **MaaS** for model discovery, embeddings, chat completions, and Ragas evaluation, and **LangChain adapters** for vector upsert and search. AutoRAG does not use a component-specific model registry: the search space is the MaaS catalog constrained by the required `embedding_models` / `generation_models` lists. Production retrieve-and-generate uses the same MaaS Connection ([ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#retrieve-and-generation)).

| Capability | Provider | Notes |
|------------|----------|-------|
| LLM discovery | **MaaS** | List available chat models for the search space |
| Chat completion | **MaaS** | Generation during trials |
| Embeddings | **MaaS** | Embedding calls during trials |
| Ragas evaluation | **MaaS** | LLM and embedding calls ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#ragas-runtime)) |
| Vector upsert / search | LangChain adapters | Milvus or PGVector for simple RAG, configured by `db_secret_name` |

Trial-time path:

```text
chunks
  → embed (MaaS)
  → vector DB upsert (LangChain adapter)
  → LangChain search (vector / keyword / hybrid)
  → MaaS chat completion
  → evaluate metrics (Unitxt; Ragas LLM and embeddings via MaaS)
```

**MaaS** (`maas_secret_name`)

| Key | Role |
|-----|------|
| `MAAS_BASE_URL` | OpenAI-compatible MaaS gateway base URL |
| `MAAS_API_KEY` | API key for discovery, embeddings, chat, and Ragas evaluation |

Mounted on `search_space_preparation`, `models_pre_selector`, and `rag_templates_optimization`.

**Database** (`db_secret_name`)

The active template selects the backend from keys in this secret, mounted on `rag_templates_optimization`:

| Backend | Keys | Selection |
|---------|------|-----------|
| Milvus | `MILVUS_URI` (required), `MILVUS_TOKEN`, `MILVUS_SERVER_CERT` | Any `MILVUS*` env var |
| PGVector | `PGVECTOR_HOST`, `PGVECTOR_PORT`, `PGVECTOR_DB`, `PGVECTOR_USER`, `PGVECTOR_PASSWORD` | Else any `PGVECTOR*` env var |
| Neo4j | `NEO4J_URI` (required; `neo4j://` or `bolt://`), `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE` (optional; default `neo4j`) | Any `NEO4J*` env var |

For simple RAG, missing Milvus and PGVector keys fails validation. Graph RAG requires Neo4j; a missing or non-Neo4j secret fails validation.

**Object storage** (`test_data_secret_name`, `input_data_secret_name`)

`documents_discovery` maps input vs test credentials to `INPUT_DATA_AWS_*` and `TEST_DATA_AWS_*`. `text_extraction` uses unprefixed `AWS_*` from the input-data secret.

---

## Presets

Each preset fixes **ingestion, chunking search-space envelope, contextual enrichment, and inference concurrency**. ai4rag still optimizes embedding, retrieval, and generation (from the required model lists) within that envelope. Both presets use the **same KFP resource tier**.

| Preset | Chunking methods | Chunk sizes | Chunk overlaps | Docling PDF tables | Contextual enrichment | `inference_max_threads` |
|--------|------------------|-------------|----------------|--------------------|-----------------------|-------------------------|
| `speed` (default) | `recursive` only | `128`, `256`, `512` | `32`, `64` | `do_table_structure: false` | none | 10 |
| `balanced` | `recursive` + `hybrid` | `512`, `1024`, `2048` | `0`, `128`, `256` | `do_table_structure: true` ([TableFormer](https://docling-project.github.io/docling/guides/pdf-processing/)) | LLM contextual enrichment on hybrid trials | 4 |

`inference_max_threads` is the benchmark query concurrency in `rag_templates_optimization` (`balanced` is lower because per-request context is larger).

Docling extraction (`text_extraction`) is fixed per run by the preset, not repeated in each `pattern.json`. Use `speed` for plain text or quick runs; `balanced` for structured PDFs/DOCX with tables, headings, and LLM contextual enrichment. Resource sizing is maintained with the pipeline implementation.

---

## Chunking methods

Chunking splits documents into embeddable segments. **`chunking`** fields control boundaries and Docling serialization (including optional structural metadata via `include_metadata`). Query-time behavior lives under **`retrieval`**.

### Methods

| `method` | Input | Behavior | Typical use |
|----------|-------|----------|-------------|
| `recursive` | Flat text or Markdown (`export_to_markdown()` from `DoclingDocument`) | Separator cascade (`\n\n` → `\n` → ` `) with `chunk_size` / `chunk_overlap` | General text, `speed` preset |
| `hybrid` | `DoclingDocument` tree | `HybridChunker`: structure-aware boundaries + tokenizer-aware split/merge; optional `include_metadata` / LLM contextual enrichment | Structured PDFs/DOCX, `balanced` preset |

[Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/) distinguish **Markdown export + post-split** (recursive path) from **native chunkers on the document model** (hybrid).

### Docling hybrid — `include_metadata`

Hybrid-only boolean on `pattern.json` under **`settings.chunking.include_metadata`**. When `true`, indexing inlines structural (and, on `balanced`, LLM) context into the text sent to the embedding model, not as separate vector fields.

| Field | Role |
|-------|------|
| `include_metadata` | Contextualize embed/BM25 text (`hybrid` only; explored on `balanced` preset) |

### Chunking parameters

| Parameter | Notes |
|-----------|-------|
| `method` | `recursive`, `hybrid` |
| `chunk_size`, `chunk_overlap` | Splitter limits (preset envelopes above) |
| `include_metadata` | Hybrid contextualization in embed text |

---

## Retrieval methods

| `search_mode` | Behavior |
|---------------|----------|
| `vector` | Embedding similarity only |
| `keyword` | BM25 only |
| `hybrid` | Vector + BM25 fused with RRF (recommended for production) |

| Parameter | Default | Role |
|-----------|---------|------|
| `method` | `simple` | Query-time retrieval strategy |
| `number_of_chunks` | `5` | Top-k chunks (typical range 3–20) |
| `search_mode` | `hybrid` | `vector`, `keyword`, or `hybrid` |
| `ranker_strategy` | — | `rrf` or `weighted` (hybrid) |
| `ranker_k` | — | RRF constant (typical 60) |
| `ranker_alpha` | — | Weighted fusion: 0 keyword ↔ 1 vector |
| `distance_metric` | `cosine` | Vector similarity metric |

ai4rag explores chunking and retrieval combinations during optimization; GAM selects the best pattern by `optimization_metric` ([ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md#optimization_metric)). Sampling respects `max_combinations` and product search-space rules.

---

## Related

- [RAG templates](./ODH-ADR-0003-rag-templates.md) — simple and Neo4j Graph RAG templates
- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md)
- [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) — metric catalog, preset-valid evaluator-qualified `optimization_metric`, and `evaluation_results.json` `document_key`; benchmark field is defined above
- [documents_rag_optimization_pipeline](https://github.com/opendatahub-io/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)
- [Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/)
