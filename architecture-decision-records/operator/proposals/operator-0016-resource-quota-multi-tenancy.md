# KEP: RHOAI Resource Quota for Multi-Tenancy

<!--
This is a design-exploration document in CNCF/Kubernetes KEP style. It is not
yet assigned a Kubernetes Enhancement Proposal number and is not part of the
formal ADR set.
-->

- Authors: Lindani Phiri (`@lphiri`), Chris Sams (`@csams`)
- Status: Provisional design discussion
- Discussion issues:
  [RHAIRFE-2922](https://redhat.atlassian.net/browse/RHAIRFE-2922),
  [RHAISTRAT-2554](https://redhat.atlassian.net/browse/RHAISTRAT-2554)
- Last updated: 2026-09-21

## Summary

RHOAI needs resource quota and fair sharing on top of its multi-tenancy
framework. The tenancy model establishes identity, hierarchy, delegation,
isolation, and observability, but deliberately leaves resource allocation to a
separate design.

This proposal adds two complementary allocation mechanisms:

1. Use Kueue to enforce accelerator and batch-compute quota, borrowing,
   lending, fair sharing, and preemption.
2. Generate Kubernetes `ResourceQuota` and `LimitRange` resources for
   directly scheduled workloads in tenant namespaces.

The tenant hierarchy maps to Kueue Cohorts, ClusterQueues, and LocalQueues.
The tenancy control plane retains responsibility for administrative capacity,
authorization, and safe delegation. Kueue remains responsible for runtime
admission and sharing.

## Motivation

The baseline multi-tenancy design does not prevent noisy-neighbor contention.
This is an accepted scope boundary for the identity and isolation framework,
but it leaves tenants without enforceable resource protection.

The immediate driver is GPU-as-a-service. GPUs are scarce and expensive, and a
shared platform must let an organization divide a fixed accelerator budget
among teams, reclaim idle allocation, and restore guaranteed allocation when
contention returns. Kubernetes `ResourceQuota` cannot express hierarchical
budgets, cross-namespace borrowing, lending, fair sharing, gang scheduling, or
priority-based preemption.

Kueue already provides Cohorts, ClusterQueues, LocalQueues, quota borrowing,
lending, fair sharing, and preemption. RHOAI also depends on Kueue for
distributed workloads. Reusing it avoids a separate scheduling and quota
engine.

Kueue does not admit every RHOAI workload. Workbenches, model servers, and
ad-hoc pods may schedule directly, so each tenant project also needs standard
namespace guardrails for CPU, memory, storage, GPUs, and object counts.

## Reference design

The multi-tenancy hierarchy, authorization model, network isolation, and
observability tiers are maintained in the draft Operator-0015 work. This KEP
builds on that design and does not redefine it:

[Operator-0015 draft PR #156](https://github.com/opendatahub-io/architecture-decision-records/pull/156/changes)

The examples use the current proof-of-concept names `PlatformTenant`,
`TenantProfile`, and `TenantProject`. If the Organization-based names proposed
by the Operator-0015 CRD design are adopted, the mapping is mechanical:

```text
PlatformTenant  -> Organization
TenantProfile   -> OrganizationProfile
TenantProject   -> OrganizationProject
tenantRef       -> organizationRef
```

The quota model and Kueue mapping do not depend on which public naming option
is selected.

## Goals

- Provide hierarchical accelerator and batch-compute quota over the tenant
  tree.
- Enforce that the sum of child allocations does not exceed the capacity
  delegated by a parent, for each resource type.
- Delegate runtime quota, fair sharing, borrowing, lending, and preemption to
  Kueue.
- Provide named sharing presets with an advanced raw Kueue override.
- Support explicit borrowing between tenants that do not share an
  administrative parent.
- Preserve each tenant's guaranteed allocation when idle capacity is lent.
- Generate per-project `ResourceQuota` and `LimitRange` resources for workloads
  that bypass Kueue admission.
- Restrict DRA DeviceClass access to a parent-constrained subset.
- Prevent tenants from increasing their own capacity or hardware scope.
- Add only optional fields to the baseline tenancy APIs.
- Attribute quota, admission, and accelerator-utilization metrics to the
  appropriate tenant subtree.

## Non-Goals

- Redefine tenant identity, hierarchy, delegation, network isolation,
  observability tiers, or MaaS provisioning.
- Federate quota across clusters. Multi-cluster fair sharing is deferred to a
  future MultiKueue design.
- Ship or repackage Kueue. Red Hat Build of Kueue (RHBoK) is an external
  prerequisite.
- Build billing, chargeback, or FinOps systems.
- Introduce a second public quota API that duplicates Kueue types.
- Optimize scheduling for cloud cost or spot capacity.
- Provide physical hardware isolation. Dedicated node pools remain a separate
  deployment choice.

## Proposal

### Add two allocation layers

The mechanisms are additive and cover different workload paths:

```text
Batch-admitted workload
  -> LocalQueue -> ClusterQueue -> Cohort -> Kueue runtime admission

Directly scheduled workload
  -> Namespace -> ResourceQuota and LimitRange -> Kubernetes admission
```

Kueue controls what admitted workloads may run against shared accelerator and
compute quota. `ResourceQuota` controls what all workloads may request within a
namespace. A project may use either mechanism or both.

### Extend the tenancy APIs

The proposal adds optional fields to the baseline CRDs:

- `PlatformTenant.spec.capacity` defines the budget a parent may allocate.
- `PlatformTenant.spec.quota` defines a leaf or standalone tenant's guaranteed
  allocation.
- `PlatformTenant.spec.hardware` defines allowed resource flavors and DRA
  DeviceClasses.
- `PlatformTenant.spec.sharing` selects explicit cross-tree sharing pools.
- `TenantProfile.spec.isolation` selects a sharing preset.
- `TenantProfile.spec.fairShareWeight` configures relative fair-share weight.
- `TenantProfile.spec.overrides` exposes advanced Kueue configuration.
- `TenantProject.spec.resourceLimits` defines namespace `ResourceQuota` and
  `LimitRange` policy.

When these fields are absent, the tenancy controller behaves as defined by the
baseline framework and does not create quota resources.

RHBoK must be present before the Kueue-backed capability is enabled. If it is
missing or incompatible, the controller leaves the requested Kueue resources
unapplied and reports a warning condition on the affected tenancy resources.
Namespace `ResourceQuota` and `LimitRange` reconciliation does not require
Kueue.

### Map the hierarchy to Kueue

The control plane renders the following Kueue resources:

```text
Parent PlatformTenant          -> Cohort
Leaf or standalone tenant      -> ClusterQueue
TenantProject                  -> LocalQueue in the project namespace
```

A parent Cohort is nested under its own parent's Cohort. A leaf ClusterQueue
joins its parent's Cohort. A standalone tenant's ClusterQueue does not join an
administrative Cohort unless the tenant explicitly joins a sharing pool.

The leaf's guaranteed quota becomes the ClusterQueue's `nominalQuota`. Each
project LocalQueue targets the ClusterQueue for its tenant.

### Enforce administrative capacity in the control plane

Capacity is the maximum allocation that a parent allows its children to
divide. It is an administrative concept and is distinct from current runtime
usage.

The validating webhook enforces this invariant for every resource type:

```text
sum(child guaranteed allocations) <= parent capacity
```

Capacity is not represented as spendable Kueue quota. Generated Cohorts use
zero nominal quota, and usable quota remains on leaf ClusterQueues. This avoids
having to distribute a single parent budget across Kueue's flavor-and-resource
tuples while preserving a clear API-level capacity ceiling.

Runtime borrowing may temporarily let a tenant consume more than its
guaranteed allocation. It does not change the administrative allocation and
therefore does not weaken the capacity invariant.

### Configure sharing with isolation presets

`TenantProfile.spec.isolation` provides named presets for the generated
ClusterQueue or Cohort:

- `hard`: neither borrow nor lend quota.
- `guaranteed`: lend idle quota but do not borrow.
- `balanced`: borrow and lend within the Cohort using Kueue defaults.
- `burst`: retain guaranteed quota and borrow up to a configured limit.

The presets configure borrowing limits, lending limits, fair sharing, and
preemption policy. `TenantProfile.spec.overrides` is an escape hatch for
advanced Kueue settings that the presets do not expose.

Preset values must be reviewed with the Distributed Workloads team before the
API is considered stable.

### Share idle capacity across tenant trees

Kueue borrowing occurs inside a Cohort tree. The hierarchy therefore supports
three cases:

- Sibling tenants share through their parent's Cohort.
- Cousin tenants share through their lowest common ancestor's nested Cohort,
  subject to limits at each level.
- Unrelated roots and standalone tenants do not share by default.

Unrelated tenants may opt into an explicit shared pool. A pool maps to a Kueue
Cohort that is independent of the administrative parent-child tree. This keeps
administration and resource sharing as separate concerns, similar to the
baseline framework's explicit network grants.

Each member declares a lending limit and borrowing limit. Membership requires
explicit authorization so one tenant cannot unilaterally consume another
tenant's idle quota.

Two invariants apply to every sharing topology:

1. Only idle quota is lent. Kueue may preempt borrowed workloads when the owner
   needs its guaranteed allocation.
2. Sharing does not change a tenant's administrative allocation or its parent's
   capacity ceiling.

The shared-pool ownership API and consent workflow remain open design items.

### Add namespace guardrails

For each `TenantProject` with `spec.resourceLimits`, the tenancy controller
creates in the project namespace:

- a `ResourceQuota` for CPU, memory, ephemeral or persistent storage, GPUs,
  and object counts such as pods, services, and PVCs; and
- a `LimitRange` with default requests and limits so an unbounded pod cannot
  consume the namespace budget.

The controller owns and reconciles both resources. Managed-resource protection
rejects direct changes that conflict with the tenancy API.

A project that also uses a Kueue LocalQueue is subject to both admission paths.
The implementation and documentation must distinguish batch quota from
namespace totals so operators do not unintentionally configure contradictory
limits.

### Gate accelerator hardware

When Kubernetes Dynamic Resource Allocation is available, a validating webhook
on `ResourceClaim` and `ResourceClaimTemplate` restricts DeviceClass references
in tenant namespaces.

`PlatformTenant.spec.hardware.deviceClasses` must be a subset of the parent's
allowed DeviceClasses. Without DRA, DeviceClass validation is inactive and the
webhook allows those claims to proceed unchanged.

### Protect delegation at four layers

1. RBAC allows tenant administrators to manage `TenantProfile` and
   `TenantProject`, and to request child tenants, but not to manage generated
   Kueue resources.
2. The validating webhook checks parent capacity, hardware subsets, immutable
   references, and baseline tenancy authorization on every create or update.
3. Managed-resource protection prevents direct edits to controller-owned
   Cohorts, ClusterQueues, LocalQueues, `ResourceQuota`s, and `LimitRange`s.
4. Kueue enforces runtime admission against the validated generated
   configuration.

Capacity, guaranteed allocation, and hardware scope remain on the
parent-controlled `PlatformTenant`. A tenant administrator cannot increase
their own allocation.

### Integrate tenant-safe observability

Kueue controller metrics and DCGM GPU exporter metrics originate from shared
platform endpoints that contain data for multiple tenants. A tenant-specific
Prometheus cannot scrape those endpoints directly without exposing other
tenants' usage.

For observability tier 3, the controller deploys an OpenTelemetry Collector as
a filtering relay. It:

1. scrapes the shared Kueue and DCGM endpoints;
2. filters series to the tenant subtree's Cohorts, ClusterQueues, and
   namespaces; and
3. remote-writes the filtered series to the tenant's Prometheus.

Tier 2 dashboards add quota utilization, borrowing and lending, admission
latency, and GPU utilization panels. Initial metric sources include:

```text
kueue_cluster_queue_resource_usage
kueue_cluster_queue_nominal_quota
kueue_admission_wait_time_seconds
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_FB_USED
```

## Custom resource examples and usage

The examples use the current proof-of-concept tenancy names. They are updated
mechanically if the Organization-based naming proposal is accepted.

### Define capacity on a parent tenant

A cluster administrator creates a root tenant with an accelerator budget and
hardware catalog:

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: PlatformTenant
metadata:
  name: research-division
spec:
  displayName: Research Division
  capacity:
    resources:
      - type: nvidia.com/gpu
        amount: "64"
  hardware:
    resourceFlavors:
      - name: a100-80gb
    deviceClasses:
      - nvidia-a100
```

```sh
kubectl apply -f platform-tenant-research-division.yaml
kubectl get platformtenant research-division
kubectl get platformtenant research-division -o yaml
```

### Allocate guaranteed quota to a leaf tenant

An authorized ancestor administrator creates a leaf tenant. The webhook rejects
the request if the guaranteed allocation exceeds the parent's remaining
capacity or the DeviceClass is not allowed by the parent.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: PlatformTenant
metadata:
  name: nlp-team
spec:
  displayName: NLP Team
  parent: research-division
  quota:
    resources:
      - name: gpu
        type: nvidia.com/gpu
        guaranteed: "16"
        flavor: a100-80gb
  hardware:
    deviceClasses:
      - nvidia-a100
  sharing:
    pools:
      - name: gpu-burst-pool
        lendingLimit: "8"
        borrowingLimit: "8"
```

```sh
kubectl apply -f platform-tenant-nlp-team.yaml
kubectl get platformtenant nlp-team
kubectl get clusterqueue
```

The controller-generated ClusterQueue and Cohort names are reported in status.
Illustrative status is:

```yaml
status:
  phase: Ready
  quota:
    clusterQueue: nlp-team
    cohorts:
      - research-division
  conditions:
    - type: QuotaReady
      status: "True"
      reason: KueueResourcesReconciled
```

The exact status schema remains part of the API review.

### Configure a sharing preset

The tenant profile selects the default sharing and fair-share policy for the
generated Kueue resources:

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: TenantProfile
metadata:
  name: nlp-team
spec:
  tenant: nlp-team
  isolation: balanced
  fairShareWeight: 1
```

```sh
kubectl apply -f tenant-profile-nlp-team.yaml
kubectl get tenantprofile nlp-team
kubectl describe clusterqueue nlp-team
```

### Provision a project with namespace limits

The project creates a namespace and LocalQueue. `resourceLimits` also creates a
`ResourceQuota` and `LimitRange` for workloads that schedule without Kueue.

```yaml
apiVersion: tenancy.opendatahub.io/v1alpha1
kind: TenantProject
metadata:
  name: sentiment-analysis
spec:
  tenant: nlp-team
  resourceLimits:
    requests.cpu: "32"
    requests.memory: 128Gi
    requests.nvidia.com/gpu: "8"
    count/pods: "50"
```

```sh
kubectl apply -f tenant-project-sentiment-analysis.yaml
kubectl get tenantproject sentiment-analysis
kubectl get localqueue,resourcequota,limitrange -n sentiment-analysis
```

### Typical administrative workflow

```sh
# 1. Define the root capacity and hardware catalog.
kubectl apply -f platform-tenant-research-division.yaml

# 2. Delegate guaranteed quota to a child tenant.
kubectl apply -f platform-tenant-nlp-team.yaml

# 3. Select the tenant's runtime sharing policy.
kubectl apply -f tenant-profile-nlp-team.yaml

# 4. Let a tenant administrator create a constrained project.
kubectl --as=alice --as-group=nlp-team-admins \
  apply -f tenant-project-sentiment-analysis.yaml

# 5. Inspect the generated admission and namespace resources.
kubectl get cohort,clusterqueue
kubectl get localqueue,resourcequota,limitrange -A
```

### Delete and cleanup

Deleting a `TenantProject` removes the controller-owned LocalQueue,
`ResourceQuota`, and `LimitRange` along with other project-owned resources.
Deleting a tenant remains subject to the baseline hierarchy checks and must not
orphan child tenants, active projects, or unmanaged Kueue resources.

The initial design does not automatically adopt or delete hand-created
ClusterQueues. Adoption requires a separately reviewed migration procedure.

## Alternatives considered

### Use only Kubernetes ResourceQuota

Per-namespace hard limits are simple and broadly available, but they cannot
express an organizational hierarchy, cascading capacity, idle-quota borrowing,
lending, gang scheduling, or preemption. Static GPU partitioning also strands
expensive accelerators.

`ResourceQuota` remains part of this proposal for namespace guardrails but is
not the accelerator-allocation engine.

### Build a quota and fair-sharing engine

The tenancy controller could maintain status rollups and implement allocation,
borrowing, and consumption enforcement itself. This duplicates Kueue's runtime
admission and scheduling semantics and creates additional correctness risk.

The proposal instead keeps policy and delegation in the tenancy control plane
and delegates runtime enforcement to Kueue.

### Use static tenant node pools

Dedicated node pools provide strong physical isolation but waste accelerators
when a tenant is idle and do not support soft sharing. They remain suitable for
customers that explicitly require hardware isolation, but are not the default
multi-tenant allocation model.

### Fold quota into the baseline tenancy ADR

An earlier draft included quota in the core multi-tenancy design. Keeping it
there would couple tenant identity to a particular quota engine and make Kueue
a prerequisite for adopting the hierarchy API.

This proposal remains separate so the foundational tenancy model can progress
independently.

### Adopt MultiKueue immediately

MultiKueue introduces cross-cluster admission and a larger operational surface.
Single-cluster quota should be validated before cross-cluster fair sharing is
designed.

## Design details

### Authorization

The baseline ancestor-administrator rules remain authoritative:

- cluster administrators create root tenants;
- an authorized parent or ancestor administrator creates a child tenant and
  assigns its capacity, quota, hardware scope, and pool membership;
- a tenant administrator manages profiles and projects but cannot modify their
  own `PlatformTenant`; and
- generated Kueue and namespace-policy resources are not user-managed APIs.

The webhook resolves the parent and uses the baseline SubjectAccessReview flow
before accepting hierarchy or allocation changes.

### Ownership

The tenancy controller owns rendered Cohorts, ClusterQueues, LocalQueues,
`ResourceQuota`s, and `LimitRange`s. Owner references and common tenant labels
connect implementation resources to their API source where Kubernetes scoping
rules permit.

Kueue owns runtime admission decisions. The tenancy controller must not infer
consumption from its own desired state or replace Kueue status.

### Validation and status

Admission validation covers:

- child allocation against parent capacity;
- child resource flavors and DeviceClasses against the parent's catalog;
- immutable parent and tenant references;
- shared-pool membership authorization; and
- the baseline hierarchy and administrator rules.

Reconciliation status should identify the generated Kueue resources and report
conditions for missing CRDs, unsupported RHBoK versions, invalid overrides, and
resource reconciliation failures. Exact condition names are finalized with the
CRD schema.

### Lifecycle

Quota fields are opt-in. Adding them creates or updates generated resources.
Removing them must first validate that doing so cannot silently strand admitted
workloads or unmanaged dependents.

Changes to guaranteed quota may trigger Kueue preemption as the new desired
state takes effect. Documentation and status must make this operational impact
clear before an administrator confirms a reduction.

### Upgrade and migration

The fields are additive and optional, so existing tenancy resources continue to
work without quota configuration.

The current API is a proof of concept. If Operator-0015 adopts Organization
terminology, manifests are renamed before the first shipped API and no
conversion webhook is required. The quota schema is otherwise unchanged.

Existing hand-created ClusterQueues are not adopted automatically. A future
migration design must verify compatible flavors, quotas, Cohorts, admission
checks, and active workloads before controller ownership changes.

## Security considerations

- Tenants cannot increase their own capacity, guaranteed allocation, or
  hardware scope because those fields are controlled by an ancestor.
- Admission checks preserve the parent capacity invariant before generated
  resources change.
- DeviceClass access is constrained by the parent's hardware catalog.
- Shared-pool membership grants resource sharing only. It grants no namespace,
  network, identity, or data access.
- Lending exposes only idle quota. Kueue reclaims it for the owner through
  preemption.
- Tenant Prometheus instances receive filtered Kueue and DCGM series rather
  than access to central all-tenant endpoints.
- Generated resources are protected from direct edits that would bypass the
  tenancy API.

## Scalability and performance

Capacity validation requires reading sibling allocations on create and update.
The webhook should use informer-cache and label-indexed lookups rather than live
list calls. Its p99 latency must be tested against the baseline tenancy scale
targets.

Capability controllers should watch only the tenancy resources and generated
resource types they reconcile. Quota metrics should use stable tenant labels so
dashboards do not require expensive hierarchy reconstruction at query time.

## Observability

Tenancy resources should expose conditions for validation, Kueue availability,
generated-resource readiness, and degraded reconciliation. Common labels should
identify the root, tenant, project, ClusterQueue, and Cohort on metrics and
managed resources.

Platform dashboards should show:

- guaranteed, borrowed, lent, and current resource usage;
- pending workload count and admission wait time;
- preemption events by reason;
- GPU allocation and utilization; and
- webhook validation failures and reconciliation errors.

Tenant-visible metrics must remain limited to the tenant's subtree.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| RHBoK is installed out of band and may be absent or incompatible. | Define a supported version floor, check required CRDs and versions, and report a warning condition before rendering Kueue resources. |
| Adopting an existing ClusterQueue may evict workloads after a hard reconcile. | Do not adopt automatically; require a documented procedure that begins with a matching spec and staged changes. |
| Kueue and `ResourceQuota` may impose contradictory limits. | Document workload routing, validate incompatible settings where possible, and test namespaces that use both mechanisms. |
| Capacity validation may become expensive on wide tenant trees. | Use informer-cache indexes and measure webhook p99 latency at the target scale. |
| Incorrect sharing presets may strand or starve accelerators. | Start with conservative values, review them with the Distributed Workloads team, and restrict raw overrides to advanced administrators. |
| A shared-pool policy may allow unintended cross-organization borrowing. | Require explicit membership authorization, bounded lending and borrowing, and auditable status. |
| Reducing quota may preempt running borrowed workloads. | Surface the impact in status and documentation, and provide a staged operational procedure. |

## Stakeholder impact

| Group | Impact |
|---|---|
| Platform and `rhods-operator` | Additive CRD fields, Kueue rendering, capacity validation, and managed-resource protection. |
| Distributed Workloads and RHBoK | Required runtime dependency; review Cohort hierarchy, presets, and supported versions. |
| Hardware and Accelerators | DeviceClass delegation and DCGM metrics. |
| Observability | Filtering relay plus quota, admission, and GPU dashboards. |
| Multi-Tenancy | Foundational API, hierarchy, authorization, and observability dependency. |
| Security and Compliance | Review self-escalation prevention, pool consent, and metric isolation. |
| Dashboard | Capacity editing plus allocation and utilization views. |
| Model Serving and component teams | Workloads must account for Kueue admission and namespace limits. |
| Documentation and UX | Explain presets, preemption, dual admission, and administrative workflows. |

## Graduation criteria

Before this design is considered stable:

- The Operator-0015 public naming and core tenancy API are approved.
- The RHBoK version floor and out-of-band installation contract are defined.
- Representative GPUaaS scenarios validate the zero-nominal-quota Cohort model,
  including multiple accelerator flavors.
- Isolation preset values and preemption semantics are reviewed by the
  Distributed Workloads team.
- Shared-pool ownership, authorization, consent, and removal behavior are
  specified.
- Webhook capacity validation meets the baseline scale and latency targets.
- Projects using both Kueue and `ResourceQuota` have conformance tests and
  operator guidance.
- DeviceClass subset validation and quota self-escalation tests pass.
- Tenant filtering prevents cross-tenant Kueue and DCGM metric exposure.
- Existing unmanaged ClusterQueues remain unaffected unless an explicit
  adoption procedure is invoked.

## Open questions

- Should generated Cohorts carry zero nominal quota in every supported GPUaaS
  topology, especially when multiple resource flavors are present?
- What is the minimum supported RHBoK/Kueue version, and how is its presence
  guaranteed when it is provisioned outside OLM dependency resolution?
- Should generated `ResourceQuota`s use scope selectors or apply flat namespace
  totals?
- What validation and operator guidance prevents unexpected interaction between
  Kueue admission and `ResourceQuota`?
- What exact borrowing, lending, fair-share, and preemption values define each
  isolation preset?
- Is a shared pool its own CRD or a field on an existing tenancy resource?
- Who authorizes pool membership, and how is a member removed without
  ungracefully evicting borrowed workloads?
- What is the safe procedure for adopting an existing ClusterQueue?
- Which conditions and status fields expose generated resources and quota
  readiness without duplicating Kueue status?

## References

- [Operator-0015 draft PR #156](https://github.com/opendatahub-io/architecture-decision-records/pull/156/changes)
  for the baseline multi-tenancy framework and strategy
- `operator/multi-tenancy-strategy.md`
- Tenancy Control Plane design in `multitenant-gpuaas`: `PLAN.md`,
  `API-REFERENCE.md`, and `KUEUE-API-REFERENCE.md`
- [RHAIRFE-2922](https://redhat.atlassian.net/browse/RHAIRFE-2922) - Tenant
  Management Control Plane for Multi-Team RHOAI Deployments
- [RHAISTRAT-2554](https://redhat.atlassian.net/browse/RHAISTRAT-2554) -
  organization hierarchy and delegated administration
- [ODH-ADR-Operator-0007](../ODH-ADR-Operator-0007-auth-crd.md) - Auth CRD
- [ODH-ADR-Operator-0012](../ODH-ADR-Operator-0012-gateway-api-authentication-architecture.md)
  - Gateway API authentication architecture
- Red Hat Build of Kueue and upstream Kueue Cohort v1beta2
- Run:ai Department and Project quota as a competitive reference

## Implementation history

- The initial multi-tenancy draft included quota and status-based consumption
  rollups in the core framework.
- Quota was split from Operator-0015 so identity and hierarchy do not depend on
  Kueue.
- The design selected Kueue for runtime accelerator and batch-compute admission.
- Kubernetes `ResourceQuota` and `LimitRange` were retained for directly
  scheduled namespace workloads.
- Shared pools were introduced to separate resource-sharing relationships from
  the administrative tenant tree.
- The API remains provisional pending the Operator-0015 naming decision and the
  graduation criteria above.
