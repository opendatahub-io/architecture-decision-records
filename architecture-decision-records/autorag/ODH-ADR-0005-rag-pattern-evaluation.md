# Open Data Hub - AutoRAG Pattern Evaluation

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | AutoRAG Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1425](https://redhat.atlassian.net/browse/RHAISTRAT-1425) · [RHAISTRAT-2623](https://redhat.atlassian.net/browse/RHAISTRAT-2623) · [RHOAIENG-88692](https://redhat.atlassian.net/browse/RHOAIENG-88692) |
| Other docs:    | [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md) |

## What

This ADR defines how AutoRAG evaluates RAG patterns during `rag_templates_optimization`: evaluator selection, the GAM objective, and evaluation artifacts.

## Why

Standardized, unambiguous metrics are required for GAM ranking, reliable artifacts, and Dashboard leaderboards. Unitxt and Ragas both expose `faithfulness`, so metric identity must include its evaluator.

## Goals

* Define preset-valid, evaluator-qualified `optimization_metric` values.
* Define aggregate and row-level evaluation artifacts.
* Preserve retrieved-document identity as `document_key`.

## Non-Goals

* Search-space dimensions and pipeline settings other than `optimization_metric` (see [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).

## How

### Overview

During **`rag_templates_optimization`**, AutoRAG scores each RAG pattern on a benchmark. It uses evaluator-qualified metrics so identically named metrics—such as Unitxt and Ragas `faithfulness`—remain distinct in artifacts and leaderboards. This ADR covers evaluation only; other pipeline settings are in [ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md).

* **`speed`** runs Unitxt and `custom:overall_score`; **`balanced`** additionally runs Ragas. `overall_score` aggregates only metrics produced by the active evaluators.
* The user selects **`optimization_metric`** (default `custom:overall_score`) with an evaluator-qualified ID such as `unitxt:faithfulness` or `ragas:context_precision`.
* Per-row detail is in **`evaluation_results.json`**; aggregates are in **`pattern.json`** → `evaluation.metrics[]` ([schema](./ODH-ADR-0004-rag-pattern-inference.md#patternjson)). GAM uses the aggregate marked `optimization_metric: true`.

### Metric catalog

The active preset determines which catalog entries are computed. Users select the GAM objective with an evaluator-qualified ID ([optimization_metric](#optimization_metric)).

| Evaluator | Metric IDs | Availability |
|-----------|------------|--------------|
| `unitxt` | `faithfulness`, `answer_correctness`, `context_correctness` | `speed`, `balanced` |
| `ragas` | `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall` | `balanced` |
| `custom` | `overall_score`, the equal-weight mean of the other active metrics | `speed`, `balanced` |

Unitxt `faithfulness` and Ragas `faithfulness` share a name; artifact identity is always `name` + `evaluator`.

### optimization_metric

Pipeline and experiment input is an **`evaluator:metric`** ID, such as `unitxt:faithfulness` ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).

Rules:

* `speed` accepts Unitxt and custom metrics; `balanced` accepts Unitxt, Ragas, and custom metrics. IDs unavailable in the selected preset are rejected.
* A metric ID resolves exactly to its named evaluator; unqualified IDs are rejected.
* The resolved pair is the only `metrics[]` entry with `optimization_metric: true`. Artifacts record `evaluator` on every score row so Unitxt and Ragas `faithfulness` stay distinct.

The default is `custom:overall_score`, which aggregates Unitxt metrics in `speed`, and Unitxt plus Ragas metrics in `balanced`.

### evaluation_results.json

Each pattern subdirectory contains **`evaluation_results.json`**: one row per benchmark question. Run-level aggregates live in `pattern.json` → `evaluation.metrics[]`; `speed` rows omit Ragas metrics.

| Field | Description |
|-------|-------------|
| `question` | Benchmark question text |
| `correct_answers` | Ground-truth answers from the benchmark JSON |
| `answer` | Generated answer for this pattern |
| `answer_contexts[]` | Retrieved chunks: `text`, `document_key` (full object key / path, not a filename) |
| `metrics[]` | Per-metric evaluator results for this row: `name`, `evaluator`, and a score when available. `name` + `evaluator` match `evaluation.metrics[]` in `pattern.json` |

Ground-truth document keys remain in the benchmark input ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md#benchmark-json)). Pattern and leaderboard source labels use the same full object path, so equal basenames remain distinct. KFP `evaluation_results.json` is the source of truth for row-level scores and retrieved context.

## Related

- [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) — full `pattern.json` schema and artifact layout
- [RAG templates](./ODH-ADR-0003-rag-templates.md) — Agentic Vector and Agentic Graph RAG templates
- [AutoRAG optimization settings](./ODH-ADR-0002-experiment-settings.md) — `optimization_metric`; corpus list; benchmark JSON (`correct_answer_document_keys`)
- [ODH-ADR-0001-autorag](./ODH-ADR-0001-autorag.md)
- [pipelines-components PR #257](https://github.com/opendatahub-io/pipelines-components/pull/257) — preset-based evaluator selection and qualified metrics
- [pipelines-components PR #204](https://github.com/opendatahub-io/pipelines-components/pull/204) — Ragas wiring on `rag_templates_optimization`
