# Extend HardwareProfile with a Dynamic Resource Allocation Reference

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-27 |
| Scope          | Operator |
| Status         | Draft |
| Authors        | [Carl Kyrillos](@carlkyrillos) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHOAIENG-87978](https://issues.redhat.com/browse/RHOAIENG-87978) — [Spike] Design HWP DRA extension |
| Other docs     | N/A |

## What

Add one new optional field, `HardwareProfileSpec.DRA` (wrapped in a `DRASpec` struct), to the
`v1` `HardwareProfile` CRD owned by opendatahub-operator. The field lets a `HardwareProfile`
reference a `ResourceClaimTemplate` (Kubernetes Dynamic Resource Allocation, DRA) that already
exists in the workload's namespace, via `dra.resourceClaimTemplateName`. This ADR covers **API
changes to the `HardwareProfile` CRD only**; the webhook/controller work that consumes the new
field to mutate workload pod specs and validate the reference is out of scope here and tracked
separately in the owning modules.

## Why

`HardwareProfile` today expresses two ways to shape a workload's compute footprint —
`Identifiers` (resource requests/limits) and `SchedulingSpec` (Kueue queue or node placement) —
but has no way to express Dynamic Resource Allocation device requests (e.g., structured GPU
sharing, `AdminAccess`, or other `resource.k8s.io` device-class-driven allocation). Workload
authors who need DRA today must bypass `HardwareProfile` entirely and hand-write
`PodSpec.resourceClaims` / `ResourceClaimTemplate` objects, losing the profile-based UX (sharing,
scheduling, and hardware-selection) that `HardwareProfile` already provides for CPU/memory/
accelerator requests.

This needs a decision now because RHOAI 3.6 scoping calls for a minimal DRA story on
`HardwareProfile`, while a fuller, GA-quality compute-allocation model
(`ComputeProfile`, [RHOAIENG-87982](https://issues.redhat.com/browse/RHOAIENG-87982)) is planned
for RHOAI 3.7. The schema decision made here needs to not foreclose or complicate that migration.

## Goals

* Let a `HardwareProfile` reference an existing `ResourceClaimTemplate` so workloads that use the
  profile can be admitted with a DRA-backed device claim.
* Ship as an additive, optional `v1` field with no CRD API version bump, and no behavior change for
  any existing `HardwareProfile` object that doesn't set it.
* Keep the schema shape open enough that inline (generated) DRA device requests could be added
  later as a non-breaking extension, without redesigning the field.
* Simplify, not complicate, the future `HardwareProfile` → `ComputeProfile` migration planned for
  RHOAI 3.7.
* Avoid opendatahub-operator or any consuming module taking on `ResourceClaimTemplate` lifecycle
  management (generation, ownership, cleanup) as part of this change.

## Non-Goals

* Implementing the webhook/controller logic that reads the new field and mutates
  `Notebook`/`InferenceService`/`LLMInferenceService` pod specs at admission time. That work is
  tracked separately in workbenches-operator and odh-model-controller (analogous to
  [RHOAIENG-62580](https://issues.redhat.com/browse/RHOAIENG-62580) and
  [RHOAIENG-62536](https://issues.redhat.com/browse/RHOAIENG-62536)).
* Designing or implementing the validating-webhook guardrail that checks a referenced
  `ResourceClaimTemplate` exists at admission time. That guardrail is proposed as part of the
  rationale for this decision (see "How" and "Risks") but its implementation belongs to the
  serving and notebooks modules, not to this ADR.
* Designing inline (generated) DRA device requests. This is documented as a considered alternative
  and deliberately kept open for a future, non-breaking addition, but is not part of what ships now.
* Designing `ComputeProfile` (RHOAI 3.7, [RHOAIENG-87982](https://issues.redhat.com/browse/RHOAIENG-87982)),
  the planned GA-quality successor to `HardwareProfile` for compute/device allocation.
* Any change to `HardwareIdentifier` or `SchedulingSpec`.

## How

### Current state

`api/infrastructure/v1/hardwareprofile_types.go` today:

```go
type HardwareProfileSpec struct {
    Identifiers    []HardwareIdentifier `json:"identifiers,omitempty"`
    SchedulingSpec *SchedulingSpec      `json:"scheduling,omitempty"`
}
```

`HardwareProfile` is namespace-scoped, but workloads can reference one in a different namespace
via the `opendatahub.io/hardware-profile-namespace` annotation (defaults to the workload's own
namespace if unset), so a single profile is routinely referenced from many namespaces at once.
This matters because a DRA reference must resolve in every namespace where the profile is used.

The logic that reads a `HardwareProfile` and mutates a workload at admission time lives outside
opendatahub-operator: workbenches-operator owns injection for `Notebook`
([RHOAIENG-62580](https://issues.redhat.com/browse/RHOAIENG-62580)), and odh-model-controller owns
injection for `InferenceService`/`LLMInferenceService`
([RHOAIENG-62536](https://issues.redhat.com/browse/RHOAIENG-62536)). opendatahub-operator's own
copy of this logic (`internal/webhook/hardwareprofile/mutating.go`) is not wired to an active
`MutatingWebhookConfiguration` and is expected to be removed; it remains only as a behavioral
reference for those two modules.

### Prior art: KServe's "Managed DRA"

`opendatahub-io/kserve`'s `LLMInferenceService` controller already ships a small, structured DRA
convenience feature (`pkg/controller/v1alpha2/llmisvc/managed_dra.go`): device class + CEL
selectors + count + target container name, explicitly scoped to be a convenience layer, not a
full `ResourceClaimTemplateSpec` embed. It reconciles a generated, per-workload
`ResourceClaimTemplate` owned by the workload object. This is the closest real precedent for DRA
UX in the product and informed the alternatives considered below, but does not itself dictate the
`HardwareProfile` schema decision.

### Decision: reference-only field, wrapped for future extension

`HardwareProfileSpec` gains one new optional field:

```go
type HardwareProfileSpec struct {
    Identifiers    []HardwareIdentifier `json:"identifiers,omitempty"`
    SchedulingSpec *SchedulingSpec      `json:"scheduling,omitempty"`

    // DRA references an existing Dynamic Resource Allocation ResourceClaimTemplate that
    // workloads using this hardware profile should attach. The referenced object must
    // already exist in the workload's namespace; it is not created, owned, or cleaned up
    // by any opendatahub-operator-family component. Independent of SchedulingSpec — a
    // profile may combine a DRA reference with either Queue or Node scheduling, or neither.
    // +optional
    DRA *DRASpec `json:"dra,omitempty"`
}

type DRASpec struct {
    // ResourceClaimTemplateName names a pre-existing ResourceClaimTemplate in the
    // workload's namespace. Must be a valid DNS subdomain name (RFC 1123), matching
    // ResourceClaimTemplate.metadata.name's own naming constraint. Existence (not just
    // well-formedness) is checked by the consuming module's validating webhook at
    // admission time, not by this CRD's own schema.
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    // +kubebuilder:validation:MaxLength=253
    // +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$`
    ResourceClaimTemplateName string `json:"resourceClaimTemplateName"`
}
```

`DRASpec` is mirrored identically into `api/infrastructure/v1alpha1`, matching how `Identifiers`
and `SchedulingSpec` are already kept in sync across both versions of this CRD today:

```go
type HardwareProfileSpec struct {
    Identifiers    []HardwareIdentifier `json:"identifiers,omitempty"`
    SchedulingSpec *SchedulingSpec      `json:"scheduling,omitempty"`

    // This field is not supported in v1alpha1 and is only present to ensure lossless
    // round-trip conversions to v1.
    // +optional
    DRA *DRASpec `json:"dra,omitempty"`
}

type DRASpec struct {
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    // +kubebuilder:validation:MaxLength=253
    // +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$`
    ResourceClaimTemplateName string `json:"resourceClaimTemplateName"`
}
```

Key properties of this shape:

* **No CRD version bump; field mirrored into both served versions.** `v1` is the storage/hub
  version, and `v1alpha1` is deprecated but still served with CRD conversion strategy `None` (no
  `Hub`/`ConvertTo` webhook code for this CRD). Under `None`, a field defined only in `v1` would
  be pruned from a `v1alpha1` client's `GET` and could be lost on a subsequent read-modify-write
  through `v1alpha1` — the same class of problem `DataScienceCluster` solves between its
  `v1`/`v2` versions via a real conversion webhook and an annotation-stash for version-only
  fields (`api/datasciencecluster/v1/datasciencecluster_conversion.go`). That machinery is
  unnecessary here: `DRASpec` is defined identically in `v1alpha1.HardwareProfileSpec` (unused by
  any `v1alpha1` code path, present only so `None` conversion round-trips losslessly), so no
  conversion webhook is needed. The existing `TestHardwareProfileAPIConversion` round-trip test
  (`internal/webhook/hardwareprofile/conversion_integration_test.go`) is extended to cover a
  profile with `dra` set round-tripping through both versions.
* **Purely additive.** Existing `HardwareProfile` objects behave identically forever; no code path
  today reads this field.
* **No new webhook validation on `HardwareProfile` itself.** The one field needs no CEL
  discriminator today, consistent with the project's preference for CEL-based validation
  (`+kubebuilder:validation:XValidation`) over new admission Go code, the same pattern
  `SchedulingSpec` already uses for its Queue/Node discriminator.
* **Name format is validated by the CRD schema; existence is not.** `ResourceClaimTemplateName`
  carries `MaxLength=253` and a DNS-1123-subdomain `Pattern`, matching the same naming constraint
  Kubernetes enforces on `ResourceClaimTemplate.metadata.name` itself. `MinLength=1` alone would
  accept malformed values (e.g. uppercase characters, embedded `/`, over-length strings) that can
  never resolve to a real object, silently deferring that failure to the consuming module's
  admission webhook. Rejecting malformed names at the CRD-schema level is plain kubebuilder
  validation, not new Go code, and is orthogonal to the existence check in "Guardrail" below,
  which still requires a live cluster lookup.
* **`DRASpec` is a struct, not a bare field**, specifically so an inline, generated-request shape
  (see "Alternatives") could be added later as an additional optional field on the same struct
  without an API version bump or breaking change.
* **Orthogonal to `SchedulingSpec`.** A DRA reference is independent of Queue-vs-Node placement; a
  workload can combine a DRA reference with Kueue scheduling, node scheduling, or neither. Nesting
  it inside `SchedulingSpec` would incorrectly imply mutual exclusivity.
* **No lifecycle ownership.** No opendatahub-operator-family component creates, owns via
  `ownerReferences`, or cleans up the referenced `ResourceClaimTemplate`. Its entire lifecycle
  belongs to whoever authors it out-of-band (an admin, user, or separate tooling), once per
  namespace where the profile is used.

### Guardrail for the cross-namespace failure mode (tracked separately)

Because a `ResourceClaimTemplate` reference must resolve per-namespace but nothing in this API
change enforces that it does, the serving module (odh-model-controller) and notebooks module
(workbenches-operator) are expected to extend their existing validating webhooks on
`InferenceService`/`LLMInferenceService` and `Notebook` respectively to check, at the moment the
`opendatahub.io/hardware-profile-name` annotation is added or changed, that the referenced
`ResourceClaimTemplate` exists in the workload's namespace. This turns a silent, stuck-`Pending`
Pod into an immediate, clear admission rejection. Implementing this is out of scope for this ADR
(API-only) and is called out here because it materially informed the schema decision (see
"Risks").

### Behavior when the referenced template changes

Neither this ADR nor the consuming modules' planned guardrail adds any reconciliation loop that
re-checks a `HardwareProfile`'s DRA reference after admission. "Changes" after admission fall into
two cases, both of which behave the same way existing `HardwareProfile` fields already do today:

* **The `HardwareProfile`'s `dra.resourceClaimTemplateName` is edited or removed.** The consuming
  module's mutating webhook (workbenches-operator for `Notebook`, odh-model-controller for
  `InferenceService`/`LLMInferenceService`) only reads a profile at the workload's admission time,
  exactly like it does today for `Identifiers` and `SchedulingSpec`. Editing an already-referenced
  profile has no retroactive effect on already-admitted workloads; it only takes effect on the next
  workload created (or recreated) against that profile.
* **The referenced `ResourceClaimTemplate`'s content is mutated in place.** This follows stock
  Kubernetes DRA semantics, not anything opendatahub-operator-family code decides: a
  `ResourceClaimTemplate` generates a `ResourceClaim` once, at Pod creation. Editing the template
  afterward does not retroactively change any `ResourceClaim` already generated from it; only Pods
  created after the edit pick up the new content.

In both cases, the only thing this design (and the planned guardrail) checks is existence of the
named `ResourceClaimTemplate` at the moment the workload is admitted — not whether it later
changes or disappears. See "Coverage gap" in "Risks" for the resulting gap.

### Feature maturity

`HardwareProfile` itself is GA. This DRA extension ships as **Tech Preview** only: the guardrail
above only covers admission time, not a template deleted after admission but before scheduling,
and this is an intentionally minimal reference-only extension rather than a fully designed
compute-allocation model. The GA-quality answer is planned as part of `ComputeProfile` in RHOAI
3.7.

## Open Questions

* None blocking. If a concrete need for inline (generated) DRA requests arises before RHOAI 3.7's
  `ComputeProfile` work lands, it can be added to `DRASpec` in a 3.6.z stream — see "Alternatives."

## Alternatives

### Inline (generated), matching KServe's Managed DRA shape

`HardwareProfile` stores a small structured device request (device class, count, selectors); the
component that owns admission for a workload type generates a per-workload
`ResourceClaimTemplate` from it, the same way `nodeSelector`/`tolerations` are generated today from
`SchedulingSpec`.

* **Pro**: Works automatically for the cross-namespace-shared-profile case with no pre-created
  objects; directly mirrors KServe's already-shipping Managed DRA pattern, lowering UX
  fragmentation risk.
* **Con**: Requires opendatahub-operator-family components to generate, own via
  `ownerReferences`, and clean up a `ResourceClaimTemplate` on every profile/workload change — a
  lifecycle-management responsibility the chosen design avoids entirely. Limited to a "one device
  class + count + selector" shape; can't express multi-request claims, `AdminAccess`, or
  cross-request constraints. Introduces a second, independent DRA code path alongside KServe's own
  Managed DRA for `LLMInferenceService` specifically, requiring explicit reconciliation rather than
  silent duplication.

### Both (reference OR inline, mutually exclusive)

Support both shapes on `DRASpec`, enforced mutually exclusive via CEL
(`XValidation`), mirroring the discriminated-union pattern already used by `SchedulingSpec`.

* **Pro**: Covers the common case (generated) and gives an escape hatch for power users who need a
  hand-authored `ResourceClaimTemplateSpec`.
* **Con**: Inherits the lifecycle-management downside of the inline option above. The reference
  half has the same unguardable-without-a-webhook cross-namespace failure mode as the chosen
  design, so it doesn't avoid the guardrail work; it only adds the inline option's cost on top.
  Since users can already bypass `HardwareProfile` and hand-write `PodSpec.resourceClaims` for
  complex cases, the added reference escape hatch is largely redundant.

### Conversion webhook instead of a mirrored v1alpha1 field

Instead of defining `DRASpec` identically in both `v1` and `v1alpha1`, `HardwareProfile` could
adopt the same versioning mechanism `DataScienceCluster` uses between its `v1` and `v2`: promote
`v1` to a `Hub`, implement `ConvertTo`/`ConvertFrom` on `v1alpha1`, patch the CRD to
`spec.conversion.strategy: Webhook`, and stash the `v1`-only `dra` value in an annotation during
down-conversion so it survives a round trip through `v1alpha1` (restoring it on up-conversion),
mirroring `api/datasciencecluster/v1/datasciencecluster_conversion.go`'s handling of `v2`-only
sub-component state.

* **Pro**: The general-purpose, textbook-correct pattern for a field that truly cannot be
  expressed in an older version's schema.
* **Con**: Requires standing up conversion-webhook infrastructure (webhook serving, RBAC, CRD
  patch) that doesn't exist for this CRD today, solely to protect a deprecated, no-longer-written
  API version. Unnecessary here because `DRASpec` is trivially expressible in `v1alpha1` too —
  mirroring the field is strictly simpler and keeps `None` conversion safe, which is why it was
  chosen instead. Kept here as the fallback pattern if a future `HardwareProfile` field genuinely
  can't be mirrored (e.g., a type only `v1`'s dependencies can express).

### Why reference-only was chosen over both alternatives

1. It's the only option where no opendatahub-operator-family component generates, owns, or cleans
   up a `ResourceClaimTemplate` — nothing is created, so there's nothing to manage.
2. A bare reference field carries over cleanly to a future `ComputeProfile` CRD with no generated-
   object lifecycle code to port or unwind, simplifying the RHOAI 3.7 migration.
3. The main argument against reference-only — a silent, unguardable cross-namespace failure — is
   mitigated by the validating-webhook guardrail described in "How," not left unaddressed.
4. The `DRASpec` wrapper keeps both alternatives available as later, non-breaking additions if a
   real need for inline generation emerges before `ComputeProfile` ships.

## Security and Privacy Considerations

* The referenced `ResourceClaimTemplate` is not validated for existence by this CRD's schema (CEL
  cannot perform live cross-object, cross-namespace lookups). Until the serving and notebooks
  modules implement their validating-webhook guardrail, a workload can reference a
  `HardwareProfile` whose DRA template doesn't exist in that workload's namespace, resulting in a
  stuck-`Pending` Pod rather than a security issue, but a confusing failure mode worth calling out.
* This ADR introduces no new RBAC, credentials, or cross-namespace read access in
  opendatahub-operator itself; `HardwareProfile` remains namespace-scoped and the new field is a
  plain string name, not a live reference resolved by this repo's code.
* **Trust model for cross-namespace-shared profiles.** A shared `HardwareProfile` assumes whoever
  creates the same-named `ResourceClaimTemplate` in each referencing namespace matches the
  profile author's intent (device class, quantities, etc.); there's no allow-list letting the
  profile owner restrict which namespaces may do so, unlike `ClusterQueue.spec.namespaceSelector`
  for this CRD's own `SchedulingSpec.Kueue` field. In practice this is low-stakes: the referenced
  `ResourceClaimTemplate` always lives in, and only ever affects, the workload's own namespace
  (never read cross-namespace), and Kubernetes 1.33+ already grants the default namespace `edit`
  role — the same access a user needs to run any workload — full create/update/delete on
  `ResourceClaimTemplate`. So this adds no new capability; anyone who could exploit a mismatched
  template could equally hand-write `PodSpec.resourceClaims` directly, bypassing `HardwareProfile`
  entirely. The realistic failure mode is a namespace's DRA behavior quietly drifting from what the
  shared profile intended — a correctness/support cost (see "Operational burden" below), not a
  security exposure — and doesn't warrant a `namespaceSelector`-style gate at this stage.

## Risks

* **Coverage gap**: the planned validating-webhook guardrail (owned by workbenches-operator and
  odh-model-controller, out of scope here) only checks template existence at admission time, not
  after. A template deleted, mutated, or repointed post-admission but pre-scheduling still results
  in a stuck Pod (or a Pod scheduled against a claim different from what was reviewed at admission
  time — see "Behavior when the referenced template changes"). This is the primary reason the
  overall feature ships as Tech Preview.
* **Operational burden**: a shared `HardwareProfile` referenced from many namespaces
  (`opendatahub.io/hardware-profile-namespace`) requires the same-named `ResourceClaimTemplate` to
  be separately pre-created in every one of those namespaces, by whoever authors it out-of-band.
  This burden is inherent to the reference-only design and does not go away with the guardrail;
  the guardrail only makes the failure loud instead of silent. See "Security and Privacy
  Considerations" for the related (low-stakes) trust model.
* **Dependency on out-of-scope work**: the value of this API change is fully realized only once
  workbenches-operator and odh-model-controller implement the consuming webhook logic and the
  guardrail. Shipping the API ahead of that work is intentional (per RHOAIENG-87978's scope) but
  means the field is inert in a cluster until those modules pick it up.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| --- | --- | --- | --- |
| AI Core Platform (Operator) | | 2026-08-27 | Yes |
| Model Serving (odh-model-controller) | | 2026-08-27 | Yes |
| Workbenches / Notebooks (workbenches-operator) | | 2026-08-27 | Yes |

* **AI Core Platform (Operator)**: Owns the `HardwareProfile` CRD/API change in this ADR, and will
  own the future `HardwareProfile` → `ComputeProfile` migration in RHOAI 3.7.
  ([RHOAIENG-87982](https://issues.redhat.com/browse/RHOAIENG-87982))
* **Model Serving (odh-model-controller)**: Owns admission-time injection for
  `InferenceService`/`LLMInferenceService` and the proposed validating-webhook guardrail for those
  types; also owns reconciling this design with KServe's own native Managed DRA mechanism to avoid
  two independent DRA code paths on `LLMInferenceService`.
* **Workbenches / Notebooks (workbenches-operator)**: Owns admission-time injection for `Notebook`
  and the proposed validating-webhook guardrail for that type.

## References

* [RHOAIENG-87978](https://issues.redhat.com/browse/RHOAIENG-87978) — [Spike] Design HWP DRA extension
* [RHOAIENG-87979](https://issues.redhat.com/browse/RHOAIENG-87979) — follow-on implementation work
* [RHOAIENG-87982](https://issues.redhat.com/browse/RHOAIENG-87982) — ComputeProfile design spike (RHOAI 3.7)
* [RHOAIENG-62580](https://issues.redhat.com/browse/RHOAIENG-62580) — HWP injection moved to workbenches-operator
* [RHOAIENG-62536](https://issues.redhat.com/browse/RHOAIENG-62536) — HWP injection moved to odh-model-controller
* [KServe Managed DRA](https://github.com/opendatahub-io/kserve/blob/97dac48d95ad7799db6a63788403595523e7168b/pkg/controller/v1alpha2/llmisvc/managed_dra.go)
* [RHOAI 3.6 Scoping Proposal Summary — DRA plan](https://docs.google.com/document/d/1GBGqQBagbnmM435VqsU-VKrrYY-UQ7lZ4s3QiPovWWE/edit?tab=t.k8q6vta8fg1y)

## Reviews

| Reviewed by | Date | Notes |
| --- | --- | --- |
| | | |
