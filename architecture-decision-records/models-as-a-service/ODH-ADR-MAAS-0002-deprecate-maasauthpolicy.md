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

The non-inference endpoints with a MaaSAuthPolicy-specific dependency are **`GET /v1/models`** (calls `accessChecker.AuthorizedModels()` to filter the model list) and the **admin inventory API** (calls `authpolicy.Checker.ModelAccessSubjects()` to list subjects per model). Both must migrate to subscription-based ownership (Phase 1, steps 3 and 4).

### Phase 1: Prepare MaaSSubscription as the authorization source (v3.x)

1. **Extend the MaaSSubscription reconciler** to generate and manage the singleton gateway-level Kuadrant AuthPolicy (`maas-gateway-auth`). The `buildGatewayAuthPolicySpec` logic moves from the MaaSAuthPolicy reconciler to the MaaSSubscription reconciler (or a shared function both can call during transition).

2. **Derive `accessAllowed` from subscriptions in maas-api.** The subscription selector (`internal/subscription/selector.go`) currently calls `authpolicy.Checker.AuthorizedModels()` to compute `accessAllowed`. Change this to derive access from the subscription's own `owner` field: if the requesting user/group matches any active MaaSSubscription's `owner` for the requested model, access is allowed.

3. **Migrate `/v1/models` listing to subscription-based filtering.** The `GET /v1/models` endpoint currently uses `accessChecker.AuthorizedModels()` (which reads MaaSAuthPolicy CRs from cache) to filter the model list to only show models the user is authorized to see. This must be changed to derive the authorized model set from subscription ownership instead: a model is visible if any MaaSSubscription exists where the requesting user/group matches the subscription's `owner` field for that model. This is the only non-inference endpoint affected by MaaSAuthPolicy removal.

4. **Migrate `SubjectInventoryProvider` to subscription-sourced data.** The admin inventory API uses `authpolicy.Checker.ModelAccessSubjects()` to list subjects per model. This must be changed to derive the subject list from MaaSSubscription `owner` fields instead.

5. **Absorb `MeteringMetadata` into `TokenMetadata`.** MaaSAuthPolicy's `MeteringMetadata` (organizationId, costCenter, labels) overlaps with MaaSSubscription's `TokenMetadata`. MaaSSubscription already has `TokenMetadata` with the same fields, plus the `Organization` field. Verify all consumers use subscription-sourced metadata and drop the MaaSAuthPolicy copy. Note: MaaSAuthPolicy `MeteringMetadata` is not referenced anywhere in the codebase today — the response headers (`x-maas-organization-id`, `X-MaaS-Cost-Center`) are already sourced from `subscription-info`, not from MaaSAuthPolicy.

### Pre-migration validation

Before switching the authorization source, audit existing clusters for divergence between MaaSAuthPolicy `subjects` and MaaSSubscription `owner` fields. Today, access is the intersection of both — the subscription selector checks MaaSAuthPolicy via `accessAllowed`. Removing that check means access is determined by subscription ownership alone. If any subscription has a broader `owner` than the corresponding MaaSAuthPolicy `subjects`, the switch would expand access for those users/groups. Run a comparison of `(subject, model)` pairs from both sources and resolve any mismatches before proceeding to Phase 2.

### Phase 2: Deprecate and Remove MaaSAuthPolicy (v3.x+1)

A MaaSAuthPolicy without a corresponding MaaSSubscription provides no usable access today — the subscription selector already rejects requests with no matching subscription. There is no scenario where a user has a MaaSAuthPolicy but no MaaSSubscription and can actually reach a model. Therefore, no dual-source fallback is needed.

1. **Mark MaaSAuthPolicy as deprecated** via CRD version metadata (`spec.versions[*].deprecated` and `deprecationWarning`), printer column, documentation, and controller log warnings.

2. **Webhook update**: Change the MaaSAuthPolicy validating webhook to emit an admission warning (not rejection) on create, informing users the resource is deprecated and will be removed in the next release.

3. **Stop reconciling MaaSAuthPolicy**: The MaaSAuthPolicy controller becomes a no-op (or is removed). The subscription reconciler now owns the singleton gateway AuthPolicy.

### Phase 3: Remove MaaSAuthPolicy (v3.x+2)

1. **Cleanup existing MaaSAuthPolicy CRs**: Provide a migration note advising administrators to delete existing MaaSAuthPolicy resources (which are no longer reconciled) while the CRD API is still available.
2. **Remove the MaaSAuthPolicy controller**, webhook, `authpolicy.Checker`, and all associated tests.
3. **Remove the maas-api `authpolicy` package** and the `accessChecker` dependency from the subscription selector.
4. **Remove the CRD from downstream consumers** (PriceTag already does not use it; update ai-gateway-operator and opendatahub-operator RBAC).
5. **Update E2E tests**: ~18 test files reference MaaSAuthPolicy; update to use MaaSSubscription for access control assertions.
6. **Remove the MaaSAuthPolicy CRD** from generated manifests and kustomize bases.

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
- **Subscription ownership becomes the authorization boundary.** The `owner` field in MaaSSubscription must be treated as a security-critical field. Ensure RBAC restricts who can create/modify subscriptions.
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
