# KEP: RHOAI Organization and Capability CRD Design

<!--
This is a design-exploration document in CNCF/Kubernetes KEP style. It is not
yet assigned a Kubernetes Enhancement Proposal number and is not part of the
formal ADR set.
-->

- Authors: RHOAI Platform
- Status: Provisional design discussion
- Discussion issue: TBD
- Last updated: 2026-09-10

## Summary

RHOAI needs a Kubernetes-native representation of organizational tenancy,
delegated administration, namespace provisioning, shared tenant configuration,
and optional platform capabilities such as MaaS and Observability.

This KEP evaluates three API-shape options:

1. Embed service configuration in a central profile.
2. Use a generic service-binding resource.
3. Use capability-specific resources such as `TenantMaaS`.

It also evaluates replacing public `Tenant` terminology with `Organization`
terminology and removing the tenancy operator's direct dependency on the MaaS
`AITenant` CRD.

## Motivation

The current model uses `PlatformTenant`, `TenantProfile`, and `TenantProject`.
The model correctly expresses hierarchy and delegation, but service-specific
configuration risks accumulating in `TenantProfile`. MaaS also introduces a
cross-team dependency because the tenancy controller currently creates and
watches the MaaS-owned `AITenant` resource.

The design must support tenancy semantics without making the Kubernetes API
depend on overloaded or implementation-specific names.

## Goals

- Represent the organizational tenancy boundary and hierarchy.
- Keep the identity and hierarchy CRD small and stable.
- Support independently configured platform capabilities.
- Give each capability independent schema, authorization, status, and lifecycle.
- Establish a one-way dependency between MaaS and the tenancy API.
- Allow `AITenant` to be removed later without changing the tenancy API.
- Provide a clear migration path from `Tenant*` to `Organization*` names.
- Share common OIDC and Gateway configuration across MaaS and Observability.

## Non-Goals

- Define the internals of the MaaS controller.
- Change the MaaS `AITenant` schema as part of the tenancy API design.
- Define resource quotas, GPU fairness, or Kueue integration.
- Solve multi-cluster organization federation.
- Define an end-user billing or chargeback system.

## Proposal

### Public naming

Use full Organization-based kinds for new or pre-release APIs:

```text
Organization
OrganizationProfile
OrganizationProject
OrganizationMaaS
```

The API still represents tenancy. The documentation defines an Organization as
the framework's tenancy identity, delegation, isolation, and service scope.

Use references and labels consistently:

```yaml
spec:
  organizationRef:
    name: nlp-team
```

```text
organization.opendatahub.io/*
```

If compatibility requires it, the existing `tenancy.opendatahub.io` API group
can remain while the resource kinds change. If the API is still entirely
pre-release, changing the group to `organization.opendatahub.io` can be
considered separately.

### Core resources

```text
Organization
  hierarchy and canonical identity

OrganizationProfile
  tenant-wide policy, defaults, and shared platform configuration

OrganizationProject
  namespace/project provisioning

OrganizationMaaS
  independently managed MaaS intent and configuration
```

The current implementation names remain `PlatformTenant`, `TenantProfile`,
`TenantProject`, and `TenantMaaS` until the naming migration is approved.

### Capability-specific MaaS resource

The existence of `OrganizationMaaS` enables MaaS for one root organization.
The reference is immutable and the resource is root-only.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: OrganizationMaaS
metadata:
  name: nlp-team
spec:
  organizationRef:
    name: nlp-team
  quotas:
    maxModels: 20
    maxSubscriptions: 100
    maxApiKeys: 50
```

MaaS-specific quotas and service-specific client settings belong on this
resource. Shared OIDC issuer and Gateway configuration do not.

### Shared platform configuration

OIDC and Gateway configuration is required by multiple capabilities, including
MaaS and Observability. The initial shape can be a small section on the profile:

```yaml
kind: OrganizationProfile
spec:
  organization: nlp-team
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    gateway:
      name: tenant-gateway
      namespace: openshift-ingress
```

If this section grows, it should become an independent
`OrganizationPlatformConfig` resource rather than making the profile a general
service registry.

The issuer is a strong candidate for shared configuration. Client IDs may
remain capability-specific if MaaS and Observability use separate OIDC clients.

### MaaS dependency boundary

The target dependency graph is:

```text
MaaS operator → Organization / OrganizationProfile / OrganizationMaaS
MaaS operator → AITenant, while AITenant exists
```

The tenancy operator must not import, watch, create, or update `AITenant`.
The MaaS operator watches `OrganizationMaaS`, validates the referenced
organization and administrators using read-only access to the organization API,
and owns the adapter from `OrganizationMaaS` to `AITenant`.

When `AITenant` is removed, the MaaS operator reconciles `OrganizationMaaS`
directly. The organization API does not change.

## Custom resource examples and usage

The examples in this section use the proposed Organization-based names. During
the transition, the equivalent current proof-of-concept kinds are
`PlatformTenant`, `TenantProfile`, `TenantProject`, and `TenantMaaS`.

### Create a root Organization

Root Organizations are created by a cluster administrator. A root has no
`spec.parent` and can contain child Organizations and Projects.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: Organization
metadata:
  name: research
  labels:
    organization.opendatahub.io/managed-by: tenancy-controller
spec:
  displayName: Research Division
```

```sh
kubectl apply -f organization-research.yaml
kubectl get organizations research
kubectl get organization research -o yaml
```

Expected observed status is illustrative:

```yaml
status:
  root: research
  phase: Ready
  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
```

### Create a child Organization

A child Organization is created under a parent by an authorized parent or
ancestor administrator. The parent reference is immutable.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: Organization
metadata:
  name: nlp-team
spec:
  displayName: NLP Team
  parent: research
```

```sh
kubectl apply -f organization-nlp-team.yaml
kubectl get organizations
kubectl get organization nlp-team -o jsonpath='{.status.root}{"\n"}'
```

The controller creates an initial restrictive `OrganizationProfile` for the
new Organization. The profile starts with no administrators, no projects, and
restrictive network defaults until the parent bootstraps administrators.

### Configure Organization policy

The profile contains tenant-wide policy and shared platform configuration. It
does not contain MaaS-specific quotas or other capability-specific blocks.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: OrganizationProfile
metadata:
  name: nlp-team
spec:
  organization: nlp-team
  admins:
    - kind: Group
      name: nlp-team-admins
  defaults:
    networkIsolation: tenant
    maxProjects: 10
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    gateway:
      name: tenant-gateway
      namespace: openshift-ingress
  observability:
    dashboards: false
    dedicatedMonitoring:
      enabled: false
```

```sh
kubectl apply -f organization-profile-nlp-team.yaml
kubectl get organizationprofile nlp-team
kubectl describe organizationprofile nlp-team
```

If shared configuration later moves to `OrganizationPlatformConfig`, the
profile retains the policy fields and the platform configuration becomes:

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: OrganizationPlatformConfig
metadata:
  name: nlp-team
spec:
  organizationRef:
    name: nlp-team
  oidc:
    issuerUrl: https://sso.example.com/realms/rhoai
  gateway:
    name: tenant-gateway
    namespace: openshift-ingress
```

### Provision an Organization Project

An Organization Project maps one-to-one to a Kubernetes Namespace or OpenShift
Project. The controller creates namespace labels, RoleBindings, and owned
NetworkPolicies.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: OrganizationProject
metadata:
  name: sentiment-analysis
spec:
  organizationRef:
    name: nlp-team
  users:
    - kind: Group
      name: nlp-team
      role: edit
    - kind: User
      name: research-viewer
      role: view
  networkIsolation: tenant
  networkGrants:
    - to: nlp-team/shared-data
      direction: egress
      ports:
        - 8080
```

```sh
kubectl apply -f organization-project-sentiment.yaml
kubectl get organizationproject sentiment-analysis
kubectl get namespace sentiment-analysis --show-labels
kubectl get networkpolicies -n sentiment-analysis
```

Illustrative observed status:

```yaml
status:
  namespace: sentiment-analysis
  conditions:
    - type: NamespaceReady
      status: "True"
      reason: Reconciled
    - type: NetworkPolicyReady
      status: "True"
      reason: Reconciled
```

### Request MaaS for an Organization

The existence of `OrganizationMaaS` requests MaaS. It is independently
configured from the Organization profile and is valid only for a root
Organization.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: OrganizationMaaS
metadata:
  name: research
spec:
  organizationRef:
    name: research
  oidc:
    clientId: research-maas
  quotas:
    maxModels: 20
    maxSubscriptions: 100
    maxApiKeys: 50
```

The service adapter resolves the shared issuer and Gateway from
`OrganizationProfile` or `OrganizationPlatformConfig`:

```sh
kubectl apply -f organization-maas-research.yaml
kubectl get organizationmaas research
kubectl describe organizationmaas research
```

Illustrative observed status:

```yaml
status:
  aiTenant: research
  ready: true
  conditions:
    - type: Ready
      status: "True"
      reason: MaaSReady
```

Creating `OrganizationMaaS` for a child Organization is rejected:

```yaml
spec:
  organizationRef:
    name: nlp-team
```

The webhook returns an error similar to:

```text
MaaS provisioning is supported only for root Organizations
```

### Transitional generated AITenant

While `AITenant` still exists, the MaaS operator may render an implementation
resource from the Organization APIs. This resource is not authored by users of
the tenancy API.

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: AITenant
metadata:
  name: research
  namespace: ai-tenants
  labels:
    organization.opendatahub.io/name: research
    maas.opendatahub.io/managed-by: maas-operator
  ownerReferences:
    - apiVersion: tenancy.opendatahub.io/v1alpha1
      kind: OrganizationMaaS
      name: research
      controller: true
      blockOwnerDeletion: true
spec:
  oidc:
    issuerUrl: https://sso.example.com/realms/rhoai
    clientId: research-maas
  gateway:
    name: tenant-gateway
  resourceQuotas:
    maxModels: 20
    maxSubscriptions: 100
    maxApiKeys: 50
```

The tenancy operator does not create or watch this resource. The MaaS operator
owns the adapter and can replace it later without changing the Organization
API.

### Typical administrative workflow

```sh
# 1. Create the hierarchy.
kubectl apply -f organization-research.yaml
kubectl apply -f organization-nlp-team.yaml

# 2. Bootstrap OrganizationProfile administrators.
kubectl apply -f organization-profile-nlp-team.yaml

# 3. Let an Organization administrator create a project.
kubectl --as=alice --as-group=nlp-team-admins \
  apply -f organization-project-sentiment.yaml

# 4. Request MaaS for the root Organization.
kubectl apply -f organization-maas-research.yaml

# 5. Inspect the complete Organization subtree.
kubectl get organizations,organizationprofiles,organizationprojects,organizationmaas
```

### Delete and cleanup

Deleting an Organization Project removes the resources owned by that project
controller. Deleting `OrganizationMaaS` removes MaaS resources owned by the MaaS
operator. Existing unmanaged namespaces and manually created MaaS resources are
not adopted or deleted automatically.

```sh
kubectl delete organizationmaas research
kubectl delete organizationproject sentiment-analysis
kubectl delete organization nlp-team
```

Deletion authorization remains subject to the validating webhook and the
Organization administrator hierarchy.

## Alternatives considered

### Embed service configuration in the profile

```yaml
spec:
  services:
    maas: ...
    observability: ...
```

This has the fewest resource kinds initially but causes the central profile to
grow with every platform service. It mixes service ownership, validation, and
lifecycle. Rejected as the default pattern.

### Generic service binding

```yaml
kind: OrganizationServiceBinding
spec:
  organizationRef:
    name: nlp-team
  serviceRef:
    group: services.opendatahub.io
    kind: MaaSService
    name: nlp-team-maas
```

This provides a common attachment mechanism but adds indirection and often
requires another service-specific resource for schema and status. It is useful
when capabilities have nearly identical lifecycles, but less suitable when
each capability is independently configured.

### Keep Tenant terminology

Keeping `PlatformTenant` and related names preserves current terminology and
avoids migration. It is valid if the project wants to emphasize the tenancy
domain, but it retains ambiguity with Kubernetes and MaaS concepts.

### Short `Org` kinds

```text
OrgProfile
OrgProject
OrgMaaS
```

Short names are concise but less self-documenting in Kubernetes output, RBAC,
and events. Full `Organization*` kinds are preferred for public APIs.

## Design details

### Authorization

Capability resources resolve their organization reference and use the
organization's administrators. Root-only capabilities are rejected by the
capability webhook. A MaaS webhook needs read-only access to the organization
resources but does not need permission to modify them.

### Ownership

The capability-owning operator owns capability implementation resources. For
the transition, the MaaS operator owns `AITenant`. The tenancy operator owns
organization resources and does not own MaaS implementation resources.

### Status

Capability status belongs on the capability resource. MaaS readiness should be
reported on `OrganizationMaaS`, not copied into the general organization
profile. Shared platform configuration status belongs on the resource that owns
that configuration.

### Lifecycle

Creating `OrganizationMaaS` requests MaaS. Deleting it removes MaaS resources
owned by the MaaS operator. Existing manually created MaaS resources remain
untouched unless an explicit adoption mechanism is introduced.

## Upgrade and migration

If the current API is still pre-release, rename kinds and fields before
production adoption:

```text
PlatformTenant  → Organization
TenantProfile   → OrganizationProfile
TenantProject   → OrganizationProject
TenantMaaS      → OrganizationMaaS
tenantRef       → organizationRef
tenant labels   → organization labels
```

If resources are already consumed, introduce conversion or compatibility APIs.
Do not perform a textual rename of JSON fields without a conversion strategy.

The `AITenant` adapter should move from the tenancy operator to the MaaS
operator before the tenancy API is considered stable.

## Security considerations

- Capability webhooks must fail closed when the organization reference is
  missing or unauthorized.
- Service operators need read-only access to organization identity and policy.
- Capability operators must not gain permission to modify organization
  hierarchy or administrator assignments.
- Generated implementation resources must be protected from unowned edits.
- Tenant or organization names must not be used as an authorization shortcut;
  authorization must resolve the referenced resource and administrators.

## Scalability and performance

Capability controllers should watch only their own capability resources and
required shared organization resources. The tenancy controller should not watch
all service implementation CRDs. This limits cache size and prevents optional
services from increasing the core controller's reconciliation surface.

## Observability

Organization and capability resources should expose conditions for validation,
provisioning, and readiness. Common organization labels should be propagated to
managed namespaces and capability resources for metrics attribution.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Too many CRDs | Add capability CRDs only for independently configured services. |
| Profile growth | Keep shared configuration small; promote it to `OrganizationPlatformConfig` when needed. |
| Cross-team API coupling | Keep the MaaS adapter in the MaaS operator. |
| Naming migration cost | Rename before API stability, or provide conversion APIs. |
| Duplicate tenant identities | Make Organization the canonical identity and treat `AITenant` as transitional. |

## Graduation criteria

Before the design is considered stable:

- Naming is approved by platform, MaaS, and Observability teams.
- Shared OIDC and Gateway ownership is defined.
- `OrganizationMaaS` authorization is tested against organization admins.
- The tenancy operator has no direct dependency on `AITenant`.
- MaaS can remove `AITenant` without changing the organization API.
- Existing unmanaged MaaS resources remain unaffected.

## Open questions

- Should the API group remain `tenancy.opendatahub.io` after the kind rename?
- Should shared configuration begin on `OrganizationProfile` or use
  `OrganizationPlatformConfig` immediately?
- Is Gateway configuration shared across all capabilities?
- Is the OIDC issuer shared while client IDs remain capability-specific?
- Should ancestor administrators manage capability resources directly?
- Which team owns the `OrganizationMaaS` API and its webhook?

## Implementation history

- Initial design used `PlatformTenant`, `TenantProfile`, and `TenantProject`.
- MaaS configuration was first embedded as `TenantProfile.spec.services.maas`.
- The design moved to capability-specific `TenantMaaS`.
- The implementation introduced a transitional tenancy-side `AITenant` adapter.
- The target architecture moves that adapter into the MaaS operator.
- Public naming is now under discussion in favor of Organization-based kinds.
