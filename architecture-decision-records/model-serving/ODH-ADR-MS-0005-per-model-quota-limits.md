# Open Data Hub - Per-Model Quota Limits

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-24 |
| Scope          | AI Gateway, Models-as-a-Service (MaaS), Quota Management |
| Status         | Draft |
| Authors        | [Marius Danciu](@marius.danciu) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This ADR introduces per-model token quota limits defined on the `MaaSModelRef` CR. These limits cap the aggregate token consumption for a model deployment across all users and subscriptions, protecting the inference backend from overload regardless of how many subscriptions reference it.

## Why

Today, `MaaSSubscription` defines token rate limits scoped to subscription owners (users/groups). These limits control how much each individual subscriber can consume, but they do not constrain the total load on the model itself. If an administrator creates many subscriptions pointing to the same model, the aggregate traffic can exceed what the inference deployment can handle. There is no mechanism to set a global ceiling that protects the model deployment.

For example, 100 subscriptions each allowing 1,000 tokens/minute on the same model could produce up to 100,000 tokens/minute of aggregate load. The model deployment has no say in this; it can only absorb or drop requests.

A per-model quota provides a fundamentally different protection layer: it caps the model regardless of how many users or subscriptions exist, ensuring the inference backend operates within its capacity.

## Goals

* Define per-model token quota limits on the `MaaSModelRef` CR
* Protect model deployments from aggregate overload across all subscriptions
* Keep the mechanism orthogonal to per-subscription limits (both enforce independently)
* Leverage existing Kuadrant/Limitador infrastructure for enforcement

## Non-Goals

* Replacing or modifying per-subscription token rate limits in `MaaSSubscription`
* Request-based (non-token) rate limiting at the model level
* Autoscaling model deployments based on quota utilization
* Per-tenant aggregate limits (may be addressed in a future ADR)
* Quota management UI/dashboard

## How

### API Changes

Add an optional `quotaLimits` field to `MaaSModelSpec`:

```go
// MaaSModelSpec defines the desired state of MaaSModelRef
type MaaSModelSpec struct {
    // ModelRef references the actual model endpoint
    ModelRef ModelReference `json:"modelRef"`
    // EndpointOverride, when set, overrides the endpoint URL
    // +optional
    EndpointOverride string `json:"endpointOverride,omitempty"`
    // TenantRef is the name of the AITenant this model belongs to
    // +optional
    TenantRef string `json:"tenantRef,omitempty"`
    // QuotaLimits defines model-wide token rate limits that cap aggregate
    // consumption across all users and subscriptions. These limits protect
    // the inference backend regardless of how many subscriptions reference
    // this model.
    // +optional
    QuotaLimits []TokenRateLimit `json:"quotaLimits,omitempty"`
}
```

The `TokenRateLimit` type already exists in `maassubscription_types.go` and is reused here:

```go
type TokenRateLimit struct {
    Limit  int64  `json:"limit"`
    Window string `json:"window"`
}
```

### Example CR

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: granite-ref
  namespace: llm
spec:
  modelRef:
    kind: LLMInferenceService
    name: granite-3b
  quotaLimits:
    - limit: 50000
      window: 1m
    - limit: 1000000
      window: 1h
```

This model allows at most 50,000 tokens/minute and 1,000,000 tokens/hour globally, regardless of how many subscriptions or users send requests.

### Enforcement

The maas-controller generates a `TokenRateLimitPolicy` (TRLP) targeting the model's HTTPRoute. Unlike subscription-generated TRLPs which use per-user counters (keyed by user identity), the model-level TRLP uses a single global counter with no user dimension:

```
Subscription TRLP counter key:  (model, user/group)  -> per-subscriber limit
Model quota TRLP counter key:   (model)              -> global model limit
```

Both policies target the same HTTPRoute. Kuadrant/Limitador evaluates them independently: a request must pass both the subscriber's limit and the model's global quota. The most restrictive one wins for any given request.

#### TRLP Generation

When `spec.quotaLimits` is non-empty, the MaaSModelRef controller creates a TRLP named `quota-<modelref-name>` in the model's namespace:

```yaml
apiVersion: kuadrant.io/v1
kind: TokenRateLimitPolicy
metadata:
  name: quota-granite-ref
  namespace: llm
  labels:
    maas.opendatahub.io/model: granite-ref
    maas.opendatahub.io/quota: "true"
  ownerReferences:
    - apiVersion: maas.opendatahub.io/v1alpha1
      kind: MaaSModelRef
      name: granite-ref
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: <model-httproute>
  limits:
    global:
      rates:
        - limit: 50000
          window: 1m
        - limit: 1000000
          window: 1h
```

The key difference from subscription TRLPs is the absence of `counters` and `when` conditions that partition by user identity. This creates a single shared counter across all callers.

#### Lifecycle

| Event | Controller action |
|-------|-------------------|
| `quotaLimits` added to MaaSModelRef | Create quota TRLP |
| `quotaLimits` updated | Update quota TRLP limits |
| `quotaLimits` removed | Delete quota TRLP |
| MaaSModelRef deleted | Quota TRLP garbage-collected via ownerReference |

### Status Reporting

Add a `QuotaStatus` field to `MaaSModelStatus` to report the state of the generated quota TRLP:

```go
type MaaSModelStatus struct {
    // ... existing fields ...

    // QuotaStatus reports the status of the generated model-level
    // TokenRateLimitPolicy, if spec.quotaLimits is set.
    // +optional
    QuotaStatus *ResourceRefStatus `json:"quotaStatus,omitempty"`
}
```

A new condition type `QuotaEnforced` is added to the MaaSModelRef conditions:

| Condition | True | False |
|-----------|------|-------|
| `QuotaEnforced` | Quota TRLP is accepted and enforced by Kuadrant | TRLP not yet enforced or errored |

The `QuotaEnforced` condition is only present when `spec.quotaLimits` is non-empty. It does not affect the existing `Phase` semantics (a model can be `Ready` without quota limits).

### Interaction with Subscription TRLPs

Both layers enforce independently on the same HTTPRoute:

```
Request arrives at HTTPRoute
    │
    ├── Subscription TRLP: is this user within their subscription limit?
    │       NO  → 429 (subscription quota exceeded)
    │       YES ↓
    │
    ├── Model Quota TRLP: is the model within its global limit?
    │       NO  → 429 (model quota exceeded)
    │       YES ↓
    │
    └── Request forwarded to inference backend
```

Kuadrant evaluates all TRLPs targeting the same HTTPRoute. Both must pass. The response headers should distinguish which limit was hit (Kuadrant includes `X-RateLimit-*` headers from Limitador).

### Shared HTTPRoute Consideration

The existing limitation documented in [quota-and-access-configuration.md](docs/content/configuration-and-management/quota-and-access-configuration.md) applies here as well: when multiple MaaSModelRefs share the same HTTPRoute, their quota TRLPs may conflict. The same mitigation applies — prefer dedicated routes per model. Because model quota TRLPs are owned by the MaaSModelRef (not a subscription), the conflict surface is simpler: at most one quota TRLP per MaaSModelRef per HTTPRoute.

## Open Questions

1. **Metrics and observability**: Should the controller expose Prometheus metrics for model quota utilization (e.g., `maas_model_quota_usage_tokens_total`, `maas_model_quota_remaining_tokens`)? Limitador already exposes counters, but a MaaS-level metric may be easier to consume. Most likely this is not needed but worths calling it out.

2. **ExternalModel support**: ExternalModels may have their own provider-side rate limits. At MaaS level we don't distinguish between external and internal model so the same rate limits apply. But for awareness the external provider may already configured other rate limits that we don't know about.

## Alternatives

### 1. New CRD: MaaSModelQuota

Introduce a dedicated `MaaSModelQuota` CR rather than extending `MaaSModelRef`.

```yaml
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelQuota
metadata:
  name: granite-quota
  namespace: llm
spec:
  modelRef:
    name: granite-ref
  tokenRateLimits:
    - limit: 50000
      window: 1m
```

**Pros:**
- Separation of concerns: model registration vs. quota management
- Could be managed by a different persona (capacity planner vs. model deployer)
- Easier to add quota-specific fields later (burst allowance, priority classes, etc.)

**Cons:**
- Adds a new CRD to the API surface (more objects to manage, more reconciliation)
- Creates a 1:1 relationship with MaaSModelRef that adds indirection without clear benefit at this stage
- The model deployer who creates MaaSModelRef is typically the same persona who knows the deployment capacity

**Decision:** Rejected for now. The quota configuration is tightly coupled to the model deployment's capacity, and the model deployer is the natural owner. A separate CRD can be introduced later if the quota management grows complex enough to warrant its own lifecycle.

### 2. Use Kuadrant RateLimitPolicy directly (no controller management)

Administrators manually create a `RateLimitPolicy` or `TokenRateLimitPolicy` targeting the model's HTTPRoute with global counters.

**Pros:**
- No controller changes needed
- Full flexibility over Kuadrant policy configuration

**Cons:**
- Breaks the managed model: administrators must understand Kuadrant internals, HTTPRoute names, and counter configuration
- No integration with MaaSModelRef status reporting
- No lifecycle management (orphaned policies when models are removed)
- Inconsistent with the declarative MaaS approach where the controller generates all gateway policies

**Decision:** Rejected. This defeats the purpose of the MaaS abstraction layer.

### 3. Aggregate limit on MaaSSubscription

Add a `globalLimit` field to `MaaSSubscription` that applies across all owners of the subscription rather than per-user.

**Pros:**
- Reuses the existing CRD

**Cons:**
- Still subscription-scoped, not model-scoped: if multiple subscriptions reference the same model, the aggregate is still uncontrolled
- Conflates per-user and per-model concerns in the same CR
- Does not protect the model from the total number of subscriptions

**Decision:** Rejected. This addresses a different problem (shared subscription budgets) and does not solve the model protection use case.

### 4. Gateway-level default limits

Set a default `TokenRateLimitPolicy` at the Gateway level that applies to all models.

**Pros:**
- Simple, single policy
- Provides a safety net across all models

**Cons:**
- One-size-fits-all: cannot differentiate between a large GPU-backed model and a small simulator
- Coarse granularity: a single limit for the entire gateway does not map to individual model capacities
- Already partially exists (the gateway-level `defaults: limit 0` for unsubscribed paths)

**Decision:** Not mutually exclusive, but insufficient alone. A gateway-level default could serve as a backstop, but per-model limits are needed for capacity-aware protection.

## Security and Privacy Considerations

- Model quota limits are visible to cluster administrators who can read MaaSModelRef CRs. This is consistent with existing RBAC: model deployers already see the model namespace.
- Quota enforcement happens at the gateway layer (Kuadrant/Limitador), which is the same trust boundary used for subscription limits. No new trust boundaries are introduced.
- The quota TRLP is owned by the MaaSModelRef via ownerReferences, ensuring it cannot outlive the model registration.

## Risks

- **Kuadrant TRLP coexistence**: Multiple TRLPs on the same HTTPRoute may interact in unexpected ways depending on the Kuadrant version. The existing shared-route limitation (documented in quota-and-access-configuration.md) already calls this out. Testing with the target Kuadrant version is required.
- **Counter accuracy under high concurrency**: Limitador counters are eventually consistent in distributed deployments. Under burst traffic the actual token count may briefly exceed the configured limit before the counter converges. This is an inherent Limitador characteristic, not specific to this feature.
- **API surface growth**: Adding `quotaLimits` to MaaSModelRef expands the API. However, the field is optional and reuses an existing type, keeping the change minimal.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| models-as-a-service           | Marius Danciu    | 2026-08-24 | yes |
| ai-gateway-operator           |                  | 2026-08-24 | yes (RBAC if new TRLP label selectors are added) |
| platform / model deployers    |                  | 2026-08-24 | yes (new optional field in MaaSModelRef) |

## References

* [MaaSSubscription token rate limits](maas-controller/api/maas/v1alpha1/maassubscription_types.go) - existing TokenRateLimit type
* [Quota and Access Configuration](docs/content/configuration-and-management/quota-and-access-configuration.md) - current subscription-based quota docs
* [Old vs New Flow](maas-controller/docs/old-vs-new-flow.md) - tier-to-subscription migration context
* [AI-Gateway multi tenancy ADR](https://github.com/opendatahub-io/architecture-decision-records/blob/main/architecture-decision-records/model-serving/ODH-ADR-MS-0003-ai-gateway-tenancy.md)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
