# ODH-ADR-MS-0005: Subscription-Based Flow Control for MaaS

|                |                                                                                                                                                                                                                                                                                                                                                                  |
|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Date           | 2026-09-08                                                                                                                                                                                                                                                                                                                                                       |
| Scope          | Model Serving, Models-as-a-Service (MaaS), AI Gateway                                                                                                                                                                                                                                                                                                            |
| Status         | Approved                                                                                                                                                                                                                                                                                                                                                         |
| Authors        | Pierangelo Di Pilato                                                                                                                                                                                                                                                                                                                                             |
| Supersedes     | N/A                                                                                                                                                                                                                                                                                                                                                              |
| Superseded by: | N/A                                                                                                                                                                                                                                                                                                                                                              |
| Tickets        | TBD                                                                                                                                                                                                                                                                                                                                                              |
| Other docs:    | [AI Gateway multi-tenancy](ODH-ADR-MS-0003-ai-gateway-tenancy.md), [Non-MaaS flow control](https://github.com/opendatahub-io/odh-model-controller/blob/main/internal/controller/resources/template/FLOW_CONTROL.md), [Design doc](https://github.com/opendatahub-io/models-as-a-service/blob/main/docs/content/architecture-internals/proposals/flow-control.md) |

## What

Integrate MaaS with Gateway API Inference Extension flow control, using the AI tenant as the fairness key and the
selected subscription as the logical objective key. An optional subscription request priority drives controller-managed
`InferenceObjective` resources for the pools backing that subscription's models. When priority is unset, requests use
scheduler priority `0` without a generated objective.

## Why

MaaS subscriptions define model access, token rate limits, and billing metadata. Rate limits constrain consumption over
time; flow control determines dispatch priority and fairness when inference capacity is scarce. Subscription
administrators should configure this once, without discovering pools or maintaining objectives in model namespaces.

## Goals

* Apply one subscription request priority to all eligible models in that subscription.
* Group requests by AI tenant for fairness within each priority band.
* Reuse existing flow-control headers and the `InferenceObjective` API.
* Keep objectives synchronized with subscription and model topology changes, with operator visibility and live updates.

## Non-Goals

* Change authorization, subscription selection, token rate limits, billing, or non-MaaS Gateway defaults.
* Introduce `service_tier` overrides, per-user fairness, per-model priorities, or subscription-level scheduler policies.
* Guarantee latency, minimum capacity, or global fairness across independent pools or schedulers.
* Support flow control for external providers or routes without an `InferencePool`.

## How

### Subscription priority and tenant fairness

Add optional `MaaSSubscription.spec.requestPriority`, a signed `int32` matching `InferenceObjective.spec.priority`.
Higher values mean higher scheduling priority. This field is independent of the existing `spec.priority`, which
continues to control automatic subscription selection.

| Request priority     | Objective lifecycle  | Scheduler priority |
|----------------------|----------------------|--------------------|
| Unset                | No objective created | Default `0`        |
| Explicit `0`         | Objective created    | `0`                |
| Other explicit value | Objective created    | Configured value   |

Preserve the distinction between unset and explicit `0` throughout the API and UI; do not default the field in the CRD.

The scheduler groups flows by `(fairness ID, priority)`. Subscriptions of the same tenant at the same priority share a
flow, so additional subscriptions do not grant additional fairness shares. Fair sharing requires an explicitly selected
scheduler fairness policy; the default global-strict policy does not provide tenant fairness. This applies only where
requests compete in the same scheduler.

### Objective ownership and discovery

The MaaS subscription controller resolves each subscription's tenant and models, validating that the models are exposed
through that tenant's Gateway. For `LLMInferenceService` models, it discovers the active pool from
`status.router.scheduler.inferencePool`, respecting the observed namespace and reference for managed and explicit pools.

When request priority is set, the controller reconciles one objective per distinct `(subscription, pool)` pair in the
pool's namespace, copying the priority and setting `spec.poolRef.name`. Models sharing a pool share that objective.
Names must be deterministic, collision-resistant across tenant/subscription/pool identities, and independent of
priority.

Reconcile priority, membership, tenant, and pool changes; remove obsolete objectives when priority is cleared or the
subscription is deleted. Track ownership and provide cross-namespace cleanup, since owner references cannot garbage
collect across namespaces. Do not take over unrelated objectives.

Publish per-model objective names, pool associations, and reconciliation state in subscription status, including
mappings when priority is unset. maas-api consumes these mappings through its informer cache, keeping pool discovery and
naming in the control plane without a Kubernetes lookup per request.

### Trusted request classification

After authentication, subscription selection, and model authorization, the MaaS AuthPolicy injects these headers from
trusted maas-api metadata for every request targeting a pool:

| Canonical header                | Legacy alias                      | Value                                      |
|---------------------------------|-----------------------------------|--------------------------------------------|
| `x-llm-d-inference-fairness-id` | `x-gateway-inference-fairness-id` | Stable, unambiguous AI tenant identity     |
| `x-llm-d-inference-objective`   | `x-gateway-inference-objective`   | Generated subscription/pool objective name |

Always inject both pairs, including when priority is unset: an absent objective resolves to scheduler priority `0`.
Replace all client-supplied values, including duplicates and case variants, with exactly one trusted value per header.
Canonical and legacy values must match. Apply the same classification to path-based and body-based model routing, using
the authorized target pool. A request without a valid subscription cannot obtain a scheduling identity from client
headers.

### Operator experience and propagation

Authorized operators can view, set, change, and clear **Request priority** independently of subscription-selection
priority. Show an unset value as **Scheduler default (0)** while preserving its unset state. Subscription details expose
objective names, pool associations, and pending or failed reconciliation; unset priority indicates no objective is
required.

Priority changes update objectives in place. Setting or clearing priority creates or removes objectives while retaining
stable names and request mappings. Updates propagate asynchronously through the controller, maas-api, and the endpoint
picker (EPP), without recreating subscriptions, changing credentials, or restarting services. Status reflects controller
progress, not confirmation from every EPP instance; requests may temporarily use the old priority or default `0`.

### Compatibility and scheduler responsibilities

Flow control requires a compatible `InferenceObjective` API and an enabled inference scheduler. Missing optional APIs
must not disrupt existing MaaS functionality. Report pending pool discovery, unsupported backends, and objective
failures per model without blocking other models or claiming that unenforced priority is active.

MaaS owns request classification and objective lifecycle. KServe and the serving stack own pools and scheduler
deployment; model owners configure fairness, dispatch gating, ordering, queue limits, and eviction. Priority is not a
capacity reservation or latency guarantee: lower-priority requests can starve, and negative-priority requests can be
interrupted when in-flight eviction is enabled. These consequences must be visible to subscription operators.

## Security and Privacy Considerations

Only authorized subscription editors may change request priority. The controller needs service/pool read access and
objective management permissions in model namespaces, reflected in both AI Gateway and ODH parent-operator RBAC.
Preserve tenant-to-model Gateway validation, restrict direct objective edits to authorized administrators, and exclude
credentials from headers and status mappings. Scheduler fallback must never bypass access control or token rate limits.

Cross-tenant pool sharing is currently unsupported. Before enabling it, define platform-controlled priority entitlements
(ranges or allowed values) to prevent tenants from granting themselves higher bands; fairness within a band does not
prevent priority escalation. Tenant-level priority restrictions are deferred from this implementation.

## Deferred Work

* **Priority governance:** Define tenant entitlements, handling of existing subscriptions when entitlements change, and
  treatment of unset priority (`0`) before introducing cross-tenant pool sharing.
* **Pricing and chargeback:** Existing `billingRate.perToken` can price subscription tiers, but changing request
  priority does not change rates automatically. Durable usage/rate attribution and billing based on actual EPP-applied
  priority require separate billing work.
* **Additional scheduling inputs:** Subscription SLO controls require trusted derivation or validation of deadline
  headers. Protecting fairness and objective headers alone does not govern those inputs.

## Alternatives

| Alternative                                  | Tradeoff                                                                                                          |
|----------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Manually manage objectives                   | Avoids controller changes but requires pool discovery and cross-namespace lifecycle management by administrators. |
| Use subscriptions as fairness keys           | Gives tenants with more subscriptions more fairness shares at the same priority.                                  |
| Use tenants as objective keys                | Cannot express different subscription priorities within a tenant.                                                 |
| Use only token rate limits                   | Controls consumption over time, but not dispatch under contention.                                                |
| Reuse subscription-selection `spec.priority` | Couples selection to scheduling and changes the effect of existing priorities.                                    |
| Reference centrally managed priority tiers   | Constrains priority choices, but adds shared configuration and governance; deferred.                              |
| Share objectives by pool and priority        | Reduces object count, but complicates ownership and cleanup and requires mapping changes when priority changes.   |

## Risks

* **Resource growth:** Objective count scales with distinct subscription/pool pairs; reconciliation must avoid
  full-cluster scans for each change.
* **Topology mismatch:** Classification and forwarding must resolve the same pool, including body-based routing. Routes
  selecting multiple pools need an explicit mapping strategy before they can be supported.
* **Eventual consistency:** Updates are not atomic across pools; missing objectives temporarily fall back to priority
  `0`
  and must be reported when an explicit priority requires them.

Validation should cover objective lifecycle across shared and changing pools, tenant/name collisions, priority
set/change/clear transitions, forged headers, both routing modes, missing optional APIs, and UI status. End-to-end
checks must verify priority ordering and configured tenant fairness under contention, canonical/legacy header
compatibility, and the distinction between unset and explicit `0`.

## Stakeholder Impacts

| Group             | Key Contacts                            | Date          | Impacted?                                                               |
|-------------------|-----------------------------------------|---------------|-------------------------------------------------------------------------|
| MaaS / AI Gateway | @pierdipi, @mariusdanciu, @jland-redhat | Sept 14, 2026 | objective reconciliation, subscription metadata, and AuthPolicy headers |
| Dashboard         | @andrewballantyne                       | Sept 17, 2026 | request priority in MaaSSubscription pages                              |

## References

* [MaaS architecture context](https://github.com/opendatahub-io/architecture-context/blob/main/architecture/rhoai.next/models-as-a-service.md)
* [MaaS subscription API](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/api/maas/v1alpha1/maassubscription_types.go)
* [MaaS subscription controller](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/pkg/controller/maas/maassubscription_controller.go)
* [MaaS AuthPolicy generation](https://github.com/opendatahub-io/models-as-a-service/blob/main/maas-controller/pkg/controller/maas/maasauthpolicy_controller.go)
* [LLMInferenceService API and observed scheduler status](https://github.com/opendatahub-io/kserve/blob/main/pkg/apis/serving/v1alpha2/llm_inference_service_types.go)
* [InferenceObjective schema bundled with KServe](https://github.com/opendatahub-io/kserve/blob/main/config/llmisvc/gateway-inference-extension.yaml)
* [llm-d EPP HTTP headers reference](https://llm-d.ai/docs/0.8/api-reference/epp-http-headers)
* [llm-d flow-control configuration API](https://github.com/llm-d/llm-d-router/blob/main/apix/config/v1alpha1/endpointpickerconfig_types.go)
* [llm-d fairness policies](https://github.com/llm-d/llm-d-router/tree/main/pkg/epp/framework/plugins/flowcontrol/fairness)
* [llm-d request-priority resolution](https://github.com/llm-d/llm-d-router/blob/main/pkg/epp/requestcontrol/director.go)

## Reviews

| Reviewed by   | Date          | Notes    |
|---------------|---------------|----------|
| @mariusdanciu | Sept 14, 2026 | Approved |
