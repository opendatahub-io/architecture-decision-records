# ODH ADR SR-0001: Skill Registry

|                |                                                                                                                                                   |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Date           | 2026-04-16                                                                                                                                        |
| Scope          | Skill Registry                                                                                                                                    |
| Status         | Draft                                                                                                                                             |
| Authors        | [Bill Murdock](@jwm4)                                                                                                                             |
| Supersedes     | N/A                                                                                                                                               |
| Superseded by: | N/A                                                                                                                                               |
| Tickets        | RHAIRFE-1713                                                                                                                                      |
| Other docs:    | [Skill Registry MVP Design](https://github.com/B-Step62/mlflow/blob/ca5d1a2f6e55691077425b417c26f45380fbf623/.agent/skill-registry-mvp-design.md) |

## What

ODH adopts MLflow's upcoming Skill Registry as the platform component
for versioning, managing, and distributing AI agent skills. The registry
enables users to register skills from GitHub repositories or local
directories, version them, browse them in the MLflow UI, and install
them into agent harnesses with metadata tracking. The initial upstream
implementation targets Claude Code, but ODH requires the architecture to
be agent-harness-neutral at its core, with harness-specific integrations
layered on top.

## Why

The ODH ecosystem currently has no mechanism for managing AI agent
skills (reusable tool definitions, workflow steps, and coding assistant
capabilities that agents consume). As agentic AI workloads grow
on the platform, teams need a way to share, version, and govern skills
across projects and organizations. Without a registry, skills
proliferate as ad-hoc files copied between repositories, with no
versioning, no discoverability, and no governance.

Databricks MLflow maintainers are actively building a skill registry
targeting late April 2026, which creates an opportunity to adopt an
upstream-first approach rather than building a parallel solution. The
agreed ODH strategy is: MLflow handles registries (governance,
versioning), Kubeflow handles catalogs (discovery, browsing). This ADR
formalizes ODH's adoption of that strategy for skills.

Regulated and disconnected customers (Federal, Financial Services,
Healthcare) need in-cluster skill management that works without access
to public registries, with FIPS compliance and multi-tenant isolation.
This requirement cannot be met by upstream MLflow alone and motivates
the downstream hardening scope.

## Goals

* Adopt MLflow's Skill Registry as the single versioning and governance
  layer for AI skills in ODH/RHOAI
* Ensure the core registry API is agent-harness-neutral: skills can be
  registered, versioned, searched, and retrieved regardless of which
  agent runtime consumes them
* Support harness-specific integration layers (starting with Claude
  Code) that add value like trace enrichment and automated installation
  without coupling the core registry to a single runtime
* Enable in-cluster, self-hosted operation for disconnected and
  air-gapped environments
* Provide multi-tenant isolation so that project-scoped skills are not
  accessible across namespace boundaries
* Support skill versioning, aliasing (e.g., "champion" or "prod"), and
  tagging for lifecycle management
* Integrate with MLflow tracing so that agent traces record which skill
  version was used

## Non-Goals

* Building a skill authoring or development environment. The registry
  manages finished skills, not the process of writing them.
* Building a separate skill catalog in Kubeflow or the RHOAI Dashboard.
  The MLflow Skill Registry includes full discovery and browsing UX
  (search, filtering, rendered SKILL.md preview, usage stats from
  traces), making a parallel catalog redundant. See "No separate skill
  catalog" below for rationale.
* Defining the skill format specification. The registry is
  format-agnostic and stores skill bundles as artifacts; the SKILL.md
  format is defined by the agent ecosystem, not by ODH.
* Building an MCP registry. That is a separate effort, though the
  MLflow architecture is designed to accommodate multiple asset registry
  types.
* Supporting every agent harness at launch. Claude Code is the first
  integration target, with others to follow based on demand.

## How

### Upstream-first approach

ODH adopts the MLflow Skill Registry being developed by Databricks,
contributing requirements and code upstream rather than building a
parallel implementation. The upstream registry provides:

* **Database-backed storage** with dedicated tables for skills, skill
  versions, tags, and aliases (not reusing the Model Registry schema)
* **SDK API** under `mlflow.genai` for registering, loading, searching,
  versioning, tagging, and aliasing skills
* **REST API** at `/ajax-api/3.0/mlflow/skills/` for programmatic access
* **CLI** (`mlflow skills register`, `mlflow skills list`,
  `mlflow skills load`) for command-line workflows
* **MLflow UI** with a Skills page for browsing, searching, and managing
  registered skills
* **Artifact storage** using MLflow's existing artifact infrastructure
  to store skill bundles (SKILL.md plus any associated scripts,
  references, and assets)

### Agent-harness-neutral core

The upstream MVP currently includes Claude Code-specific integration
(`mlflow.genai.install_skill()` writes to `~/.claude/skills/` and
enriches Claude Code traces). ODH requires the core registry to be
usable without any Claude Code dependency. Review of the upstream
design shows that this separation largely already exists:

1. **Core registry layer** (already harness-neutral): register, version,
   search, retrieve, tag, alias skills via SDK/REST/CLI. The database
   schema, REST API, SDK functions (`register_skill()`, `load_skill()`,
   `search_skills()`, `set_skill_alias()`, `set_skill_tag()`), CLI
   commands, and MLflow UI are all independent of any agent runtime.
2. **Harness integration layer** (currently Claude Code only): two
   features are Claude Code-specific. `install_skill()` writes to
   `~/.claude/skills/` with a metadata sidecar, and the tracing
   enhancement in `claude_code/tracing.py` enriches spans with skill
   version info. These are the only coupling points.

The decoupling work needed is modest. The main change would be making
`install_skill()` pluggable for different target directories rather
than hardcoding the Claude Code path. The core registry does not need
redesigning. Whether this change goes upstream or is maintained
downstream is an open question (see below). Edson Tirelli and Matt
Prahl have indicated willingness to facilitate upstream discussions
with Databricks.

### Value proposition without harness-specific integration

The Claude Code integration provides clear benefits for Claude Code
users: automatic skill installation, metadata sidecars for trace
enrichment, and tracing spans that show exactly which skill version was
used. But the registry must also justify itself for users of other agent
harnesses that lack these integrations.

The core value proposition is organizational governance of skills as
first-class AI assets, analogous to why organizations adopt a model
registry instead of keeping models in object storage. Even without
harness-specific integration, the registry provides:

* **Versioning with semantic aliases.** Aliases like "prod" or
  "champion" can be moved between versions, enabling promotion
  workflows. Git tags are static and flat by comparison.
* **Governance and lifecycle management.** Tag versions as approved,
  deprecate old versions, control who can promote skills to production.
  A GitHub repo has no governance layer for these workflows.
* **Centralized discovery across sources.** Skills may originate in
  many different repositories. The registry provides a single
  searchable catalog regardless of where the skill was developed.
* **Usage analytics.** MLflow traces show which skills are being used
  and how often, giving visibility into adoption across the
  organization.
* **Artifact bundling.** The registry versions the complete skill bundle
  (SKILL.md, scripts, references, assets) as a single unit, rather
  than relying on convention to keep related files together in a repo.
* **Disconnected operation.** The registry works in-cluster without
  network access. GitHub-based distribution does not.

Harness-specific integrations like the Claude Code sidecar enhance the
experience for specific runtimes but are not the core value. The
registry earns adoption by solving the organizational problems of
versioning, discovery, and governance that every team faces regardless
of which agent harness they use.

### No separate skill catalog

The agreed ODH strategy is: MLflow for registries (governance,
versioning), Kubeflow for catalogs (discovery, browsing). However, the
MLflow Skill Registry as designed by Databricks includes substantial
catalog-like functionality built directly into the MLflow UI:

* A **Skills list page** with search filtering, showing name,
  description, latest version, tags, and creation time
* A **Skills detail page** with rendered SKILL.md preview, version
  history, tag and alias management
* A **search API** (`search_skills()`) for programmatic discovery
* **Usage dashboards** showing skill stats derived from MLflow trace
  metadata, so users can see which skills are actively used and how they
  perform

This means the registry-vs-catalog distinction that applies to models
(where Model Registry handles governance and the RHOAI Dashboard
provides a separate storefront) does not cleanly apply to skills. The
MLflow UI *is* the catalog for skills. Users browse, search, inspect,
and install skills from the same interface that governs them.

Building a parallel skill catalog in the RHOAI Dashboard would duplicate
this functionality, force users to choose between two browsing
experiences for the same assets, and fight the upstream investment
Databricks is making in MLflow's discovery UX. ODH should not build a
separate skill catalog unless a concrete user need emerges that the
MLflow UI cannot address.

### Downstream hardening for RHOAI

On top of the upstream registry, RHOAI adds enterprise requirements:

* **Disconnected/air-gapped operation**: The registry must function
  entirely in-cluster without external network access. Skill bundles are
  stored in the MLflow artifact store (which maps to persistent storage
  in OpenShift), not fetched from GitHub at runtime.
* **FIPS compliance**: All cryptographic operations within the registry
  lifecycle must use FIPS-validated modules.
* **Multi-tenant isolation**: RHOAI deploys a separate MLflow instance
  per data science project (namespace). Because the upstream skill
  registry schema is single-tenant (skill names are unique within an
  MLflow instance, with no workspace or tenant column), namespace
  isolation is achieved through these per-namespace MLflow instances,
  not through workspace-level scoping within a shared instance.
  Namespace RBAC ensures that project-scoped skills in one namespace
  are not accessible from another. The registry also supports global
  (org-wide) skills visible across namespaces; the mechanism for
  cross-namespace skill sharing (e.g., a shared MLflow instance for
  global skills or a federation approach) is not yet designed and will
  be addressed in Phase 2.
* **Operator integration**: The MLflow instance (and thus the skill
  registry) is deployed and managed by the ODH Operator via the
  `DataScienceCluster` CRD. No additional operator is needed for the
  skill registry. It is a feature of the MLflow deployment. No changes
  to the DataScienceCluster CRD schema are expected, though the
  operator's MLflow manifests may need updates to accommodate new
  database migrations or configuration flags shipped by upstream.

### Operational considerations

The skill registry is not a new standalone component; it rides on the
existing MLflow deployment and inherits its operational posture
(health probes, resource limits, Prometheus metrics, operator-managed
lifecycle). The incremental resource impact is expected to be small:
skill bundles are lightweight (typically under 1 MB), and the number
of registered skills in Phase 1 is expected to be in the tens to low
hundreds. No new pods, services, or databases are required beyond the
existing MLflow deployment.

The skill registry's blast radius is limited. If the MLflow instance
becomes unavailable, previously installed skills (already written to
the agent harness's local filesystem) continue to work. New skill
registration, versioning, and search are unavailable until the
registry recovers, but agent execution is not blocked.

Observability for the skill registry should follow the platform's
established patterns (see ODH-ADR-Operator-0010 for component metrics
scraping). Whether MLflow's existing `/metrics` endpoint covers skill
registry API operations or whether new metrics are needed will be
validated during Phase 1.

### Phased rollout

1. **Phase 1 (near-term)**: Adopt upstream MLflow Skill Registry as
   shipped by Databricks. Validate in ODH. Contribute
   agent-harness-neutrality requirements upstream.
2. **Phase 2**: Add RHOAI downstream hardening: disconnected support,
   FIPS, multi-tenancy isolation, operator lifecycle management.
3. **Phase 3**: Connect to the broader AI asset registry strategy (MCP
   registry, agent registry). Evaluate whether the MLflow UI fully meets
   user discovery needs or whether lightweight Dashboard integration is
   warranted.

## Open Questions

* The upstream design is already largely harness-neutral (see above),
  with Claude Code coupling confined to `install_skill()` and trace
  enrichment. The remaining question is whether Databricks will accept
  a contribution to make `install_skill()` pluggable, or whether ODH
  should maintain that generalization downstream.
* Should the skill registry support a lockfile mechanism for
  reproducible skill installations? In package managers like npm, a
  lockfile records the exact versions that were resolved at install
  time, so that anyone cloning the repo gets the same dependencies.
  Applied to skills: a project might depend on the "copilot" skill at
  alias "prod", which today resolves to version 3. A month later, "prod"
  might point to version 5. A lockfile would pin the resolved version
  and MLflow server URL so that `mlflow skills load` reproduces the
  exact same skill set. Matt Prahl has proposed this, including content
  hashes and workspace identifiers. If this is desirable, is it upstream
  or downstream scope?
* Does the downstream RHOAI MLflow team need to do anything beyond
  their standard hardening process to support the skill registry?

## Alternatives

### Alternative 1: Build a custom skill registry in ODH

Build a standalone skill registry component, independent of MLflow,
deployed as its own operator-managed service.

**Pros:**

* Full control over the design and API surface
* No dependency on Databricks upstream timeline
* Can be purpose-built for OpenShift and RHOAI constraints from day one

**Cons:**

* Duplicates effort that Databricks is already investing in upstream
* Creates a divergent ecosystem. Skills registered in ODH's custom
  registry would not be visible in MLflow UI or connected to MLflow
  traces.
* Contradicts the agreed strategy that MLflow handles registries
* Significant engineering investment for a capability that is being
  delivered upstream for free

### Alternative 2: Use Kubeflow Model Registry for skills

Extend the existing Kubeflow Model Registry to store skill metadata
alongside models.

**Pros:**

* Leverages existing infrastructure already deployed in ODH/RHOAI
* Keeps all AI assets in a single catalog

**Cons:**

* Model Registry's schema and UX are designed for ML models, not skill
  bundles
* The agreed strategy places registries in MLflow and catalogs in
  Kubeflow
* Would require significant schema changes to accommodate
  skill-specific concepts (SKILL.md parsing, versioning semantics,
  harness integration)
* Goes against the strategic direction agreed by stakeholders

### Alternative 3: GitHub repository as skill registry (interim)

Use a curated GitHub repository (e.g., an ODH skills repo) as a
lightweight registry, with skills versioned via Git tags.

**Pros:**

* Simple and fast to set up
* Familiar workflow for developers
* Has been discussed as a Phase 1 stopgap

**Cons:**

* No governance features (tagging, aliasing, access control,
  multi-tenancy)
* Cannot work in disconnected environments without additional mirroring
  infrastructure
* No integration with MLflow tracing or observability
* Not a long-term solution; would need to be migrated to MLflow
  eventually

This approach may still serve as a short-term bridge before the MLflow
registry is production-ready, but should not be confused with the
registry itself.

## Security and Privacy Considerations

* **Multi-tenancy isolation**: Skills scoped to a namespace must not be
  accessible from other namespaces. The MLflow workspace-to-namespace
  mapping enforced by RBAC provides this boundary.
* **Skill content trust**: The registry stores arbitrary files (SKILL.md,
  scripts, etc.) as artifacts. Skills may contain executable code
  (Python scripts, shell scripts). Organizations need a mechanism to
  review and approve skill content before promoting it (e.g., tagging a
  version as "prod" or "approved"). The registry provides versioning and
  tagging but does not itself enforce content review workflows. As a
  Phase 2 consideration, artifact signing for skill bundles should be
  evaluated, using ODH-ADR-MR-0001-Sign as a potential model for the
  approach.
* **Disconnected operation**: In air-gapped environments, skills must be
  loaded from in-cluster storage. The `register_skill()` flow that
  fetches from GitHub URLs is not available; skills must be registered
  from local paths or pre-staged artifacts.
* **FIPS compliance**: The registry's database operations and artifact
  storage must use FIPS-validated cryptographic modules. This is
  inherited from the RHOAI MLflow deployment's FIPS configuration.

## Risks

* **Upstream timeline dependency**: If Databricks delays the skill
  registry or ships it in a form that is too tightly coupled to Claude
  Code, ODH's adoption timeline slips. Mitigation: engage directly with
  upstream maintainers (Yuki Watanabe, Matt Prahl has offered to
  facilitate) and contribute decoupling work.
* **Scope creep across registry types**: The broader AI asset registry
  strategy includes MCP registries, agent registries, and potentially
  others. There is a risk that the skill registry becomes a catch-all
  for "anything that isn't a model." Mitigation: maintain clear
  boundaries. Skills are defined by the SKILL.md manifest format, and
  this ADR covers only that asset type.
* **Perception of inconsistency with the model registry/catalog
  pattern**: For models, ODH separates the registry (MLflow) from the
  catalog (RHOAI Dashboard). This ADR explicitly does not follow that
  pattern for skills, because the MLflow Skill Registry already includes
  full discovery UX. This may confuse stakeholders who expect the same
  split. Mitigation: document the rationale clearly (this ADR does that)
  and revisit if the MLflow UI proves insufficient for discovery needs.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| ----- | ------------ | ---- | --------- |
| MLflow / Model and Agent Observability | Matt Prahl | 2026-04-16 | Yes |
| ODH Operator | unknown | 2026-04-16 | Probably not |
| Dashboard / AI Hub | Eder Ignatowicz | 2026-04-16 | Yes |
| AgentDev | Aakanksha Duggal | 2026-04-16 | Yes |
| Model Serving & Registry (PM) | Adam Bellusci | 2026-04-16 | Yes |
| Agentic AI (PM) | Peter Double, Adel Zaalouk | 2026-04-16 | Yes |
| Model Serving (KServe) | | 2026-04-16 | No |
| Data Science Pipelines | | 2026-04-16 | Maybe |
| Distributed Workloads | | 2026-04-16 | No |
| TrustyAI / Explainability | | 2026-04-16 | No |

**Notes on stakeholder impacts:**

* **MLflow**: Primary implementation target. The skill registry is a
  new MLflow feature.
* **ODH Operator**: The skill registry is internal to MLflow, not a new
  component. However, the operator's MLflow manifests may need updates
  to accommodate new database migrations or configuration flags from
  upstream.
* **Dashboard / AI Hub**: This ADR does not propose a separate skill
  catalog in the Dashboard, but the Dashboard team may want to embed
  or link to the MLflow skill registry experience in the future.
* **AgentDev**: Agents consume skills. AgentDev needs to integrate
  with the registry for skill resolution.
* **Model Serving & Registry (PM)**: Owns the overall AI asset registry
  strategy.
* **Agentic AI (PM)**: Agents consume skills; the skill registry is
  core infrastructure for agentic AI workflows.
* **Data Science Pipelines**: Pipelines that orchestrate agent workflows
  may need to resolve skills from the registry.

## References

* [Skill Registry MVP Design](https://github.com/B-Step62/mlflow/blob/ca5d1a2f6e55691077425b417c26f45380fbf623/.agent/skill-registry-mvp-design.md)
* [MLflow Skill Registry MVP branch](https://github.com/B-Step62/mlflow/tree/skill-registry-mvp)
* [RHAIRFE-1713: In-Cluster Skills Registry](https://redhat.atlassian.net/browse/RHAIRFE-1713)
* [Anthropic MCP Registry Specification](https://registry.modelcontextprotocol.io/docs)
* [Agent Skills Specification](https://agentskills.io/specification)
* [OpenAI Skills API](https://developers.openai.com/api/docs/guides/tools-skills)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
