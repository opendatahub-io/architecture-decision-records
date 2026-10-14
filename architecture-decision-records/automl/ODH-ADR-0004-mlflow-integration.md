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

Integrate MLflow experiment tracking into AutoML pipelines and reusable components.

## Why

MLflow provides the supported RHOAI experience for comparing AutoGluon models, inspecting their configuration and metrics, and navigating from a pipeline run to its tracked results.

## Goals

* Track enabled AutoML runs as a KFP parent run with nested model-level child runs.
* Log configuration, aggregate metrics, model metrics, and references to pipeline artifacts.
* Make the parent run discoverable by Dashboard without changing the AutoML artifact contract.

## Non-goals

* AutoGluon-native MLflow autologging.
* MLflow Model Registry integration.
* Defining the platform KFP-to-MLflow configuration schema or MLflow artifact layout.

## How

## Enablement

MLflow tracking is enabled only when the pipeline platform injects valid MLflow configuration into the run. It is best-effort: invalid or unavailable configuration must not prevent model training. AutoML does not add a public pipeline parameter to enable or disable MLflow.

## Run mapping

The tabular and time-series pipelines use the same hierarchy.

| MLflow concept | AutoML contract |
| --- | --- |
| Experiment | The KFP-managed experiment for the pipeline execution. |
| Parent run | One KFP-associated run for the AutoML execution. It records pipeline identity, non-secret training configuration, and aggregate metrics. |
| Child run | One nested run for each refitted model with a complete `model.json`, predictor, metrics, and predictor-notebook artifact set. The model name is the join key between the child run and these artifacts, enabling side-by-side comparison. |
| Parent metrics | Best, worst, and mean score; number of trained models; and overall fit time when available. |
| Child metrics | Task-appropriate metrics from the leaderboard and model metrics artifacts. |
| Child artifacts | References to the model predictor, metrics, and notebook pipeline artifacts. Pipeline artifacts remain authoritative. |

Non-secret parameters may include task type, evaluation metric, preset, `top_n`, and dataset identifiers or hashes. Credentials and other secret material must never be logged.

## Dashboard discovery

The existing component-stage map remains the Dashboard join artifact. When tracking is active, it exposes the resolved MLflow experiment and parent-run reference, including the information needed to link to the MLflow UI. The MLflow tracking endpoint is for in-cluster service communication only; any browser link must use the browser-accessible MLflow UI base from the injected configuration (for example, `mlflowBaseURL`), not the tracking endpoint. The exact configuration parsing and stage-map representation are implementation details of the pipeline integration.

When tracking is active, the top-level `mlflow` object in `component_stage_map.json` contains:

| Field | Purpose |
| --- | --- |
| `tracking_uri` | In-cluster MLflow tracking endpoint used by pipeline components; not a browser URL. |
| `experiment_id` | Resolved MLflow experiment for this pipeline execution. |
| `run_id` | Resolved parent MLflow run for this pipeline execution. |
| `workspace` | OpenShift AI project or namespace when MLflow workspaces are enabled. |
| `run_url` | Browser-accessible link to the parent run, derived from the injected MLflow UI base (for example, `mlflowBaseURL`), not `tracking_uri`. |

## Artifact and model boundaries

MLflow records references to pipeline artifacts; it does not replace the predictor, metrics, or notebook contracts in [ODH-ADR-0003](./ODH-ADR-0003-model-insights.md). The integration does not require an MLflow model flavor or duplicate the model artifact layout.

## Alternatives

KFP's generic automatic MLflow logging cannot represent the required model-level hierarchy and AutoGluon artifacts. A custom tracking system would duplicate the supported MLflow UI and platform integration. Explicit AutoML integration with the parent/child mapping above is therefore selected.

## Risks

The integration has a platform-configuration dependency and requires explicit logging because AutoGluon has no native MLflow autologging module. Best-effort enablement and references to authoritative pipeline artifacts limit the impact of either concern.

## References

* [AutoML architecture](./ODH-ADR-0001-automl.md)
* [Model insights and artifact contract](./ODH-ADR-0003-model-insights.md)
* [AutoML components](https://github.com/opendatahub-io/pipelines-components/tree/main/components/training/automl)
* [AutoML pipelines](https://github.com/opendatahub-io/pipelines-components/tree/main/pipelines/training/automl)
* [MLflow Tracking](https://mlflow.org/docs/latest/tracking.html)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Nelesh Singla                 | 27 May     |       |
