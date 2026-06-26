# Chat Completions vs Responses — Prompt Mapping

Reference: how HPO **Chat Completions** prompts map to exported **`responses_template`** JSON, what **OGX injects at runtime**, and template-family variations. Registered model families (sections 1–10) share one export transformation—see the [consolidated table](#registered-model-families); edge-case sections retain full before/after detail.

**Current code:** `build_pattern_json()` uses **export parity** via `build_responses_system_input()`. The prior **PR baseline** shape (raw `system_message_text` only, no merge) is described in [Export modes](#export-modes) for comparison.

Source: `ai4rag/search_space/src/model_props.py` + `ai4rag/components/assets_generator/pattern_builder.py`. OGX config: [`benchmarking/rag/config.yaml`](https://github.com/ogx-ai/ogx/blob/main/benchmarking/rag/config.yaml).

## OGX vs export — responsibility split

When patterns run on OGX, `file_search` injects retrieval context **after** the exported `input[]` messages. Export must **not** duplicate those runtime elements.

| Concern | OGX config key | Exported `responses_template` |
|---------|----------------|-------------------------------|
| Tool result wrapper | `file_search_params.header/footer_template` | **Omit** |
| Chunk presentation | `context_prompt_params.chunk_annotation_template` | **Omit** |
| Retrieval framing | `context_prompt_params.context_template` | **Omit** redundant user grounding |
| Citation format | `annotation_prompt_params.annotation_instruction_template` | **Omit** HPO `[1],[2]`; no export citation text |
| Per-chunk cite hint | `annotation_prompt_params.chunk_annotation_template` | **Omit** |
| Model persona | — | **Keep** adapted `system_message_text` |
| Answer length / language | — | **Keep** merged from HPO user template |

## What must be configured in OGX (Responses + file_search)

Exported `responses_template` patterns depend on the OGX **Responses API** and **file_search** tool runtime. Other rag-benchmarks settings (inference, vector_io, embedding, ingestion) are standard OGX distro setup — see [`benchmarking/rag/config.yaml`](https://github.com/ogx-ai/ogx/blob/main/benchmarking/rag/config.yaml).

### 1. Responses API and file_search providers

```yaml
apis:
  - responses          # Responses API endpoint
  - tool_runtime       # file_search execution

providers:
  responses:
    - provider_id: builtin
      provider_type: inline::builtin
  tool_runtime:
    - provider_id: file-search
      provider_type: inline::file-search
```

### 2. file_search prompt injection (do not duplicate in export)

**Critical:** OGX injects retrieval/citation text from `vector_stores` at runtime. Exported `responses_template.input[system]` must **not** repeat these keys.

```yaml
vector_stores:
  file_search_params:
    header_template: |
      file_search tool found {num_chunks} chunks:

      BEGIN of file_search tool results.

    footer_template: |
      END of file_search tool results.

  context_prompt_params:
    chunk_annotation_template: |
      Result {index}
      Content: {chunk.content}
      Metadata: {metadata}

    context_template: |
      The above results were retrieved to help answer the user's
      query: "{query}". Use them as supporting information only in
      answering this query. {annotation_instruction}

  annotation_prompt_params:
    enable_annotations: true            # MUST be true for <|file-id|> citations
    annotation_instruction_template: |
      Cite sources immediately at the end of sentences before punctuation,
      using `<|file-id|>` format like 'This is a fact <|file-Cn3MSNn72ENTiiq11Qda4A|>.'
      Do not add extra punctuation. Use only the file IDs provided,
      do not invent new ones.
    chunk_annotation_template: |
      [{index}] {metadata_text} cite as <|{file_id}|>
      {chunk_text}
```

| OGX key | Required | If missing or wrong |
|---------|----------|---------------------|
| `enable_annotations` | `true` | No `<\|file-id\|>` citation instructions |
| `annotation_instruction_template` | `<\|file-id\|>` format | Conflicts with HPO `[1],[2]` if duplicated in export |
| `context_template` | `{query}` + `{annotation_instruction}` | Weaker retrieval framing |
| `file_search_params` header/footer | non-empty | Unstructured tool output |

### 3. Pattern fields bound at deploy time

Each exported pattern supplies (via `build_pattern_json()`):

| `responses_template` field | Role |
|------------------------------|------|
| `tool_choice.type: file_search` | Force tool call before answer |
| `tools[0].type: file_search` | Enable retrieval tool |
| `tools[0].vector_store_ids` | Target collection |
| `tools[0].max_num_results` | Chunk count (`retrieval.number_of_chunks`) |
| `tools[0].ranking_options` | RRF / weighted ranker from HPO retrieval params |
| `input[1]` user placeholder | End-user question at inference |
| `include: [file_search_call.results]` | Return retrieved chunks in response |

## High-level mapping

| Chat Completions (HPO) | Responses export (`responses_template`) | OGX at runtime |
|------------------------|----------------------------------------|----------------|
| `messages[0]` → `system_message_text` | `input[0]` → `build_responses_system_input()` | — |
| `messages[1]` → formatted `user_message_text` | `input[1]` → `<user_query_placeholder>` | — |
| `{reference_documents}` in user message | **Not exported** | `file_search` + chunk templates |
| `[1], [2]` citation instructions | **Dropped** from export | `annotation_prompt_params` (`<\|file-id\|>`) |
| `temperature`, `max_completion_tokens` | `temperature`, `max_output_tokens` | — |

**Export rule:** drop HPO citation lines — do not replace with export-side citation text. **Keep in export:** persona, answer scaffold, language policy, persona-level grounding rephrase.

## Transformation rules (export parity)

| HPO / user-template content | Export behavior |
|-----------------------------|-----------------|
| `Answer using ONLY the provided documents.` | Rephrased → `…retrieved via file search.` |
| `Answer ONLY using information from the documents below…` | **Dropped** if system has grounding |
| `You MUST cite sources using [1], [2]…` | **Dropped** — OGX handles citation |
| Granite RAG persona in user template | **Dropped** (duplicates system) |
| `{reference_documents}` | **Not exported** |
| `Question: {question}` | **Not exported** — runtime user `input` |
| `Answer (max 150 words, with citations):` | **Merged** into system |
| English-only / multilingual line | **Merged** into system |

## Side-by-side summary (IBM Granite)

| Piece | Chat Completions | Exported pattern | OGX at runtime |
|-------|------------------|------------------|----------------|
| Persona + grounding | System | System (rephrased) | — |
| User grounding block | User (top) | **Removed** | `context_template` |
| `[1],[2]` citation | User | **Removed** | `annotation_instruction_template` |
| Retrieved documents | User body | **Removed** | `file_search` + chunk templates |
| Question | User | `input[role=user]` | Referenced in `context_template` |
| Answer length + language | User (suffix) | System (merged) | — |

Edge-case variations: citation dedupe → [`test-dedupe-citation`](#test-dedupe-citation); minimal system → [`legacy-prefix`](#legacy-prefix).

## End-to-end walkthrough (IBM Granite, english-only)

Four layers at inference. Layers **0–2** are the export/mapping concern; layer **3** is OGX-only.

### Layer 0 — Chat Completions (HPO runtime)

```text
[system]
You are a retrieval-augmented assistant. Answer using ONLY the provided documents. You are Granite Chat, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behaviour.

[user — filled]
Answer ONLY using information from the documents below. Do not use outside knowledge. If the documents do not contain the answer, say you do not have enough information.
You MUST cite sources using [1], [2], etc. matching the document numbers for every factual claim.

You are a specialized Retrieval Augmented Generation (RAG) assistant. Prioritize correctness and ensure your response is grounded in the documents.

Documents:
Document 1:
Revenue grew 12% in Q3.

Document 2:
The product launched in March 2024.

Question: When did the product launch?

Answer (max 150 words, with citations):
You MUST write your entire answer in English only. Do NOT use any other language, even if the question or documents are in another language. Every word of your answer must be in English.

```

Full `messages[]` JSON: see [IBM Granite (english-only)](#granite-english-only).

### Layer 1 — exported `responses_template.input[system]`

```text
You are a retrieval-augmented assistant. Answer using ONLY information from documents retrieved via file search. You are Granite Chat, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behaviour.

Answer (max 150 words, with citations):
You MUST write your entire answer in English only. Do NOT use any other language, even if the question or documents are in another language. Every word of your answer must be in English.
```

### Layer 2 — exported `responses_template.input[user]` (question at runtime)

```text
When did the product launch?
```

### Layer 3 — OGX-injected after `file_search` (from config, not in export)

```text
file_search tool found 2 chunks:

BEGIN of file_search tool results.

[1] cite as <|file-abc123|>
Revenue grew 12% in Q3.

[2] cite as <|file-def456|>
The product launched in March 2024.

END of file_search tool results.

The above results were retrieved to help answer the user's query: "When did the product launch?". Use them as supporting information only in answering this query. Cite sources immediately at the end of sentences before punctuation, using `<|file-id|>` format like 'This is a fact <|file-Cn3MSNn72ENTiiq11Qda4A|>.' Do not add extra punctuation. Use only the file IDs provided, do not invent new ones.
```

## Sample runtime values (used in variation examples)

**Question:** `When did the product launch?`

**Reference documents (formatted via context template):**

```text
Document 1:
Revenue grew 12% in Q3.

Document 2:
The product launched in March 2024.
```

**Context template (Granite/Llama/default):** `Document {doc_number}:
{document}
`

## Export modes

| Mode | `input[role=system]` | `input[role=user]` | Documents |
|------|----------------------|--------------------|-----------|
| **Chat Completions** | `system_message_text` | formatted `user_message_text` | Inlined in user message |
| **Responses — PR baseline** | raw `system_message_text` only | `<user_query_placeholder>` | `file_search` tool |
| **Responses — export parity** | `build_responses_system_input()` (current `build_pattern_json`) | `<user_query_placeholder>` | `file_search` tool |

## Registered models → template family

| Template family | Registered `model_id` values |
|-----------------|------------------------------|
| Generic default | `unknown-model` |
| IBM Granite | `ibm/granite-3-8b-instruct`, `ibm/granite-3-3-8b-instruct` |
| Meta Llama | `meta-llama/llama-3-1-8b-instruct`, `meta-llama/llama-3-1-70b-instruct`, `meta-llama/llama-3-3-70b-instruct`, `meta-llama/llama-4-maverick-17b-128e-instruct-fp8` |
| Mistral | `mistralai/mistral-small-3-1-24b-instruct-2503`, `mistralai/mistral-medium-2505`, `mistralai/mistral-large` |
| OpenAI GPT-OSS | `openai/gpt-oss-120b` |

Heuristic fallbacks (`granite`/`llama`/`mistral`/`gpt` in name) use the same family templates.

## All variations

### Variation index

| # | Section ID | Title |
|---|------------|-------|
| 1 | `generic-default-english-only` | Generic default fallback (english-only) |
| 2 | `generic-default-multilingual` | Generic default fallback (multilingual) |
| 3 | `granite-english-only` | IBM Granite (english-only) |
| 4 | `granite-multilingual` | IBM Granite (multilingual) |
| 5 | `llama-english-only` | Meta Llama (english-only) |
| 6 | `llama-multilingual` | Meta Llama (multilingual) |
| 7 | `mistral-english-only` | Mistral (english-only) |
| 8 | `mistral-multilingual` | Mistral (multilingual) |
| 9 | `openai-english-only` | OpenAI GPT-OSS (english-only) |
| 10 | `openai-multilingual` | OpenAI GPT-OSS (multilingual) |
| 11 | `test-minimal` | Unit-test minimal pattern |
| 12 | `test-merge-rules` | Custom — merge non-redundant user rules |
| 13 | `test-dedupe-citation` | Custom — citation already in system |
| 14 | `legacy-prefix` | Legacy — minimal system, rules in user template |
| 15 | `openai-cookbook-minimal` | Reference — OpenAI cookbook (no system prompt) |

Sections **1–10** are summarized in [Registered model families](#registered-model-families); **11–15** are edge cases with full examples below.

---

<a id="registered-model-families"></a>

## Registered model families (sections 1–10)

Sections **1–10** in the [variation index](#variation-index) share the **same export transformation** (see [Transformation rules](#transformation-rules-export-parity) and the [end-to-end walkthrough](#end-to-end-walkthrough-ibm-granite-english-only)). Differences are limited to **model persona** (`system_message_text` suffix), **language policy** (english-only vs multilingual answer line), and— for IBM Granite only—an extra RAG persona line in the user template that export drops.

**PR baseline** (`raw system_message_text` only) is identical across families: `input[0]` = unmodified HPO system, `input[1]` = `<user_query_placeholder>`. See [Export modes](#export-modes); do not duplicate per family.

### Shared export transformation

- User grounding block (`documents below`) not copied verbatim into export
- HPO `[1], [2]` citation instructions **dropped** — OGX injects `<|file-id|>` citation via `annotation_prompt_params` at `file_search` runtime
- Citation lines omitted from export (no duplication with OGX file_search annotations)
- System grounding rephrased for tool-based retrieval (persona-level only)
- Documents slot removed from export (OGX `file_search_params` + chunk templates)
- Question slot moved to `input[role=user]` at runtime
- Answer scaffold merged into export system text (HPO-specific, not OGX-injected)

### Per-family differences

| Section ID | Family | Representative `model_id` | Language | Extra system persona (after RAG prefix) | User-only content dropped at export |
|------------|--------|---------------------------|----------|----------------------------------------|-------------------------------------|
| `generic-default-english-only` | Generic default | `unknown-model` | English only | `If the question is unanswerable from the documents, say you cannot answer.` | — |
| `generic-default-multilingual` | Generic default | `unknown-model` | Multilingual | same as english-only | Language line: match question language |
| `granite-english-only` | IBM Granite | `ibm/granite-3-8b-instruct` | English only | Granite Chat / IBM cautious-assistant persona | Granite RAG assistant line in user template |
| `granite-multilingual` | IBM Granite | `ibm/granite-3-8b-instruct` | Multilingual | same persona as english-only | RAG assistant line + multilingual language line |
| `llama-english-only` | Meta Llama | `meta-llama/llama-3-1-8b-instruct` | English only | Helpful / respectful / honest Llama safety persona | — |
| `llama-multilingual` | Meta Llama | `meta-llama/llama-3-1-8b-instruct` | Multilingual | same persona as english-only | Multilingual language line |
| `mistral-english-only` | Mistral | `mistralai/mistral-large` | English only | Same helpful / honest persona as Llama | — |
| `mistral-multilingual` | Mistral | `mistralai/mistral-large` | Multilingual | same persona as english-only | Multilingual language line |
| `openai-english-only` | OpenAI GPT-OSS | `openai/gpt-oss-120b` | English only | Correctness / grounding / refusal template | — |
| `openai-multilingual` | OpenAI GPT-OSS | `openai/gpt-oss-120b` | Multilingual | same persona as english-only | Multilingual language line |

**Language suffixes** merged into export system text:

- **English only:** `You MUST write your entire answer in English only. Do NOT use any other language…`
- **Multilingual:** `You MUST write your entire answer in the same language as the question. Do NOT respond in any other language…`

Full worked example for the most common family:

<a id="granite-english-only"></a>

## IBM Granite (english-only) — canonical example

**Section ID:** `granite-english-only`  
**Model:** `ibm/granite-3-8b-instruct`  
**Language autodetect:** `False`

### Chat Completions (HPO runtime)

#### `messages[0]` — system

```text
You are a retrieval-augmented assistant. Answer using ONLY the provided documents. You are Granite Chat, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behaviour.
```

#### `messages[1]` — user (template)

```text
Answer ONLY using information from the documents below. Do not use outside knowledge. If the documents do not contain the answer, say you do not have enough information.
You MUST cite sources using [1], [2], etc. matching the document numbers for every factual claim.

You are a specialized Retrieval Augmented Generation (RAG) assistant. Prioritize correctness and ensure your response is grounded in the documents.

Documents:
{reference_documents}

Question: {question}

Answer (max 150 words, with citations):
You MUST write your entire answer in English only. Do NOT use any other language, even if the question or documents are in another language. Every word of your answer must be in English.
```

### Responses export — **export parity** (`build_responses_system_input`)

#### `responses_template.input[0]` — system

```text
You are a retrieval-augmented assistant. Answer using ONLY information from documents retrieved via file search. You are Granite Chat, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behaviour.

Answer (max 150 words, with citations):
You MUST write your entire answer in English only. Do NOT use any other language, even if the question or documents are in another language. Every word of your answer must be in English.
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": "You are a retrieval-augmented assistant. Answer using ONLY information from documents retrieved via file search. You are Granite Chat, an AI language model developed by IBM. You are a cautious assistant. You carefully follow instructions. You are helpful and harmless and you follow ethical guidelines and promote positive behaviour.\n\nAnswer (max 150 words, with citations):\nYou MUST write your entire answer in English only. Do NOT use any other language, even if the question or documents are in another language. Every word of your answer must be in English."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

---


## Edge-case variations

<a id="test-minimal"></a>

## Unit-test minimal pattern

**Section ID:** `test-minimal`  
**Model:** `ibm/granite-3-8b-instruct`  
**Language autodetect:** `False`  
**Notes:** From `tests/unit/.../test_pattern_builder.py::_make_pattern`  

### Chat Completions (HPO runtime)

#### `messages[0]` — system

```text
Answer based on context only.
```

#### `messages[1]` — user (template)

```text
Context: {reference_documents}
Q: {question}
```




### Responses export — **export parity** (`build_responses_system_input`)

_Uses `build_responses_system_input()` — adapted persona + HPO supplements; citation/retrieval framing omitted (OGX injects at `file_search` runtime)._

#### `responses_template.input[0]` — system

```text
Answer based on context only.
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": "Answer based on context only."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

### Transformation summary

- Documents slot removed from export (OGX `file_search_params` + chunk templates)
- Question slot moved to `input[role=user]` at runtime
- Export system equals raw system (no merge/rephrase)

<a id="test-merge-rules"></a>

## Custom — merge non-redundant user rules

**Section ID:** `test-merge-rules`  
**Model:** `(custom)`  
**Language autodetect:** `False`  
**Notes:** From `test_export_system_input_merges_non_redundant_user_rules`  

### Chat Completions (HPO runtime)

#### `messages[0]` — system

```text
You are a retrieval-augmented assistant. Answer using ONLY the provided documents.
```

#### `messages[1]` — user (template)

```text
Answer ONLY using information from the documents below. Do not use outside knowledge.
You MUST cite sources using [1], [2], etc.

Documents:
{reference_documents}

Question: {question}

Answer (max 150 words, with citations):
You MUST write your entire answer in English only.
```




### Responses export — **export parity** (`build_responses_system_input`)

_Uses `build_responses_system_input()` — adapted persona + HPO supplements; citation/retrieval framing omitted (OGX injects at `file_search` runtime)._

#### `responses_template.input[0]` — system

```text
You are a retrieval-augmented assistant. Answer using ONLY information from documents retrieved via file search.

Answer (max 150 words, with citations):
You MUST write your entire answer in English only.
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": "You are a retrieval-augmented assistant. Answer using ONLY information from documents retrieved via file search.\n\nAnswer (max 150 words, with citations):\nYou MUST write your entire answer in English only."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

### Transformation summary

- User grounding block (`documents below`) not copied verbatim into export
- HPO `[1], [2]` citation instructions **dropped** — OGX injects `<|file-id|>` citation via `annotation_prompt_params` at `file_search` runtime
- Citation lines omitted from export (no duplication with OGX file_search annotations)
- System grounding rephrased for tool-based retrieval (persona-level only)
- Documents slot removed from export (OGX `file_search_params` + chunk templates)
- Question slot moved to `input[role=user]` at runtime
- Answer scaffold merged into export system text (HPO-specific, not OGX-injected)

<a id="test-dedupe-citation"></a>

## Custom — citation already in system

**Section ID:** `test-dedupe-citation`  
**Model:** `(custom)`  
**Language autodetect:** `False`  
**Notes:** From `test_export_system_input_skips_duplicate_citation_and_keeps_answer_scaffold`  

### Chat Completions (HPO runtime)

#### `messages[0]` — system

```text
You are a retrieval-augmented assistant. You MUST cite sources using [1], [2].
```

#### `messages[1]` — user (template)

```text
You MUST cite sources using [1], [2], etc.

Documents:
{reference_documents}

Question: {question}

Answer (max 150 words, with citations):
You MUST write your entire answer in English only.
```




### Responses export — **export parity** (`build_responses_system_input`)

_Uses `build_responses_system_input()` — adapted persona + HPO supplements; citation/retrieval framing omitted (OGX injects at `file_search` runtime)._

#### `responses_template.input[0]` — system

```text
You are a retrieval-augmented assistant.

Answer (max 150 words, with citations):
You MUST write your entire answer in English only.
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": "You are a retrieval-augmented assistant.\n\nAnswer (max 150 words, with citations):\nYou MUST write your entire answer in English only."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

### Transformation summary

- HPO `[1], [2]` citation instructions **dropped** — OGX injects `<|file-id|>` citation via `annotation_prompt_params` at `file_search` runtime
- Citation lines omitted from export (no duplication with OGX file_search annotations)
- Documents slot removed from export (OGX `file_search_params` + chunk templates)
- Question slot moved to `input[role=user]` at runtime
- Answer scaffold merged into export system text (HPO-specific, not OGX-injected)

<a id="legacy-prefix"></a>

## Legacy — minimal system, rules in user template

**Section ID:** `legacy-prefix`  
**Model:** `(custom)`  
**Language autodetect:** `False`  
**Notes:** From `test_build_responses_system_input_legacy_user_prefix`  

### Chat Completions (HPO runtime)

#### `messages[0]` — system

```text
Short system prefix.
```

#### `messages[1]` — user (template)

```text
Answer ONLY using information from the documents below.
You MUST cite sources using [1], [2].

Context: {reference_documents}

Question: {question}

```




### Responses export — **export parity** (`build_responses_system_input`)

_Uses `build_responses_system_input()` — adapted persona + HPO supplements; citation/retrieval framing omitted (OGX injects at `file_search` runtime)._

#### `responses_template.input[0]` — system

```text
Short system prefix.

Answer ONLY using information from documents retrieved via file search. Do not use outside knowledge. If the retrieved documents do not contain the answer, say you do not have enough information.
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": "Short system prefix.\n\nAnswer ONLY using information from documents retrieved via file search. Do not use outside knowledge. If the retrieved documents do not contain the answer, say you do not have enough information."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

### Transformation summary

- User grounding block (`documents below`) not copied verbatim into export
- HPO `[1], [2]` citation instructions **dropped** — OGX injects `<|file-id|>` citation via `annotation_prompt_params` at `file_search` runtime
- Citation lines omitted from export (no duplication with OGX file_search annotations)
- Documents slot removed from export (OGX `file_search_params` + chunk templates)
- Question slot moved to `input[role=user]` at runtime
- Export system extends raw system with merged supplements

<a id="openai-cookbook-minimal"></a>

## Reference — OpenAI cookbook (no system prompt)

**Section ID:** `openai-cookbook-minimal`  
**Model:** `(external reference)`  
**Language autodetect:** `False`  
**Notes:** Official [File Search Responses cookbook](https://developers.openai.com/cookbook/examples/file_search_responses) uses question-only `input` with no system/instructions.  

### Chat Completions (HPO runtime)

_Not applicable — no system message in this reference pattern._

#### User content at runtime

```text
When did the product launch?
```


### Responses export — **export parity** (`build_responses_system_input`)

_Uses `build_responses_system_input()` — adapted persona + HPO supplements; citation/retrieval framing omitted (OGX injects at `file_search` runtime)._

#### `responses_template.input[0]` — system

```text
(empty — question-only pattern)
```

#### `responses_template.input`

```json
[
  {
    "role": "system",
    "content": [
      {
        "type": "input_text",
        "text": ""
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "<user_query_placeholder>"
      }
    ]
  }
]
```

### Transformation summary

- Question slot moved to `input[role=user]` at runtime
- Export system equals raw system (no merge/rephrase)

## Shared `responses_template` fields (all variations)

Non-prompt fields exported identically by `build_pattern_json()`:

```json
{
  "model": "<generation.model_id>",
  "stream": false,
  "store": false,
  "max_output_tokens": "<generation.max_completion_tokens>",
  "temperature": "<generation.temperature>",
  "tool_choice": {
    "mode": "required",
    "tools": [
      {}
    ],
    "type": "file_search"
  },
  "tools": [
    {
      "type": "file_search",
      "vector_store_ids": [
        "<vector_store_id>"
      ],
      "max_num_results": "<retrieval.number_of_chunks>",
      "ranking_options": "<hybrid RRF | weighted | vector-only default>"
    }
  ],
  "include": [
    "file_search_call.results"
  ]
}
```

---

_Regenerate: `python3 dev_utils/generate_prompt_variations_md.py` → `artifacts/prompt-mapping-comparison.md`_
