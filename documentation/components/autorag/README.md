# AutoRAG

Document RAG optimization on **Data Science Pipelines**, implemented in [pipelines-components](https://github.com/red-hat-data-services/pipelines-components) with **ai4rag**.

**Architecture:** [ODH-ADR-0001-autorag](../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md) — goals, lifecycle (optimize → index → infer), artifacts, scope.

**Feature docs:** [experiment settings](./features/experiment_settings.md) · [pattern inference](./features/rag_pattern_inference.md) · [pattern evaluation](./features/rag_pattern_evaluation.md)

## pipelines-components layout

| Area | Path |
|------|------|
| Optimization pipeline | [`pipelines/training/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag) |
| Indexing pipeline | [`pipelines/data_processing/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag) |
| Training components | [`components/training/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag) |
| Data processing components | [`components/data_processing/autorag/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag) |

## Pipelines

| Pipeline | Repository entry |
|----------|------------------|
| Documents RAG Optimization | [`documents_rag_optimization_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) |
| Documents Indexing | [`documents_indexing_pipeline/`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline) |

Pipeline parameters, presets, and search space → [experiment settings](./features/experiment_settings.md). Per-pipeline `README.md` / `metadata.yaml` in **pipelines-components** are canonical for dependencies and storage layout.

## Components

**Data processing:** [test_data_loader](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/test_data_loader) · [documents_discovery](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/documents_discovery) · [text_extraction](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/text_extraction) · [documents_indexing](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/data_processing/autorag/documents_indexing)

**Training:** [search_space_preparation](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/search_space_preparation) · [rag_templates_optimization](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/rag_templates_optimization) · [component_stage_map_publisher](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/component_stage_map_publisher) · [shared](https://github.com/red-hat-data-services/pipelines-components/tree/main/components/training/autorag/shared)

## Platform

Runs on [Data Science Pipelines](../pipelines/README.md). Update this page when **pipelines-components** or [ai4rag](https://github.com/IBM/ai4rag) layout changes.
