# Open Data Hub - AutoML Model Insights

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-29 |
| Scope          | AutoML Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1491](https://redhat.atlassian.net/browse/RHAISTRAT-1491) |
| Other docs:    | [ODH-ADR-0001-automl](./ODH-ADR-0001-automl.md) · [ODH-ADR-0002-experiment-settings](./ODH-ADR-0002-experiment-settings.md) · [ODH-ADR-0004-mlflow-integration](./ODH-ADR-0004-mlflow-integration.md) |

## What

This ADR defines the durable artifacts under each refitted AutoML model directory (`{model_name}_FULL/`): predictors, metrics, notebooks, `model.json`, and KServe scoring schemas for tabular and time-series pipelines.

## Why

Dashboard, Model Registry, and KServe clients need a stable contract for locating a selected predictor, inspecting its metrics, and constructing a scoring request without loading AutoGluon.

## Goals

* Define per-model artifact paths for tabular and time-series pipelines.
* Define `model.json` and `inference.input_data_schema`.
* Define the metric artifacts available for visualization.

## Non-Goals

* Pipeline inputs, presets, and evaluation metric selection ([ODH-ADR-0002](./ODH-ADR-0002-experiment-settings.md)).
* Automatic Model Registry registration or KServe deployment.
* MLflow run layout ([ODH-ADR-0004](./ODH-ADR-0004-mlflow-integration.md)).
* Notebook or Dashboard visualization implementation.

## How

Artifacts are produced for each refitted model. Filenames and schemas should be confirmed on the applicable **pipelines-components** tag. The run-level `experiment_notebook` is distinct from the per-model predictor notebooks; see [ODH-ADR-0001](./ODH-ADR-0001-automl.md#artifacts-generated).

## Shared model contract

Each refitted model has a `model.json` record with these fields.

| Field | Description |
|-------|-------------|
| `name` | Refitted model identifier, normally ending in `_FULL`. |
| `location.model_directory` | Model directory relative to the models artifact root. |
| `location.predictor` | Deployable `clone_for_deployment` predictor directory. This is the Model Registry / KServe storage URI. |
| `location.notebook` | Per-model predictor notebook. |
| `location.metrics` | Metrics directory. |
| `metrics.test_data` | Finite test-data scores, mirroring `metrics/metrics.json`. |
| `inference.input_data_schema` | Offline scoring contract. |
| `inference.sample_payload` | Optional non-PII placeholder request body. |

`location.back_testing` is present only for time-series models that emit `metrics/back_testing.json`.

## Tabular pipeline

`autogluon_models_training` emits a combined `Model` artifact containing one directory per refitted model. `autogluon_leaderboard_evaluation` emits the run-level leaderboard HTML.

### Per-model artifacts

| Path | Content |
|------|---------|
| `predictor/` | `clone_for_deployment` export. |
| `model.json` | Shared model contract and tabular scoring schema. |
| `metrics/metrics.json` | Test-split `evaluate_predictions` scores; non-finite values are removed. |
| `metrics/feature_importance.json` | Feature importance from the test split. |
| `metrics/confusion_matrix.json` | Classification only. |
| `metrics/curves.json` | Classification only: ROC and precision-recall data. |
| `notebooks/automl_predictor_notebook.ipynb` | Per-model predictor notebook. |

`metadata.model_names` is a JSON-encoded list for KFP metadata compatibility. `metadata.context.models[]` mirrors each `model.json`; `eval_metric` determines the leaderboard sort column.

### Tabular scoring schema

`inference.input_data_schema` is derived from `predictor.features()` and AutoGluon feature metadata. It provides a `v1_json` `instances` object with `required` and `fields`.

| Field property | Meaning |
|----------------|---------|
| `name` | v1 row key and v2 input name. |
| `datatype` | `integer`, `number`, `string`, or `boolean`. |
| `shape` | `[-1]` for the v2 batch dimension. |
| `role` | `feature`; labels are never scoring inputs. |
| `required` | Required in every request row. |

For the v1 endpoint, each tabular value is a one-element list. The schema also maps to KServe v2 types: integer → `INT64`, number → `FP64`, boolean → `BOOL`, and categorical/text/datetime values → `BYTES`. Classification predictions return labels by default; probabilities require `PREDICT_PROBA=true`.

```json
{
  "protocol": "v1_json",
  "instances": {
    "required": true,
    "fields": [
      {"name": "bedrooms", "datatype": "integer", "shape": [-1], "role": "feature", "required": true},
      {"name": "location", "datatype": "string", "shape": [-1], "role": "feature", "required": true}
    ]
  }
}
```

## Time-series pipeline

`autogluon_timeseries_models_full_refit` emits one `Model` artifact per refitted model. Persisted insight artifacts are refit outputs and leaderboard HTML.

### Per-model artifacts

| Path | Content |
|------|---------|
| `predictor/` | Saved `TimeSeriesPredictor`. |
| `predictor/predictor_metadata.json` | Model id, `prediction_length`, `eval_metric`, target, identifier, and timestamp columns. |
| `model.json` | Shared contract and time-series scoring schema. |
| `metrics/metrics.json` | Held-out `evaluate` results using available metrics; non-finite values are removed. |
| `metrics/back_testing.json` | Optional multi-window back-test summary. |
| `notebooks/automl_predictor_notebook.ipynb` | Per-model time-series predictor notebook. |

Time-series models do not emit `feature_importance.json`, `confusion_matrix.json`, or `curves.json`.

### Time-series scoring schema

The time-series schema uses `v1_json` and bare scalar values. It contains:

| Field | Meaning |
|-------|---------|
| `prediction_length` | Forecast horizon; not part of the request body. |
| `instances` | Required history rows, including identifier, timestamp, and target. |
| `known_covariates` | Required only when the model was trained with them. It contains forecast-horizon rows for each series. |
| `fields[]` | `name`, `datatype`, `role`, and `required`; roles are `id`, `timestamp`, `target`, and `known_covariate`. |

The response contains predictions keyed by identifier and timestamp, with `mean` and available quantiles. `known_covariates` and `location.back_testing` are omitted when not applicable.

## Visualization artifact contract

Visualization consumers read persisted JSON rather than reconstructing model state.

| Artifact | Contract |
|----------|----------|
| `curves.json` | Classification only. Binary records ROC and precision-recall arrays, thresholds, AUC, average precision, and class-balance baseline. Multiclass records one-vs-rest per-class curves, support, and macro/weighted aggregates. |
| `confusion_matrix.json` | Classification-only confusion matrix. |
| `feature_importance.json` | Tabular feature importance from the test split. |
| `back_testing.json` | Time-series multi-window metrics and optional best/worst series forecast detail. It records model identity, horizon, metric, window boundaries, and per-window scores. |

The Dashboard and predictor notebooks own rendering, chart selection, and explanatory text; this ADR owns only the file and data contracts.

## Related

- [AutoML architecture](./ODH-ADR-0001-automl.md)
- [AutoML experiment settings](./ODH-ADR-0002-experiment-settings.md)
- [AutoML MLflow integration](./ODH-ADR-0004-mlflow-integration.md)
- [AutoGluon TabularPredictor](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html)
- [AutoGluon TimeSeriesPredictor](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.html)
