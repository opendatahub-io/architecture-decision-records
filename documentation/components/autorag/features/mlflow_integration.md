# AutoRAG and MLflow (Documents RAG optimization pipeline)

This page describes how **MLflow** relates to **AutoRAG** when you run the **Documents RAG optimization** Kubeflow pipeline from [pipelines-components](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) on **OpenShift AI Data Science Pipelines**.

**Alignment with AutoML:** AutoRAG uses the **same KFP + MLflow integration model** as [AutoML MLflow integration](../../automl/features/mlflow_integration.md): **OpenShift AI 3.5+** injects the standard tracking environment variables into pipeline pods; components treat **`MLFLOW_TRACKING_URI`** as the opt-in switch; **one KFP-managed parent run** (`MLFLOW_RUN_ID`) is resumed for pipeline-level tags, params, and leaderboard artifacts; **nested child runs** capture each comparable unit (for AutoRAG, each **RAG pattern** directory—analogous to each **leaderboard model** in AutoML). There is no `mlflow.autorag` autologger—only **explicit** `mlflow.log_*` calls, for the same reasons AutoML avoids relying on framework autologging under Kubeflow.

## Table of Contents

- [KFP MLflow integration modes (OpenShift AI 3.5+)](#kfp-mlflow-integration-modes-openshift-ai-35)
  - [Environment variables (RHOAI 3.5+)](#environment-variables-rhoai-35)
- [Product intent](#product-intent)
- [KFP artifacts produced by the pipeline (RHOAI 3.5+)](#kfp-artifacts-produced-by-the-pipeline-rhoai-35)
- [MLflow mapping model (AutoRAG)](#mlflow-mapping-model-autorag)
- [AutoRAG artifact → MLflow logging method](#autorag-artifact-mlflow-logging-method)
- [Component-level logging matrix (documents RAG optimization)](#component-level-logging-matrix-documents-rag-optimization)
- [KFP Pipeline Integration: MLflow Tracking Artifact](#kfp-pipeline-integration-mlflow-tracking-artifact)
  - [Artifact purpose](#artifact-purpose)
  - [Component output definition](#component-output-definition)
  - [Artifact schema](#artifact-schema)
- [Related](#related)

---

## KFP MLflow integration modes (OpenShift AI 3.5+)

Starting with **OpenShift AI 3.5**, Data Science Pipelines supports the same two MLflow integration modes described in the [AutoML MLflow integration](../../automl/features/mlflow_integration.md#kfp-mlflow-integration-modes-openshift-ai-35) document:

1. **Automatic KFP-to-MLflow logging:** Simple pipeline metrics and parameters are recorded without custom code.

2. **Opt-in with environment variables (recommended for AutoRAG):** Pipeline pods receive **`MLFLOW_TRACKING_URI`**, **`MLFLOW_RUN_ID`**, **`MLFLOW_EXPERIMENT_ID`**, and related variables so components can call the MLflow Tracking API with full control over **per-pattern** artifacts (`pattern.json`, notebooks, evaluation payloads) and nested runs—mirroring AutoML’s use of mode **2** for AutoGluon leaderboards.

**AutoRAG integration approach:** Components check whether **`MLFLOW_TRACKING_URI`** is set. If yes, they **resume the KFP parent run** and create **nested child runs** (one per RAG pattern), using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.log_artifact()`—the same lifecycle pattern as AutoML’s **one child run per leaderboard row / refitted model**.

### Environment variables (RHOAI 3.5+)

When MLflow integration is enabled at the project level, OpenShift AI injects the same variables into pipeline step pods as for AutoML (see [AutoML — Environment variables](../../automl/features/mlflow_integration.md#environment-variables-rhoai-35)):

| Environment Variable | Purpose | Example Value | Notes |
|---------------------|---------|---------------|-------|
| `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint | `https://mlflow-server.example.com` | If unset, optional AutoRAG MLflow logging is skipped; pipeline and KFP artifacts are unchanged |
| `MLFLOW_WORKSPACE` | Workspace identifier for the project | `data-science-project` | OpenShift AI project / namespace |
| `MLFLOW_EXPERIMENT_ID` | Experiment for this pipeline | `"1"` | KFP-managed experiment |
| `MLFLOW_RUN_ID` | Parent run for this pipeline execution | `"abc123..."` | Components resume this run and create **nested** pattern child runs under it |
| `MLFLOW_TRACKING_AUTH` | Authentication mechanism | `kubernetes-namespaced` | Same Kubernetes-aware auth behavior as AutoML (MLflow SDK expectations apply) |

---

## Product intent

Upstream lists **MLflow >= 2.0.0** as an **external dependency** and describes **experiment tracking** as **optional**: MLflow complements **KFP artifacts** (RAG patterns, HTML leaderboard, Responses API template in **`pattern.json`** for RHOAI 3.5+); the pipeline still completes when tracking is off.

The AutoRAG ADR describes optional logging **during** the **ai4rag** optimization loop (configuration and metrics) and **run-level** summaries when tracking is enabled. See [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md).

When tracking is on, prefer **lazy-importing** `mlflow` and branching on **`MLFLOW_TRACKING_URI`** so behavior matches AutoML: **no tracking URI** means **no MLflow client** and no change to step semantics.

---

## KFP artifacts produced by the pipeline (RHOAI 3.5+)

These paths represent the **RHOAI 3.5+** artifact structure (see upstream `pipeline.py` and `rag_templates_optimization` / `leaderboard_evaluation` sources). They are the **source of truth** for what MLflow logging should attach to.

| Artifact / path role | Producing step | Layout and role |
|---------------------|----------------|-----------------|
| **Test data** | `test_data_loader` | Benchmark JSON on disk (input to search prep + optimization). |
| **Discovered documents** | `documents_discovery` | Descriptor of corpus objects for extraction. |
| **Extracted text** | `text_extraction` | Extracted document text (e.g. via **docling**) for ai4rag. |
| **`search_space_prep_report`** | `search_space_preparation` | YAML-serialized **search space** after phase-one validation. |
| **`rag_patterns`** (directory artifact) | `rag_templates_optimization` | One **subdirectory per pattern** (pattern name = folder name). Under each: **`pattern.json`** (with embedded Responses API template), **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, **`create_model_response.py`**. |
| **HTML leaderboard** | `leaderboard_evaluation` | Single HTML file built from each subdir’s **`pattern.json`**. |

**Per-pattern artifacts (RHOAI 3.5+):**

| Artifact | Type | Purpose |
|----------|------|---------|
| **`pattern.json`** | JSON | **Consolidated pattern record** containing: chunking, embedding, retrieval, generation settings; **Responses API request template** (`input`, `tools`, `instructions`, `metadata`) embedded in `settings.responses_template`; vector store binding; aggregate scores and timing. Single source of truth for registration, deployment, and code generation. |
| **`indexing_notebook.ipynb`** | Jupyter Notebook | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| **`inference_notebook.ipynb`** | Jupyter Notebook | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| **`evaluation_results.json`** | JSON | Per-question evaluation metrics (context precision, answer relevance, faithfulness), traces, metadata |
| **`create_model_response.py`** | Python | Interactive client script for testing patterns (embeds Llama Stack base URL from env, reads Responses config from `pattern.json`) |

---

## MLflow mapping model (AutoRAG)

Design follows the same concepts as the AutoML doc: **one parent run per KFP pipeline execution**, optional **child runs per RAG pattern**, explicit APIs (there is **no** `mlflow.autorag` autologger).

| MLflow concept | Proposed mapping |
|----------------|------------------|
| **Experiment** | **RHOAI 3.5+:** use KFP-managed experiment (`MLFLOW_EXPERIMENT_ID`). **Otherwise:** e.g. `autorag_documents_rag_optimization` plus optional suffix (team, cluster). |
| **Parent run** | **RHOAI 3.5+:** resume KFP parent (`MLFLOW_RUN_ID`). **Tags:** `kfp_run_id`, `pipeline_name` (e.g. `documents-rag-optimization-pipeline`), `namespace` / workspace, non-secret **bucket/prefix hints** or hashes for corpus and test keys—not raw secrets. **Params:** `optimization_metric`, `optimization_max_rag_patterns`, `llama_stack_vector_io_provider_id` (non-secret id), pipeline image / digest if available. |
| **Child runs** | **One nested child run per RAG pattern** (folder name or `pattern.json` `name`), created under the parent during or after **`rag_templates_optimization`**. Enables side-by-side comparison in the MLflow UI for faithfulness / answer_correctness / context_correctness and chunking / retrieval / model choices. |
| **Params (child)** | From **`pattern.json`** → `settings`: e.g. `chunking.*`, `embedding.model_id`, `retrieval.*`, `generation.model_id`. **RHOAI 3.5+:** also log **`vector_store_binding`** fields (`provider_id`, `provider_type`, `vector_store_id`, `vector_store_name`) and key **`responses_template`** fields as flattened params if useful for comparison (keep payloads size-conscious). **Older layouts:** legacy **`vector_store.datasource_type`** / **`collection_name`** when present. Prefer **`mlflow.log_params`** with **flattened keys** (dotted names) or a **curated** subset to avoid UI clutter—same discipline as AutoML child runs. |
| **Metrics (child)** | From **`pattern.json`**: `final_score`, `duration_seconds`, `iteration`; from **`scores`**: per-metric means (e.g. keys aligned with `faithfulness`, `answer_correctness`, `context_correctness`)—use **`mlflow.log_metric`** for scalars. |
| **Metrics (parent)** | Aggregate over children: e.g. `best_final_score`, `num_patterns`, total optimization wall time if available. |
| **Artifacts (child)** | **`pattern.json`** (full JSON with embedded `responses_template`, size-capped), optional **`evaluation_results.json`**, optional **`create_model_response.py`**—see privacy row below. |
| **Artifacts (parent)** | **HTML leaderboard** (same artifact as KFP HTML output; cap file size like AutoML), optional **`search_space_prep_report`** YAML. |

---

## AutoRAG artifact → MLflow logging method

Use **explicit** MLflow Tracking APIs (`mlflow.log_param`, `mlflow.log_params`, `mlflow.log_metric`, `mlflow.log_metrics`, `mlflow.log_artifact`, `mlflow.set_tags`). Lazy-import **mlflow** when `MLFLOW_TRACKING_URI` is unset so pipeline steps behave the same with tracking disabled.

| Source (KFP / disk) | MLflow method | What to record | Typical run scope |
|---------------------|---------------|----------------|-------------------|
| KFP pipeline **inputs** (`optimization_metric`, `optimization_max_rag_patterns`, `input_data_key`, `test_data_key`, vector I/O provider id) | `log_params` / `set_tags` | Optimization objective and non-secret location hints | Parent |
| **`search_space_prep_report`** (YAML) | `log_artifact` | Full or redacted YAML as a file | Parent |
| **`pattern.json`** (per pattern) | `log_params` + `log_metrics` + optional `log_artifact` | Flatten `settings.*` into params; `final_score`, `duration_seconds`, scalar aggregates from `scores` as metrics; attach JSON file for reproducibility | Child |
| **`evaluation_results.json`** (per pattern) | `log_artifact` **or omit** | Per-question answers and contexts | **Child, optional** — often contains **PII / sensitive text**; prefer logging **only** aggregates already in `pattern.json` / `scores`, or a **redacted** copy under governance rules |
| **`indexing_notebook.ipynb`** / **`inference_notebook.ipynb`** | `log_artifact` (optional) | Indexing / inference notebooks | Child — can be large; consider size limits or store URI only in a tag |
| **`create_model_response.py`** (per pattern) | `log_artifact` (optional) | Interactive client script (no API keys in file) | Child |
| **Leaderboard HTML** | `log_artifact` | Single HTML table | Parent |
| **`autorag_run_summary`** / run log (when present) | `log_artifact` | Markdown or text summary | Parent |

---

## Component-level logging matrix (documents RAG optimization)

This matrix is **implementation guidance** aligned with upstream step names. Cells list **intended** MLflow operations when `MLFLOW_TRACKING_URI` is set; confirm what is actually merged in the **pipelines-components** branch you ship.

| Component | MLflow operation | Parameters logged (examples) | Metrics logged (examples) | Artifacts logged | Run type |
|-----------|------------------|------------------------------|---------------------------|------------------|----------|
| **test_data_loader** | Params / metrics | `test_data_key` (if non-sensitive) | `test_data_questions_count` (if derived), file size | — (avoid raw JSON artifact if sensitive) | Parent |
| **documents_discovery** | Metrics | — | `documents_discovered_count` | — | Parent |
| **text_extraction** | Metrics | — | `documents_extracted`, `extracted_chars` or similar | — | Parent |
| **search_space_preparation** | Params + artifact | Embedding / generation model list fingerprints | `search_space_size` or parameter count | `search_space_prep_report` (YAML) | Parent |
| **rag_templates_optimization** | **Parent tags + child runs** | (parent) `optimization_metric`, `max_number_of_rag_patterns`, provider id | (parent) `num_patterns`, `best_final_score`, wall-clock if available | — | Parent + **N children** |
| **rag_templates_optimization** (per pattern) | Child run | From **`pattern.json`** `settings.*` (including embedded `responses_template`) | `final_score`, `duration_seconds`, per-metric means from `scores` | `pattern.json`; optional `evaluation_results.json`, `indexing_notebook.ipynb`, `inference_notebook.ipynb`, `create_model_response.py` | Child (nested) |
| **leaderboard_evaluation** | Artifact | — | — | `leaderboard.html` (or KFP output name); optional **`mlflow_tracking_artifact`** JSON (see [KFP Pipeline Integration: MLflow Tracking Artifact](#kfp-pipeline-integration-mlflow-tracking-artifact))—same contract as AutoML when upstream implements it | Parent |

---

## KFP Pipeline Integration: MLflow Tracking Artifact

AutoRAG should use the **same dedicated KFP output** as [AutoML’s MLflow tracking artifact](../../automl/features/mlflow_integration.md#kfp-pipeline-integration-mlflow-tracking-artifact): a JSON file exposed as a pipeline step output so the **AutoRAG Dashboard**, KFP UI, and automation can resolve **MLflow experiment ID**, **parent run ID**, and **deep links** without scraping logs or guessing from workspace filters alone.

**Target producer:** the **`leaderboard_evaluation`** step in **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)** (alongside the HTML leaderboard) is the logical component to emit **`mlflow_tracking_artifact`**. The **[`documents_indexing_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline)** is a **separate** pipeline; if it later emits a tracking artifact for indexing-only runs, reuse the **same JSON shape** for consistency.

**Implementation:** Until this output exists in the **red-hat-data-services / opendatahub-io** branch you deploy, treat the following as the **product contract** to implement and verify—mirroring the AutoML field definitions [here](../../automl/features/mlflow_integration.md#artifact-schema).

### Artifact purpose

The **`mlflow_tracking_artifact`** JSON is the **single source of truth** for linking a KFP pipeline execution to MLflow tracking:

1. **Dashboard integration:** AutoRAG Dashboard reads the artifact via the KFP API to obtain `mlflow_experiment_id`, `mlflow_run_id`, and UI URLs.
2. **Deep-linking:** Users jump from Data Science Pipelines to the MLflow UI for the parent run (and nested pattern child runs).
3. **Programmatic access:** CI/CD and scripts resolve tracking IDs without parsing component logs.
4. **Audit trail:** Stable record of which MLflow parent run belongs to which KFP run.

### Component output definition

The leaderboard (or equivalent final) component declares a KFP artifact output—for example:

```python
mlflow_tracking_artifact: dsl.Output[dsl.Artifact]
```

The step writes **one JSON file** to `mlflow_tracking_artifact.path` before completing.

### Artifact schema

**When MLflow tracking is enabled** (`MLFLOW_TRACKING_URI` set during the run):

```json
{
  "tracking_enabled": true,
  "mlflow_tracking_uri": "https://mlflow-server.example.com",
  "mlflow_experiment_id": "5",
  "mlflow_run_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
  "mlflow_workspace": "data-science-project",
  "mlflow_run_url": "https://mlflow-server.example.com/#/experiments/5/runs/a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "documents-rag-optimization-pipeline"
}
```

| Field | Type | Typical source | Description |
|-------|------|----------------|-------------|
| `tracking_enabled` | bool | Derived | `true` when tracking was active for this execution |
| `mlflow_tracking_uri` | string | `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint |
| `mlflow_experiment_id` | string | `MLFLOW_EXPERIMENT_ID` | KFP-managed experiment for this pipeline |
| `mlflow_run_id` | string | `MLFLOW_RUN_ID` | **Parent** MLflow run for this pipeline execution (pattern runs nest under it) |
| `mlflow_workspace` | string | `MLFLOW_WORKSPACE` | OpenShift AI project / namespace |
| `mlflow_run_url` | string | Computed | Deep link to the parent run in the MLflow UI |
| `kfp_run_id` | string | KFP context / placeholder | Kubeflow pipeline run ID for cross-reference |
| `pipeline_name` | string | KFP context / placeholder | e.g. `documents-rag-optimization-pipeline` |

**When MLflow tracking is disabled** (`MLFLOW_TRACKING_URI` unset):

```json
{
  "tracking_enabled": false,
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "documents-rag-optimization-pipeline"
}
```

---

## Related

- AutoRAG component overview: [../README.md](../README.md)
- AutoRAG ADR: [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- Shared platform MLflow details: [AutoML MLflow integration](../../automl/features/mlflow_integration.md)
- Data Science Pipelines architecture: [../../pipelines/README.md](../../pipelines/README.md)
