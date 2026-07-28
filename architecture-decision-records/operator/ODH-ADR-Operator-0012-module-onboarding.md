# Open Data Hub - Module Onboarding Architecture

|                |                                  |
| -------------- |----------------------------------|
| Date           | Jan 19, 2026                     |
| Scope          | Open Data Hub Operator           |
| Status         | Approved                         |
| Authors        | [Luca Burgazzoli](@burgazzoli)   |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        |                                  |
| Other docs:    | [Module Onboarding Guide](design/module-onboarding-guide.md) |

## What

This ADR defines the architecture for onboarding new modules to the Open Data Hub (ODH) Operator. It establishes the separation of concerns between the ODH Operator (meta-operator) and module controllers (independent operators).

## Terminology

To ensure clarity throughout this document, the following terms are defined:

- **Meta-Operator**: The ODH Operator that orchestrates module controllers (not application workloads). It manages the lifecycle of module controllers but does not directly manage the operands.
- **Module**: A platform component that provides specific functionality (e.g., Kserve, Dashboard, Model Registry, Workbenches, Data Science Pipelines).
- **Module Controller**: An independent operator that manages the lifecycle of a specific module. Designed to be standalone and self-sufficient.
- **Module CR**: A cluster-scoped singleton custom resource that serves as the configuration interface for a module.
- **Operand**: Application resources deployed and managed by the module controller (e.g., Deployments, Services, Routes, ConfigMaps). These are the actual workload components that provide module functionality.

## Why

The ODH Operator originally managed all platform modules directly - manifests, reconciliation logic, and domain-specific behavior lived in a single codebase. As the platform grew in both the number of modules and the number of contributing teams, three pressures compounded:

1. **Team velocity does not scale in a monolith.** Adding developers does not add velocity; it adds coordination overhead. Multiple teams sharing one operator codebase means merge conflicts, shared test infrastructure where one failure blocks everyone, and coordinated releases that move at the speed of the slowest module.

2. **Domain expertise is in the wrong place.** Each module has its own operational domain (serving infrastructure, pipeline orchestration, model registries, etc.), yet all that knowledge must be encoded inside the platform operator. This includes integration with shared platform concerns (hardware profiles, Kueue, networking, storage), upgrade logic, and environment-specific behavior - topics that require deep module-specific knowledge. Module teams depend on the platform team for every change, instead of owning their module end-to-end.

3. **Failures and lifecycles are coupled.** A bug in one module's reconciliation logic can block releases for all modules. A resource leak or crash in one controller affects every module on the cluster. There is no isolation boundary.

4. **Permissions are too broad.** Because the ODH Operator manages every module's resources directly, it requires near cluster-admin privileges. The principle of least privilege demands that each component only holds the permissions it actually needs. Separating module controllers allows each to request only the RBAC required for its specific domain, while the ODH Operator's permissions are reduced to core Kubernetes types.

These pressures compound - fixing one does not help if the others remain. The architecture addresses all three by separating the ODH Operator into a **meta-operator** that handles only lifecycle orchestration (install, upgrade, uninstall) and **independent module controllers** that are the domain experts for their specific modules. Each module team owns their own operator, develops and releases independently, and failures are isolated. The ODH Operator manages module controller lifecycles directly, without relying on OLM for installation or upgrades.

By narrowing the ODH Operator's scope to platform foundations (controller lifecycle, platform configuration, status aggregation), it becomes a stable and focused layer that everything else builds on. Module teams build on this foundation for their specific domains, but the same pattern also applies to platform-wide concerns: new platform logic (e.g., hardware provisioning, quota management, observability) can be implemented as dedicated modules rather than growing the core operator. Each module, whether domain-specific or platform-wide, follows standard Kubernetes patterns for composition and discovery, reducing coupling and behaving like any other Kubernetes extension.

**Module Controllers are Independent Operators:**
- Each module controller is designed as a standalone operator that could run independently
- Contains all logic, manifests, and intelligence required to manage its module's lifecycle
- Acts as the domain expert for its specific module

**ODH Operator Manages Module Controller Lifecycles:**
- Handles installation, upgrades, and uninstallation of module controllers
- Orchestrates module controllers but does not manage the deep internal resources of modules
- Renders platform configuration (auth, TLS, observability, networking) into each module's ConfigMap

**Two Operational Modes:**

The ODH Operator is always present and responsible for deploying module controllers and injecting platform configuration via ConfigMap. The ConfigMap injection ensures the platform behaves consistently regardless of the operational mode and regardless of who owns the Module CR spec. The `DataScienceCluster` (DSC) CR is an optional high-level entry point that controls who creates Module CRs:

- **DSC mode** (e.g., RHAI): The ODH Operator watches the DSC, creates/updates Module CRs based on `spec.components`, and aggregates module status back to the DSC. The ODH Operator owns the Module CR spec and will revert manual edits to maintain consistency with the DSC.
  ![DSC Mode](assets/ODH-ADR-Operator-0012/dsc.jpg)

- **Standalone module mode** (e.g., RHAII): Users or GitOps pipelines create and own Module CRs directly. The ODH Operator still deploys module controllers and projects platform configuration via ConfigMap, but does not manage Module CR lifecycle.
  ![Standalone Mode](assets/ODH-ADR-Operator-0012/non-dsc.jpg)

## Goals

- Establish clear architectural boundaries between the ODH Operator and module controllers
- Enable independent development and lifecycle management of modules
- Ensure modules are designed as standalone operators
- Provide consistent API contracts for integration with the platform

## How

The ODH Operator acts as a meta-operator that manages module controllers through the following approach:

**ODH Operator Responsibilities:**
- Manages the lifecycle (install, upgrade, uninstall) of module controllers
- Deploys module controller manifests (Deployment, RBAC, CRDs)
- Renders platform configuration into each module's ConfigMap
- Prunes module resources when modules are disabled or removed
  - **DSC mode:** additionally creates and updates module CRs based on DataScienceCluster configuration, and aggregates status from module CRs back to the DataScienceCluster
  - **Standalone mode:** does not manage Module CR lifecycle; users or GitOps own Module CRs directly

**Module Controller Responsibilities:**
- Reconciles the module-specific CR
- Manages the complete lifecycle of the module's application resources
- Configuration merging: reads platform configuration from its ConfigMap and merges it with the user's spec to produce the effective configuration
- Detects cluster capabilities (FIPS mode, platform variant, available CRDs) and adjusts behavior accordingly
- Discovers dependencies dynamically and handles missing dependencies gracefully (degrade vs. fail)
- Manages internal TLS certificates (e.g., for webhooks, mTLS) using cert-manager
- Reports health and status information back to the module CR, surfacing module context (auth mode, endpoints, TLS settings) in status for platform discoverability

### API Contracts (Summary)

**Module CR Requirements:**
- **Scope:** Cluster-scoped singleton CR (enforced via CEL validation)
- **API Group:**
  - `components.platform.opendatahub.io` for modules (e.g., Kserve, Workbenches)
  - `services.platform.opendatahub.io` for platform services (e.g., observability)
- **Version:** Must match support level (`vXalphaY` for developer preview, `vXbetaY` for tech preview, `vX` for GA)
- **Required Status Fields:** `observedGeneration`, `conditions` (Ready, ProvisioningSucceeded, Degraded), `distribution` (name + version), `releases`
- **Labeling:** `components.platform.opendatahub.io/managed-by: <module-name>`
- **Condition Severity:** Conditions support `Error` (blocking, default) and `Info` (non-blocking) severity levels, helping external observers determine whether a `False` condition represents an actual problem or is informational
- **Spec Ownership:** The CRD spec is user-owned; platform-enforced settings are delivered via ConfigMap, not spec fields
- **Platform Configuration:** Each module ships a ConfigMap with sensible defaults; the ODH Operator overrides platform-managed keys and enforces them

See the [Module Onboarding Guide](design/module-onboarding-guide.md) for complete API specifications, validation rules, and status aggregation semantics.

### Implementation Requirements

**Manifest Packaging:**

Helm is the preferred method for packaging module controller manifests. Kustomize is supported but switching to Helm is highly encouraged. Manifests are embedded in the ODH controller binary at build time, ensuring the operator is self-contained.

The manifests that the ODH Operator installs for a module controller must be limited to core Kubernetes types (Deployment, ServiceAccount, RBAC, CRD). This constraint is driven by the principle of least privilege: the ODH Operator today operates with near cluster-admin permissions, and reducing its scope to core Kubernetes types only - with no knowledge of workload-specific CRDs - is a key goal of this architecture.

The module operator is the orchestrator for its feature area and should run as a separate Deployment from its operand controllers to maintain failure isolation and independent scaling. Common patterns include a single image with multiple entrypoints (recommended), multiple images when running upstream controllers alongside the module operator, or a single-process deployment for very simple modules.

See the [Module Onboarding Guide](design/module-onboarding-guide.md) for complete implementation requirements including deployment pattern details, dependency management, certificate management, and RBAC guidelines.

## Open Questions

N/A

## Responsibility

The ODH Platform team is responsible for maintaining this architecture and the [odh-platform-utilities](https://github.com/opendatahub-io/odh-platform-utilities) shared library to facilitate module development.

## Alternatives

N/A

## Stakeholder Impacts

| Group                     | Key Contacts                    | Date       | Impacted? |
| ------------------------- | ------------------------------- | ---------- | --------- |
| ODH Platform Team         | @lphiri, @burgazzoli            | 2026/01/19 | YES       |


## References

- [Detailed Module Onboarding Guide](design/module-onboarding-guide.md)
- [ODH Platform Utilities](https://github.com/opendatahub-io/odh-platform-utilities)
- [ODH-ADR-Operator-0006: Internal API](ODH-ADR-Operator-0006-internal-api.md)
- [ODH-ADR-Operator-0008: Resource Lifecycle Management](ODH-ADR-Operator-0008-resources-lifecycle.md)
