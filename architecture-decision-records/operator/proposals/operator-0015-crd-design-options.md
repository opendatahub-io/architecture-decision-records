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
3. Use capability-specific resources such as `MaaSConfiguration`.

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

## Reference design

The baseline multi-tenancy framework and strategy are maintained in the draft
Operator-0015 pull request. This KEP is a focused design exploration and does
not duplicate those documents:

[Operator-0015 draft PR #156](https://github.com/opendatahub-io/architecture-decision-records/pull/156/changes)

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

Use descriptive, Organization-based kinds for identity resources and a
capability-specific kind for MaaS:

```text
Organization
OrganizationProfile
OrganizationProject
MaaSConfiguration
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

Display-only names use metadata annotations rather than API fields; they do not
affect organization placement, authorization, or reconciliation.

Because the current implementation is only a proof of concept and no
`tenancy.opendatahub.io` API has shipped, use a clean API group:

```text
organization.opendatahub.io/v1alpha1
```

No conversion webhook, compatibility CRD, or field-preservation strategy is
required for the POC rename.

The CRDs retain these full kinds as their canonical API names and expose the
following lowercase `kubectl` short names through API discovery:

| Kind | Resource name | Short name |
|---|---|---|
| `Organization` | `organizations` | `org` |
| `OrganizationProfile` | `organizationprofiles` | `orgprof` |
| `OrganizationProject` | `organizationprojects` | `orgproj` |
| `MaaSConfiguration` | `maasconfigurations` | `maascfg` |

For example, the complete organization view can be inspected with:

```sh
kubectl get org,orgprof,orgproj,maascfg
```

### Core resources

```text
Organization
  hierarchy and canonical identity

OrganizationProfile
  tenant-wide policy, defaults, and shared platform configuration

OrganizationProject
  namespace/project provisioning

MaaSConfiguration
  independently managed MaaS intent and configuration
```

The proof-of-concept names are `PlatformTenant`, `TenantProfile`,
`TenantProject`, and `TenantMaaS`. They are replaced by the proposed
Organization-based names before the API is shipped. MaaS uses the
capability-specific name `MaaSConfiguration` because its `organizationRef`
already identifies the organization it serves.

### Capability-specific MaaS resource

The existence of `MaaSConfiguration` enables MaaS for one root organization.
The reference is immutable and the resource is root-only. The MaaS admission
webhook validates the root-only constraint, the immutable reference, and the
one-configuration-per-root constraint; readiness is reported later through
status conditions.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: MaaSConfiguration
metadata:
  name: research
spec:
  organizationRef:
    name: research
  quotas:
    maxModels: 20
    maxSubscriptions: 100
    maxApiKeys: 50
```

MaaS-specific quotas and service-specific client settings belong on this
resource. Shared OIDC issuer and ingress Gateway configuration do not.

### Shared platform configuration

OIDC and ingress Gateway configuration is required by multiple capabilities,
including MaaS and Observability. The initial shape can be a small section on
the profile. In this document, Gateway means the shared Kubernetes Gateway API
`Gateway` used for tenant HTTP ingress; it does not mean the AI Gateway, MCP
Gateway, or another application-level gateway.

```yaml
kind: OrganizationProfile
spec:
  organizationRef:
    name: nlp-team
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    ingressGatewayRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: tenant-gateway
      namespace: openshift-ingress
```

If this section grows, it should become an independent
`OrganizationPlatformConfig` resource rather than making the profile a general
service registry.

The issuer is a strong candidate for shared configuration. Client IDs may
remain capability-specific if MaaS and Observability use separate OIDC clients.

Observability should follow the same ownership pattern as MaaS when its API is
stabilized: an observability capability resource is reconciled by the
observability operator, reads Organization identity and policy, and reports its
own status. The profile's observability block is therefore transitional shared
configuration, not a commitment that the tenancy operator will own
observability resources.

### MaaS dependency boundary

The target dependency graph is:

```text
MaaS operator → Organization / OrganizationProfile / MaaSConfiguration
MaaS operator → AITenant, while AITenant exists
```

The tenancy operator must not import, watch, create, or update `AITenant`.
The MaaS operator owns and reconciles `MaaSConfiguration`, validates the
referenced organization and administrators using read-only access to the
organization API, and owns the adapter from `MaaSConfiguration` to `AITenant`.
The tenancy operator owns only `Organization`, `OrganizationProfile`, and
`OrganizationProject`.

When `AITenant` is removed, the MaaS operator reconciles
`MaaSConfiguration` directly. The organization API does not change.

## Custom resource examples and usage

The examples in this section use the proposed Organization-based names. During
the transition, the equivalent current proof-of-concept kinds are
`PlatformTenant`, `TenantProfile`, `TenantProject`, and `TenantMaaS`.

### Create a root Organization

Root Organizations are created by a cluster administrator. A root has no
`spec.parent` and can contain child Organizations and Projects.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: Organization
metadata:
  name: research
  labels:
    organization.opendatahub.io/managed-by: tenancy-controller
  annotations:
    organization.opendatahub.io/display-name: Research Division
spec: {}
```

```sh
kubectl apply -f organization-research.yaml
kubectl get org research
kubectl get org research -o yaml
```

Expected observed status is illustrative:

```yaml
status:
  root: research
  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
```

### Create a child Organization

A child Organization is created under a parent by an authorized parent or
ancestor administrator. The parent reference is immutable.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: Organization
metadata:
  name: nlp-team
  annotations:
    organization.opendatahub.io/display-name: NLP Team
spec:
  parent: research
```

```sh
kubectl apply -f organization-nlp-team.yaml
kubectl get org
kubectl get org nlp-team -o jsonpath='{.status.root}{"\n"}'
```

The controller creates an initial restrictive `OrganizationProfile` for the
new Organization. The profile starts with no administrators, no projects, and
restrictive network defaults until the parent bootstraps administrators.

### Configure Organization policy

The profile contains tenant-wide policy and shared platform configuration. It
does not contain MaaS-specific quotas or other capability-specific blocks.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: OrganizationProfile
metadata:
  name: nlp-team
spec:
  organizationRef:
    name: nlp-team
  admins:
    - kind: Group
      name: nlp-team-admins
  defaults:
    networkIsolation: tenant
    maxProjects: 10
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    ingressGatewayRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: tenant-gateway
      namespace: openshift-ingress
  observability:
    dashboards: false
    dedicatedMonitoring:
      enabled: false
```

```sh
kubectl apply -f organization-profile-nlp-team.yaml
kubectl get orgprof nlp-team
kubectl describe orgprof nlp-team
```

If shared configuration later moves to `OrganizationPlatformConfig`, the
profile retains the policy fields and the platform configuration becomes:

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: OrganizationPlatformConfig
metadata:
  name: nlp-team
spec:
  organizationRef:
    name: nlp-team
  oidc:
    issuerUrl: https://sso.example.com/realms/rhoai
  ingressGatewayRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: tenant-gateway
    namespace: openshift-ingress
```

### Provision an Organization Project

An Organization Project maps one-to-one to a Kubernetes Namespace or OpenShift
Project. The controller creates namespace labels, RoleBindings, and tenant
baseline NetworkPolicies that it owns. It does not own module or operand
NetworkPolicies; those remain with the responsible component controller under
the [module NetworkPolicy platform contract](https://github.com/opendatahub-io/architecture-decision-records/pull/164).
The policies are additive and no policy object has more than one lifecycle
manager.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
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

Cross-organization grants require the administrators of both the source and
target projects to approve the same grant before either side is changed. The
controller creates only additive policies that it owns and never deletes a
policy owned by a component or another controller.

```sh
kubectl apply -f organization-project-sentiment.yaml
kubectl get orgproj sentiment-analysis
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

The existence of `MaaSConfiguration` requests MaaS. It is independently
configured from the Organization profile and is valid only for a root
Organization.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: MaaSConfiguration
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

The service adapter resolves the shared issuer and ingress Gateway from
`OrganizationProfile` or `OrganizationPlatformConfig`:

```sh
kubectl apply -f organization-maas-research.yaml
kubectl get maascfg research
kubectl describe maascfg research
```

Illustrative observed status:

```yaml
status:
  ready: true
  conditions:
    - type: Ready
      status: "True"
      reason: MaaSReady
```

`MaaSConfiguration` is a capability request, not a request to change the
DataScienceCluster component configuration. In an RHOAI deployment, the MaaS
operator requires `aigateway.modelsAsAService: Managed` (or the equivalent
component-management signal) before provisioning begins. It does not force
that setting. If the capability is not managed, the resource remains present
but reports a `CapabilityNotManaged` condition and no `AITenant` or MaaS-owned
runtime resources are created. Structural constraints, such as root-only
scope, remain webhook validation; this component-availability check is runtime
status.

Creating `MaaSConfiguration` for a child Organization is rejected at webhook
admission, before the MaaS operator reconciles it:

```yaml
spec:
  organizationRef:
    name: nlp-team
```

The webhook returns an error similar to:

```text
MaaSConfiguration is supported only for root Organizations
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
    - apiVersion: organization.opendatahub.io/v1alpha1
      kind: MaaSConfiguration
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

The legacy `AITenant.spec.gateway` value is populated from the shared
`ingressGatewayRef`; it refers to the same Kubernetes Gateway API ingress
object and not to an AI Gateway or MCP Gateway.

The tenancy operator does not create or watch this resource. The MaaS operator
owns the adapter and can replace it later without changing the Organization
API. During the transition, only the MaaS operator may create, update, or
delete this generated `AITenant`; the tenancy operator's old adapter must be
disabled before the MaaS operator takes ownership. The owner reference and
managed-by label are the handoff markers, and a cutover check must confirm that
no two reconcilers are writing the same `AITenant`.

The transition is ordered as follows:

1. Deploy the MaaS operator's `MaaSConfiguration` reconciler with read-only
   access to Organization resources and write access to its own `AITenant`
   adapter resources.
2. Stop and remove the tenancy operator's `AITenant` watch and reconciler;
   existing tenancy-side resources are not updated during the handoff.
3. Verify that each generated `AITenant` has the `MaaSConfiguration`
   owner-reference and MaaS managed-by label, then allow the MaaS operator to
   resume reconciliation.
4. After all generated resources are owned by the MaaS operator, remove the
   tenancy-side adapter code. If the `AITenant` CRD is later removed, the MaaS
   operator switches to direct `MaaSConfiguration` reconciliation without an
   Organization API change.

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
kubectl get org,orgprof,orgproj,maascfg
```

### Delete and cleanup

Deleting an Organization Project removes the resources owned by that project
controller. The only disable operation in this proposal is deleting the
`MaaSConfiguration`; there is no separate `enabled` field. Deletion is handled
by the MaaS operator's finalizer and follows this contract:

1. Stop accepting new subscriptions and API-key issuance for the organization.
2. Revoke active API keys while retaining revocation/audit records for 90 days.
3. Delete MaaS-owned tenant control-plane resources, including the MaaS API
   namespace and policies. Deployed models and model data are retained unless
   their own lifecycle explicitly deletes them.
4. Leave the shared Gateway untouched because it is not MaaS-owned.
5. Remove the finalizer only after the owned cleanup has completed.

Transient failures retain the finalizer, set a deletion-failure condition, and
are retried idempotently. If cleanup partially succeeds, subsequent retries
reconcile the remaining MaaS-owned resources; they never adopt or delete
unmanaged namespaces, manually created MaaS resources, or the shared Gateway.

```sh
kubectl delete maascfg research
kubectl delete orgproj sentiment-analysis
kubectl delete org nlp-team
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

The capability-owning operator owns the capability resource and its
implementation resources. The MaaS operator owns `MaaSConfiguration` and,
during the transition, its generated `AITenant`. The tenancy operator owns
Organization resources and does not own MaaS implementation resources.

### Status

Capability status belongs on the capability resource. MaaS readiness is
reported on `MaaSConfiguration`, not copied into the general organization
profile; the stable status uses conditions and does not expose the transitional
`AITenant` name. Shared platform configuration status belongs on the resource
that owns that configuration.

### Lifecycle

Creating `MaaSConfiguration` requests MaaS. The MaaS operator owns its
reconciliation and applies the deletion contract above. It waits for the
RHOAI AI Gateway MaaS capability to be managed and does not mutate that
component setting. Existing manually created MaaS resources remain untouched
unless an explicit adoption mechanism is introduced.

## Upgrade and migration

The current API is a proof of concept only. Rename kinds and fields before
shipping the proposed API:

```text
PlatformTenant  → Organization
TenantProfile   → OrganizationProfile
TenantProject   → OrganizationProject
TenantMaaS      → MaaSConfiguration
tenantRef       → organizationRef
tenant labels   → organization labels
```

Because no `tenancy.opendatahub.io` API has shipped, no conversion or
compatibility API is required. The POC manifests can be replaced directly with
the proposed API group and resource names.

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
| Naming migration cost | Complete the rename before the first shipped release. |
| Duplicate tenant identities | Make Organization the canonical identity and treat `AITenant` as transitional. |

## Graduation criteria

Before the design is considered stable:

- Naming is approved by platform, MaaS, and Observability teams.
- Shared OIDC and Gateway ownership is defined.
- `MaaSConfiguration` authorization is tested against organization admins.
- The tenancy operator has no direct dependency on `AITenant`.
- MaaS can remove `AITenant` without changing the organization API.
- Existing unmanaged MaaS resources remain unaffected.
- MaaS deletion, disable-as-delete, retry, and partial-failure behavior is
  covered by controller tests before the adapter ownership handoff.
- The AITenant cutover proves that only one controller reconciles each
  generated resource at a time.

## Open questions

- Should the proposed API group be `organization.opendatahub.io`, or should a
  broader platform API group be selected?
- Should shared configuration begin on `OrganizationProfile` or use
  `OrganizationPlatformConfig` immediately?
- Is Gateway configuration shared across all capabilities?
- Is the OIDC issuer shared while client IDs remain capability-specific?
- Should ancestor administrators manage capability resources directly?

## Implementation history

- Initial design used `PlatformTenant`, `TenantProfile`, and `TenantProject`.
- MaaS configuration was first embedded as `TenantProfile.spec.services.maas`.
- The design moved to the capability-specific `MaaSConfiguration` resource.
- The implementation introduced a transitional tenancy-side `AITenant` adapter.
- The target architecture moves that adapter into the MaaS operator.
- Public naming is now under discussion in favor of Organization-based kinds.
