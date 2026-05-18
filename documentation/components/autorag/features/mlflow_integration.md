# AutoRAG and MLflow (Documents RAG optimization pipeline)

This page describes how **MLflow** relates to **AutoRAG** when you run the **Documents RAG optimization** Kubeflow pipeline from [pipelines-components](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) on **OpenShift AI Data Science Pipelines**.

**RHOAI 3.5 scope:** Core MLflow integration with KFP environment variables, parent/child run hierarchy, and pattern artifacts (pattern.json, evaluation results, notebooks).

**RHOAI 3.6+ scope:** Native **MLflow Tracing** for RAG operations (retrieval, generation, evaluation) and **MLflow Evals** integration for automated quality metrics. The **ai4rag** optimization engine accepts an initialized MLflow client from the pipeline component to log traces during pattern exploration.

**Alignment with AutoML:** AutoRAG uses the **same KFP + MLflow integration model** as [AutoML MLflow integration](../../automl/features/mlflow_integration.md): components treat **`MLFLOW_TRACKING_URI`** as the opt-in switch; **one KFP-managed parent run** (`MLFLOW_RUN_ID`) is resumed for pipeline-level tags, params, and leaderboard artifacts; **nested child runs** capture each comparable unit (for AutoRAG, each **RAG pattern** directory—analogous to each **leaderboard model** in AutoML). There is no `mlflow.autorag` autologger—only **explicit** `mlflow.log_*` calls, for the same reasons AutoML avoids relying on framework autologging under Kubeflow.

## Table of Contents

- [KFP MLflow integration modes (OpenShift AI 3.5+)](#kfp-mlflow-integration-modes-openshift-ai-35)
  - [Environment variables (RHOAI 3.5+)](#environment-variables-rhoai-35)
- [KFP artifacts produced by the pipeline (RHOAI 3.5+)](#kfp-artifacts-produced-by-the-pipeline-rhoai-35)
- [MLflow mapping model (AutoRAG)](#mlflow-mapping-model-autorag)
- [AutoRAG artifact → MLflow logging method](#autorag-artifact-mlflow-logging-method)
- [MLflow Tracing for RAG Operations (RHOAI 3.6+)](#mlflow-tracing-for-rag-operations-rhoai-36)
  - [Trace hierarchy for RAG patterns](#trace-hierarchy-for-rag-patterns)
  - [ai4rag integration with MLflow client](#ai4rag-integration-with-mlflow-client)
  - [Implementation example](#implementation-example)
- [MLflow Evals Integration (RHOAI 3.6+)](#mlflow-evals-integration-rhoai-36)
  - [RAG evaluation metrics](#rag-evaluation-metrics)
  - [Integration with pattern evaluation](#integration-with-pattern-evaluation)
- [KFP Pipeline Integration: MLflow Tracking Artifact](#kfp-pipeline-integration-mlflow-tracking-artifact)
  - [Artifact purpose](#artifact-purpose)
  - [Component output definition](#component-output-definition)
  - [Artifact schema](#artifact-schema)
- [Related](#related)

---

## KFP MLflow integration modes (OpenShift AI 3.5+)

Starting with **OpenShift AI 3.5**, Data Science Pipelines provides **opt-in with environment variables** (recommended for AutoRAG): MLflow client environment variables are pre-configured in pipeline step pods, allowing custom logging with full control over complex artifacts and metrics.

AutoRAG uses this approach because RAG optimization produces complex artifacts (pattern configurations, evaluation results, notebooks, Responses API templates) that require explicit logging control beyond what KFP automatic logging can capture.

**AutoRAG integration approach:** Components check whether **`MLFLOW_TRACKING_URI`** is set. If yes, they **resume the KFP parent run** and create **nested child runs** (one per RAG pattern), using explicit `mlflow.log_params()` / `mlflow.log_metrics()` / `mlflow.set_tags()` to log pattern metadata and pointers to KFP artifacts—the same lifecycle pattern as AutoML’s **one child run per leaderboard row / refitted model**.

### Environment variables (RHOAI 3.5+)

When MLflow integration is enabled at the project level, OpenShift AI injects the same variables into pipeline step pods as for AutoML (see [AutoML — Environment variables](../../automl/features/mlflow_integration.md#environment-variables-rhoai-35)):

| Environment Variable | Purpose | Example Value | Notes |
|---------------------|---------|---------------|-------|
| `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint | `https://mlflow-server.example.com` | If unset, optional AutoRAG MLflow logging is skipped; pipeline and KFP artifacts are unchanged |
| `MLFLOW_WORKSPACE` | Workspace identifier for the project | `data-science-project` | OpenShift AI project / namespace |
| `MLFLOW_EXPERIMENT_ID` | Experiment for this pipeline | `"1"` | KFP-managed experiment |
| `MLFLOW_RUN_ID` | Parent run for this pipeline execution | `"abc123..."` | Components resume this run and create **nested** pattern child runs under it |
| `MLFLOW_TRACKING_AUTH` | Authentication mechanism | `kubernetes-namespaced` | Same Kubernetes-aware auth behavior as AutoML (MLflow SDK expectations apply) |

---

## KFP artifacts produced by the pipeline (RHOAI 3.5+)

These paths represent the **RHOAI 3.5+** artifact structure (see upstream `pipeline.py` and `rag_templates_optimization` / `leaderboard_evaluation` sources). They are the **source of truth** for what MLflow logging should attach to.

| Artifact / path role | Producing step | Layout and role |
|---------------------|----------------|-----------------|
| **Test data** | `test_data_loader` | Benchmark JSON on disk (input to search prep + optimization). |
| **Discovered documents** | `documents_discovery` | Descriptor of corpus objects for extraction. |
| **Extracted text** | `text_extraction` | Extracted document text (e.g. via **docling**) for ai4rag. |
| **`search_space_prep_report`** | `search_space_preparation` | YAML-serialized **search space** after phase-one validation. |
| **`rag_patterns`** (directory artifact) | `rag_templates_optimization` | One **subdirectory per pattern** (pattern name = folder name). Under each: **`pattern.json`** (with embedded Responses API template), **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, **`create_model_response.py`**. |
| **HTML leaderboard** | `leaderboard_evaluation` | Single HTML file built from each subdir’s **`pattern.json`**. |

**Per-pattern artifacts (RHOAI 3.5+):**

| Artifact | Type | Purpose |
|----------|------|---------|
| **`pattern.json`** | JSON | **Consolidated pattern record** containing: chunking, embedding, retrieval, generation settings; **Responses API request template** (`input`, `tools`, `instructions`, `metadata`) embedded in `settings.responses_template`; vector store binding; aggregate scores and timing. Single source of truth for registration, deployment, and code generation. |
| **`indexing_notebook.ipynb`** | Jupyter Notebook | Indexing notebook instantiated from templates (e.g., `ls_indexing_template.ipynb`), parameterized for this pattern |
| **`inference_notebook.ipynb`** | Jupyter Notebook | Inference notebook instantiated from templates (e.g., `ls_inference_template.ipynb`), parameterized for this pattern |
| **`evaluation_results.json`** | JSON | Per-question evaluation metrics (context precision, answer relevance, faithfulness), traces, metadata |
| **`create_model_response.py`** | Python | Interactive client script for testing patterns (embeds Llama Stack base URL from env, reads Responses config from `pattern.json`) |

---

## MLflow mapping model (AutoRAG)

Design follows the same concepts as the AutoML doc: **one parent run per KFP pipeline execution**, optional **child runs per RAG pattern**, explicit APIs (there is **no** `mlflow.autorag` autologger).

| MLflow concept | Proposed mapping                                                                                                                                                                                                                                                                           |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Experiment** | Use KFP-managed experiment (`MLFLOW_EXPERIMENT_ID`). **Otherwise:** e.g. `autorag_documents_rag_optimization` plus optional suffix (team, cluster).                                                                                                                                        |
| **Parent run** | Resume KFP parent (`MLFLOW_RUN_ID`). **Tags:** `kfp_run_id`, `kfp_run_name`, `pipeline_name`, `dataset hashes or URIs (non-secret)`. **Params:** `optimization_metric`, `optimization_max_rag_patterns`, `llama_stack_vector_io_provider_id`, `image`, `kfp_version`, `ai4rag_version`     |
| **Child runs** | **One nested child run per RAG pattern** (folder name or `pattern.json` `name`). Enables side-by-side comparison in the MLflow UI for faithfulness / answer_correctness / context_correctness and chunking / retrieval / model choices.                                                    |
| **Params (child)** | From **`pattern.json`** → `settings`: e.g. `chunking.*`, `embedding.model_id`, `retrieval.*`, `generation.model_id`, `vector_store_binding`** fields (`provider_id`, `provider_type`, `vector_store_id`, `vector_store_name`) and key **`responses_template`** fields as flattened params. |
| **Metrics (child)** | From **`pattern.json`**: `final_score`, `duration_seconds`, `iteration`; from **`scores`**: per-metric means (`faithfulness`, `answer_correctness`, `context_correctness`) — use **`mlflow.log_metric`** for scalars.                                                                      |
| **Child Artifacts** | Pointers to KFP artifacts: **`pattern.json`**, **`evaluation_results.json`**, **`indexing_notebook.ipynb`**, **`inference_notebook.ipynb`**, **`create_model_response.py`**. Logged as params containing artifact URIs/paths, not copied into MLflow. |

---

## AutoRAG artifact → MLflow logging method

Use **explicit** MLflow Tracking APIs (`mlflow.log_param`, `mlflow.log_params`, `mlflow.log_metric`, `mlflow.log_metrics`, `mlflow.set_tags`). Lazy-import **mlflow** when `MLFLOW_TRACKING_URI` is unset so pipeline steps behave the same with tracking disabled.

**Artifact strategy:** Log **pointers/URIs** to KFP-produced artifacts as params, not the actual files. This avoids duplicating large artifacts and keeps MLflow runs lightweight while maintaining traceability to the authoritative KFP artifact store.

| Source (KFP / disk) | MLflow method | What to record | Typical run scope |
|---------------------|---------------|----------------|-------------------|
| KFP pipeline **inputs** (`optimization_metric`, `optimization_max_rag_patterns`, `input_data_key`, `test_data_key`, vector I/O provider id) | `log_params` / `set_tags` | Optimization objective and non-secret location hints | Parent |
| **`search_space_prep_report`** (YAML) | `log_param` | URI/path to YAML artifact in KFP artifact store | Parent |
| **`pattern.json`** (per pattern) | `log_params` + `log_metrics` + `log_param` | Flatten `settings.*` into params; `final_score`, `duration_seconds`, scalar aggregates from `scores` as metrics; **log URI** to pattern.json in KFP artifact store | Child |
| **`evaluation_results.json`** (per pattern) | `log_param` (optional) | URI/path to evaluation results in KFP artifact store | **Child, optional** — contains PII / sensitive text; only log pointer if needed for traceability |
| **`indexing_notebook.ipynb`** / **`inference_notebook.ipynb`** | `log_param` | URIs/paths to notebooks in KFP artifact store | Child |
| **`create_model_response.py`** (per pattern) | `log_param` | URI/path to client script in KFP artifact store | Child |
| **Leaderboard HTML** | `log_param` | URI/path to HTML in KFP artifact store | Parent |

---

## MLflow Tracing for RAG Operations (RHOAI 3.6+)

Starting with **MLflow 2.14+**, native tracing capabilities enable detailed observability of RAG operations. AutoRAG extends this to log **retrieval**, **generation**, and **evaluation** traces during pattern optimization, providing visibility into each step of the RAG pipeline.

**Key benefits:**
- **Hierarchical trace structure** - Parent trace per pattern evaluation with child spans for retrieval, generation, and scoring
- **Automatic token tracking** - Input/output token counts and latency for each generation call
- **Full request/response capture** - Complete payloads for debugging and analysis
- **Search and filtering** - Query traces by pattern, metric scores, or operation type in MLflow UI

### Trace hierarchy for RAG patterns

Each RAG pattern evaluation during optimization creates a **top-level trace** with nested spans:

| Trace level | Scope | Logged information |
|-------------|-------|-------------------|
| **Pattern Trace** | One per pattern during optimization | Pattern name, iteration number, configuration hash, total duration |
| **Retrieval Span** | One per query in test dataset | Query text, retrieved chunk IDs, retrieval scores, latency, vector store provider |
| **Generation Span** | One per query in test dataset | Input context (chunks + query), generated response, model ID, token counts (input/output), latency |
| **Evaluation Span** | One per query in test dataset | Ground truth, predicted answer, metric scores (faithfulness, answer_correctness, context_correctness), evaluation model |

**OTLP protocol support:** MLflow accepts traces via `/v1/traces` endpoint using OpenTelemetry Protocol (OTLP), enabling decoupled instrumentation.

### ai4rag integration with MLflow client

The **ai4rag** optimization engine is extended to accept an **initialized MLflow client** from the KFP pipeline component. This allows ai4rag to log traces during pattern exploration without coupling the engine to KFP-specific environment variables.

**Component-level integration pattern:**

```python
import mlflow
import os
from ai4rag import RAGOptimizer  # Example - adjust to actual ai4rag API

def rag_templates_optimization_component(
    test_data: Input[Dataset],
    search_space: Input[Artifact],
    rag_patterns: Output[Artifact],
    # ... other params
):
    """RAG optimization component with MLflow tracing."""
    
    # Initialize MLflow client if tracking is enabled
    mlflow_client = None
    parent_run_id = os.getenv("MLFLOW_RUN_ID")
    
    if parent_run_id and os.getenv("MLFLOW_TRACKING_URI"):
        mlflow_client = mlflow.MlflowClient()
        # Enable tracing for this component
        mlflow.tracing.enable()
    
    # Pass MLflow client to ai4rag optimizer
    optimizer = RAGOptimizer(
        search_space=search_space.path,
        test_data=test_data.path,
        mlflow_client=mlflow_client,  # ai4rag uses this for trace logging
        mlflow_parent_run_id=parent_run_id  # Traces nest under KFP parent run
    )
    
    # Run optimization - ai4rag logs traces internally
    patterns = optimizer.optimize(
        metric="faithfulness",
        max_patterns=8
    )
    
    # Write pattern artifacts to KFP output
    patterns.save(rag_patterns.path)
```

**ai4rag internal trace logging** (example implementation within ai4rag engine):

```python
class RAGOptimizer:
    def __init__(self, mlflow_client=None, mlflow_parent_run_id=None, ...):
        self.mlflow_client = mlflow_client
        self.parent_run_id = mlflow_parent_run_id
        self.tracing_enabled = mlflow_client is not None
    
    def evaluate_pattern(self, pattern_config, test_queries):
        """Evaluate a single RAG pattern with tracing."""
        
        # Create child run for this pattern
        if self.tracing_enabled:
            with mlflow.start_run(run_id=self.parent_run_id):
                pattern_run = mlflow.start_run(
                    run_name=f"pattern_{pattern_config['name']}",
                    nested=True
                )
        
        results = []
        for query_idx, query_item in enumerate(test_queries):
            
            # Start trace for this query evaluation
            if self.tracing_enabled:
                trace_name = f"{pattern_config['name']}_query_{query_idx}"
                with mlflow.start_span(name=trace_name) as trace_span:
                    trace_span.set_attribute("pattern_name", pattern_config['name'])
                    trace_span.set_attribute("query_index", query_idx)
                    
                    # Retrieval span
                    result = self._evaluate_single_query(
                        pattern_config, query_item, trace_span
                    )
            else:
                result = self._evaluate_single_query(pattern_config, query_item)
            
            results.append(result)
        
        if self.tracing_enabled:
            pattern_run.__exit__(None, None, None)
        
        return self._aggregate_results(results)
    
    def _evaluate_single_query(self, pattern, query, parent_span=None):
        """Execute RAG pipeline for one query with nested spans."""
        
        # Retrieval span
        if parent_span:
            with mlflow.start_span(name="retrieval", parent=parent_span) as ret_span:
                ret_span.set_inputs({"query": query['question']})
                chunks = self._retrieve(pattern, query['question'])
                ret_span.set_outputs({
                    "chunks": [c['id'] for c in chunks],
                    "scores": [c['score'] for c in chunks]
                })
                ret_span.set_attribute("num_chunks", len(chunks))
                ret_span.set_attribute("vector_store", pattern['vector_store'])
        else:
            chunks = self._retrieve(pattern, query['question'])
        
        # Generation span
        if parent_span:
            with mlflow.start_span(name="generation", parent=parent_span) as gen_span:
                gen_span.set_inputs({
                    "query": query['question'],
                    "context": [c['text'] for c in chunks]
                })
                
                response = self._generate(pattern, query['question'], chunks)
                
                gen_span.set_outputs({"answer": response['text']})
                gen_span.set_attribute("model_id", pattern['generation_model'])
                gen_span.set_attribute("input_tokens", response.get('usage', {}).get('prompt_tokens', 0))
                gen_span.set_attribute("output_tokens", response.get('usage', {}).get('completion_tokens', 0))
        else:
            response = self._generate(pattern, query['question'], chunks)
        
        # Evaluation span
        if parent_span:
            with mlflow.start_span(name="evaluation", parent=parent_span) as eval_span:
                eval_span.set_inputs({
                    "ground_truth": query.get('answer', ''),
                    "predicted_answer": response['text'],
                    "context": [c['text'] for c in chunks]
                })
                
                scores = self._compute_metrics(query, response, chunks)
                
                eval_span.set_outputs(scores)
                for metric, value in scores.items():
                    eval_span.set_attribute(f"metric_{metric}", value)
        else:
            scores = self._compute_metrics(query, response, chunks)
        
        return {
            "query": query['question'],
            "retrieved_chunks": chunks,
            "response": response,
            "scores": scores
        }
```

### Implementation example

**Component signature extension** (in `rag_templates_optimization` component):

```python
from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Artifact
import mlflow
import mlflow.tracing

@dsl.component(
    base_image="quay.io/opendatahub/autorag:latest",
    packages_to_install=["mlflow>=2.14.0"]
)
def rag_templates_optimization(
    test_data: Input[Dataset],
    search_space_prep_report: Input[Artifact],
    rag_patterns: Output[Artifact],
    optimization_metric: str = "faithfulness",
    optimization_max_rag_patterns: int = 8,
) -> dict:
    """
    RAG optimization with MLflow tracing.
    
    Returns:
        dict: Summary with pattern count and best score
    """
    import os
    import json
    from pathlib import Path
    
    # Check if MLflow tracking is enabled
    tracking_uri = os.getenv("MLFLOW_TRACKING_URI")
    parent_run_id = os.getenv("MLFLOW_RUN_ID")
    
    mlflow_client = None
    if tracking_uri and parent_run_id:
        mlflow.set_tracking_uri(tracking_uri)
        mlflow.tracing.enable()
        mlflow_client = mlflow.MlflowClient()
        print(f"MLflow tracing enabled for parent run: {parent_run_id}")
    else:
        print("MLflow tracking disabled - traces will not be logged")
    
    # Load search space
    with open(search_space_prep_report.path) as f:
        search_space = json.load(f)
    
    # Initialize ai4rag optimizer with MLflow client
    from ai4rag import RAGOptimizer
    
    optimizer = RAGOptimizer(
        search_space=search_space,
        mlflow_client=mlflow_client,
        mlflow_parent_run_id=parent_run_id
    )
    
    # Run optimization - ai4rag logs traces for each pattern evaluation
    patterns = optimizer.optimize(
        test_data_path=test_data.path,
        metric=optimization_metric,
        max_patterns=optimization_max_rag_patterns
    )
    
    # Save pattern artifacts
    output_dir = Path(rag_patterns.path)
    output_dir.mkdir(parents=True, exist_ok=True)
    
    for pattern in patterns:
        pattern_dir = output_dir / pattern['name']
        pattern_dir.mkdir(exist_ok=True)
        
        # Write pattern.json
        with open(pattern_dir / "pattern.json", "w") as f:
            json.dump(pattern, f, indent=2)
        
        # Log pattern to MLflow as child run (in addition to traces)
        if mlflow_client and parent_run_id:
            with mlflow.start_run(run_id=parent_run_id):
                with mlflow.start_run(run_name=pattern['name'], nested=True):
                    # Log pattern config as params
                    mlflow.log_params({
                        f"chunking_{k}": v 
                        for k, v in pattern.get('chunking', {}).items()
                    })
                    mlflow.log_param("embedding_model", pattern.get('embedding', {}).get('model_id'))
                    mlflow.log_param("generation_model", pattern.get('generation', {}).get('model_id'))
                    
                    # Log aggregate metrics
                    mlflow.log_metric("final_score", pattern['final_score'])
                    mlflow.log_metric("faithfulness", pattern['scores'].get('faithfulness', 0))
                    mlflow.log_metric("answer_correctness", pattern['scores'].get('answer_correctness', 0))
                    
                    # Log pattern.json URI
                    mlflow.log_param("pattern_json_path", str(pattern_dir / "pattern.json"))
    
    return {
        "num_patterns": len(patterns),
        "best_score": max(p['final_score'] for p in patterns)
    }
```

**Trace verification:**

```bash
# Search traces for this experiment
MLFLOW_TRACKING_URI=https://mlflow-server.example.com \
mlflow traces search --experiment-id 5

# Query traces programmatically
import mlflow
client = mlflow.MlflowClient()
traces = client.search_traces(
    experiment_ids=["5"],
    filter_string="attributes.pattern_name = 'pattern_001'"
)
for trace in traces:
    print(f"Trace: {trace.info.request_id}")
    for span in trace.data.spans:
        print(f"  Span: {span.name}, duration: {span.end_time_ns - span.start_time_ns}ns")
```

---

## Logging ai4rag Evaluation Results (RHOAI 3.6+)

The **ai4rag** optimization engine already computes evaluation metrics (faithfulness, answer_correctness, context_precision, etc.) during pattern exploration. We simply **log these pre-computed results** to MLflow—no need to re-run evaluation.

### ai4rag evaluation output structure

Each pattern evaluation produces:

**Per-pattern aggregates** (from `pattern.json`):
- `final_score` - Optimization metric score
- `scores` - Dict with metric means (e.g., `{"faithfulness": 0.85, "answer_correctness": 0.78}`)
- `duration_seconds` - Total evaluation time

**Per-query details** (from `evaluation_results.json`):
- `question` - Test query
- `ground_truth` - Expected answer
- `generated_answer` - RAG pipeline output
- `retrieved_contexts` - Retrieved chunks
- `scores` - Per-query metric scores (e.g., `{"faithfulness": 0.9, "answer_correctness": 0.8}`)

### Logging pattern evaluation results

**Log aggregate metrics to child runs:**

```python
def log_pattern_results_to_mlflow(
    pattern_result: dict,
    parent_run_id: str,
    mlflow_client: mlflow.MlflowClient
):
    """
    Log ai4rag pattern evaluation results to MLflow.
    
    Args:
        pattern_result: Pattern result from ai4rag (contains scores, config, etc.)
        parent_run_id: KFP parent run ID
        mlflow_client: Initialized MLflow client
    """
    with mlflow.start_run(run_id=parent_run_id):
        # Create child run for this pattern
        with mlflow.start_run(run_name=pattern_result['name'], nested=True):
            
            # Log pattern configuration as params
            mlflow.log_params({
                "embedding_model": pattern_result['settings']['embedding']['model_id'],
                "generation_model": pattern_result['settings']['generation']['model_id'],
                "chunk_size": pattern_result['settings']['chunking']['chunk_size'],
                "chunk_overlap": pattern_result['settings']['chunking']['overlap'],
                "retrieval_top_k": pattern_result['settings']['retrieval'].get('top_k', 5),
            })
            
            # Log aggregate evaluation metrics
            for metric_name, metric_value in pattern_result['scores'].items():
                mlflow.log_metric(metric_name, metric_value)
            
            # Log final score (optimization metric)
            mlflow.log_metric("final_score", pattern_result['final_score'])
            mlflow.log_metric("duration_seconds", pattern_result['duration_seconds'])
            
            # Log pointers to KFP artifacts
            mlflow.log_param("pattern_json_path", pattern_result['artifact_paths']['pattern_json'])
            mlflow.log_param("evaluation_results_path", pattern_result['artifact_paths']['evaluation_results'])
```

**Log per-query details in evaluation spans:**

```python
def log_query_evaluation_spans(
    evaluation_results: list,
    pattern_name: str,
    parent_span
):
    """
    Log per-query evaluation details as nested spans.
    
    Args:
        evaluation_results: List of per-query results from ai4rag evaluation_results.json
        pattern_name: Pattern identifier
        parent_span: Parent MLflow span for this pattern trace
    """
    for idx, query_result in enumerate(evaluation_results):
        # Create evaluation span for this query
        with mlflow.start_span(
            name=f"autorag.query_{idx}_evaluation",
            parent=parent_span
        ) as eval_span:
            
            # Log inputs
            eval_span.set_inputs({
                "question": query_result['question'],
                "ground_truth": query_result['ground_truth'],
                "retrieved_contexts": query_result['retrieved_contexts']
            })
            
            # Log outputs
            eval_span.set_outputs({
                "generated_answer": query_result['generated_answer']
            })
            
            # Log per-query metric scores as span attributes
            for metric_name, score in query_result['scores'].items():
                eval_span.set_attribute(f"metric.{metric_name}", score)
            
            # Log metadata
            eval_span.set_attribute("num_retrieved_chunks", len(query_result['retrieved_contexts']))
            eval_span.set_attribute("pattern_name", pattern_name)
```

### Integration example in rag_templates_optimization

```python
def rag_templates_optimization_with_mlflow_logging(
    test_data: Input[Dataset],
    rag_patterns: Output[Artifact],
    mlflow_client: mlflow.MlflowClient,
    parent_run_id: str
):
    """
    Run ai4rag optimization and log results to MLflow.
    """
    from ai4rag import RAGOptimizer
    
    # Initialize optimizer with MLflow client for tracing
    optimizer = RAGOptimizer(
        mlflow_client=mlflow_client,
        mlflow_parent_run_id=parent_run_id
    )
    
    # Run optimization - ai4rag computes metrics internally
    patterns = optimizer.optimize(
        test_data_path=test_data.path,
        metric="faithfulness",
        max_patterns=8
    )
    
    # Log each pattern's results to MLflow
    for pattern in patterns:
        log_pattern_results_to_mlflow(
            pattern_result=pattern,
            parent_run_id=parent_run_id,
            mlflow_client=mlflow_client
        )
        
        # Optionally: Log per-query details if evaluation_results exist
        if pattern.get('evaluation_results'):
            # This would be done within ai4rag during optimization
            # if mlflow_client is provided (see tracing section)
            pass
    
    # Save patterns to KFP artifact
    save_patterns_to_artifact(patterns, rag_patterns.path)
```

### Evaluation metrics logged

ai4rag computes RAGAS-compatible metrics that are logged to MLflow:

| Metric | Description | Logged As |
|--------|-------------|-----------|
| **Faithfulness** | Answer grounded in retrieved context (no hallucinations) | `mlflow.log_metric("faithfulness", score)` |
| **Answer Correctness** | Semantic + factual overlap with ground truth | `mlflow.log_metric("answer_correctness", score)` |
| **Context Precision** | Fraction of retrieved chunks relevant to question | `mlflow.log_metric("context_precision", score)` |
| **Context Recall** | Ground truth supported by retrieved context | `mlflow.log_metric("context_recall", score)` |
| **Answer Relevancy** | Answer directly addresses the question | `mlflow.log_metric("answer_relevancy", score)` |

All metrics are **computed by ai4rag** during optimization. The MLflow integration simply logs the results for tracking and comparison.

### Viewing results in MLflow UI

After running the pipeline:

1. Navigate to MLflow UI: `https://mlflow-server.example.com/#/experiments/5`
2. Select the parent run (KFP pipeline execution)
3. View child runs (one per pattern)
4. Compare patterns side-by-side:
   - Sort by `faithfulness`, `final_score`, or other metrics
   - View metric distributions across patterns
   - Identify best-performing pattern configurations
5. Drill into individual pattern run:
   - View all evaluation metrics (aggregate scores)
   - Check pattern configuration params
   - Access KFP artifact paths for detailed results

---

## KFP Pipeline Integration: MLflow Tracking Artifact

AutoRAG should use the **same dedicated KFP output** as [AutoML’s MLflow tracking artifact](../../automl/features/mlflow_integration.md#kfp-pipeline-integration-mlflow-tracking-artifact): a JSON file exposed as a pipeline step output so the **AutoRAG Dashboard**, KFP UI, and automation can resolve **MLflow experiment ID**, **parent run ID**, and **deep links** without scraping logs or guessing from workspace filters alone.

**Target producer:** the **`leaderboard_evaluation`** step in **[`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline)** (alongside the HTML leaderboard) is the logical component to emit **`mlflow_tracking_artifact`**. The **[`documents_indexing_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/data_processing/autorag/documents_indexing_pipeline)** is a **separate** pipeline; if it later emits a tracking artifact for indexing-only runs, reuse the **same JSON shape** for consistency.

**Implementation:** Until this output exists in the **red-hat-data-services / opendatahub-io** branch you deploy, treat the following as the **product contract** to implement and verify—mirroring the AutoML field definitions [here](../../automl/features/mlflow_integration.md#artifact-schema).

### Artifact purpose

The **`mlflow_tracking_artifact`** JSON is the **single source of truth** for linking a KFP pipeline execution to MLflow tracking:

1. **Dashboard integration:** AutoRAG Dashboard reads the artifact via the KFP API to obtain `mlflow_experiment_id`, `mlflow_run_id`, and UI URLs.
2. **Deep-linking:** Users jump from Data Science Pipelines to the MLflow UI for the parent run (and nested pattern child runs).
3. **Programmatic access:** CI/CD and scripts resolve tracking IDs without parsing component logs.
4. **Audit trail:** Stable record of which MLflow parent run belongs to which KFP run.

### Component output definition

The leaderboard (or equivalent final) component declares a KFP artifact output—for example:

```python
mlflow_tracking_artifact: dsl.Output[dsl.Artifact]
```

The step writes **one JSON file** to `mlflow_tracking_artifact.path` before completing.

### Artifact schema

**When MLflow tracking is enabled** (`MLFLOW_TRACKING_URI` set during the run):

```json
{
  "tracking_enabled": true,
  "mlflow_tracking_uri": "https://mlflow-server.example.com",
  "mlflow_experiment_id": "5",
  "mlflow_run_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
  "mlflow_workspace": "data-science-project",
  "mlflow_run_url": "https://mlflow-server.example.com/#/experiments/5/runs/a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5",
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "documents-rag-optimization-pipeline"
}
```

| Field | Type | Typical source | Description |
|-------|------|----------------|-------------|
| `tracking_enabled` | bool | Derived | `true` when tracking was active for this execution |
| `mlflow_tracking_uri` | string | `MLFLOW_TRACKING_URI` | MLflow tracking server endpoint |
| `mlflow_experiment_id` | string | `MLFLOW_EXPERIMENT_ID` | KFP-managed experiment for this pipeline |
| `mlflow_run_id` | string | `MLFLOW_RUN_ID` | **Parent** MLflow run for this pipeline execution (pattern runs nest under it) |
| `mlflow_workspace` | string | `MLFLOW_WORKSPACE` | OpenShift AI project / namespace |
| `mlflow_run_url` | string | Computed | Deep link to the parent run in the MLflow UI |
| `kfp_run_id` | string | KFP context / placeholder | Kubeflow pipeline run ID for cross-reference |
| `pipeline_name` | string | KFP context / placeholder | e.g. `documents-rag-optimization-pipeline` |

**When MLflow tracking is disabled** (`MLFLOW_TRACKING_URI` unset):

```json
{
  "tracking_enabled": false,
  "kfp_run_id": "run-abc123-def456",
  "pipeline_name": "documents-rag-optimization-pipeline"
}
```

---

## Related

- AutoRAG component overview: [../README.md](../README.md)
- AutoRAG ADR: [ODH-ADR-0001-autorag](../../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md)
- Shared platform MLflow details: [AutoML MLflow integration](../../automl/features/mlflow_integration.md)
- Data Science Pipelines architecture: [../../pipelines/README.md](../../pipelines/README.md)
