# Open Data Hub - Align Catalogs and Registries Architecture

|                |            |
| -------------- | ---------- |
| Date           | September 3, 2026 |
| Scope          | OpenShift AI — AI Asset Catalogs and Registries Architecture (excludes data assets) |
| Status         | Draft |
| Authors        | [Edson Tirelli](@etirelli) |
| Amends         | [ODH-ADR-ML-0001](./mlflow/ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md) |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [ODH-ADR-ML-0001 Consolidate AI Asset Registries on MLflow](./mlflow/ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md) |

## What

RHOAI uses the words "catalog" and "registry" to describe three fundamentally different requirements. This ADR names those three concerns explicitly, keeps them architecturally separate, and assigns each to a distinct layer:

1. **Asset Library** — Red Hat (or a partner) acting as a vendor, delivering curated and certified assets to the platform.
2. **Asset Catalog** — a RHOAI administrator curating which assets are visible and usable, platform-wide.
3. **Asset Registry** — end users and teams registering and governing the assets they create, scoped to their workspace.

The decision is to draw the primary architectural boundary at the **Asset Library**: the seam between assets that Red Hat or a partner delivers as a vendor and assets that the platform and its users manage. The Asset Library feeds the Asset Catalog; the Asset Catalog exposes approved assets for consumption; users import assets into their Asset Registry (MLflow) for governance and lifecycle management.

The Asset Catalog and the Asset Registry have virtually the same requirements, so they share one implementation — MLflow — differentiated by namespace scope and RBAC rather than by separate systems. The Asset Catalog is not a separate AI Hub implementation. The Asset Library is a distinct layer, and is built on AI Hub — today's curated storefront and Red Hat delivery channel.

This ADR covers **AI assets** — models, MCP servers, skills, agents, prompts, guardrails, knowledge sources, and runtimes. It does **not** apply to **data assets**. Data asset management, the data registry, and connection management are a separate domain with their own personas, lifecycle, and governance, and are out of scope here.

## Why

[ODH-ADR-ML-0001](./mlflow/ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md) established MLflow as the unified registry backend for AI assets and preserved a registry–catalog separation, naming AI Hub as the catalog layer. That decision addressed *governance* but left the term "catalog" overloaded, and it assumed the catalog would be a separate implementation. In practice, RHOAI is consolidating around MLflow for asset management, AI Hub is positioned as a curated storefront and Red Hat delivery channel, and the model-registry module ships its own Model Catalog and MCP Catalog. Three components now carry overlapping "catalog" and "registry" responsibilities without an agreed boundary.

Closer analysis shows that catalog and registry requirements are virtually the same: both need versioning, metadata, browsing, lifecycle, and RBAC over a set of assets. The difference is scope and access, not mechanism. That makes a separate catalog implementation unnecessary and points the real, unavoidable boundary at content delivery — where vendor delivery obligations genuinely differ from platform and user concerns.

Collapsing three concerns into two words has concrete costs:

- **Duplicated surfaces.** Model Catalog, MCP Catalog (both in the model-registry module), AI Hub, and the MLflow registry each implement discovery, browsing, or curation with no clear division of responsibility. This risks building the same capability more than once.
- **Unclear ownership.** Teams cannot tell whether a given capability (source management, curation policy, delivery, lifecycle) belongs to model-registry, MLflow, AI Hub, or the operator. This slows delivery and produces inconsistent behavior across asset types.
- **Vendor delivery requirements have no home.** Red Hat's delivery obligations — certified vs. community assets, delivery of image-backed assets via `registry.redhat.io` container images, asset versioning, disconnected/air-gapped mirroring, and cross-version governance — are properties of the vendor delivery channel, not of a user's registry or an admin's catalog. Without a named layer, these requirements attach to the wrong component.
- **Conflated personas.** The vendor, the RHOAI administrator, and the end user have different permissions, scopes, and workflows. A single "catalog" surface cannot serve all three cleanly.

Naming the three concerns and fixing the boundary at the Asset Library lets each component own one responsibility and gives the vendor-delivery requirements a well-defined place to live. This refines ML-0001: registry and catalog remain distinct roles but share one MLflow substrate, and the catalog is no longer a separate AI Hub implementation.

## Goals

* Define three distinct concerns — Asset Library, Asset Catalog, and Asset Registry — as first-class architectural layers with distinct personas and scopes.
* Draw the primary architectural boundary at the Asset Library, separating vendor-delivered content from platform- and user-managed content.
* Assign each concern to a single owning layer so that responsibilities do not overlap across components.
* Keep the Asset Catalog and Asset Registry as distinct roles while sharing one implementation — MLflow, differentiated by namespace scope and RBAC. This refines the registry–catalog separation in ODH-ADR-ML-0001 by placing the catalog on MLflow rather than a separate AI Hub implementation.
* Give Red Hat's vendor-delivery requirements (certification, `registry.redhat.io` delivery for image-backed assets, disconnected/air-gapped support, cross-version governance) an explicit home in the Asset Library.
* Support versioned AI assets, so that multiple versions of an asset can be delivered, curated, and governed. This ADR is neutral on delivery cadence; when and how new versions reach a cluster is a separate requirement.
* Support the RHOAI administrator persona: platform-wide curation of which assets are visible, hidden, or require approval.
* Support the user persona: per-workspace registration and lifecycle management of self-authored assets.

## Non-Goals

* **Redefining the MLflow registry backend.** The registry design is set by ODH-ADR-ML-0001 and is not revisited here.
* **Prescribing the migration path** from the existing Kubeflow-based Model Registry to MLflow. That remains a separate, implementation-level decision.
* **Specifying implementation mechanics** — the Asset Library delivery pipeline (mirroring, signing, image build) and the exact MLflow namespace and RBAC design for the Asset Catalog. This ADR fixes the roles and the boundary; the shared-namespace-with-RBAC approach is the chosen direction, and details belong in follow-up designs.
* **Defining per-asset-type metadata schemas or lifecycle states.** Those belong to per-asset-type design documents and ADRs.
* **Specifying the deployment/serving path** for assets once registered.
* **Covering data assets.** Data asset management, the data registry, and connection management are a distinct domain and are not governed by this ADR. The three-layer model here applies to AI assets only.

## How

The architecture is organized as three layers with a single boundary drawn at the Asset Library.

### Layer 1: Asset Library (Red Hat or a partner as vendor)

The channel through which Red Hat or a partner delivers curated, certified assets into a cluster. Its defining properties are vendor and supply-chain concerns:

* Curated and certified content, distinguished as Supported vs. Community.
* Image-backed assets delivered by Red Hat are distributed as `registry.redhat.io` container images. Packaging and distribution formats for other assets, such as prompts and knowledge sources, remain undecided and must be specified in per-asset-type designs; this ADR does not require every asset to be packaged as a container image or delivered through `registry.redhat.io`.
* Versioned, so that multiple versions of an asset can be delivered and referenced.
* Supports disconnected/air-gapped installs through delivery mechanisms appropriate to each format, including mirroring and registry overrides for image-backed assets.
* Requires cross-version governance so assets can be reasoned about across their versions.
* Versioning, provenance, integrity verification, and disconnected-delivery requirements apply regardless of packaging format; per-asset-type designs must specify how they are met.

The Asset Library is built on AI Hub, today's curated storefront and Red Hat delivery channel, which already delivers curated, certified Red Hat content from `registry.redhat.io`. AI Hub is positioned as the vendor content-delivery layer, distinct from the admin-curated Asset Catalog.

### Layer 2: Asset Catalog (admin-curated)

Catalog curation is performed by the RHOAI administrator using delegated RHOAI permissions; it does not require the Kubernetes `cluster-admin` role. These permissions must align with OpenShift RBAC. The RHOAI administrator decides what is available on the platform:

* Admin selects which models, runtimes, skills, or agents are available.
* Admin configures policy per asset: enabled, hidden, or requiring approval.
* End users browse and consume; the catalog is read-only to them.
* Scope is platform-wide, not per-project.

The Asset Catalog consumes content from the Asset Library and accepts assets contributed from workspace Asset Registries, exposing the RHOAI administrator-approved subset for discovery. Because catalog and registry requirements are virtually the same, the Asset Catalog reuses the Asset Registry mechanism rather than being a separate system: it runs on MLflow, using shared namespaces governed by RBAC. RHOAI administrators have the delegated permissions needed to curate catalog content; users have read-only access to browse and consume. The Asset Catalog and Asset Registry share this common substrate and differ primarily in namespace scope (shared and platform-wide vs. a user or team workspace) and access policy. Each role still layers its own semantics on top — for example, admin curation and approval policy for the catalog, and the lifecycle promotion, cross-workspace sharing, catalog contribution, and dependency governance workflows for the registry defined in ODH-ADR-ML-0001. Which specific APIs and controls are shared versus role-specific is an implementation detail for a follow-up design. The catalog is therefore not a separate AI Hub implementation.

This ADR fixes the roles, the boundary, and the substrate: MLflow is the chosen implementation for both the Asset Catalog and the Asset Registry, using shared namespaces governed by RBAC. What remains deferred to a follow-up design is the detailed namespace layout and RBAC configuration, not the choice of MLflow itself.

### Layer 3: Asset Registry (user-managed)

Individual users and teams register, version, and govern the assets they create, scoped to their workspace. This is the MLflow registry defined by ODH-ADR-ML-0001. Users import curated artifacts from the catalog into their workspace registry for governance and lifecycle management. Users and teams can also offer workspace assets for inclusion in the Asset Catalog; the RHOAI administrator controls acceptance and platform-wide availability. Contribution does not grant users direct write access to catalog content.

### Relationship to the existing catalog experience

Today's catalog experience becomes the Asset Library experience. The workflows and capabilities that connect Asset Library → Asset Catalog → Asset Registry, including RHOAI administrator curation and workspace registration, are net-new additions. This alignment is not expected to require migration of existing catalog data or assets.

The existing browsing, discovery, and consumption experience is intended to continue in the Asset Library. The new Asset Catalog workflows add administrator-controlled availability and approval policies. The three architectural layers do not require three separate UI surfaces; detailed UI design and validation belong in follow-up designs. If implementation reveals a need for data or asset migration, a separate ADR and/or design must address it.

### Catalog import and removal

Consistent with the metadata-first design in [ODH-ADR-ML-0001](./mlflow/ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md), importing an asset creates a workspace-owned registration that preserves source provenance and references a specific asset version or digest. Import does not require a deep copy of the underlying artifact. Artifact storage and retention mechanics remain implementation details for follow-up designs.

RHOAI administrators can deprecate catalog entries or withdraw them from future discovery and import. Withdrawal does not automatically delete existing workspace registrations. A retained registration does not guarantee that its referenced artifact remains available; artifact availability, retention, and security revocation require explicit lifecycle policies in follow-up designs. Catalog withdrawal and security revocation are distinct operations.

### The boundary

The primary boundary is drawn at the Asset Library — between what Red Hat or a partner delivers as a vendor and what the platform and its users manage. This is the seam where supply-chain, certification, and versioning concerns end and where admin curation and user governance begin. Because the Asset Catalog and Asset Registry share one MLflow substrate, the Asset Library is the only place a hard architectural boundary is required; the catalog/registry distinction is a matter of namespace scope and RBAC, not separate systems. Drawing the boundary here keeps vendor obligations out of the catalog and registry surfaces: delivery across the Asset Library boundary is one-way, from Red Hat or a partner as a vendor into the platform. The supported flow directions are:

```text
Asset Library → Asset Catalog → Asset Registry
Asset Registry → Asset Catalog (subject to RHOAI administrator approval)
```

Neither the Asset Catalog nor the Asset Registry publishes assets back into the vendor Asset Library. Registry-to-catalog contribution is supported; detailed promotion and import mechanics are left to follow-up design documents.

## Alternatives

### 1. Keep the two-term model (status quo)

Continue using "catalog" and "registry" without naming the third concern, letting each component grow its own catalog and registry surfaces as needed.

**Tradeoffs:** No coordination cost up front. But the duplication between Model Catalog, MCP Catalog, AI Hub, and the MLflow registry persists and grows, ownership stays ambiguous, and vendor-delivery requirements continue to attach to whichever component happens to implement them. **Rejected** — it perpetuates the problem this ADR exists to solve.

### 2. Draw the boundary at the persona/UI layer

Keep a single shared backend for catalog and registry content and separate the three concerns only in the UI, presenting different views to the vendor, admin, and user personas.

**Tradeoffs:** Simpler backend and one storage model. But the three concerns differ most in their *delivery and supply-chain* properties, not their presentation: disconnected mirroring, asset versioning, and certification are backend concerns that a UI split cannot express. A single backend would have to satisfy vendor-delivery, admin-curation, and user-governance requirements at once. **Rejected** — the boundary must sit where the requirements actually diverge, which is the Asset Library.

### 3. Keep the catalog as a separate implementation

Implement the Asset Catalog as its own system — for example, a dedicated AI Hub catalog service — distinct from the Asset Registry, as ML-0001 originally assumed.

**Tradeoffs:** A clear, standalone catalog product surface. But catalog and registry requirements are virtually the same — versioning, metadata, RBAC, browsing, lifecycle — so a separate catalog system reimplements what MLflow already provides for the registry. It duplicates effort, risks divergent behavior, and gives administrators a second system to operate. **Rejected** — the Asset Catalog reuses the MLflow registry mechanism (shared namespaces with RBAC) instead of being a separate implementation.

### 4. Merge all three concerns into one system

Collapse content delivery, catalog, and registry into a single system that both delivers vendor content and hosts platform- and user-managed assets.

**Tradeoffs:** Fewer moving parts. But this mixes vendor-delivered content with platform- and user-created content in one lifecycle, reintroducing exactly the conflation this ADR removes and forcing supply-chain and air-gapped concerns onto surfaces that should not carry them. **Rejected** — the boundary at the Asset Library must remain, even though catalog and registry share a substrate below it.

## Security and Privacy Considerations

* **Supply chain.** The Asset Library is the vehicle through which the customer consumes certified, signed, and provenance enforced assets. 
* **Disconnected/air-gapped.** The boundary at the Asset Library is what makes air-gapped support tractable: only the Asset Library needs mirroring; catalog and registry operate on already-mirrored content.
* **Admin policy enforcement.** Asset Catalog policies (enabled, hidden, requires-approval) are an authorization surface and must be enforced server-side, not only in the UI.
* **Executable content.** Some assets (skills, MCP servers, agents) reference executable code. Promotion gates for executable content, as noted in ODH-ADR-ML-0001, apply across the Asset Library, Asset Catalog, and Asset Registry.
* **RBAC alignment.** The Asset Catalog and Asset Registry both run on MLflow with access controlled by RBAC over namespaces. The Asset Catalog uses shared, platform-wide namespaces where RHOAI administrators have delegated catalog-curation permissions and users have read-only access to browse and consume; Asset Registries are user or team workspaces. This RBAC must align with OpenShift namespace RBAC so that catalog curation, catalog consumption, and registry governance respect the same permission model. Because the catalog and registry share one substrate, namespace boundaries and RBAC are the primary control preventing users from writing to catalog content or reading across workspaces they should not access.

## Risks

* **Cross-team coordination.** The change spans AI Hub (Asset Library), MLflow (Asset Catalog and Asset Registry), the model-registry catalog surfaces, and the operator. Aligning ownership requires sign-off from multiple teams; without it, the conflation re-emerges at the seams.
* **Catalog on shared namespaces.** Serving the Asset Catalog from shared MLflow namespaces makes RBAC the load-bearing control for the admin-curate / user-read-only split. A misconfiguration exposes write access or cross-tenant reads. This must be validated carefully.
* **Asset Library delivery maturity.** Versioning and cross-version governance are new capabilities for some asset types. 
* **Implementation continuity.** No data or asset migration is currently expected for this catalog/library alignment: the existing catalog experience becomes the Library experience, and the connecting Library → Catalog → Registry workflows are net-new capabilities. These additions must preserve existing behavior. If a migration requirement is identified later, it must be addressed in a separate ADR and/or design. This expectation does not cover the separate Kubeflow Model Registry-to-MLflow migration excluded above.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Architects Team               | @opendatahub-io/architects | | Yes |
| AI Hub / Catalog              | Edson Tirelli | | Yes |
| Model Registry                | Edson Tirelli  | | Yes |
| MLflow Core                   | Matt Prahl | | Yes |
| ODH Dashboard                 | Andrew Ballantyne | | Yes |
| ODH Operator                  | Luca Burgazzolli | | Yes |
| Product Management            | Adam Bellusci, Peter Double, Myriam Fernandez | | Yes |

Notes:

* **AI Hub / Catalog** — AI Hub is the base for the Asset Library (vendor content delivery). The admin-curated Asset Catalog is not an AI Hub implementation; it moves onto MLflow.
* **MLflow** — owns both the Asset Registry and the Asset Catalog, which share the MLflow substrate (user workspaces vs. shared namespaces with RBAC).
* **Dashboard** — presents the appropriate capabilities to each persona and enforces the visible effects of administrator policy. The architectural layers do not require separate UI surfaces; preserve familiar user workflows as described above.

## References

* [ODH-ADR-ML-0001 — Consolidate AI Asset Registries on MLflow](./mlflow/ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md)
* [ODH-ADR-ML-0002 — Shared Workspace for Cross-Namespace Resource Sharing](./mlflow/ODH-ADR-ML-0002-shared-workspace-for-cross-namespace-resource-sharing.md)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Daniele Zonca                 | September 11, 2026 |       |
| Ann Marie                     | September 10, 2026 |       |
| Matthew Prahl                 |            |       |


