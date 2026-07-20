# RAG pattern evaluation

## Table of contents

- [Overview](#overview)
- [Judge model](#judge-model)
- [Artifacts](#artifacts)
- [evaluation_results.json](#evaluation_resultsjson)
- [Related](#related)

---

## Overview

During **`rag_templates_optimization`**, ai4rag scores each RAG pattern on a benchmark. Aggregates land in **`pattern.json`** → `evaluation`, per-row detail in **`evaluation_results.json`**, and GAM ranks patterns by **`optimization_metric`**.

AutoRAG routes metrics to internal backends; callers do not configure an **`evaluator`**.

| Metric | Backend (`evaluator`) | Question answered |
|--------|----------------------|-------------------|
| `faithfulness` | `unitxt` | Is the answer supported by retrieved context? |
| `answer_correctness` | `unitxt` | Does the answer match ground truth? |
| `context_correctness` | `unitxt` | Was retrieval sufficient? |
| `answer_relevance` | `judge` | Does the answer address the question? |
| `overall_score` | `custom` | Equal-weight mean of the four metrics |

Per-question values are **0–1** floats. At pattern level, each `evaluation.metrics[]` entry carries **`scores`** with **`mean`, `ci_low`, `ci_high`** (aggregated across benchmark rows). All metrics run every optimization; the pipeline **`optimization_metric`** parameter (default **`overall_score`**) selects the GAM objective. The matching `metrics[]` entry is marked with **`optimization_metric: true`** — use its **`scores.mean`** for pattern ranking.

`answer_relevance` cost scales with benchmark rows × patterns.

---

## Judge model

One judge per optimization run, used only for **`answer_relevance`**. ai4rag auto-selects it in **`search_space_preparation`** from the validated generation-model pool. The chosen **`model_id`** is recorded on the `answer_relevance` metric entry (`evaluator: judge`). Prefer a judge distinct from the **generation** model; when only one model is deployed, judge **`model_id` may equal `generation.model_id`**.

| Available models | Selection |
|------------------|-----------|
| **1** | Use that model |
| **2+** | [Calibration](#calibration) on `min(20, 10% of rows)` |

Judge rubrics and temperature use **ai4rag defaults**.

### Calibration

On a fixed calibration subset, using answers from a reference RAG configuration:

1. Score each row with **`answer_relevance`** for every candidate judge model.
2. Pick the candidate with the best **`judge_calibration_score`** (spread and stability on the subset).

**Tie-breakers:** non-generation candidate → higher search-space prep rank → lowest `model_id` lexicographically. Selection metadata is logged only.

---

## Artifacts

`pattern.json` → **`evaluation`** holds run-level aggregates in **`metrics[]`**. Field reference below; full `pattern.json` example in [RAG pattern inference](./rag_pattern_inference.md#example-patternjson).

| Field | Description |
|-------|-------------|
| `metrics[]` | One entry per metric: `evaluator`, `name`, `description`, `scores`; `model_id` on judge entries; `optimization_metric: true` on the GAM objective entry |

---

## evaluation_results.json

Each pattern subdirectory under **`rag_patterns/<pattern_name>/`** also contains **`evaluation_results.json`**: a JSON **array** with one object per benchmark row. Use it to inspect failures, compare retrieval quality across patterns, or audit judge scores — `pattern.json` → `evaluation.metrics[]` aggregates (`scores.mean`, `ci_low`, `ci_high`) are computed from `metrics[].score` across these rows.

| Field | Description |
|-------|-------------|
| `question` | Benchmark question text |
| `correct_answers` | Ground-truth answers from `benchmark_data.json` |
| `answer` | Generated answer for this pattern |
| `answer_contexts[]` | Retrieved chunks: `text`, `document_id` |
| `metrics[]` | Per-metric scores for this row: `name`, `evaluator`, `score` (**0–1** float); `name` matches `evaluation.metrics[].name` in `pattern.json` |

KFP **`evaluation_results.json`** is the source of truth for row-level scores and retrieved context.

```json
[
  {
    "question": "What warranty period applies to the XR-200 controller?",
    "correct_answers": ["The XR-200 controller has a 24-month warranty."],
    "answer": "The XR-200 controller is covered by a 24-month warranty from the date of purchase.",
    "answer_contexts": [
      {
        "text": "XR-200 Controller — Warranty: 24 months from purchase date.",
        "document_id": "xr-200-manual.pdf"
      },
      {
        "text": "Register your XR-200 within 30 days to activate warranty coverage.",
        "document_id": "warranty-policy.pdf"
      }
    ],
    "metrics": [
      { "name": "faithfulness", "evaluator": "unitxt", "score": 0.94 },
      { "name": "answer_correctness", "evaluator": "unitxt", "score": 0.88 },
      { "name": "context_correctness", "evaluator": "unitxt", "score": 0.82 },
      { "name": "answer_relevance", "evaluator": "judge", "score": 0.91 },
      { "name": "overall_score", "evaluator": "custom", "score": 0.89 }
    ]
  },
  {
    "question": "Which firmware versions support remote diagnostics?",
    "correct_answers": ["Firmware 3.2 and later supports remote diagnostics."],
    "answer": "Remote diagnostics require firmware 3.2 or newer.",
    "answer_contexts": [
      {
        "text": "Remote diagnostics are available starting in firmware release 3.2.",
        "document_id": "release-notes-3.2.pdf"
      }
    ],
    "metrics": [
      { "name": "faithfulness", "evaluator": "unitxt", "score": 0.97 },
      { "name": "answer_correctness", "evaluator": "unitxt", "score": 0.85 },
      { "name": "context_correctness", "evaluator": "unitxt", "score": 0.78 },
      { "name": "answer_relevance", "evaluator": "judge", "score": 0.90 },
      { "name": "overall_score", "evaluator": "custom", "score": 0.88 }
    ]
  }
]
```

---

## Related

- [RAG pattern inference](./rag_pattern_inference.md) — full `pattern.json` schema and artifact layout
- [AutoRAG optimization settings](./experiment_settings.md) — `optimization_metric` pipeline parameter
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [ai4rag evaluation guide](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)
