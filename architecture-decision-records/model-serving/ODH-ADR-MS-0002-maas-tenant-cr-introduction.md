# ODH-ADR-MS-0002: MaaS Tenant CR Introduction for User Configuration

|                |            |
| -------------- | ---------- |
| Date           | 2026-04-21 |
| Scope          | Model Serving, Models As A Service |
| Status         | Draft |
| Authors        | Lindani Phiri, Ishita Sequeira, Jamie Land |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [ODH Operator PR #3412](https://github.com/opendatahub-io/opendatahub-operator/pull/3412), [MaaS PR #735](https://github.com/opendatahub-io/models-as-a-service/pull/735) |

## What

Introduction of a new `Tenant` custom resource for user-facing Models-as-a-Service platform configuration while retaining the existing `ModelsAsService` CR for internal ODH operator integration.

## Why

This architectural change addresses several design and operational concerns:

- **ODH Operator Design Conformance**: Align with ODH operator design principles by separating internal operator integration (`components.platform.opendatahub.io`) from user-facing configuration
- **Clear API Separation**: Establish distinct APIs for operator integration vs. user configuration, improving clarity and reducing coupling
- **Future Migration Preparation**: Prepare for eventual transition from cluster-scoped to namespace-scoped configuration with appropriate RBAC constraints  
- **Component Modularity**: Locate component-specific user configuration within the component itself, reducing coupling and improving modularity
- **Strategic Timing**: Current release has no existing user-facing configuration to migrate, allowing introduction of new user-facing configuration without future migration burden

## Goals

* Introduce namespace-scoped `Tenant` CR for user-facing configuration
* Retain `ModelsAsService` CR for internal ODH operator integration
* Enable self-bootstrapping MaaS platform via `maas-controller`
* Establish clear separation between operator integration and user configuration APIs
* Implement singleton enforcement to maintain current behavior
* Support namespace-scoped multi-tenancy architecture for future enhancement
* Maintain full backward compatibility with existing operator integration

## Non-Goals

* Complete multi-tenancy architecture implementation (requires RHOAI-level multi-tenant architecture completion first)
* Immediate production multi-tenant support (application remains single-tenant but uses tenant-ready naming conventions)
* Breaking changes to existing MaaS functionality during migration

## How

The implementation involves a dual-CR architecture across the ODH Operator and MaaS repositories:

### ODH Operator Changes
- Remove existing ModelsAsService controller; retain CRD and component handler for DSC enablement checks and status aggregation  
- Delegate Tenant CR creation to maas-controller; operator only reads Tenant status and deletes (eventually moved to maas-controller) it on MaaS disable  
- Aggregate MaaS health into DataScienceCluster status by reading Tenant conditions (replaces previous ModelsAsService CR-based status)  
- Manifests to deploy maas-controller (CRDs, RBAC, Deployment) when MaaS is enabled in DSC

### Implementation Approach

The migration moves implementation from the common platform API to the dedicated maas.opendatahub.io API:

* **Tenant Configuration CR Creation**: Establish a namespace-scoped `Tenant` Custom Resource. This is intended to be co-located with other CRs  (MaaSAuthPolicy, MaaSSubscription) in a tenant adminstration (control plane) namespace to customize:
  - API key policies  
  - External OIDC settings
  - Gateway configuration
  - Telemetry policies   
* **Singleton Pattern**: Enforce single tenant via CEL validation requiring name `default-tenant`. This is a temporary measure until a full tenancy architecture is defined.
* **Self-Bootstrapping**: maas-controller creates the default Tenant automatically
* **Cross-Namespace Resource Management**: Use tracking labels (`maas.opendatahub.io/tenant-name` and `maas.opendatahub.io/tenant-namespace`) for resources across different namespaces


### Dual-CR Architecture
- `ModelsAsService` CR (`components.platform.opendatahub.io`): Internal operator integration, cluster-scoped. This is the Maas global controller CR and follows the established pattern for other RHOAI components.
- `Tenant` CR (`maas.opendatahub.io`): User-facing configuration, namespace-scoped
- MaaS controller reconciles user configuration from `Tenant` CR independently
- `ModelsAsService` CRD (components.platform.opendatahub.io): Retained for DSC enablement checks and status aggregation; no CR instance is created on-cluster. Planned to be updated as a follow up in future releases.

## Alternatives

### Alternative 1: Single CR Approach (ModelsAsService only)
**Pros**: Simpler architecture, single API surface, no additional complexity  
**Cons**: Continues using internal ODH operator CRs for user configuration, doesn't prepare for future multi-tenancy, maintains tight coupling between operator and component concerns, violates separation of internal vs user-facing APIs

### Alternative 2: Complete ModelsAsService Replacement
**Pros**: Clean break, single user-facing API, removes internal CR complexity  
**Cons**: Breaks existing operator integration patterns, requires migration of all ODH operator component handling, higher risk and complexity for operator changes

### Alternative 3: Cluster-scoped Tenant CR  
**Pros**: Simpler RBAC, consistent with current ModelsAsService scope, easier cross-namespace access  
**Cons**: CRD scope is immutable, would require complex migration path for future namespace-scoped multi-tenancy, doesn't align with future architecture direction, limits RBAC isolation

### Alternative 4: Enhanced ModelsAsService CR
**Pros**: Leverages existing CR, no new API complexity, maintains operator integration  
**Cons**: Mixes internal operator concerns with user configuration, harder to evolve independently, doesn't prepare for namespace-scoped multi-tenancy

## Security and Privacy Considerations

- Namespace-scoped CRs enable future RBAC-constrained tenant isolation
- Cross-namespace resource management requires careful label-based tracking to prevent privilege escalation
- Self-bootstrapping mechanism must ensure appropriate security context for tenant creation

## Risks

* **Migration Complexity**: Risk of configuration inconsistencies during the transition period
* **Cross-Namespace Resource Management**: Potential for resource orphaning if tracking labels are not properly managed
* **Terminology Confusion**: Using "Tenant" terminology before full multi-tenancy implementation may confuse operators and developers. Recommend comprehensive documentation to clarify usage for our customers.


## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Platform Core Operator Team            | TBD              | TBD        | Yes |
| MaaS/Model Serving Team      | TBD              | TBD        | Yes |
| Documentation Team           | TBD              | TBD        | Yes |
| QE/Testing Team             | TBD              | TBD        | Yes |

**Impact Details:**
- **ODH Operator Team**: Minor changes to support dual-CR architecture, existing ModelsAsService integration preserved
- **MaaS/Model Serving Team**: New controller implementation for Tenant CR, additional reconciliation architecture  
- **Documentation Team**: New user configuration documentation, installation guide updates
- **QE/Testing Team**: Additional test scenarios for Tenant CR lifecycle, existing operator integration tests preserved

## References

* [ODH Operator PR #3412](https://github.com/opendatahub-io/opendatahub-operator/pull/3412) - Operator-side implementation
* [MaaS PR #735](https://github.com/opendatahub-io/models-as-a-service/pull/735) - Platform controller implementation  
* [Tenant CR Architecture Decision](https://gist.github.com/ishitasequeira/36ac5b301736ed2716fa94333a9e3cb3) - Detailed architectural analysis
* [MaaS Tenant CR Overview](https://gist.github.com/ishitasequeira/9eceab2c847f2ac48a50a1a14a24f2ab) - Technical implementation details

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|  Marius Danciu                    |     04-24-2026       |       |