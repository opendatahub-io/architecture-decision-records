# Jira Issue: Docling Hierarchical Chunker POC and Benchmarking

**Project:** RHOAIENG  
**Issue Type:** Task  
**Summary:** POC and benchmark Docling hierarchical chunker with contextualize vs contextual retrieval

---

## Description

### Objective

Evaluate **Docling's HierarchicalChunker with `contextualize()`** as an alternative to Anthropic's contextual retrieval approach for improving RAG accuracy in RHOAI 3.5.

### Scope

#### 1. POC Implementation

Implement and test Docling's hierarchical chunking approach:

- **HierarchicalChunker**: Structure-aware chunking directly from `DoclingDocument` (preserves headings, sections, tables, lists)
- **`contextualize()` method**: Enriches chunks with metadata (section path, heading hierarchy, document context)
- **HybridChunker** (optional): Adds tokenizer-aware split/merge on top of hierarchical chunks

#### 2. Benchmarking

Run AutoRAG optimization pipeline with test datasets comparing:

**Approach A: Docling Hierarchical + Contextualize**
- Chunking: `HierarchicalChunker` with `contextualize()` enabled
- Retrieval: Hybrid search (vector + BM25 with RRF)

**Approach B: Contextual Retrieval**
- Chunking: Recursive chunking (current default)
- Retrieval: Contextual retrieval (LLM-generated context prepended before embedding)

**Approach C: Baseline**
- Chunking: Recursive chunking
- Retrieval: Hybrid search (no contextual enhancement)

#### 3. Evaluation Metrics

Compare approaches using:
- Retrieval accuracy (precision, recall, MRR)
- Failed retrieval rate reduction (%)
- Indexing time and cost (LLM calls, tokens)
- Query-time latency
- Storage overhead (embedding size, metadata)

### Expected Outcomes

- **Quantitative comparison**: Which approach provides better retrieval accuracy?
- **Cost-benefit analysis**: Trade-offs between ingestion cost and accuracy improvement
- **Recommendation**: Should RHOAI 3.5 support both approaches, or prioritize one?

### Test Datasets

- Technical manuals (structured PDFs with tables, sections)
- Research papers (hierarchical documents)
- Legal/compliance documents (long-form, multi-section)

### Success Criteria

- POC demonstrates Docling hierarchical chunking can be integrated into AutoRAG pipeline
- Benchmark results show measurable accuracy improvement vs baseline
- Clear recommendation on whether to include hierarchical chunking in RHOAI 3.5 alongside contextual retrieval

### References

- [Docling Chunking Concepts](https://docling-project.github.io/docling/concepts/chunking/)
- [Anthropic Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) (49-67% failed retrieval reduction)
- [AutoRAG Chunking Methods Documentation](https://github.com/red-hat-data-services/architecture-decision-records/blob/main/documentation/components/autorag/features/chunking_and_retrievals_methods.md)

---

## Instructions to Create in Jira

1. Go to https://redhat.atlassian.net/browse/RHOAIENG-60180
2. Click "Create" or use the "+" button in the top navigation
3. Select Project: **RHOAIENG**
4. Issue Type: **Task** (or **Story** if your team uses stories for this work)
5. Copy the Summary and Description sections above
6. Set appropriate:
   - Assignee
   - Sprint/Epic
   - Priority
   - Labels (e.g., `autorag`, `poc`, `benchmarking`, `docling`)
7. Link to RHOAIENG-60180 if related
