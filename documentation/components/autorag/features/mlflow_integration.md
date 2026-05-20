# AutoRAG and MLflow (Documents RAG optimization pipeline)

This page describes how **MLflow** relates to **AutoRAG** when you run the **Documents RAG optimization** Kubeflow pipeline from [pipelines-components](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) on **OpenShift AI Data Science Pipelines**.

**Alignment with AutoML:** AutoRAG uses the **same KFP + MLflow integration model** as [AutoML MLflow integration](../../automl/features/mlflow_integration.md): components treat **`MLFLOW_TRACKING_URI`** as the opt-in switch; **one KFP-managed parent run** (`MLFLOW_RUN_ID`) is resumed for pipeline-level tags and params; **nested child runs** capture each comparable unit (for AutoRAG, each **RAG pattern** directory—analogous to each **leaderboard model** in AutoML). When MLflow tracking is enabled, **traces and stage spans are required**: **one trace per benchmark request** with **`autorag.retrieval`**, **`autorag.generation`**, and **`autorag.evaluation`** spans—see [Tracing per pattern child run](#tracing-per-pattern-child-run).

## Table of Contents

- [KFP MLflow integration (OpenShift AI 3.5+)](#kfp-mlflow-integration-openshift-ai-35)
  - [Environment variables (RHOAI 3.5+)](#environment-variables-rhoai-35)
- [KFP artifacts produced by the pipeline (RHOAI 3.5+)](#kfp-artifacts-produced-by-the-pipeline-rhoai-35)
- [MLflow mapping model (AutoRAG)](#mlflow-mapping-model-autorag)
- [Minimal implementation (first release)](#minimal-implementation-first-release)
  - [What to implement](#what-to-implement)
  - [Reference implementation](#reference-implementation)
  - [Metrics logged (child runs)](#metrics-logged-child-runs)
- [Tracing per pattern child run](#tracing-per-pattern-child-run)
  - [Trace and run hierarchy](#trace-and-run-hierarchy)
  - [Spans and span types](#spans-and-span-types)
  - [Implementation approach](#implementation-approach)
  - [Verification](#verification)
- [Related](#related)

---

## KFP MLflow integration (OpenShift AI 3.5+)

**AutoRAG integration approach:** Components check whether **`MLFLOW_TRACKING_URI`** is set. If yes, they **resume the KFP parent run** and create **nested child runs** (one per RAG pattern), using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.set_tags()` to log pattern metadata and pointers to KFP artifacts—the same lifecycle pattern as AutoML’s **one child run per leaderboard row / refitted model**.

### Environment variables (RHOAI 3.5+)

When MLflow integration is enabled at the project level, OpenShift AI injects the same variables into pipeline step pods as for AutoML (see [AutoML — Environment variables](../../automl/features/mlflow_integration.md#environment-variables-rhoai-35)).

---

## KFP artifacts produced by the pipeline (RHOAI 3.5+)

These paths represent the **RHOAI 3.5+** artifact structure (see upstream `pipeline.py` and `rag_templates_optimization` sources). They are the **source of truth** for what MLflow logging should attach to. The **`leaderboard_evaluation`** step produces an HTML leaderboard from the same **`pattern.json`** files; it does not add separate MLflow metrics.

| Artifact / path role | Producing step | Layout and role |
|---------------------|----------------|-----------------|
| **Test data** | `test_data_loader` | Benchmark JSON on disk (input to search prep + optimization). |
| **Discovered documents** | `documents_discovery` | Descriptor of corpus objects for extraction. |
| **Extracted text** | `text_extraction` | Extracted document text (e.g. via **docling**) for ai4rag. |
| **`search_space_prep_report`** | `search_space_preparation` | YAML-serialized **search space** after phase-one validation. |
| **`rag_patterns`** (directory artifact) | `rag_templates_optimization` | One **subdirectory per pattern** (pattern name = folder name). Under each: **`pattern.json`** (with embedded Responses API template), **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, **`create_model_response.py`**. |



**Per-pattern artifacts:**

| Artifact | Type | Purpose |
|----------|------|---------|
| **`pattern.json`** | JSON | **Consolidated pattern record** containing: chunking, embedding, retrieval, generation settings; **Responses API request template** (`input`, `tools`, `instructions`, `metadata`) embedded in `settings.responses_template`; vector store binding; aggregate scores and timing. Single source of truth for registration, deployment, and code generation. |
| **`indexing_notebook.ipynb`** | Jupyter Notebook | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| **`inference_notebook.ipynb`** | Jupyter Notebook | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| **`evaluation_results.json`** | JSON | Per-question evaluation metrics (context precision, answer relevance, faithfulness), traces, metadata |
| **`create_model_response.py`** | Python | Interactive client script for testing patterns (embeds Llama Stack base URL from env, reads Responses config from `pattern.json`) |

---

## MLflow mapping model (AutoRAG)

Design follows the same concepts as the AutoML doc: **one parent run per KFP pipeline execution**; **one nested child run per RAG pattern** when MLflow tracking is enabled (metrics, params, and traces).

| MLflow concept | Proposed mapping                                                                                                                                                                                                                                                                           |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Experiment** | Use KFP-managed experiment (`MLFLOW_EXPERIMENT_ID`). **Otherwise:** e.g. `autorag_documents_rag_optimization` plus optional suffix (team, cluster).                                                                                                                                        |
| **Parent run** | Resume KFP parent (`MLFLOW_RUN_ID`). **Tags:** `kfp_run_id`, `kfp_run_name`, `pipeline_name`, `dataset hashes or URIs (non-secret)`. **Params:** `optimization_metric`, `optimization_max_rag_patterns`, `llama_stack_vector_io_provider_id`, `image`, `kfp_version`, `ai4rag_version`     |
| **Child runs** | **One nested child run per RAG pattern** (folder name or `pattern.json` `name`). Enables side-by-side comparison in the MLflow UI for faithfulness / answer_correctness / context_correctness and chunking / retrieval / model choices.                                                    |
| **Traces** | **Required** when `MLFLOW_TRACKING_URI` is set. **One trace per benchmark request**, attached to the **pattern child run** (not the parent). If the eval set has *N* rows and the pipeline produces *P* patterns, expect **P × N** traces total (*N* under each pattern’s child run). |
| **Spans** | **Required** under each trace: `autorag.retrieval`, `autorag.generation`, `autorag.evaluation` with [MLflow `SpanType`](https://mlflow.org/docs/latest/genai/concepts/span#span-types) where applicable. Generation may include nested spans from `mlflow.openai.autolog()` for OGX `responses.create`. |
| **Params (child)** | From **`pattern.json`** → `settings`: e.g. `chunking.*`, `embedding.model_id`, `retrieval.*`, `generation.model_id`, `vector_store_binding`** fields (`provider_id`, `provider_type`, `vector_store_id`, `vector_store_name`) and key **`responses_template`** fields as flattened params. |
| **Metrics (child)** | From **`pattern.json`**: `final_score`, `duration_seconds`, `iteration`; from **`scores`**: per-metric means (`faithfulness`, `answer_correctness`, `context_correctness`) — use **`mlflow.log_metric`** for scalars.                                                                      |
| **Child Artifacts** | Pointers to KFP artifacts: **`pattern.json`**, **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, **`create_model_response.py`**. Logged as params containing artifact URIs/paths, not copied into MLflow. |

---

## Minimal implementation (first release)

**Goal:** MLflow integration aligned with [AutoML](../../automl/features/mlflow_integration.md): parent/child runs, metrics, **mandatory per-benchmark traces**, and **stage spans** (`autorag.retrieval`, `autorag.generation`, `autorag.evaluation`) on each pattern child run when `MLFLOW_TRACKING_URI` is set. All logging is driven from **`rag_templates_optimization`** using each pattern’s **`pattern.json`** (and related KFP paths under `rag_patterns/`). No separate MLflow work in **`leaderboard_evaluation`**—the HTML leaderboard and MLflow UI both reflect the same pattern-level scores already written during optimization.

### What to implement

| Step | Where | Action |
|------|-------|--------|
| 1 | `rag_templates_optimization` | If `MLFLOW_TRACKING_URI` is unset → skip all MLflow code (lazy-import `mlflow`). |
| 2 | Same | Resume parent run: `mlflow.start_run(run_id=os.environ["MLFLOW_RUN_ID"])`. Log parent **params/tags** (optimization metric, max patterns, search-space URI, non-secret dataset hints). |
| 3 | Same, before optimization | `mlflow.tracing.enable()` and `mlflow.openai.autolog()` (see [Tracing per pattern child run](#tracing-per-pattern-child-run)). |
| 4 | Per pattern (in ai4rag or component) | Open a **nested child run**; for each benchmark row create **one trace** with **`autorag.*` spans** and `SpanType`s (see [Spans and span types](#spans-and-span-types)); then `log_params` / `log_metrics` / KFP pointers from **`pattern.json`**. |

**Parent run discovery:** KFP-injected **`MLFLOW_RUN_ID`** and **`MLFLOW_EXPERIMENT_ID`** are sufficient to open the pipeline run and nested pattern child runs in the MLflow UI. A separate **`mlflow_tracking_artifact`** JSON step is **not required**.

**Explicitly out of scope for the first release:**

- Copying `evaluation_results.json` content into MLflow (KFP remains source of truth; log URI only if needed)
- OTel Collector, `mlflow.genai.evaluate()` on traces, `log_expectation` / `log_feedback`, `mlflow.log_input()` on parent run

### Reference implementation

```python
import os
import json
from pathlib import Path


def log_pattern_to_mlflow(pattern: dict, pattern_dir: Path, parent_run_id: str) -> None:
    """Log params/metrics after eval. For traces, eval must run inside this child-run context—see Tracing section."""
    if not os.getenv("MLFLOW_TRACKING_URI"):
        return
    import mlflow

    settings = pattern.get("settings", pattern)
    with mlflow.start_run(run_id=parent_run_id):
        with mlflow.start_run(run_name=pattern["name"], nested=True):
            mlflow.log_params({
                "embedding_model": settings.get("embedding", {}).get("model_id", ""),
                "generation_model": settings.get("generation", {}).get("model_id", ""),
                "chunk_size": str(settings.get("chunking", {}).get("chunk_size", "")),
                "retrieval_top_k": str(settings.get("retrieval", {}).get("top_k", "")),
            })
            mlflow.log_metric("final_score", pattern.get("final_score", 0))
            if "duration_seconds" in pattern:
                mlflow.log_metric("duration_seconds", pattern["duration_seconds"])
            for name, value in pattern.get("scores", {}).items():
                mlflow.log_metric(name, value)
            mlflow.log_param("pattern_json_path", str(pattern_dir / "pattern.json"))


def rag_templates_optimization(...):
    parent_run_id = os.getenv("MLFLOW_RUN_ID")
    tracking = bool(os.getenv("MLFLOW_TRACKING_URI") and parent_run_id)

    if tracking:
        import mlflow
        mlflow.set_tracking_uri(os.environ["MLFLOW_TRACKING_URI"])
        mlflow.tracing.enable()
        mlflow.openai.autolog()
        with mlflow.start_run(run_id=parent_run_id):
            mlflow.log_params({
                "optimization_metric": optimization_metric,
                "optimization_max_rag_patterns": str(optimization_max_rag_patterns),
            })

    # Traces + autorag.* spans: one per benchmark row per pattern child run—see Tracing section.
    patterns = optimizer.optimize(...)

    output_dir = Path(rag_patterns.path)
    for pattern in patterns:
        pattern_dir = output_dir / pattern["name"]
        pattern_dir.mkdir(parents=True, exist_ok=True)
        with open(pattern_dir / "pattern.json", "w") as f:
            json.dump(pattern, f, indent=2)
        if tracking:
            log_pattern_to_mlflow(pattern, pattern_dir, parent_run_id)

    return {"num_patterns": len(patterns), "best_score": max(p["final_score"] for p in patterns)}
```

### Metrics logged (child runs)

ai4rag already computes RAGAS-style metrics during optimization. Log **aggregates only** on the pattern child run:

| Source field | MLflow |
|--------------|--------|
| `final_score` | `log_metric("final_score", ...)` |
| `scores` (e.g. `faithfulness`, `answer_correctness`, `context_precision`) | `log_metric` per key |
| `duration_seconds` | `log_metric("duration_seconds", ...)` |

Per-question rows stay in KFP **`evaluation_results.json`**.

---

## Tracing per pattern child run

When `MLFLOW_TRACKING_URI` is set, traces and **stage spans** are **mandatory**, scoped to the **pattern child run**: one MLflow trace per **benchmark row**, each with **`autorag.retrieval`**, **`autorag.generation`**, and **`autorag.evaluation`** spans (and `SpanType` where applicable). Aggregate metrics still go on the same child run via `log_metric`.

### Trace and run hierarchy

| Level | Scope | Count (example: 3 patterns, 10 benchmark rows) |
|-------|-------|--------------------------------------------------|
| **Parent run** | One KFP pipeline execution | 1 |
| **Child run** | One RAG pattern | 3 |
| **Trace** | One eval request (one benchmark row through that pattern’s OGX call path) | **10 per child run** → **30 total** |

```
Parent run (pipeline)
└── Child run: pattern_A
    ├── Trace: autorag.pattern_A.query_0
    │   ├── autorag.retrieval      (SpanType.RETRIEVER)
    │   ├── autorag.generation     (SpanType.CHAT_MODEL; OGX autolog may add nested LLM spans)
    │   └── autorag.evaluation
    ├── Trace: autorag.pattern_A.query_1
    │   └── …
    └── … (N traces = benchmark rows)
```

**Invariant:** For pattern `P`, `count(traces under child_run(P)) == count(benchmark eval requests for P)`. Do **not** attach pattern eval traces to the parent run only, and do **not** use a single trace per pattern for the whole eval set.

### Spans and span types

Each **benchmark row** gets one **trace** (root span name e.g. `autorag.{pattern_name}.query_{idx}`) with attributes `pattern_name`, `query_index`. Under that trace, log **three stage spans** in ai4rag:

| Span name | `SpanType` | Inputs / outputs (summary) |
|-----------|------------|----------------------------|
| `autorag.retrieval` | `RETRIEVER` | Query in; retrieved documents out (`page_content`, `metadata.doc_uri`, `metadata.chunk_id`) per [MLflow retriever schema](https://mlflow.org/docs/latest/genai/concepts/span#retriever-spans) |
| `autorag.generation` | `CHAT_MODEL` | Query + context in; answer out; `mlflow.chat.tokenUsage`; `ai.model.name` / `ai.model.provider` |
| `autorag.evaluation` | (default) | Ground truth, prediction, context in; per-metric scores out; `metric.{name}` attributes |

Enable **`mlflow.openai.autolog()`** at component start so OGX **`responses.create`** calls inside `autorag.generation` produce **nested spans** (tokens, latency) without duplicating the full request body on both parent and child spans. See [Tracing OGX Applications with MLflow](https://ogx-ai.github.io/blog/mlflow-observability).

### Implementation approach

**Prerequisite:** Each pattern’s benchmark eval runs while its **nested child run is active** (in **ai4rag** or the component).

1. Component start: `mlflow.set_tracking_uri(...)`, `mlflow.tracing.enable()`, `mlflow.openai.autolog()`.
2. Per pattern: nested child run → for each benchmark row, one trace root + three `autorag.*` spans → `log_params` / `log_metrics` on the child run.
3. Parent run: pipeline params/tags only.

```python
from mlflow.entities import SpanType

def evaluate_single_query(pattern, query, query_idx, trace_root):
    with mlflow.start_span(name="autorag.retrieval", parent=trace_root, span_type=SpanType.RETRIEVER) as ret_span:
        ret_span.set_inputs({"query": query["question"]})
        chunks = retrieve(pattern, query["question"])
        ret_span.set_outputs([{"page_content": c["text"], "metadata": {"chunk_id": c["id"]}, "id": c["id"]} for c in chunks])

    with mlflow.start_span(name="autorag.generation", parent=trace_root, span_type=SpanType.CHAT_MODEL) as gen_span:
        response = client.responses.create(...)  # autolog adds nested spans when enabled
        gen_span.set_outputs({"answer": response["text"]})
        gen_span.set_attribute("mlflow.chat.tokenUsage", {...})

    with mlflow.start_span(name="autorag.evaluation", parent=trace_root) as eval_span:
        scores = compute_metrics(query, response, chunks)
        eval_span.set_outputs(scores)
        for metric, value in scores.items():
            eval_span.set_attribute(f"metric.{metric}", value)
```

If **`optimizer.optimize()`** evaluates all patterns in one call, add an **ai4rag** hook so each pattern opens its child run and each benchmark row creates the trace + span tree above.

**Dependencies:** `mlflow>=2.22` (Responses API autolog); `openai` in the component or ai4rag image.

### Verification

```python
import mlflow

client = mlflow.MlflowClient()
child_run_id = "..."  # pattern child run
traces = client.search_traces(
    filter_string=f"mlflow.run_id = '{child_run_id}'",
)
print(len(traces))  # expect == number of benchmark rows
```

In the MLflow UI: parent run → child run → **Traces** (N rows) → expand one trace to see **`autorag.retrieval`**, **`autorag.generation`**, **`autorag.evaluation`**.

Further reading: [OpenAI tracing](https://mlflow.org/docs/latest/genai/tracing/integrations/listing/openai/), [MLflow span types](https://mlflow.org/docs/latest/genai/concepts/span#span-types), [OGX MLflow tracing](https://ogx-ai.github.io/blog/mlflow-observability).

---

## Related

- OGX / Llama Stack MLflow tracing: https://ogx-ai.github.io/blog/mlflow-observability
- AutoRAG component overview: [../README.md](../README.md)
- AutoRAG ADR: [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- Shared platform MLflow details: [AutoML MLflow integration](../../automl/features/mlflow_integration.md)
- Data Science Pipelines architecture: [../../pipelines/README.md](../../pipelines/README.md)
