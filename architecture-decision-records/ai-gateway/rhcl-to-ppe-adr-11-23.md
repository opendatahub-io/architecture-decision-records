# Replace RHCL with PPE on the MaaS inference path

| Date | 2026-11-23 |
| :---- | :---- |
| Scope | Models-as-a-Service (MaaS), AI Gateway — inference authn/authz and rate limiting |
| Status | Draft (local; not filed as a numbered ODH ADR yet) |
| Authors | jland |
| Tickets | RHAIRFE-3914 (broader Praxis-as-gateway), RHOAIENG-104076 / 104077 |
| Companion | [ADR: Route External Models through Praxis (PR 191)](https://github.com/opendatahub-io/architecture-decision-records/pull/191), [ODH-ADR-MS-0005](https://github.com/opendatahub-io/architecture-decision-records/pull/165) (ExternalModel / ExternalProvider two-plane architecture) |
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
gateway) and narrower than ODH-ADR-MS-0005 (ExternalModel two-plane split).

## Why

- Biggest win: this removes the external dependency on the RHCL operator. Setup
  complexity drops massively — the RHCL operator install and its wiring are one
  of our biggest complaints today.
- RHCL's wasm shim runs in-process in Envoy, but its callouts to Authorino
  and Limitador are out-of-process — extra network hops between two Praxis
  hops on the inference path.
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
- Per-tenant selection and rollback via annotation; absent annotation means
  `ppe` (the end state). Existing tenants are backfilled to `rhcl` during
  migration so the default flip never moves a tenant implicitly.
- Authorino trust-boundary parity before RHCL leaves a surface (see Security).
- Do not remove RHCL from a GA surface until every auth/limit method that surface
  exposes exists in PPE (or an accepted host-side equivalent).

## Non-Goals

- Removing Envoy / making Praxis the public listener, or migrating the Gateway
  load-balancer URL off Istio (RHAIRFE-3914 / Full Praxis notes — users hit a
  URL we do not control).
- Absorbing maas-api in the first cut (it stays as the API-key / subscription
  HTTP directory initially).
- Claiming full Kuadrant CR API compatibility for customized gateways.
- Removing IPP / payload-processing swap (decided separately; the ExtProc
  migration is its own track).
- ExternalModel HTTPRoute → Proxy routing, SE/DR egress, CONNECT proxy
  ([companion ADR, PR 191](https://github.com/opendatahub-io/architecture-decision-records/pull/191)).
- Guardrails migration; KServe routing / weighted `backendRefs` / EPP parity /
  Istio-metric dashboards / ODH Model Controller TLS EnvoyFilter / Dashboard
  gateway refs (Full Praxis notes + KServe track).
- Metering / Price Tag admission productization (Tokenomics path) — except
  where "report quota usage" is called out below as a PPE/QoS gap.
- Changing InferencePool / EPP (optional, stays separate).
- Migrating MaaS to PPE-minted tokens. MaaS API already mints the keys /
  tokens in use; PPE token minting ([policy#380](https://github.com/praxis-proxy/policy/issues/380))
  is a future option only — see Future plans. Any move to PPE-minted tokens
  needs its own migration plan, since live credentials would have to carry
  over without breaking clients.

## How

```
Today:  Pre Processing (Praxis) → RHCL wasm shim (RHLC) → Post-Processing (Praxis) → backend
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
- Praxis must **hot-reload the PPE document on ConfigMap change** — required, not
  optional. Policy changes take effect without a pod roll; RHCL/Kuadrant
  reconcile continuously today and PPE cannot regress that.
- **Credential injection** (provider credential injection for external models)
  is planned in PPE ([policy#380](https://github.com/praxis-proxy/policy/issues/380))
  and is covered in the [companion ExternalModel ADR (PR 191)](https://github.com/opendatahub-io/architecture-decision-records/pull/191),
  not here.

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

**Renderer ownership (migration):** maas-controller renders the PPE document
into the **ConfigMap** (or whatever the underlying mounted artifact is)
directly, until the `Policy` / AI Gateway governance CRDs are ready; once they
are, the same MaaS intent lowers through those CRDs instead of hand-mounting
YAML. There is no transpiler in this path — the
[authpolicy-transpiler demo](https://github.com/praxis-proxy/demos/tree/main/demos/authpolicy-transpiler)
is not actively maintained, and migration is explicitly out of scope from the
Praxis/PPE side: each client owns it. For MaaS that means the maas-controller
code that renders Kuadrant CRs today grows the PPE rendering path — a
control-plane change, not a data migration customers run. Parity is audited
against the reference AuthPolicy in [#195](https://github.com/praxis-proxy/policy/issues/195)
and the MaaS demo in [#348](https://github.com/praxis-proxy/policy/issues/348).

Alternatives considered: ai-gateway-controller renders from published MaaS
data (inverts ownership).

**Reload:** Praxis must hot-reload the PPE document on ConfigMap / CR change
([policy#303](https://github.com/praxis-proxy/policy/issues/303) — honor plugin
lifecycle on configuration reload). This is a hard requirement from Praxis, not
a preference: policy and filter-chain changes must take effect without a pod
roll, matching Kuadrant's continuous reconcile. A config-hash Deployment roll
remains only as a stopgap for secret changes, which PPE refuses to mutate on
reload today; closing that gap is upstream work to track.

### Coexistence and cut-over

Tenants select the inference policy engine with an annotation on the
**`AITenant` object**. Rollback is flipping the annotation; Kuadrant and RHCL
remain installed during migration. Placing it on `AITenant` follows the pattern
already used for Praxis / IPP payload-processing selection — tenant-scoped, one
place to look, read by both `ai-gateway-controller` and `maas-controller`.

```
maas.opendatahub.io/policy-engine: rhcl | ppe
```

**Default when the annotation is absent: `ppe`.** The end state is no
annotation at all — every tenant on PPE. **RHCL is not supported on the
inference path in 3.7**; the `rhcl` annotation value and the backfill below
are a migration technique for upgrades from existing installs, used only
where a tenant cannot move to PPE at upgrade time — not a supported
long-term configuration. (The consumer-portal RHCL dependency is a separate
surface.)

Praxis ships **with RHOAI** — it is not an external dependency — so the
absent-means-`ppe` default is safe: the capabilities in "Upstream / product
requirements" below ship in the same release, and the default goes live only
when they do. The annotation exists for cut-over, not as a capability
selector.

To get there without silently
flipping existing tenants:

- **Migration backfill:** before the absent-means-`ppe` default ships,
  `ai-gateway-controller` writes `maas.opendatahub.io/policy-engine: rhcl`
  onto every `AITenant` that lacks the annotation — as a **continuous
  reconcile**, not a one-shot scan, so tenants created after the backfill
  starts and before the default ships are also covered. Every `AITenant`
  present when the default takes effect carries an explicit annotation;
  existing tenants keep RHCL until they are explicitly migrated.
- **Per-tenant cut-over:** flip the annotation to `ppe`, or remove it entirely
  once absent-means-`ppe` is live.
- **End state:** the annotation is gone; absent means `ppe`.

Independent of `payload-processing-type` (a tenant
can run Praxis payload processing with RHCL policy, or Praxis with PPE).

- `ai-gateway-controller` branches the tenant dataplane render on `policy-engine`.
- `maas-controller` keeps the Kuadrant renderer for `rhcl` tenants; adds PPE YAML
  renderer for `ppe` tenants.
- Rollout: dev tenant → EA → GA. Existing Kuadrant `AuthPolicy` / `TRLP`
  objects remain in place until the new Praxis policy has been fully applied
  for the tenant; they are removed only after PPE enforcement is confirmed.
  Which engine enforces during that cut-over window is
  [open question 9](#open-product-decisions).
- Consumer portal and any tenant that still needs RHCL keep Kuadrant installed.
- **Documentation:** since RHCL is no longer a dependency of the MaaS
  inference path, the release must document **optional removal of the RHCL
  operator** for clusters that no longer need it (including the consumer-portal
  caveat above and the DSC / odh-operator install implications in Risks).

## Upstream / product requirements (gate RHCL retirement)

POC carve-outs are temporary; they are not GA exits. Host-only workarounds that
permanently drop a 3.6 surface are rejected unless product explicitly deprecates
that surface. Partial migration (e.g. API-key only on PPE; TokenReview / TRLP
on RHCL) is allowed only via explicit product carve-out — not by silent gap.
Matrix below is the MaaS checklist this ADR owns (Auth + Rate
Limiting / QoS). KServe and Istio→Praxis gateway items stay in Non-Goals.

### Auth

| Capability | Today (RHCL) | PPE / product status | Tracking |
| :---- | :---- | :---- | :---- |
| API key | Authorino + maas-api validate | ✅ Covered (`identity/api-key`, HTTP directory) | [policy#108](https://github.com/praxis-proxy/policy/issues/108) (closed epic) |
| OIDC JWT (inference) | Authorino OIDC / JWKS; `azp == clientId`, group-name safety checks | ⚠️ Partial. `identity/jwt` verifies signature / JWKS / audiences and maps claims into attributes; `azp` and group checks are then `require(...)` expressions over mapped claims — expressible, but MaaS's exact `azp == clientId` and group-name-safety rules are **not yet proven** in PPE and must be part of the parity suite. Key minting flow is consumer-portal OIDC and is out of scope here. **Not** browser/Authorization-Code OIDC | Inference JWT ≠ [policy#129](https://github.com/praxis-proxy/policy/issues/129) (portal; out of scope); parity: [#195](https://github.com/praxis-proxy/policy/issues/195), [#348](https://github.com/praxis-proxy/policy/issues/348) |
| JWT audience | Authorino `audiences` optional | PPE refuses omitted/empty `audiences`. The `skip_audience_validation: true` hatch is **not permitted** on the MaaS path — with it, a token minted for a different audience passes issuer, `azp`, and group checks | Migration contract (see below); [#16](https://github.com/praxis-proxy/policy/issues/16) fix |
| TokenReview + SAR | Authorino `kubernetes` + optional SAR | ⚠️ Partial. `identity/kubernetes-token-review` available in `main` and planned for PPE 0.9.0 (due to Dec 18th); SAR is an active WIP but also safely targeting 0.9.0. | [policy#213](https://github.com/praxis-proxy/policy/issues/213), [#214](https://github.com/praxis-proxy/policy/issues/214) |
| Metadata callouts | Authorino HTTP → maas-api select/validate | Needed if maas-api stays a live PIP | [policy#184](https://github.com/praxis-proxy/policy/issues/184) |
| Rego / OPA authz | maas-controller generates **Rego v0** today | PPE expects Rego **v1**; transpiler / generator must migrate | **Control-plane work** (no dedicated upstream issue found) |
| Equivalence demo | Live AuthPolicy on RHCL | Sample PPE policy proving parity with existing AuthPolicy | Reference: [#195](https://github.com/praxis-proxy/policy/issues/195); demo: [#348](https://github.com/praxis-proxy/policy/issues/348) |
| Hot reload on config change | Kuadrant reconciles continuously | Required: policy / filter-chain changes apply without a pod roll; Deployment roll is a fallback **for secret changes only** | [policy#303](https://github.com/praxis-proxy/policy/issues/303) |

TokenReview + SAR is **required, period** — PPE does not ship as the inference
default without it. No shipping with missing requirements.

### Rate limiting and QoS

| Capability | Today (RHCL) | PPE / product status | Tracking |
| :---- | :---- | :---- | :---- |
| Request-count rate limits | Limitador via RHCL wasm | Plugin is request-count with **embedded in-process** Limitador today | Graduation: [policy#308](https://github.com/praxis-proxy/policy/issues/308); epic [#143](https://github.com/praxis-proxy/policy/issues/143) |
| Token rate limiting (TRLP parity) | Kuadrant TokenRateLimitPolicy | Must cover existing token-RL capabilities on PPE path | [policy#308](https://github.com/praxis-proxy/policy/issues/308) |
| Header-scoped (per-model) limits | TRLP on HTTPRoute / `X-Gateway-Model-Name` | `http:` routes cannot match headers yet | [policy#352](https://github.com/praxis-proxy/policy/issues/352) (also under [#195](https://github.com/praxis-proxy/policy/issues/195)) |
| Shared counters / Redis | Limitador + Redis, global | **Required, not optional.** In-process counters do not survive a pod restart (policy roll, crash, node drain): the replacement pod starts with a fresh budget and a tenant can exceed its limit within the same window. Single-replica solves concurrency, **not** restart continuity. PPE must be able to **connect to a Redis instance** — either the same instance RHCL's Limitador used (no counter loss, no migration) or a new one, in which case **Praxis must supply the migration plan** for carrying counters over | [policy#127](https://github.com/praxis-proxy/policy/issues/127), [#203](https://github.com/praxis-proxy/policy/issues/203) (open question on Redis migration) |
| Inference priority / InferenceObjective | Header injection for QoS / scheduler hint | Must survive RHCL removal (PPE assertions or adjacent Praxis filter) | **Product / host** — confirm owner; not a filed PPE issue |
| Report quota usage | New requirement | Emit / report usage for quota accounting when PPE owns limits | **Unfiled** — needs an upstream or product issue |

PPE does not ship as the inference default without **both** auth and rate
limiting complete — including the shared counter store. There is no
partial ship where one lands and the other stays on RHCL. Experimental
`quota/limitador` remote path: verify API compatibility for budget carry-over
separately from request RL.

### Verified (not a missing capability — migration contract)

| Topic | Finding | Action |
| :---- | :---- | :---- |
| JWT `audiences` | PPE refuses omitted/empty `audiences` at load ([changelog](https://github.com/praxis-proxy/policy/blob/main/CHANGELOG.md) / [#16](https://github.com/praxis-proxy/policy/issues/16)). Authorino made audiences optional; MaaS often omits it. PPE offers `skip_audience_validation: true` as a hatch. | Transpiler **always sets audiences** (the MaaS inference audience). `skip_audience_validation` is **not used** on the MaaS path — no exceptions. Parity suite must **not** reproduce Authorino's optional-audience hole. Tightening is OK if migration-documented. |

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
| Audience / azp | PPE path must not weaken binding. `skip_audience_validation` is not used; the inference audience is always set. Tightening vs Authorino's optional `audiences` is allowed if migration-documented |

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
- Shared RL via #127 gates the whole PPE default: if the durable counter store
  slips, PPE does not ship — there is no single-replica or auth-only partial
  ship.
- Reproducing Authorino's optional-audience hole in hand-written PPE policy
  (do **not**).
- MaaS (and related) test suites have **hard assumptions on RHCL** / Kuadrant
  objects. Refactoring e2e and unit fixtures to be policy-engine-agnostic can
  start before PPE implementation is complete — otherwise cut-over is blocked
  on test debt.
- DSC / odh-operator still install and warn about RHCL; product messaging and
  operator conditions must track coexistence and eventual removal.

<a id="open-product-decisions"></a>

## Open product decisions

1. Who renders the PPE document long-term, and when the writer moves from ConfigMap to `Policy` / AI Gateway CRDs (recommend authpolicy-transpiler path for migration). Clarify ODH AI Gateway controller vs `gateway-controller` CR ownership.
2. Shared rate-limit store: connect PPE to the existing RHCL Limitador Redis instance (preferred — no counter migration), or a new store with a **migration plan supplied by Praxis** ([#203](https://github.com/praxis-proxy/policy/issues/203)).
3. Long-term maas-api: HTTP directory vs absorbed vs PIP ([#184](https://github.com/praxis-proxy/policy/issues/184)).
4. FIPS: ship non-FIPS PPE path, or block FIPS tenants on RHCL until a FIPS policy build exists.
5. Whether per-model limits may move to LLM-entity routes as an accepted equivalent to header-matched TRLP (fallback if #352 slips).
6. **Migration vs parallel support** of existing RHCL setups (annotation coexistence is the proposed answer; full-fleet flip is rejected).
7. Where InferenceObjective / inference-priority header injection lives after RHCL.
8. Who owns "report quota usage" and whether it blocks GA for subscription token limits.
9. **Enforcement during cut-over:** while both the RHCL and PPE configurations
   exist for a tenant, which engine actually enforces, and is there a window
   where both (or neither) do? Needs an explicit answer before dev-tenant
   rollout.

## Future plans

- **PPE token minting** ([policy#380](https://github.com/praxis-proxy/policy/issues/380)):
  PPE plans credential injection and token minting. MaaS API already mints the
  keys / tokens in use; moving to PPE-minted tokens is a future option, not
  expected in 3.7. If it happens, it needs a migration plan agreed with the
  Praxis side so live credentials carry over without breaking clients — keep
  this coordinated, since MaaS is already minting tokens today.

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
