# Open Data Hub - Architecture Decision Record: Deprecate MaaSAuthPolicy in Favor of MaaSSubscription

|                |            |
| -------------- | ---------- |
| Date           | 2026-10-30 |
| Scope          | MaaS Controller, MaaS API |
| Status         | Proposed |
| Authors        | [Marius Danciu](https://github.com/mariusdanciu) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHAIRFE-3737](https://redhat.atlassian.net/browse/RHAIRFE-3737) |
| Other docs     | none |

## What

This ADR proposes deprecating and eventually removing the `MaaSAuthPolicy` custom resource, consolidating its authorization semantics into `MaaSSubscription`. The Authorino gateway pipeline (authentication, CEL checks, header injection, anti-spoofing) will be driven by subscription data instead of a parallel CRD.

## Why

`MaaSAuthPolicy` and `MaaSSubscription` both express the same subject-to-model binding — "these groups/users can access these models." Today the system maintains two independent declarations of this relationship:

- **MaaSAuthPolicy** defines `subjects` (groups/users) and `modelRefs` for authorization.
- **MaaSSubscription** defines `owner` (groups/users) and `modelRefs` for entitlement, rate limits, and flow control.

The actual access decision is already made by the subscription selector (`/internal/v1/subscriptions/select`), which is called as a metadata step inside the Authorino pipeline. The `accessAllowed` field in the `SelectResponse` is gated on the `MaaSAuthPolicy` checker, creating a circular dependency: the subscription selector checks MaaSAuthPolicy to determine access, then the Authorino CEL rules check `subscription-info.accessAllowed`.

This dual-source design leads to:

1. **Redundant configuration**: Administrators must create and maintain both a MaaSAuthPolicy and a MaaSSubscription for the same user-model pair. Forgetting either blocks access or creates inconsistencies.
2. **Drift risk**: MaaSAuthPolicy subjects and MaaSSubscription owners can diverge, causing confusing authorization failures.
3. **Increased complexity**: The MaaSAuthPolicy reconciler (~2500 lines) builds a singleton Kuadrant AuthPolicy with complex CEL/OPA rules. Much of this logic exists to bridge the gap between two data sources that encode the same relationship.
4. **Downstream fragmentation**: PriceTag (redhat-et/pricetag) already operates without MaaSAuthPolicy, using subscriptions as the sole access control mechanism. Maintaining a CRD that downstream consumers skip adds no value.

## Goals

* Eliminate the redundant subject-to-model binding by using MaaSSubscription as the single source of truth for "who can access which models."
* Preserve the full Authorino gateway pipeline (authn methods, CEL checks, header injection, anti-spoofing, error responses) currently wired by the MaaSAuthPolicy reconciler.
* Provide a backward-compatible migration path that supports running both old and new semantics during a transition period.
* Remove the MaaSAuthPolicy CRD, webhook, controller, and maas-api checker after the deprecation period.

## Non-Goals

* Changing the authentication methods (API key, OIDC, Kubernetes TokenReview) — these are infrastructure concerns orthogonal to the authorization source.
* Modifying the MaaSSubscription spec fields for rate limiting, billing, guardrails, or InferenceObjective management.
* Removing Kuadrant AuthPolicy or Authorino from the data path — only the source of the authorization data changes.

## How

### Scope of impact on non-inference endpoints

The singleton `maas-gateway-auth` Kuadrant AuthPolicy targets the **Gateway object** itself, so the authentication pipeline (API key, OIDC, K8s TokenReview) and anti-spoofing rules apply to all routes through the gateway — including management endpoints like `/maas-api/v1/api-keys` and `/maas-api/v1/subscriptions`. However, the model-specific authorization checks (subscription validity, group membership) have a `when` guard that skips them for paths starting with `/v1/` or `/maas-api/`. These gateway-wide authn and anti-spoofing rules are infrastructure plumbing and will be preserved unchanged in the subscription-driven AuthPolicy.

Two non-inference endpoints have a MaaSAuthPolicy-specific dependency:

- **`GET /v1/models`**: calls `accessChecker.AuthorizedModels()` to filter the model list based on MaaSAuthPolicy CRs. This filtering must migrate to subscription-based ownership (Phase 1, step 3).
- **Admin inventory API**: uses `SubjectInventoryProvider.ModelAccessSubjects()` to list subjects per model, scoped by organization. This must migrate to subscription `owner` fields (Phase 1, step 4).

### Phase 1: Prepare MaaSSubscription as the authorization source (v3.x)

1. **Extend the MaaSSubscription reconciler** to generate and manage the singleton gateway-level Kuadrant AuthPolicy (`maas-gateway-auth`). The `buildGatewayAuthPolicySpec` logic moves from the MaaSAuthPolicy reconciler to the MaaSSubscription reconciler (or a shared function both can call during transition). **Single-writer invariant**: a feature gate must ensure only one reconciler writes the gateway AuthPolicy at any time. When subscription-based authorization is enabled, the MaaSAuthPolicy reconciler must skip gateway policy reconciliation; otherwise, the MaaSAuthPolicy reconciler remains the sole writer. Two reconcilers writing the same resource concurrently can cause intermittent denial or retention of stale grants.

2. **Derive `accessAllowed` from the selected subscription in maas-api.** The subscription selector (`internal/subscription/selector.go`) currently calls `authpolicy.Checker.AuthorizedModels()` to compute `accessAllowed`. Change this to bind authorization to the selected subscription: access is allowed only when the selected subscription exists, is active, and includes the requested model in its `modelRefs`. Do not authorize based on ownership of a different active subscription — this prevents cross-subscription authorization where a user could leverage subscription A while the requested model exists only in subscription B.

3. **Migrate `/v1/models` listing to subscription-based filtering.** The `GET /v1/models` endpoint currently uses `accessChecker.AuthorizedModels()` (which reads MaaSAuthPolicy CRs from cache) to filter the model list to only show models the user is authorized to see. This must be changed to derive the authorized model set from subscription ownership instead: a model is visible if a MaaSSubscription exists where the requesting user/group matches the subscription's `owner` field for that model.

4. **Migrate `SubjectInventoryProvider` to subscription-sourced data.** The admin inventory API uses `authpolicy.Checker.ModelAccessSubjects()` to list subjects per model, scoped by `organizationID`. This must be changed to derive the subject list from MaaSSubscription `owner` fields instead, preserving organization scoping: only return owners from subscriptions whose `Organization` field matches the requesting organization. Subscriptions without an `Organization` field must be handled explicitly (e.g. excluded from org-scoped queries, or matched only to a default org). Add tests confirming that owners from other organizations are not exposed in shared-model scenarios.

5. **Absorb `MeteringMetadata` into `TokenMetadata`.** MaaSAuthPolicy's `MeteringMetadata` (organizationId, costCenter, labels) overlaps with MaaSSubscription's `TokenMetadata`. MaaSSubscription already has `TokenMetadata` with the same fields, plus the `Organization` field. Verify all consumers use subscription-sourced metadata and drop the MaaSAuthPolicy copy. Note: MaaSAuthPolicy `MeteringMetadata` is not referenced anywhere in the codebase today — the response headers (`x-maas-organization-id`, `X-MaaS-Cost-Center`) are already sourced from `subscription-info`, not from MaaSAuthPolicy.

6. **Reconcile divergent grants before switching.** If a MaaSSubscription's `owner` is broader than the corresponding MaaSAuthPolicy's `subjects`, switching to subscription-based authorization could grant access to subjects that were previously denied. Before enabling the feature gate, provide a preflight validation tool that compares `(subject, model)` pairs from both resources and reports mismatches. Administrators must acknowledge and resolve divergences (by narrowing subscription owners or confirming the broader grant is intentional) before the switch is activated.

### Phase 2: Deprecate and Remove MaaSAuthPolicy (v3.x+1)

A MaaSAuthPolicy without a corresponding MaaSSubscription provides no usable access today — the subscription selector already rejects requests with no matching subscription. There is no scenario where a user has a MaaSAuthPolicy but no MaaSSubscription and can actually reach a model. Therefore, no dual-source fallback is needed.

1. **Mark MaaSAuthPolicy as deprecated** via the CRD version's `deprecated` and `deprecationWarning` fields (so Kubernetes emits API deprecation warnings on use), documentation, and controller log warnings. A printer column may be added as an additional display aid but is not sufficient on its own.

2. **Webhook update**: Change the MaaSAuthPolicy validating webhook to emit an admission warning (not rejection) on create, informing users the resource is deprecated and will be removed in the next release.

3. **Stop reconciling MaaSAuthPolicy**: The MaaSAuthPolicy controller becomes a no-op (or is removed). The subscription reconciler now owns the singleton gateway AuthPolicy.

### Phase 3: Remove MaaSAuthPolicy (v3.x+2)

1. **Validate all access grants are covered by MaaSSubscription.** Verify that no MaaSAuthPolicy CR grants access to a `(subject, model)` pair not covered by a corresponding subscription.
2. **Adopt the existing `maas-gateway-auth` AuthPolicy** from the MaaSSubscription reconciler and verify no stale authorization rules remain from the MaaSAuthPolicy-managed version.
3. **Cleanup existing MaaSAuthPolicy CRs** while the CRD API is still available. Verify that none remain in the cluster.
4. **Remove the MaaSAuthPolicy controller**, webhook, `authpolicy.Checker`, and all associated tests.
5. **Remove the maas-api `authpolicy` package** and the `accessChecker` dependency from the subscription selector.
6. **Update E2E tests**: ~18 test files reference MaaSAuthPolicy; update to use MaaSSubscription for access control assertions.
7. **Remove the CRD from downstream consumers** (PriceTag already does not use it; update ai-gateway-operator and opendatahub-operator RBAC).
8. **Remove the MaaSAuthPolicy CRD** from generated manifests and kustomize bases — only after all migration checks have passed.

### Affected components

| Component | Change |
|-----------|--------|
| `maas-controller/api/maas/v1alpha1/maasauthpolicy_types.go` | Deprecated in Phase 2, removed in Phase 3 |
| `maas-controller/pkg/controller/maas/maasauthpolicy_controller.go` | Gateway AuthPolicy logic moves to subscription reconciler; removed in Phase 3 |
| `maas-controller/pkg/controller/maas/maassubscription_controller.go` | Gains gateway AuthPolicy reconciliation |
| `maas-controller/pkg/webhook/maasauthpolicy_webhook.go` | Deprecation warning in Phase 2, removed in Phase 3 |
| `maas-api/internal/authpolicy/checker.go` | Removed in Phase 3 |
| `maas-api/internal/subscription/selector.go` | Derives `accessAllowed` from subscription ownership |
| `maas-api` `/v1/models` handler | Model list filtering switches from MaaSAuthPolicy to subscription ownership |
| `deployment/base/maas-controller/rbac/` | RBAC rules for maasauthpolicies removed in Phase 3 |
| E2E tests (~18 files) | Updated to use MaaSSubscription for access control |

## Open Questions

1. **Should the gateway AuthPolicy be owned by a dedicated controller or the subscription reconciler?** The current MaaSAuthPolicy reconciler manages the singleton `maas-gateway-auth` AuthPolicy. Moving this to the subscription reconciler adds complexity to an already large controller. An alternative is a standalone "gateway auth" controller that watches both subscriptions and tenant config, keeping subscription reconciliation focused on rate limits and InferenceObjectives.

2. **How should the deprecation be communicated?** Options: release notes only, CRD status condition, admission warning, or controller event. Recommendation: all of the above.

## Alternatives

### Alternative 1: Keep MaaSAuthPolicy as a higher-level policy abstraction

Keep MaaSAuthPolicy as a "policy template" that generates subscriptions automatically. A MaaSAuthPolicy would create MaaSSubscription CRs for each subject/model combination with default rate limits.

**Tradeoffs:**
- (+) Preserves the existing user experience for administrators who think in terms of "allow group X to access model Y."
- (-) Adds a generation layer that must be kept in sync. Subscription edits could conflict with generated subscriptions.
- (-) Does not eliminate the dual-source problem — just automates the duplication.

### Alternative 2: Merge MaaSAuthPolicy fields into MaaSSubscription spec

Add `subjects` and `meteringMetadata` fields directly to MaaSSubscription, making it a superset.

**Tradeoffs:**
- (+) Single CRD, no migration gap.
- (-) MaaSSubscription already has `owner` (groups/users) which serves the same purpose as `subjects`. Adding `subjects` would create confusion about which field is authoritative.
- (-) `MeteringMetadata` is already covered by `TokenMetadata` + `Organization`.

This alternative is essentially what this ADR proposes, but reusing the existing `owner` field rather than adding new fields.

### Alternative 3: Status quo — keep both CRDs

Continue maintaining MaaSAuthPolicy and MaaSSubscription as separate concerns.

**Tradeoffs:**
- (+) No migration effort.
- (-) Continued redundancy, drift risk, and maintenance burden.
- (-) Downstream consumers (PriceTag) already skip MaaSAuthPolicy, validating that it is not essential.

## Security and Privacy Considerations

- **No reduction in security posture.** The Authorino gateway pipeline (authn, anti-spoofing, header injection) is preserved unchanged. Only the source of the "who can access which model" decision changes.
- **Subscription entitlement becomes the authorization boundary.** When MaaSSubscription is the sole authorization source, all fields that affect access decisions — `owner`, `modelRefs`, and `Organization` — must be treated as security-critical. A principal who can create subscriptions with an arbitrary `owner` bypasses any immutability protection on existing resources. Modifying `modelRefs` can grant access to additional models without changing `owner`. Changing `Organization` can move a grant across organization boundaries. Protection requires: (a) restricting subscription create, update, and delete operations via RBAC to platform-level principals only, (b) adding a validating webhook that enforces authorization policy on changes to any entitlement field (`owner`, `modelRefs`, `Organization`), not just `owner`. Existing roles that grant subscription write access must be audited to confirm they are appropriately scoped.
- **No migration gap.** A MaaSAuthPolicy without a corresponding MaaSSubscription provides no usable access today — the subscription selector already rejects such requests. Removing MaaSAuthPolicy does not revoke any access that was actually functional.

## Risks

- **Behavioral change for maas-api consumers.** The `accessAllowed` field in `SelectResponse` will be sourced differently. Any consumer that assumes `accessAllowed` is MaaSAuthPolicy-derived may need updating. In practice, the Authorino CEL rules consume this field opaquely.
- **Increased subscription reconciler complexity.** The subscription controller gains the gateway AuthPolicy build logic (~500 lines of CEL/OPA construction). This can be mitigated by extracting the builder into a shared package.
- **Orphaned MaaSAuthPolicy CRs.** After removal, existing MaaSAuthPolicy CRs will remain in the cluster as unreconciled resources. Administrators must be informed to clean them up. This is cosmetic — they have no effect without a controller.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| MaaS Controller maintainers   |                  | 2026-10-30 | Yes — controller refactoring |
| MaaS API maintainers          |                  | 2026-10-30 | Yes — subscription selector changes |
| AI Gateway Operator           |                  | 2026-10-30 | Yes — RBAC updates in Phase 3 |
| OpenDataHub Operator          |                  | 2026-10-30 | Yes — RBAC updates in Phase 3 |
| PriceTag (redhat-et/pricetag) |                  | 2026-10-30 | Minimal — already operates without MaaSAuthPolicy |
| Platform administrators       |                  | 2026-10-30 | Yes — must ensure subscriptions cover all access grants before Phase 3 |

## References

* `maas-controller/api/maas/v1alpha1/maasauthpolicy_types.go` — MaaSAuthPolicy type definition
* `maas-controller/api/maas/v1alpha1/maassubscription_types.go` — MaaSSubscription type definition
* `maas-controller/pkg/controller/maas/maasauthpolicy_controller.go` — MaaSAuthPolicy reconciler (~2500 lines)
* `maas-api/internal/authpolicy/checker.go` — MaaSAuthPolicy access checker used by subscription selector
* `maas-api/internal/subscription/selector.go` — Subscription selector with `accessAllowed` integration

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
