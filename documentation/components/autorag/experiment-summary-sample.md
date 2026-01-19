<div style="margin-bottom: 0;">

![AutoRAG Banner](images/autorag-banner.svg)

</div>

<div style="margin-top: -10px; padding: 20px 0 10px 0;">

**Experiment Name:** AutoRAG Experiment 1  
**Run ID:** `71c46718-8faa-4a0e-a018-073edfdca527`  
**Created At:** 2025-12-11T11:42:40.380Z  
**Status:** <span style="color: #28a745; font-weight: bold;">✓ Completed</span>  
**Duration:** 2h 15m 32s

</div>

---

## 📋 Table of Contents

- [Executive Summary](#executive-summary)
- [Experiment Configuration](#experiment-configuration)
- [Data Preparation](#data-preparation)
- [Search Space Definition](#search-space-definition)
- [Explored Configurations](#explored-configurations)
- [Leaderboard](#leaderboard)
- [Pattern Details](#pattern-details)
- [Optimization Insights](#optimization-insights)
- [Artifacts and Resources](#artifacts-and-resources)
- [Next Steps](#next-steps)
- [Technical Details](#technical-details)

---

## 🎯 Executive Summary

This experiment explored **24** RAG configurations and generated **4** optimized RAG Patterns. The optimization focused on maximizing **answer_correctness** metric. The best performing pattern achieved an answer_correctness score of **0.612** with faithfulness of **0.789**.

<div style="background-color: #f8f9fa; color: #212529; padding: 15px; border-radius: 5px; border-left: 4px solid #EE0000; margin: 20px 0;">

**🏆 Key Results:**
- **Top Pattern:** `RAGPattern1`
- **Best Metric Score:** `0.612` (answer_correctness)
- **Total Configurations Explored:** 24
- **Optimization Iterations:** 12

</div>

---

## ⚙️ Experiment Configuration

<details>
<summary><b>Click to expand configuration details</b></summary>

### 📊 Input Data Sources

**Document Data:**
- **Connection:** `s3-documents-connection`
- **Bucket:** `my-documents-bucket`
- **Path:** `rh_documents/`
- **Total Documents:** 1,234
- **Total Size:** 2.3 GB

**Test Data:**
- **Connection:** `s3-benchmarks-connection`
- **Bucket:** `autorag_benchmarks`
- **Path:** `my-folder/test_data.json`
- **Test Cases:** 150
- **Questions per Case:** 1

### 🏗️ Infrastructure

| Component | Configuration |
|-----------|--------------|
| **Vector Database** | Milvus (`milvus-database`) |
| **LLM Provider** | Llama-stack |
| **Results Storage** | `s3://results/autorag/71c46718-8faa-4a0e-a018-073edfdca527/` |

### 🔧 Optimization Settings

| Parameter | Value |
|-----------|-------|
| **Max RAG Patterns** | 4 |
| **Optimization Metric** | `answer_correctness` |
| **Search Strategy** | GAM-based prediction |

</details>

---

## 📊 Data Preparation

### Document Processing

| Setting | Value |
|---------|-------|
| **Sampling Method** | Test data driven sampling |
| **Sampled Documents** | 450 |
| &nbsp;&nbsp;→ Documents referenced in test data | 150 |
| &nbsp;&nbsp;→ Noise documents added | 300 |
| **Sampling Limit** | 1GB (in-memory) |
| **Text Extraction** | docling library |
| **Extracted Text Size** | 850 MB |

### Document Statistics

| Metric | Value |
|--------|-------|
| **Average Document Length** | 2,450 tokens |
| **Document Types** | PDF (60%), DOCX (25%), Markdown (10%), HTML (5%) |
| **Languages** | English (100%) |

---

## 🔍 Search Space Definition

<details>
<summary><b>Click to expand search space details</b></summary>

### Chunking Constraints

| Parameter | Values |
|-----------|--------|
| **Method** | Recursive |
| **Chunk Sizes Explored** | `[512, 1024, 2048]` |
| **Chunk Overlaps Explored** | `[0, 128, 256]` |
| **Total Chunking Configurations** | 9 |

### Embedding Models

- `ibm/slate-125m-english-rtrvr-v2`
- `intfloat/multilingual-e5-large`

**Total Embedding Configurations:** 2

### Generation Models

- `mistralai/mixtral-8x7b-instruct-v01`
- `ibm/granite-13b-instruct-v2`
- `ibm/granite-3-8b-instruct`

**Total Generation Configurations:** 3

### Retrieval Methods

- Simple retrieval (`number_of_chunks`: `[2, 4, 6]`)
- Simple with hybrid ranker (`alpha`: `[0.4, 0.6, 0.8]`)

**Total Retrieval Configurations:** 6

### Total Search Space

| Metric | Value |
|--------|-------|
| **Theoretical Combinations** | 324 (9 × 2 × 3 × 6) |
| **Validated Configurations** | 24 |
| **Configurations Explored** | 12 |
| **Configurations Evaluated** | 12 |

</details>

---

## 🔬 Explored Configurations

| Iteration | Configuration ID | Chunking | Embeddings | Generation | Retrieval | Answer Correctness | Faithfulness | Context Correctness |
|:---------:|------------------|----------|------------|------------|-----------|:------------------:|:------------:|:-------------------:|
| 1 | Config-001 | recursive (2048/256) | slate-125m | mixtral-8x7b | simple (2) | 0.481 | 0.612 | 1.000 |
| 2 | Config-002 | recursive (1024/128) | multilingual-e5 | granite-13b | simple (4) | 0.523 | 0.645 | 1.000 |
| 3 | Config-003 | recursive (2048/0) | slate-125m | granite-3-8b | hybrid (0.6) | 0.498 | 0.623 | 0.987 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
| 12 | Config-012 | recursive (2048/256) | multilingual-e5 | granite-13b | hybrid (0.8) | **0.612** | **0.789** | 1.000 |

---

## 🏆 Leaderboard

### Top RAG Patterns

| Rank | Pattern Name | Answer Correctness | Faithfulness | Context Correctness | Configuration |
|:----:|--------------|:------------------:|:------------:|:-------------------:|---------------|
| 🥇 | **RAGPattern1** | **0.612** | 0.789 | 1.000 | recursive (2048/256) + multilingual-e5 + granite-13b + hybrid (0.8) |
| 🥈 | RAGPattern2 | 0.589 | 0.756 | 1.000 | recursive (1024/128) + multilingual-e5 + mixtral-8x7b + simple (4) |
| 🥉 | RAGPattern3 | 0.567 | 0.734 | 0.987 | recursive (2048/0) + slate-125m + granite-3-8b + hybrid (0.6) |
| 4 | RAGPattern4 | 0.545 | 0.712 | 1.000 | recursive (1024/256) + multilingual-e5 + granite-13b + simple (6) |

### Performance Metrics Summary

<details>
<summary><b>View detailed metrics statistics</b></summary>

**Answer Correctness:**
- **Mean:** 0.551
- **Std Dev:** 0.032
- **Best:** 0.612 (RAGPattern1)
- **Worst:** 0.481 (Config-001)

**Faithfulness:**
- **Mean:** 0.722
- **Std Dev:** 0.058
- **Best:** 0.789 (RAGPattern1)
- **Worst:** 0.612 (Config-001)

**Context Correctness:**
- **Mean:** 0.997
- **Std Dev:** 0.006
- **Best:** 1.000 (Multiple patterns)
- **Worst:** 0.987 (Config-003)

</details>

---

## 🎯 Pattern Details

### RAGPattern1 (Best Performing)

<div style="background-color: #fff3cd; color: #212529; padding: 15px; border-radius: 5px; border-left: 4px solid #ffc107; margin: 20px 0;">

**Configuration:**
- **Chunking:** Recursive method, `chunk_size=2048`, `chunk_overlap=256`
- **Embeddings:** `intfloat/multilingual-e5-large`
- **Generation:** `ibm/granite-13b-instruct-v2`
- **Retrieval:** Simple with hybrid ranker (`strategy=weighted`, `alpha=0.8`, `number_of_chunks=4`)

</div>

#### Performance Metrics

| Metric | Value | Confidence Interval |
|-------|:-----:|:------------------:|
| **Answer Correctness** | 0.612 | 0.349 - 0.610 |
| **Faithfulness** | 0.789 | 0.460 - 0.760 |
| **Context Correctness** | 1.000 | - |

#### Artifacts

- **Indexing Notebook:** `autorag/results/.../RAGPattern1/indexing_notebook.ipynb`
- **Inference Notebook:** `autorag/results/.../RAGPattern1/inference_notebook.ipynb`
- **Pattern Archive:** `autorag/results/.../RAGPattern1/rag_pattern_1.tar.gz`

---

## 💡 Optimization Insights

### Key Findings

1. **Chunking:** Larger chunk sizes (2048) with moderate overlap (256) performed best
2. **Embeddings:** Multilingual models showed superior performance for English content
3. **Generation:** Granite-13b-instruct-v2 provided the best balance of correctness and faithfulness
4. **Retrieval:** Hybrid ranker with higher alpha (0.8) improved answer quality

### Configuration Trends

| Component | Best Choice | Usage in Top Patterns |
|-----------|-------------|----------------------|
| **Chunking** | recursive (2048/256) | Appeared in top 3 patterns |
| **Embedding** | multilingual-e5-large | Used in all top 4 patterns |
| **Generation** | granite-13b-instruct-v2 | Appeared in 3 of top 4 patterns |
| **Retrieval** | Hybrid ranker with alpha ≥ 0.6 | Used in top 2 patterns |

---

## 📦 Artifacts and Resources

### Generated Artifacts

- ✅ **RAG Pattern Artifacts:** 4 patterns with executable notebooks
- ✅ **Run Output Log:** `autorag/results/.../run_output.log`
- ✅ **Metrics Artifacts:** Per-pattern evaluation metrics

### Quick Links

- [📁 View All Artifacts](./artifacts/)
- [⬇️ Download RAGPattern1](./RAGPattern1/rag_pattern_1.tar.gz)
- [📝 View Run Log](./run_output.log)

---

## 🚀 Next Steps

### Recommended Actions

1. ✅ **Deploy RAGPattern1** for production use
2. 🔄 **Further Optimization:** Consider exploring:
   - Additional chunk sizes: `[1536, 2560]`
   - Different hybrid ranker strategies
   - Fine-tuned generation parameters
3. 📊 **Evaluation:** Test RAGPattern1 on additional test datasets
4. 📈 **Monitoring:** Track performance metrics in production

---

## 🔧 Technical Details

<details>
<summary><b>View technical details</b></summary>

### Runtime Information

| Component | Version/Value |
|-----------|--------------|
| **AutoRAG Version** | 3.3.0 |
| **ai4rag Version** | 1.2.0 |
| **Pipeline Execution Time** | 2h 15m 32s |
| **Average Configuration Evaluation Time** | 11m 18s |

### Search Space Validation

| Process | Details |
|---------|---------|
| **Model Validation** | 24 configurations validated using in-memory vector database |
| **Preselection** | 12 configurations selected for evaluation |
| **GAM Model** | Used for configuration selection and score prediction |

</details>

---

<div align="center" style="margin-top: 40px; padding-top: 20px; border-top: 1px solid #ddd; color: #666; font-size: 0.9em;">

_Report generated by AutoRAG on 2025-12-11T11:42:40.380Z_

</div>
