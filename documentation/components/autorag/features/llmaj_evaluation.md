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

**LLMaJ** adds an explicit **judge model** that scores `(question, contexts, answer[, ground_truth])` using judge rubrics (bundled **default** profile in ai4rag for the standard trio). Prefer a judge distinct from the **generation** model; when only one model is deployed, **`judge_model_id` may equal `generation.model_id`**.

---

## Design overview

### Evaluator naming

Pipeline parameter **`evaluator`**: **`judge`** \| **`ragas`**. Recorded in `pattern.json` → `evaluation.evaluator`.

| `evaluator` | ai4rag class | Notes |
|-------------|--------------|-------|
| **`judge`** | `LLMaJEvaluator` | **Default** — explicit judge model + rubrics |
| `ragas` | `UnitxtEvaluator` | Legacy bundled RAGAS-style metric suite |

Pipeline parameters on **`documents_rag_optimization_pipeline`**: 

| Parameter | Default | Role |
|-----------|---------|------|
| `evaluator` | **`judge`** | `judge` \| `ragas` |
| `judge_model_id` | *(auto)* | Optional override; when omitted, ai4rag [auto-selects](#auto-selecting-the-judge-model) at `search_space_preparation` |
| `optimization_metric` | **`faithfulness`** | GAM objective (`objective_metric` in ai4rag) |

Default path uses the judge evaluator; set `evaluator=ragas` to keep today’s `UnitxtEvaluator` behavior. Judge cost scales with benchmark rows × patterns.

---

## Judge model

One judge per optimization run (pipeline-level); patterns vary generation/retrieval only.

- **Validation:** `search_space_preparation` confirms the judge model is registered and reachable.
- **Explicit override:** set pipeline `judge_model_id` to skip auto-selection.

### Auto-selecting the judge model

When **`judge_model_id`** is omitted and `evaluator=judge`, **ai4rag** picks the judge once in **`search_space_preparation`** from the **validated generation-model pool** (same instruct models already admitted to the search space). The user does not choose from a list.

| Available models | Selection |
|------------------|-----------|
| **1** | Use that model (may equal every pattern’s `generation.model_id`) |
| **2+** | Run [calibration](#calibration-metric) below; record result in `evaluation.judge_model_id` for the whole run |

**Goals for auto-selection:** (1) judge scores should track a trustworthy reference on a small calibration slice; (2) prefer a model **different** from generation candidates when possible (independence); (3) fixed cost at prep time, not per GAM trial.

#### Calibration metric

On a **fixed calibration subset** of the benchmark (`min(20, 10% of rows)`, stratified if possible):

1. Use answers already produced for a **reference RAG configuration** (first GAM trial or a lightweight default pattern from search-space prep).
2. Score each row with **`ragas` path `faithfulness`** (`UnitxtEvaluator` reference).
3. For each **candidate judge model**, score the same rows with **`judge` `faithfulness`** (single metric, one judge call per row per candidate).
4. Compute **`judge_calibration_score`** = Pearson correlation between ragas-path and judge faithfulness on the subset.
5. **Select** the candidate with the **highest** `judge_calibration_score`.

**Tie-breakers (in order):** prefer a model **not** listed as a generation candidate in the search space; else prefer the model ranked **higher** in the existing search-space prep validation; else lowest `model_id` lexicographically.

**Why `faithfulness` for selection:** it is the default `optimization_metric`, detects hallucination (the main judge failure mode), and does not require extra ground-truth parsing beyond what the benchmark already provides.


Calibration metadata (`judge_selection`, `judge_calibration_score`, candidate rankings) is **logged only**.

`pattern.json` → **`evaluation`** records only `evaluator` and `judge_model_id` (same judge for every pattern in the run). See [example `pattern.json`](#example-patternjson). Judge rubrics and temperature use **ai4rag defaults** (not pipeline parameters).

---

## Metrics and artifacts

### What changes

| Area | Change |
|------|--------|
| Pipeline | `evaluator` (default **`judge`**); optional `judge_model_id` (else [auto-select](#auto-selecting-the-judge-model)) |
| `pattern.json` | New **`evaluation`** block; new derived **`scores.overall_score`** |
| Evaluator | `LLMaJEvaluator` replaces `UnitxtEvaluator` on the default path |
| Everything else | Same metric names, `scores` / `final_score` shape, `evaluation_results.json`, GAM flow |

### Standard metrics

| Metric | Question answered |
|--------|-------------------|
| `faithfulness` | Is the answer supported by retrieved context? (default `optimization_metric`) |
| `answer_correctness` | Does the answer match ground truth? |
| `context_correctness` | Was retrieval sufficient? |
| `overall_score` | Equal-weight blend of the trio ([LightRAG `ragas_score`](https://github.com/HKUDS/LightRAG/blob/main/lightrag/evaluation/eval_rag_quality.py) pattern) |

Per-question scores are **0–1** floats; aggregates use **`mean`, `ci_low`, `ci_high`** ([ai4rag shape](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)). All standard metrics are computed every run; only **`optimization_metric`** selects the GAM objective.

**`overall_score`** is derived: unweighted mean of the standard trio (NaNs skipped), at pattern level and per row in `evaluation_results.json`.

### Example `pattern.json`

Adds **`evaluation`** and **`scores.overall_score`** to the existing schema ([full shape](./rag_pattern_inference.md#example-patternjson)).

```json
{
  "name": "pattern_01",
  "iteration": 0,
  "max_combinations": 24,
  "duration_seconds": 120.5,
  "settings": { "generation": { "model_id": "granite-3.3-8b-instruct" } },
  "evaluation": {
    "evaluator": "judge",
    "judge_model_id": "granite-3.3-8b-instruct"
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

---

## Related

- [RAG pattern inference](./rag_pattern_inference.md) — artifact layout
- [MLflow integration](./mlflow_integration.md) — tracing and runs
- [Chunking and retrieval methods](./chunking_and_retrievals_methods.md)
- [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- [LightRAG RAGAS evaluation](https://github.com/HKUDS/LightRAG/blob/main/lightrag/evaluation/README_EVALUASTION_RAGAS.md)
- [ai4rag evaluation guide](https://ibm.github.io/ai4rag/latest/user-guide/evaluation/)
