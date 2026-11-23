# Replace RHCL with PPE on the MaaS inference path

| Date | 2026-11-23 |
| :---- | :---- |
| Scope | Models-as-a-Service (MaaS), AI Gateway — inference authn/authz and rate limiting |
| Status | Draft (local; not filed as a numbered ODH ADR yet) |
| Authors | jland |
| Tickets | RHAIRFE-3914 (broader Praxis-as-gateway), RHOAIENG-104076 / 104077 |
| Companion | [external-model-praxis-routing-adr-11-23.md](./external-model-praxis-routing-adr-11-23.md), [ODH-ADR-MS-0005](https://github.com/opendatahub-io/architecture-decision-records/pull/165) (IPP → Praxis ExtProc) |
| Other docs | [praxis-proxy/policy#195 — Support identified subset of AuthPolicy capabilities](https://github.com/praxis-proxy/policy/issues/195) (canonical PPE feature subset for MaaS AuthPolicy parity); [policy#348 — MaaS AuthPolicy demo](https://github.com/praxis-proxy/policy/issues/348); [Full Praxis notes](https://docs.google.com/document/d/1Brgpq2ow4Kjq_6O1NOecPfM3yCuSUk3FXJbl10lJI9o/edit?tab=t.q8npwz8ylxm1) |
| Sources | [PPE adoption draft](https://github.com/jland-redhat/ai-gateway-controller/blob/early-praxis-policy-engine-ppe/docs/proposals/praxis-policy-engine-adoption.md), [analysis gist](https://gist.github.com/jland-redhat/df169c033507aa8a69be7ddfcdc4e3ae), [POC gist](https://gist.github.com/jland-redhat/eb2b5e7b1abb5829d5bb006b1bc6c5b4) |

## What

On the **inference request path**, replace RHCL-injected Authorino + Limitador
(wasm shim calling maas-api) with the **Praxis Policy Engine (PPE)** embedded
in a single Praxis deployment per tenant. Policy becomes a PPE document that
host consumes in-process — rendered first as mounted YAML, later from Praxis
`Policy` / AI Gateway governance CRDs — not Kuadrant `AuthPolicy` /
`RateLimitPolicy` / `TokenRateLimitPolicy` on that path.

| Acronym | Meaning here |
| :---- | :---- |
| **RHCL** | Red Hat Connectivity Link — productized Kuadrant (Authorino + Limitador + wasm filter) |
| **PPE** | Praxis Policy Engine (`praxis-proxy/policy`) — Rust library embedded in Praxis |

This ADR is deliberately narrower than RHAIRFE-3914 (Praxis as the public
gateway) and narrower than ODH-ADR-MS-0005 (payload-processing → Praxis ExtProc).

## Why

- Biggest win: this removes the external dependency on the RHCL operator. Setup
  complexity drops massively — the RHCL operator install and its wiring are one
  of our biggest complaints today.
- RHCL inserts an out-of-process wasm shim between two Praxis hops on the
  inference path.
- Connectivity Link supportability and Kuadrant cardinality limits are a product risk.
- PPE is the policy library Praxis already embeds; migrating inference policy onto
  it is the stepping stone toward a simpler AI Gateway dataplane.
- POC (Nov 2026) already proved mint / models / 200 / 401 / 429 with PPE in the chain.
- The thesis is not only "PPE enforces policy" but **one Praxis deployment owns the
  path** — pre-auth, identity, rate limit, authz, and payload processing.

## Goals

- Preserve observable behavior for covered inference scenarios: API-key auth,
  subscription selection, rate limiting, `x-maas-*` injection, Authorization strip,
  fail-closed denials, streaming parity.
- One Praxis deployment per PPE tenant owns the full inference filter chain; no
  RHCL wasm shim on that path.
- Inference policy is a PPE document rendered by a controller (ConfigMap first
  cut; Praxis `Policy` / AI Gateway governance CRDs as they land) — not live
  Kuadrant CRs.
- Per-tenant opt-in and rollback via annotation (default remains RHCL).
- Authorino trust-boundary parity before RHCL leaves a surface (see Security).
- Do not remove RHCL from a GA surface until every auth/limit method that surface
  exposes exists in PPE (or an accepted host-side equivalent).

## Non-Goals

- Removing Envoy / making Praxis the public listener, or migrating the Gateway
  load-balancer URL off Istio (RHAIRFE-3914 / Full Praxis notes — users hit a
  URL we do not control).
- Moving consumer-portal OIDC session / anonymous AuthPolicies off RHCL
  (browser OIDC is [policy#129](https://github.com/praxis-proxy/policy/issues/129);
  portal attachment stays on RHCL for now).
- Absorbing maas-api in the first cut (it stays as the API-key / subscription
  HTTP directory initially).
- Claiming full Kuadrant CR API compatibility for customized gateways.
- Removing IPP / payload-processing swap (ODH-ADR-MS-0005).
- ExternalModel HTTPRoute → Proxy routing, SE/DR egress, CONNECT proxy
  ([companion ADR](./external-model-praxis-routing-adr-11-23.md)).
- Guardrails migration; KServe routing / weighted `backendRefs` / EPP parity /
  Istio-metric dashboards / ODH Model Controller TLS EnvoyFilter / Dashboard
  gateway refs (Full Praxis notes + KServe track).
- Metering / Price Tag admission productization (Tokenomics path) — except
  where "report quota usage" is called out below as a PPE/QoS gap.
- Changing InferencePool / EPP (optional, stays separate).

## How

```
Today:  Praxis → RHCL wasm shim (Authorino/Limitador→maas-api) → Praxis → backend
Target: Praxis (pre-auth → PPE identity/ratelimit/authz → payload) → backend
```

### Dataplane shape (single Praxis deployment)

For tenants on PPE, collapse today's **Praxis → wasm shim → Praxis** sandwich
into **one** Praxis deployment whose filter chain runs:

1. Pre-auth BBR / model extraction — must stay **before** PPE
2. PPE (`policy` filter): identity → rate limit → authz / assertions
3. Post-auth payload chain

No RHCL wasm shim and no priority dance against Kuadrant on that path. Pre-auth
work (body → model name) stays ahead of PPE identity / ratelimit by ordering
inside one chain via PPE hook phases (`ratelimit.pre_identity` before identity;
authz after).

- Fail-closed policy enforcement on the inference path.
- Config hash on the Deployment rolls the pod after both filter-chain and policy
  changes (hash must include the merged artifact).

### Policy surface (controller-rendered PPE document)

PPE the library still loads one policy document in-process — it is not itself a
Kubernetes API server. The **durable control-plane surface** is CRDs (Praxis
`Proxy` / `Policy`, and the AI Gateway governance facade that lowers onto them);
a controller owns rendering that into the PPE document the Praxis deployment
runs. Kuadrant `AuthPolicy` / `RateLimitPolicy` / `TokenRateLimitPolicy` are
**not** the runtime policy surface for PPE tenants.

For the first cut — before those CRDs are the default path for MaaS — auth /
subscription / rate-limit intent is rendered into a **`policy.yaml`** mounted
on the per-tenant Praxis deployment (alongside the server / filter-chain
config). Same end artifact PPE always consumes; only the writer moves from
ConfigMap render to CRD-backed controller later.

**Relation to `praxis-proxy/gateway-controller`:** ODH / MaaS controllers remain
the product owners of tenant intent (`AITenant`, subscriptions, auth policies).
`gateway-controller` owns the Praxis Core CRDs (`Proxy`, `Policy`, …) and the
rendered proxy config. This ADR does not rename or replace those CRDs; it
decides that inference auth/RL leave Kuadrant and land as PPE documents those
controllers (eventually) lower into. Clarifying the exact CR ownership split
with the AI Gateway facade is an open product decision below.

**Recommended renderer ownership (migration):** maas-controller keeps a
Kuadrant-shaped intermediate and runs **authpolicy-transpiler** → PPE YAML.
Migration safety: a diff of transpile output is the parity audit — backed by
the reference AuthPolicy in [#195](https://github.com/praxis-proxy/policy/issues/195)
and the MaaS demo in [#348](https://github.com/praxis-proxy/policy/issues/348).
Once `Policy` / AI Gateway governance are ready, the same MaaS intent lowers
through those CRDs instead of hand-mounting YAML.

Alternatives considered: maas-controller renders PPE YAML directly; or
ai-gateway-controller renders from published MaaS data (inverts ownership).

**Reload:** prefer hot-reload of the PPE document on ConfigMap / CR change
([policy#303](https://github.com/praxis-proxy/policy/issues/303) — honor plugin
lifecycle on configuration reload). Until that is solid, the existing
config-sha256 Deployment roll remains the safe path. Secrets may still force
restart (PPE refuses secret mutation on reload today).

### Coexistence and cut-over

Tenants select the inference policy engine with an annotation, defaulting to RHCL.
PPE is opt-in per tenant. Rollback is flipping the annotation; Kuadrant and RHCL
remain installed during migration.

```
maas.opendatahub.io/policy-engine: rhcl | ppe
```

Default when absent: `rhcl`. Independent of `payload-processing-type` (a tenant
can run Praxis payload processing with RHCL policy, or Praxis with PPE).

- `ai-gateway-controller` branches the tenant dataplane render on `policy-engine`.
- `maas-controller` keeps the Kuadrant renderer for `rhcl` tenants; adds PPE YAML
  renderer for `ppe` tenants.
- Rollout: dev tenant → EA → GA. Shadow mode (both enforcing) is a migration tool,
  not the end state.
- Consumer portal and any tenant that still needs RHCL keep Kuadrant installed.

### Decision rule for leaving RHCL on a surface

```
surface S may set policy-engine=ppe for GA
  iff every auth/limit method S exposes is covered in "Upstream requirements" below
  AND trust-boundary parity tests pass for S
```

Partial migration (e.g. API-key only on PPE; TokenReview / TRLP on RHCL) is
allowed only via explicit product carve-out — not by silent gap.

## Upstream / product requirements (gate RHCL retirement)

POC carve-outs are temporary; they are not GA exits. Host-only workarounds that
permanently drop a 3.6 surface are rejected unless product explicitly deprecates
that surface. Matrix below is the MaaS checklist this ADR owns (Auth + Rate
Limiting / QoS). KServe and Istio→Praxis gateway items stay in Non-Goals.

### Auth

| Capability | Today (RHCL) | PPE / product status | Tracking |
| :---- | :---- | :---- | :---- |
| API key | Authorino + maas-api validate | ✅ Covered (`identity/api-key`, HTTP directory) | [policy#108](https://github.com/praxis-proxy/policy/issues/108) (closed epic) |
| OIDC JWT (inference) | Authorino OIDC / JWKS; `azp` / group checks | ✅ `identity/jwt` covers bearer JWT. **Not** browser/Authorization-Code OIDC | Inference JWT ≠ [policy#129](https://github.com/praxis-proxy/policy/issues/129) (portal; out of scope) |
| JWT audience | Authorino `audiences` optional | PPE refuses omitted/empty `audiences`; hatch `skip_audience_validation: true` | Migration contract (see below); [#16](https://github.com/praxis-proxy/policy/issues/16) fix |
| TokenReview + SAR | Authorino `kubernetes` + optional SAR | ❌ Missing; third method in overlay 0032 | [policy#213](https://github.com/praxis-proxy/policy/issues/213), [#214](https://github.com/praxis-proxy/policy/issues/214) |
| Metadata callouts | Authorino HTTP → maas-api select/validate | Needed if maas-api stays a live PIP | [policy#184](https://github.com/praxis-proxy/policy/issues/184) |
| Rego / OPA authz | maas-controller generates **Rego v0** today | PPE expects Rego **v1**; transpiler / generator must migrate | **Control-plane work** (no dedicated upstream issue found) |
| Equivalence demo | Live AuthPolicy on RHCL | Sample PPE policy proving parity with existing AuthPolicy | Reference: [#195](https://github.com/praxis-proxy/policy/issues/195); demo: [#348](https://github.com/praxis-proxy/policy/issues/348) |
| Hot reload on config change | Kuadrant reconciles continuously | Prefer reload without pod restart; honor plugin lifecycle | [policy#303](https://github.com/praxis-proxy/policy/issues/303); Deployment roll is fallback |

Until TokenReview/SAR ship: SA-token inference clients stay on `policy-engine: rhcl`.

### Rate limiting and QoS

| Capability | Today (RHCL) | PPE / product status | Tracking |
| :---- | :---- | :---- | :---- |
| Request-count rate limits | Limitador via RHCL wasm | Plugin is request-count with **embedded in-process** Limitador today | Graduation: [policy#308](https://github.com/praxis-proxy/policy/issues/308); epic [#143](https://github.com/praxis-proxy/policy/issues/143) |
| Token rate limiting (TRLP parity) | Kuadrant TokenRateLimitPolicy | Must cover existing token-RL capabilities on PPE path | [policy#308](https://github.com/praxis-proxy/policy/issues/308) |
| Header-scoped (per-model) limits | TRLP on HTTPRoute / `X-Gateway-Model-Name` | `http:` routes cannot match headers yet | [policy#352](https://github.com/praxis-proxy/policy/issues/352) (also under [#195](https://github.com/praxis-proxy/policy/issues/195)) |
| Shared counters / Redis | Limitador + Redis, global | Host-injected store (Redis/Valkey); Redis client alignment + **migration plan** from RHCL Redis | [policy#127](https://github.com/praxis-proxy/policy/issues/127), [#203](https://github.com/praxis-proxy/policy/issues/203) (open question on Redis migration) |
| Inference priority / InferenceObjective | Header injection for QoS / scheduler hint | Must survive RHCL removal (PPE assertions or adjacent Praxis filter) | **Product / host** — confirm owner; not a filed PPE issue |
| Report quota usage | New requirement | Emit / report usage for quota accounting when PPE owns limits | **Unfiled** — needs an upstream or product issue |

Until shared counters ship: PPE tenants single-replica **or** request/token RL stay on RHCL. Experimental `quota/limitador` remote path: verify API compatibility for budget carry-over separately from request RL.

### Verified (not a missing capability — migration contract)

| Topic | Finding | Action |
| :---- | :---- | :---- |
| JWT `audiences` | PPE refuses omitted/empty `audiences` at load; hatch is `skip_audience_validation: true` ([changelog](https://github.com/praxis-proxy/policy/blob/main/CHANGELOG.md) / [#16](https://github.com/praxis-proxy/policy/issues/16)). Authorino made audiences optional; MaaS often omits it. | Transpiler always sets audiences (or documents intentional skip). Parity suite must **not** reproduce Authorino's optional-audience hole. Tightening is OK if migration-documented. |

### Not tracked upstream (need issues or product decisions)

| Gap | Why it matters | Suggested home | Notes |
| :---- | :---- | :---- | :---- |
| **Limitador gRPC client for request rate limits** (reuse standalone Limitador as-is) | #127 / #203 assume an **embedded** store the host injects — not a remote gRPC client to the existing RHCL Limitador Deployment. | `praxis-proxy/policy` | Optional if Valkey/Redis via #127 is the accepted shared-counter answer. |
| **FIPS-compatible `policy-engine` build** | Stock FIPS praxis-extproc **excludes** `policy-engine` by design ([docs/fips.md](https://github.com/opendatahub-io/praxis-extproc/blob/main/docs/fips.md)) | Product / `opendatahub-io/praxis-extproc` (and possibly praxis-proxy/ai) | POC used a custom non-FIPS image — not a product exit. |
| **Report quota usage** | New QoS/accounting requirement when PPE owns limits | `praxis-proxy/policy` or Tokenomics | Unfiled. |
| **Rego v0 → v1** in maas-controller / transpiler | Generated AuthPolicy Rego is v0; PPE wants v1 | MaaS control plane (+ PPE docs if semantics differ) | Unfiled as a dedicated issue. |
| **InferenceObjective / priority header** | QoS hint today must not disappear with RHCL | Host filter or PPE assertions | Confirm current RHCL/Authorino owner before filing. |

### Explicitly not required for first PPE GA of API-key inference

- OIDC Authorization Code / browser flows (consumer portal stays on RHCL; [#129](https://github.com/praxis-proxy/policy/issues/129)).
- Opaque OAuth2 token introspection (MaaS surfaces are JWT or API key).
- mTLS / HMAC Authorino identities (unused by MaaS today).
- Response-body policy in PPE (stays in payload filters).
- Secret hot-reload without restart (Deployment roll OK for first cut; [#303](https://github.com/praxis-proxy/policy/issues/303) still desired for policy document reload).

## Security and Privacy Considerations

RHCL/Authorino is not only a feature provider — it is the **trust boundary**
between untrusted clients and maas-api / backends. Before a surface stops using
RHCL, PPE (and its host) must reproduce that boundary.

Today (overlay 0032), gateway-enforced auth is: MaaS API key → External OIDC JWT
(`azp == clientId`, group-name safety) → Kubernetes TokenReview. Authorino then
injects `X-MaaS-*` (and rejects client-supplied values), strips `Authorization`
before the backend, and calls maas-api internals that have **no caller auth** —
only NetworkPolicy (`maas-authorino-allow` → Authorino pods).

maas-api trusts `X-MaaS-*` and does not re-validate JWTs. Widening the
NetworkPolicy to tenant-facing gateway pods without a new caller auth story
turns the key-validation endpoint into a hash-lookup oracle one hop from the
public listener.

| Requirement | Pass criterion |
| :---- | :---- |
| Identity priority | Same first-match order: API key → OIDC JWT → TokenReview |
| Header injection | `x-maas-*` (or agreed successors) set from resolved identity only |
| Header denial | Client-supplied `x-maas-*` rejected / stripped before trust |
| Authorization strip | Bearer not forwarded to inference backend |
| Fail-closed | Auth / directory / PPE failure → deny (not allow) |
| maas-api callbacks | Directory / validate / select callers authenticated or netpol-equivalent that does **not** expose the oracle to the public listener |
| Audience / azp | PPE path must not weaken binding; tightening vs Authorino's optional `audiences` is allowed if migration-documented |

Keep directory calls **pod → maas-api** (cluster DNS), never back through the
public inference listener. Replace `maas-authorino-allow` with a PPE-era trust
design **before** Authorino is removed for that tenant (preferred: authenticated
m2m; or narrowly scoped NetworkPolicy to the Praxis SA **plus** proof it cannot
be a confused deputy from the public listener).

## Risks and Considerations

- Operators toggling `policy-engine` without waiting for ConfigMap roll + pod
  digest → "deployed but old policy."
- Stock FIPS image cannot run the `policy` filter; Limitador-in-PPE needs
  `panic=unwind` in the host image.
- Treating NetworkPolicy widen-to-Praxis as "good enough" without caller auth.
- Shared RL via #127 still leaves multi-replica correctness gated until shipped;
  single-replica PPE is a temporary EA posture, not GA for multi-replica gateways.
- Reproducing Authorino's optional-audience hole in hand-written PPE policy
  (do **not**).
- MaaS (and related) test suites have **hard assumptions on RHCL** / Kuadrant
  objects. Refactoring e2e and unit fixtures to be policy-engine-agnostic can
  start before PPE implementation is complete — otherwise cut-over is blocked
  on test debt.
- DSC / odh-operator still install and warn about RHCL; product messaging and
  operator conditions must track coexistence and eventual removal.

## Open product decisions

1. Who renders the PPE document long-term, and when the writer moves from ConfigMap to `Policy` / AI Gateway CRDs (recommend authpolicy-transpiler path for migration). Clarify ODH AI Gateway controller vs `gateway-controller` CR ownership.
2. Shared rate-limit store: remote Limitador gRPC (unfiled) vs Valkey/Redis via #127 vs single-replica for EA. Explicit **Redis migration plan** from RHCL Limitador Redis ([#203](https://github.com/praxis-proxy/policy/issues/203)).
3. Long-term maas-api: HTTP directory vs absorbed vs PIP ([#184](https://github.com/praxis-proxy/policy/issues/184)).
4. FIPS: ship non-FIPS PPE path, or block FIPS tenants on RHCL until a FIPS policy build exists.
5. Whether per-model limits may move to LLM-entity routes as an accepted equivalent to header-matched TRLP (fallback if #352 slips).
6. **Migration vs parallel support** of existing RHCL setups (annotation coexistence is the proposed answer; full-fleet flip is rejected).
7. Where InferenceObjective / inference-priority header injection lives after RHCL.
8. Who owns "report quota usage" and whether it blocks GA for subscription token limits.

## Alternatives

| Option | Tradeoff |
| :---- | :---- |
| Keep RHCL | Stable today; keeps Praxis → wasm → Praxis; blocks simpler SKU path. |
| Collapse Praxis hops but keep Authorino | Fewer hops, still out-of-process auth and Kuadrant CRs. |
| Keep two Praxis deployments with PPE between them | Smallest PPE diff; fails the "one deployment" requirement. |
| Cluster-wide policy-engine flag only | No per-tenant rollback. |
| **Adopt PPE + single Praxis deployment + ConfigMap policy + per-tenant annotation (this ADR)** | In-process policy; depends on upstream gaps above. |

## Stakeholder Impacts

| Group | Impacted? |
| :---- | :---- |
| AI Gateway (`ai-gateway-controller`) | Single Praxis deployment render; drop RHCL wasm for PPE tenants; `policy-engine` annotation; mount `policy.yaml`; relation to `gateway-controller` CRs. |
| Models-as-a-Service (`maas-controller`) | PPE YAML renderer / transpiler (incl. Rego v1); keep Kuadrant renderer for RHCL; maas-api as directory; test suite de-RHCL. |
| AI Gateway (Praxis / `gateway-controller`) | Embed PPE; `Policy` CR path; FIPS feature decision; host transport + backend stores. |
| Platform / RHCL / DSC / odh-operator | Remains for RHCL tenants and consumer portal; warnings and install path track coexistence → removal. |
| Dashboard / users | No change to mint / call UX for API-key inference once parity holds. |
| QE / MaaS testing | Hard RHCL assumptions must be refactored; can start before PPE lands. |

## References

- [praxis-proxy/policy](https://github.com/praxis-proxy/policy)
- POC branch `early-praxis-policy-engine-ppe` (ai-gateway-controller + praxis-extproc)
- [praxis-extproc FIPS docs](https://github.com/opendatahub-io/praxis-extproc/blob/main/docs/fips.md)
- Kuadrant-parity subset: [policy#195](https://github.com/praxis-proxy/policy/issues/195); demo [#348](https://github.com/praxis-proxy/policy/issues/348)
- [Full Praxis notes](https://docs.google.com/document/d/1Brgpq2ow4Kjq_6O1NOecPfM3yCuSUk3FXJbl10lJI9o/edit?tab=t.q8npwz8ylxm1)
