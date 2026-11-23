# ADR Plugin-Based Catalog Architecture and catalog-gen Tool

|                |            |
| -------------- | ---------- |
| Date           | July 2026 |
| Scope          | AI Hub |
| Status         | Approved |
| Authors        | [Alessio Pragliola](@Al-Pragliola) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [kubeflow/hub#2220](https://github.com/kubeflow/hub/issues/2220) |
| Other docs     | Companion PR: [kubeflow/hub#2219](https://github.com/kubeflow/hub/pull/2219) |

## What

The Model Catalog is refactored into a plugin-based architecture where each AI asset type (models, MCP servers, agents, datasets, etc.) is a self-contained plugin running inside a unified catalog server. A code generation tool, `catalog-gen`, scaffolds new plugins from CLI flag definitions so that adding a new catalog type is primarily a schema definition exercise.

## Why

The AI ecosystem is expanding beyond models. Users need to manage MCP servers, AI agents, datasets, prompt templates, evaluation benchmarks, and other asset types. Building a separate service for each asset type would duplicate infrastructure (database access, filtering, pagination, configuration, health checks) and increase operational complexity.

Two constraints shaped the design:

1. **Zero breaking changes** for existing Model Catalog consumers. All current API paths, schemas, and behaviors must be preserved.
2. **Minimal effort to add new catalog types.** Adding a new AI asset should require defining a schema and implementing data providers, not rebuilding infrastructure.

## Goals

* Evolve the catalog into a generic, extensible platform where each asset type is a self-contained plugin
* Preserve all existing Model Catalog API paths and schemas without modification
* Provide a shared framework for filtering, pagination, data loading, and source management across all catalog types
* Build a code generation tool (`catalog-gen`) that scaffolds complete plugins from CLI flag definitions
* Validate the architecture by implementing MCP Server and Agent catalog types as plugins

## Non-Goals

* Changing the Model Catalog's API surface or behavior
* Runtime dynamic plugin loading (plugins are compiled in via Go blank imports)
* Supporting plugins written in languages other than Go
* Multi-process or microservice-based plugin isolation (all plugins run in-process)
* Replacing the existing database layer or storage backends

## How

### Unified Catalog Server

All plugins run inside a single `catalog-server` process. Each plugin owns its API routes, database tables, data providers, and OpenAPI specification. Plugins share the database connection, configuration system, HTTP server, and leader election infrastructure. If any plugin fails during `Init()`, route mounting, or `Start()`, the server aborts startup and stops already-started plugins in reverse order. Each plugin tracks its own health status independently, and the server's `/readyz` endpoint reflects the aggregate health of all registered plugins.

```text
┌──────────────────────────────────────────────────────────────┐
│                      catalog-server                          │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Model     │  │    MCP      │  │   Agent     │  ...     │
│  │   Plugin    │  │   Plugin    │  │   Plugin    │          │
│  │             │  │             │  │             │          │
│  │ /api/model_ │  │ /api/mcp_   │  │ /api/agent_ │          │
│  │ catalog/v1  │  │ catalog/v1  │  │ catalog/v1  │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                  │
│  ┌──────┴────────────────┴────────────────┴──────┐          │
│  │          Shared Database (GORM)               │          │
│  │     SQLite / MySQL / PostgreSQL               │          │
│  └───────────────────────────────────────────────┘          │
│                                                              │
│  Health: /healthz, /readyz    Plugins: /api/plugins          │
└──────────────────────────────────────────────────────────────┘
```

### Plugin Interface

Every plugin implements a core `CatalogPlugin` interface with lifecycle methods:

```go
type CatalogPlugin interface {
    Name() string
    Version() string
    Description() string
    Init(ctx context.Context, cfg Config) error
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Healthy() bool
    RegisterRoutes(router chi.Router) error
    Migrations() []Migration
}
```

Seven optional interfaces allow plugins to customize behavior: overriding the API base path, specifying a configuration key in `sources.yaml`, registering CLI flags, participating in leader election, contributing datastore entities, and refreshing state after database reconnection.

### Plugin Lifecycle

Plugins register at compile time via Go `init()` functions. A blank import in `catalog/cmd/catalog.go` is all that's needed to activate a plugin. At runtime, the server:

1. Calls `Init()` on each plugin with the shared database, config paths, and repository set
2. Mounts each plugin's HTTP routes under its base path
3. Calls `Start()` to begin background operations (file watching, parsing `sources.yaml`, hot-reload)
4. On leader election, applies database migrations, reconnects plugins, and notifies `LeaderAware` plugins for write operations (data loading, source cleanup)
5. On shutdown, calls `Stop()` in reverse registration order

### Shared PluginBase

A `PluginBase` struct provides reusable lifecycle implementations. Concrete plugins embed it and configure domain-specific loaders and repositories during `Init()`. The base handles file watching for `sources.yaml` hot-reload, leader-aware data loading, health status tracking, and source ID collection.

### Generic Catalog Framework

Shared building blocks in `catalog/internal/catalog/basecatalog/` provide:

* **Loader**: Generic data loader that fetches entities and artifacts from multiple sources concurrently, persists via callbacks, and supports hot-reload with file watching
* **Provider Registry**: Typed provider system where each provider type (YAML, HTTP) registers a factory function
* **Source Configuration**: Shared source definition with include/exclude glob patterns, property bags, and enable/disable state
* **Filter Engine**: SQL-like `filterQuery` parameter on all list endpoints, supporting comparison operators, pattern matching, set membership, and logical combinators
* **Pagination**: Consistent page-token-based pagination and ordering via `BaseResourceList` response envelope

### Configuration

Plugins are configured via a unified `sources.yaml` file:

```yaml
model_catalogs:
  - id: "huggingface"
    type: "yaml"
    properties:
      yamlCatalogPath: "./data/models.yaml"

mcp_catalogs:
  - id: "internal-servers"
    type: "yaml"
    properties:
      yamlCatalogPath: "./data/mcp-servers.yaml"

agent_catalogs:
  - id: "community-agents"
    type: "yaml"
    properties:
      yamlCatalogPath: "./data/agents.yaml"
```

Each plugin reads only its own section. Validation enforces unique source IDs across all sections.

### OpenAPI Spec Strategy

Each plugin owns its OpenAPI specification, co-located under `api/openapi/src/plugins/` (e.g., `model.yaml`, `mcp.yaml`, `agent.yaml`). Shared schemas (`BaseResource`, `BaseResourceList`, `MetadataValue`) live in a central `common.yaml` at `api/openapi/src/lib/`. Plugin schemas compose with `BaseResource` via `allOf`, inheriting standard fields without duplication.

A merge script combines all plugin specs into a unified `catalog.yaml` by auto-discovering plugin specs, merging shared libraries, and re-ordering keys for deterministic output. Per-plugin assembly scripts produce standalone specs for server code generation. The merge process is purely additive: new plugins add paths under their own base URL without modifying existing paths.

### catalog-gen Code Generator

`catalog-gen` is a CLI tool that scaffolds new plugins deterministically from CLI flags (e.g., `--name`, `--entity Name:type`). From a single command, it generates:

* Plugin lifecycle code (`plugin.go`, `register.go`)
* Entity models using generic `BaseEntity` interfaces with dynamic schema registration via `DatastoreEntries()`
* Type-safe repositories using `GenericRepository`
* OpenAPI specification with `allOf` composition against `BaseResource`
* Per-plugin OpenAPI server generation script
* Filter field-to-column mappings
* Loader scaffold with placeholders for data provider implementation

All generated files are create-once: the generator skips any file that already exists, ensuring re-runs are safe and custom business logic is never overwritten. Data providers (e.g., YAML loaders) are not generated and must be implemented by hand.

The tool also modifies shared files: adding the new source type to `SourceConfig`, the validation call, and the blank import to `catalog/cmd/catalog.go`.

### Validation

The architecture was validated by implementing two non-model plugins:

* **MCP Plugin**: Serves MCP server metadata at `/api/mcp_catalog/v1alpha1/mcpservers` with full filtering, pagination, and source management
* **Agent Plugin**: Serves AI agent metadata at `/api/agent_catalog/v1alpha1/agents` with template artifacts

Both were scaffolded via `catalog-gen` and follow the same lifecycle, configuration, and API patterns as the model catalog.

## Alternatives

### Separate microservices per asset type

Each catalog type would be its own service with its own database, deployment, and API. This provides strong isolation but duplicates significant infrastructure (filtering, pagination, configuration, health checks, leader election). Operational cost scales linearly with the number of asset types. Cross-catalog queries (e.g., showing models alongside their MCP servers) require service-to-service calls. Rejected because the duplication outweighs the isolation benefits for catalog-style read-heavy workloads.

### Monolithic approach with type switches

Keep a single codebase but use type switches or conditional logic to handle different asset types. This avoids the plugin abstraction but makes the codebase increasingly tangled as asset types grow. Adding a new type requires modifying existing code paths throughout the stack. Rejected because it doesn't scale in maintainability and violates the open-closed principle.

### Dynamic plugin loading at runtime

Load plugins as shared libraries (Go plugins) or via RPC at runtime without recompilation. This provides maximum flexibility but Go's plugin system is fragile (requires identical build environments), adds complexity for debugging, and makes dependency management harder. Rejected because compile-time registration via blank imports is simpler, safer, and sufficient for the expected rate of new catalog types.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| ----- | ------------ | ---- | --------- |
| AI Hub | AI Hub team | July 2026 | Yes |

This is an internal refactoring of the catalog server. The plugin boundary is an implementation detail: no external APIs change, no new CRDs are introduced, and deployment manifests are unaffected. Only the AI Hub team, which owns the catalog server codebase, is impacted.

## References

* GitHub proposal: [kubeflow/hub#2220](https://github.com/kubeflow/hub/issues/2220)
* Unified plugin server: [kubeflow/hub#2724](https://github.com/kubeflow/hub/pull/2724)
* Per-domain plugin split: [kubeflow/hub#2751](https://github.com/kubeflow/hub/pull/2751)
* Per-plugin OpenAPI split: [kubeflow/hub#2735](https://github.com/kubeflow/hub/pull/2735)
* catalog-gen tool: [kubeflow/hub#2762](https://github.com/kubeflow/hub/pull/2762)
* Agent plugin scaffolding: [kubeflow/hub#2887](https://github.com/kubeflow/hub/pull/2887)
* Agent artifacts endpoint: [kubeflow/hub#2928](https://github.com/kubeflow/hub/pull/2928)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
