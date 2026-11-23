# Open Data Hub - AutoRAG Custom RAG Template

|                |            |
| -------------- | ---------- |
| Date           | 2026-11-23 |
| Scope          | AutoRAG Component |
| Status         | Proposed |
| Authors        | Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAIRFE-3647](https://redhat.atlassian.net/browse/RHAIRFE-3647) · [RHAISTRAT-2946](https://redhat.atlassian.net/browse/RHAISTRAT-2946) |
| Other docs:    | [ODH-ADR-0001](./ODH-ADR-0001-autorag.md) · [ODH-ADR-0003](./ODH-ADR-0003-rag-templates.md) · [ODH-ADR-0004](./ODH-ADR-0004-rag-pattern-inference.md) · [ODH-ADR-0005](./ODH-ADR-0005-rag-pattern-evaluation.md) · [ODH-ADR-0006](./ODH-ADR-0006-mlflow-integration.md) |

## Decision

Provide a dedicated KFP pipeline to evaluate and optimize deployed applications through a provider-neutral Responses API contract, `responses.v1`. Every run supplies one configuration and validates requests against an effective schema. AutoRAG supplies the standard wire schema. The endpoint owner adds any provider constraints or extensions. A request can use one or more tools, including web search, MCP, and custom tools, or run without tools. The endpoint returns a completed text answer in a single synchronous call.

AutoRAG owns request validation, experiment orchestration, evaluation, ranking, artifacts, and MLflow tracking. The endpoint owns tool execution, retrieval and generation, any existing index, and the semantics of every advertised control. A customer-managed proxy can implement the same contract for an existing service.

## Why

Customers can tune services they already run without importing their code or preparing a second corpus. The same contract drives the Dashboard, validation, and replay of the winning request. Separating request compatibility from evaluation evidence lets the same pipeline evaluate endpoints with different tools and makes it clear which metrics each configuration can support.

## Scope

The first version supports independent benchmark questions and a single final text answer per request. Streaming, background jobs, conversation state, client-executed tools, approval or continuation loops, index construction and promotion, arbitrary REST adapters, and customer code execution in AutoRAG are outside scope.

## Configuration

| Pipeline input | Role |
| --- | --- |
| `custom_rag_secret_name` | Authorized RHOAI Connection to the endpoint or customer proxy. Supplies the URI and credentials. |
| `custom_rag_config` | One endpoint configuration, with its base request, schema additions, and evidence rules. |
| `search_space_config` | Optional finite candidate choices for that base request. |
| `test_data_secret_name`, `test_data_bucket_name`, `test_data_key` | Benchmark access and location. See [ADR 0002](./ODH-ADR-0002-experiment-settings.md#benchmark-json). |
| `optimization_metric`, `optimization_max_rag_patterns` | Evaluation objective and candidate budget. Evaluator model bindings and Connections are supplied separately. See [ADR 0005](./ODH-ADR-0005-rag-pattern-evaluation.md). |

### Required endpoint contract

`custom_rag_config` directly describes one mode. An endpoint owner can publish multiple templates for the same endpoint as separate ConfigMaps, GitOps files, or KFP presets. Each run supplies one template and can reuse the same Connection. The base request defines the mode, data sources, and access scope for the experiment.

| Field | Requirement |
| --- | --- |
| `request` | Required complete base payload for the mode. |
| `request_schema` | Optional endpoint-specific constraints on standard fields and the schema for `rag_options` if used. |
| `tool_schemas` | Optional closed schemas keyed by tool type for provider-defined tool types absent from the built-in schema. Each requires a `type` constant matching its key. |
| `evidence` | Optional rules for extracting retrieved passages and their canonical benchmark document keys. |
| `api_version`, `name` | Optional wire profile and display name. The profile defaults to `responses.v1`. AutoRAG derives the contract digest. |

AutoRAG ships an immutable wire schema for `responses.v1`, including standard fields and tool definitions keyed by type. The effective schema combines the built-in schema, declared additional tool types, and endpoint constraints. Constraints can narrow standard fields. `rag_options` is the top-level opt-in extension for proprietary controls and requires a closed schema when used. Unknown fields and tool types are rejected. The Dashboard and pipeline use the same effective schema, including for fixed runs. The endpoint remains responsible for honoring its supported settings and rejecting unsupported combinations.

Endpoint-supplied schemas use a bounded JSON Schema Draft 2020-12 vocabulary with `type`, `properties`, `required`, `additionalProperties: false`, one `items` schema per array, `enum`, `const`, and numeric, string-length, and array-length bounds. Reject unknown keywords and external references. Standard fields retain their wire names. Schema and configuration metadata stay out of the HTTP body.

The Connection controls the endpoint URI and authentication. The endpoint owns credentials for its tools and downstream services. Template settings are JSON body fields. If a downstream service requires proprietary headers, the customer proxy adds them. AutoRAG injects `input`, `stream: false`, and `store: false` after validating the endpoint-owned payload. Those fields are reserved, along with conversation, prior-response, and background controls. A configuration may declare fixed `tool_choice` and `include` values appropriate for its provider and tools.

### Fixed settings and candidate selection

Search entries use JSON Pointers relative to `request`, such as `/temperature` or `/tools/0/max_num_results`. They target existing scalar values. The Dashboard offers eligible fields from the base request and validates choices against the effective schema. A schema `const` or single-value `enum` can freeze an endpoint-specific value.

* `remote_request` maps pointers to finite, non-empty lists of values for independent choices. Expand the Cartesian product in lexicographic pointer order, with earlier pointers as outer dimensions and values in supplied order.
* `remote_request_sets` lists explicit override maps for dependent choices. Read them in supplied order. Each map is one candidate.
* Accept at most one form. Normalize either form to an ordered sequence of override maps. Apply each map to the base request, retain omitted values, and deduplicate identical resulting requests in first-seen order. Use that sequence for preflight and selection within the candidate budget. Large grids may be expanded lazily.

An empty search space evaluates the base request once. In a sweep, the base request is evaluated only if a supplied combination reproduces it. Both cases use the same preflight and artifacts.

AutoRAG rejects pointers into fixed fields and their descendants, and rejects object or array replacements. The built-in profile defines protected paths for each standard tool type. For example, it protects `type`, `server_url`, `require_approval`, `vector_store_ids`, and `filters` wherever they apply under `/tools/<index>/`, as well as top-level `tool_choice` and `include`. Tool count and order, identities, destinations, credentials, approval settings, data-source bindings, and access filters stay fixed. For provider-defined tools and `rag_options`, the owner declares any additional fields that control these boundaries using `const` or single-value `enum` constraints. Endpoint authorization is enforced independently of the schema. AutoRAG validates the base requests and search declarations before preflight, then validates every composed request before submission.

## Wire contract and evaluation

`RemoteRAGTemplate` in ai4rag sends `POST {uri}/v1/responses`. A successful call returns HTTP 200, `object: "response"`, `status: "completed"`, and an unambiguous final assistant message with `output_text`. The client joins that message's text blocks into `answer`. Intermediate tool outputs may accompany the final answer. The endpoint must resolve any tool calls needed to answer before returning. AutoRAG does not execute calls or resume runs.

### Evidence and metric eligibility

The configuration may declare how to read evidence from completed Responses output items. Each rule selects an `output_type` from `output[]` and uses a literal JSON Pointer, `results_path`, relative to that item to locate an array. `text_path` and `document_id_path` are relative to each result and must resolve to non-empty strings. Multiple rules can support different tool output types, with one rule per type. JSON Pointers address existing values. If a tool returns a singleton object or a JSON-encoded string, the endpoint or customer proxy must expose a parsed array before AutoRAG can extract evidence.

The endpoint or proxy returns the benchmark's canonical document key directly at `document_id_path`. AutoRAG copies it unchanged to `answer_contexts[].document_key` and compares it by exact string equality against the row's `correct_answer_document_keys`. A non-empty string with no match counts as a miss for document-hit metrics, even if it comes from a different namespace. If all returned keys miss, the row records zero document hits. A missing, empty, or non-string ID is a contract error. [Benchmark JSON](./ODH-ADR-0002-experiment-settings.md#benchmark-json)

Emit passages in output-item and result order. Preserve duplicates, step boundaries, and source metadata such as scores or queries when provided. Record `context_source: "retrieval_results"` for this evidence. Concatenated results do not establish a global rank or prove which passages generation used.

| Configuration | Eligible objectives |
| --- | --- |
| No evidence rules | Objectives that can be computed from the question, answer, and reference answer. Evaluation rows omit `answer_contexts` to indicate unavailable evidence. |
| Evidence rules declared | Objectives requiring passages or benchmark document identity, provided the benchmark has the required references. A rank-based objective also needs an ordering model the evaluator understands. |

For a configuration that declares evidence, each response must include at least one matching output item. Every matching item must contain a valid results array. An empty array represents a completed retrieval with no matches. Missing evidence, malformed results, and invalid document IDs are contract errors.

Hosted web search commonly returns URL citations and source lists. If those are the only evidence, the configuration cannot support metrics such as Faithfulness or Context Precision. The endpoint or proxy must expose the actual retrieved passages and their benchmark document identities to enable those metrics. [Web search citations and sources](https://developers.openai.com/api/docs/guides/tools-web-search)

AutoRAG computes evaluator scores locally. A composite objective is eligible only if the configuration and benchmark provide the inputs required by all of its constituent metrics. Unavailable usage is recorded as unavailable.

### Preflight and failures

Validate the contract, search space, and metric eligibility locally. Preflight sends the same representative benchmark question for the first 10 requests in the normalized candidate sequence, or every request if the sequence is shorter. A fixed run probes its base request. Preflight checks endpoint access, final-answer conformance, and declared evidence. The pipeline records which candidates were probed and their results.

Request timeouts, concurrency, and retries are pipeline execution settings. Retry transient failures within the execution budget. A candidate that fails preflight is recorded and excluded. Unprobed candidates remain eligible for evaluation. Preflight calls are separate from benchmark scoring and do not count toward `optimization_max_rag_patterns`.

### Benchmark completeness and ranking

Each candidate must return a valid response and produce all scores required by `optimization_metric` for every benchmark row. An exhausted timeout, a refusal that leaves no valid answer, invalid evidence, or a missing required evaluator score makes the candidate incomplete. Retain the row failures and completed row scores for diagnosis. Compute the ranking objective over the full benchmark only. Incomplete candidates receive no ranking score and cannot win, even if 95% of their rows succeeded.

Stop immediately on endpoint authentication or authorization errors or a confirmed contract failure shared by all candidates. A failure attributed to particular request values or benchmark inputs affects that candidate. Continue within budget while eligible candidates remain. If none produces a complete evaluation, surface an unsuccessful run with diagnostics and no winner. A run with at least one complete candidate may rank the complete candidates and reports incomplete candidates separately.

## Artifacts and inference

* `template_id: "custom:remote_responses"` and `inference.kind: "remote_responses"` select remote invocation. The optimization pipeline generates no indexing recipe, indexing notebook, deployment `runtime_spec`, or starter kit.
* `settings.request` records the effective non-secret winning HTTP body without per-question `input`, including any tool configuration and fixed data-source bindings.
* `inference.remote_spec` records the Connection reference, optional display name, and a contract snapshot of the resolved profile, effective request schema, base request, and evidence rules. Compute `contract_digest` as SHA-256 over the snapshot serialized as canonical JSON. The run also retains the effective search space, evaluation rows, and candidate completeness. [JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)
* Optimization, generated inference notebooks, and Dashboard testing use the same remote client. Inference supplies a new question, resolves the authorized Connection, and reuses the saved request. The saved configuration supports replay. Endpoint implementation and data may change independently.
* MLflow uses the parent run and per-pattern child runs from [ADR 0006](./ODH-ADR-0006-mlflow-integration.md). Record the contract digest, optional display name, non-secret settings, metrics, and artifact references.

## Security and privacy

Benchmark questions and instructions go to the selected endpoint. Endpoint access uses authorized Connections and verified TLS. Credentials stay out of request settings, schemas, logs, and artifacts. The endpoint owns downstream tool authorization. Users select a configuration whose data sources and tools are appropriate for the benchmark. Retrieved content follows evaluation-artifact access and retention rules. Model-based evaluators receive the answers and contexts they require.

## Alternatives

| Alternative | Decision |
| --- | --- |
| Hard-code a single `file_search` profile | A small first implementation, but creates an AutoRAG change for each new tool or provider request shape. Use it as a reference configuration. |
| Extend the document-indexing pipeline | A dedicated remote pipeline has fewer inputs because the remote endpoint already owns its data and tools. |
| Execute customer code or tool calls in AutoRAG | Keep tool execution and any protocol translation with the endpoint or its proxy. |

## Examples

The file-search and MCP examples are separate templates. Both can use the same endpoint Connection.

### File search with retrieval evidence

This template uses the built-in schema alone. The endpoint returns the full benchmark key, such as `product-manuals/xr-200-manual.pdf`, in each result's `attributes.doc_id`. `/filename` also works if the endpoint returns the full canonical key in that field. The sweep produces three candidates, including the base request because 10 appears in the list. [Responses API reference](https://developers.openai.com/api/reference/resources/responses/methods/create)

```yaml
custom_rag_config:
  name: warranty_search
  request:
    model: customer-rag
    tools:
      - type: file_search
        vector_store_ids: [warranty-documents]
        max_num_results: 10
    tool_choice: {type: file_search}
    include: [file_search_call.results]
  evidence:
    - output_type: file_search_call
      results_path: /results
      text_path: /text
      document_id_path: /attributes/doc_id
search_space_config:
  apiVersion: autorag.opendatahub.io/v1alpha1
  kind: AutoRAGSearchSpace
  remote_request:
    /tools/0/max_num_results: [5, 10, 20]
```

### MCP with proprietary ranking settings

The endpoint or proxy implements `rag_options`. Each override map supplies a supported pair, and GAM keeps the values in each pair together. MCP tool settings and its destination stay fixed. Select an answer-based objective because this template supplies no evidence rules.

```yaml
custom_rag_config:
  name: catalog_search
  request:
    model: customer-catalog
    tools:
      - type: mcp
        server_label: catalog
        server_url: https://retrieval.example.com/mcp
        require_approval: never
    rag_options: {hybrid_alpha: 0.5, reranker_top_k: 10}
  request_schema:
    properties:
      rag_options:
        type: object
        additionalProperties: false
        required: [hybrid_alpha, reranker_top_k]
        properties:
          hybrid_alpha: {type: number, minimum: 0, maximum: 1}
          reranker_top_k: {type: integer, enum: [10, 20]}
search_space_config:
  apiVersion: autorag.opendatahub.io/v1alpha1
  kind: AutoRAGSearchSpace
  remote_request_sets:
    - /rag_options/hybrid_alpha: 0.3
      /rag_options/reranker_top_k: 10
    - /rag_options/hybrid_alpha: 0.7
      /rag_options/reranker_top_k: 20
```

## Related

* [Responses API reference](https://developers.openai.com/api/reference/resources/responses/methods/create) · [File search](https://developers.openai.com/api/docs/guides/tools-file-search) · [Web search](https://developers.openai.com/api/docs/guides/tools-web-search) · [MCP tools](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)
* [Optimization settings and benchmark identity](./ODH-ADR-0002-experiment-settings.md)
* [RAG pattern inference](./ODH-ADR-0004-rag-pattern-inference.md) · [RAG pattern evaluation](./ODH-ADR-0005-rag-pattern-evaluation.md) · [MLflow integration](./ODH-ADR-0006-mlflow-integration.md)
