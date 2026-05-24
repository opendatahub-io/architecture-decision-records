# Open Data Hub - Consolidate AI Asset Registries on MLflow

|                |            |
| -------------- | ---------- |
| Date           | May 24, 2026 |
| Scope          | OpenShift AI — AI Asset Registry Strategy |
| Status         | Draft |
| Authors        | [Edson Tirelli](@etirelli) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | AI Asset Registry PRD (internal document), MLflow Tiger Team Final Status (internal document), [Skills Registry RFC](https://github.com/mlflow/rfcs/pull/10), [MCP Registry](https://github.com/mlflow/rfcs/tree/main/rfcs/0004-mcp-registry) |

## What

Adopt MLflow as the unified registry backend for all AI asset types in OpenShift AI. Rather than building or maintaining a separate registry per asset type (models, MCP servers, skills, agents, prompts, guardrails, knowledge sources), all asset registries converge on MLflow as the single governance layer, with seamless integration allowing each asset type to define its own metadata, lifecycle, and relationships; it also enables cross-reference between assets for observability, auditing, evaluation, and automation.

## Why

As OpenShift AI expands, the platform must govern an increasing number of asset types — models, agents, MCP servers, skills, prompts, guardrails, and knowledge sources. Without a consolidation strategy, each asset type gets its own registry with its own governance policies and maintenance burden. This leads to:

- **Governance fragmentation** — inconsistent lifecycle management, access policies, and auditing across asset types.
- **Duplicated infrastructure** — each registry reimplements versioning, tagging, aliasing, workspace scoping, and RBAC from scratch.
- **No cross-asset visibility** — an agent references models, prompts, tools, and guardrails, but if each is in a separate registry, there is no unified view of what an agent depends on and whether those dependencies are approved for production.
- **Increased operational burden** — platform administrators must learn and operate multiple registry systems, each with different APIs and configuration models.

MLflow is already the strategic platform for experiment tracking, tracing, and evaluation in OpenShift AI. It natively provides model registry and prompt registry capabilities. The upstream MLflow community (Databricks) is actively investing in expanding registry capabilities to cover skills, MCP servers, agents and other asset types. Consolidating on MLflow avoids duplicating this investment and aligns Red Hat's product direction with upstream momentum.

The MLflow Tiger Team (January–March 2026) validated the integration architecture and confirmed that MLflow is the right foundation for registry consolidation. The registries sync (May 4, 2026) formalized the decision to centralize MCP server, skill, and agent registries in MLflow, with sign-off from architecture stakeholders.

## Goals

* Establish MLflow as the single registry backend for all AI asset types in OpenShift AI.
* Provide unified governance — lifecycle management, versioning, access policies, and auditing — across all asset types through a consistent set of MLflow APIs and patterns.
* Maintain a federated registry solution where each asset type defines its own metadata schema, lifecycle states, and validation rules while sharing common infrastructure.
* Preserve the separation between registry (governance and lifecycle) and catalog (discovery and consumption), with AI Hub remaining the catalog layer.
* Enable cross-asset relationship tracking so that composed assets (e.g., agents referencing models, tools, and guardrails) can express and govern their dependency graphs.
* Align with upstream MLflow development to leverage community investment and minimize downstream-only maintenance.

## Non-Goals

* **Prescribing the migration path for existing registries** — the migration from Kubeflow Model Registry to MLflow Model Registry is a separate, implementation-level decision that will be documented in its own ADR.
* **Defining the complete data model for each asset type** — individual asset type registries (MCP servers, skills, agents) will have their own design documents and, where warranted, their own ADRs.
* **Replacing AI Hub** — AI Hub remains the catalog and discovery layer. This ADR concerns the governance backend, not the consumption experience.
* **Building a new abstraction layer above MLflow** — the intent is to use MLflow directly as the registry backend with asset-type plugins, not to build a separate framework that wraps MLflow behind another API surface.

## How

### Architectural Principles

**1. Registry–Catalog Separation**

Registries and catalogs serve complementary but distinct functions. The registry handles governance: ownership, versioning, lifecycle state, access policies, approval status, and audit trails. The catalog is the vehicle for Red Hat to distribute approved, tested, and verified assets, and to provide users with a way to handle consumption: discovery, browsing, understanding, and adopting assets.

These responsibilities must remain architecturally separate. MLflow is the registry. Users import curated artifacts from the catalog (e.g., AI Hub) into their workspace registry for governance and lifecycle management; the two systems are not merged.

This principle is already present in the current architecture and must be preserved through the consolidation.

**2. Metadata-First Design**

The registry stores metadata and references, not the assets themselves. Models live in object storage. MCP servers are installed from OCI images, npm, or PyPI. Skills are bundled as artifacts. The registry records what exists, who owns it, what version it is, what lifecycle state it is in, and where to find the actual asset — following the same pattern MLflow uses for experiment tracking.

**3. Federated Plugin Model**

Each asset type participates in the shared registry infrastructure and defines:

- **Metadata schema** — what fields are required and optional for this asset type.
- **Lifecycle states** — what progression an asset follows (e.g., draft → candidate → published → deprecated → retired).
- **Relationships** — how this asset type relates to others (e.g., an agent references models, prompts, and MCP servers).
- **Validation rules** — what invariants must hold (e.g., a published asset must have an approved status).

Each asset type encodes its domain-specific knowledge without forcing all assets through a generic interface, while sharing common infrastructure for versioning, tagging, aliasing, workspace scoping, and RBAC.

**4. Upstream-First Development**

Registry capabilities should be developed upstream in MLflow wherever possible. Red Hat contributes features that benefit the broader community and consumes them in the product. Downstream-only extensions are limited to enterprise integration points (e.g., OpenShift RBAC integration, operator-managed deployment) that are not appropriate for upstream.

### Asset Type Coverage

The following non-exhaustive list of asset types are examples of in scope assets for registry consolidation, in no particular order:

| Asset Type | Current State | Consolidation Path |
|-----------|---------------|-------------------|
| Models | Kubeflow Model Registry (existing) | Migrate to MLflow Model Registry (separate ADR) |
| Prompts | MLflow Prompt Registry (existing upstream) | Already consolidated |
| Skills | MLflow Skill Registry (upstream RFC in progress) | Adopt upstream |
| MCP Servers | MLflow MCP Registry (upstream RFC in progress) | New MLflow MCP Registry |
| Agents | No registry | New MLflow registry (design pending) |
| Guardrails | No registry | Future — design pending |
| Knowledge Sources | No registry | Future — design pending |

## Alternatives

### 1. Continue Building Separate Registries Per Asset Type

Each new asset type (MCP servers, skills, agents) gets its own independent registry with its own data model, API, storage, and governance implementation.

**Pros:**
- Each registry can be developed and evolve independently.
- No coordination overhead between registry implementations.

**Cons:**
- Governance policies may become inconsistent across asset types — different lifecycle models, different access control, different auditing.
- Cross-asset relationships are complex to express and govern — an agent's dependencies on models, tools, and prompts are invisible to the platform. Cross-asset relationships create cross-registry dependencies defeating the benefit of independent development.
- Every registry reimplements versioning, tagging, workspace scoping, and RBAC — substantial duplicated effort.
- Operational burden increases exponentially with the number of asset types.

**Decision: Rejected.** The governance fragmentation and duplicated effort outweigh the flexibility gains, especially as the number of asset types grows.

### 2. Build a New Unified Registry Framework Above MLflow

Create a new abstraction layer — a "meta-registry" — that provides a unified API surface above multiple backend registries (MLflow, Kubeflow, future backends). Each backend is a pluggable adapter behind the unified interface.

**Pros:**
- Backend-agnostic — not locked into MLflow.
- Could incorporate non-MLflow registries without migration.
- Clean separation between the unified interface and backend specifics.

**Cons:**
- Adds a translation layer that introduces latency and impedance mismatch — every query passes through an abstraction that must reconcile different backend semantics.
- Requires defining and maintaining a universal schema that is the lowest common denominator of all backends.
- MLflow is already absorbing the registry capabilities we need (models, prompts, skills, MCP servers). Building an abstraction above it duplicates the integration surface without adding value.
- The abstraction must evolve every time a backend changes, creating an ongoing maintenance tax.
- Engineering investment goes into plumbing rather than registry capabilities that benefit users.
- The meta-registry is a new abstraction layer with no production track record, adding reliability and operability risk on top of MLflow, which is already a hardened registry foundation in the product.

**Decision: Rejected.** Since MLflow is already the strategic platform and is actively expanding its registry capabilities upstream, building an abstraction above it adds complexity without proportional benefit.

## Architectural Principles — Consequences

Adopting this decision establishes the following architectural constraints for OpenShift AI:

1. **New AI asset types requiring governed lifecycle management must use MLflow as the registry backend.** Teams must not introduce new standalone registries without an explicit exception approved through the ADR process.

2. **Registry and catalog remain separate systems.** Components that need discovery and browsing capabilities should adopt a catalog solution. Components that need governance and lifecycle management should adopt the registry solution defined in this ADR. 

3. **Asset type plugins must follow established MLflow patterns.** New registry plugins use the entity model, store interface, API surface, and workspace scoping patterns established by the MLflow project. Deviations require justification.

4. **Upstream-first contribution is the default.** Registry capabilities are developed upstream unless they are Red Hat-specific enterprise integration points. Downstream-only forks of registry functionality are discouraged.

## Security and Privacy Considerations

- **Executable content** — Some registered assets (skills, MCP servers) are or reference executable code. The registry must integrate with product security review workflows before these assets can be promoted to production lifecycle states. 

- **Access control** — Registry RBAC must integrate with OpenShift RBAC to ensure that workspace-level access policies in MLflow align with namespace-level permissions in the cluster. Misalignment would allow users to register or promote assets they should not have access to.

## Risks

- **Upstream dependency** — This decision ties the registry strategy to MLflow's upstream roadmap. If Databricks deprioritizes registry capabilities or takes them in a direction incompatible with OpenShift AI's needs, Red Hat must either influence the upstream direction or maintain downstream forks. *Mitigation*: Active upstream engagement is already established (Red Hat engineers as members of the MLflow maintainers team, RFC contributions, direct collaboration with Databricks).

- **Migration complexity** — Existing assets in the Kubeflow Model Registry must eventually be migrated to MLflow. This is a non-trivial effort involving data migration, API changes, dashboard updates, and customer communication. *Mitigation*: This is scoped as a separate effort with its own ADR and phased rollout (deprecation → dual-stack → removal).

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Architects Team               | @opendatahub-io/architects | | Yes |
| MLflow Core                   | Edson Tirelli, Matt Prahl | | Yes |
| Model Registry                | Chris Hambridge  | | Yes |
| AI Hub / Catalog              | Jessica Forrester | | Yes |
| Skills Registry               | Bill Murdock     | | Yes |
| Agent Platform / Kagenti      | Dimitri Saridakis | | Yes |
| Product Management            | Peter Double, Myriam Fernandez | | Yes |
| ODH Operator                  | Luca Burgazzolli | | Yes |

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Edson Tirelli | May 24th, 2026 |       |
|               |                |       |
