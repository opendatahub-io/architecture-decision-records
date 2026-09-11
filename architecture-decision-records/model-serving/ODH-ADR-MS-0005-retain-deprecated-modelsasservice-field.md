# ODH-ADR-MS-0005: Retain deprecated `kserve.modelsAsService` field

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-17 |
| Scope          | Model Serving, Models-as-a-Service (MaaS), AI Gateway |
| Status         | Draft |
| Authors        | Luca Burgazzoli |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-60628](https://redhat.atlassian.net/browse/RHOAIENG-60628), [RHOAIENG-76610](https://redhat.atlassian.net/browse/RHOAIENG-76610) |
| Other docs:    | [odh-cli PR #140](https://github.com/opendatahub-io/odh-cli/pull/140), [opendatahub-operator PR #3723](https://github.com/opendatahub-io/opendatahub-operator/pull/3723) |

## What

As part of moving Models-as-a-Service (MaaS) into the AI Gateway module, the DataScienceCluster (DSC) field that enables/disables MaaS moved from `spec.components.kserve.modelsAsService` to `spec.components.aigateway.modelsAsAService`. We decided to retain the deprecated `kserve.modelsAsService` field in the DSC v2 API rather than remove or relocate it, default it to `Removed`, and restrict it to a one-directional `Managed -> Removed` transition.

This ADR is scoped to the `kserve.modelsAsService` field specifically. It does not cover the separate, internal-only `ModelsAsService` CR/component (`components.platform.opendatahub.io`, introduced by [ODH-ADR-MS-0002](./ODH-ADR-MS-0002-maas-tenant-cr-introduction.md) for status aggregation), which PR #3723 removed entirely rather than retained.

## Why

DSC v2 is a GA API, which constrains what we can do with the deprecated field:

- We can't remove or relocate `kserve.modelsAsService` without breaking the GA v2 API contract. CRDs use structural schemas that prune unrecognized fields on write, so any existing manifest or GitOps pipeline still setting this field directly would silently lose that configuration the moment the field disappeared from the schema.
- We can't introduce a new enum value (e.g. `Moved`) for `managementState` either. Adding values to a `status` enum is usually safe, since status is observational and well-behaved clients already tolerate unknown values; `managementState` is a `spec` field, though, so it exists to be validated and acted on. Existing clients that pre-validate or pattern-match against today's closed two-value set (`odh-cli`, generated typed clients, other controllers/policies reading this field) can reject the new value or mishandle it until updated, this is a breaking change for that client ecosystem, even though the API server itself would accept the new value once the CRD schema ships it. On top of that, a `Moved` value would misuse the field's semantics: `managementState` expresses the desired state the operator should reconcile toward, not an informational status about where the feature now lives.

Given these constraints, the only lever available is behavioral: keep the field, default it to `Removed`, and use validation to block re-enabling it, rather than change its shape or vocabulary.

## Goals

- Preserve full backward compatibility of the DSC v2 API: no field removal, no relocation, no new enum values.
- Give users and tooling a clear, non-blocking signal that `kserve.modelsAsService` is deprecated in favor of `aigateway.modelsAsAService`.
- Allow users to clean up the deprecated field (`Managed -> Removed`) after migrating.
- Prevent the deprecated field from being used to silently re-enable MaaS after cleanup.
- Support automatic detection of the deprecated configuration during 3.4 -> 3.5 upgrade assessments.

## Non-Goals

- Removing `kserve.modelsAsService` from the DSC v2 API. The field is committed to being respected "at least through 3.6"; full removal is future work tied to a separate, explicit breaking-change decision.

## How

- `aigateway.modelsAsAService.managementState` is the canonical MaaS toggle and takes precedence whenever it's set; `kserve.modelsAsService` is only consulted as a fallback when it isn't, so there's no ambiguity if both are present.
- `kserve.modelsAsService` stays in the API, documented as deprecated, and now defaults to `Removed` instead of `Managed`.
- Its allowed transitions are one-directional: `Managed -> Removed` is allowed, so users can clean up after migrating, but `Removed -> Managed` is blocked, so the deprecated field can't be used to silently re-enable MaaS.
- Setting `kserve.modelsAsService` to `Managed` now produces a non-blocking deprecation warning that points at the new field.
- Migration is automatic and non-disruptive: existing `Managed` configurations keep working (via conversion and/or fallback), so clusters and GitOps manifests aren't forced to change immediately.
- `odh-cli` proactively surfaces the deprecated usage as an upgrade-assessment advisory, rather than relying solely on the runtime warning.

## Alternatives

### Alternative 1: Remove `kserve.modelsAsService` from the DSC v2 API

**Pros**: Cleanest API surface, no deprecated field to carry forward.
**Cons**: Breaking change to a GA API; clusters or GitOps manifests still setting the field directly would have that configuration silently pruned by the structural-schema CRD validation the moment the field left the schema.

### Alternative 2: Introduce a new enum value (e.g. `Moved`) for `managementState`

**Pros**: Makes the migration state explicit in the field itself.
**Cons**: Breaking change to a GA CRD enum, and misuses the field's semantics (see Why).

### Alternative 3: Keep the original fully-immutable CEL (`self == oldSelf`)

**Pros**: Simplest possible rule; deterministic, with no risk of accidentally re-enabling the deprecated path.
**Cons**: Blocks legitimate post-migration cleanup. A user who migrates to `aigateway.modelsAsAService` could never clear the old `Managed` value, leaving a permanently-`Managed` deprecated field and a persistent deprecation `Warning` on every `oc apply`.

### Alternative 4 (chosen): Retain the field, default to `Removed`, one-directional CEL, admission `Warning`

**Pros**: No breaking change; users can migrate and clean up; re-enabling the deprecated path is explicitly blocked; tooling (`odh-cli`) can flag the situation during upgrades.
**Cons**: The deprecated field lingers in the API for at least one more release, and can still drive behavior via the `IsEnabled()` fallback, so tools and docs must point at `aigateway.modelsAsAService` as the field to check rather than assuming `kserve.modelsAsService` is now purely informational.

## Risks

- **Source-of-truth confusion**: consumers (dashboards, CLIs, docs, scripts) that still read `kserve.modelsAsService` may misinterpret it; `aigateway.modelsAsAService` must be documented as the field to check for actual MaaS state, since the deprecated field can still drive behavior through the `IsEnabled()` fallback.
- **Deferred cleanup and warning fatigue**: because clearing the field is optional, clusters may carry `kserve.modelsAsService: Managed` indefinitely, producing a deprecation `Warning` on every `oc apply`/GitOps sync until it's cleared.
- **Larger upgrade test matrix**: whether MaaS is enabled now depends on API version and storage history (fresh v2, v1-converted, or already-stored v2 with the deprecated field still `Managed`), which adds paths to validate on upgrade.
- **Future removal is still open**: this ADR doesn't resolve how or when `kserve.modelsAsService` is eventually removed from the API; that requires a future breaking-change decision.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| ODH Operator team              |                  |            | Yes |
| MaaS / AI Gateway team         |                  |            | Yes |
| odh-cli / tooling team         |                  |            | Yes |
| Documentation team             |                  |            | Yes |

## References

- [RHOAIENG-60628](https://redhat.atlassian.net/browse/RHOAIENG-60628) - Onboard Models as a Service to Modular Architecture (Platform Orchestrator)
- [RHOAIENG-76610](https://redhat.atlassian.net/browse/RHOAIENG-76610) - Replace `self == oldSelf` with one-directional CEL + admission warning on deprecated `kserve.modelsAsService`
- [opendatahub-operator PR #3723](https://github.com/opendatahub-io/opendatahub-operator/pull/3723) - feat: nest ModelsAsAService under AIGateway module
- [odh-cli PR #140](https://github.com/opendatahub-io/odh-cli/pull/140) - feat(lint): add MaaS modularization support and 3.4 -> 3.5 field migration check
- [Migration guide: maas-to-aigateway-3.5.md](https://github.com/opendatahub-io/opendatahub-operator/blob/main/docs/migration-guides/maas-to-aigateway-3.5.md)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
