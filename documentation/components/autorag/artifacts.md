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

The `dsl.Markdown` Artifact describing the experiment details.


### Example of artifact
```json
{  
  "name": "AutoRAG_<run_name>_summary",  
  "uri": "path/summary_report.md",  
  "metadata": {  
  }  
}
```