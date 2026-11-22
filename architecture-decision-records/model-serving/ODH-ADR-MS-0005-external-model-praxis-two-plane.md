# Open Data Hub - Architecture Decision Record

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-08 |
| Scope          | RHOAI AI gateway — `ai-gateway-controller` (control plane) + `praxis-extproc` (data plane), 3.6 external-model migration |
| Status         | Accepted |
| Authors        | Yossi Ovadia(@yossiovadia) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | RHOAI 3.6 external-model migration ([ai-gateway-controller#10](https://github.com/opendatahub-io/ai-gateway-controller/issues/10)) |
| Other docs:    | ExternalModel/ExternalProvider Reconciler Port Plan v2 (implementation detail, parity inventory, milestones) · IPP → Praxis Migration Plan — RHOAI 3.6 (workstream split) · praxis-ai #540 (overlay contract), #731 (picker policies), #386 (trust boundary) |

Format: [opendatahub-io/architecture-decision-records](https://github.com/opendatahub-io/architecture-decision-records) house template (`ODH-ADR-0000`).

## What

How external-model traffic moves from the `ExternalModel`/`ExternalProvider` CRDs to Praxis routing in 3.6: a single Go controller resolves the CRs and is the **sole writer** of the routing state; Praxis consumes a published, **content-addressed** `routing-overlay.json` envelope read-only. The Envoy plane (HTTPRoute + Istio ServiceEntry/DestinationRule + ExternalName Service) stays as upstream plumbing and the policy-attachment surface — never a second selector.

## Why

* The 3.6 data-plane migration (IPP → Praxis) must not break the frozen user contract: existing `ExternalModel`/`ExternalProvider` objects keep working, **with no auto-migration and no object mutation**.
* `maas-api`/`maas-controller` attach Kuadrant policies to `status.httpRouteName` gated on `status.phase` — that consumer contract must survive unchanged.
* Both planes' details need to be documented and agreed by their owners before implementation continues. This record is that document: decisions already made (with evidence), and the open items that still need a call.

## Goals

* One resolved route set feeds both planes in the same reconcile; no divergence by construction.
* Every published overlay state is independently verifiable (hash-checkable) on both sides of the wire.
* Zero-restart propagation: overlay changes hot-swap in `praxis-extproc` via the existing file watcher.
* Partial applies and torn states are observable, not silent.

## Non-Goals (3.6)

* Praxis-native upstreams (removing the HTTPRoute/SE/DR side-effects) — deferred post-3.6.
* Proportional (weighted) canaries — the picker cannot express them; see How/D5.
* Kafka/org-id metering deltas, guardrails, translation, credential rotation — other workstreams.
* Changing the `ExternalModel`/`ExternalProvider` CRDs themselves — frozen API contract.
* xDS-style config distribution / embeddable composition (praxis core #1076) — must not be precluded; no work now.

## How

Six decisions. "Accepted-in-practice" = code already implements it (evidence linked); "open" = needs the team call listed in Open Questions.

* **D1 — Two-plane authority, single writer** *(accepted-in-practice)*. The controller is the sole writer of routing state; Praxis is a read-only consumer. Post-cutover the overlay is the *only* model→provider selector for new-world traffic; the Envoy plane is plumbing (Host rewrite, TLS) plus the Kuadrant policy-attachment surface. Apply order within a reconcile: provider resources → HTTPRoute → overlay ConfigMap **last** (it is the one artifact whose inconsistency fails at request time, not reload time). *Why:* two live selectors diverge silently; one dynamic-config surface matches the contract the Praxis routing filter was built to consume; keeping the Envoy side-effects preserves `httpRouteName` so `maas-api` needs zero changes.
* **D2 — Backward-compatible types: mirror, don't import; no migration by construction** *(accepted-in-practice)*. The CRD types are a frozen mirror in this repo's `api/` package (not an import of the IPP module), kept in sync by a CI drift-golden test while both controllers coexist. Nothing in this design rewrites existing objects; **no migration path exists** — the no-auto-migration requirement holds by construction. *Why:* importing pins this repo to the module the migration supersedes; the CRD shape is contractually frozen, which is exactly when a copy is cheap and permanent.
* **D3 — The routing-overlay envelope is the wire contract (v1)** *(accepted-in-practice; contract, not code)*. `schema_version: "1.0.0"`; **content-addressed revision** = SHA-256 over the RFC 8785 (JCS)-canonicalized overlay content (`network`, `local_site`, `candidates`, `selection_policy` when present) — provenance (who/when/generation) is deliberately not in the digest. The consumer **recomputes and rejects on mismatch**. `source_generation` is strictly monotonic (advances only on digest change). Credential objects are **references, never bytes**, exactly `{strategy, secretRef{name, namespace, key}}`; the consumer rejects unknown fields inside them. Cross-language byte-compatibility is pinned by **12 golden vectors** — 4 pinned accept digests, 1 legacy accept, 7 reject verdicts — with the same fixtures vendored from praxis-ai (provenance in NOTICE) running in both the Go CI and the Rust CI. *Why:* content addressing turns "is the ConfigMap what we published?" into a hash check; generation chaining makes partial applies observable.
* **D4 — Publish semantics** *(accepted-in-practice)*. The envelope lives in a ConfigMap (`routing-overlay`, key `routing-overlay.json`), server-side-applied by a single field manager; the mount into Praxis must **not** use `subPath` (the `..data` symlink swap is the reload trigger). Publish is idempotent — byte-identical envelopes are not rewritten, `rendered_at` is inherited when the digest is unchanged, so a no-op reconcile cannot churn the ConfigMap or trigger a reload. A baseline whose bytes no longer hash to its declared revision is **refused loudly**, never chained from. *Why:* the data plane hot-swaps on file change; churn is a needless reload, and a tampered baseline laundered into generation N+1 would destroy the audit trail.
* **D5 — Weights in 3.6: uniform only, guarded at render time** *(accepted-in-practice)*. Within a model, all surviving weights must be equal (unset ≡ 1); a non-uniform set **refuses the whole reconcile** (`Ready=False`, `WeightUnsupported` condition, generation does not advance). Weight-0 keeps IPP semantics (candidate omitted = disabled provider). *Why:* the Praxis picker has no weighted-random policy, so 70/30 would silently route 50/50; the guard makes that a loud failure at `kubectl apply` time. The only sanctioned path to proportional weights is a `Weighted` picker upstreamed to Praxis — faking weights by duplicating candidates is forbidden (breaks session-affinity identity). *Surfaced with the control-plane/data-plane owners (2026-09-08) as a **documented capability difference from IPP, not a silent behavior change**: uniform-only for 3.6, proportional deferred behind the `Weighted` picker. Audit of the dogfood cluster (`kind-maas-experimental`) found all 6 ExternalModels on uniform (nil) weights, so no live user is affected; a prod-tenant sweep is recommended before 3.6 GA to confirm.*
* **D6 — Status contract: sole writer, attests only what it can verify** *(accepted-in-practice for the contract; reconciler writes land in M2)*. `Ready=True` requires all of: refs resolved, provider resources applied, HTTPRoute applied, overlay applied (partial-apply contract — a step-N failure leaves the previously distributed envelope in place). The overlay condition is **`OverlayDistributed`** (digest + generation), not "serving": the controller can truthfully attest the ConfigMap was applied with a given digest; serving is a request-time observation Praxis does not yet report, and a status contract must not be able to lie. *Why:* two status writers is the bug class the old stack already has; `Ready` now implies the route actually exists, so policies can never attach against a rejected route.

## Open Questions

| # | Question | Current posture | Leaning |
|---|----------|-----------------|---------|
| O1 | **Credential strategies.** The CRD says `apikey\|sigv4\|oauth2`; the wire contract accepts only `bearer_token`. Until resolved, unrepresentable types render **no credential** (routing still works; injection stays at the provider gateway). | Omit, no wire lie | Widen the Praxis strategy enum |
| O2 | **AITenant contract.** Which field decides praxis-vs-maas mode per tenant, and does the controller watch AITenant or is the flip purely operator-driven? | Not designed | Watch + filter (enables soft-downtime) |
| O3 | **Downtime budget.** Hard ~1–2 min, or a soft-downtime variant for 3.6 (old plane keeps routing until the new plane confirms the overlay loaded)? | Not decided | Soft-downtime for 3.6 |
| O4 | **Model aliases.** One ExternalModel answering to several body-model names: duplicate-identity candidates (producer-side) or an alias-resolution step in the filter? | Unchanged IPP behavior | Producer-side, revisit at CF |

## Alternatives

* **Import the IPP API module instead of mirroring (D2).** Keeps one source of truth, but pins this repo's interface freeze to the module being superseded — a dead pseudo-versioned dependency (plus its transitive gateway-api/istio graph) once IPP freezes. Rejected: the CRD is contractually frozen, which makes the mirror cheap and the drift-golden test sufficient.
* **Drop the Envoy side-effects now; route purely from the overlay (D1).** Smaller surface, but breaks `status.httpRouteName` (maas-api policy attachment) and multi-tenancy handling until praxis-native upstreams exist. Deferred post-3.6 by the migration plan.
* **Fake proportional weights by duplicating candidates (D5).** Expressible in today's picker, but breaks `stable_id`/session-affinity identity and lies in the audit trail. Forbidden.
* **Let the data plane watch the CRs directly (D1/D5).** A second live selector with its own view — the divergence this ADR exists to prevent. Rejected.

## Security and Privacy Considerations

* Credential material never crosses the wire: envelope credential objects are **references** (`secretRef`), resolved at the data plane; the shape is closed (unknown fields rejected).
* The published ConfigMap is hash-verifiable out-of-band (D3/D4): an out-of-band edit is detected and refused, not laundered into the next generation.
* The data-plane mount is read-only; the writer is a single, named SSA field manager (`ai-gateway-controller`).
* O1 (credential strategies) is a live gap until resolved — until then, unrepresentable auth types omit the reference rather than emit a wrong one (no wire lie the injection filter could act on).

## Risks

* **Unknown-cluster references (high):** every `cluster` in the overlay must already exist in the extproc `load_balancer` config or requests fail at runtime. Mitigation: renderer validates against the declared cluster allowlist and refuses on unknown; two-step add-provider flow documented in the install guide.
* **Dual control-plane coexistence (med):** during dogfood, IPP's controller and this one must never both be enabled for one tenant. Mitigation: scripted cutover/adoption order (disable IPP → enable new; SSA adoption of same-named objects, ownerRefs unchanged → no delete/recreate), CI check on the dogfood overlay.
* **Mirror drift (med):** two repos carrying the same API until cutover. Mitigation: drift-golden test in CI while both controllers exist; the mirror is retired after cutover.
* **Weighted canaries requested live (med):** users who need 70/30 get a loud refusal in 3.6. Mitigation: `WeightUnsupported` message points at the sanctioned path (upstreamed `Weighted` picker).

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| ----- | ------------ | ---- | --------- |
| MaaS (reconciler logic owners) | Yossi Ovadia, Noy Itzikowitz | 2026-09-08 | Yes — owns D1–D6 implementation |
| ai-gateway-controller team (runtime, E2E) | Jamie Land | 2026-09-08 | Yes — consumes the frozen surface; reviews O1–O3 |
| Praxis data plane | (praxis-ai maintainers) | 2026-09-08 | Yes — D3 contract, O1 enum, D5 picker |
| maas-api / maas-controller consumers | (MaaS) | 2026-09-08 | No change — `httpRouteName`/`phase` contract preserved |

## References

* Port plan v2 — *ExternalModel/ExternalProvider Reconciler Port (IPP → ai-gateway-controller)* — workspace doc, to be mirrored into this repo's `docs/` at handoff
* [opendatahub-io/architecture-decision-records](https://github.com/opendatahub-io/architecture-decision-records) — org ADR repo and house template (`ODH-ADR-0000`)
* praxis-ai: [tests/fixtures/overlay-contract/v1](https://github.com/praxis-proxy/ai/tree/main/tests/fixtures/overlay-contract/v1) (golden vector source, `1ef8a53e`), #540 envelope contract, #731 picker policies, #386 trust boundary
* [e2e evidence — routing-overlay hot-swap harness](https://github.com/yossiovadia/ai-gateway-controller/tree/feat/external-model-reconciler/test/overlay-e2e)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
| (pending — control-plane/data-plane/MaaS walkthrough) | | |
