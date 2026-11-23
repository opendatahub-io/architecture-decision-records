# KEP: RHOAI Organization and Capability CRD Design

<!--
This is a design-exploration document in CNCF/Kubernetes KEP style. It is not
yet assigned a Kubernetes Enhancement Proposal number and is not part of the
formal ADR set.
-->

- Authors: RHOAI Platform
- Status: Provisional design discussion
- Last updated: 2026-09-28

## Summary

RHOAI needs a Kubernetes-native representation of organizational tenancy,
delegated administration, namespace provisioning, shared organization
configuration, and optional platform capabilities such as MaaS and
Observability.

This KEP compares two MaaS API designs over the same `Organization`,
`OrganizationProfile`, and `OrganizationProject` foundation:

1. A separately managed `MaaSConfiguration` reconciled by the MaaS operator.
2. A MaaS request on `OrganizationProfile` composed into an `AITenant` by an
   isolated adapter, with explicit adoption of manually created tenants.

Both preserve a separate MaaS controller. The second design is an alternative
for discussion, not a replacement or a selected decision.

## Motivation

The Organization model expresses hierarchy and delegation. A separate MaaS
resource gives the capability its own schema, status, and lifecycle, while a
profile request gives administrators one higher-level Organization API. The
tenancy controller currently creates and watches the MaaS-owned `AITenant`
resource, mixing hierarchy reconciliation with capability composition.

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
- Compare a separate MaaS request with profile-based MaaS composition.
- Keep the core hierarchy controller and MaaS controller independently
  reconcilable.
- Keep `AITenant` an implementation resource under either API design.
- Allow MaaS to replace `AITenant` later without changing the selected
  user-facing request API.
- Evaluate in-place adoption of manually created `AITenant` resources for an
  upgrade to profile-based composition.
- Use consistent Organization-based names for the hierarchy resources.
- Share common OIDC and Gateway configuration across MaaS and Observability.

## Non-Goals

- Define the internals of the MaaS controller.
- Redesign the MaaS `AITenant.spec` as part of the organization API design.
- Adopt a generic composition engine for all platform capabilities.
- Define compute quotas, GPU fairness, or Kueue integration.
- Solve multi-cluster organization federation.
- Define an end-user billing or chargeback system.

## Shared API foundation

### Public naming

Both designs use descriptive, Organization-based kinds for the hierarchy:

```text
Organization
OrganizationProfile
OrganizationProject
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

Because the API is pre-release, use a clean API group:

```text
organization.opendatahub.io/v1alpha1
```

No conversion webhook or compatibility CRD is required.

The CRDs retain these full kinds as their canonical API names and expose the
following lowercase `kubectl` short names through API discovery:

| Kind | Resource name | Short name |
|---|---|---|
| `Organization` | `organizations` | `org` |
| `OrganizationProfile` | `organizationprofiles` | `orgprof` |
| `OrganizationProject` | `organizationprojects` | `orgproj` |

The separate-resource design also exposes `MaaSConfiguration` with resource
name `maasconfigurations` and short name `maascfg`.

For example, the complete organization view can be inspected with:

```sh
kubectl get org,orgprof,orgproj
```

### Core resources

```text
Organization
  hierarchy and canonical identity

OrganizationProfile
  organization-wide policy, defaults, and shared platform configuration

OrganizationProject
  namespace/project provisioning
```

The MaaS request differs between the two designs below. Both treat `AITenant`
in `ai-tenants` as a MaaS implementation resource.

## MaaS API options

### Option A: Separate MaaSConfiguration

This retains a capability-specific user-managed resource. A root Organization
has at most one `MaaSConfiguration`; its immutable `organizationRef` identifies
the Organization. The MaaS operator validates the reference and reconciles the
request. Its webhook enforces root-only scope and one configuration per root.

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

Shared OIDC issuer and ingress Gateway settings remain on
`OrganizationProfile`. The MaaS operator reads them and renders `AITenant`
while that implementation API exists. `MaaSConfiguration` owns the MaaS
request, status, and deletion lifecycle; the core hierarchy controller does
not read or write `AITenant`. Deleting `MaaSConfiguration` requests MaaS
cleanup. This option preserves independent MaaS schema and status, at the cost
of a second resource for Organization administrators.

```sh
kubectl apply -f maas-configuration-research.yaml
kubectl get maascfg research
```

The resource reports its own `Ready` and `CapabilityNotManaged` conditions;
it does not change the DataScienceCluster component-management setting.
The MaaS operator uses read-only Organization access to authorize the request
and resolve shared settings. It creates an `AITenant` with a
`MaaSConfiguration` owner reference, writes only its spec, and leaves status,
finalization, and runtime resources to the MaaS controller. The core hierarchy
controller has no `AITenant` dependency. If `AITenant` is retired, the MaaS
operator can reconcile `MaaSConfiguration` directly.

Deleting `MaaSConfiguration` runs the MaaS cleanup contract: stop new
subscriptions and keys, revoke active keys, remove MaaS-owned control-plane
resources, and retain the shared Gateway. The request finalizer remains until
cleanup completes. An unreferenced manual `AITenant` is not deleted.

```text
MaaSConfiguration + OrganizationProfile → MaaS operator → AITenant
MaaS controller → AITenant → MaaS runtime resources
```

Existing manually created `AITenant` resources remain outside this option's
managed set unless a separate adoption mechanism is specified.

### Option B: Profile-composed AITenant

There is no fourth user-facing tenancy CRD in this option. The MaaS request
is a field on `OrganizationProfile`.

The presence of `spec.capabilities.maas` requests MaaS for a root
Organization. Removing the block disables MaaS. Profile admission rejects the
block on a child Organization. The existing one-profile-per-Organization
relationship allows one MaaS request per root without another identity or
binding resource.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: OrganizationProfile
metadata:
  name: research
spec:
  organizationRef:
    name: research
  capabilities:
    maas:
      oidc:
        clientId: research-maas
      quotas:
        maxModels: 20
        maxSubscriptions: 100
        maxApiKeys: 50
```

The profile holds MaaS-specific request fields, but no MaaS implementation
resources or runtime policy. Shared OIDC issuer and ingress Gateway settings
remain under `spec.platform`. This is a versioned public request contract;
the composition adapter maps it to the MaaS-owned `AITenant` API.
The optional `spec.capabilities.maas.tls.certificateRef` maps to
`AITenant.spec.tls.certificateRef` for organizations that use a MaaS-specific
certificate. An optional `adoptExistingRef` selects an existing `AITenant` for
an in-place upgrade; new MaaS requests omit it.

### Shared platform configuration in both options

OIDC and ingress Gateway configuration is required by multiple capabilities,
including MaaS and Observability. The initial shape can be a small section on
the profile. In this document, Gateway means the shared Kubernetes Gateway API
`Gateway` used for organization HTTP ingress; it does not mean the AI Gateway, MCP
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

Observability's request and runtime ownership remain an independent design
decision. Neither MaaS option makes the hierarchy controller own its runtime
resources.

### Option B dependency boundary

The profile-composition reconciliation graph is:

```text
OrganizationProfile → platform MaaS composition adapter → AITenant (create or adopt)
MaaS controller → AITenant → MaaS runtime resources
```

The core hierarchy controller owns `Organization`, `OrganizationProfile`, and
`OrganizationProject`. A separate platform composition adapter watches the
profile's MaaS request and creates, adopts, or updates the managed `AITenant`.
It owns `AITenant.spec`; for these managed objects, the MaaS controller owns
`AITenant.status`, finalization, and MaaS runtime resources. The MaaS controller
does not need to read or modify the organization API. The composition adapter
is the only component coupled to both the public profile schema and the
`AITenant` schema.

The adapter is isolated from hierarchy reconciliation so optional MaaS
integration does not enlarge the hierarchy controller's watch or write scope.
If MaaS replaces `AITenant`, the adapter's output changes; the public profile
request can stay the same.

### Comparison

| Concern | Option A: separate resource | Option B: profile composition |
|---|---|---|
| User-managed MaaS request | `MaaSConfiguration` | `OrganizationProfile.spec.capabilities.maas` |
| MaaS readiness | `MaaSConfiguration.status` | Projected into `OrganizationProfile.status.capabilities.maas` |
| Organization API dependency | MaaS operator reads Organization resources | Composition adapter reads Organization resources; MaaS controller reads only `AITenant` |
| `AITenant.spec` writer | MaaS operator adapter | Platform composition adapter |
| Existing manual `AITenant` upgrade | Requires a separate adoption design | Explicit UID-pinned adoption and release described below |

No option is selected in this document. The detailed adoption and release
protocol below is part of Option B.

## Custom resource examples and usage

The Organization and Project examples apply to both options. The MaaS examples
in this section illustrate Option B; Option A's request is shown above.

### Create a root Organization

Root Organizations are created by a cluster administrator. A root has no
`spec.parent` and can contain child Organizations and Projects. The controller
creates an initial restrictive `OrganizationProfile` for every Organization,
including roots, so administrators configure an existing profile.

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

The initial profile starts with no administrators, no projects, and restrictive
network defaults until the parent bootstraps administrators.

### Configure Organization policy

The profile contains organization-wide policy and shared platform
configuration. In Option B, a root profile can also contain a MaaS request;
child profiles omit that root-only block.

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
Project. The controller creates namespace labels, RoleBindings, and
organization baseline NetworkPolicies that it owns. It does not own module or
operand NetworkPolicies; those remain with the responsible component controller under
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

### Option B: Request MaaS for an Organization

An administrator requests MaaS by adding `spec.capabilities.maas` to the root
Organization's profile. This is the only user-managed MaaS request in the
organization API. The root administrator also supplies the shared OIDC issuer
and ingress Gateway reference on the same profile.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: OrganizationProfile
metadata:
  name: research
spec:
  organizationRef:
    name: research
  admins:
    - kind: Group
      name: research-admins
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    ingressGatewayRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: tenant-gateway
      namespace: openshift-ingress
  capabilities:
    maas:
      oidc:
        clientId: research-maas
      quotas:
        maxModels: 20
        maxSubscriptions: 100
        maxApiKeys: 50
```

The composition adapter resolves shared settings from the same profile and
renders `AITenant` in the MaaS registry namespace:

```sh
kubectl apply -f organization-profile-research.yaml
kubectl get orgprof research
kubectl describe orgprof research
```

Illustrative observed profile status, projected from the generated `AITenant`:

```yaml
status:
  capabilities:
    maas:
      conditions:
        - type: Ready
          status: "True"
          reason: AITenantReady
          observedGeneration: 1
```

The profile request does not change the DataScienceCluster component
configuration. In an RHOAI deployment, the adapter requires
`aigateway.modelsAsAService: Managed` (or the equivalent management signal)
before it creates `AITenant`; it does not force that setting. If MaaS is not
managed, the profile request remains and its MaaS condition reports
`CapabilityNotManaged`. Component availability is runtime status, while
root-only scope is admission validation.

Adding the MaaS block to a child profile is rejected at admission, before the
adapter creates an `AITenant`:

```yaml
spec:
  organizationRef:
    name: nlp-team
  capabilities:
    maas:
      oidc:
        clientId: nlp-team-maas
      quotas:
        maxModels: 20
```

The webhook returns an error similar to:

```text
spec.capabilities.maas is supported only for root Organizations
```

### Option B: Managed AITenant

For a new MaaS request, the composition adapter renders the existing MaaS-owned
`AITenant` API from the root profile. The MaaS controller continues to reconcile it as
described in the [AI Gateway tenancy ADR](../../model-serving/ODH-ADR-MS-0003-ai-gateway-tenancy.md).
Users of the organization API do not author this resource.

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: AITenant
metadata:
  name: research
  namespace: ai-tenants
  labels:
    organization.opendatahub.io/name: research
    organization.opendatahub.io/managed-by: maas-composition-adapter
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

The adapter sets a controller owner reference to the cluster-scoped
`OrganizationProfile`, including its UID, and owns the managed object's spec.
Kubernetes permits a namespaced dependent to have a cluster-scoped
[owner](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#owners-and-dependents).
The `AITenant.spec.gateway.name` value comes from the shared
`ingressGatewayRef`; it names the same Kubernetes Gateway API ingress object.
The adapter checks that the shared Gateway reference can be represented by
the current name-only MaaS gateway field. MaaS owns status, its cleanup
finalizer, and the tenant runtime resources. An unreferenced existing
`AITenant` with the same name remains an ownership conflict. Explicit
adoption uses the protocol below.

### Option B: Adopt an existing AITenant

A cluster administrator can bind a root profile to a manually created
`AITenant` in the fixed `ai-tenants` namespace. The `adoptExistingRef` includes
the object's name and observed UID, so a replacement object with the same name
cannot be claimed accidentally. The reference remains on the profile after
adoption and is immutable while the MaaS block exists. The existing
`AITenant` name is retained even when it differs from the Organization name.

```yaml
apiVersion: organization.opendatahub.io/v1alpha1
kind: OrganizationProfile
metadata:
  name: research
spec:
  organizationRef:
    name: research
  platform:
    oidc:
      issuerUrl: https://sso.example.com/realms/rhoai
    ingressGatewayRef:
      group: gateway.networking.k8s.io
      kind: Gateway
      name: tenant-gateway
      namespace: openshift-ingress
  capabilities:
    maas:
      adoptExistingRef:
        name: research-legacy
        uid: 123e4567-e89b-42d3-a456-426614174000
      oidc:
        clientId: research-maas
      tls:
        certificateRef:
          name: research-tls
          namespace: ai-tenants
      quotas:
        maxModels: 20
        maxSubscriptions: 100
        maxApiKeys: 50
```

The UID above is illustrative. The administrator reads the live object's UID
and copies it into the profile:

```sh
kubectl get aitenant research-legacy -n ai-tenants -o jsonpath='{.metadata.uid}'
```

Admission allows only a cluster administrator to add or change
`adoptExistingRef`. The Organization administrator may manage the remaining
MaaS settings after adoption.

Adoption proceeds without deleting or recreating the `AITenant`:

1. The administrator stops any GitOps or manual process that writes the target
   `AITenant` and prevents its prune policy from deleting the object, then sets
   the profile's shared and MaaS settings to the live values. The adapter
   compares the rendered `AITenant.spec` with the live spec, including OIDC,
   Gateway, TLS, quotas, and defaults. Unsupported or differing fields produce
   `AdoptionMismatch` on the profile; no target fields change.
2. The adapter verifies the target UID, that it is not deleting, and that it
   has no owner reference or managed-by marker from another controller. The
   MaaS-created default tenant is outside this manual-tenant adoption path.
3. The adapter adds its finalizer and records the target name and UID in its
   protected profile status subtree before claiming the target. This binding
   survives removal of the MaaS request. It then conditionally patches only
   the target's owner reference and managed-by marker. The patch tests the
   target UID and resource version so a concurrent replacement or edit fails
   the claim. A resource-version conflict is retried from a fresh read; a UID
   or ownership conflict requires administrator action.
4. The adapter [server-side applies](https://kubernetes.io/docs/reference/using-api/server-side-apply/#transferring-ownership)
   the matching spec under its field manager. Equal-valued fields may initially
   have shared ownership. The previous writer stays stopped; on later profile
   changes the adapter may force field conflicts only for mapped fields on the
   already claimed UID. Direct edits to the claimed `AITenant.spec` are
   rejected. The MaaS controller keeps its status, finalizer, and runtime
   ownership throughout.
5. The adapter reports `Adopted` after it verifies that the same `AITenant`
   UID is owned by the profile and its mapped fields are managed by the
   adapter. The separate Ready condition follows current MaaS status. Adoption
   preserves the existing runtime resources, API keys, and `AITenant` status.

Once adopted, the `AITenant` follows the same profile-driven update and
deletion contract as a newly generated one. By default, removing the MaaS block
deletes the adopted `AITenant` and triggers MaaS cleanup, including API-key
revocation.
Unreferenced manually created `AITenant` resources remain untouched.

To return an adopted tenant to manual management, a cluster administrator
atomically removes the MaaS block and sets a one-shot
`organization.opendatahub.io/release-aitenant-uid` annotation to the claimed
UID. Admission permits setting this annotation only for a claimed adoption
and only in the same update that removes the MaaS block. For example:

```sh
kubectl patch orgprof research --type=merge \
  -p='{"metadata":{"annotations":{"organization.opendatahub.io/release-aitenant-uid":"123e4567-e89b-42d3-a456-426614174000"}},"spec":{"capabilities":{"maas":null}}}'
```

The adapter stops applying the spec, verifies the UID and profile owner
reference against its protected status binding, and conditionally removes
only its owner reference and managed-by marker. It then clears its binding,
finalizer, and one-shot annotation. The `AITenant`, its status, runtime
resources, and API keys remain. If the claim changed, it reports
`ReleaseConflict` and retains the finalizer without deleting the tenant.
The administrator verifies the owner reference is gone before restarting the
manual writer; an SSA-based writer may need to take field ownership with
force-conflicts on its first apply. A later adoption requires a new explicit
name-and-UID reference. Release must complete before deleting the profile or
root Organization when rollback needs to retain the manual tenant.

The proof-of-concept writer cutover is ordered as follows:

1. Add the profile's MaaS request field and deploy the composition adapter
   without letting it write names still managed by the proof-of-concept
   `AITenant` writer.
2. Stop the old writer of generated `AITenant` specs.
3. Replace proof-of-concept generated resources or explicitly hand them over
   using their provenance and new profile owner reference. Never adopt a
   hand-created or default `AITenant` implicitly.
4. Enable profile-based composition and verify that the adapter alone writes
   each generated `AITenant.spec`, while the MaaS controller continues to
   reconcile its status and runtime resources.

### Option B administrative workflow

```sh
# 1. Create the hierarchy.
kubectl apply -f organization-research.yaml
kubectl apply -f organization-nlp-team.yaml

# 2. Bootstrap root and child OrganizationProfile administrators.
kubectl apply -f organization-profile-research-bootstrap.yaml
kubectl apply -f organization-profile-nlp-team.yaml

# 3. Let an Organization administrator create a project.
kubectl --as=alice --as-group=nlp-team-admins \
  apply -f organization-project-sentiment.yaml

# 4. Let a root Organization administrator request MaaS on its profile.
kubectl --as=bob --as-group=research-admins \
  apply -f organization-profile-research.yaml

# 5. Inspect the complete Organization subtree.
kubectl get org,orgprof,orgproj
kubectl describe orgprof research
```

### Option B delete and cleanup

Deleting an Organization Project removes the resources owned by that project
controller. Removing `spec.capabilities.maas` from the root profile requests
MaaS deletion unless an adopted tenant is explicitly released in the same
update. There is no separate `enabled` field. The composition adapter deletes
only the `AITenant` it created or explicitly adopted and waits for the MaaS
controller's cleanup finalizer. MaaS cleanup follows this contract:

1. Stop accepting new subscriptions and API-key issuance for the organization.
2. Revoke active API keys while retaining revocation/audit records for 90 days.
3. Delete MaaS-owned tenant control-plane resources, including the MaaS API
   namespace and policies. Deployed models and model data are retained unless
   their own lifecycle explicitly deletes them.
4. Leave the shared Gateway untouched because it is not MaaS-owned.
5. Remove the finalizer only after the owned cleanup has completed.

The adapter uses the protected name-and-UID binding in profile status to find
the target after the MaaS block is removed. It deletes the target only if its
owner reference still points to this profile UID; an unclaimed manual object
is left alone. The adapter retains its profile finalizer until the claimed
`AITenant` is gone or an explicit release has removed the owner reference.
This also orders profile and root Organization deletion after MaaS cleanup,
so the MaaS controller can finish while the organization identity is still
available. Transient failures retain finalizers, report a deletion failure
on the profile, and are retried. Unreferenced hand-created `AITenant`
resources, unmanaged namespaces, and the shared Gateway remain untouched.

```sh
kubectl patch orgprof research --type=json \
  -p='[{"op":"remove","path":"/spec/capabilities/maas"}]'
kubectl delete orgproj sentiment-analysis
kubectl delete org nlp-team
```

Deletion authorization remains subject to the validating webhook and the
Organization administrator hierarchy.

## Other shapes considered

### MaaS controller watches the profile directly

The MaaS controller could read `OrganizationProfile.spec.capabilities.maas` and
create runtime resources without a generated `AITenant`. This removes the
composition hop, but ties the MaaS controller to the organization API and
requires it to own part of profile status. The adapter preserves the existing
`AITenant` reconciliation boundary.

### Use Crossplane or kro for composition

[Crossplane Compositions](https://docs.crossplane.io/latest/composition/composite-resources/)
and [kro ResourceGraphDefinitions](https://kro.run/docs/concepts/rgd/overview/)
can define a higher-level API and manage composed resources. Using either to
generate the existing `OrganizationProfile` CRD would transfer API ownership
to that engine; adding a new composite kind would add another public API.
Option B uses the composition pattern with a focused adapter; adopting a
general composition engine would be a separate platform decision.

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

This provides a common attachment mechanism but adds indirection and still
requires another resource for MaaS configuration and status. It could serve a
broader service catalog, but neither MaaS option here requires that catalog.

### Short `Org` kinds

```text
OrgProfile
OrgProject
```

Short names are concise but less self-documenting in Kubernetes output, RBAC,
and events. Full `Organization*` kinds are preferred for public APIs.

## Option B design details

### Authorization

The profile webhook resolves `spec.organizationRef` and allows an Organization
administrator to change that Organization's capability request. Existing
ancestor authority over hierarchy and administrator assignments does not grant
configuration self-management. The webhook rejects MaaS requests on child
Organizations and fails closed when the reference or administrator cannot be
resolved. Organization administrators cannot edit managed `AITenant`
resources in `ai-tenants`.

### Ownership

The hierarchy controller owns Organization resources. The composition adapter
is the sole active writer of created or adopted `AITenant.spec` and its profile
owner reference.
The MaaS controller owns `AITenant.status`, its cleanup finalizer, and the
runtime resources it creates. Neither controller overwrites the other's
fields. The adapter writes only its MaaS subtree in `OrganizationProfile.status`.

### Status

The MaaS controller reports detailed provisioning status on `AITenant`. The
adapter projects its Ready condition, failures, and current observed generation
to `OrganizationProfile.status.capabilities.maas`. It also records the managed
`AITenant` name and UID there for safe cleanup after the request is removed.
The profile reports `CapabilityNotManaged` when the MaaS component is
unavailable, `CapabilityUnavailable` when the `AITenant` API is absent, and
`OwnershipConflict` if the target name belongs to a different owner. An
adoption request can also report `AdoptionMismatch` for differing specs or
`AdoptionConflict` for a stale UID or concurrent claim. The adapter reports
Ready only after `AITenant` has observed the profile's current desired
configuration. This requires MaaS status to identify its observed
generation; if it does not, the adapter cannot claim current-generation
readiness. A failed release reports `ReleaseConflict`. After cleanup or
release, the adapter clears the MaaS status subtree. Other profile status
remains owned by the hierarchy controller.

### Lifecycle

The MaaS block's presence requests management; its removal requests deletion
unless a cluster administrator explicitly releases an adopted `AITenant`.
The adapter creates `AITenant` only when the RHOAI AI Gateway MaaS capability
is managed and never mutates that component setting. It attaches a profile
finalizer before creating or adopting `AITenant`; on disable or profile
deletion, it requests child deletion and waits for MaaS cleanup to complete
unless the adopted tenant was explicitly released first. The hierarchy
controller retains the root Organization until its profile is gone. Existing
unreferenced hand-created or default `AITenant` resources remain untouched.
A transition of the component to Unmanaged after provisioning must run the
same child cleanup path before the MaaS controller stops; a failed cleanup
keeps the profile in a deletion or unavailable condition rather than
orphaning the tenant runtime.

## Rollout considerations

Publish `Organization`, `OrganizationProfile`, and `OrganizationProject` with
the new API group and Organization-based references and labels. The API has not
shipped, so proof-of-concept manifests can be replaced directly without a
conversion or compatibility API.

If Option A is selected, also publish `MaaSConfiguration`. The MaaS operator
owns that resource and the adapter to `AITenant`; the tenancy-side `AITenant`
writer stops before the MaaS operator takes over generated resources. Existing
manually created tenants remain unmanaged unless an adoption path is added.

If Option B is selected, the following profile-composition cutover applies.
The existing tenancy-side `AITenant` writer is replaced by the isolated
profile composition adapter. The MaaS controller continues to reconcile
`AITenant` and does not need an Organization API dependency. A proof-of-concept
generated `AITenant` is recreated or explicitly handed over after the old
writer stops. An existing manually created `AITenant` is claimed in place only
through the UID-pinned, cluster-administrator-authorized adoption path.

For each manual tenant, the upgrade records the live UID and effective spec,
stops the old writer without pruning, configures a matching root profile, and
waits for `Adopted` before allowing profile-driven changes. Verification checks
that the `AITenant` UID and MaaS runtime resources did not change. A rollback
uses the explicit release action before the manual writer resumes. Manual
tenants with no adoption request continue under their existing management.

## Security considerations

For Option A, the MaaS webhook validates the Organization reference and
administrator authorization, and the MaaS operator reads Organization policy
without writing hierarchy resources. Option B has these additional controls:

- Profile admission fails closed when the organization reference is missing or
  the editor is unauthorized.
- Only a cluster administrator may set or change the UID-pinned
  `adoptExistingRef` or request release; normal Organization admins may edit
  other MaaS fields.
- The composition adapter may write created or adopted `AITenant` specs and its
  MaaS profile status subtree, but cannot change organization hierarchy or
  admins.
- The MaaS controller needs no write access to Organization resources.
- Managed `AITenant` specs are protected from direct user edits; existing
  hand-created resources are claimed only by an authorized name-and-UID
  reference after spec comparison.
- Resource names must not be used as an authorization shortcut; authorization
  must resolve the referenced Organization and its administrators.

## Scalability and performance

Under Option A, the MaaS operator watches its configuration resources and
reads the Organization resources needed to render `AITenant`.

Under Option B, the following adapter boundary applies.
The composition adapter watches profile changes, the MaaS component management
signal, and its created or adopted `AITenant` resources. The MaaS controller
watches `AITenant`, not Organization resources. The core hierarchy controller
does not watch MaaS runtime CRDs, so optional capabilities do not enlarge its
reconciliation surface.

## Observability

Option A reports MaaS readiness on `MaaSConfiguration`; Option B projects it
onto `OrganizationProfile`. In both options, `AITenant` keeps detailed MaaS
provisioning conditions and common Organization labels are propagated to
managed resources and namespaces for metrics attribution.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Option A adds a second user-managed request | Give `MaaSConfiguration` its own schema, status, and clear administration workflow. |
| Option A couples the MaaS operator to Organization APIs | Limit its access to the identity and shared settings it needs; keep hierarchy writes separate. |
| Option B grows the profile | Expose only high-level capability intent; keep MaaS runtime fields on `AITenant`. |
| Option B couples the adapter to `AITenant` | Confine the mapping to the composition adapter so a future MaaS API replacement does not change the profile request. |
| Cleanup blocked by MaaS failure | Keep the request owner's finalizer and report a deletion condition until cleanup completes. |
| Option B name collision with a manual `AITenant` | Report `OwnershipConflict` unless a cluster administrator explicitly pins that object's name and UID for adoption. |
| Option B legacy writer or spec drift during adoption | Stop the old writer, compare the full effective spec, and claim metadata with UID and resource-version checks before managing fields. |
| Option B cannot express an existing spec | Report `AdoptionMismatch` and extend the adapter contract before claiming that tenant. |
| Option B rollback deletes an adopted tenant | Require an explicit UID-pinned release action that removes adapter ownership before the manual writer resumes. |
| Public naming drift | Publish Organization-based hierarchy kinds in the first release. |
| Duplicate organization identities | Make Organization the canonical identity and treat `AITenant` as an implementation resource. |

## Decision and validation criteria

Before choosing an option:

- Platform, MaaS, and Observability teams agree on public naming and shared
  OIDC and Gateway ownership.
- Compare whether an independent MaaS schema and status justify the extra
  `MaaSConfiguration` resource, or whether a single profile workflow is more
  valuable.
- Confirm the core hierarchy controller does not write `AITenant` under either
  design and that MaaS can replace `AITenant` without changing the selected
  user-facing request API.

If Option A is selected, validate root-only `MaaSConfiguration` admission,
delegated authorization, status, finalization, and the handoff from the
tenancy-side `AITenant` writer. Unreferenced manual tenants remain untouched.

If Option B is selected, validate root-only profile admission, delegated
authorization, readiness projection, and profile-driven deletion. Adoption
must preserve the `AITenant` UID, runtime resources, API keys, and status;
stale UIDs, spec mismatches, conflicting ownership, and an active old writer
must leave the target unchanged. Release must preserve the same tenant and
runtime state. Controller tests must cover cleanup retries, component disable,
one active `AITenant.spec` writer, and MaaS status freshness.

## Open questions

- Which MaaS request API should be selected: a separate
  `MaaSConfiguration` or profile composition?
- Should the proposed API group be `organization.opendatahub.io`, or should a
  broader platform API group be selected?
- Should shared configuration begin on `OrganizationProfile` or use
  `OrganizationPlatformConfig` immediately?
- Is Gateway configuration shared across all capabilities?
- Is the OIDC issuer shared while client IDs remain capability-specific?
- Should additional capabilities use the same profile composition pattern, or
  should they expose separate user-facing APIs?

## Implementation history

- The initial design used a separate user-managed MaaS request resource.
- The implementation introduced a transitional tenancy-side `AITenant` adapter.
- Profile composition is now documented as an alternative to that separate
  request. It leaves the MaaS controller on its existing `AITenant` API.
- The profile-composition option includes explicit, in-place adoption and
  release of manually created `AITenant` resources.
- The public hierarchy kinds use Organization-based names.
