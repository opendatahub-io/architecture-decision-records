# RAG Pattern Vector Store Creation

This page documents **AutoRAG pattern vector store creation** from optimization completion through production knowledge base building.

## Table of contents

**RHOAI 3.4 (current)**

- [Current approach](#current-approach)
- [Indexing notebook artifacts](#indexing-notebook-artifacts)
- [Manual vector store rebuild workflow](#manual-vector-store-rebuild-workflow)
- [Limitations](#limitations)

**RHOAI 3.5 and later (planned)**

- [Documents Indexing Pipeline](#documents-indexing-pipeline)
- [Per-pattern compiled pipelines](#per-pattern-compiled-pipelines)
- [Pipeline architecture](#pipeline-architecture)
- [Generated artifacts](#generated-artifacts)
- [Automated deployment workflow](#automated-deployment-workflow)
  - [AutoRAG Dashboard → AI Pipelines integration](#autorag-dashboard--ai-pipelines-integration)

---

## Current state (RHOAI 3.4)

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

### Manual vector store rebuild workflow

Operators and data scientists follow a **manual** notebook-based workflow:

1. **Pipeline completes** → artifacts written to object storage, including `indexing_notebook.ipynb` per pattern
2. **Download** the indexing notebook from the pipeline UI or object store
3. **Configure environment:**
   - Set vector store connection credentials (DSN, API keys, certificates)
   - Mount or configure access to source documents
   - Ensure embedding model endpoint is accessible
4. **Execute notebook:**
   - Run cells sequentially to process documents
   - Monitor progress (chunking, embedding, indexing)
   - Handle errors manually (connection failures, quota limits, etc.)
5. **Verify vector store:**
   - Check collection exists and has expected document count
   - Test retrieval with sample queries
   - Validate embeddings and metadata

### Limitations

The notebook-based approach has several limitations for production deployments:

1. **Manual execution:** Requires human intervention to run cells, handle errors, and verify completion
2. **No orchestration:** Cannot be triggered automatically when source documents change or patterns are updated
3. **Limited observability:** No standardized logging, metrics, or progress tracking across notebook executions
4. **Environment dependencies:** Requires setting up Jupyter environment with correct libraries, credentials, and network access
5. **No versioning:** Difficult to track which version of the indexing code was used for a given vector store build
6. **Error handling:** Manual error recovery; no automatic retries or failure notifications
7. **Scalability:** Notebook kernel resource limits; difficult to parallelize chunking or embedding for large document sets

---

## RHOAI 3.5 and later

### Documents Indexing Pipeline

RHOAI 3.5 introduces a **production-ready KFP pipeline** for vector store creation based on the **[`documents_indexing_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline)** in the pipelines-components repository.

This pipeline automates the entire indexing workflow as a reusable, orchestrated, and observable Kubeflow Pipelines task. Instead of manually executing notebooks, operators can deploy patterns by simply running the pattern-specific compiled pipeline with a single click.

**Architecture:** AutoRAG generates a **compiled, pre-configured pipeline YAML per pattern** with all settings from `pattern.json` baked in as defaults. This provides a superior user experience—operators just click "Deploy" without needing to provide configuration parameters.

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

#### AutoRAG Dashboard → AI Pipelines integration

**Step-by-step user workflow:**

1. **Pattern selection in AutoRAG Dashboard:**
   - User completes pattern optimization (or views existing patterns)
   - Dashboard displays optimized patterns with evaluation metrics
   - User selects a pattern to deploy

2. **Save to AI Pipelines:**
   - User clicks "Deploy Pattern" or "Save to Pipelines" button
   - Dashboard uploads `indexing_pipeline.yaml` to Data Science Pipelines (KFP)
   - Pipeline is registered in the user's AI Pipelines project
   - Pipeline appears in RHOAI Pipelines UI with pattern name (e.g., "pattern-01-indexing")

3. **Run and monitor via AI Pipelines UI:**
   - User navigates to **Data Science Pipelines** → **Pipelines** in RHOAI
   - Locates the pattern indexing pipeline
   - Clicks "Create run" to start pipeline execution
   - Provides runtime parameters
   - Monitors execution in Pipelines UI:
     - Real-time step progress
     - Component logs (chunking, embedding, indexing)
     - Metrics and artifacts
     - Error messages and debugging info
4. **Post-completion:**
   - Pipeline run completes successfully → vector store is ready
   - User can view run artifacts (stats, logs) in Pipelines UI
   - User can trigger re-runs (e.g., when source documents update) by clicking "Clone run"

**Benefits of this integration:**

- ✅ **Single platform experience:** AutoRAG optimization → AI Pipelines deployment, all within RHOAI UI
- ✅ **No manual pipeline upload:** Dashboard automates KFP registration
- ✅ **Standard monitoring:** Users leverage existing AI Pipelines UI skills for monitoring
- ✅ **Run history:** All indexing runs tracked in Pipelines UI, enabling auditing and comparison
- ✅ **Easy re-runs:** Clone existing runs to rebuild vector stores when documents change

**Pipeline execution options:**

1. **AutoRAG Dashboard → AI Pipelines UI (recommended):**
   - **Step 1:** AutoRAG Dashboard "Deploy Pattern" button saves compiled pipeline to AI Pipelines project
   - **Step 2:** User navigates to Data Science Pipelines UI in RHOAI
   - **Step 3:** User creates and monitors pipeline runs via standard Pipelines UI
   - **Benefits:** Integrated RHOAI experience, visual monitoring, run history, easy re-runs
   - **Target users:** Data scientists and operators using RHOAI Dashboard

2. **Direct API invocation (automation/CI/CD):**
   - CI/CD pipelines or automation scripts call the KFP API directly
   - Example: `kfp run create --pipeline-id pattern-01-indexing --parameter document_source=s3://my-docs/`
   - **Benefits:** Scriptable, automated, GitOps-friendly
   - **Target users:** Platform teams, DevOps engineers

3. **Scheduled (batch processing):**
   - Cron-based pipeline runs to keep vector store in sync with changing documents
   - Configured via KFP recurring runs or external schedulers
   - **Benefits:** Regular refresh without manual intervention
   - **Target users:** Knowledge bases with daily/weekly document updates

---

## Related Documentation

- [RAG Pattern Inference](./rag_pattern_inference.md) - AutoRAG pattern deployment and inference workflow
- [Documents Indexing Pipeline](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline) - Pipeline implementation in pipelines-components repository
