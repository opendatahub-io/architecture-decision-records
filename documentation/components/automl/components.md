# AutoML Kubeflow Pipelines Components Structure

> ⚠️ **Warning:** This document is a work in progress and represents a proposed structure. The final implementation may vary based on actual development requirements, KFP Components repository standards, and community feedback. This is not a final version.

This document outlines the proposed structure for AutoML components following the [Kubeflow Pipelines Components repository](https://github.com/kubeflow/pipelines-components) standards.

## Repository Structure

Based on the KFP Components repository structure, AutoML components would be organized as follows:

```
kubeflow/pipelines-components/
├── components/
│   └── automl/
│       ├── data-processing/
│       │   ├── data-loader/
│       │   │   ├── component.py          # Component implementation
│       │   │   ├── metadata.yaml         # Component metadata
│       │   │   ├── README.md              # Component documentation
│       │   │   ├── OWNERS                 # Maintainer information
│       │   │   ├── tests/
│       │   │   │   └── test_component.py  # Unit tests
│       │   │   └── example_pipelines.py  # Usage examples
│       │   │
│       │   └── train-test-split/
│       │       ├── component.py
│       │       ├── metadata.yaml
│       │       ├── README.md
│       │       ├── OWNERS
│       │       ├── tests/
│       │       └── example_pipelines.py
│       │
│       ├── model-training/
│       │   ├── model-building-selection/
│       │   │   ├── component.py
│       │   │   ├── metadata.yaml
│       │   │   ├── README.md
│       │   │   ├── OWNERS
│       │   │   ├── tests/
│       │   │   └── example_pipelines.py
│       │   │
│       │   └── model-refit/
│       │       ├── component.py
│       │       ├── metadata.yaml
│       │       ├── README.md
│       │       ├── OWNERS
│       │       ├── tests/
│       │       └── example_pipelines.py
│       │
│       ├── model-evaluation/
│       │   └── leaderboard-evaluation/
│       │       ├── component.py
│       │       ├── metadata.yaml
│       │       ├── README.md
│       │       ├── OWNERS
│       │       ├── tests/
│       │       └── example_pipelines.py
│       │
│       └── model-deployment/
│           ├── model-registry/
│           │   ├── component.py
│           │   ├── metadata.yaml
│           │   ├── README.md
│           │   ├── OWNERS
│           │   ├── tests/
│           │   └── example_pipelines.py
│           │
│           └── kserve-deployment/
│               ├── component.py
│               ├── metadata.yaml
│               ├── README.md
│               ├── OWNERS
│               ├── tests/
│               └── example_pipelines.py
│
└── pipelines/
    └── automl/
        ├── automl-classification-regression-pipeline/
        │   ├── pipeline.py               # Classification & Regression pipeline
        │   ├── metadata.yaml
        │   ├── README.md
        │   ├── OWNERS
        │   ├── tests/
        │   └── example_usage.py
        │
        └── automl-time-series-pipeline/
            ├── pipeline.py                # Time-Series pipeline
            ├── metadata.yaml
            ├── README.md
            ├── OWNERS
            ├── tests/
            └── example_usage.py
```

## Component Categories

### Data Processing Components

Components responsible for data ingestion, preparation, and transformation:

1. **data-loader** - Loads tabular data from S3 or local filesystem
   - Inputs: `input_data_reference` (Dict), `test_data_reference` (Dict, optional)
   - Outputs: `tabular_data` (Artifact), `test_data` (Artifact, optional)
   - Dependencies: RHOAI Connections API, pandas, pyarrow
   - **Note**: Supports CSV, Parquet, XLSX formats. Torch compatible data loader to be explored. Data cache support to be explored in next stage.

2. **train-test-split** - Splits data into train/test sets and performs sampling
   - Inputs: `tabular_data` (Artifact), `sampling_config` (Dict, optional), `split_config` (Dict, optional), `task_type` (str), `label_column` (str)
   - Outputs: `train_data` (Artifact), `test_data` (Artifact), `sampled_train_data` (Artifact)
   - Dependencies: pandas, scikit-learn
   - **Note**: Supports random, stratified, and time-series sampling methods. Samples subset for initial model building (default: 500 samples).

### Model Training Components

Components responsible for model building, selection, and training:

1. **model-building-selection** - Builds multiple models using sampled data and selects top performers
   - Inputs: `sampled_train_data` (Artifact), `task_type` (str), `label_column` (str), `selection_config` (Dict, optional)
   - Outputs: `candidate_models` (Artifacts), `model_leaderboard` (Artifact)
   - Dependencies: AutoGluon library
   - **Note**: Uses AutoGluon's ensembling approach (stacking/bagging) rather than traditional hyperparameter optimization. Promotes top N models to refit stage.

2. **model-refit** - Refits best candidate models on full training dataset
   - Inputs: `train_data` (Artifact), `candidate_models` (Artifacts), `selection_config` (Dict, optional)
   - Outputs: `refit_models` (Artifacts)
   - Dependencies: AutoGluon library, Kubeflow Katib (to be explored for distributed training)
   - **Note**: Produces fully trained models ready for evaluation. Explores Kubeflow Katib for distributed computing and experiment/trials logging.

### Model Evaluation Components

Components responsible for model evaluation and metrics generation:

1. **leaderboard-evaluation** - Evaluates models and generates leaderboard with metrics
   - Inputs: `refit_models` (Artifacts), `test_data` (Artifact), `task_type` (str), `eval_metric` (str, optional)
   - Outputs: `model_artifacts` (Model Artifacts), `metrics` (ClassificationMetrics/Metrics Artifacts, optional), `leaderboard` (Artifact)
   - Dependencies: AutoGluon library, scikit-learn
   - **Note**: Generates comprehensive performance metrics, confusion matrix, ROC curve (for classification), and feature importances. Creates multiple Model artifacts (Predictor1, Predictor2, etc.).

### Model Deployment Components

Components responsible for model registration and serving (optional):

1. **model-registry** - Registers best model with Model Registry
   - Inputs: `best_model` (Model Artifact), `model_metadata` (Dict), `auto_register` (bool)
   - Outputs: `registered_model` (Artifact)
   - Dependencies: RHOAI Model Registry API
   - **Note**: Optional step controlled by `auto_register` parameter. Registers AutoGluon Predictor with best model metadata for deployment purposes.

2. **kserve-deployment** - Deploys model using KServe with AutoGluon runtime
   - Inputs: `model` (Model Artifact), `auto_deploy` (bool), `deployment_config` (Dict, optional)
   - Outputs: `inference_service` (Artifact)
   - Dependencies: KServe, AutoGluon runtime (custom)
   - **Note**: Optional step controlled by `auto_deploy` parameter. Uses custom AutoGluon runtime. Contribution to KServe with new runtime to be considered.

## Component Metadata Structure

Each component should include a `metadata.yaml` file with the following structure:

```yaml
name: data-loader
version: 1.0.0
description: Loads tabular data from S3 or local filesystem for AutoML processing
stability: alpha  # alpha, beta, or stable
category: data-processing
dependencies:
  kubeflow:
    min_version: "2.0.0"
  external:
    - name: RHOAI Connections API
      version: ">=1.0.0"
    - name: AutoGluon
      version: ">=1.0.0"
lastVerified: "2025-01-15"
maintainers:
  - name: AutoML Team
    email: automl-team@example.com
tags:
  - automl
  - data-processing
  - tabular-data
```

## Pipeline Structure

### Classification & Regression Pipeline

The complete Classification & Regression pipeline would compose these components:

```python
from kfp import dsl
from kfp_components.components.automl.data_processing import (
    data_loader,
    train_test_split
)
from kfp_components.components.automl.model_training import (
    model_building_selection,
    model_refit
)
from kfp_components.components.automl.model_evaluation import (
    leaderboard_evaluation
)
from kfp_components.components.automl.model_deployment import (
    model_registry,
    kserve_deployment
)

@dsl.pipeline(
    name='automl-classification-regression-pipeline',
    description='AutoML pipeline for classification and regression tasks'
)
def automl_classification_regression_pipeline(
    name: str,
    input_data_reference: Dict,
    results_reference: Dict,
    task_type: str,
    label_column: str,
    test_data_reference: Dict = None,
    sampling_config: Dict = None,
    split_config: Dict = None,
    selection_config: Dict = None,
    auto_register: bool = False,
    auto_deploy: bool = False
):
    # Data Processing Steps
    data = data_loader(
        input_data_reference=input_data_reference,
        test_data_reference=test_data_reference
    )
    
    # Split and sample data
    split_result = train_test_split(
        tabular_data=data.outputs['tabular_data'],
        task_type=task_type,
        label_column=label_column,
        sampling_config=sampling_config,
        split_config=split_config
    )
    
    # Model Training Steps
    # Build models on sampled data and select top performers
    candidates = model_building_selection(
        sampled_train_data=split_result.outputs['sampled_train_data'],
        task_type=task_type,
        label_column=label_column,
        selection_config=selection_config
    )
    
    # Refit best models on full training data
    refit_models = model_refit(
        train_data=split_result.outputs['train_data'],
        candidate_models=candidates.outputs['candidate_models'],
        selection_config=selection_config
    )
    
    # Model Evaluation Step
    evaluation = leaderboard_evaluation(
        refit_models=refit_models.outputs['refit_models'],
        test_data=split_result.outputs['test_data'],
        task_type=task_type,
        eval_metric=selection_config.get('eval_metric') if selection_config else None
    )
    
    # Optional Deployment Steps
    with dsl.Condition(auto_register == True):
        registry = model_registry(
            best_model=evaluation.outputs['best_model'],
            model_metadata={
                'task_type': task_type,
                'label_column': label_column,
                'experiment_name': name
            }
        )
    
    with dsl.Condition(auto_deploy == True):
        deployment = kserve_deployment(
            model=evaluation.outputs['best_model']
        )
```

### Time-Series Pipeline

The complete Time-Series pipeline would compose these components:

```python
@dsl.pipeline(
    name='automl-time-series-pipeline',
    description='AutoML pipeline for time-series forecasting tasks'
)
def automl_time_series_pipeline(
    name: str,
    input_data_reference: Dict,
    results_reference: Dict,
    timestamp_column: str,
    target: str,
    test_data_reference: Dict = None,
    prediction_length: int = None,
    sampling_config: Dict = None,
    split_config: Dict = None,
    time_series_config: Dict = None,
    selection_config: Dict = None,
    auto_register: bool = False,
    auto_deploy: bool = False
):
    # Data Processing Steps
    data = data_loader(
        input_data_reference=input_data_reference,
        test_data_reference=test_data_reference
    )
    
    # Split and sample data (time-series specific)
    split_result = train_test_split(
        tabular_data=data.outputs['tabular_data'],
        task_type="time_series",
        label_column=target,
        sampling_config=sampling_config,
        split_config=split_config
    )
    
    # Model Training Steps
    candidates = model_building_selection(
        sampled_train_data=split_result.outputs['sampled_train_data'],
        task_type="time_series",
        label_column=target,
        selection_config=selection_config,
        time_series_config={
            'timestamp_column': timestamp_column,
            'prediction_length': prediction_length,
            **(time_series_config or {})
        }
    )
    
    refit_models = model_refit(
        train_data=split_result.outputs['train_data'],
        candidate_models=candidates.outputs['candidate_models'],
        selection_config=selection_config,
        time_series_config={
            'timestamp_column': timestamp_column,
            'prediction_length': prediction_length,
            **(time_series_config or {})
        }
    )
    
    # Model Evaluation Step
    evaluation = leaderboard_evaluation(
        refit_models=refit_models.outputs['refit_models'],
        test_data=split_result.outputs['test_data'],
        task_type="time_series",
        eval_metric=selection_config.get('eval_metric') if selection_config else None
    )
    
    # Optional Deployment Steps
    with dsl.Condition(auto_register == True):
        registry = model_registry(
            best_model=evaluation.outputs['best_model'],
            model_metadata={
                'task_type': 'time_series',
                'target': target,
                'timestamp_column': timestamp_column,
                'experiment_name': name
            }
        )
    
    with dsl.Condition(auto_deploy == True):
        deployment = kserve_deployment(
            model=evaluation.outputs['best_model']
        )
```

## Quality Standards

Following KFP Components repository standards, each component must:

- ✅ Pass linting and formatting checks (Black, pydocstyle)
- ✅ Include comprehensive docstrings
- ✅ Compile successfully with `kfp.compiler`
- ✅ Include metadata with fresh `lastVerified` date
- ✅ Pass automated CI/CD checks
- ✅ Include unit tests with >80% coverage
- ✅ Provide usage examples in `example_pipelines.py`

## Maintenance

- **Verification**: Components flagged when `lastVerified` is older than 9 months
- **Ownership**: Each component has designated owners in `OWNERS` file
- **Removal Process**: Components not verified within 12 months are proposed for removal

## References

- [Kubeflow Pipelines Components Repository](https://github.com/kubeflow/pipelines-components)
- [KFP Component Specification](https://www.kubeflow.org/docs/components/pipelines/reference/component-spec/)
- [Creating KFP Components](https://www.kubeflow.org/docs/components/pipelines/sdk/v2/component-development/)
- [AutoGluon Documentation](https://github.com/autogluon/autogluon)
