# AutoML MLflow integration

This document proposes an **MLflow** integration for OpenShift AI / ODH **AutoML** workflows implemented in [opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components) under **`components/training/automl`** and **`pipelines/training/automl`**, with product goals: **experiment tracking, comparison, reproducibility**.


## Table of Contents

- [KFP MLflow integration modes (OpenShift AI 3.5+)](#kfp-mlflow-integration-modes-openshift-ai-35)
  - [Environment variables (RHOAI 3.5+)](#environment-variables-rhoai-35)
- [MLflow mapping model](#mlflow-mapping-model)
  - [Optional alignment with AutoGluon-native logging](#optional-alignment-with-autogluon-native-logging)
- [Per-component logging details](#per-component-logging-details)
  - [Component-level logging matrix (tabular pipeline)](#component-level-logging-matrix-tabular-pipeline)
  - [Component-level logging matrix (timeseries pipeline)](#component-level-logging-matrix-timeseries-pipeline)
  - [Data loader metrics (valuable additions)](#data-loader-metrics-valuable-additions)
    - [Tabular data loader](#tabular-data-loader)
    - [Timeseries data loader](#timeseries-data-loader)
  - [Run lifecycle and hierarchy](#run-lifecycle-and-hierarchy)
  - [Comparison to MLflow framework integrations](#comparison-to-mlflow-framework-integrations)
  - [Concrete parameter sources (tabular)](#concrete-parameter-sources-tabular)
  - [Concrete metric sources (tabular)](#concrete-metric-sources-tabular)
  - [Artifacts and naming conventions](#artifacts-and-naming-conventions)
  - [Error handling and partial logging](#error-handling-and-partial-logging)
- [KFP Pipeline Integration: MLflow Tracking Artifact](#kfp-pipeline-integration-mlflow-tracking-artifact)
  - [Artifact Purpose](#artifact-purpose)
  - [Component Output Definition](#component-output-definition)
  - [Artifact Schema](#artifact-schema)
  - [Component Implementation](#component-implementation)
  - [Retrieval from AutoML Dashboard](#retrieval-from-automl-dashboard)
  - [KFP UI Display](#kfp-ui-display)
  - [Dashboard Integration Workflow](#dashboard-integration-workflow)
  - [Alternative: HTML Artifact Metadata (Fallback)](#alternative-html-artifact-metadata-fallback)
  - [Benefits of Dedicated Artifact vs Metadata-Only](#benefits-of-dedicated-artifact-vs-metadata-only)
- [References](#references)
  - [AutoML Implementation](#automl-implementation)
  - [MLflow on RHOAI](#mlflow-on-rhoai)
  - [MLflow Framework Integration](#mlflow-framework-integration)

---

## KFP MLflow integration modes (OpenShift AI 3.5+)

Starting with **OpenShift AI 3.5**, Data Science Pipelines provides two modes of MLflow integration:

1. **Automatic KFP-to-MLflow logging**: All KFP output metrics and input parameters are automatically recorded in MLflow without code changes. Works well for simple metrics.

2. **Opt-in with environment variables** (recommended for AutoML): MLflow client environment variables are pre-configured in pipeline step pods, allowing custom logging with full control over complex artifacts and metrics.

AutoML uses **mode 2** because AutoGluon produces complex artifacts (leaderboards, ensemble hierarchies, nested model metrics) that require explicit logging control beyond what KFP automatic logging can capture.

### Environment variables (RHOAI 3.5+)

When MLflow integration is enabled at the project level, the following environment variables are automatically injected into pipeline step pods:

| Environment Variable | Purpose | Example Value | Notes |
|---------------------|---------|---------------|-------|
| `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint | `https://mlflow-server.example.com` | If unset, MLflow logging is disabled (optional integration) |
| `MLFLOW_WORKSPACE` | Workspace identifier for the project | `data-science-project` | Corresponds to the OpenShift AI project/namespace |
| `MLFLOW_EXPERIMENT_ID` | Auto-created experiment ID for the pipeline | `"1"` | KFP creates one experiment per pipeline; components can log to this experiment directly |
| `MLFLOW_RUN_ID` | Parent run ID for the KFP pipeline execution | `"abc123..."` | KFP creates a parent run per pipeline execution; components create child runs under this |
| `MLFLOW_TRACKING_AUTH` | Authentication mechanism | `kubernetes-namespaced` | Uses Kubernetes client auth plugin (MLflow SDK 3.11+); reads token from pod's service account or local `~/.kube/config` |

**AutoML integration approach:** Components check if `MLFLOW_TRACKING_URI` is set. If yes, they create **child runs** under the KFP-managed parent run (`MLFLOW_RUN_ID`) for each refitted model, using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.log_artifact()` calls to capture AutoGluon-specific data.

---

## MLflow mapping model

Design for **two pipeline families** (tabular vs timeseries) with the **same conceptual mapping**.

| MLflow concept | Proposed mapping |
|----------------|------------------|
| **Experiment** | **RHOAI 3.5+:** KFP auto-creates one experiment per pipeline (accessible via `MLFLOW_EXPERIMENT_ID`). |
| **Parent run** | **RHOAI 3.5+:** KFP auto-creates one parent run per pipeline execution (accessible via `MLFLOW_RUN_ID`). AutoML components **resume this run** to add tags and params. **Tags:** `kfp_run_id`, `pipeline_name`, `task_type` (`tabular` \| `timeseries`), `namespace` (from `MLFLOW_WORKSPACE`), dataset **hashes or URIs** (non-secret). **Params:** `eval_metric`, `preset`, `top_n`, `pipeline_version`, `autogluon_version`. |
| **Child runs** | **One child run per leaderboard row / refitted model** (each name in `model_names` or equivalent for timeseries), created by AutoML components as nested runs under the KFP parent. Enables side-by-side comparison in the MLflow UI across models from the same pipeline run. |
| **Params** | **Parent:** `eval_metric`, `preset`, `top_n`, `task_type`, `pipeline_version`, `autogluon_version`. **Child:** `model_type`, `stack_level`, `fit_time`, `predict_time`, `num_bag_folds` / `num_stack_levels` when exposed. |
| **Metrics** | **Parent:** `best_model_score`, `total_training_time`, `num_models_trained`. **Child:** Primary validation score (`score_val`, `score_test`), task-specific metrics from AutoGluon leaderboard / `metrics.json` (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`, `mae`), **fit time**. |
| **Artifacts** | **Per child:** subset of **`metrics.json`**, optional **`feature_importance.json`**. **Per parent:** **HTML leaderboard** (size-capped). |

### Optional alignment with AutoGluon-native logging

AutoGluon can integrate with experiment trackers in some setups. Prefer **explicit MLflow calls in our components** first so KFP + OpenShift behavior stays predictable; revisit **native AutoGluon callbacks** only if they reduce duplication without fighting Kubeflow’s filesystem layout.

---

## Per-component logging details

This section provides concrete implementation guidance for MLflow integration in each AutoML pipeline component, following patterns established by MLflow’s scikit-learn and XGBoost integrations (autologging, nested runs, model signatures).

> **Important:** MLflow does **not** have native AutoGluon support. There is no `mlflow.autogluon` module or autologging capability. All tracking must be implemented via **explicit MLflow API calls** (`mlflow.log_params()`, `mlflow.log_metrics()`, `mlflow.log_artifact()`). Model logging would require a custom `mlflow.pyfunc` wrapper since AutoGluon predictors cannot be logged with standard MLflow flavors.

### Component-level logging matrix (tabular pipeline)

| Component | MLflow Operation | Parameters Logged | Metrics Logged | Artifacts Logged | Run Type |
|-----------|------------------|-------------------|----------------|------------------|----------|
| **tabular_data_loader** | Params + metrics | `sampling_method`, `task_type`, `label_column`, `test_size`, `random_state`, `stratify` | `n_features`, `train_rows`, `test_rows`, `selection_train_rows`, `total_rows_loaded`, `sampling_ratio`, `dataset_memory_mb` | — | Parent |
| **autogluon_models_training** | Tags + params | `preset`, `eval_metric`, `time_limit`, `num_bag_folds`, `num_stack_levels`, `top_n`, `task_type` | `selection_time_seconds` | — | Parent |
| **autogluon_leaderboard_evaluation** | **Parent run creation + child runs** | (per parent): `pipeline_version`, `autogluon_version`, `kfp_run_id`, `namespace` | (per parent): `best_model_score`, `total_training_time` | (per parent): `leaderboard.html` | Parent + N children |
| **autogluon_leaderboard_evaluation** (per model) | Child run per model | (per child): `model_type`, `num_models_in_stack`, `fit_time`, `predict_time`, `stack_level` | (per child): `score_val`, `score_test` (if available), task-specific metrics (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`), **RHOAI 3.5+:** `auc`, `average_precision`, `num_samples`, `num_positive`/`num_negative` (binary), `num_classes`, `auc_macro`, `auc_weighted` (multiclass) | (per child): `model_metrics.json`, `feature_importance.json`, `confusion_matrix.json` (classification), **RHOAI 3.5+:** `roc_curve.json`, `precision_recall_curve.json` (classification) | Child (nested) |

### Component-level logging matrix (timeseries pipeline)

| Component | MLflow Operation | Parameters Logged | Metrics Logged | Artifacts Logged | Run Type |
|-----------|------------------|-------------------|----------------|------------------|----------|
| **timeseries_data_loader** | Params + metrics | `id_column`, `timestamp_column`, `target`, `known_covariates_names`, `prediction_length`, `test_size`, `selection_train_size` | `n_unique_series`, `total_rows_loaded`, `sampled_rows`, `duplicates_dropped`, `avg_series_length`, `selection_train_rows`, `extra_train_rows`, `test_rows` | — | Parent |
| **autogluon_timeseries_models_selection** | Tags + params | `preset`, `eval_metric`, `prediction_length`, `target`, `known_covariates_names`, `top_n` | `selection_time_seconds` | — | Parent |
| **autogluon_timeseries_models_full_refit** | Metrics per model | — | `refit_time_seconds` (per model) | — | Parent |
| **autogluon_timeseries_leaderboard_evaluation** | **Parent run creation + child runs** | (per parent): `pipeline_version`, `autogluon_version`, `kfp_run_id`, `namespace` | (per parent): `best_model_score`, `total_training_time` | (per parent): `leaderboard.html` | Parent + N children |
| **autogluon_timeseries_leaderboard_evaluation** (per model) | Child run per model | (per child): `model_type`, `fit_time`, `predict_time` | (per child): `score_val`, task-specific metrics (e.g., `WQL`, `MAPE`, `MASE`, `RMSE`) | (per child): `model_metrics.json` | Child (nested) |

### Data loader metrics (valuable additions)

Data loading components currently compute valuable metrics that **should be logged to MLflow** for dataset profiling, reproducibility, and data drift detection:

#### Tabular data loader

| Metric/Parameter | Type | Source | Value Example | Purpose |
|------------------|------|--------|---------------|---------|
| `n_features` | Metric | `len(X.columns)` after loading | `47` | Track feature set size for schema validation |
| `train_rows` | Metric | `len(X_train)` after split | `8000` | Actual training set size |
| `test_rows` | Metric | `len(X_test)` after split | `2000` | Actual test set size |
| `selection_train_rows` | Metric | `len(X_sel)` (sampled for model selection) | `500` | Rows used for fast model selection |
| `total_rows_loaded` | Metric | Total rows from S3 before sampling | `100000` | Original dataset size |
| `sampling_ratio` | Metric | `n_samples / total_rows` | `0.10` | Fraction of data actually used |
| `dataset_memory_mb` | Metric | `df.memory_usage(deep=True).sum() / (1024**2)` | `85.3` | In-memory footprint |
| `sampling_method` | Param | Derived (`random`, `stratified`, `none`) | `"stratified"` | How sampling was done |
| `label_column` | Param | KFP input | `"target"` | Target column name |
| `task_type` | Param | KFP input | `"binary"` | Classification or regression |

**Currently computed but NOT persisted:** Class distribution counts (for stratified sampling), exact column types, missing value counts per column.

#### Timeseries data loader

| Metric/Parameter | Type | Source | Value Example | Purpose |
|------------------|------|--------|---------------|---------|
| `n_unique_series` | Metric | `df[id_column].nunique()` | `1000` | Number of distinct time series |
| `total_rows_loaded` | Metric | Total rows from S3 | `50000` | Original dataset size |
| `sampled_rows` | Metric | Rows after duplicate removal | `48500` | Cleaned dataset size |
| `duplicates_dropped` | Metric | Rows removed (duplicate `(id, timestamp)`) | `1500` | Data quality indicator |
| `avg_series_length` | Metric | `len(df) / n_unique_series` | `48.5` | Average observations per series |
| `selection_train_rows` | Metric | `len(selection_train_df)` | `30000` | Rows for model selection |
| `extra_train_rows` | Metric | `len(extra_train_df)` | `15000` | Additional rows for full refit |
| `test_rows` | Metric | `len(test_df)` | `3500` | Held-out test set size |
| `id_column` | Param | KFP input | `"product_id"` | Series identifier column |
| `timestamp_column` | Param | KFP input | `"date"` | Time column name |
| `target` | Param | KFP input | `"sales"` | Forecast target column |
| `prediction_length` | Param | KFP input | `7` | Forecast horizon (time steps) |

**Currently computed but NOT persisted:** Time range (min/max timestamps), series length distribution (min/max/median), frequency detection.

**Implementation note:** These metrics should be logged in the data loader components **immediately after split computation** (before returning artifacts) by checking if `MLFLOW_TRACKING_URI` is set and resuming the parent run to log data-level metrics.

### Run lifecycle and hierarchy

Following MLflow’s nested run pattern (similar to GridSearchCV with parent-child structure):

```python
import mlflow
import os
import json
from typing import Optional

def log_automl_results(
    models_artifact_path: str,
    metrics_json_path: str,
    model_names: list[str],
    pipeline_params: dict
):
    """
    Log AutoML training results to MLflow.
    Pattern: Use KFP-managed parent run (MLFLOW_RUN_ID), create child runs per model.
    
    In RHOAI 3.5+, KFP automatically sets:
    - MLFLOW_TRACKING_URI: tracking server endpoint
    - MLFLOW_RUN_ID: parent run for this pipeline execution
    - MLFLOW_EXPERIMENT_ID: experiment for this pipeline
    - MLFLOW_TRACKING_AUTH: pod service account token
    - MLFLOW_WORKSPACE: project/namespace
    """
    # Check if MLflow tracking is enabled
    tracking_uri = os.getenv("MLFLOW_TRACKING_URI")
    if not tracking_uri:
        print("MLFLOW_TRACKING_URI not set, skipping MLflow logging")
        return
    
    # Get KFP-managed parent run ID (set by KFP in RHOAI 3.5+)
    parent_run_id = os.getenv("MLFLOW_RUN_ID")
    if not parent_run_id:
        print("MLFLOW_RUN_ID not set, cannot create child runs")
        return
    
    # Resume the KFP-managed parent run to add pipeline-level metadata
    with mlflow.start_run(run_id=parent_run_id) as parent_run:
        
        # --- Parent run: Pipeline-level metadata ---
        
        # Log tags for KFP correlation
        # Note: kfp_run_id can be extracted from KFP env vars or pipeline context
        kfp_run_id = os.getenv("KFP_RUN_ID", parent_run_id)  # Fallback to MLflow run ID
        mlflow.set_tags({
            "kfp_run_id": kfp_run_id,
            "pipeline_name": "autogluon_tabular_training_pipeline",
            "task_type": pipeline_params.get("task_type", "unknown"),
            "namespace": os.getenv("MLFLOW_WORKSPACE", "unknown")
        })
        
        # Log pipeline parameters (from KFP inputs)
        import autogluon
        mlflow.log_params({
            "eval_metric": pipeline_params.get("eval_metric"),
            "preset": pipeline_params.get("preset", "medium_quality"),
            "top_n": pipeline_params.get("top_n", 3),
            "task_type": pipeline_params.get("task_type"),
            "pipeline_version": os.getenv("PIPELINE_VERSION", "unknown"),  # From image label or git SHA
            "autogluon_version": autogluon.__version__
        })
        
        # Log parent-level metrics
        with open(metrics_json_path) as f:
            all_metrics = json.load(f)
        
        best_score = max(m["score_val"] for m in all_metrics.values())
        mlflow.log_metric("best_model_score", best_score)
        mlflow.log_metric("total_training_time", sum(m.get("fit_time", 0) for m in all_metrics.values()))
        mlflow.log_metric("num_models_trained", len(all_metrics))
        
        # Log parent-level artifacts
        mlflow.log_artifact("leaderboard.html", artifact_path="reports")
        
        # --- Child runs: One per refitted model ---
        
        for model_name in model_names:
            model_metrics = all_metrics.get(model_name, {})
            
            with mlflow.start_run(
                run_name=model_name, 
                nested=True,
                tags={"model_name": model_name}
            ) as child_run:
                
                # Log model-specific parameters
                # Extract model type and stack level from model name (e.g., "CatBoost_BAG_L2")
                model_type = model_name.split("_")[0] if "_" in model_name else model_name
                stack_level = int(model_name.split("_L")[-1]) if "_L" in model_name else 1
                
                mlflow.log_params({
                    "model_type": model_type,  # e.g., "WeightedEnsemble", "CatBoost"
                    "stack_level": stack_level,  # e.g., 2, 3
                    "fit_time": model_metrics.get("fit_time", 0),
                    "predict_time": model_metrics.get("pred_time_val", 0)
                })
                
                # Log model-specific metrics
                # Primary score
                mlflow.log_metric("score_val", model_metrics.get("score_val", 0))
                if "score_test" in model_metrics:
                    mlflow.log_metric("score_test", model_metrics["score_test"])
                
                # Task-specific metrics (classification example)
                if pipeline_params.get("task_type") in ["binary", "multiclass"]:
                    for metric in ["accuracy", "f1", "precision", "recall", "roc_auc"]:
                        if metric in model_metrics:
                            mlflow.log_metric(metric, model_metrics[metric])
                
                # Regression metrics
                elif pipeline_params.get("task_type") == "regression":
                    for metric in ["rmse", "mae", "r2", "mse"]:
                        if metric in model_metrics:
                            mlflow.log_metric(metric, model_metrics[metric])
                
                # Log model-specific artifacts (small JSON only)
                model_metrics_file = f"{model_name}_metrics.json"
                with open(model_metrics_file, "w") as f:
                    json.dump(model_metrics, f, indent=2)
                mlflow.log_artifact(model_metrics_file, artifact_path="model_metrics")
                
                # Optional: log feature importance if available
                if "feature_importance" in model_metrics:
                    importance_file = f"{model_name}_feature_importance.json"
                    with open(importance_file, "w") as f:
                        json.dump(model_metrics["feature_importance"], f, indent=2)
                    mlflow.log_artifact(importance_file, artifact_path="feature_importance")
        
        print(f"Logged {len(model_names)} models to MLflow parent run: {parent_run.info.run_id}")
```

### Comparison to MLflow framework integrations

| Aspect | MLflow scikit-learn/XGBoost Autolog | AutoML MLflow Integration (RHOAI 3.5+) |
|--------|-------------------------------------|----------------------------------------|
| **Automatic logging** | `mlflow.sklearn.autolog()` captures all params/metrics automatically | **Manual logging** via explicit `mlflow.log_params()` / `mlflow.log_metrics()` calls for predictability in KFP environment. KFP **mode 1** (auto KFP→MLflow) logs simple metrics, but AutoML uses **mode 2** (custom logging) for complex artifacts. |
| **Nested runs** | GridSearchCV/RandomizedSearchCV auto-creates parent-child hierarchy | **Explicit nested runs**: parent = KFP pipeline run, children = refitted models from leaderboard |
| **Parameter logging** | Auto-captures via `estimator.get_params()` | **Selective logging**: pipeline inputs (`preset`, `eval_metric`, `top_n`) + derived params (`autogluon_version`, `stack_level`) |
| **Metric logging** | Training scores from `.score()` method, per-iteration for boosting | **Leaderboard-based**: validation scores (`score_val`, `score_test`) + task-specific metrics from `metrics.json` |
| **Model logging** | `mlflow.sklearn.log_model()` with model signature and input example | **Not implemented**: logs **URIs/hashes** only. Full model logging would require `mlflow.pyfunc.log_model()` with custom AutoGluon wrapper and storage policy definition (MLflow has no native AutoGluon flavor) |
| **Artifact logging** | Plots (feature importance, confusion matrix) via autolog | **Explicit artifacts**: `leaderboard.html`, per-model `metrics.json`, optional `feature_importance.json` |
| **Dependencies** | Auto-generates `requirements.txt`, `conda.yaml` from environment | **Reproducibility via parameters**: `autogluon_version`, `pipeline_version` logged as params for tracking. |
| **Run timing** | Runs created/closed automatically around `fit()` call | **Component-level**: parent run created in `leaderboard_evaluation` component after all training completes |

### Concrete parameter sources (tabular)

Parameters logged to the **parent run** are sourced from:

| Parameter | Source | Example Value | Notes |
|-----------|--------|---------------|-------|
| `eval_metric` | KFP pipeline input `eval_metric` | `"accuracy"` | From pipeline.py parameter (see [experiment_settings.md](./experiment_settings.md) for all supported metrics) |
| `preset` | KFP pipeline input `preset` | `"medium_quality"` | From pipeline.py parameter (RHOAI 3.5+; see [experiment_settings.md](./experiment_settings.md) for supported presets) |
| `top_n` | KFP pipeline input `top_n` | `3` | From pipeline.py parameter |
| `task_type` | KFP pipeline input `task_type` | `"binary"` | From pipeline.py parameter (`binary`, `multiclass`, `regression`) |
| `label_column` | KFP pipeline input `label_column` | `"target"` | From pipeline.py parameter (tabular) |
| `train_data_bucket_name` | KFP pipeline input | `"my-training-data"` | S3 bucket name (non-secret metadata) |
| `train_data_file_key` | KFP pipeline input | `"datasets/train.csv"` | S3 object key (non-secret metadata) |
| `sampling_method` | Derived in data loader | `"stratified"` | Based on data size and task type |
| `test_size` | Data loader split config | `0.2` | Train/test split ratio |
| `pipeline_version` | Git SHA or image digest | `"a1b2c3d"` | From `PIPELINE_VERSION` env var, image label, or git SHA |
| `autogluon_version` | Python package version | `"1.1.0"` | Via `autogluon.__version__` |
| `kfp_run_id` | KFP workflow environment variable | `"run-abc123"` | From `KFP_RUN_ID` env var or context |
| `namespace` | Kubernetes namespace | `"data-science-project"` | From pod metadata or env var |

**Timeseries-specific parameters (additional for timeseries pipeline):**

| Parameter | Source | Example Value | Notes |
|-----------|--------|---------------|-------|
| `target` | KFP pipeline input | `"sales"` | Target column to forecast |
| `id_column` | KFP pipeline input | `"product_id"` | Series identifier column |
| `timestamp_column` | KFP pipeline input | `"date"` | Timestamp column |
| `prediction_length` | KFP pipeline input | `7` | Forecast horizon (time steps) |
| `known_covariates_names` | KFP pipeline input | `["holiday", "promo"]` | Known future covariates (if used) |

Parameters logged to each **child run** (per model):

| Parameter | Source | Example Value | Notes |
|-----------|--------|---------------|-------|
| `model_type` | Model name parsing | `"CatBoost"` | Extracted from model name like `CatBoost_BAG_L2` |
| `stack_level` | Model name parsing | `2` | Extracted from suffix `_L2` in model name |
| `num_models_in_stack` | Metadata or model inspection | `5` | If available from AutoGluon metadata |
| `fit_time` | `metrics.json` → `fit_time` | `42.5` | Seconds to train the model |
| `predict_time` | `metrics.json` → `pred_time_val` | `0.8` | Seconds to predict on validation set |

### Concrete metric sources (tabular)

Metrics logged to the **parent run**:

**Data-level metrics (from data loader):**

| Metric | Source | Example Value | Notes |
|--------|--------|---------------|-------|
| `n_features` | `len(X.columns)` | `47` | Number of feature columns |
| `train_rows` | `len(X_train)` | `8000` | Training set size |
| `test_rows` | `len(X_test)` | `2000` | Test set size |
| `selection_train_rows` | `len(X_sel)` | `500` | Rows for model selection (sampled) |
| `total_rows_loaded` | Rows from S3 | `100000` | Original dataset size |
| `sampling_ratio` | `n_samples / total_rows` | `0.10` | Fraction of data used |
| `dataset_memory_mb` | `df.memory_usage(deep=True).sum() / (1024**2)` | `85.3` | Memory footprint |

**Training-level metrics (from leaderboard evaluation):**

| Metric | Source | Example Value | Notes |
|--------|--------|---------------|-------|
| `best_model_score` | Max `score_val` from `metrics.json` | `0.947` | Best validation score across all models |
| `total_training_time` | Sum of `fit_time` from `metrics.json` | `187.3` | Total seconds spent training all models |
| `num_models_trained` | Count of entries in `metrics.json` | `15` | Including intermediate and refitted models |

Metrics logged to each **child run** (per model):

| Metric | Source | Example Value | Notes |
|--------|--------|---------------|-------|
| `score_val` | `metrics.json` → `score_val` | `0.947` | Validation score for ranking (higher is better after AutoGluon transform) |
| `score_test` | `metrics.json` → `score_test` | `0.942` | Test score if available |
| `accuracy` | `metrics.json` → `accuracy` | `0.947` | Classification: fraction correct |
| `f1` | `metrics.json` → `f1` | `0.935` | Classification: F1 score |
| `roc_auc` | `metrics.json` → `roc_auc` | `0.982` | Classification: ROC AUC |
| `precision` | `metrics.json` → `precision` | `0.928` | Classification: precision |
| `recall` | `metrics.json` → `recall` | `0.943` | Classification: recall |
| `rmse` | `metrics.json` → `rmse` | `12.4` | Regression: root mean squared error |
| `mae` | `metrics.json` → `mae` | `9.1` | Regression: mean absolute error |
| `r2` | `metrics.json` → `r2` | `0.89` | Regression: R² score |

**Note:** Metric names in `metrics.json` align with AutoGluon’s leaderboard output. The evaluation component should map these to MLflow metrics based on the `task_type` to ensure consistency.

### Artifacts and naming conventions

**RHOAI 3.4 artifacts:**

| Artifact | Type | Location | Size Guidance | Logged To |
|----------|------|----------|---------------|-----------|
| `leaderboard.html` | HTML report | `mlflow-artifacts/<run_id>/reports/leaderboard.html` | < 1 MB (cap at 5000 rows if needed) | Parent run |
| `<model>_metrics.json` | JSON metrics | `mlflow-artifacts/<run_id>/model_metrics/<model>_metrics.json` | < 50 KB | Child run (per model) |
| `<model>_feature_importance.json` | JSON feature importance | `mlflow-artifacts/<run_id>/feature_importance/<model>_feature_importance.json` | < 100 KB | Child run (tabular only, classification/regression) |
| `<model>_confusion_matrix.json` | Confusion matrix | `mlflow-artifacts/<run_id>/model_metrics/<model>_confusion_matrix.json` | < 20 KB | Child run (tabular only, classification) |

**RHOAI 3.5+ planned artifacts** (see [model_insights.md](./model_insights.md) for schema details):

| Artifact | Type | Location | Size Guidance | Logged To | Availability |
|----------|------|----------|---------------|-----------|--------------|
| `<model>_roc_curve.json` | ROC curve data | `mlflow-artifacts/<run_id>/model_metrics/<model>_roc_curve.json` | < 50 KB | Child run (tabular classification only) | **RHOAI 3.5+** |
| `<model>_precision_recall_curve.json` | PR curve data | `mlflow-artifacts/<run_id>/model_metrics/<model>_precision_recall_curve.json` | < 50 KB | Child run (tabular classification only) | **RHOAI 3.5+** |
| `<model>_back_testing.json` | Back-testing results | `mlflow-artifacts/<run_id>/model_metrics/<model>_back_testing.json` | < 100 KB | Child run (timeseries only) | **RHOAI 3.5+** |

**Additional metrics from RHOAI 3.5+ artifacts** (logged as MLflow metrics when artifacts are generated):

**For tabular classification** (binary):
- `num_samples`: Total test samples
- `num_positive`: Positive class samples  
- `num_negative`: Negative class samples
- `auc`: ROC AUC (from `roc_curve.json`)
- `average_precision`: AP score (from `precision_recall_curve.json`)

**For tabular classification** (multiclass):
- `num_classes`: Number of classes
- `auc_macro`: Macro-averaged AUC (from `roc_curve.json`)
- `auc_weighted`: Weighted AUC (from `roc_curve.json`)
- `average_precision_macro`: Macro-averaged AP (from `precision_recall_curve.json`)
- `average_precision_weighted`: Weighted AP (from `precision_recall_curve.json`)

**For timeseries** (from `back_testing.json`):
- `num_val_windows`: Number of backtesting windows
- `num_series_evaluated`: Total series in backtest
- `series_with_degraded_performance`: Count of problematic series
- Per-window metrics: `WQL`, `MAPE`, `MASE`, `RMSE`, `MAE` (logged as metric arrays or separate metrics per window)

**Best practice:** Follow MLflow’s artifact organization by using `artifact_path` parameter in `mlflow.log_artifact()` to create logical groupings (`reports/`, `metadata/`, `model_metrics/`, `feature_importance/`, `curves/`).

---

## KFP Pipeline Integration: MLflow Tracking Artifact

To enable the AutoML Dashboard and end users to discover and access MLflow tracking information from KFP pipeline runs, the `autogluon_leaderboard_evaluation` component produces a dedicated **MLflow tracking artifact** as a KFP output.

### Artifact Purpose

The `mlflow_tracking_artifact` serves as the **single source of truth** for linking KFP pipeline executions to MLflow experiment tracking, enabling:

1. **Dashboard integration:** AutoML Dashboard fetches this artifact via KFP API to retrieve MLflow experiment/run IDs
2. **Deep-linking:** Users can navigate directly from KFP UI to MLflow UI to view detailed metrics, charts, and model comparisons
3. **Programmatic access:** CI/CD pipelines and automation scripts can query MLflow tracking info without parsing component logs
4. **Audit trail:** Permanent record of which MLflow run corresponds to each KFP pipeline execution

### Component Output Definition

```json
 mlflow_tracking_artifact: dsl.Output[dsl.Artifact]
```

### Artifact Schema

The artifact is a JSON file written to `mlflow_tracking_artifact.path` with the following structure:

```json
{
  "tracking_enabled": true,
  "mlflow_tracking_uri": "https://mlflow-server.example.com",
  "mlflow_experiment_id": "5",
  "mlflow_run_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
  "mlflow_workspace": "data-science-project",
  "mlflow_run_url": "https://mlflow-server.example.com/#/experiments/5/runs/a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5"
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "autogluon-tabular-training-pipeline",
}
```

**Field descriptions:**

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `mlflow_tracking_uri` | string | `MLFLOW_TRACKING_URI` env var | MLflow tracking server endpoint |
| `mlflow_experiment_id` | string | `MLFLOW_EXPERIMENT_ID` env var | Experiment ID (KFP creates one per pipeline) |
| `mlflow_run_id` | string | `MLFLOW_RUN_ID` env var | Parent run ID for this pipeline execution |
| `mlflow_workspace` | string | `MLFLOW_WORKSPACE` env var | OpenShift AI project/namespace |
| `kfp_run_id` | string | `dsl.PIPELINE_JOB_ID_PLACEHOLDER` | KFP pipeline run ID for cross-referencing |
| `pipeline_name` | string | `dsl.PIPELINE_JOB_RESOURCE_NAME_PLACEHOLDER` | Pipeline name |
| `tracking_enabled` | bool | `bool(MLFLOW_TRACKING_URI)` | Whether MLflow tracking was active |
| `mlflow_run_url` | string | Computed | Deep-link URL to MLflow UI parent run view |

**When MLflow tracking is disabled** (`MLFLOW_TRACKING_URI` not set):

```json
{
  "tracking_enabled": false,
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "autogluon-tabular-training-pipeline",
}
```

---


## References

### AutoML Implementation

- Upstream AutoML components: [opendatahub-io/pipelines-components — `components/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl)
- Upstream AutoML pipelines: [opendatahub-io/pipelines-components — `pipelines/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/automl)
- End-user examples (RH): [red-hat-ai-examples — `examples/automl`](https://github.com/red-hat-data-services/red-hat-ai-examples/tree/main/examples/automl)

### MLflow on RHOAI

- **MLflow on RHOAI Integration Guide** (internal): Contact Matt Prahl or Humair Khan in `#wg-openshift-ai-mlflow-integration` for the latest integration guide
- MLflow Operator: [opendatahub-io/mlflow-operator](https://github.com/opendatahub-io/mlflow-operator)
- MLflow Workspaces documentation: [MLflow 3.10 release notes](https://github.com/mlflow/mlflow/releases/tag/v3.10.0) (workspace feature contributed by Red Hat)
- MLflow RBAC authorization plugin: [Kubernetes auth plugin documentation](https://mlflow.org/docs/latest/auth/index.html#kubernetes-authorization)

### MLflow Framework Integration

- MLflow Python Function (custom models): [MLflow pyfunc documentation](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#creating-custom-pyfunc-models)
- MLflow Tracking: [MLflow Tracking documentation](https://mlflow.org/docs/latest/tracking.html)
- MLflow Model Registry: [MLflow Model Registry documentation](https://mlflow.org/docs/latest/model-registry.html)
