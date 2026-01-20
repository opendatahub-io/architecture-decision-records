# AutoML Artifacts

AutoML Pipeline run produces the following artifacts.

## Table of Contents

- [Naming Convention](#naming-convention)
- [Model Artifact(s)](#model-artifact)
  - [Sample artifact](#sample-artifact)
- [AutoML Run Output Artifact](#automl-run-output-artifact)
  - [Sample artifact](#sample-artifact-1)
- [Leaderboard](#leaderboard)
  - [Sample artifact](#sample-artifact-2)
- [Experiment summary](#experiment-summary)
  - [Example of artifact](#example-of-artifact-1)
  - [Proposed Summary Report Structure](#proposed-summary-report-structure)

## Naming Convention

<a id="naming-convention"></a>

Artifact names follow these conventions:
- **Model artifacts**: Simple predictor identifiers based on models names (e.g., `WeightedEnsemble_L3`, `CatBoost_BAG_L2`)
- **Run-level artifacts**: Use `automl_` prefix followed by artifact type suffix:
  - Run Output: `automl_run_output`
  - Experiment Summary: `automl_run_summary`
  - Leaderboard: `automl_leaderboard`


## Model Artifact(s)

<a id="model-artifact"></a>

Model `dsl.Model` artifact consisting of:
- Model `name` following the template `WeightedEnsemble_L3`, `CatBoost_BAG_L2`, etc.
- `uri` to trained AutoGluon Predictor model directory (tar.gz archive)
- `metadata` describing model configuration, performance metrics, and leaderboard rankings

📝 **Note:** There will be multiple Model artifacts per single AutoML run. Each artifact represents a trained AutoGluon Model (part of the Predictor). The final predictor includes all trained models and uses the best model for prediction.

📝 **Note:** The models and predictor binaries are stored under specified location at the end of experiment run. The intermediate models cannot be used before run completion.
The limitation can be removed based on feedback and RFEs opened.

### Sample artifact

<a id="sample-artifact"></a>

📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "name": "CatBoost_BAG_L2",  
  "uri": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/autogluon.tar.gz",
  "metadata": {  
    "context": {  
      "task_type": "classification",
      "label_column": "target",
      "runtime_ver": {  
        "name": "autogluon_1.0.0"  
      },
      "model_config": {
        "preset": "best_quality",
        "eval_metric": "roc_auc",
        "time_limit": 3600
      },
      "data_config": {
        "sampling_config": {
          "n_samples": 500,
          "sampling_method": "stratified"
        },
        "split_config": {
          "test_size": 0.2
        }
      },
      "location": {  
        "model_directory": "autogluon/models/CatBoost_BAG_L2",
        "predictor": "autogluon/predictor.pkl"
      }
    },  
    "metrics": {  
      "val_data": {  
        "roc_auc": 0.9456,
        "accuracy": 0.9012,
        "f1": 0.8823,
        "precision": 0.9105,
        "recall": 0.8567
      },
      "test_data": {  
        "roc_auc": 0.9123,
        "accuracy": 0.8765,
        "f1": 0.8543,
        "precision": 0.8891,
        "recall": 0.8321
      }
    }
  }  
}
```

📝 **Note:** confusion matrix, ROC curve data to be added to `metrics` based on the feedback.

## AutoML Run Output Artifact

<a id="automl-run-output-artifact"></a>

Run Output `dsl.Artifact` consisting of:
- Run Output `name` `automl_output`
- `uri` to log file persisted in result directory
- `metadata` describing the run status

### Sample artifact

<a id="sample-artifact-1"></a>

📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "name": "automl_output",  
  "uri": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/run_output.log",  
  "metadata": {  
    "status": {  
      "completed_at": "2025-12-11T11:42:40.380Z",  
      "message": {  
        "level": "info",  
        "text": "AutoML execution completed successfully."  
      },  
      "running_at": "2025-12-11T11:40:15.000Z",  
      "state": "completed",  
      "step": "leaderboard_evaluation"  
    },  
    "execution": {
      "duration_seconds": 145.5,
      "total_models_trained": 15,
      "models_promoted_to_refit": 3,
      "final_models_evaluated": 3
    }
  }  
}
```

## Leaderboard

<a id="leaderboard"></a>

Leaderboard `dsl.HTML` consisting of:
- Leaderboard `name` `automl_leaderboard`
- `uri` to html file persisted in result directory
- `metadata` contains serialized leaderboard data

### Sample artifact

<a id="sample-artifact-2"></a>

📝 **Note:** this is just a draft version - changes can be made

```json
{
   "name":"automl_leaderboard",
   "uri":"automl/results/71c46718-8faa-4a0e-a018-073edfdca527/leaderboard.html",
   "metadata":{
      "model":{
         "0":"LightGBMXT_FULL",
         "1":"CatBoost",
         "2":"RandomForestGini",
         "3":"RandomForestEntr",
         "4":"XGBoost"
      },
      "score_test":{
         "0":0.8426655748,
         "1":0.8424608455,
         "2":0.8424608455,
         "3":0.8409253762,
         "4":0.83990173
      },
      "score_val":{
         "0":null,
         "1":0.85,
         "2":0.84,
         "3":0.83,
         "4":0.85
      },
      "eval_metric":{
         "0":"accuracy",
         "1":"accuracy",
         "2":"accuracy",
         "3":"accuracy",
         "4":"accuracy"
      },
      "pred_time_test":{
         "0":0.0056340694,
         "1":0.0067238808,
         "2":0.0481259823,
         "3":0.0558469296,
         "4":0.0198101997
      },
      "pred_time_val":{
         "0":null,
         "1":0.0015211105,
         "2":0.0292029381,
         "3":0.0137529373,
         "4":0.0025968552
      },
      "fit_time":{
         "0":0.2220532894,
         "1":1.2829272747,
         "2":0.6016447544,
         "3":0.2239229679,
         "4":0.6792140007
      },
      "fit_order":{
         "0":12,
         "1":5,
         "2":3,
         "3":4,
         "4":9
      }
   }
}
```

## Experiment summary

<a id="experiment-summary"></a>

The `dsl.Markdown` Artifact describing the experiment details. This is a comprehensive report that provides an overview of the AutoML optimization run, including data preparation, model building, selection process, and results.

### Example of artifact

<a id="example-of-artifact-1"></a>

```json
{  
  "name": "AutoML_<run_name>_summary",  
  "uri": "path/summary_report.md",  
  "metadata": {  
    "experiment_name": "AutoML Classification Experiment",
    "run_id": "71c46718-8faa-4a0e-a018-073edfdca527",
    "created_at": "2025-12-11T11:42:40.380Z",
    "task_type": "classification",
    "total_models_trained": 15,
    "optimization_metric": "roc_auc"
  }  
}
```

### Proposed Summary Report Structure

<a id="proposed-summary-report-structure"></a>

Based on AutoGluon's approach and best practices, the experiment summary Markdown report should include the following sections:

See [Sample Experiment Summary](experiment-summary-sample.md) for a complete example.

### Summary Report Sections

The proposed summary report includes:

1. **Executive Summary** - High-level overview with key metrics and best model
2. **Experiment Configuration** - Input data sources, infrastructure, and model settings
3. **Data Preparation** - Data statistics, quality checks, and preprocessing details
4. **Model Building Process** - Phases of model training (sampling, refit, evaluation)
5. **Leaderboard** - Ranked list of all models with performance metrics
6. **Best Model Details** - Comprehensive information about the top-performing model
7. **Model Insights** - Key findings, model family performance, and configuration impact
8. **Artifacts and Resources** - Links to generated artifacts
9. **Next Steps** - Recommended actions and further optimization suggestions
10. **Technical Details** - Runtime information, training breakdown, and resource usage

This structure provides a comprehensive view of the AutoML experiment, making it easy to understand what models were trained, which performed best, and how to proceed with deployment.
