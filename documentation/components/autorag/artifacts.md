# AutoRAG Artifacts

AutoRAG Pipeline run produces the following artifacts.

## Naming Convention

Artifact names follow these conventions:
- **RAG Pattern artifacts**: Simple pattern identifiers (e.g., `RAGPattern1`, `RAGPattern2`)
- **Run-level artifacts**: Use `AutoRAG_<run_name>_` prefix followed by artifact type suffix:
  - Run Output: `AutoRAG_<run_name>_output`
  - Experiment Summary: `AutoRAG_<run_name>_summary`
- **Pattern-specific run artifacts**: Combine run name and pattern identifier:
  - Metrics: `AutoRAG_<run_name>_RAGPattern1_metrics`

## RAG Pattern

RAG Pattern `dsl.Artifact` consisting of:
- RAG Pattern `name` `RAGPattern1`
- `uri` to tar.gz with generated pattern code as pair of notebooks
- `metadata` describing used RAG configuration settings and evaluation metrics


📝 **Note:** There will be multiple RAG Pattern artifacts per single run.

### Sample artifact

📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "uri": "rag_pattern_1.tar.gz",  
  "name": "RAGPattern1",  
  "metadata": {  
    "context": {  
      "iteration": 3,  
      "max_combinations": 24,  
      "rag_pattern": {  
        "composition_steps": [  
          "model_selection",  
          "chunking",  
          "embeddings",  
          "retrieval",  
          "generation"  
        ],  
        "duration_seconds": 19,  
        "location": {  
          "evaluation_results": "autorag/results/71c46718-8faa-4a0e-a018-073edfdca527/Pattern4/evaluation_results.json",  
          "indexing_notebook": "autorag/results/71c46718-8faa-4a0e-a018-073edfdca527/Pattern4/indexing_notebook.ipynb",  
          "inference_notebook": "autorag/results/71c46718-8faa-4a0e-a018-073edfdca527/Pattern4/inference_notebook.ipynb"  
        },  
        "name": "Pattern4",  
        "settings": {  
          "agent": {  
            "description": "Sequential graph with single index retriever.",  
            "framework": "langgraph",  
            "type": "sequential"  
          },  
          "chunking": {  
            "chunk_overlap": 0,  
            "chunk_size": 2048,  
            "method": "semantic"  
          },  
          "embeddings": {  
            "model_id": "intfloat/multilingual-e5-large",  
            "truncate_input_tokens": 512,  
            "truncate_strategy": "left"  
          },  
          "generation": {  
            "chat_template_messages": {  
              "system_message_text": "You are a AI language model designed to function as a specialized Retrieval Augmented Generation (RAG) assistant. When generating responses, prioritize correctness, i.e., ensure that your response is correct given the context and user query, and that it is grounded in the context. Furthermore, make sure that the response is supported by the given document or context. When the question cannot be answered using the context or document, output the following response: 'I am sorry, I do not have the information you are looking for in my knowledge base.'. Always make sure that your response is relevant to the question. If an explanation is needed, first provide the explanation or reasoning, and then give the final answer.\nAnswer Length: concise.\n\n",  
              "user_message_text": "[Document]\n{reference_documents}\n[End]\n{question}. \nRespond exclusively in the language of the question, regardless of any other language used in the provided context. Ensure that your entire response is in the same language as the question."  
            },  
            "context_template_text": "{document}",  
            "model_id": "openai/gpt-oss-120b",  
            "parameters": {  
              "max_completion_tokens": 2048,  
              "temperature": 0.2  
            },  
            "word_to_token_ratio": 2  
          },  
          "retrieval": {  
            "hybrid_ranker": {  
              "alpha": 0.8,  
              "sparse_vectors": {  
                "model_id": ".elser_model_2_linux-x86_64"  
              },  
              "strategy": "weighted"  
            },  
            "method": "window",  
            "number_of_chunks": 4,  
            "window_size": 2  
          },  
          "vector_store": {  
            "datasource_type": "elasticsearch",  
            "index_name": "autoai_rag_71c46718_20251211114212"
          }  
        }
      },  
      "runtime_ver": {  
        "name": "autorag_3.3.0"  
      }  
    },  
    "metrics": {  
      "test_data": [  
        {  
          "ci_high": 0.6105,  
          "ci_low": 0.3495,  
          "mean": 0.481,  
          "metric_name": "answer_correctness"  
        },  
        {  
          "ci_high": 0.76,  
          "ci_low": 0.4599,  
          "mean": 0.6118,  
          "metric_name": "faithfulness"  
        },  
        {  
          "mean": 1,  
          "metric_name": "context_correctness"  
        }  
      ]  
    }  
  }  
}
```

## AutoRAG Run Output Artifact
Run Output `dsl.Artifact` consisting of:
- Run Output `name` `AutoRAG_<run_name>_output`
- `uri` to log file persisted in result directory
- `metadata` describing the run status

### Sample artifact
📝 **Note:** this is just a draft version - changes can be made

```json
{  
  "name": "AutoRAG_<run_name>_output",  
  "uri": "/path/run_output.log",  
  "metadata": {  
    "status": {  
      "completed_at": "2025-12-11T11:42:40.380Z",  
      "message": {  
        "level": "info",  
        "text": "AutoAI RAG execution completed."  
      },  
      "running_at": "2025-12-11T11:42:40.000Z",  
      "state": "completed",  
      "step": "generation"  
    },  
    "input_settings": {  
      "all-that": "came-from-user"  
    }  
  }  
}
```

## Metrics

Metrics artifact of type `dsl.Metrics` to keep evaluation metrics per RAG Pattern (or all Patterns as yet another alternative).

⚠️ **Warning:** To be confirmed if this Metrics artifact is required, as similar information is already part of the RAG Pattern artifact.


### Example of artifact
```json
{  
  "name": "AutoRAG_<run_name>_RAGPattern1_metrics",  
  "uri": "link-to-rag-pattern-artifact?",  
  "metadata": {  
    "context": {  
      "rag_pattern_artifact_name": "RAGPattern1",  
      "rag_pattern_artifact_id": "---id---" 
    },  
    "faithfulness_avg": 0.79,  
    "answer_correctness_avg": 0.5,  
    "context_correctness_avg": 1  
  }  
}
```

## Experiment summary

The `dsl.Markdown` Artifact describing the experiment details. This is a comprehensive report that provides an overview of the AutoRAG optimization run, including data preparation, search space exploration, and results.

### Example of artifact
```json
{  
  "name": "AutoRAG_<run_name>_summary",  
  "uri": "path/summary_report.md",  
  "metadata": {  
    "experiment_name": "AutoRAG Experiment 1",
    "run_id": "71c46718-8faa-4a0e-a018-073edfdca527",
    "created_at": "2025-12-11T11:42:40.380Z",
    "total_patterns": 4,
    "optimization_metric": "answer_correctness"
  }  
}
```

### Proposed Summary Report Structure

Based on AutoAI RAG (IBM) approach, the experiment summary Markdown report should include the following sections:

```markdown
# AutoRAG Experiment Summary

**Experiment Name:** AutoRAG Experiment 1  
**Run ID:** 71c46718-8faa-4a0e-a018-073edfdca527  
**Created At:** 2025-12-11T11:42:40.380Z  
**Status:** Completed  
**Duration:** 2h 15m 32s

---

## Executive Summary

This experiment explored **24** RAG configurations and generated **4** optimized RAG Patterns. The optimization focused on maximizing **answer_correctness** metric. The best performing pattern achieved an answer_correctness score of **0.612** with faithfulness of **0.789**.

**Top Pattern:** RAGPattern1  
**Best Metric Score:** 0.612 (answer_correctness)  
**Total Configurations Explored:** 24  
**Optimization Iterations:** 12

---

## Experiment Configuration

### Input Data Sources
- **Document Data:**
  - Connection: `s3-documents-connection`
  - Bucket: `my-documents-bucket`
  - Path: `rh_documents/`
  - Total Documents: 1,234
  - Total Size: 2.3 GB

- **Test Data:**
  - Connection: `s3-benchmarks-connection`
  - Bucket: `autorag_benchmarks`
  - Path: `my-folder/test_data.json`
  - Test Cases: 150
  - Questions per Case: 1

### Infrastructure
- **Vector Database:** Milvus (milvus-database)
- **LLM Provider:** Llama-stack
- **Results Storage:** `s3://results/autorag/71c46718-8faa-4a0e-a018-073edfdca527/`

### Optimization Settings
- **Max RAG Patterns:** 4
- **Optimization Metric:** answer_correctness
- **Search Strategy:** GAM-based prediction

---

## Data Preparation

### Document Processing
- **Sampling Method:** Test data driven sampling
- **Sampled Documents:** 450
  - Documents referenced in test data: 150
  - Noise documents added: 300
- **Sampling Limit:** 1GB (in-memory)
- **Text Extraction:** docling library
- **Extracted Text Size:** 850 MB

### Document Statistics
- **Average Document Length:** 2,450 tokens
- **Document Types:** PDF (60%), DOCX (25%), Markdown (10%), HTML (5%)
- **Languages:** English (100%)

---

## Search Space Definition

### Chunking Constraints
- **Method:** Recursive
- **Chunk Sizes Explored:** [512, 1024, 2048]
- **Chunk Overlaps Explored:** [0, 128, 256]
- **Total Chunking Configurations:** 9

### Embedding Models
- `ibm/slate-125m-english-rtrvr-v2`
- `intfloat/multilingual-e5-large`
- **Total Embedding Configurations:** 2

### Generation Models
- `mistralai/mixtral-8x7b-instruct-v01`
- `ibm/granite-13b-instruct-v2`
- `ibm/granite-3-8b-instruct`
- **Total Generation Configurations:** 3

### Retrieval Methods
- Simple retrieval (number_of_chunks: [2, 4, 6])
- Simple with hybrid ranker (alpha: [0.4, 0.6, 0.8])
- **Total Retrieval Configurations:** 6

### Total Search Space
- **Theoretical Combinations:** 324 (9 × 2 × 3 × 6)
- **Validated Configurations:** 24
- **Configurations Explored:** 12
- **Configurations Evaluated:** 12

---

## Explored Configurations

| Iteration | Configuration ID | Chunking | Embeddings | Generation | Retrieval | Answer Correctness | Faithfulness | Context Correctness |
|-----------|------------------|----------|------------|------------|-----------|-------------------|--------------|---------------------|
| 1 | Config-001 | recursive (2048/256) | slate-125m | mixtral-8x7b | simple (2) | 0.481 | 0.612 | 1.000 |
| 2 | Config-002 | recursive (1024/128) | multilingual-e5 | granite-13b | simple (4) | 0.523 | 0.645 | 1.000 |
| 3 | Config-003 | recursive (2048/0) | slate-125m | granite-3-8b | hybrid (0.6) | 0.498 | 0.623 | 0.987 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
| 12 | Config-012 | recursive (2048/256) | multilingual-e5 | granite-13b | hybrid (0.8) | **0.612** | **0.789** | 1.000 |

---

## Leaderboard

### Top RAG Patterns

| Rank | Pattern Name | Answer Correctness | Faithfulness | Context Correctness | Configuration |
|------|--------------|-------------------|--------------|---------------------|---------------|
| 🥇 1 | RAGPattern1 | **0.612** | 0.789 | 1.000 | recursive (2048/256) + multilingual-e5 + granite-13b + hybrid (0.8) |
| 🥈 2 | RAGPattern2 | 0.589 | 0.756 | 1.000 | recursive (1024/128) + multilingual-e5 + mixtral-8x7b + simple (4) |
| 🥉 3 | RAGPattern3 | 0.567 | 0.734 | 0.987 | recursive (2048/0) + slate-125m + granite-3-8b + hybrid (0.6) |
| 4 | RAGPattern4 | 0.545 | 0.712 | 1.000 | recursive (1024/256) + multilingual-e5 + granite-13b + simple (6) |

### Performance Metrics Summary

**Answer Correctness:**
- Mean: 0.551
- Std Dev: 0.032
- Best: 0.612 (RAGPattern1)
- Worst: 0.481 (Config-001)

**Faithfulness:**
- Mean: 0.722
- Std Dev: 0.058
- Best: 0.789 (RAGPattern1)
- Worst: 0.612 (Config-001)

**Context Correctness:**
- Mean: 0.997
- Std Dev: 0.006
- Best: 1.000 (Multiple patterns)
- Worst: 0.987 (Config-003)

---

## Pattern Details

### RAGPattern1 (Best Performing)

**Configuration:**
- **Chunking:** Recursive method, chunk_size=2048, chunk_overlap=256
- **Embeddings:** intfloat/multilingual-e5-large
- **Generation:** ibm/granite-13b-instruct-v2
- **Retrieval:** Simple with hybrid ranker (strategy=weighted, alpha=0.8, number_of_chunks=4)

**Performance Metrics:**
- Answer Correctness: 0.612 (CI: 0.349 - 0.610)
- Faithfulness: 0.789 (CI: 0.460 - 0.760)
- Context Correctness: 1.000

**Artifacts:**
- Indexing Notebook: `autorag/results/.../RAGPattern1/indexing_notebook.ipynb`
- Inference Notebook: `autorag/results/.../RAGPattern1/inference_notebook.ipynb`
- Pattern Archive: `autorag/results/.../RAGPattern1/rag_pattern_1.tar.gz`

---

## Optimization Insights

### Key Findings
1. **Chunking:** Larger chunk sizes (2048) with moderate overlap (256) performed best
2. **Embeddings:** Multilingual models showed superior performance for English content
3. **Generation:** Granite-13b-instruct-v2 provided the best balance of correctness and faithfulness
4. **Retrieval:** Hybrid ranker with higher alpha (0.8) improved answer quality

### Configuration Trends
- **Best Chunking:** recursive (2048/256) - appeared in top 3 patterns
- **Best Embedding:** multilingual-e5-large - used in all top 4 patterns
- **Best Generation:** granite-13b-instruct-v2 - appeared in 3 of top 4 patterns
- **Best Retrieval:** Hybrid ranker with alpha ≥ 0.6 - used in top 2 patterns

---

## Artifacts and Resources

### Generated Artifacts
- **RAG Pattern Artifacts:** 4 patterns with executable notebooks
- **Run Output Log:** `autorag/results/.../run_output.log`
- **Metrics Artifacts:** Per-pattern evaluation metrics

### Links
- [View All Artifacts](./artifacts/)
- [Download RAGPattern1](./RAGPattern1/rag_pattern_1.tar.gz)
- [View Run Log](./run_output.log)

---

## Next Steps

### Recommended Actions
1. **Deploy RAGPattern1** for production use
2. **Further Optimization:** Consider exploring:
   - Additional chunk sizes: [1536, 2560]
   - Different hybrid ranker strategies
   - Fine-tuned generation parameters
3. **Evaluation:** Test RAGPattern1 on additional test datasets
4. **Monitoring:** Track performance metrics in production

---

## Technical Details

### Runtime Information
- **AutoRAG Version:** 3.3.0
- **ai4rag Version:** 1.2.0
- **Pipeline Execution Time:** 2h 15m 32s
- **Average Configuration Evaluation Time:** 11m 18s

### Search Space Validation
- **Model Validation:** 24 configurations validated using in-memory vector database
- **Preselection:** 12 configurations selected for evaluation
- **GAM Model:** Used for configuration selection and score prediction

---

*Report generated by AutoRAG on 2025-12-11T11:42:40.380Z*
```

### Summary Report Sections

The proposed summary report includes:

1. **Executive Summary** - High-level overview with key metrics and top performer
2. **Experiment Configuration** - Input data sources, infrastructure, and optimization settings
3. **Data Preparation** - Document processing details, sampling statistics, and text extraction info
4. **Search Space Definition** - Complete breakdown of constraints and explored configurations
5. **Explored Configurations** - Table of all evaluated configurations with metrics
6. **Leaderboard** - Ranked list of top RAG Patterns with performance metrics
7. **Pattern Details** - Detailed information about the best performing pattern
8. **Optimization Insights** - Key findings and configuration trends
9. **Artifacts and Resources** - Links to generated artifacts
10. **Next Steps** - Recommended actions and further optimization suggestions
11. **Technical Details** - Runtime information and validation details

This structure provides a comprehensive view of the optimization experiment, making it easy to understand what was explored, what performed best, and how to proceed with the results.