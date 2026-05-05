# RAG Pattern Vector Store Creation

This page documents **AutoRAG pattern vector store creation** from optimization completion through production knowledge base building.

## Table of Contents

**RHOAI 3.4**

- [Current approach](#current-approach)
- [Indexing notebook artifacts](#indexing-notebook-artifacts)

**RHOAI 3.5 and later**

- [Documents Indexing Pipeline](#documents-indexing-pipeline)
- [Per-pattern compiled pipelines](#per-pattern-compiled-pipelines)
- [Pipeline architecture](#pipeline-architecture)
- [Generated artifacts](#generated-artifacts)
- [Automated deployment workflow](#automated-deployment-workflow)

**Related**

- [Related Documentation](#related-documentation)

---

## RHOAI 3.4

### Current approach

In RHOAI 3.4, vector store creation for optimized RAG patterns relies on **Jupyter notebooks** generated during the optimization pipeline. These notebooks contain parameterized code for rebuilding the vector store with the exact chunking, embedding, and indexing settings that were used during pattern optimization.

The **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/autorag/documents_rag_optimization_pipeline/pipeline.py)** writes indexing notebooks as part of the `rag_patterns` artifact output.

### Indexing notebook artifacts

Each optimized pattern's artifact directory (`<pattern_subdir>/`) includes an **`indexing_notebook.ipynb`** file:

| File | Type | Purpose |
|------|------|---------|
| `indexing_notebook.ipynb` | Jupyter Notebook | **Indexing** notebook instantiated from templates (e.g., **`ls_indexing_template.ipynb`**), parameterized for this pattern's chunking, embedding, and vector store settings. Contains all code needed to rebuild the vector store from source documents. |

**What's in the notebook:**
- Document loading and preprocessing code
- Chunking logic parameterized with the optimized `chunk_size` and `chunk_overlap` from `pattern.json`
- Embedding model configuration matching the pattern's `embedding.model_id`
- Vector store connection and indexing code for the specified `datasource_type` (e.g., Milvus, PGVector)
- Collection/index creation with the optimized settings (distance metric, dimension, etc.)

---

## RHOAI 3.5 and later

### Documents Indexing Pipeline

RHOAI 3.5 introduces a **production-ready KFP pipeline** for vector store creation based on the **[`documents_indexing_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline)** in the pipelines-components repository.

This pipeline automates the entire indexing workflow as a reusable, orchestrated, and observable Kubeflow Pipelines task. Instead of manually executing notebooks, operators can deploy patterns by simply running the pattern-specific compiled pipeline with a single click.

**Architecture:** 
- AutoRAG image contains the index building pipeline & pipeline components source code.
- AutoRAG generates a **compiled, pre-configured pipeline YAML per pattern** with all settings from `pattern.json` baked in as defaults. This provides a superior user experience—operators just click "Deploy" without needing to provide configuration parameters.
- AutoRAG injects the currently used image digest to compiled pipeline.
 

### Per-pattern compiled pipelines

AutoRAG generates a **compiled, pre-configured KFP pipeline YAML** per optimized pattern. Each pipeline has all `pattern.json` values **pre-filled as defaults** in the pipeline specification, enabling one-click deployment without parameter configuration.

**Architecture:**

```
AutoRAG Optimization Completes
    │
    ├─> Generates pattern.json (settings/configuration)
    │
    └─> Compiles indexing_pipeline.yaml
          (pipeline with defaults pre-filled from pattern.json)
```

**Artifacts per pattern:**

```
<pattern_01>/
  ├── pattern.json                           # Pattern configuration (source of truth)
  ├── indexing_notebook.ipynb                
  ├── inference_notebook.ipynb
  └── indexing_pipeline.yaml                 # NEW: Compiled KFP pipeline
```
**Benefits:**

- ✅ **One-click deployment** - Dashboard users just click "Deploy Pattern", no configuration needed
- ✅ **Self-contained** - All necessary assets (config, notebooks, pipeline) in one pattern folder
- ✅ **Discoverable settings** - Inspect pipeline YAML to see exact configuration without reading JSON
- ✅ **Self-documenting** - Pipeline spec shows what will run, making it easy to review before execution
- ✅ **Aligned with 3.4** - Conceptually consistent with per-pattern notebooks approach
- ✅ **Override flexibility** - Advanced users can still override individual parameters at runtime
- ✅ **Version tracking** - Each pattern has its own pipeline version, simplifying rollback and auditing

### Pipeline architecture

The **`documents_indexing_pipeline`** orchestrates vector store creation through a series of containerized components:

```
┌─────────────────────────────────────────────────────────────────┐
│ Documents Indexing Pipeline                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Load Pattern Config                                         │
│     └─> Read pattern.json (chunking, embedding, vector store)   │
│                                                                 │
│  2. Document Ingestion                                          │
│     └─> Load documents from S3/PVC/URL                          │
│     └─> Validate document formats                               │
│                                                                 │
│  3. Chunking                                                    │
│     └─> Apply chunking strategy (recursive, semantic, etc.)     │
│     └─> Parameterized by pattern.json settings                  │
│                                                                 │
│  4. Embedding Generation                                        │
│     └─> Call embedding model endpoint                           │
│     └─> Batch processing for efficiency                         │
│     └─> Retry logic for transient failures                      │
│                                                                 │
│  5. Vector Store Indexing                                       │
│     └─> Create/update collection/index                          │
│     └─> Insert embeddings with metadata                         │
│     └─> Configure distance metric, dimension, etc.              │
│                                                                 │
│  6. Validation & Metrics                                        │
│     └─> Verify document count                                   │
│     └─> Log indexing statistics (time, tokens, errors)          │
│     └─> Write completion artifact                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key pipeline features:**

- **Parameterized by `pattern.json`:** The pipeline accepts the optimized pattern configuration as input, ensuring production vector store uses identical settings to optimization
- **Scalable:** Components can request appropriate resources (CPU, memory, GPU) for large-scale document processing
- **Observable:** Each step logs metrics to MLflow/KFP, enabling monitoring and debugging
- **Reusable:** Same pipeline can rebuild vector stores when documents change or patterns are updated
- **Automated triggers:** Can be invoked via API, scheduled (cron), or event-driven (document upload)

### Generated artifacts

The indexing pipeline produces artifacts that complement the pattern artifacts:

| Artifact | Type | Purpose |
|----------|------|---------|
| `indexing_run_metadata.json` | JSON | Pipeline execution metadata: start time, end time, duration, component versions, resource usage |
| `vector_store_stats.json` | JSON | Vector store statistics: document count, chunk count, embedding dimension, collection name, index size |
| `indexing_logs/` | Directory | Per-component logs for debugging (document loading, chunking, embedding, indexing) |
| `pattern_config_snapshot.json` | JSON | Copy of the `pattern.json` used for this indexing run, for reproducibility and auditing |

**Example `vector_store_stats.json`:**

```json
{
  "collection_name": "coll_pattern_01_prod",
  "vector_store_type": "milvus",
  "document_count": 1523,
  "chunk_count": 8947,
  "embedding_model": "text-embedding-3-small",
  "embedding_dimension": 768,
  "distance_metric": "cosine",
  "index_size_mb": 2341,
  "indexing_duration_seconds": 428,
  "created_at": "2026-04-27T10:30:00Z",
  "pattern_id": "pattern_01",
  "pattern_version": "v1"
}
```

### Automated deployment workflow

The compiled indexing pipeline enables automated vector store deployment with seamless integration between AutoRAG Dashboard and Data Science Pipelines (AI Pipelines) UI:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Pattern Optimization Completes                               │
│    └─> pattern.json written to artifact store                   │
│    └─> indexing_pipeline.yaml compiled (per pattern folder)     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. AutoRAG Dashboard Integration                                │
│    └─> User views optimized patterns in AutoRAG Dashboard       │
│    └─> Clicks "Deploy Pattern" on selected pattern              │
│    └─> Dashboard saves indexing_pipeline.yaml to KFP            │
│         as pipeline in AI Pipelines project                     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. AI Pipelines UI (Data Science Pipelines)                     │
│    └─> User navigates to AI Pipelines in RHOAI                  │
│    └─> Sees pattern indexing pipeline registered                │
│    └─> Creates pipeline run with parameters:                    │
│         • document_source (S3 bucket, PVC, URL)                 │
│         • vector_store_dsn (if not default)                     │
│         • credentials (secrets)                                 │
│    └─> Monitors execution via Pipelines UI                      │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Pipeline Executes                                             │
│    └─> Loads documents                                           │
│    └─> Chunks, embeds, indexes according to pattern.json        │
│    └─> Logs progress to MLflow and KFP UI                       │
│    └─> User monitors progress, logs, and metrics in Pipelines   │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Vector Store Ready                                            │
│    └─> Collection created and populated                         │
│    └─> Stats and logs written to artifact store                 │
│    └─> Completion status visible in AI Pipelines UI             │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Pattern Inference Deployment                                 │
│    └─> Llama Stack configured to use new vector store           │
│    └─> /v1/responses API ready for queries                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Related Documentation

- [RAG Pattern Inference](./rag_pattern_inference.md) - AutoRAG pattern deployment and inference workflow
- [Documents Indexing Pipeline](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline) - Pipeline implementation in pipelines-components repository
