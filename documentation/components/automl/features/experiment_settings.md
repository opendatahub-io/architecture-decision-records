# Experiment settings (pipeline parameters)

This page documents the **top-level input parameters** for the two AutoML training pipelines shipped in [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) under `pipelines/training/automl/`.
Higher-level architecture and ADR-level parameter groups are in [ODH-ADR-0001-automl](../../../../architecture-decision-records/automl/ODH-ADR-0001-automl.md). Component layout is summarized in the [AutoML component README](../README.md).

## Table of contents

- [Autogluon tabular training pipeline](#autogluon-tabular-training-pipeline)
  - [Current shipping parameters](#current-shipping-parameters)
  - [OpenShift AI 3.5 and later](#openshift-ai-35-and-later)
- [Autogluon timeseries training pipeline](#autogluon-timeseries-training-pipeline)
  - [Current shipping parameters](#current-shipping-parameters-1)
  - [OpenShift AI 3.5 and later](#openshift-ai-35-and-later-1)
- [Preset support](#preset-support)
- [Evaluation metrics](#evaluation-metrics)
  - [Tabular — classification (`task_type` `binary` or `multiclass`)](#tabular--classification-task_type-binary-or-multiclass)
  - [Tabular — regression (`task_type` `regression`)](#tabular--regression-task_type-regression)
  - [Time series (`TimeSeriesPredictor`)](#time-series-timeseriespredictor)

---

## Autogluon tabular training pipeline

These parameters are the public surface of `autogluon_tabular_training_pipeline` in [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/automl/autogluon_tabular_training_pipeline/pipeline.py).

### Current shipping parameters

Arguments exposed today on the tabular training pipeline (verify on your **pipelines-components** tag or branch):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `train_data_secret_name` | `str` | (required) | Kubernetes **Secret** name holding S3-compatible credentials. Expected keys: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`, `AWS_DEFAULT_REGION`. Mapped into the data loader task with `use_secret_as_env`. |
| `train_data_bucket_name` | `str` | (required) | Bucket containing the training table file. |
| `train_data_file_key` | `str` | (required) | Object key of the **CSV** file (features plus label column). |
| `label_column` | `str` | (required) | Name of the target / label column. |
| `task_type` | `str` | (required) | One of `binary`, `multiclass`, or `regression`. Drives metrics and AutoGluon problem type. |
| `top_n` | `int` | `3` | How many top models to keep after selection and **refit on the full train data**. Upstream documents a valid range **[1, 10]**. |

### OpenShift AI 3.5 and later

Additional KFP pipeline parameters planned for the tabular graph (the same two names are intended for the **timeseries** pipeline as well; see below). Confirm names and optionality in **`pipeline.py`** for your build. Semantics and defaults follow **[AutoGluon Tabular](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html)** and **`presets` / `eval_metric`** on [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html). **`preset`:** use only values in [Preset support](#preset-support). **`eval_metric`:** allowed tabular strings are under [Evaluation metrics](#evaluation-metrics) (tabular tables).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preset` | `str` | `medium_quality` | Passed through to AutoGluon **`fit(..., presets=...)`** as a string preset. Use only values listed under [Preset support](#preset-support) for typical **CPU-only** pipeline steps sized around **~16 GiB RAM / 8 vCPU**; full AutoGluon catalog and definitions remain in [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html). |
| `eval_metric` | `str` | **`accuracy`** if `task_type` is `binary` or `multiclass`; **`r2`** if `task_type` is `regression` | Passed to AutoGluon as **`eval_metric`**. Omitted / `None` resolves to those defaults from **`problem_type`** (see **`eval_metric`** on [`TabularPredictor`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html)). Other common string metrics include `roc_auc`, `f1`, `log_loss`, `balanced_accuracy`, `root_mean_squared_error`, `mean_absolute_error`, etc., subject to AutoGluon’s validity rules for the chosen task. |


---

## Autogluon timeseries training pipeline

These parameters are the public surface of `autogluon_timeseries_training_pipeline` in [`pipeline.py`](https://github.com/red-hat-data-services/pipelines-components/blob/main/pipelines/training/automl/autogluon_timeseries_training_pipeline/pipeline.py).

AutoGluon’s **[`TimeSeriesPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.fit.html)** supports **`presets`** and the predictor supports **`eval_metric`** the same way as tabular, but with **different allowed values** (time-series presets and metrics are not interchangeable with tabular). For **OpenShift AI 3.5+**, the same pipeline-level names **`preset`** (mapped to `fit(..., presets=...)`) and **`eval_metric`** are intended to apply here as well; confirm in **`pipeline.py`**. **`preset`:** only [Preset support](#preset-support) time-series rows are in scope for typical CPU-only step sizing.

### Current shipping parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `train_data_secret_name` | `str` | (required) | Kubernetes **Secret** for S3 access. Same key convention as tabular (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`, `AWS_DEFAULT_REGION`). |
| `train_data_bucket_name` | `str` | (required) | Bucket containing the time series file. |
| `train_data_file_key` | `str` | (required) | Object key of the dataset (**CSV or Parquet**). Rows must support building a `TimeSeriesDataFrame`: identifiers, timestamps, target; optional covariate columns as below. |
| `target` | `str` | (required) | Column with the **numeric value to forecast** (AutoGluon time series target). |
| `id_column` | `str` | (required) | Column that identifies each series (for example `product_id`). Passed as `id_column` when constructing the time series frame; the internal frame uses `item_id`. |
| `timestamp_column` | `str` | (required) | Column with timestamps for each observation. Passed as `timestamp_column`; internal index level is `timestamp`. |
| `known_covariates_names` | `Optional[List[str]]` | `None` | Optional list of column names **known for the forecast horizon** (for example holidays, promotions). Maps to AutoGluon `known_covariates_names`. Omit if not used. |
| `prediction_length` | `int` | `1` | Forecast horizon length in **time steps** (positive integer). |
| `top_n` | `int` | `3` | Number of top models to carry through selection, **full refit**, and leaderboard. |

### OpenShift AI 3.5 and later

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preset` | `str` | `fast_training` | Passed to AutoGluon **`TimeSeriesPredictor.fit(..., presets=...)`**. Time-series names differ from tabular; use only values under [Preset support](#preset-support). Full list and semantics: [`TimeSeriesPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.fit.html). If `presets` is not set in `fit`, AutoGluon uses **`hyperparameters`** defaults. |
| `eval_metric` | `str` | `MASE` | Passed to **`TimeSeriesPredictor(..., eval_metric=...)`**. AutoGluon’s built-in default is **`WQL`** (weighted quantile loss), but the pipeline defaults to **`MASE`** (mean absolute scaled error). Allowed string values are the time-series metrics in [Evaluation metrics](#evaluation-metrics) (time series table). |


---

## Preset support

AutoGluon documents many **`presets`** values for [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html) and [`TimeSeriesPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.timeseries.TimeSeriesPredictor.fit.html). **OpenShift AI AutoML** pipeline tasks in **pipelines-components** are typically sized for **CPU-only** workers. The tables below list presets that **fit that envelope** when datasets are moderate in rows/features/series count.

**Same preset string does not imply the same models.** Tabular and time-series presets are defined in separate code paths (for example [`presets_configs.py` (tabular)](https://github.com/autogluon/autogluon/blob/stable/tabular/src/autogluon/tabular/configs/presets_configs.py) vs [`predictor_presets.py` (time series)](https://github.com/autogluon/autogluon/blob/stable/timeseries/src/autogluon/timeseries/configs/predictor_presets.py)); the per-mode tables below summarize only the subset recommended for typical constrained RHOAI steps, aligned with the stable API docs linked in the section intro.

### Tabular (`TabularPredictor.fit`)

| Preset | Min resources (model training step) | Role (summary) |
|--------|--------------------------------------|----------------|
| `medium_quality` | 4 vCPU / 16 GiB RAM | AutoGluon default tier: moderate accuracy, **fast** training and inference; **`auto_stack`: False** in preset definition (see [TabularPredictor.fit](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html)). Recommended default for constrained steps. The pipeline overrides `max_depth=10` for **RandomForest (RF)** and **ExtraTrees (XT)** models as an optimisation measure. |
| `good_quality` | 8 vCPU / 32 GiB RAM | Stronger accuracy than `medium_quality` using the **`light`** hyperparameter portfolio with refit. The pipeline overwrites the following preset-level flags with fixed values — `refit_full=False`, `set_best_to_refit_full=False`, `save_bag_folds=True` — because a dedicated refit step is performed afterwards (these preset defaults would otherwise conflict with that step). |

> **Note:** AutoGluon also defines **`optimize_for_deployment`** as a preset that shrinks **disk and inference** footprint after training. It is not a “quality tier” on its own; pass it **together with** a quality preset as a **grouped** `presets` argument to `fit` (for example `['good_quality', 'optimize_for_deployment']`). See [`TabularPredictor.fit`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.fit.html).

### Time series (`TimeSeriesPredictor.fit`)

| Preset | Min resources (model training step) | Role (summary) |
|--------|--------------------------------------|----------------|
| `fast_training` | 4 vCPU / 16 GiB RAM | **Fastest** path: simpler statistical and tree/ML models per AutoGluon docs. |
| `medium_quality` | 8 vCPU / 32 GiB RAM | Adds **TemporalFusionTransformer** and **Chronos-2 (small)** on top of `fast_training` models; the intended “balanced” tier for CPU-only if data volume fits in memory. The pipeline **excludes all Chronos-type models** from training under this preset to reduce resource consumption. |


> **Tabular vs time series:** Preset **strings** are not aligned across modes. Tabular **`good_quality`** and time-series **`medium_quality`** are the closest pair by AutoGluon wiring: both use the **`light`** hyperparameter portfolio in [`presets_configs.py` (tabular)](https://github.com/autogluon/autogluon/blob/stable/tabular/src/autogluon/tabular/configs/presets_configs.py) and [`predictor_presets.py` (time series)](https://github.com/autogluon/autogluon/blob/stable/timeseries/src/autogluon/timeseries/configs/predictor_presets.py), respectively.

---

## Evaluation metrics

AutoGluon’s **leaderboard and reported scores** are expressed in a **higher-is-better** convention: for **loss / error** metrics, the underlying quantity is usually **minimized** during training, but the value shown in the UI is often **sign-flipped** so that sorting “best first” stays intuitive. See the [time-series metrics tutorial](https://auto.gluon.ai/stable/tutorials/timeseries/forecasting-metrics.html#forecasting-metrics) and [tabular in-depth](https://auto.gluon.ai/stable/tutorials/tabular/tabular-indepth.html) discussions.

The **Direction** column below means: how you improve the **raw** metric when comparing models on held-out data. The **Leaderboard** column states how AutoGluon typically presents that same choice (after any transformation).

### Tabular — classification (`task_type` `binary` or `multiclass`)

These **`eval_metric`** string names are taken from [`TabularPredictor`](https://auto.gluon.ai/stable/api/autogluon.tabular.TabularPredictor.html) (classification list). Each must be valid for your **`task_type`**; behavior aligns with **scikit-learn** metrics where applicable ([sklearn.metrics](https://scikit-learn.org/stable/modules/classes.html#module-sklearn.metrics)).

| `eval_metric` | Description | Direction (raw) | Leaderboard |
|---------------|-------------|-----------------|---------------|
| `accuracy` | Fraction of labels predicted correctly. Default when `eval_metric=None` for binary and multiclass. | Higher is better | Same (higher is better) |
| `balanced_accuracy` | Balanced accuracy: average recall per class, accounting for class imbalance. | Higher is better | Same |
| `log_loss` | Logarithmic loss (cross-entropy) on predicted probabilities. | **Lower** log-loss is better | Higher displayed score is better (sign flipped) |
| `f1` | F1 score for the **positive class** in binary problems (see **`positive_class`** on `TabularPredictor`); for multiclass, uses default averaging. | Higher is better | Same |
| `f1_macro` | F1 averaged **unweighted** across classes. | Higher is better | Same |
| `f1_micro` | F1 computed globally over all TP / FP / FN. | Higher is better | Same |
| `f1_weighted` | F1 averaged **weighted** by support per class. | Higher is better | Same |
| `roc_auc` | Area under the ROC curve (binary one-vs-rest style as implemented by AutoGluon / sklearn). | Higher is better | Same |
| `roc_auc_ovo` | ROC AUC with **one-vs-one** multiclass strategy. | Higher is better | Same |
| `roc_auc_ovo_macro` | One-vs-one ROC AUC, **macro**-averaged. | Higher is better | Same |
| `roc_auc_ovo_weighted` | One-vs-one ROC AUC, **weighted** by support. | Higher is better | Same |
| `roc_auc_ovr` | ROC AUC with **one-vs-rest** multiclass strategy. | Higher is better | Same |
| `roc_auc_ovr_macro` | One-vs-rest ROC AUC, **macro**-averaged. | Higher is better | Same |
| `roc_auc_ovr_micro` | One-vs-rest ROC AUC, **micro**-averaged. | Higher is better | Same |
| `roc_auc_ovr_weighted` | One-vs-rest ROC AUC, **weighted** by support. | Higher is better | Same |
| `average_precision` | Area under the precision–recall curve (from predicted scores). | Higher is better | Same |
| `precision` | Precision for the positive class (binary) or default averaging (multiclass). | Higher is better | Same |
| `precision_macro` | Precision, **macro** average. | Higher is better | Same |
| `precision_micro` | Precision, **micro** average. | Higher is better | Same |
| `precision_weighted` | Precision, **weighted** by support. | Higher is better | Same |
| `recall` | Recall for the positive class (binary) or default averaging (multiclass). | Higher is better | Same |
| `recall_macro` | Recall, **macro** average. | Higher is better | Same |
| `recall_micro` | Recall, **micro** average. | Higher is better | Same |
| `recall_weighted` | Recall, **weighted** by support. | Higher is better | Same |
| `mcc` | Matthews correlation coefficient (–1 to +1). | Higher is better (toward +1) | Same |
| `pac_score` | Probabilistic accuracy (PAC) score from AutoGluon’s metric definitions. | Higher is better | Same |

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

When in doubt, prefer the **README and `pipeline.py` on the exact branch or tag** of pipelines-components that ships with your OpenShift AI version.
