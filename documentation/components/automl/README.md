# AutoML

AutoML in Red Hat OpenShift AI automates tabular machine learning with **Kubeflow Pipelines (Data Science Pipelines)**. Training logic is delivered as versioned **pipelines** and **reusable components** from the **pipelines-components** repository, orchestrating **AutoGluon** for model search, ensembling, refit, and leaderboard reporting.

For the formal architecture decision (goals, alternatives, roadmap), see the AutoML ADR: [ODH-ADR-0001-automl](../../../architecture-decision-records/automl/ODH-ADR-0001-automl.md).

## Source repository

Implementation assets live under:

- **Product fork:** [red-hat-data-services/pipelines-components](https://github.com/red-hat-data-services/pipelines-components) (fork of [opendatahub-io/pipelines-components](https://github.com/opendatahub-io/pipelines-components))

AutoML-specific paths:

| Area | Path in repository |
|------|---------------------|
| End-to-end pipelines | [`pipelines/training/automl/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/automl) |
| Training components | [`components/training/automl/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl) |
| Data processing components | [`components/data_processing/automl/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/automl) |
| CI / image build (example) | [`Dockerfile.konflux.automl`](https://github.com/red-hat-data-services/pipelines-components/blob/main/Dockerfile.konflux.automl), Tekton under [`.tekton/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/.tekton) |

Upstream README for the overall repo layout (components vs pipelines, categories, contribution expectations): [pipelines-components README](https://github.com/red-hat-data-services/pipelines-components/blob/main/README.md).

## Pipelines

The **Automl** pipeline group ships two managed training pipelines:

| Pipeline | Purpose | Entry (Python package layout) |
|----------|---------|-------------------------------|
| **Autogluon Tabular Training** | Binary / multiclass classification and regression on tabular CSV from S3 | [`pipelines/training/automl/autogluon_tabular_training_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/automl/autogluon_tabular_training_pipeline) |
| **Autogluon Timeseries Training** | Time series forecasting with per-series temporal splits | [`pipelines/training/automl/autogluon_timeseries_training_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/automl/autogluon_timeseries_training_pipeline) |

Declared **dependencies** in pipeline metadata include **Kubeflow Pipelines >= 2.15.2** and **Kubernetes >= 1.28.0**. See each pipeline’s `metadata.yaml` and `README.md` for `lastVerified`, owners, and tags.

## Reusable components

### Data processing (`components/data_processing/automl`)

| Component | Role |
|-----------|------|
| [tabular_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/automl/tabular_data_loader) | Load tabular CSV from S3-compatible storage, sample (up to ~1 GB), stratified or random splits, write workspace paths and test artifact for downstream steps |
| [timeseries_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/automl/timeseries_data_loader) | Load time series data from S3, temporal per-series splits, workspace layout for selection vs extra train data |

### Training (`components/training/automl`)

| Component | Role |
|-----------|------|
| [autogluon_models_training](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_models_training) | Tabular: train with AutoGluon ensembling (e.g. stacking + bagging), pick top **N**, sequential **refit_full** on full train data; emit combined **models** artifact |
| [autogluon_leaderboard_evaluation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_leaderboard_evaluation) | Tabular: build HTML leaderboard from refitted models’ metrics |
| [autogluon_timeseries_models_selection](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_models_selection) | Time series: train and rank models; emit top **N** for refit |
| [autogluon_timeseries_models_full_refit](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_models_full_refit) | Time series: refit each top model on full train portion; **ParallelFor** parallelism is constrained (often **1**) because the pipeline workspace PVC is **ReadWriteOnce** |
| [autogluon_timeseries_leaderboard_evaluation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_leaderboard_evaluation) | Time series: HTML leaderboard from refit metrics |

Shared helpers (for example leaderboard HTML utilities) live under [`components/training/automl/shared/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/shared).

## Runtime and storage model

- **Workspace PVC:** Pipelines configure a **Kubernetes workspace** (`PipelineConfig.workspace`) so intermediate train splits and large files stay on a shared PVC instead of being round-tripped only as remote artifacts. Tabular pipeline code documents a default workspace size (see `pipeline.py` in the tabular pipeline; treat as tunable for your cluster class).
- **S3 credentials:** Data load steps use **`use_secret_as_env`** (KFP Kubernetes integration) to map a namespace **Secret** (for example the keys expected for S3-compatible access: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_ENDPOINT`, `AWS_DEFAULT_REGION`) into the loader task. In OpenShift AI this aligns with **Connections** materialized as Secrets for pipeline workloads.
- **Artifacts:** The **test** split and final **model / leaderboard** outputs are stored under the cluster’s **artifact store** (S3-compatible backend configured for Data Science Pipelines). Exact key layout (`<pipeline_name>/<run_id>/...`) is specified in each pipeline README under **Files stored in user storage**.

## Pipeline parameters and configuration

For complete documentation of pipeline inputs, presets, evaluation metrics, and defaults for both tabular and timeseries pipelines, see:

- **[Experiment settings (pipeline parameters)](./features/experiment_settings.md)** — Comprehensive documentation of all KFP pipeline inputs, parameter types, defaults, allowed values, presets, and evaluation metrics for both tabular and timeseries training pipelines.

For details on what artifacts and metrics are produced by each pipeline:

- **[Model insights (artifacts and metrics)](./features/model_insights.md)** — Complete artifact directory structure, file formats, and metric outputs after training for both tabular and timeseries pipelines.

## Experiment tracking

For MLflow integration with AutoML workflows:

- **[MLflow integration](./features/mlflow_integration.md)** — How experiment tracking relates to Data Science Pipelines and AutoML runs, including KFP native integration (RHOAI 3.5+), data loader metrics, nested run hierarchy, and MLflow UI visualization examples.

## Platform context

AutoML runs **on** the same Data Science Pipelines platform described in [Data Science Pipelines](../pipelines/README.md): DSPA stacks, Argo-based execution, and KFP v2-style pipeline definitions.

## Keeping this page accurate

When behavior, parameters, or artifact layouts change, update this document from the canonical **README.md** / **pipeline.py** / **metadata.yaml** files in **pipelines-components** for the release you document.
