# AutoRAG Chunking and Retrieval Methods

This page documents the **chunking strategies** and **retrieval methods** supported by AutoRAG optimization pipeline for RHOAI, based on the [`documents_rag_optimization_pipeline`](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) and ai4rag implementation.

## Table of Contents

- [Chunking Methods](#chunking-methods)
  - [Recursive Chunking](#recursive-chunking)
  - [Docling: Markdown export vs native chunking](#docling-markdown-export-vs-native-chunking)
  - [Extraction persistence and optimization flow](#extraction-persistence-and-optimization-flow)
  - [LLM contextual enrichment (index time, RHOAI 3.5+)](#llm-contextual-enrichment-index-time-rhoai-35)
  - [Chunking parameters reference](#chunking-parameters-reference)
- [Retrieval Methods](#retrieval-methods)
  - [Simple Retrieval](#simple-retrieval)
  - [Hybrid Search](#hybrid-search)
  - [Retrieval Parameters](#retrieval-parameters)
  - [Optimization Workflow](#optimization-workflow)
- [Related Documentation](#related-documentation)
- [External Resources](#external-resources)

---

## Chunking Methods

Chunking splits source documents into smaller segments (chunks) that fit within embedding model and LLM context windows while preserving semantic coherence. AutoRAG supports multiple chunking strategies during optimization to find the best approach for your documents.

**Where settings live:** boundaries and **Docling** serialization (`include_context`) are **`chunking`** fields. **[Anthropic-style contextual retrieval](https://www.anthropic.com/news/contextual-retrieval)** (an LLM writes short document-situating text **before** embedding) is also configured under **`chunking`** as **`contextual_enrichment`**, because it runs during **indexing** (documents indexing pipeline / vector store build), not during the query-time retrieval step. Query behavior (`number_of_chunks`, `search_mode`, fusion) stays under **`retrieval`**.

### Recursive Chunking

**Recursive chunking** divides text using a cascade of separators (`\n\n` → `\n` → ` ` → `""`), preserving document structure while ensuring chunks fit size limits. This is the default chunking method in AutoRAG.

**Configuration:**
```json
{
  "chunking": {
    "method": "recursive",
    "chunk_size": 2048,
    "chunk_overlap": 256
  }
}
```

**Use for:** Structured documents (manuals, technical guides), general-purpose RAG (69% accuracy, [benchmark 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag))

**Pros:** Preserves structure, fast, deterministic | **Cons:** May split mid-concept for dense paragraphs

### Docling: Markdown export vs native chunking

Documents in the **documents RAG optimization** flow are parsed with **[Docling](https://docling-project.github.io/docling/)**. Chunking parameters in **`pattern.json`** (for example `recursive` with `chunk_size` / `chunk_overlap`) describe how **text is split for indexing and retrieval** after content is available for the RAG stack.

Official Docling documentation distinguishes **two** chunking strategies in principle ([Chunking concepts](https://docling-project.github.io/docling/concepts/chunking/)):

| Approach | Idea | Typical trade-offs |
|----------|------|-------------------|
| **Markdown (or similar) export, then post-processing** | Serialize **`DoclingDocument`** to Markdown (or related text) and run **user-defined** splitting (for example LangChain-style recursive splitters on the string). | Simple to wire into existing RAG stacks; easy to inspect. Structure (sections, tables, list groupings, doc item references) may be **flattened or weakened**, so downstream chunk boundaries may not align with Docling’s native document elements. Example recipe: [RAG with LangChain / Markdown-oriented flow](https://docling-project.github.io/docling/examples/rag_langchain/). |
| **Native Docling chunkers on `DoclingDocument`** | Call chunkers that **iterate over the parsed document model**, not over a single pre-flattened string. Native chunkers use detected structure (headings, captions, elements; list merging is configurable) and can apply tokenizer-aware split/merge aligned with embedding limits. | **Structure-aware** chunks and **`contextualize()`** for metadata-enriched strings for embeddings. **`BaseChunker`** allows **custom or third-party** implementations and is the integration surface for frameworks such as LlamaIndex (same concepts page). Example: [Advanced chunking and serialization](https://docling-project.github.io/docling/examples/advanced_chunking_and_serialization/). |

**Implication for AutoRAG:** the optimization surface documented on this page (`recursive`, token limits, `pattern.json`) applies to **text fed into the vector store** after Docling extraction. AutoRAG today optimizes **recursive** splitting on exported Markdown (or flat text). Docling’s **native structure-aware chunkers** remain an upstream Docling capability; they are not separate `chunking.method` values in the AutoRAG search space documented here.

### Extraction persistence and optimization flow

This section describes the **recommended end-to-end shape** for [`pipelines-components`](https://github.com/red-hat-data-services/pipelines-components): parse once, persist **`DoclingDocument`**, then run optimization trials that **export and recursively split** text without re-parsing PDFs.

#### 1. Text extraction persists `DoclingDocument` (JSON or YAML)

After Docling parses a source file, the **`text_extraction`** step should write a **canonical serialized document** per input (object store prefix or shared workspace, then upload). Use **docling-core** APIs so reload round-trips through the same schema.

**Save to JSON**

```python
from pathlib import Path
from docling_core.types.doc.base import ImageRefMode
from docling_core.types.doc.document import DoclingDocument

doc: DoclingDocument = ...  # e.g. result.document from DocumentConverter

out_path = Path("/workspace/parsed/<doc_id>/docling.json")
doc.save_as_json(
    out_path,
    artifacts_dir=out_path.parent / "artifacts",
    image_mode=ImageRefMode.REFERENCED,  # smaller JSON; ship `artifacts/` with the file
    indent=2,
)
```

**Save to YAML** (equivalent; choose one format per product convention)

```python
from pathlib import Path
from docling_core.types.doc.base import ImageRefMode
from docling_core.types.doc.document import DoclingDocument

doc: DoclingDocument = ...

out_path = Path("/workspace/parsed/<doc_id>/docling.yaml")
doc.save_as_yaml(
    out_path,
    artifacts_dir=out_path.parent / "artifacts",
    image_mode=ImageRefMode.REFERENCED,
)
```

**Load back** (same document for downstream chunking or export)

```python
from docling_core.types.doc.document import DoclingDocument

doc = DoclingDocument.load_from_json("/workspace/parsed/<doc_id>/docling.json")
# or: doc = DoclingDocument.load_from_yaml("/workspace/parsed/<doc_id>/docling.yaml")
```

**Notes**

- Prefer **`ImageRefMode.REFERENCED`** plus **`artifacts_dir`** for large PDFs so the JSON/YAML stays smaller; the pipeline must preserve **both** the main file and the **artifacts** directory for reloads.
- Optional: gzip the file in object storage; the **loader** task decompresses before `load_from_json` / `load_from_yaml`.
- Record **docling / docling-core version** (or a content hash) in a sidecar manifest so upgrades do not silently break deserialization.

#### 2. ai4rag consumes persisted documents and branches on the search space

**`rag_templates_optimization`** (ai4rag) receives **paths or URIs** to the extracted corpus (for example a **manifest** listing each `doc_id` and its `docling.json` / `docling.yaml` path). It does **not** need to re-parse PDFs for each trial if the persisted **`DoclingDocument`** is the single source of truth.

For **each optimization trial**, ai4rag reads the **search space / template** and applies the trial’s **`chunking.method`** and related parameters. With **`recursive`** chunking (the AutoRAG default), trials start from the persisted **`DoclingDocument`**, call **`export_to_markdown()`** on demand (or read a **Markdown file** produced once during extraction to save CPU), then apply recursive / token splitting with the trial’s **`chunk_size`** / **`chunk_overlap`**.

**Flow summary**

```
text_extraction  →  DoclingDocument.save_as_json|yaml  →  manifest + object store
                                                      ↘
rag_templates_optimization (ai4rag)  →  load DoclingDocument per trial input
                                     →  export_to_markdown (or read pre-derived MD) → recursive split
```

This keeps **one parse per document** while still letting AutoRAG **explore** chunking methods in the optimization loop.

### LLM contextual enrichment (index time, RHOAI 3.5+)

This pattern is often called **contextual retrieval** in blog posts; in **`pattern.json`** it should appear under **`chunking.contextual_enrichment`**, not under **`retrieval`**, because the LLM runs while **building the index** (after chunk boundaries are fixed), not when serving a user query.

**Docling vs LLM:** Docling’s **`contextualize()`** only serializes **structured metadata** plus chunk text (see [Docling chunking concepts](https://docling-project.github.io/docling/concepts/chunking/)). **`contextual_enrichment`** is a **separate** optional step: an LLM writes 1–2 sentences that place the chunk in the **whole document**, then that text is stored and embedded.

**How it works:**
1. **Context generation (at ingestion time)**: For each chunk, use an LLM (e.g., Claude, Granite) to generate a brief description that "situates" the chunk within its source document
2. **Contextual embeddings**: Prepend the generated context to the chunk before embedding, creating richer vector representations
3. **Contextual BM25**: Index the same **enriched** string for sparse search where applicable
4. **Query time**: Use normal **`retrieval`** settings (`search_mode`, top‑k, ranker)—no extra LLM call for context generation per query

**Example context generation prompt:**
```
<document>
{ENTIRE_DOCUMENT}
</document>

Here is the chunk we want to situate within the whole document:
<chunk>
{CHUNK_CONTENT}
</chunk>

Provide a brief 1-2 sentence context that explains what this chunk is about 
within the document. Only output the context, nothing else.
```

**Example output:**
```
Original chunk: "The refund process takes 5-7 business days."

Generated context: "This chunk describes the refund processing timeline 
in the company's customer service policy document."

Stored for embedding: "This chunk describes the refund processing timeline 
in the company's customer service policy document. The refund process takes 
5-7 business days."
```

**Example configuration (RHOAI 3.5):**
```json
{
  "chunking": {
    "method": "recursive",
    "chunk_size": 1024,
    "chunk_overlap": 128,
    "contextual_enrichment": {
      "enabled": true,
      "context_generation_model": "granite-3.1-8b-instruct"
    }
  },
  "retrieval": {
    "method": "simple",
    "number_of_chunks": 5,
    "search_mode": "hybrid"
  }
}
```

**When to use LLM contextual enrichment:**
- **Long, complex documents**: Technical manuals, research papers, legal documents where chunks lose meaning when isolated
- **Multi-section documents**: Books, reports with many chapters/sections
- **Ambiguous content**: Documents where similar text appears in different contexts (e.g., "the system" referring to different systems)
- **High-precision requirements**: Critical applications where retrieval accuracy is paramount
- **Production RAG systems**: Maximum accuracy justifies the additional ingestion cost

**Advantages:**
- ✅ **Significantly better accuracy**: 49-67% reduction in failed retrievals
- ✅ **Rich context**: Chunks include document-level context automatically
- ✅ **No query-time overhead**: Context generated once at ingestion, not per query
- ✅ **Compatible with hybrid search**: Can combine with vector + keyword + RRF
- ✅ **Handles ambiguity**: Disambiguates chunks that would be unclear in isolation

**Limitations:**
- ❌ **Higher ingestion cost**: Requires LLM call per chunk to generate context (mitigated with prompt caching)
- ❌ **Increased storage**: Context text adds to chunk size (typically 1-2 sentences)
- ❌ **Slower indexing**: Context generation adds latency during vector store creation
- ❌ **Requires entire document**: LLM needs full document context to generate chunk context

**Cost optimization:**
- Use **prompt caching** (e.g., Claude prompt caching) to reduce cost of passing entire document multiple times
- Generate contexts in **batch** for all chunks from same document in single LLM call
- Consider **selective contextualization**: Apply only to documents where chunks are likely ambiguous

### Chunking parameters reference

Common **`chunking`** fields (exact names may match the pipeline / ai4rag template version in use):

| Parameter | Description | Notes |
|-----------|-------------|--------|
| `method` | Chunking algorithm | e.g. `recursive` |
| `chunk_size` / `chunk_overlap` | Size and overlap for token- or char-aware splitters | Primary tunables for `recursive` |
| **`contextual_enrichment`** | Object controlling **LLM** situating text before embed | Consumed by **documents indexing** / vector store build |
| **`contextual_enrichment.enabled`** | Turn LLM enrichment on or off | Default off where unsupported |
| **`contextual_enrichment.context_generation_model`** | Model id or endpoint for context sentences | Runs at **ingestion**, not retrieval |

---

## Retrieval Methods

Retrieval methods determine how chunks are selected from the vector store to answer user queries. AutoRAG supports multiple retrieval strategies optimized during pattern evaluation.

### Simple Retrieval

**Simple retrieval** performs vector similarity search, returning the top-k most similar chunks to the query embedding.

**Configuration:**
```json
{
  "retrieval": {
    "method": "simple",
    "number_of_chunks": 5,
    "search_mode": "vector",
    "distance_metric": "cosine"
  }
}
```

**Use for:** Semantic search, well-defined queries, minimal latency requirements

**Pros:** Fast, handles paraphrasing/synonyms | **Cons:** Misses exact keyword matches (product codes, IDs)

### Hybrid Search

**Hybrid search** combines vector similarity search with BM25 keyword search, fused using Reciprocal Rank Fusion (RRF). Provides [15-30% accuracy improvement](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) over single-method approaches.

**Configuration:**
```json
{
  "retrieval": {
    "method": "simple",
    "number_of_chunks": 5,
    "search_mode": "hybrid",
    "ranker_strategy": "rrf",
    "ranker_k": 60,
    "ranker_alpha": 0.5
  }
}
```

**Use for:** Mixed semantic + keyword queries, exact term matching (IDs, codes), production RAG systems ([91% recall@10](https://blog.supermemory.ai/hybrid-search-guide/))

**Pros:** Best of both worlds (semantic + keyword), robust to embedding limitations | **Cons:** Higher latency, requires BM25 index

**Search modes:** `vector` (semantic only), `keyword` (BM25 only), `hybrid` (fused, recommended for production)

### Retrieval Parameters

AutoRAG optimizes these retrieval parameters during pattern evaluation:

| Parameter | Description | Typical Range                        | Default | Impact |
|-----------|-------------|--------------------------------------|---------|--------|
| `method` | Query-time retrieval strategy | `simple` (and future values aligned with the inference stack) | `simple` | How the stack fetches chunks from the vector store at **query** time. **LLM contextual enrichment** is **not** selected here; configure it under **`chunking.contextual_enrichment`**. |
| `number_of_chunks` | Top-k chunks to retrieve | 3-20                                 | 5 | More chunks: more context but potential noise. Fewer chunks: precise but may miss relevant info. |
| `search_mode` | Search type | `vector`, `keyword`, `hybrid`        | `hybrid` | Hybrid provides best accuracy for most use cases. |
| `ranker_strategy` | Ranking fusion algorithm | `rrf`, `weighted` | Not set | RRF (Reciprocal Rank Fusion) recommended for hybrid search. |                                                                                                
| `ranker_k` | RRF constant parameter | 20-100 | Not set | Higher k: more gradual rank decay. Typical: 60. |                                                                                                                                
| `ranker_alpha` | Weighted fusion balance | 0.0-1.0 | Not set | 0.0: pure keyword, 1.0: pure vector, 0.5: balanced. |              
| `distance_metric` | Vector similarity metric | `cosine`, `dot_product`, `euclidean` | `cosine` | Cosine is most common; normalized similarity regardless of vector magnitude. |
---

### Optimization Workflow

AutoRAG automatically tests combinations of these parameters during optimization:

1. **Define search space**: Specify ranges for chunk_size, chunk_overlap, optional **`contextual_enrichment`** (e.g. `enabled` grid), number_of_chunks, search_mode, ranker parameters
2. **Run optimization**: AutoRAG evaluates all combinations against evaluation questions
3. **Measure metrics**: Faithfulness, answer correctness, context precision/recall
4. **Select best pattern**: Highest-scoring combination becomes the recommended pattern
5. **Deploy**: Use optimized `pattern.json` for production vector store creation and inference

**Example optimization search space:**
```json
{
  "chunking": {
    "method": ["recursive"],
    "chunk_size": [512, 1024, 2048],
    "chunk_overlap": [100, 200, 256],
    "contextual_enrichment": {
      "enabled": [false, true],
      "context_generation_model": ["granite-3.1-8b-instruct"]
    }
  },
  "retrieval": {
    "method": ["simple"],
    "number_of_chunks": [3, 5, 10],
    "search_mode": ["vector", "hybrid"],
    "ranker_strategy": ["rrf"],
    "ranker_k": [30, 60, 90],
    "ranker_alpha": [0.3, 0.5, 0.7]
  }
}
```

This generates up to `3 × 3 × 2 × 1 × 3 × 2 × 3 × 3 = 972` combinations before any AutoRAG sampling cap (AutoRAG samples based on `max_combinations` and product-specific search-space rules).

---

## Related Documentation

- [RAG Pattern Inference](./rag_pattern_inference.md) - Pattern deployment and inference workflow
- [RAG Pattern Vector Store Creation](./rag_pattern_vector_store_creation.md) - Vector store indexing with optimized patterns
- [AutoRAG ADR](../../../architecture-decision-records/autorag/ODH-ADR-0001-autorag.md) - Architecture decisions and optimization workflow
- [Documents RAG Optimization Pipeline](https://github.com/red-hat-data-services/pipelines-components/tree/main/pipelines/training/autorag/documents_rag_optimization_pipeline) - Pipeline implementation

## External Resources

### Docling (extraction and chunking model)

- [Chunking concepts — Markdown export vs native `DoclingDocument` chunkers](https://docling-project.github.io/docling/concepts/chunking/)
- [Advanced chunking and serialization](https://docling-project.github.io/docling/examples/advanced_chunking_and_serialization/)

### Chunking Strategies
- [Chunking Strategies for RAG (Weaviate)](https://weaviate.io/blog/chunking-strategies-for-rag)
- [Best Chunking Strategies for RAG in 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)

### Hybrid Search & Ranking
- [Optimizing RAG with Hybrid Search & Reranking (Superlinked)](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking)
- [Understanding Reciprocal Rank Fusion in Hybrid Search](https://glaforge.dev/posts/2026/02/10/advanced-rag-understanding-reciprocal-rank-fusion-in-hybrid-search/)
- [Hybrid Search Guide (April 2026)](https://blog.supermemory.ai/hybrid-search-guide/)
- [Azure AI Search: Hybrid Retrieval and Reranking](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/azure-ai-search-outperforming-vector-search-with-hybrid-retrieval-and-reranking/3929167)

### Contextual Retrieval
- [Anthropic's Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- [Enhancing RAG with Contextual Retrieval (Claude Cookbook)](https://platform.claude.com/cookbook/capabilities-contextual-embeddings-guide)
- [Contextual Retrieval Guide (DataCamp)](https://www.datacamp.com/tutorial/contextual-retrieval-anthropic)
- [Implementing Contextual Retrieval (Towards Data Science)](https://towardsdatascience.com/implementing-anthropics-contextual-retrieval-for-powerful-rag-performance-b85173a65b83/)
- [Building Production RAG with Contextual Retrieval](https://medium.com/@reliabledataengineering/building-production-rag-with-anthropics-contextual-retrieval-complete-python-implementation-f8a436095860)
