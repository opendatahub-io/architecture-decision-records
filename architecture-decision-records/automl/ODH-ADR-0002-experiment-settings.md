# Open Data Hub - AutoML Experiment Settings

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-29 |
| Scope          | AutoML Component |
| Status         | Approved |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1492](https://redhat.atlassian.net/browse/RHAISTRAT-1492) |
| Other docs:    | [ODH-ADR-0001-automl](./ODH-ADR-0001-automl.md) |

## What

This ADR documents the public input parameters, quality presets, and evaluation metrics for the AutoML tabular and time-series training pipelines shipped in pipelines-components.

## Why

Dashboard and API integrations need a stable contract for how AutoML runs are configured (data sources, task type, presets, eval metrics) and how those map to AutoGluon TabularPredictor / TimeSeriesPredictor fits. Capturing this as an ADR keeps the contract versioned alongside the AutoML architecture.

## Goals

* Define the public parameter surface of autogluon_tabular_training_pipeline and autogluon_timeseries_training_pipeline
* Document speed and balanced presets (resources and AutoGluon mapping) for tabular and time-series modes
* Specify allowed eval_metric values and leaderboard direction conventions per task type

## Non-Goals

* Per-model artifact layout and inference schema (see ODH-ADR-0003-model-insights)
* Model Registry registration or KServe deployment as pipeline steps (post-training platform actions)


## How

This page documents the **top-level input parameters** for the two AutoML training pipelines shipped in [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) under `pipelines/training/automl/`.
Higher-level architecture and ADR-level parameter groups are in [ODH-ADR-0001-automl](./ODH-ADR-0001-automl.md). Component layout is summarized in the [AutoML component README](../../documentation/components/automl/README.md).

## Table of contents

- [Autogluon tabular training pipeline](#autogluon-tabular-training-pipeline)
- [Autogluon timeseries training pipeline](#autogluon-timeseries-training-pipeline)
- [Preset support](#preset-support)
- [Evaluation metrics](#evaluation-metrics)
  - [Tabular — binary classification (`task_type` `binary`)](#tabular--binary-classification-task_type-binary)
  - [Tabular — multiclass classification (`task_type` `multiclass`)](#tabular--multiclass-classification-task_type-multiclass)
  - [Tabular — regression (`task_type` `regression`)](#tabular--regression-task_type-regression)
  - [Time series (`TimeSeriesPredictor`)](#time-series-timeseriespredictor)

---

## Autogluon tabular training pipeline

These parameters are the public surface of `autogluon_tabular_training_pipeline` in [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/automl/autogluon_tabular_training_pipeline/pipeline.py). Confirm names and defaults on your **pipelines-components** tag. Semantics for **`preset`** / **`eval_metric`** follow **[AutoGluon Tabular](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html)** and [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html). **`preset`:** use only values in [Preset support](#preset-support). **`eval_metric`:** allowed strings are under [Evaluation metrics](#evaluation-metrics).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `train_data_secret_name` | `str` | (required) | Kubernetes **Secret** name holding S3-compatible credentials. Expected keys: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`, `AWS_DEFAULT_REGION`. Mapped into the data loader task with `use_secret_as_env`. |
| `train_data_bucket_name` | `str` | (required) | Bucket containing the training table file. |
| `train_data_file_key` | `str` | (required) | Object key of the **CSV** file (features plus label column). |
| `test_data_secret_name` | `Optional[str]` | `None` | Optional Kubernetes **Secret** for an **external holdout / test table**. Same key convention as `train_data_secret_name`. When set with `test_data_bucket_name` and `test_data_file_key`, the pipeline uses this file for evaluation instead of splitting test data from the training file. |
| `test_data_bucket_name` | `Optional[str]` | `None` | Bucket containing the external test table. Required together with the other `test_data_*` parameters when external test data is used. |
| `test_data_file_key` | `Optional[str]` | `None` | Object key of the external test **CSV** (same schema expectations as training: features plus label column). |
| `label_column` | `str` | (required) | Name of the target / label column. |
| `task_type` | `str` | (required) | One of `binary`, `multiclass`, or `regression`. Drives metrics and AutoGluon problem type. |
| `top_n` | `int` | `3` | How many top models to keep after selection and **refit on the full train data**. Upstream documents a valid range **[1, 10]**. |
| `preset` | `str` | `speed` | Pipeline-level quality tier. The pipeline maps this to an underlying AutoGluon preset before calling **`fit(..., presets=...)`**. Use only values listed under [Preset support](#preset-support). |
| `eval_metric` | `str` | **`accuracy`** if `task_type` is `binary` or `multiclass`; **`r2`** if `task_type` is `regression` | Passed to AutoGluon as **`eval_metric`**. Omitted / `None` resolves to those defaults from **`problem_type`**. Other common string metrics include `roc_auc`, `f1`, `log_loss`, `balanced_accuracy`, `root_mean_squared_error`, `mean_absolute_error`, etc., subject to AutoGluon’s validity rules for the chosen task. |

---

## Autogluon timeseries training pipeline

These parameters are the public surface of `autogluon_timeseries_training_pipeline` in [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/automl/autogluon_timeseries_training_pipeline/pipeline.py).

AutoGluon’s **[`TimeSeriesPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.fit.html)** supports **`presets`** and **`eval_metric`** with **different allowed values** than tabular (not interchangeable). Confirm names in **`pipeline.py`**. **`preset`:** only [Preset support](#preset-support) time-series rows are in scope for typical CPU-only step sizing.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `train_data_secret_name` | `str` | (required) | Kubernetes **Secret** for S3 access. Same key convention as tabular (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`, `AWS_DEFAULT_REGION`). |
| `train_data_bucket_name` | `str` | (required) | Bucket containing the time series file. |
| `train_data_file_key` | `str` | (required) | Object key of the dataset (**CSV or Parquet**). Rows must support building a `TimeSeriesDataFrame`: identifiers, timestamps, target; optional covariate columns as below. |
| `test_data_secret_name` | `Optional[str]` | `None` | Optional Kubernetes **Secret** for an **external holdout / test series file**. Same key convention as `train_data_secret_name`. When set with `test_data_bucket_name` and `test_data_file_key`, the pipeline uses this file for evaluation instead of a temporal test split from the training file. |
| `test_data_bucket_name` | `Optional[str]` | `None` | Bucket containing the external test series file. Required together with the other `test_data_*` parameters when external test data is used. |
| `test_data_file_key` | `Optional[str]` | `None` | Object key of the external test dataset (**CSV or Parquet**); same column expectations as training (`id_column`, `timestamp_column`, `target`, optional covariates). |
| `target` | `str` | (required) | Column with the **numeric value to forecast** (AutoGluon time series target). |
| `id_column` | `str` | (required) | Column that identifies each series (for example `product_id`). Passed as `id_column` when constructing the time series frame; the internal frame uses `item_id`. |
| `timestamp_column` | `str` | (required) | Column with timestamps for each observation. Passed as `timestamp_column`; internal index level is `timestamp`. |
| `known_covariates_names` | `Optional[List[str]]` | `None` | Optional list of column names **known for the forecast horizon** (for example holidays, promotions). Maps to AutoGluon `known_covariates_names`. Omit if not used. |
| `prediction_length` | `int` | `1` | Forecast horizon length in **time steps** (positive integer). |
| `top_n` | `int` | `3` | Number of top models to carry through selection, **full refit**, and leaderboard. |
| `preset` | `str` | `speed` | Pipeline-level quality tier. The pipeline maps this to an underlying AutoGluon preset before calling **`TimeSeriesPredictor.fit(..., presets=...)`**. Use only values under [Preset support](#preset-support). |
| `eval_metric` | `str` | `MASE` | Passed to **`TimeSeriesPredictor(..., eval_metric=...)`**. AutoGluon’s built-in default is **`WQL`**, but the pipeline defaults to **`MASE`**. Allowed string values are under [Evaluation metrics](#evaluation-metrics) (time series table). |


---

## Preset support

AutoGluon documents many **`presets`** values for [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html) and [`TimeSeriesPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.fit.html). **OpenShift AI AutoML** pipeline tasks in **pipelines-components** are typically sized for **CPU-only** workers. The tables below list presets that **fit that envelope** when datasets are moderate in rows/features/series count.

Each pipeline-level preset also sets the **training-data subsample size** used before model selection: **`speed`** up to **100 MB**, **`balanced`** up to **1 GB**. Larger inputs are reduced to that envelope for the selection stage. Preset also changes the **model-training** task resource requests and the AutoGluon quality tier. Top models are still **refit on the full train portion** (selection + extra splits) when the pipeline’s full-refit path runs.

Resource columns below are Kubernetes **requests** on the AutoGluon models-training task; both tiers share the same **limits** (32 CPU / 64 GiB).

**Same preset string does not imply the same models.** Tabular and time-series presets are defined in separate code paths (for example [`presets_configs.py` (tabular)](https://github.com/autogluon/autogluon/blob/stable/tabular/src/autogluon/tabular/configs/presets_configs.py) vs [`predictor_presets.py` (time series)](https://github.com/autogluon/autogluon/blob/stable/timeseries/src/autogluon/timeseries/configs/predictor_presets.py)); the per-mode tables below summarize only the subset recommended for typical constrained RHOAI steps, aligned with the stable API docs linked in the section intro.

### Tabular (`TabularPredictor.fit`)

| Preset | Training task requests | Data subsample | Role (summary) |
|--------|------------------------|----------------|----------------|
| `speed` | **4** vCPU / **16 GiB** | Up to **100 MB** | Good accuracy/speed trade-off (45-min time limit). Uses a **`light`** hyperparameter portfolio. |
| `balanced` | **8** vCPU / **32 GiB** | Up to **1 GB** | Stronger accuracy at higher resource cost (may run more than 2× longer). Uses a **`zeroshot`** hyperparameter portfolio with a larger model candidate set and longer training. |


### Time series (`TimeSeriesPredictor.fit`)

| Preset | Training task requests | Data subsample | Role (summary) |
|--------|------------------------|----------------|----------------|
| `speed` | **4** vCPU / **16 GiB** | Up to **100 MB** | Fastest training path: simpler statistical and tree/ML models. |
| `balanced` | **8** vCPU / **32 GiB** | Up to **1 GB** | Stronger models at higher resource cost: adds more complex models on top of `speed` models. **Chronos-2** foundation model is supported under `balanced`. |


> **Tabular vs time series:** The pipeline preset names (`speed`, `balanced`) are shared across modes, but each mode maps them to different underlying AutoGluon presets. The model sets and resource requirements therefore differ between tabular and time-series runs even for the same tier name. See [`presets_configs.py` (tabular)](https://github.com/autogluon/autogluon/blob/stable/tabular/src/autogluon/tabular/configs/presets_configs.py) and [`predictor_presets.py` (time series)](https://github.com/autogluon/autogluon/blob/stable/timeseries/src/autogluon/timeseries/configs/predictor_presets.py) for the full AutoGluon preset definitions.

---

## Evaluation metrics

AutoGluon’s **leaderboard and reported scores** are expressed in a **higher-is-better** convention: for **loss / error** metrics, the underlying quantity is usually **minimized** during training, but the value shown in the UI is often **sign-flipped** so that sorting “best first” stays intuitive. See the [time-series metrics tutorial](https://auto.gluon.ai/stable/tutorials/timeseries/forecasting-metrics.html#forecasting-metrics) and [tabular in-depth](https://auto.gluon.ai/stable/tutorials/tabular/tabular-indepth.html) discussions.

The **Direction** column below means: how you improve the **raw** metric when comparing models on held-out data. The **Leaderboard** column states how AutoGluon typically presents that same choice (after any transformation).

### Tabular — binary classification (`task_type` `binary`)

These **`eval_metric`** string names are taken from [`TabularPredictor`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html) (binary classification list). Behavior aligns with **scikit-learn** metrics where applicable ([sklearn.metrics](https://scikit-learn.org/stable/modules/classes.html#module-sklearn.metrics)). Metrics marked "both" also apply to `multiclass`; see the [multiclass table](#tabular--multiclass-classification-task_type-multiclass) for the full multiclass set.

| `eval_metric` | Description | Direction (raw) | Leaderboard |
|---------------|-------------|-----------------|---------------|
| `accuracy` | Fraction of labels predicted correctly. Default when `eval_metric=None`. | Higher is better | Same (higher is better) |
| `balanced_accuracy` | Average recall per class, accounting for class imbalance. | Higher is better | Same |
| `log_loss` | Logarithmic loss (cross-entropy) on predicted probabilities. | **Lower** is better | Higher displayed score is better (sign flipped) |
| `f1` | F1 score for the **positive class** (see **`positive_class`** on `TabularPredictor`). | Higher is better | Same |
| `f1_macro` | F1 averaged **unweighted** across classes. | Higher is better | Same |
| `f1_micro` | F1 computed globally over all TP / FP / FN. | Higher is better | Same |
| `f1_weighted` | F1 averaged **weighted** by support per class. | Higher is better | Same |
| `roc_auc` | Area under the ROC curve for the positive class (binary one-vs-rest, as implemented by AutoGluon / sklearn). | Higher is better | Same |
| `average_precision` | Area under the precision–recall curve (from predicted scores for the positive class). | Higher is better | Same |
| `precision` | Precision for the **positive class** (see **`positive_class`** on `TabularPredictor`). | Higher is better | Same |
| `precision_macro` | Precision, **macro** average. | Higher is better | Same |
| `precision_micro` | Precision, **micro** average. | Higher is better | Same |
| `precision_weighted` | Precision, **weighted** by support. | Higher is better | Same |
| `recall` | Recall for the **positive class** (see **`positive_class`** on `TabularPredictor`). | Higher is better | Same |
| `recall_macro` | Recall, **macro** average. | Higher is better | Same |
| `recall_micro` | Recall, **micro** average. | Higher is better | Same |
| `recall_weighted` | Recall, **weighted** by support. | Higher is better | Same |
| `mcc` | Matthews correlation coefficient (–1 to +1). | Higher is better (toward +1) | Same |
| `pac` / `pac_score` | Probabilistic accuracy score (PAC). `pac_score` is a registered alias for `pac`. | Higher is better | Same |

### Tabular — multiclass classification (`task_type` `multiclass`)

These **`eval_metric`** string names are taken from [`TabularPredictor`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html) (multiclass classification list). Behavior aligns with **scikit-learn** metrics where applicable ([sklearn.metrics](https://scikit-learn.org/stable/modules/classes.html#module-sklearn.metrics)).

| `eval_metric` | Description | Direction (raw) | Leaderboard |
|---------------|-------------|-----------------|---------------|
| `accuracy` | Fraction of labels predicted correctly. Default when `eval_metric=None`. | Higher is better | Same (higher is better) |
| `balanced_accuracy` | Average recall per class, accounting for class imbalance. | Higher is better | Same |
| `log_loss` | Logarithmic loss (cross-entropy) on predicted probabilities. | **Lower** is better | Higher displayed score is better (sign flipped) |
| `f1_macro` | F1 averaged **unweighted** across classes. | Higher is better | Same |
| `f1_micro` | F1 computed globally over all TP / FP / FN. | Higher is better | Same |
| `f1_weighted` | F1 averaged **weighted** by support per class. | Higher is better | Same |
| `roc_auc_ovo` / `roc_auc_ovo_macro` | ROC AUC with **one-vs-one** multiclass strategy, **macro**-averaged across pairs. `roc_auc_ovo_macro` is a registered alias. | Higher is better | Same |
| `roc_auc_ovo_weighted` | One-vs-one ROC AUC, **weighted** by class prevalence. | Higher is better | Same |
| `roc_auc_ovr` / `roc_auc_ovr_macro` | ROC AUC with **one-vs-rest** multiclass strategy, **macro**-averaged across classes. `roc_auc_ovr_macro` is a registered alias. | Higher is better | Same |
| `roc_auc_ovr_micro` | One-vs-rest ROC AUC, **micro**-averaged. | Higher is better | Same |
| `roc_auc_ovr_weighted` | One-vs-rest ROC AUC, **weighted** by class prevalence. | Higher is better | Same |
| `precision_macro` | Precision, **macro** average. | Higher is better | Same |
| `precision_micro` | Precision, **micro** average. | Higher is better | Same |
| `precision_weighted` | Precision, **weighted** by support. | Higher is better | Same |
| `recall_macro` | Recall, **macro** average. | Higher is better | Same |
| `recall_micro` | Recall, **micro** average. | Higher is better | Same |
| `recall_weighted` | Recall, **weighted** by support. | Higher is better | Same |
| `mcc` | Matthews correlation coefficient (–1 to +1). | Higher is better (toward +1) | Same |
| `pac` / `pac_score` | Probabilistic accuracy score (PAC). `pac_score` is a registered alias for `pac`. | Higher is better | Same |

### Tabular — regression (`task_type` `regression`)

From the same [`TabularPredictor`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html) **regression** list:

| `eval_metric` | Description | Direction (raw) | Leaderboard |
|---------------|-------------|-----------------|---------------|
| `root_mean_squared_error` | Square root of mean squared error. Default when `eval_metric=None` for regression. | **Lower** RMSE is better | Higher displayed score is better (sign flipped) |
| `mean_squared_error` | Mean squared error of predictions. | **Lower** MSE is better | Higher displayed score is better (sign flipped) |
| `mean_absolute_error` | Mean absolute error of predictions. | **Lower** MAE is better | Higher displayed score is better (sign flipped) |
| `median_absolute_error` | Median absolute error (robust to outliers). | **Lower** is better | Higher displayed score is better (sign flipped) |
| `mean_absolute_percentage_error` | MAPE: mean absolute percentage error (requires meaningful scale; avoid if targets can be zero). | **Lower** MAPE is better | Higher displayed score is better (sign flipped) |
| `r2` | Coefficient of determination \(R^2\). | Higher \(R^2\) is better | Same (higher is better) |
| `symmetric_mean_absolute_percentage_error` | Symmetric MAPE variant (sMAPE-style behavior as defined in AutoGluon). | **Lower** sMAPE is better | Higher displayed score is better (sign flipped) |

### Time series (`TimeSeriesPredictor`)

These **`eval_metric`** strings are listed on [`TimeSeriesPredictor`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.html) and described in [Forecasting Time Series — Evaluation Metrics](https://auto.gluon.ai/stable/tutorials/timeseries/forecasting-metrics.html#forecasting-metrics). **`SQL`** and **`WQL`** evaluate **quantile** forecasts (for default `quantile_levels`); the rest are **point** metrics evaluated on the **`mean`** forecast column when used as `eval_metric`.

| `eval_metric` | Description | Direction (raw) | Leaderboard |
|---------------|-------------|-----------------|---------------|
| `SQL` | **Scaled quantile loss** — probabilistic metric; scale-normalized quantile loss across the horizon. | **Lower** loss is better | Higher displayed score is better (sign flipped per AutoGluon) |
| `WQL` | **Weighted quantile loss** — default in `TimeSeriesPredictor`; emphasizes distributional forecast accuracy over quantiles. | **Lower** loss is better | Higher displayed score is better (sign flipped per AutoGluon) |
| `MAE` | **Mean absolute error** on point (mean) forecasts. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `MAPE` | **Mean absolute percentage error** on point forecasts. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `MASE` | **Mean absolute scaled error** — error scaled by a seasonal naive baseline (see `eval_metric_seasonal_period` on the predictor). | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `MSE` | **Mean squared error** on point forecasts. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `RMSE` | **Root mean squared error** on point forecasts. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `RMSLE` | **Root mean squared logarithmic error** — RMSE in log space (useful for heavy-tailed positive targets). | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `RMSSE` | **Root mean squared scaled error** — RMSE-style error scaled similarly to MASE. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `SMAPE` | **Symmetric mean absolute percentage error** on point forecasts. | **Lower** error is better | Higher displayed score is better (sign flipped) |
| `WAPE` | **Weighted absolute percentage error** — scale-dependent aggregate error across series. | **Lower** error is better | Higher displayed score is better (sign flipped) |

---

When in doubt, prefer the **README and `pipeline.py` on the exact branch or tag** of pipelines-components you deploy.
