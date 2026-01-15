# AutoML Artifacts

AutoML Pipeline run produces the following artifacts.

## Table of Contents

- [Naming Convention](#naming-convention)
- [Model Artifact](#model-artifact)
  - [Sample artifact](#sample-artifact)
- [AutoML Run Output Artifact](#automl-run-output-artifact)
  - [Sample artifact](#sample-artifact-1)
- [Metrics](#metrics)
  - [Example of artifact](#example-of-artifact)
- [Experiment summary](#experiment-summary)
  - [Example of artifact](#example-of-artifact-1)
  - [Proposed Summary Report Structure](#proposed-summary-report-structure)

## Naming Convention

Artifact names follow these conventions:
- **Model artifacts**: Simple predictor identifiers (e.g., `Predictor1`, `Predictor2`, `Predictor3`)
- **Run-level artifacts**: Use `AutoML_<run_name>_` prefix followed by artifact type suffix:
  - Run Output: `AutoML_<run_name>_output`
  - Experiment Summary: `AutoML_<run_name>_summary`
- **Metrics artifacts**: `AutoML_<run_name>_metrics`

## Model Artifact

Model `dsl.Model` artifact consisting of:
- Model `name` following the template `Predictor1`, `Predictor2`, `Predictor3`, etc.
- `uri` to trained AutoGluon Predictor model directory (tar.gz archive)
- `metadata` describing model configuration, performance metrics, and leaderboard rankings

📝 **Note:** There will be multiple Model artifacts per single AutoML run. Each artifact represents a trained AutoGluon Predictor, which includes all trained models (ensemble) and the best model selected for deployment. The artifacts are numbered sequentially (Predictor1, Predictor2, etc.) based on their creation order or performance ranking.

### Sample artifact

📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "uri": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/Predictor1/model.tar.gz",  
  "name": "Predictor1",  
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
        "model_directory": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/models/",
        "leaderboard": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/leaderboard.csv",
        "model_summary": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/model_summary.json"
      }
    },  
    "metrics": {  
      "test_data": {  
        "roc_auc": 0.9123,
        "accuracy": 0.8765,
        "f1": 0.8543,
        "precision": 0.8891,
        "recall": 0.8321
      },
      "train_data": {  
        "roc_auc": 0.9456,
        "accuracy": 0.9012,
        "f1": 0.8823,
        "precision": 0.9105,
        "recall": 0.8567
      }
    },
    "leaderboard": {
      "top_models": [
        {
          "model_name": "WeightedEnsemble_L3",
          "score_val": 0.9123,
          "score_test": 0.9123,
          "fit_time": 125.45,
          "pred_time": 0.12
        },
        {
          "model_name": "LightGBM_BAG_L2",
          "score_val": 0.9101,
          "score_test": 0.9101,
          "fit_time": 89.23,
          "pred_time": 0.08
        },
        {
          "model_name": "CatBoost_BAG_L2",
          "score_val": 0.9087,
          "score_test": 0.9087,
          "fit_time": 156.78,
          "pred_time": 0.15
        }
      ]
    }
  }  
}
```

## AutoML Run Output Artifact

Run Output `dsl.Artifact` consisting of:
- Run Output `name` `AutoML_<run_name>_output`
- `uri` to log file persisted in result directory
- `metadata` describing the run status

### Sample artifact

📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "name": "AutoML_<run_name>_output",  
  "uri": "/path/run_output.log",  
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

## Metrics

AutoML uses Kubeflow Pipelines Metrics artifacts to visualize and track model performance. Two types of metrics artifacts are used:

1. **`dsl.ClassificationMetrics`** - For classification-specific visualizations (confusion matrix, ROC curve)
2. **`dsl.Metrics`** - For scalar metrics (accuracy, precision, recall, F1, etc.)

### Using ClassificationMetrics for Visual Metrics

The `ClassificationMetrics` artifact enables the Kubeflow Pipelines UI to render interactive visualizations:

#### Confusion Matrix

The confusion matrix is logged using `log_confusion_matrix()` method, which enables the KFP UI to display an interactive confusion matrix heatmap.

**Example code:**
```python
from kfp.dsl import component, Output, ClassificationMetrics
from sklearn.metrics import confusion_matrix

@component
def evaluate_classification_model(
    y_true: list,
    y_pred: list,
    class_labels: list,
    classification_metrics: Output[ClassificationMetrics]
):
    # Compute confusion matrix
    cm = confusion_matrix(y_true, y_pred, labels=class_labels)
    
    # Log confusion matrix for KFP UI visualization
    classification_metrics.log_confusion_matrix(
        categories=class_labels,  # e.g., ['Class 0', 'Class 1'] or ['Negative', 'Positive']
        matrix=cm.tolist()
    )
```

#### ROC Curve

The ROC curve is logged using `log_roc_curve()` method, which enables the KFP UI to display an interactive ROC curve plot.

**Example code:**
```python
from sklearn.metrics import roc_curve

@component
def evaluate_binary_classification(
    y_true: list,
    y_scores: list,  # Predicted probabilities for positive class
    classification_metrics: Output[ClassificationMetrics]
):
    # Compute ROC curve
    fpr, tpr, thresholds = roc_curve(y_true, y_scores)
    
    # Log ROC curve for KFP UI visualization
    classification_metrics.log_roc_curve(
        fpr=fpr.tolist(),
        tpr=tpr.tolist(),
        threshold=thresholds.tolist()
    )
```

### Using Metrics for Scalar Values

The `Metrics` artifact is used to log scalar performance metrics that are displayed in the KFP UI metrics tab.

**Example code:**
```python
from kfp.dsl import component, Output, Metrics
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score

@component
def log_scalar_metrics(
    y_true: list,
    y_pred: list,
    y_scores: list,
    metrics: Output[Metrics]
):
    # Compute scalar metrics
    accuracy = accuracy_score(y_true, y_pred)
    precision = precision_score(y_true, y_pred, average='weighted')
    recall = recall_score(y_true, y_pred, average='weighted')
    f1 = f1_score(y_true, y_pred, average='weighted')
    roc_auc = roc_auc_score(y_true, y_scores)
    
    # Log scalar metrics for KFP UI display
    metrics.log_metric('accuracy', accuracy)
    metrics.log_metric('precision', precision)
    metrics.log_metric('recall', recall)
    metrics.log_metric('f1_score', f1)
    metrics.log_metric('roc_auc', roc_auc)
```

### Example Artifacts

#### ClassificationMetrics Artifact

```json
{  
  "name": "AutoML_<run_name>_classification_metrics",  
  "uri": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/classification_metrics.json",  
  "metadata": {  
    "context": {  
      "model_artifact_name": "Predictor1",  
      "model_artifact_id": "---id---",
      "task_type": "classification",
      "eval_metric": "roc_auc"
    },
    "confusion_matrix": {
      "categories": ["Class 0", "Class 1"],
      "matrix": [[450, 35], [28, 487]]
    },
    "roc_curve": {
      "fpr": [0.0, 0.01, 0.02, 0.03, 0.05, ...],
      "tpr": [0.0, 0.15, 0.28, 0.42, 0.58, ...],
      "thresholds": [1.0, 0.95, 0.90, 0.85, 0.80, ...]
    }
  }  
}
```

#### Metrics Artifact (Scalar Values)

```json
{  
  "name": "AutoML_<run_name>_metrics",  
  "uri": "automl/results/71c46718-8faa-4a0e-a018-073edfdca527/metrics.json",  
  "metadata": {  
    "context": {  
      "model_artifact_name": "Predictor1",  
      "model_artifact_id": "---id---",
      "task_type": "classification",
      "eval_metric": "roc_auc"
    },
    "metrics": {
      "accuracy": 0.8765,
      "precision": 0.8891,
      "recall": 0.8321,
      "f1_score": 0.8543,
      "roc_auc": 0.9123
    },
    "leaderboard_summary": {
      "total_models": 15,
      "top_3_avg_score": 0.9104,
      "ensemble_improvement": 0.0022
    }
  }  
}
```

### Supported Features by Task Type

#### Classification Tasks

**ClassificationMetrics supports:**
- ✅ Confusion Matrix (binary and multiclass)
- ✅ ROC Curve (binary classification)
- ✅ Precision-Recall Curve (via custom logging)

**Metrics supports:**
- ✅ Accuracy
- ✅ Precision (macro, micro, weighted)
- ✅ Recall (macro, micro, weighted)
- ✅ F1 Score (macro, micro, weighted)
- ✅ ROC-AUC (binary)
- ✅ Log Loss
- ✅ Custom metrics

#### Regression Tasks

**Metrics supports:**
- ✅ R² Score
- ✅ Root Mean Squared Error (RMSE)
- ✅ Mean Absolute Error (MAE)
- ✅ Mean Squared Error (MSE)
- ✅ Mean Absolute Percentage Error (MAPE)
- ✅ Custom metrics

#### Time-Series Forecasting Tasks

**Metrics supports:**
- ✅ MAPE (Mean Absolute Percentage Error)
- ✅ sMAPE (Symmetric MAPE)
- ✅ MASE (Mean Absolute Scaled Error)
- ✅ RMSE (Root Mean Squared Error)
- ✅ MAE (Mean Absolute Error)
- ✅ WQL (Weighted Quantile Loss)
- ✅ Custom metrics

### Best Practices

1. **Use ClassificationMetrics for Visual Metrics**: Always use `ClassificationMetrics` for confusion matrix and ROC curve to enable UI visualization
2. **Use Metrics for Scalars**: Use `Metrics` for all scalar values that should appear in the metrics tab
3. **Log Per-Model Metrics**: Create separate metrics artifacts for each predictor (Predictor1, Predictor2, etc.) to enable comparison
4. **Include Context**: Always include context metadata linking metrics to the corresponding model artifact
5. **Consistent Naming**: Use consistent metric names across runs for easy comparison in the KFP UI

## Experiment summary

The `dsl.Markdown` Artifact describing the experiment details. This is a comprehensive report that provides an overview of the AutoML optimization run, including data preparation, model building, selection process, and results.

### Example of artifact

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

Based on AutoGluon's approach and best practices, the experiment summary Markdown report should include the following sections:

```markdown
# AutoML Experiment Summary

**Experiment Name:** AutoML Classification Experiment  
**Run ID:** 71c46718-8faa-4a0e-a018-073edfdca527  
**Created At:** 2025-12-11T11:42:40.380Z  
**Status:** Completed  
**Duration:** 2m 25s

---

## Executive Summary

This experiment trained **15** models using AutoGluon and selected the best performing ensemble model. The optimization focused on maximizing **roc_auc** metric. The best performing model achieved a ROC-AUC score of **0.9123** with accuracy of **0.8765**.

**Best Model:** WeightedEnsemble_L3  
**Best Metric Score:** 0.9123 (ROC-AUC)  
**Total Models Trained:** 15  
**Models Promoted to Refit:** 3  
**Final Models Evaluated:** 3

---

## Experiment Configuration

### Input Data Sources
- **Training Data:**
  - Connection: `s3-data-connection`
  - Bucket: `my-ml-data-bucket`
  - Path: `tabular_data/train.csv`
  - Total Rows: 10,000
  - Total Features: 25

- **Test Data:** (Optional)
  - Connection: `s3-data-connection`
  - Bucket: `my-ml-data-bucket`
  - Path: `tabular_data/test.csv`
  - Total Rows: 2,000

### Infrastructure
- **Results Storage:** `s3://results/automl/71c46718-8faa-4a0e-a018-073edfdca527/`
- **Model Registry:** RHOAI Model Registry (registered: Yes)
- **Model Serving:** KServe (deployed: No)

### Model Configuration
- **Task Type:** Classification
- **Label Column:** target
- **Preset:** best_quality
- **Evaluation Metric:** roc_auc
- **Time Limit:** 3600 seconds

### Data Preparation
- **Sampling Method:** Stratified
- **Sample Size:** 500 (for initial model building)
- **Train/Test Split:** 0.2 (test size)
- **Random State:** 42

---

## Data Preparation

### Data Statistics
- **Training Set Size:** 8,000 rows
- **Test Set Size:** 2,000 rows
- **Features:** 25
  - Numeric: 18
  - Categorical: 7
- **Target Distribution:**
  - Class 0: 4,200 (52.5%)
  - Class 1: 3,800 (47.5%)

### Data Quality
- **Missing Values:** 0.5% (handled automatically by AutoGluon)
- **Duplicate Rows:** 0
- **Outliers:** Detected and handled during preprocessing

---

## Model Building Process

### Phase 1: Initial Model Building (Sampled Data)
- **Sample Size:** 500 rows
- **Models Trained:** 12
- **Duration:** 45 seconds
- **Top Models Selected:** 3

### Phase 2: Model Refit (Full Training Data)
- **Training Set Size:** 8,000 rows
- **Models Refit:** 3
- **Duration:** 98 seconds

### Phase 3: Leaderboard Evaluation
- **Models Evaluated:** 3 (refit models) + 12 (initial models)
- **Evaluation Metric:** ROC-AUC
- **Duration:** 2.5 seconds

---

## Leaderboard

### Top Models

| Rank | Model Name | ROC-AUC | Accuracy | F1 Score | Precision | Recall | Fit Time | Pred Time |
|------|------------|---------|----------|----------|-----------|--------|----------|-----------|
| 🥇 1 | WeightedEnsemble_L3 | **0.9123** | 0.8765 | 0.8543 | 0.8891 | 0.8321 | 125.45s | 0.12s |
| 🥈 2 | LightGBM_BAG_L2 | 0.9101 | 0.8742 | 0.8512 | 0.8865 | 0.8289 | 89.23s | 0.08s |
| 🥉 3 | CatBoost_BAG_L2 | 0.9087 | 0.8721 | 0.8489 | 0.8843 | 0.8256 | 156.78s | 0.15s |
| 4 | XGBoost_BAG_L2 | 0.9056 | 0.8698 | 0.8456 | 0.8812 | 0.8223 | 112.34s | 0.10s |
| 5 | NeuralNetFastAI_BAG_L2 | 0.9034 | 0.8675 | 0.8423 | 0.8789 | 0.8190 | 203.45s | 0.18s |

### Performance Metrics Summary

**ROC-AUC:**
- Mean: 0.8987
- Std Dev: 0.0045
- Best: 0.9123 (WeightedEnsemble_L3)
- Worst: 0.8892 (RandomForest_BAG_L1)

**Accuracy:**
- Mean: 0.8643
- Std Dev: 0.0034
- Best: 0.8765 (WeightedEnsemble_L3)
- Worst: 0.8567 (RandomForest_BAG_L1)

**F1 Score:**
- Mean: 0.8421
- Std Dev: 0.0032
- Best: 0.8543 (WeightedEnsemble_L3)
- Worst: 0.8323 (RandomForest_BAG_L1)

---

## Best Model Details

### WeightedEnsemble_L3 (Best Performing)

**Model Type:** Weighted Ensemble (Stacking)
**Base Models:**
- LightGBM_BAG_L2 (weight: 0.45)
- CatBoost_BAG_L2 (weight: 0.35)
- XGBoost_BAG_L2 (weight: 0.20)

**Performance Metrics:**
- ROC-AUC: 0.9123 (Test: 0.9123, Train: 0.9456)
- Accuracy: 0.8765 (Test: 0.8765, Train: 0.9012)
- F1 Score: 0.8543
- Precision: 0.8891
- Recall: 0.8321

**Training Details:**
- Fit Time: 125.45 seconds
- Prediction Time: 0.12 seconds (per 1000 samples)
- Model Size: 45.2 MB

**Confusion Matrix:**
```
                Predicted
              Class 0  Class 1
Actual Class 0   450      35
Actual Class 1    28     487
```

**Feature Importances (Top 10):**
1. feature_1: 0.234
2. feature_2: 0.189
3. feature_3: 0.156
4. feature_4: 0.123
5. feature_5: 0.098
6. feature_6: 0.076
7. feature_7: 0.054
8. feature_8: 0.043
9. feature_9: 0.032
10. feature_10: 0.025

**Artifacts:**
- Model Directory: `automl/results/.../models/WeightedEnsemble_L3/`
- Model Archive: `automl/results/.../model.tar.gz`
- Leaderboard: `automl/results/.../leaderboard.csv`

---

## Model Insights

### Key Findings
1. **Ensembling:** Weighted ensemble outperformed individual models by 0.2-0.3% in ROC-AUC
2. **Tree-based Models:** LightGBM, CatBoost, and XGBoost were the top individual performers
3. **Feature Engineering:** AutoGluon automatically handled missing values and categorical encoding
4. **Training Efficiency:** Bagging (BAG_L2) improved model robustness with minimal time overhead

### Model Family Performance
- **Tree-based Models:** Best individual performance (LightGBM: 0.9101)
- **Neural Networks:** Good performance but slower training (NeuralNetFastAI: 0.9034)
- **Linear Models:** Baseline performance (0.8892-0.8956)

### Configuration Impact
- **Preset (best_quality):** Enabled full model exploration and ensembling
- **Time Limit (3600s):** Sufficient for complete model training
- **Stratified Sampling:** Maintained class balance during initial exploration

---

## Artifacts and Resources

### Generated Artifacts
- **Model Artifact:** Complete AutoGluon Predictor with all trained models
- **Run Output Log:** `automl/results/.../run_output.log`
- **Metrics Artifact:** Evaluation metrics and leaderboard
- **Experiment Summary:** This report

### Links
- [View All Artifacts](./artifacts/)
- [Download Model](./model.tar.gz)
- [View Leaderboard](./leaderboard.csv)
- [View Run Log](./run_output.log)

---

## Next Steps

### Recommended Actions
1. **Deploy Best Model** (WeightedEnsemble_L3) for production use
2. **Model Registry:** Model has been registered with metadata
3. **Further Optimization:** Consider exploring:
   - Different presets (high_quality for faster inference)
   - Feature engineering improvements
   - Additional data collection for underrepresented classes
4. **Evaluation:** Test model on additional validation datasets
5. **Monitoring:** Track performance metrics in production

---

## Technical Details

### Runtime Information
- **AutoGluon Version:** 1.0.0
- **Pipeline Execution Time:** 2m 25s
- **Total Models Trained:** 15
- **Average Model Training Time:** 8.3s

### Model Training Breakdown
- **Initial Model Building:** 45s (12 models on 500 samples)
- **Model Refit:** 98s (3 models on 8,000 samples)
- **Leaderboard Evaluation:** 2.5s (15 models)
- **Ensemble Creation:** 0.5s

### Resource Usage
- **Peak Memory:** 2.3 GB
- **CPU Cores Used:** 4
- **GPU:** Not used

---

*Report generated by AutoML on 2025-12-11T11:42:40.380Z*
```

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
