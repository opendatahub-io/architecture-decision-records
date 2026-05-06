# Model insights (artifacts and metrics)
## Table of contents

- [OpenShift AI 3.4](#openshift-ai-34)
  - [Tabular pipeline](#tabular-pipeline)
    - [Per refitted model (`{model_name}_FULL`)](#per-refitted-model-model_name_full)
  - [Timeseries pipeline](#timeseries-pipeline)
    - [Per refitted model (`{model_name}_FULL`)](#per-refitted-model-model_name_full-1)
  - [Leaderboards and URIs](#leaderboards-and-uris)
- [OpenShift AI 3.5 and later](#openshift-ai-35-and-later)
  - [Tabular (planned)](#tabular-planned)
    - [Example: `roc_curve.json` (binary classification)](#example-roc_curvejson-binary-classification)
    - [Example: `roc_curve.json` (multiclass, one-vs-rest)](#example-roc_curvejson-multiclass-one-vs-rest)
    - [Example: `precision_recall_curve.json` (binary classification)](#example-precision_recall_curvejson-binary-classification)
    - [Example: `precision_recall_curve.json` (multiclass, one-vs-rest)](#example-precision_recall_curvejson-multiclass-one-vs-rest)
  - [Timeseries (planned)](#timeseries-planned)
    - [Example: `back_testing.json` (time series)](#example-back_testingjson-time-series)
    - [Visualization Use Cases](#visualization-use-cases)

---
## OpenShift AI 3.4
### Tabular pipeline

One combined **`Model`** artifact from **`autogluon_models_training`** (`models_artifact.path`); then **`HTML`** from **`autogluon_leaderboard_evaluation`**.

#### Per refitted model (`{model_name}_FULL`)

| Path | Content |
|------|---------|
| `metrics/metrics.json` | Test-split **`evaluate_predictions`**; non-finite values stripped for KFP serialization. |
| `metrics/feature_importance.json` | **`feature_importance`** (test split, `subsample_size=2000`) as dict. |
| `metrics/confusion_matrix.json` | **Classification only** (`binary` / `multiclass`); omitted for **regression**. |
| `notebooks/automl_predictor_notebook.ipynb` | Embedded regression/classification template with run, pipeline, model, and sample-row placeholders. |
| `predictor/` | **`clone_for_deployment`** export for that model. |
| `model.json` | **`name`**, **`location`**, **`metrics.test_data`** (same scores as `metrics/metrics.json`). |

**`metadata`:** `model_names` (JSON **string** list, KFP workaround) and **`context`** (`task_type`, `label_column`, `model_config`, `data_config`, `models` mirroring each `model.json`). Returned **`eval_metric`** wires the leaderboard sort column.

---

### Timeseries pipeline

**`autogluon_timeseries_models_full_refit`** emits **one `Model` artifact per refit** (often **`ParallelFor`**, parallelism **1**). Selection **`leaderboard`** is not written to disk; persisted insight material is **refit** outputs plus leaderboard **HTML** below.

#### Per refitted model (`{model_name}_FULL`)

| Path | Content |
|------|---------|
| `metrics/metrics.json` | **`evaluate`** on held-out test data with **all** `AVAILABLE_METRICS` keys; non-finite stripped. |
| `predictor/` | Saved **`TimeSeriesPredictor`**. |
| `predictor/predictor_metadata.json` | Model id, **`prediction_length`**, **`eval_metric`**, **`target`**, **`id_column`**, **`timestamp_column`**. |
| `notebooks/automl_predictor_notebook.ipynb` | **`timeseries_notebook.ipynb`** template with run / pipeline / model / sample / column placeholders. |
| `model.json` | **`name`**, **`base_model`**, **`location`**, **`metrics.test_data`**. |

**No** `feature_importance.json` or `confusion_matrix.json` for time-series.

---

### Leaderboards and URIs

Both leaderboard components read **`…/{model}_FULL/metrics/metrics.json`**, sort by pipeline **`eval_metric`** (AutoGluon **higher-is-better** display), render HTML via shared [`leaderboard_html_template.html`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/shared), and set **`metadata.display_name`** to **`automl_leaderboard`** (tabular) or **`automl_timeseries_leaderboard`** (timeseries), plus **`metadata.data`** (rounded table).

HTML rows link **`{artifact_uri}/{model}/predictor`** and **`…/notebooks/automl_predictor_notebook.ipynb`**. For timeseries **`dsl.Collected`** inputs, metrics are found by **`glob("*/metrics/metrics.json")`**; missing files are skipped with a warning.

---

## OpenShift AI 3.5 and later

The following **artifact files are planned** for richer model-quality insights. They are **not** produced on **`main`** of [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) today. Confirm **filenames**, **directories**, and **JSON schema** in **`autogluon_models_training`** / **`autogluon_timeseries_models_full_refit`** (and any notebook or UI consumers) for the OpenShift AI build you document.

**Implementation Alignment:**

These JSON schemas are designed to match the **exact output** of AutoGluon and scikit-learn methods:

- **ROC/PR curves (Tabular)**: Direct serialization of `sklearn.metrics.roc_curve()` and `sklearn.metrics.precision_recall_curve()` outputs, which AutoGluon uses internally. Probabilities obtained via `predictor.predict_proba(X_test)`.
  
- **Back-testing (Time-series)**: Aggregation of AutoGluon's `predictor.backtest_predictions()`, `predictor.backtest_targets()`, and `predictor.evaluate(cutoff=...)` outputs across multiple validation windows.

**Reference Examples:**
- AutoGluon Tabular discussion on sklearn metrics: [GitHub Discussion #2720](https://github.com/autogluon/autogluon/discussions/2720)
- AutoGluon Time-series backtesting tutorial: [Forecasting In-Depth](https://auto.gluon.ai/dev/tutorials/timeseries/forecasting-indepth.html)
- Backtesting API: [backtest_predictions()](https://auto.gluon.ai/dev/api/autogluon.timeseries.TimeSeriesPredictor.backtest_predictions.html)

### Tabular (planned)

Under each **`{model_name}_FULL/`** directory next to existing `metrics/` files:

| Path | Content (planned) |
|------|---------------------|
| `metrics/roc_curve.json` | **ROC curve** data (points / thresholds) for plotting or downstream visualization; intended for **binary** and **multiclass** classification runs where probabilistic scores support an ROC construction (same applicability envelope as today’s **`confusion_matrix.json`**). |
| `metrics/precision_recall_curve.json` | **Precision–recall curve** data for the same classification context. |

**Regression** runs would **not** emit these files (no class probability curve in the same sense).

#### Example: `roc_curve.json` (binary classification)

```json
{
  "task_type": "binary",
  "positive_class": 1,
  "auc": 0.9821,
  "fpr": [0.0, 0.0, 0.0123, 0.0456, 0.0891, 0.1234, 0.2345, 0.4567, 0.7890, 1.0],
  "tpr": [0.0, 0.2345, 0.5678, 0.7890, 0.8901, 0.9345, 0.9678, 0.9890, 0.9987, 1.0],
  "thresholds": ["inf", 0.9876, 0.8765, 0.7654, 0.6543, 0.5432, 0.4321, 0.3210, 0.2109, 0.1098],
  "num_samples": 2000,
  "num_positive": 1024,
  "num_negative": 976
}
```

**Schema notes (aligned with `sklearn.metrics.roc_curve`):**
- **`fpr`** (false positive rate): Increasing array from 0 to 1, **directly from** `sklearn.metrics.roc_curve(y_true, y_pred_proba[:, 1])[0]`
- **`tpr`** (true positive rate): Increasing array from 0 to 1, **directly from** `sklearn.metrics.roc_curve()[1]`
- **`thresholds`**: Decreasing decision thresholds starting with `"inf"` (scikit-learn 1.3+: represents always-predict-negative classifier), **directly from** `sklearn.metrics.roc_curve()[2]`
- Arrays must have equal length: `len(fpr) == len(tpr) == len(thresholds)`
- **`auc`**: Area under the ROC curve, computed via `sklearn.metrics.roc_auc_score(y_true, y_pred_proba[:, 1])`
- **AutoGluon note**: AutoGluon uses `sklearn.metrics` directly; obtain `y_pred_proba` via `predictor.predict_proba(X_test)`

**Visualization:**

```
ROC Curve - WeightedEnsemble_L3 (Binary Classification)
AUC = 0.9821

TPR │
1.0 ┤                    ●●●●
    │                 ●●●
    │              ●●●
0.8 ┤           ●●●
    │        ●●●
    │      ●●
0.6 ┤    ●●
    │   ●
    │  ●
0.4 ┤ ●
    │●
    │
0.2 ┤●
    │
    ●────────────────────────────────
0.0 └─────────────────────────────── FPR
    0.0   0.2   0.4   0.6   0.8   1.0

● Model (AUC=0.98)
─ Random Classifier (AUC=0.50)

Best Operating Point: TPR=0.89, FPR=0.09 (threshold=0.65)
```

#### Example: `roc_curve.json` (multiclass, one-vs-rest)

```json
{
  "task_type": "multiclass",
  "strategy": "ovr",
  "num_classes": 3,
  "classes": [0, 1, 2],
  "auc_macro": 0.9456,
  "auc_weighted": 0.9512,
  "per_class": {
    "0": {
      "auc": 0.9821,
      "fpr": [0.0, 0.0234, 0.0567, 0.1234, 0.3456, 0.6789, 1.0],
      "tpr": [0.0, 0.4567, 0.7890, 0.9012, 0.9678, 0.9945, 1.0],
      "thresholds": ["inf", 0.8901, 0.7654, 0.6321, 0.4567, 0.2345, 0.1012]
    },
    "1": {
      "auc": 0.9234,
      "fpr": [0.0, 0.0345, 0.0789, 0.1567, 0.3890, 0.7123, 1.0],
      "tpr": [0.0, 0.5123, 0.8012, 0.9123, 0.9701, 0.9923, 1.0],
      "thresholds": ["inf", 0.9012, 0.7890, 0.6543, 0.4789, 0.2567, 0.1234]
    },
    "2": {
      "auc": 0.9312,
      "fpr": [0.0, 0.0212, 0.0678, 0.1456, 0.3678, 0.6912, 1.0],
      "tpr": [0.0, 0.4890, 0.7923, 0.9089, 0.9656, 0.9934, 1.0],
      "thresholds": ["inf", 0.8765, 0.7543, 0.6234, 0.4512, 0.2389, 0.1056]
    }
  },
  "num_samples": 3000
}
```

**Schema notes (multiclass):**
- **`strategy`**: `"ovr"` (one-vs-rest) or `"ovo"` (one-vs-one)
- **`per_class`**: Dict mapping class labels (as strings) to individual ROC curves
- **`auc_macro`**: Unweighted mean of per-class AUC scores
- **`auc_weighted`**: Weighted mean by support (class frequency)

**Visualization:**

```
ROC Curves - CatBoost_BAG_L2 (Multiclass, One-vs-Rest)
Macro AUC = 0.9456 | Weighted AUC = 0.9512

TPR │
1.0 ┤              ●●●●     ▲▲▲▲     ■■■■
    │           ●●●      ▲▲▲      ■■■
    │        ●●●      ▲▲▲      ■■■
0.8 ┤      ●●●     ▲▲▲     ■■■
    │    ●●●    ▲▲▲    ■■■
    │   ●●    ▲▲    ■■
0.6 ┤  ●●   ▲▲   ■■
    │ ●●  ▲▲  ■■
    │ ●  ▲  ■
0.4 ┤●  ▲ ■
    │● ▲■
    │●▲■
0.2 ┤●▲■
    │●▲■
    ●▲■─────────────────────────────
0.0 └─────────────────────────────── FPR
    0.0   0.2   0.4   0.6   0.8   1.0

● Class 0 (AUC=0.98, support=1034)
▲ Class 1 (AUC=0.92, support=987)
■ Class 2 (AUC=0.93, support=979)
─ Random Classifier (AUC=0.50)
```

#### Example: `precision_recall_curve.json` (binary classification)

```json
{
  "task_type": "binary",
  "positive_class": 1,
  "average_precision": 0.9567,
  "precision": [0.5120, 0.6789, 0.7890, 0.8456, 0.9012, 0.9345, 0.9678, 0.9890, 1.0],
  "recall": [1.0, 0.9876, 0.9456, 0.8901, 0.8234, 0.7456, 0.6012, 0.4567, 0.0],
  "thresholds": [0.1234, 0.2345, 0.3456, 0.4567, 0.5678, 0.6789, 0.7890, 0.8901],
  "num_samples": 2000,
  "num_positive": 1024,
  "num_negative": 976,
  "baseline_precision": 0.512
}
```

**Schema notes (aligned with `sklearn.metrics.precision_recall_curve`):**
- **`precision`**: Starts at class balance (baseline), ends at 1.0; **directly from** `sklearn.metrics.precision_recall_curve(y_true, y_pred_proba[:, 1])[0]`
- **`recall`**: Starts at 1.0 (all positives captured), ends at 0.0; **directly from** `sklearn.metrics.precision_recall_curve()[1]`
- **`thresholds`**: Increasing array; note `len(thresholds) == len(precision) - 1` per scikit-learn convention (last precision/recall pair has no threshold); **directly from** `sklearn.metrics.precision_recall_curve()[2]`
- **`average_precision`**: Area under the precision-recall curve (AP score), computed via `sklearn.metrics.average_precision_score(y_true, y_pred_proba[:, 1])`
- **`baseline_precision`**: No-skill baseline (class balance: `num_positive / total_samples`)
- **AutoGluon note**: Use `predictor.predict_proba(X_test)` to get probability scores for curve computation

**Visualization:**

```
Precision-Recall Curve - WeightedEnsemble_L3 (Binary Classification)
Average Precision = 0.9567

Precision │
     1.0 ┤                           ●
         │                         ●●
         │                       ●●
     0.8 ┤                     ●●
         │                   ●●
         │                 ●●
     0.6 ┤               ●●
         │             ●●
         │           ●●
     0.4 ┤         ●●
         │       ●●
         │     ●●
     0.2 ┤   ●●
         │ ●●
         ●●──────────────────────────
     0.0 └──────────────────────────── Recall
         1.0   0.8   0.6   0.4   0.2  0.0

● Model (AP=0.96)
─ No-Skill Baseline (precision=0.51)

Note: Recall decreases from left (1.0) to right (0.0)
High precision at low recall: threshold is very strict
```

#### Example: `precision_recall_curve.json` (multiclass, one-vs-rest)

```json
{
  "task_type": "multiclass",
  "strategy": "ovr",
  "num_classes": 3,
  "classes": [0, 1, 2],
  "average_precision_macro": 0.9234,
  "average_precision_weighted": 0.9312,
  "per_class": {
    "0": {
      "average_precision": 0.9567,
      "precision": [0.3456, 0.5678, 0.7890, 0.8901, 0.9456, 0.9789, 1.0],
      "recall": [1.0, 0.9876, 0.9234, 0.8567, 0.7456, 0.5678, 0.0],
      "thresholds": [0.1012, 0.2345, 0.4123, 0.5678, 0.6901, 0.8234],
      "support": 1034
    },
    "1": {
      "average_precision": 0.9123,
      "precision": [0.3234, 0.5456, 0.7678, 0.8789, 0.9312, 0.9701, 1.0],
      "recall": [1.0, 0.9890, 0.9345, 0.8678, 0.7567, 0.5890, 0.0],
      "thresholds": [0.0987, 0.2123, 0.3987, 0.5456, 0.6789, 0.8123],
      "support": 987
    },
    "2": {
      "average_precision": 0.9012,
      "precision": [0.3123, 0.5234, 0.7456, 0.8678, 0.9234, 0.9656, 1.0],
      "recall": [1.0, 0.9912, 0.9401, 0.8789, 0.7678, 0.6012, 0.0],
      "thresholds": [0.0876, 0.2012, 0.3876, 0.5234, 0.6678, 0.7987],
      "support": 979
    }
  },
  "num_samples": 3000
}
```

**Schema notes (multiclass):**
- **`average_precision_macro`**: Unweighted mean of per-class AP scores
- **`average_precision_weighted`**: Weighted mean by class support
- **`support`**: Number of samples per class (for weighted averaging)

**Visualization:**

```
Precision-Recall Curves - CatBoost_BAG_L2 (Multiclass, One-vs-Rest)
Macro AP = 0.9234 | Weighted AP = 0.9312

Precision │
     1.0 ┤                    ●●●  ▲▲▲  ■■■
         │                  ●●   ▲▲   ■■
         │                ●●   ▲▲   ■■
     0.8 ┤              ●●   ▲▲   ■■
         │            ●●   ▲▲   ■■
         │          ●●   ▲▲   ■■
     0.6 ┤        ●●   ▲▲   ■■
         │      ●●   ▲▲   ■■
         │    ●●   ▲▲   ■■
     0.4 ┤  ●●   ▲▲   ■■
         │ ●●  ▲▲  ■■
         │●●  ▲▲  ■■
     0.2 ┤●  ▲▲  ■■
         │● ▲▲ ■■
         ●●▲▲■■────────────────────
     0.0 └──────────────────────────── Recall
         1.0   0.8   0.6   0.4   0.2  0.0

● Class 0 (AP=0.96, baseline=0.35, support=1034)
▲ Class 1 (AP=0.91, baseline=0.32, support=987)
■ Class 2 (AP=0.90, baseline=0.31, support=979)

Per-class baselines shown as horizontal lines at respective precision values
```

### Timeseries (planned)

Under each **`{model_name}_FULL/metrics/`**:

| Path                         | Content (planned)                                                                                                                                                                                        |
|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `metrics/back_testing.json`  | **Back-testing** summary (for example per-window or per-series holdout results, scores, and identifiers) aligned with AutoGluon time-series validation / backtest behavior once wired in the refit step. |
| `metrics/forecast_data.json` | **Back-testing** data points grouped by window and id (best 3 and worst 3) aligned with AutoGluon time-series validation / backtest behavior once wired in the refit step.                               |

Leaderboard steps may later be extended to **surface links or summaries** from these files; until then they remain **companion files** alongside **`metrics/metrics.json`**.

#### Example: `back_testing.json` (time series)

```json
{
  "model_name": "DeepAR_FULL",
  "prediction_length": 7,
  "num_val_windows": 3,
  "eval_metric": "WQL",
  "target": "sales",
  "id_column": "product_id",
  "timestamp_column": "date",
  "overall_metrics": {
    "WQL": 0.2341,
    "MAPE": 12.34,
    "MASE": 0.8765,
    "RMSE": 45.67,
    "MAE": 34.21
  },
  "per_window_metrics": [
    {
      "window_id": 0,
      "train_end": "2025-12-07",
      "test_start": "2025-12-08",
      "test_end": "2025-12-14",
      "num_series": 1000,
      "metrics": {
        "WQL": 0.2198,
        "MAPE": 11.87,
        "MASE": 0.8432,
        "RMSE": 43.21,
        "MAE": 32.56
      }
    },
    {
      "window_id": 1,
      "train_end": "2025-12-14",
      "test_start": "2025-12-15",
      "test_end": "2025-12-21",
      "num_series": 1000,
      "metrics": {
        "WQL": 0.2367,
        "MAPE": 12.45,
        "MASE": 0.8876,
        "RMSE": 46.78,
        "MAE": 35.12
      }
    },
    {
      "window_id": 2,
      "train_end": "2025-12-21",
      "test_start": "2025-12-22",
      "test_end": "2025-12-28",
      "num_series": 1000,
      "metrics": {
        "WQL": 0.2458,
        "MAPE": 12.98,
        "MASE": 0.8987,
        "RMSE": 47.02,
        "MAE": 35.95
      }
    }
  ],
  "per_series_summary": {
    "num_series_evaluated": 1000,
    "series_with_degraded_performance": 47,
    "best_series": [
      {
        "item_id": "product_123",
        "avg_WQL": 0.1234,
        "avg_MAPE": 5.67
      },
      {
        "item_id": "product_456",
        "avg_WQL": 0.1345,
        "avg_MAPE": 6.12
      },
      {
        "item_id": "product_789",
        "avg_WQL": 0.1456,
        "avg_MAPE": 6.78
      }
    ],
    "worst_series": [
      {
        "item_id": "product_234",
        "avg_WQL": 0.8765,
        "avg_MAPE": 45.67
      },
      {
        "item_id": "product_567",
        "avg_WQL": 0.8234,
        "avg_MAPE": 42.34
      },
      {
        "item_id": "product_890",
        "avg_WQL": 0.7890,
        "avg_MAPE": 39.12
      }
    ]
  },
  "quantile_coverage": {
    "0.1": 0.0987,
    "0.5": 0.4876,
    "0.9": 0.8912
  },
  "metadata": {
    "num_items": 1000,
    "avg_series_length": 365,
    "total_predictions": 21000,
    "computation_time_seconds": 127.8
  }
}
```

**Schema notes (aligned with AutoGluon TimeSeriesPredictor API):**
- **`overall_metrics`**: Aggregated metrics across all validation windows (mean of per-window scores)
- **`per_window_metrics`**: Array of metrics for each backtesting window; **obtained by iterating** `evaluate()` with different `cutoff` values or using `ExpandingWindowSplitter`
  - Example: `for cutoff in range(-num_val_windows * prediction_length, 0, prediction_length): score = predictor.evaluate(test_data, cutoff=cutoff)`
  - Returns dict with keys from `AVAILABLE_METRICS`: `{'WQL': 0.234, 'MAPE': 12.34, 'MASE': 0.876, 'RMSE': 45.67, 'MAE': 34.21}`
- **`per_series_summary`**: Optional summary; **derived from** `predictor.backtest_predictions()` and custom per-series metric computation (not built-in AutoGluon)
- **`quantile_coverage`**: Actual empirical coverage of predicted quantiles; **derived from** comparing `backtest_predictions()` quantile columns (`0.1`, `0.5`, `0.9`) against `backtest_targets()`
- **AutoGluon methods used**:
  - `predictor.backtest_predictions(test_data, num_val_windows=3)` → list of TimeSeriesDataFrame (predictions per window)
  - `predictor.backtest_targets(test_data, num_val_windows=3)` → list of TimeSeriesDataFrame (targets per window)
  - `predictor.evaluate(test_data, cutoff=...)` → dict of metrics for specific window

**Visualization 1: Per-Window Metric Trends**

```
Back-testing Performance - DeepAR_FULL (3 validation windows)
Overall WQL = 0.2341 | Overall MAPE = 12.34%

WQL  │
0.30 ┤
     │                                    ●
0.25 ┤                        ●          
     │                                    
0.20 ┤            ●                       
     │                                    
0.15 ┤                                    
     │                                    
0.10 ┤                                    
     │                                    
0.05 ┤                                    
     │                                    
0.00 └─────────────────────────────────────
     Window 0      Window 1      Window 2
     12/07-12/14   12/14-12/21   12/21-12/28

MAPE │
(%)  │
15.0 ┤                                    ●
     │                        ●          
12.5 ┤                                    
     │            ●                       
10.0 ┤                                    
     │                                    
 7.5 ┤                                    
     │                                    
 5.0 ┤                                    
     │                                    
 2.5 ┤                                    
     │                                    
 0.0 └─────────────────────────────────────
     Window 0      Window 1      Window 2

⚠️  Performance degradation detected: WQL increased from 0.22 to 0.25 (+11.8%)
    MAPE increased from 11.87% to 12.98% (+9.3%)
```

**Visualization 2: Per-Series Performance Summary**

```
Top 3 Best Performing Series (by avg WQL):
┌──────────────┬──────────┬──────────┬────────────────┐
│ Item ID      │ Avg WQL  │ Avg MAPE │ Category       │
├──────────────┼──────────┼──────────┼────────────────┤
│ product_123  │  0.1234  │   5.67%  │ ⭐ Excellent   │
│ product_456  │  0.1345  │   6.12%  │ ⭐ Excellent   │
│ product_789  │  0.1456  │   6.78%  │ ⭐ Excellent   │
└──────────────┴──────────┴──────────┴────────────────┘

Top 3 Worst Performing Series (by avg WQL):
┌──────────────┬──────────┬──────────┬────────────────┐
│ Item ID      │ Avg WQL  │ Avg MAPE │ Category       │
├──────────────┼──────────┼──────────┼────────────────┤
│ product_234  │  0.8765  │  45.67%  │ ⚠️  Poor       │
│ product_567  │  0.8234  │  42.34%  │ ⚠️  Poor       │
│ product_890  │  0.7890  │  39.12%  │ ⚠️  Poor       │
└──────────────┴──────────┴──────────┴────────────────┘

📊 47 out of 1000 series (4.7%) show degraded performance
```

**Visualization 3: Quantile Coverage Analysis**

```
Quantile Coverage (Expected vs Actual)

Coverage │
   100% ┤                     ─────●─────
        │               ─────●
        │         ─────●
    75% ┤   ─────●
        │ ●●
        │
    50% ┤
        │
        │
    25% ┤
        │
        │
     0% └─────────────────────────────────
        0.1    0.2    0.3    0.4    0.5    0.6    0.7    0.8    0.9
                           Quantile Level

● Actual Coverage
─ Expected Coverage (perfect calibration)

Calibration Quality:
✅ 0.1 quantile: 9.87% actual (expected: 10%) - well calibrated
✅ 0.5 quantile: 48.76% actual (expected: 50%) - well calibrated  
⚠️  0.9 quantile: 89.12% actual (expected: 90%) - slight underestimation

Overall: Model is well-calibrated (avg deviation: 1.2%)
```

#### Example: `forecast_data.json` (time series)

```json
{
  "metadata": {
    "prediction_length": 24,
    "num_backtest_windows": 3,
    "total_series": 147,
    "metric_used_for_ranking": "MAPE"
  },
  "best_performer": {
    "item_id": "series_042",
    "avg_metrics": {
      "MAPE": 2.45,
      "RMSE": 13.1,
      "MAE": 8.7,
      "SMAPE": 2.2,
      "MASE": 0.89
    },
    "windows": [
      {
        "window_id": 0,
        "metrics": {
          "MAPE": 2.58,
          "RMSE": 13.8,
          "MAE": 9.1,
          "SMAPE": 2.3,
          "MASE": 0.92
        },
        "backtest_window": {
          "train_end": "2026-02-14T23:00:00Z",
          "test_start": "2026-02-15T00:00:00Z",
          "test_end": "2026-02-15T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-02-15T00:00:00Z",
            "actual": 515.2,
            "predicted": 512.8,
            "lower_bound": 496.5,
            "upper_bound": 529.1,
            "quantile_0.1": 499.3,
            "quantile_0.5": 512.8,
            "quantile_0.9": 526.3
          },
          {
            "timestamp": "2026-02-15T01:00:00Z",
            "actual": 510.5,
            "predicted": 509.2,
            "lower_bound": 493.0,
            "upper_bound": 525.4,
            "quantile_0.1": 495.7,
            "quantile_0.5": 509.2,
            "quantile_0.9": 522.7
          },
          {
            "timestamp": "2026-02-15T02:00:00Z",
            "actual": 507.8,
            "predicted": 506.5,
            "lower_bound": 490.4,
            "upper_bound": 522.6,
            "quantile_0.1": 493.1,
            "quantile_0.5": 506.5,
            "quantile_0.9": 519.9
          }
        ]
      },
      {
        "window_id": 1,
        "metrics": {
          "MAPE": 2.41,
          "RMSE": 12.9,
          "MAE": 8.5,
          "SMAPE": 2.2,
          "MASE": 0.88
        },
        "backtest_window": {
          "train_end": "2026-03-07T23:00:00Z",
          "test_start": "2026-03-08T00:00:00Z",
          "test_end": "2026-03-08T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-03-08T00:00:00Z",
            "actual": 520.8,
            "predicted": 519.3,
            "lower_bound": 502.9,
            "upper_bound": 535.7,
            "quantile_0.1": 505.7,
            "quantile_0.5": 519.3,
            "quantile_0.9": 532.9
          },
          {
            "timestamp": "2026-03-08T01:00:00Z",
            "actual": 516.4,
            "predicted": 517.1,
            "lower_bound": 500.8,
            "upper_bound": 533.4,
            "quantile_0.1": 503.6,
            "quantile_0.5": 517.1,
            "quantile_0.9": 530.6
          },
          {
            "timestamp": "2026-03-08T02:00:00Z",
            "actual": 513.2,
            "predicted": 514.8,
            "lower_bound": 498.6,
            "upper_bound": 531.0,
            "quantile_0.1": 501.4,
            "quantile_0.5": 514.8,
            "quantile_0.9": 528.2
          }
        ]
      },
      {
        "window_id": 2,
        "metrics": {
          "MAPE": 2.34,
          "RMSE": 12.5,
          "MAE": 8.2,
          "SMAPE": 2.1,
          "MASE": 0.85
        },
        "backtest_window": {
          "train_end": "2026-03-31T23:00:00Z",
          "test_start": "2026-04-01T00:00:00Z",
          "test_end": "2026-04-01T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-04-01T00:00:00Z",
            "actual": 523.4,
            "predicted": 521.8,
            "lower_bound": 505.2,
            "upper_bound": 538.4,
            "quantile_0.1": 508.3,
            "quantile_0.5": 521.8,
            "quantile_0.9": 535.3
          },
          {
            "timestamp": "2026-04-01T01:00:00Z",
            "actual": 518.2,
            "predicted": 519.5,
            "lower_bound": 502.9,
            "upper_bound": 536.1,
            "quantile_0.1": 505.8,
            "quantile_0.5": 519.5,
            "quantile_0.9": 533.2
          },
          {
            "timestamp": "2026-04-01T02:00:00Z",
            "actual": 512.8,
            "predicted": 514.2,
            "lower_bound": 497.8,
            "upper_bound": 530.6,
            "quantile_0.1": 500.5,
            "quantile_0.5": 514.2,
            "quantile_0.9": 527.9
          }
        ]
      }
    ]
  },
  "worst_performer": {
    "item_id": "series_104",
    "avg_metrics": {
      "MAPE": 42.58,
      "RMSE": 253.4,
      "MAE": 158.7,
      "SMAPE": 37.2,
      "MASE": 4.68
    },
    "windows": [
      {
        "window_id": 0,
        "metrics": {
          "MAPE": 41.23,
          "RMSE": 245.8,
          "MAE": 152.4,
          "SMAPE": 35.7,
          "MASE": 4.52
        },
        "backtest_window": {
          "train_end": "2026-02-14T23:00:00Z",
          "test_start": "2026-02-15T00:00:00Z",
          "test_end": "2026-02-15T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-02-15T00:00:00Z",
            "actual": 89.4,
            "predicted": 198.7,
            "lower_bound": 168.3,
            "upper_bound": 229.1,
            "quantile_0.1": 172.5,
            "quantile_0.5": 198.7,
            "quantile_0.9": 224.9
          },
          {
            "timestamp": "2026-02-15T01:00:00Z",
            "actual": 92.1,
            "predicted": 203.5,
            "lower_bound": 172.8,
            "upper_bound": 234.2,
            "quantile_0.1": 177.1,
            "quantile_0.5": 203.5,
            "quantile_0.9": 229.9
          },
          {
            "timestamp": "2026-02-15T02:00:00Z",
            "actual": 87.6,
            "predicted": 207.2,
            "lower_bound": 176.0,
            "upper_bound": 238.4,
            "quantile_0.1": 180.4,
            "quantile_0.5": 207.2,
            "quantile_0.9": 234.0
          }
        ]
      },
      {
        "window_id": 1,
        "metrics": {
          "MAPE": 44.85,
          "RMSE": 265.3,
          "MAE": 168.2,
          "SMAPE": 39.4,
          "MASE": 4.92
        },
        "backtest_window": {
          "train_end": "2026-03-07T23:00:00Z",
          "test_start": "2026-03-08T00:00:00Z",
          "test_end": "2026-03-08T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-03-08T00:00:00Z",
            "actual": 82.7,
            "predicted": 205.3,
            "lower_bound": 174.2,
            "upper_bound": 236.4,
            "quantile_0.1": 178.6,
            "quantile_0.5": 205.3,
            "quantile_0.9": 232.0
          },
          {
            "timestamp": "2026-03-08T01:00:00Z",
            "actual": 78.5,
            "predicted": 210.8,
            "lower_bound": 179.3,
            "upper_bound": 242.3,
            "quantile_0.1": 183.8,
            "quantile_0.5": 210.8,
            "quantile_0.9": 237.8
          },
          {
            "timestamp": "2026-03-08T02:00:00Z",
            "actual": 75.2,
            "predicted": 214.5,
            "lower_bound": 182.7,
            "upper_bound": 246.3,
            "quantile_0.1": 187.2,
            "quantile_0.5": 214.5,
            "quantile_0.9": 241.8
          }
        ]
      },
      {
        "window_id": 2,
        "metrics": {
          "MAPE": 41.65,
          "RMSE": 249.1,
          "MAE": 155.6,
          "SMAPE": 36.5,
          "MASE": 4.61
        },
        "backtest_window": {
          "train_end": "2026-03-31T23:00:00Z",
          "test_start": "2026-04-01T00:00:00Z",
          "test_end": "2026-04-01T23:00:00Z"
        },
        "forecast_data": [
          {
            "timestamp": "2026-04-01T00:00:00Z",
            "actual": 95.8,
            "predicted": 211.2,
            "lower_bound": 179.5,
            "upper_bound": 242.9,
            "quantile_0.1": 184.0,
            "quantile_0.5": 211.2,
            "quantile_0.9": 238.4
          },
          {
            "timestamp": "2026-04-01T01:00:00Z",
            "actual": 91.3,
            "predicted": 216.5,
            "lower_bound": 184.4,
            "upper_bound": 248.6,
            "quantile_0.1": 188.9,
            "quantile_0.5": 216.5,
            "quantile_0.9": 244.1
          },
          {
            "timestamp": "2026-04-01T02:00:00Z",
            "actual": 88.7,
            "predicted": 220.3,
            "lower_bound": 187.9,
            "upper_bound": 252.7,
            "quantile_0.1": 192.4,
            "quantile_0.5": 220.3,
            "quantile_0.9": 248.2
          }
        ]
      }
    ]
  }
}
```

**Schema notes (aligned with AutoGluon TimeSeriesPredictor API):**
- **`best_performer`** and **`worst_performer`**: Single series ranked by average MAPE across all windows
- **`avg_metrics`**: Average performance metrics across all backtest windows for the series
- **`windows`**: Array containing all backtest windows (typically 3) for the series
- **`forecast_data`**: Array of timestamp-level predictions with actual values, point forecasts, confidence bounds, and quantiles
  - **`actual`**: Ground truth value from test data
  - **`predicted`**: Point forecast (typically median/quantile 0.5)
  - **`lower_bound`** / **`upper_bound`**: Prediction interval (e.g., 90% confidence interval)
  - **`quantile_0.1`**, **`quantile_0.5`**, **`quantile_0.9`**: Probabilistic forecasts at different quantile levels
- **AutoGluon methods used**:
  - `predictor.backtest_predictions(test_data, num_val_windows=3)` → predictions per window
  - `predictor.backtest_targets(test_data, num_val_windows=3)` → actual values per window
  - Per-series metrics computed by filtering predictions/targets by `item_id`

**Visualization: Best vs Worst Performer Across Backtest Windows**

Similar to AutoGluon's `predictor.plot()` output, showing observed time series with predictions from all backtest windows overlaid.

```
═══════════════════════════════════════════════════════════════════════════════
Best Performer: series_042 (Avg MAPE = 2.45%)
Backtest predictions across 3 validation windows
═══════════════════════════════════════════════════════════════════════════════

Value │
  540 ┤              ●●●●●●●●●●●●●●●  │  ●●●●●●●●●●●●●●●  │  ●●●●●●●●●●●●●●●
      │           ●●●              ●●│●●              ●●│●●              ●●
  520 ┤         ●●                  ●│●              ─────────────────────────
      │       ●●                  ──┼┼──          ─── │ ───        ───  │ ───
  500 ┤     ●●                 ───  │ │ ───   ───     │    ───  ───     │
      │   ●●                ───     │ │    ───        │       ───        │
  480 ┤ ●●               ───        │ │               │                  │
      │●              ───           │ │               │                  │
  460 ┤           ───               │ │               │                  │
      └───────────────────────────────────────────────────────────────────────
      Feb 10    Feb 14  │  Feb 15   Mar 2   Mar 7  │  Mar 8   Mar 27  Mar 31 │ Apr 1
                     Window 0 ↑             Window 1 ↑              Window 2 ↑
                     Test Period             Test Period              Test Period
                     (Feb 15-Feb 15)         (Mar 8-Mar 8)            (Apr 1-Apr 1)

● Observed values
─ Predictions (median/quantile 0.5)
│ Backtest window cutoffs (dashed gray lines in AutoGluon plots)

Window Metrics:
  Window 0: MAPE=2.58%, RMSE=13.8    ✓ Accurate tracking
  Window 1: MAPE=2.41%, RMSE=12.9    ✓ Improved performance
  Window 2: MAPE=2.34%, RMSE=12.5    ✓ Best performance (consistent improvement)


═══════════════════════════════════════════════════════════════════════════════
Worst Performer: series_104 (Avg MAPE = 42.58%)
Backtest predictions across 3 validation windows
═══════════════════════════════════════════════════════════════════════════════

Value │
  250 ┤                           ───────────────────────────────────────────
      │                        ───│              ─── │              ───  │
  200 ┤                     ───   │           ───    │           ───     │
      │                  ───      │        ───       │        ───        │
  150 ┤               ───         │     ───          │     ───           │
      │            ───            │  ───             │  ───              │
  100 ┤         ───               │──                │──                 │
      │   ●●●●●●●●●●●●●●●●●●●●●   │   ●●●●●●●●●●●●●  │  ●●●●●●●●●●●●●●●
   50 ┤ ●●                    ●●● │ ●●            ●● │ ●●            ●●●
      └───────────────────────────────────────────────────────────────────────
      Feb 10    Feb 14  │  Feb 15   Mar 2   Mar 7  │  Mar 8   Mar 27  Mar 31 │ Apr 1
                     Window 0 ↑             Window 1 ↑              Window 2 ↑
                     Test Period             Test Period              Test Period
                     (Feb 15-Feb 15)         (Mar 8-Mar 8)            (Apr 1-Apr 1)

● Observed values (consistently LOW)
─ Predictions (consistently HIGH - severe systematic bias)
│ Backtest window cutoffs

Window Metrics:
  Window 0: MAPE=41.23%, RMSE=245.8  ⚠️  Severe overprediction
  Window 1: MAPE=44.85%, RMSE=265.3  ⚠️  Degrading (predictions diverge further)
  Window 2: MAPE=41.65%, RMSE=249.1  ⚠️  Persistent bias across all windows

Issues Detected Across All Windows:
  ⚠️  Structural break: Observed values dropped ~60% but model didn't adapt
  ⚠️  Outliers in test periods: Actual values consistently outside prediction bounds
  ⚠️  Seasonality mismatch: Model pattern doesn't match actual time series behavior
  ⚠️  Extreme variance: High prediction uncertainty (wide confidence intervals)

Root Cause Analysis Guidance:
  → This series likely needs custom preprocessing (outlier handling, detrending)
  → Consider separate model training for this series or similar low-value cohorts
  → Inspect historical data for data quality issues (missing values, measurement errors)
```

**Use Cases:**
- **Visual inspection across time**: Linear timeline shows how predictions track actual values across all backtest windows, similar to `predictor.plot()` in AutoGluon
- **Performance consistency**: Best performer (series_042) shows consistent improvement (MAPE decreases from 2.58% → 2.41% → 2.34%), while worst performer (series_104) shows persistent bias across all windows
- **Systematic bias detection**: Timeline reveals whether prediction errors are random or systematic (series_104 consistently overpredicts by ~60%)
- **Temporal degradation**: Visualize if model quality changes over time (series_104 degrades in window 1, suggesting concept drift)
- **Series-specific troubleshooting**: Worst performers highlight specific time series requiring custom preprocessing, different models, or data quality investigation
- **Implementation**: Aligns with AutoGluon's `predictor.plot(test_data, predictions, max_history_length=...)` visualization pattern

#### Visualization Use Cases

See mocked visualizations above for each artifact type. These examples demonstrate:

**ROC Curve (`roc_curve.json`):**
- **Binary**: Single curve comparing model (AUC=0.98) against random baseline (AUC=0.50)
- **Multiclass**: Overlaid one-vs-rest curves showing per-class performance with support counts
- **Key insight**: Identify best operating point (threshold) balancing TPR/FPR for deployment
- **Implementation**: matplotlib `plt.plot(fpr, tpr)`, plotly `go.Scatter(x=fpr, y=tpr)`

**Precision-Recall Curve (`precision_recall_curve.json`):**
- **Binary**: Shows model maintains high precision (>0.9) even at moderate recall (>0.6)
- **Multiclass**: Per-class curves with baselines; Class 0 performs best (AP=0.96)
- **Key insight**: Critical for imbalanced datasets; precision drop-off indicates threshold sensitivity
- **Implementation**: matplotlib `plt.plot(recall, precision)`, add `plt.hlines()` for baseline

**Back-testing (`back_testing.json`):**
- **Trend plot**: Detects performance degradation (WQL increased 11.8% from window 0 to window 2)
- **Per-series summary**: Identifies 47 problematic series (4.7%) requiring investigation
- **Quantile coverage**: Validates probabilistic forecast calibration (well-calibrated at 1.2% avg deviation)
- **Key insight**: Time-varying performance and series-specific issues guide model refinement
- **Implementation**: pandas `df.plot()` for trends, seaborn `heatmap()` for series matrix, bar charts for quantiles

**Forecast Data (`forecast_data.json`):**
- **Timeline visualization**: Linear plot showing observed values and predictions across all backtest windows (similar to `predictor.plot()` in AutoGluon)
- **Window-by-window comparison**: Visual inspection of forecast quality across multiple time periods with cutoff points marked
- **Systematic bias detection**: Timeline reveals consistent overprediction/underprediction patterns (e.g., series_104 overpredicts by ~60% across all windows)
- **Performance trends**: Identify whether model improves, degrades, or remains stable over successive windows
- **Key insight**: Provides detailed timestamp-level view for best and worst performers, enabling deep-dive analysis of specific failure modes
- **Implementation**: `predictor.plot(test_data, predictions, max_history_length=300)` or custom matplotlib/plotly timeline with shaded regions for test periods

**Integration with Jupyter Notebooks:**

The planned `automl_predictor_notebook.ipynb` template (per model) can automatically load these JSON artifacts and render interactive visualizations.

---

**Implementation References:**
- [AutoGluon Forecasting In-Depth Tutorial](https://auto.gluon.ai/dev/tutorials/timeseries/forecasting-indepth.html) (multi-window backtesting examples)
- [AutoGluon Forecasting Notebook (Colab)](https://colab.research.google.com/github/autogluon/autogluon/blob/master/docs/tutorials/timeseries/forecasting-indepth.ipynb)
- [TabularPredictor.evaluate_predictions API](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.evaluate_predictions.html)
- [TimeSeriesPredictor.plot API](https://auto.gluon.ai/dev/api/autogluon.timeseries.TimeSeriesPredictor.plot.html) (built-in visualization method)

---

Verify paths on the **pipelines-components** revision you ship; **current** layouts are on **`main`**; **3.5** rows above are **intent** until implemented in code.
