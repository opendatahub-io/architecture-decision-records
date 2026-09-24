# AutoRAG MLflow Integration

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG |
| Status         | Approved |
| Authors        | [Lukasz Cmielowski](@LukaszCmielowski) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [AutoML MLflow Integration](../automl/ODH-ADR-0004-mlflow-integration.md) |

## What

Integrate MLflow experiment tracking, pattern metrics, and per-benchmark tracing into the AutoRAG Documents RAG optimization pipeline.

## Why

AutoRAG needs a unified way to compare RAG patterns and trace individual benchmark requests through retrieval, generation, and evaluation. MLflow provides this through the same KFP integration model as AutoML.

## Goals

* Use KFP-injected `KFP_MLFLOW_CONFIG` for opt-in tracking with parent/child MLflow runs.
* Log one child run per RAG pattern, its settings, aggregate metrics, and KFP artifact pointers.
* Create one trace per benchmark request with retrieval, generation, and evaluation spans.

## Non-Goals

* Copying `evaluation_results.json` content into MLflow (KFP remains the source of truth)
* OTel Collector integration, `mlflow.genai.evaluate()` on traces, `log_expectation` / `log_feedback`, `mlflow.log_input()` on parent run
* MLflow Model Registry integration

## How

### KFP MLflow integration mode

When MLflow is enabled on the pipeline server, KFP injects `KFP_MLFLOW_CONFIG` into every step pod. AutoRAG uses this JSON blob for opt-in tracking; absent, empty, invalid, or endpoint-less configuration disables tracking without failing optimization.

| Environment variable | Purpose |
|----------------------|---------|
| `KFP_MLFLOW_CONFIG` | Platform MLflow configuration JSON; the only injected MLflow configuration variable consumed by AutoRAG. |

The JSON schema is a platform KFP-to-MLflow contract and is not defined here. With valid configuration, components resume or create the parent run, create nested pattern runs, and explicitly call `mlflow.log_params()`, `mlflow.log_metrics()`, `mlflow.set_tags()`, and `mlflow.log_artifact()`.

### KFP artifacts produced by the pipeline

MLflow logs pointers—not copies—to the [per-pattern files](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts).

| Artifact / path role | Producing step | Layout and role |
|---------------------|----------------|-----------------|
| **Test data** | `test_data_loader` | Benchmark JSON on disk (input to search prep + optimization). |
| **Discovered documents** | `documents_discovery` | Descriptor of corpus objects for extraction. |
| **Extracted text** | `text_extraction` | Extracted document text (for example, via **docling**) for AutoRAG. |
| **`search_space_prep_report`** | `search_space_preparation` | YAML-serialized **search space** after phase-one validation. |
| **`rag_patterns`** (directory artifact) | `rag_templates_optimization` | One subdirectory per pattern. Contents: [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts). |

### MLflow mapping model

One parent run represents a KFP execution; each RAG pattern has one nested child run.

| MLflow concept | Proposed mapping |
|----------------|------------------|
| **Experiment** | KFP-managed experiment from `KFP_MLFLOW_CONFIG.experimentId`; otherwise `autorag_documents_rag_optimization` with an optional suffix. |
| **Parent run** | KFP-managed parent from `KFP_MLFLOW_CONFIG.parentRunId` (resume when set; create when unset). **Tags:** `kfp_run_id`, `kfp_run_name`, `pipeline_name`, dataset hashes or URIs (non-secret). **Params:** `preset`, `optimization_metric`, `optimization_max_rag_patterns`, `db_secret_name`, `image`, `kfp_version`, `autorag_version`. |
| **Child runs** | One nested child run per RAG pattern (folder name or `pattern.json` `name`). Enables side-by-side comparison of Unitxt / Ragas / custom metrics and chunking / retrieval / model choices. |
| **Traces** | Required when `KFP_MLFLOW_CONFIG` is valid. One trace per benchmark request, attached to the pattern child run. P patterns × N benchmark rows = P × N traces. |
| **Spans** | **Required** under each trace: `autorag.retrieval`, `autorag.generation`, `autorag.evaluation` with MLflow `SpanType` where applicable. Generation may include nested spans from `mlflow.openai.autolog()` for MaaS chat completions. |
| **Params (child)** | From `pattern.json` `settings`: `chunking.*`, `embedding.model_id`, `retrieval.*`, `generation.model_id` / prompt fields, `store_binding` (`provider_type`, `collection_name`). |
| **Metrics (child)** | Aggregate scores from `pattern.json` `evaluation.metrics[]`. Keying: [Metrics logged](#metrics-logged-child-runs). |
| **Child Artifacts** | Pointers (URIs/paths) to the per-pattern files in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts). Not copied into MLflow. |

### Implementation approach

`rag_templates_optimization` parses `KFP_MLFLOW_CONFIG` (lazy-importing `mlflow` only when valid), configures the client, and resumes or creates the parent run. It enables tracing and OpenAI autologging, then opens one nested run per pattern to log its parameters, aggregates, artifact pointers, and one trace per benchmark row from `pattern.json`. No separate work in `leaderboard_evaluation` or tracking artifact is needed.

### Metrics logged (child runs)

AutoRAG computes Unitxt and Ragas metrics during optimization. Log **aggregates only** on the pattern child run. Unitxt and Ragas both emit `faithfulness`; MLflow keys must include **evaluator** so the two series do not collide.

| Source field | MLflow |
|--------------|--------|
| Objective metric `scores.mean` | `log_metric("final_score", ...)` from the `metrics[]` entry with `optimization_metric: true` (resolved `optimization_metric` + evaluator) |
| `metrics[].name` + `metrics[].evaluator` | `log_metric("{name}.{evaluator}", scores.mean)` for each entry in `evaluation.metrics` (e.g. `faithfulness.unitxt`, `faithfulness.ragas`, `overall_score.custom`) |
| `duration_seconds` | `log_metric("duration_seconds", ...)` |

Per-question rows stay in KFP `evaluation_results.json`.

### Tracing per pattern child run

With valid `KFP_MLFLOW_CONFIG`, traces and stage spans are required and scoped to the pattern child run.

```
Parent run (pipeline)
└── Child run: pattern_A
    ├── Trace: autorag.pattern_A.query_0
    │   ├── autorag.retrieval      (SpanType.RETRIEVER)
    │   ├── autorag.generation     (SpanType.CHAT_MODEL)
    │   └── autorag.evaluation
    ├── Trace: autorag.pattern_A.query_1
    │   └── …
    └── … (N traces = benchmark rows)
```

For each pattern, the trace count equals its benchmark evaluation-request count.

**Spans and span types:**

| Span name | `SpanType` | Inputs / outputs (summary) |
|-----------|------------|----------------------------|
| `autorag.retrieval` | `RETRIEVER` | Query in; retrieved documents out (`page_content`, `metadata.doc_uri`, `metadata.chunk_id`) per MLflow retriever schema |
| `autorag.generation` | `CHAT_MODEL` | Query + context in; answer out; `mlflow.chat.tokenUsage`; `ai.model.name` / `ai.model.provider` |
| `autorag.evaluation` | (default) | Ground truth, prediction, context in; per-metric scores out; `metric.{name}.{evaluator}` attributes |

Enable `mlflow.openai.autolog()` at component start so MaaS OpenAI-compatible calls produce nested generation spans without duplicating request bodies. This requires `mlflow>=2.22` and `openai` in the component or AutoRAG image. Verify in the MLflow UI: parent run → child run → traces → retrieval, generation, and evaluation spans.

## Alternatives

### 1. Attach traces to the parent run instead of pattern child runs

**Discarded:** It loses per-pattern grouping and makes P × N traces difficult to filter and compare.

### 2. Use a single trace per pattern (all benchmark rows in one trace)

**Discarded:** It violates the one-trace-per-request convention and produces unwieldy traces.

### 3. Copy evaluation_results.json content into MLflow

**Discarded:** KFP artifacts are the row-level source of truth; copying them adds storage and synchronization costs.

## Risks

* **Trace volume:** P patterns × N rows can create many traces (for example, 5,000 for 50 × 100). *Mitigation:* monitor server capacity; consider future sampling.
* **`mlflow>=2.22`:** OpenAI autolog may be unavailable in some RHOAI deployments. *Mitigation:* lazy-import and version-check; fall back to manual generation spans.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| AutoRAG team                  |                  |            | Yes |
| Data Science Pipelines        |                  |            | Yes |
| MLflow integration (RHOAI)    | Matt Prahl, Humair Khan |  | Yes |

## References

* AutoML MLflow Integration ADR: [ODH-ADR-0004-mlflow-integration](../automl/ODH-ADR-0004-mlflow-integration.md)
* AutoRAG ADR: [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
* AutoRAG optimization settings: [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md)
* RAG pattern artifacts: [ODH-ADR-0004-rag-pattern-inference](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts)
* RAG pattern evaluation: [ODH-ADR-0005-rag-pattern-evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md)
* MLflow Tracking: [MLflow Tracking documentation](https://mlflow.org/docs/latest/tracking.html)
* MLflow span types: [MLflow span types documentation](https://mlflow.org/docs/latest/genai/concepts/span#span-types)
* MLflow retriever schema: [MLflow retriever spans](https://mlflow.org/docs/latest/genai/concepts/span#retriever-spans)
* OpenAI tracing: [MLflow OpenAI tracing](https://mlflow.org/docs/latest/genai/tracing/integrations/listing/openai/)
* Upstream pipeline: [pipelines-components — documents_rag_optimization_pipeline](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Nelesh Singla                 | 27 May     |       |
