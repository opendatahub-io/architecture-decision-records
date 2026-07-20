# AutoML

Tabular ML training on **Data Science Pipelines**, implemented in [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) with **AutoGluon**.

**Architecture:** [ODH-ADR-0001-automl](../../../architecture-decision-records/automl/ODH-ADR-0001-automl.md) — goals, workflow, Model Registry registration, KServe / AutoGluon ServingRuntime deploy, scope.

**Feature docs:** [experiment settings](./features/experiment_settings.md) · [model insights](./features/model_insights.md) · [MLflow integration](./features/mlflow_integration.md)


## Pipelines

| Pipeline | Repository entry |
|----------|------------------|
| Autogluon Tabular Training | [`autogluon_tabular_training_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/automl/autogluon_tabular_training_pipeline) |
| Autogluon Timeseries Training | [`autogluon_timeseries_training_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/automl/autogluon_timeseries_training_pipeline) |

Pipeline parameters, presets, and metrics → [experiment settings](./features/experiment_settings.md). Artifacts and outputs → [model insights](./features/model_insights.md). Per-pipeline `README.md` / `metadata.yaml` in **pipelines-components** are canonical for dependencies, workspace PVC, and storage layout.

## Components

**Data processing:** [tabular_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/automl/tabular_data_loader) · [timeseries_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/automl/timeseries_data_loader)

**Training:** [autogluon_models_training](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_models_training) · [autogluon_leaderboard_evaluation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_leaderboard_evaluation) · [autogluon_timeseries_models_selection](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_models_selection) · [autogluon_timeseries_models_full_refit](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_models_full_refit) · [autogluon_timeseries_leaderboard_evaluation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/autogluon_timeseries_leaderboard_evaluation) · [shared](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/automl/shared)

## Platform

Runs on [Data Science Pipelines](../pipelines/README.md). Update this page when **pipelines-components** layout changes.
