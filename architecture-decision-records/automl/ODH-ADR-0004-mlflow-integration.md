# AutoML MLflow Integration

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-25 |
| Scope          | AutoML |
| Status         | Approved |
| Authors        | [Lukasz Cmielowski](@LukaszCmielowski) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1493](https://redhat.atlassian.net/browse/RHAISTRAT-1493) |
| Other docs:    | [ODH-ADR-0001-automl](./ODH-ADR-0001-automl.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0003-model-insights](./ODH-ADR-0003-model-insights.md) |

## What

Integrate MLflow experiment tracking into the AutoML components and pipelines in [pipelines-components](https://github.com/opendatahub-io/pipelines-components).

## Why

MLflow provides the OpenShift AI-supported UI needed to compare AutoGluon models, trace parameters and metrics, and reproduce runs.

## Goals

* Track AutoML experiments with parent/child MLflow runs configured by KFP-injected `KFP_MLFLOW_CONFIG`.
* Log existing training artifacts and expose MLflow discovery and deep-linking through `component_stage_map`.

## Non-Goals

* Native AutoGluon MLflow autologging (no `mlflow.autogluon` module exists)
* MLflow Model Registry integration (model registration is out of scope)

## How

### KFP MLflow integration mode

When MLflow is enabled on the pipeline server, KFP injects `KFP_MLFLOW_CONFIG` into every step pod. AutoML uses it for opt-in tracking.

| Environment variable | Purpose | Notes |
|----------------------|---------|-------|
| `KFP_MLFLOW_CONFIG` | Platform MLflow config (JSON) | Absent, empty, invalid JSON, or missing `endpoint` disables tracking (best-effort; training still runs) |

| JSON field | Purpose | Notes |
|------------|---------|-------|
| `endpoint` | MLflow tracking server URL | Required for tracking. Kubernetes auth is refused on non-HTTPS endpoints (no bearer token over cleartext). |
| `experimentId` | KFP-managed experiment ID | Optional. If missing, the stage-map publisher creates or uses an experiment named after the KFP run. |
| `parentRunId` | KFP-managed parent run ID | When set, the stage-map publisher records this run. When unset, the publisher creates the parent run before writing the stage map. |
| `workspacesEnabled` / `workspace` | Multi-tenant workspace | `workspace` is used only when `workspacesEnabled` is true. Sent as the `x-mlflow-workspace` request header. |
| `authType` | Authentication | `kubernetes`: read the pod service-account token and set `MLFLOW_TRACKING_TOKEN` (Bearer). |
| `timeout` | HTTP timeout | Applied as `MLFLOW_HTTP_REQUEST_TIMEOUT` (whole seconds, e.g. `"30s"` → `30`). |

Without a valid blob, tracking is disabled; there is no pipeline parameter to turn logging off. With one, the stage-map publisher resolves the parent run and training components create child runs and explicitly call `mlflow.log_params()`, `mlflow.log_metrics()`, and `mlflow.log_artifact()`.

### MLflow mapping model

Tabular and timeseries pipelines use the same mapping.

| MLflow concept      | Proposed mapping |
|---------------------|------------------|
| **Experiment**      | KFP-managed experiment from `KFP_MLFLOW_CONFIG.experimentId`; if absent, the stage-map publisher creates or uses one named for the KFP run. |
| **Parent run**      | KFP-managed parent from `KFP_MLFLOW_CONFIG.parentRunId`, or a parent created by the stage-map publisher before training begins. AutoML components add tags, params, and **aggregate metrics**. **Tags:** `pipeline_name`, `kfp_run_name`, `kfp_run_id`, `kfp_version`, `image`, `autogluon_version`. **Params:** `task_type` (`binary` \| `multiclass` \| `regression` \| `time_series`), `eval_metric`, `preset`, `top_n`, dataset **hashes or URIs** (non-secret). |
| **Parent Metrics**  | `best_score`, `worst_score`, `mean_score`, `num_models_trained`, `total_fit_time_seconds`. |
| **Child runs**      | **One child run per leaderboard row / refitted model** (each `name` in `model_names` or equivalent for timeseries), created as nested runs under the KFP parent. Enables side-by-side comparison in MLflow UI. Params: `model_type`, `stack_level`, `fit_time`, `predict_time`, `num_bag_folds` / `num_stack_levels` when exposed. |
| **Child Metrics**   | Task-specific metrics from AutoGluon leaderboard / `metrics.json` (e.g., `accuracy`, `f1`, `roc_auc`, `rmse`, `mae`). |
| **Child Artifacts** | Pointer to **`metrics`** (containing model's insights like confusion matrix etc.), pointer to trained model binaries **`predictor`**, and pointer to **`notebook`**. |

### Implementation approach

MLflow has no native AutoGluon support, so tracking uses explicit APIs and nested runs. Logging a predictor requires a custom `mlflow.pyfunc` wrapper.

### Alignment with AutoGluon-native logging

Revisit native AutoGluon callbacks for incremental logging as models complete.

### Extending `component_stage_map.json`

`publish_component_stage_map`, the first pipeline task, resolves the MLflow experiment and parent run, then writes `component_stage_map.json`. When `KFP_MLFLOW_CONFIG` has a valid `endpoint`, it adds the following top-level `mlflow` object; existing fields remain unchanged.

| `mlflow` field | Type | Source (`KFP_MLFLOW_CONFIG`) | Description |
|----------------|------|------------------------------|-------------|
| `tracking_uri` | string | `endpoint` | MLflow tracking server endpoint (omitted when disabled) |
| `experiment_id` | string | `experimentId` or publisher-created experiment | MLflow experiment for this pipeline |
| `run_id` | string | `parentRunId` or publisher-created parent run | Parent run for this execution |
| `workspace` | string | `workspace` (when `workspacesEnabled`) | OpenShift AI project / namespace |
| `run_url` | string | Computed from `tracking_uri`, `experiment_id`, `run_id` | Deep-link to MLflow UI parent run |

The publisher populates this block at pipeline start after resolving the parent run. No new output parameter or task is needed: `component_stage_map` remains the dashboard join artifact, and training components use its resolved run ID for nested child runs.

## Alternatives

### 1. KFP automatic MLflow logging

**Discarded:** It cannot represent AutoGluon's complex artifacts, ensemble hierarchy, nested model metrics, or required parent/child structure.

### 2. Custom experiment tracking solution

**Discarded:** It would duplicate the RHOAI-supported MLflow UI, workspace isolation, Kubernetes-auth RBAC, and familiar data-scientist interface.

## Risks

* **No native AutoGluon support:** explicit API calls increase maintenance. *Mitigation:* follow MLflow scikit-learn/XGBoost patterns; use `mlflow.pyfunc` for predictor logging.
* **Configuration dependency:** `KFP_MLFLOW_CONFIG` must be valid. *Mitigation:* gracefully disable tracking when it is unset, invalid, or lacks `endpoint`; the stage map reflects the actual state.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| AutoML team                   |                  |            | Yes |
| Data Science Pipelines        |                  |            | Yes |
| MLflow integration (RHOAI)    | Matt Prahl, Humair Khan |  | Yes |
| AutoML Dashboard              |                  |            | Yes |

## References

* Component stage map publisher: [opendatahub-io/pipelines-components — `component_stage_map_publisher`](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl/component_stage_map_publisher)
* Upstream AutoML components: [opendatahub-io/pipelines-components — `components/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl)
* Upstream AutoML pipelines: [opendatahub-io/pipelines-components — `pipelines/training/automl`](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/automl)
* End-user examples (RH): [red-hat-ai-examples — `examples/automl`](https://github.com/red-hat-data-services/red-hat-ai-examples/tree/main/examples/automl)
* Sample notebook (mocked MLflow data): [mlflow_mocks.ipynb](https://github.com/LukaszCmielowski/prototypes/blob/main/AutoML/mlflow_integration/mlflow_mocks.ipynb)
* MLflow on RHOAI Integration Guide (internal): Contact Matt Prahl or Humair Khan in `#wg-openshift-ai-mlflow-integration`
* MLflow Operator: [opendatahub-io/mlflow-operator](https://github.com/opendatahub-io/mlflow-operator)
* MLflow Workspaces documentation: [MLflow 3.10 release notes](https://github.com/mlflow/mlflow/releases/tag/v3.10.0)
* MLflow RBAC authorization plugin: [Kubernetes auth plugin documentation](https://mlflow.org/docs/latest/auth/index.html#kubernetes-authorization)
* MLflow Python Function (custom models): [MLflow pyfunc documentation](https://mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#creating-custom-pyfunc-models)
* MLflow Tracking: [MLflow Tracking documentation](https://mlflow.org/docs/latest/tracking.html)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
