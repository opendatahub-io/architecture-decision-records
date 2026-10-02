# Open Data Hub - Per-Module Namespace Isolation

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2026-09-08                       |
| Scope          | Open Data Hub Operator           |
| Status         | Approved                         |
| Authors        | [Luca Burgazzoli](@burgazzoli)   |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        | [RHOAIENG-90465](https://redhat.atlassian.net/browse/RHOAIENG-90465) |
| Other docs:    | [Module Onboarding Architecture](ODH-ADR-Operator-0012-module-onboarding.md) |

## What

This ADR defines per-module namespaces as the ownership and isolation boundary for module operators and their operands. It extends [ODH-ADR-Operator-0012](ODH-ADR-Operator-0012-module-onboarding.md), which remains authoritative for the broader module-controller architecture and responsibility boundaries.

Each module operator runs in a dedicated system namespace. The module operator is always installed in that namespace, while the module team decides whether its operands run there or in one or more additional module-owned namespaces.

Namespaces used by a module are reserved for resources belonging to that module and its operands. This gives namespace-scoped resources, policies, and permissions a module-specific scope and makes ownership and cleanup easier to reason about instead of coupling them to a shared namespace. They are not user workload namespaces and are not, by themselves, a universal security or lifecycle boundary.

Network policies are one enforcement mechanism applied to these namespaces. The exact NetworkPolicy boundaries, bootstrap behavior, and ongoing reconciliation are to be defined in a separate ADR and are out of scope for this ADR.

## Why

The shared `redhat-ods-applications` namespace makes it difficult to establish independent module boundaries. In particular:

* Network policies and other namespace-scoped policies are shared across unrelated modules.
* A change made for one module can unintentionally affect another module.
* Roles, bindings, and permissions are harder to scope and reason about when unrelated module resources share a namespace.
* Resource ownership, cleanup, quotas, admission behavior, and namespace-specific integrations become coupled to a shared legacy namespace.
* Module teams cannot independently define the boundary and operational ownership of their operator and operands.

Per-module namespaces reduce this coupling. They give each module a dedicated place for its resources and allow module teams to define the policies, permissions, and lifecycle behavior appropriate to their workloads while keeping platform-wide responsibilities separate.

## Goals

* Place each module operator in a dedicated system namespace.
* Allow module operands to run in the operator namespace or in additional module-owned namespaces.
* Keep each module-owned namespace dedicated to the module and its operands, so module controllers can manage and clean up namespace-scoped resources and policies without affecting unrelated modules.
* Allow the ODH Operator to apply module-scoped baseline resources and policies, such as dedicated RBAC resources and a default-deny NetworkPolicy, without sharing those baselines across modules.
* Give each module team a dedicated namespace for defining module-specific policies and access controls, while leaving the exact roles and permissions to the module design.

## Non-Goals

* Defining the detailed NetworkPolicy contract, including exact boundaries, bootstrap sequencing, readiness checks, and policy reconciliation. These are to be defined in a separate ADR and are out of scope for this ADR.
* Defining module-specific authorization, including Roles, RoleBindings, ClusterRoles, ClusterRoleBindings, or other permission sets. Cluster-scoped permissions remain a separate concern.
* Defining detailed ownership, precedence, conflict resolution, and reconciliation rules for specific policy domains. Those rules belong in dedicated topic ADRs, such as networking or permissions.
* Defining how module controllers create, identify, validate, or delete additional operand namespaces. Those lifecycle mechanics depend on each module's operand topology and remain module-defined.
* Replacing the module-controller responsibility and manifest-packaging decisions defined by ADR-0012.
* Defining egress, metrics, observability, or other adjacent policy concerns.
* Defining workload or data migration, rollout sequencing, namespace transition procedures, or project-management gates.
* Defining a universal namespace security or tenant-isolation model. A namespace alone does not replace NetworkPolicies, RBAC review, cluster-scoped authorization, or other platform security controls.

## How

### Module operator namespace

The ODH Operator creates and initializes one dedicated system namespace for each module operator. The ODH Operator derives the module's default namespace from the module's preferred namespace in registration metadata, or uses `redhat-ai-${module-name}-system` when no preferred namespace is declared. A user may override this default through ODH Operator configuration or environment variables. Namespace selection is not part of a top-level CR such as the `DataScienceCluster` because the ODH Operator must resolve the namespace before configuring its cache and watch scope.

The ODH Operator enforces one module operator per resolved system namespace. If the user override or derived default namespace is already associated with another module, the ODH Operator detects the collision before bootstrap, reports the provisioning failure on the applicable top-level CR when one is available, such as the `DataScienceCluster`, and emits a Kubernetes event. The ODH Operator does not deploy the module, adopt the conflicting namespace, or modify resources belonging to the other module.

### Operand placement

The module operator is responsible for its operands and may deploy them:

* in the module operator's system namespace; or
* in one or more additional, module-owned operand namespaces.

For example, a KServe module operator may run in the system namespace `redhat-ai-kserve-system`, while its KServe controller and ODH-specific controller operands run in the dedicated operand namespace `redhat-ai-kserve`. In this case, `redhat-ai-kserve` is a child namespace owned by the parent module and is reserved for those module operands.

Additional operand namespaces must be dedicated to, and logically owned by, the parent module. A module must not create, adopt, or impose its module-specific policies on a namespace containing unrelated user workloads. The mechanism used to create, identify, validate, and delete these namespaces is module-defined.

### Module-scoped platform baselines

The ODH Operator may apply platform-mandated baseline resources and policies to a module's dedicated namespaces. Examples include module-specific Roles and RoleBindings, and ClusterRoles referenced by module-specific RoleBindings when broader permissions are required, as well as a default-deny NetworkPolicy. These baselines must be associated with the owning module and must not be shared with unrelated modules or user workloads.

The ODH Operator is responsible only for the baselines it owns. The module controller remains responsible for its operand resources and module-specific runtime policies. This ADR establishes that high-level split, but the detailed ownership, precedence, conflict resolution, and reconciliation rules for each policy domain are outside its scope and must be defined in dedicated topic ADRs.

### Ownership and permissions

The module team owns the resources and operational policies for the module operator and its operands, subject to any platform-mandated baselines owned by the ODH Operator. The module controller remains responsible for the lifecycle of its operands and for choosing the permissions required to reconcile them.

The per-module namespace provides a natural scope for namespace-scoped Roles and RoleBindings and makes permission ownership easier to audit. This does not eliminate the need for cluster-scoped permissions where they are required, and this ADR does not prescribe the exact RBAC design of any module.

The ODH Operator remains responsible for the lifecycle of module controller installation and for platform-wide concerns defined by ADR-0012. It must not take ownership of module operands merely because they run in a module-owned namespace.

## Alternatives

### Keep all modules in `redhat-ods-applications`

This preserves the current deployment model but retains shared policy, permission, ownership, and lifecycle boundaries that couple unrelated modules.

### Use a dedicated namespace only for the module operator

This would isolate the operator but leave operand resources coupled to a shared or unrelated namespace. It would not provide a complete ownership boundary for the module.

### Make the ODH Operator manage all module resources and permissions

This centralizes management but expands the ODH Operator's scope, increases its required permissions, and couples it to module-specific behavior. It conflicts with the independent module-controller architecture.

## Security and Privacy Considerations

Per-module namespaces reduce accidental interaction between unrelated module resources and provide a clearer basis for namespace-scoped policies and permissions. They do not replace NetworkPolicies, RBAC review, cluster-scoped authorization, or other platform security controls.

The exact NetworkPolicy boundaries and network isolation behavior are to be defined in a separate ADR and are out of scope for this ADR. This ADR does not define egress behavior or module-specific policy rules.

## Risks

* Additional namespaces increase the operational surface that module teams must own and monitor.
* Cross-namespace communication and permissions must be explicitly designed by each module.
* Moving resources from the shared namespace may require workload and data migration planning, which is outside this ADR.

## Stakeholder Impacts

| Group                  | Key Contacts | Date       | Impacted? |
| ---------------------- | ------------ | ---------- | --------- |
| ODH Platform Team      |              | 2026/09/08 | YES       |
| Module Teams            |              | 2026/09/08 | YES       |
| Security/Productivity   |              | 2026/09/08 | YES       |

## References

* [RHOAIENG-90465: Network Policies for RHOAI Operands](https://redhat.atlassian.net/browse/RHOAIENG-90465)
* [ODH-ADR-Operator-0012: Module Onboarding Architecture](ODH-ADR-Operator-0012-module-onboarding.md)
* [Module Onboarding Guide](design/module-onboarding-guide.md)
