# AutoML MLflow integration

This document proposes an **MLflow** integration for OpenShift AI / ODH **AutoML** workflows implemented in [opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components) under **`components/training/automl`** and **`pipelines/training/automl`**, with product goals: **experiment tracking, comparison, reproducibility**.

**RHOAI 3.5 scope:** Core MLflow integration with KFP environment variables, parent/child run hierarchy, and existing artifacts (leaderboard, metrics, feature importance, confusion matrix).


## Table of Contents

- [KFP MLflow integration modes (OpenShift AI 3.5+)](#kfp-mlflow-integration-modes-openshift-ai-35)
  - [Environment variables (RHOAI 3.5+)](#environment-variables-rhoai-35)
- [MLflow mapping model](#mlflow-mapping-model)
  - [Optional alignment with AutoGluon-native logging](#optional-alignment-with-autogluon-native-logging)
- [Per-component logging details](#per-component-logging-details)
  - [Component-level logging matrix (tabular pipeline)](#component-level-logging-matrix-tabular-pipeline)
  - [Component-level logging matrix (timeseries pipeline)](#component-level-logging-matrix-timeseries-pipeline)
  - [Run lifecycle and hierarchy](#run-lifecycle-and-hierarchy)
  - [Key differences from MLflow framework integrations](#key-differences-from-mlflow-framework-integrations)
  - [Artifacts and naming conventions](#artifacts-and-naming-conventions)
- [KFP Pipeline Integration: MLflow Tracking Artifact](#kfp-pipeline-integration-mlflow-tracking-artifact)
  - [Artifact Purpose](#artifact-purpose)
  - [Component Output Definition](#component-output-definition)
  - [Artifact Schema](#artifact-schema)
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

| MLflow concept | Proposed mapping                                                                                                                                                                                                                                                                                                                                                                                                                |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Experiment** | KFP auto-creates one experiment per pipeline (accessible via `MLFLOW_EXPERIMENT_ID`).                                                                                                                                                                                                                                                                                                                           |
| **Parent run** | KFP auto-creates one parent run per pipeline execution (accessible via `MLFLOW_RUN_ID`). AutoML components **resume this run** to add tags and params. **Tags:** `kfp_run_id`, `pipeline_name`, `task_type` (`tabular` \| `timeseries`), `namespace` (from `MLFLOW_WORKSPACE`), dataset **hashes or URIs** (non-secret). **Params:** `eval_metric`, `preset`, `top_n`, `pipeline_version`, `autogluon_version`. |
| **Child runs** | **One child run per leaderboard row / refitted model** (each name in `model_names` or equivalent for timeseries), created by AutoML components as nested runs under the KFP parent. Enables side-by-side comparison in the MLflow UI across models from the same pipeline run.                                                                                                                                                  |
| **Params** | **Parent:** `eval_metric`, `preset`, `top_n`, `task_type`, `pipeline_version`, `autogluon_version`. **Child:** `model_type`, `stack_level`, `fit_time`, `predict_time`, `num_bag_folds` / `num_stack_levels` when exposed.                                                                                                                                                                                                      |
| **Metrics** | **Parent:** `best_model_score`, `total_training_time`, `num_models_trained`. **Child:** Primary validation score (`score_val`, `score_test`), task-specific metrics from AutoGluon leaderboard / `metrics.json` (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`, `mae`), **fit time**.                                                                                                                                               |
| **Artifacts** | **Per child:** pointer to **`metrics`**, and models **`predictor`**.                                                                                                                                                                                                                                                                                                                                                            |

### Optional alignment with AutoGluon-native logging

AutoGluon can integrate with experiment trackers in some setups. Prefer **explicit MLflow calls in our components** first so KFP + OpenShift behavior stays predictable; revisit **native AutoGluon callbacks** only if they reduce duplication without fighting Kubeflow’s filesystem layout.

---

## Per-component logging details

This section provides concrete implementation guidance for MLflow integration in each AutoML pipeline component, following patterns established by MLflow’s scikit-learn and XGBoost integrations (autologging, nested runs, model signatures).

> **Important:** MLflow does **not** have native AutoGluon support. There is no `mlflow.autogluon` module or autologging capability. All tracking must be implemented via **explicit MLflow API calls** (`mlflow.log_params()`, `mlflow.log_metrics()`, `mlflow.log_artifact()`). Model logging would require a custom `mlflow.pyfunc` wrapper since AutoGluon predictors cannot be logged with standard MLflow flavors.

### Component-level logging matrix (tabular pipeline)

| Component | MLflow Operation | Parameters Logged | Metrics Logged | Artifacts Logged | Run Type |
|-----------|------------------|-------------------|----------------|------------------|----------|
| **tabular_data_loader** | Params + metrics | `sampling_method`, `task_type`, `label_column`, `test_size`, `random_state`, `stratify` | — | — | Parent |
| **autogluon_models_training** | Tags + params | `preset`, `eval_metric`, `time_limit`, `num_bag_folds`, `num_stack_levels`, `top_n`, `task_type` | `selection_time_seconds` | — | Parent |
| **autogluon_leaderboard_evaluation** | **Parent run creation + child runs** | (per parent): `pipeline_version`, `autogluon_version`, `kfp_run_id`, `namespace` | (per parent): `best_model_score`, `total_training_time` | (per parent): `leaderboard.html` | Parent + N children |
| **autogluon_leaderboard_evaluation** (per model) | Child run per model | (per child): `model_type`, `num_models_in_stack`, `fit_time`, `predict_time`, `stack_level` | (per child): `score_val`, `score_test` (if available), task-specific metrics (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`) | (per child): `model_metrics.json`, `feature_importance.json`, `confusion_matrix.json` (classification) | Child (nested) |

### Component-level logging matrix (timeseries pipeline)

| Component | MLflow Operation | Parameters Logged | Metrics Logged | Artifacts Logged | Run Type |
|-----------|------------------|-------------------|----------------|------------------|----------|
| **timeseries_data_loader** | Params + metrics | `id_column`, `timestamp_column`, `target`, `known_covariates_names`, `prediction_length`, `test_size`, `selection_train_size` | — | — | Parent |
| **autogluon_timeseries_models_selection** | Tags + params | `preset`, `eval_metric`, `prediction_length`, `target`, `known_covariates_names`, `top_n` | `selection_time_seconds` | — | Parent |
| **autogluon_timeseries_models_full_refit** | Metrics per model | — | `refit_time_seconds` (per model) | — | Parent |
| **autogluon_timeseries_leaderboard_evaluation** | **Parent run creation + child runs** | (per parent): `pipeline_version`, `autogluon_version`, `kfp_run_id`, `namespace` | (per parent): `best_model_score`, `total_training_time` | (per parent): `leaderboard.html` | Parent + N children |
| **autogluon_timeseries_leaderboard_evaluation** (per model) | Child run per model | (per child): `model_type`, `fit_time`, `predict_time` | (per child): `score_val`, task-specific metrics (e.g., `WQL`, `MAPE`, `MASE`, `RMSE`) | (per child): `model_metrics.json` | Child (nested) |

### Run lifecycle and hierarchy

Following MLflow’s nested run pattern (similar to GridSearchCV with parent-child structure):

```python
import mlflow
import os
import json

def log_automl_results(metrics_json_path: str, model_names: list[str], pipeline_params: dict):
    """
    Log AutoML results to MLflow using KFP-managed parent run.
    KFP automatically sets MLFLOW_TRACKING_URI, MLFLOW_RUN_ID, MLFLOW_EXPERIMENT_ID.
    """
    parent_run_id = os.getenv("MLFLOW_RUN_ID")
    if not parent_run_id:
        print("MLflow not enabled, skipping logging")
        return
    
    # Resume KFP-managed parent run
    with mlflow.start_run(run_id=parent_run_id):
        # Log pipeline-level params and metrics
        import autogluon
        mlflow.log_params({
            "preset": pipeline_params.get("preset", "medium_quality"),
            "eval_metric": pipeline_params.get("eval_metric"),
            "autogluon_version": autogluon.__version__
        })
        
        with open(metrics_json_path) as f:
            all_metrics = json.load(f)
        
        mlflow.log_metric("best_model_score", max(m["score_val"] for m in all_metrics.values()))
        mlflow.log_artifact("leaderboard.html", artifact_path="reports")
        
        # Create child runs for each model
        for model_name in model_names:
            model_metrics = all_metrics.get(model_name, {})
            with mlflow.start_run(run_name=model_name, nested=True):
                mlflow.log_metric("score_val", model_metrics.get("score_val", 0))
                # Log task-specific metrics
                for metric in ["accuracy", "f1", "rmse", "mae"]:
                    if metric in model_metrics:
                        mlflow.log_metric(metric, model_metrics[metric])
```

### Key differences from MLflow framework integrations

**AutoML uses explicit manual logging** instead of MLflow's autolog feature because:

- **Predictability in KFP**: AutoML uses explicit `mlflow.log_params()` / `mlflow.log_metrics()` calls rather than `mlflow.sklearn.autolog()` to control exactly what gets logged in the KFP pipeline environment
- **Explicit nested runs**: Parent run = KFP pipeline execution, child runs = individual refitted models from the leaderboard (not auto-created by GridSearchCV)
- **Selective logging**: Logs pipeline inputs (`preset`, `eval_metric`, `top_n`) plus derived parameters (`autogluon_version`, `stack_level`) rather than capturing all estimator parameters via `.get_params()`
- **Leaderboard-based metrics**: Logs validation scores (`score_val`, `score_test`) and task-specific metrics from `metrics.json` rather than training scores from `.score()` method
- **No model artifacts**: Logs model URIs/hashes only. Full model logging would require `mlflow.pyfunc.log_model()` with custom AutoGluon wrapper (MLflow has no native AutoGluon flavor)
- **Component-level timing**: Parent run created in `leaderboard_evaluation` component after all training completes, not automatically around `fit()` call


### Artifacts and naming conventions

**RHOAI 3.4 artifacts:**

| Artifact | Type | Location | Size Guidance | Logged To |
|----------|------|----------|---------------|-----------|
| `leaderboard.html` | HTML report | `mlflow-artifacts/<run_id>/reports/leaderboard.html` | < 1 MB (cap at 5000 rows if needed) | Parent run |
| `<model>_metrics.json` | JSON metrics | `mlflow-artifacts/<run_id>/model_metrics/<model>_metrics.json` | < 50 KB | Child run (per model) |
| `<model>_feature_importance.json` | JSON feature importance | `mlflow-artifacts/<run_id>/feature_importance/<model>_feature_importance.json` | < 100 KB | Child run (tabular only, classification/regression) |
| `<model>_confusion_matrix.json` | Confusion matrix | `mlflow-artifacts/<run_id>/model_metrics/<model>_confusion_matrix.json` | < 20 KB | Child run (tabular only, classification) |

**RHOAI 3.5 scope (bare minimum):**

RHOAI 3.5 introduces **MLflow integration with KFP environment variables** and the **parent/child run hierarchy** for experiment tracking. The 3.5 release focuses on **core tracking capabilities** with the existing artifact set from 3.4 (leaderboard, metrics, feature importance, confusion matrix).

**Best practice:** Follow MLflow’s artifact organization by using `artifact_path` parameter in `mlflow.log_artifact()` to create logical groupings (`reports/`, `metadata/`, `model_metrics/`, `feature_importance/`).

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
