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

The ODH Operator functions as a **meta-operator** that manages the complete lifecycle (install, upgrade, uninstall) of module controllers without relying on Operator Lifecycle Manager (OLM).

**Module Controllers are Independent Operators:**
- Each module controller is designed as a standalone operator that could run independently
- Contains all logic, manifests, and intelligence required to manage its module's lifecycle
- Acts as the domain expert for its specific module

**ODH Operator Manages Module Controller Lifecycles:**
- Handles installation, upgrades, and uninstallation of module controllers
- Orchestrates module controllers but does not manage the deep internal resources of modules
- No OLM dependency - the ODH Operator handles all lifecycle management directly

![Module Architecture](assets/ODH-ADR-Operator-0012/module.png)

This architectural separation provides:
- Scalability through independent reconciliation loops per module
- Reduced coupling between the platform and individual modules
- Better isolation of failures and easier troubleshooting
- Independent development and evolution of modules

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
- Creates and updates module CRs based on DataScienceCluster configuration
- Aggregates status from module CRs back to the DataScienceCluster
- Prunes module resources when modules are disabled or removed

**Module Controller Responsibilities:**
- Reconciles the module-specific CR
- Manages the complete lifecycle of the module's application resources
- Detects cluster capabilities and adjusts behavior accordingly
- Reports health and status information back to the module CR
- Extends upstream components with platform-specific functionality when needed

### API Contracts (Summary)

**Module Controller Requirements:**
- **Scope:** Cluster-scoped singleton CR (enforced via CEL validation)
- **API Group:** `components.platform.opendatahub.io` or `services.platform.opendatahub.io`
- **Version:** Must match support level:
  - Developer preview: `vXalphaY` (e.g., `v1alpha1`)
  - Technology preview: `vXbetaY` (e.g., `v1beta1`)
  - General availability: `vX` (e.g., `v1`)
- **Required Status Fields:**
  - `observedGeneration` (int64)
  - `conditions` ([]metav1.Condition) - must include `Ready`, `ProvisioningSucceeded`, `Degraded`
  - `releases` (Array) - list of deployed components with `name`, `version`, `repoUrl`
- **Labeling:** `components.platform.opendatahub.io/managed-by: <module-name>`

**Module CR Spec Configuration:**

Module CRs can expose additional fields beyond what is configured in the DataScienceCluster. Users can set these fields directly on the Module CR for advanced configuration (e.g., resource requirements, replicas, advanced networking settings).

**Platform Configuration Projection:**

Module CRs receive platform-wide settings (authentication, certificates, monitoring) projected from DataScienceCluster and DSCInitialization. Module controllers MUST respect these projected configurations. Manual user modifications to platform-managed fields will be reverted by the ODH Operator to maintain platform compliance.

**Status Aggregation:**

The ODH Operator aggregates Module CR status to DataScienceCluster as follows:
- Module CR conditions are reflected to DSC.status.conditions with module-prefixed types (e.g., Ready becomes <Module>Ready)
- Module CR releases are copied to DSC.status.components.<module>.releases
- Module CR observedGeneration is used to detect reconciliation state

See [Module Onboarding Guide](design/module-onboarding-guide.md) for complete status aggregation rules and specifications.

### Lifecycle State Transitions

| Trigger | ODH Operator Action | Module Controller Action |
|---------|---------------------|--------------------------|
| DSC enables module | Deploy controller manifests (Deployment, RBAC, CRD), create Module CR | N/A (not running yet) |
| Module CR created/updated | Watch CR status, project configuration updates | Reconcile module operands |
| DSC configuration changes | Update Module CR spec with new configuration | Reconcile affected operands |
| Module operand fails | Watch CR status | Update conditions (Ready=False, Degraded=True) |
| DSC disables module | Delete Module CR, prune controller resources | Graceful shutdown, cleanup operands |

**Implementation Details:**

All detailed requirements, including complete CRD specifications, validation rules, manifest types, dependency management, and integration patterns are documented in the [Module Onboarding Guide](design/module-onboarding-guide.md).

## Illustrative Example

**Note:** This example illustrates the basic architectural pattern in simplified form. For complete step-by-step walkthroughs with full YAML specifications and additional scenarios, see the [Module Onboarding Guide](design/module-onboarding-guide.md#53-complete-lifecycle-flow-example).

**Basic Flow:**

1. **User enables module in DataScienceCluster:**
   ```yaml
   spec:
     components:
       kserve:
         managementState: Managed
   ```

2. **ODH Operator (meta-operator) actions:**
   - Deploys Kserve module controller (Deployment, RBAC, CRD)
   - Creates Kserve CR with projected configuration from DSC/DSCI

3. **Kserve module controller (independent operator) actions:**
   - Reconciles Kserve CR
   - Deploys operands (kserve-controller, odh-model-controller, odh-maas-controller, Services, Routes, etc.)
   - Reports status back to Kserve CR

4. **ODH Operator aggregates status to DSC:**
   - Module CR `Ready` condition becomes `KserveReady` in DSC
   - Module CR `releases` copied to DSC component status

**Key Patterns:**
- **Configuration updates:** ODH Operator projects DSC changes to Module CR spec
- **Operand failures:** Module controller sets Ready=False, Degraded=True
- **Platform enforcement:** Manual edits to platform-managed fields are automatically reverted

See [Module Onboarding Guide](design/module-onboarding-guide.md#53-complete-lifecycle-flow-example) for:
- Complete step-by-step walkthroughs with full YAML
- Configuration update scenarios
- Failure handling examples
- Platform enforcement demonstrations

## Open Questions

N/A

## Responsibility

The ODH Platform team is responsible for maintaining this architecture and providing shared utilities to facilitate module development.

## Alternatives

N/A

## Stakeholder Impacts

| Group                     | Key Contacts                    | Date       | Impacted? |
| ------------------------- | ------------------------------- | ---------- | --------- |
| ODH Platform Team         | @lphiri, @burgazzoli            | 2026/01/19 | YES       |


## References

- [Detailed Module Onboarding Guide](design/module-onboarding-guide.md)
- [ODH-ADR-Operator-0006: Internal API](ODH-ADR-Operator-0006-internal-api.md)
- [ODH-ADR-Operator-0008: Resource Lifecycle Management](ODH-ADR-Operator-0008-resources-lifecycle.md)
