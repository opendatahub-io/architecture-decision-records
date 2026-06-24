# LLM-as-a-Judge (LLMaJ) evaluation

## Table of contents

- [Context](#context)
- [Design overview](#design-overview)
- [Judge model](#judge-model)
- [Metrics and artifacts](#metrics-and-artifacts)
- [Related](#related)

---

## Context

During **`rag_templates_optimization`**, ai4rag scores each RAG pattern on a benchmark; aggregates land in **`pattern.json`** → `scores`, per-row detail in **`evaluation_results.json`**, and GAM ranks patterns by **`optimization_metric`**. Today this uses **`UnitxtEvaluator`**; this design makes **`judge`** the default evaluator going forward.

Unitxt works for regression and leaderboards but rubrics are opaque, fixed, and coupled to the generation path.

**LLMaJ** adds an explicit **judge model** (Llama Stack) that scores `(question, contexts, answer[, ground_truth])` using judge rubrics (bundled **default** profile in ai4rag for the standard trio). Prefer a judge distinct from the **generation** model; when only one model is deployed, **`judge_model_id` may equal `generation.model_id`**.

---

## Design overview

### Evaluator naming

| Value | ai4rag class | Notes |
|-------|--------------|-------|
| **`judge`** | `LLMaJEvaluator` | **Default** — explicit judge model + bundled rubrics on Llama Stack |
| `unitxt` | `UnitxtEvaluator` | Legacy path; opt in via `evaluator=unitxt` |

Recorded in artifacts as `pattern.json` → `evaluation.evaluator` (same strings as the pipeline parameter).

Pipeline parameters on **`documents_rag_optimization_pipeline`**: 

| Parameter | Default | Role |
|-----------|---------|------|
| `evaluator` | **`judge`** | `judge` \| `unitxt` |
| `judge_model_id` | pattern `generation.model_id` | Used when `evaluator=judge`; may equal generation model |
| `optimization_metric` | **`faithfulness`** | GAM objective (`objective_metric` in ai4rag) |

Default path uses the judge evaluator; set `evaluator=unitxt` to keep today’s `UnitxtEvaluator` behavior. Judge cost scales with benchmark rows × patterns.

---

## Judge model

One judge per optimization run (pipeline-level); patterns vary generation/retrieval only.

- **Single-model clusters:** set `judge_model_id` to the deployed model (often same as `generation.model_id`). Rubrics and judge calls remain separate from generation; independence is lower than with a dedicated judge.
- **Default:** if `judge_model_id` is omitted, use the pattern’s `generation.model_id` (default `evaluator=judge`).
- **Validation:** `search_space_preparation` confirms the judge is reachable on Llama Stack.

Recorded per pattern in `pattern.json` → `evaluation` (provenance for leaderboard / audit):

| Field | Meaning |
|-------|---------|
| `evaluator` | `judge` or `unitxt` — which ai4rag evaluator produced scores |
| `judge_model_id` | Llama Stack judge model (`judge` evaluator; omitted from `evaluation` when `evaluator=unitxt`) |
| `metrics` | Score keys written to `scores` (standard trio for initial delivery) |

```json
{
  "evaluation": {
    "evaluator": "judge",
    "judge_model_id": "granite-3.3-8b-instruct",
    "metrics": ["faithfulness", "answer_correctness", "context_correctness"]
  }
}
```

Judge rubrics and temperature use **ai4rag defaults** (not pipeline parameters).

---

## Metrics and artifacts

### Standard metrics

| Metric | Question answered |
|--------|-------------------|
| `faithfulness` | Is the answer supported by retrieved context? (default `optimization_metric`) |
| `answer_correctness` | Does the answer match ground truth? |
| `context_correctness` | Was retrieval sufficient? |
| `overall_score` | Equal-weight blend of the trio ([LightRAG `ragas_score`](https://github.com/HKUDS/LightRAG/blob/main/lightrag/evaluation/eval_rag_quality.py) pattern) |

Per-question scores are **0–1** floats; aggregates use **`mean`, `ci_low`, `ci_high`** ([ai4rag shape](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)).

**`final_score`** = `scores[optimization_metric].mean`. Metric keys are **stable** for `unitxt` and `judge` (no `faithfulness_judge` suffix).

**`overall_score`** is derived: unweighted mean of the standard trio (NaNs skipped), at pattern level and per row in `evaluation_results.json`.

### Example `pattern.json`

```json
{
  "name": "pattern_01",
  "evaluation": {
    "evaluator": "judge",
    "judge_model_id": "granite-3.3-8b-instruct",
    "metrics": ["faithfulness", "answer_correctness", "context_correctness"]
  },
  "scores": {
    "faithfulness": { "mean": 0.91, "ci_low": 0.88, "ci_high": 0.94 },
    "answer_correctness": { "mean": 0.82, "ci_low": 0.78, "ci_high": 0.86 },
    "context_correctness": { "mean": 0.80, "ci_low": 0.70, "ci_high": 0.90 },
    "overall_score": { "mean": 0.84, "ci_low": 0.79, "ci_high": 0.89 }
  },
  "final_score": 0.91
}
```

### `evaluation_results.json` (per row)

**Unchanged schema** for `unitxt` and `judge`. Initial delivery does not add per-row provenance fields; judge metadata lives in `pattern.json` → `evaluation` only.

| Field | Initial delivery |
|-------|------------------|
| `question`, `answer`, `contexts` | same as today |
| `scores` | `faithfulness`, `answer_correctness`, `context_correctness`, `overall_score` |

Only the **evaluator plugin** differs between `judge` and `unitxt`; the row JSON shape does not.

Leaderboard reads `pattern.json` as today; pattern detail shows `evaluation.evaluator` and judge model.

---

## Related

- [RAG pattern inference](./rag_pattern_inference.md) — artifact layout
- [MLflow integration](./mlflow_integration.md) — tracing and runs
- [Chunking and retrieval methods](./chunking_and_retrievals_methods.md)
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [LightRAG RAGAS evaluation](https://github.com/HKUDS/LightRAG/blob/main/lightrag/evaluation/README_EVALUASTION_RAGAS.md)
- [ai4rag evaluation guide](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)
