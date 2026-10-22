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

* Use platform-provided KFP MLflow context for opt-in tracking with parent/child MLflow runs.
* Log one child run per RAG pattern, its settings, aggregate metrics, and KFP artifact pointers.
* Create one trace per benchmark request with retrieval, generation, and evaluation spans.

## Non-Goals

* Copying row-level evaluation or diagnostic artifacts into MLflow (KFP remains the source of truth)
* OTel Collector integration, `mlflow.genai.evaluate()` on traces, `log_expectation` / `log_feedback`, `mlflow.log_input()` on parent run
* MLflow Model Registry integration

## How

### Tracking mode

When MLflow is enabled for the KFP pipeline server, AutoRAG uses the platform-provided tracking context. Absent or unusable context disables tracking without failing optimization.

MLflow records references to KFP and per-pattern artifacts; it does not duplicate their contents. The artifact layout and source of truth are defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts).

### MLflow mapping model

One parent run represents a KFP execution; each RAG pattern has one nested child run.

| MLflow concept | Proposed mapping |
|----------------|------------------|
| **Experiment** | The KFP-managed MLflow experiment for the AutoRAG execution. |
| **Parent run** | The KFP execution. It records nonsecret run identity, data references, and optimization context. |
| **Child runs** | One nested child run per RAG pattern (folder name or `pattern.json` `name`). Enables side-by-side comparison of Unitxt / Ragas / custom metrics and chunking / retrieval / model choices. |
| **Traces** | When tracing is enabled, request-level observability is associated with the pattern child run. The per-request source is `evaluation_results.json`; P patterns × N benchmark rows can yield P × N traces. |
| **Spans** | Implementations may record retrieval, generation, and evaluation observability within a trace. Span names, types, and instrumentation are implementation details. |
| **Params (child)** | Selected nonsecret pattern settings from `pattern.json`. |
| **Metrics (child)** | Aggregate evaluator-qualified scores from `pattern.json` `evaluation.metrics[]`. Keying: [Metrics logged](#metrics-logged-child-runs). |
| **Child Artifacts** | References to the per-pattern artifacts defined in [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md#pattern-artifacts). Contents are not copied into MLflow. |

### Implementation approach

AutoRAG resumes or creates the parent run, then opens one nested run per pattern to log its selected configuration, aggregates, and artifact references. When tracing is enabled, request-level data is sourced from `evaluation_results.json`; `pattern.json` remains the selected-configuration and aggregate record. Instrumentation mechanics remain implementation details.

### Metrics logged (child runs)

AutoRAG computes Unitxt and Ragas metrics during optimization. Log **aggregates only** on the pattern child run. Unitxt and Ragas both emit `faithfulness`; MLflow keys must include **evaluator** so the two series do not collide.

| Source field | MLflow |
|--------------|--------|
| Objective metric `scores.mean` | `final_score`, from the `metrics[]` entry marked `optimization_metric: true` |
| `metrics[].name` + `metrics[].evaluator` | `{name}.{evaluator}` for each aggregate metric (for example, `faithfulness.unitxt`, `faithfulness.ragas`, `overall_score.custom`) |
| `duration_seconds` | `duration_seconds` |

Per-question rows stay in KFP `evaluation_results.json`.

### Optional tracing per pattern child run

When enabled by the platform configuration, tracing associates request-level observability with the relevant pattern child run. `evaluation_results.json` remains the source for those request-level records; tracing complements rather than replaces the pipeline artifacts.

Span structure and instrumentation are implementation details. Implementations retain the parent-run / pattern-child-run relationship so operators can navigate from aggregate experiment results to request-level observability.

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
