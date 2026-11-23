# Routing External Models through Praxis

| Date | 2026-11-23 |
| :---- | :---- |
| Scope | Models-as-a-Service (MaaS), AI Gateway, External Models |
| Status | Draft |
| Authors | TBD |
| Companion | [Routing MaaS-published KServe models through Praxis](https://bartoszmajsak.github.io/kserve-praxis-integration/) (Approved, 2026-11-20) |
| Other docs | [ai-gateway-controller DESIGN.md](https://github.com/opendatahub-io/ai-gateway-controller/blob/main/DESIGN.md), [ODH-ADR-MS-0005](https://github.com/opendatahub-io/architecture-decision-records/pull/165) (control-plane / ExtProc split) |

## What

The data plane moves from Gateway API on Istio to Praxis. This decision covers how an `ExternalModel` that maps to one or more `ExternalProvider`s is served in that setup: what the control plane publishes about the model, which component writes its routing in Praxis, and what users write.

Users keep writing `ExternalModel` and `ExternalProvider`. `ai-gateway-controller` remains the control-plane owner of those objects. The per-model `HTTPRoute` and Envoy route-reselection path go away. Routing (including multi-provider selection and credential injection) moves into the tenant's Praxis `Proxy` configuration. The `ModelProvider` API can be adopted later without a user-facing change.

**HTTPRoute removal and provider egress (SE/DR) are separate.** Today Istio `ServiceEntry` / `DestinationRule` own TLS origination, SNI, optional CA, and mesh-external allowlisting for third-party providers. If Praxis dials those providers itself, **Praxis must own TLS / SNI / CA / egress** — a larger dependency than dropping the HTTPRoute. That requirement is called out explicitly below; it is not assumed to be free.

## Why

The move to Praxis removes the per-model HTTPRoute. The KServe / MaaS ADR defines what MaaS and KServe do once it is gone: MaaS writes routing into the tenant `Proxy`. External models still create an `HTTPRoute` (`external-model-<name>`) today and rely on Envoy to reselect that route after ExtProc chooses a provider. Leaving them on Gateway API while KServe models move to Praxis would split the tenant dataplane: one front door, two routing systems, and policies that still attach to routes for only half the models.

[`gateway-controller`](https://github.com/praxis-proxy/gateway-controller) is not an ingress controller. It configures a Praxis proxy behind the platform's existing ingress. A Gateway can still be the entry point and forward all traffic to Praxis. **The per-model HTTPRoute is removed.** Praxis routes on the `model` field of the request body, and ["no route is ever written"](https://github.com/praxis-proxy/gateway-controller/blob/f6a12554d9294130010ed16abea775657914bb6f/docs/design/api.md?plain=1#L311).

### Today: who does what, and why

- The user writes `ExternalModel` and `ExternalProvider` in the tenant namespace. `ExternalModel` names the client-facing model and binds providers (`targetModel`, `apiFormat`, `path`, weights, optional auth overrides). Backend facts live on those specs (`ExternalProvider.spec.endpoint`, etc.) — there is no KServe-like status contract to discover them.
- **`ai-gateway-controller`** watches those CRs (plus Secrets, Services, `AITenant`). For a Praxis-selected, Active tenant it:
  - Publishes a content-addressed **routing overlay** consumed by tenant-local ExtProc (`intelligent_route`, `credential_inject`).
  - Creates a per-model **HTTPRoute** (`external-model-<name>`) parented to the tenant Gateway from `AITenant.status.gatewayRef`.
  - Creates provider **Service** + Istio **ServiceEntry** / **DestinationRule** for the external HTTPS hop (TLS `SIMPLE`, SNI = provider hostname, optional CA, per-tenant DR isolation in a shared Gateway namespace).
  - Writes `status.httpRouteName` (intended for consumers; see MaaS below).
- **`praxis-extproc`** runs as ExtProc on the Gateway: reads the body model name, selects a provider, injects credentials from projected Secrets, sets selection headers, and clears the Envoy route cache so Envoy reselects an HTTPRoute rule.
- **Envoy / Istio** still owns the provider hop via the HTTPRoute backendRefs and SE/DR.
- **MaaS** owns the client-facing publish object **`MaaSModelRef`**. For external models it fills `MaaSModelRef.status` (`Endpoint`, `HTTPRoute*`, `Phase`) from the HTTPRoute / Gateway — the same shape as the LLMIS path. It does **not** read `ExternalModel` status today; readiness for the MaaS path is effectively “route accepted,” and endpoint/hostnames come from route/Gateway listeners, not from a KServe-style Addresses/URL contract on ExternalModel.

### With Praxis: who does what, and why

- The platform ingress only forwards. The proxy is not tied to any Gateway API implementation for per-model routing.
- MaaS creates a `Proxy` and a `Policy` per tenant (same as the KServe ADR).
- **`ai-gateway-controller`** stops creating per-model HTTPRoutes. After providers and credentials resolve, it publishes routing facts on `ExternalModel` status; **MaaS** — the sole Proxy writer, see "Proxy ownership" — writes the routing entry into the tenant `Proxy`; Ready is set only after the entry is applied.
- **`gateway-controller`** renders the Praxis config from the `Proxy` and the `Policy`. Exactly one rendered config owner.
- Credential injection and multi-provider selection run **inside Praxis**, not as ExtProc + Envoy reselection.
- **Provider egress:** either Praxis originates TLS to external providers (must support TLS mode, SNI, CA, and allow-listed egress — what DestinationRule + ServiceEntry do today), or Istio SE/DR remain for egress under an explicit hybrid that is *not* the preferred end state. Bundling “drop HTTPRoute” with “drop SE/DR” without that Praxis capability is a massive ask.
- Nothing requires a per-model HTTPRoute for readiness or for policy attachment. `MaaSModelRef` endpoint/hostnames come from the tenant ingress host (same fix as the companion ADR), not from route status.

## Shortcomings in today's integration

Praxis ExtProc already participates, but model routing and policy attachment still depend on Gateway API. Egress TLS is a separate Istio concern.

| Shortcoming | What it causes | Fix |
| :---- | :---- | :---- |
| Per-model routing is an HTTPRoute, and ExtProc only *chooses* — Envoy *routes*. | Removing the route breaks provider selection even when the overlay is correct. Route reselection is an Istio/Envoy pattern, not a Praxis one. | Praxis owns model → provider *routing*. Stop creating `external-model-*` HTTPRoutes. |
| `MaaSModelRef` endpoint / `httpRoute*` / Ready are filled from the HTTPRoute. | Without a route the MaaS publish object has no endpoint and stays Pending, even though ExternalModel specs already know the backend. MaaS does not read ExternalModel status. | Take endpoint from the tenant ingress host; stop gating Ready on route acceptance (same class of fix as the companion ADR). No KServe-like Addresses/URL contract needed on ExternalModel. |
| Per-model Kuadrant policies attach to the HTTPRoute. | Without a route, MaaS has nowhere to hang per-model limits. | Generate per-model limits and auth in the Praxis `Policy`. |
| DestinationRule / ServiceEntry own TLS, SNI, CA, and mesh-external allowlisting. | If Praxis dials providers without an equivalent, HTTPS to OpenAI/Anthropic/etc. regresses (wrong SNI, no custom CA, open egress). | **Requirement:** Praxis upstream config must own TLS / SNI / CA / egress for external providers before SE/DR can be removed. Track as an explicit Praxis dependency, not as free fallout of HTTPRoute removal. |
| Ready on ExternalModel means "networking resources created" (route + transport). | Couples control-plane Ready to Gateway API objects that are going away. A definition that publishes routing only for Ready models would be circular: a new model has no routing entry, so it could never become Ready. | Ready means providers + credentials resolve and the routing entry was applied to the Proxy (or ModelProvider), independent of HTTPRoute status. The entry is published first, after validation; Ready follows the applied entry. |
| Tenant identity and Gateway binding still gate ExternalModel reconcile. | Correct for today's ExtProc-on-Gateway install; Gateway becomes forward-only under Praxis. | Keep the tenant Gateway as entry (ADR non-goal to remove it). Point the tenant at a `Proxy` the same way the KServe ADR does. |
| Credential material is projected for ExtProc file reads. | Works for ExtProc; Praxis needs an explicit secret / credential reference model without embedding secret bytes in config. | Keep reference-only credentials; map them into Praxis credential injection. |

## Goals

- An ExternalModel in a Praxis tenant is served through Praxis, not through a per-model HTTPRoute.
- Users keep writing `ExternalModel` / `ExternalProvider`. Clients keep the hostname, the request URL with the model in the body, and the model ID (`spec.modelName` or `metadata.name`).
- One writer per object. `ai-gateway-controller` owns ExternalModel control plane; `gateway-controller` owns rendered proxy config; MaaS owns tenant `Proxy` / `Policy` scaffolding, `MaaSModelRef`, and subscription-derived policy rules. The tenant `Proxy` itself has exactly **one** routing writer (see "Proxy ownership" below) — the Proxy API's atomic lists do not allow two.
- Align with the KServe ADR: same tenant cut-over unit, same ingress-forward model, `ModelProvider` available later as an alternative shape.
- Treat **routing** (HTTPRoute) and **egress TLS** (SE/DR) as separable deliverables; do not remove SE/DR until Praxis owns TLS / SNI / CA / egress.
- A tenant can move in stages and be rolled back.

## Non-Goals

- The move to Praxis itself. It is the starting point and is decided by the platform / companion ADR.
- Changing the user-facing ExternalModel / ExternalProvider spec APIs. Status changes are in scope: routing facts for MaaS replace the HTTPRoute-naming fields.
- Defining the `ModelProvider` API (see Alternatives).
- Inventing a KServe-like Addresses/URL discovery contract on ExternalModel. MaaS does read `ExternalModel` status under this ADR, but only routing facts `ai-gateway-controller` resolves from spec — providers, weights, paths, credential references, validation result.
- An ingress without Gateway API. The tenant Gateway stays in front of the proxy.
- Per-model path URLs of the form `/<namespace>/<name>/v1/...` through Praxis (same non-goal as the companion ADR).
- Moving a single model. The unit of the cut-over is the tenant.
- Replacing MaaS ownership of auth subscriptions / API keys; only the attachment point moves from HTTPRoute to Praxis `Policy`.
- In-cluster KServe / `LLMInferenceService` routing (covered by the companion ADR).

## How

We propose to keep `ExternalModel` and `ExternalProvider` as the objects a user writes. `ai-gateway-controller` already knows the backends, provider bindings, weights, paths, and credential references from those specs. For Praxis-selected tenants it:

1. **Stops creating** `HTTPRoute` `external-model-*` (and Envoy reselection that depends on them) once the tenant is cut over to the proxy — see the migration gate below.
2. **Publishes** resolved routing facts (providers, weights, paths, credential references, validation result, revision) on `ExternalModel` status once providers and credentials resolve, then sets Ready once MaaS reports the routing entry applied **at the current revision** — see "Proxy ownership" for why MaaS, not `ai-gateway-controller`, writes the entry. If a new revision fails validation, Ready is cleared for that revision: status must not report Ready based on an older applied report while the live spec is invalid. The last successfully applied revision keeps serving until a valid successor is applied (or the entry is removed).
3. **Keeps** Istio Service / ServiceEntry / DestinationRule until Praxis can express the same TLS / SNI / CA / egress semantics on the provider hop — then removes them in a follow-on.
4. Moves credential injection and multi-provider selection from ExtProc-on-Envoy into Praxis filters.

`gateway-controller` renders `Proxy` + `Policy`. No `ModelProvider` is required for the first step.

### Migration gate for HTTPRoute removal

A Praxis-selected tenant may still serve traffic through its per-model HTTPRoutes. Routes are removed only when the tenant is cut over to the standalone proxy:

- `ai-gateway-controller` already gates on a Praxis-selected, Active `AITenant`; MaaS records the tenant's `Proxy` and `Policy` on the `AITenant` (shared with the companion ADR). No separate Gateway signal is required, unlike the KServe path, because `ai-gateway-controller` reads `AITenant` directly.
- **Per-model gate:** an `external-model-*` HTTPRoute is deleted only when that model's routing entry — at the current revision — is applied in the tenant `Proxy`, as reported by MaaS. "Prepared" means exactly this: the recorded `Proxy` exists, is rendered by `gateway-controller`, and serves the entry. A tenant-level Active/Proxy condition alone is not sufficient; route deletion must never remove the only active path for a model. Models created during or after cutover never receive an HTTPRoute and skip this check.
- **Authorization gate:** the per-model HTTPRoutes carry today's Kuadrant authorization. They are deleted only after the tenant's Praxis `Policy` is applied and effective. Models created during or after cutover have no HTTPRoute, so the proxy must reject their requests until the `Policy` is effective — route deletion alone cannot cover them.
- Rollback recreates removed routes through the existing `modelHTTPRouteSet` behavior; the overlay publication path stays available during hybrid operation so a tenant can move back without data loss.

```
Today:    Client → Gateway/Envoy → ExtProc (choose + creds) → HTTPRoute reselection → SE/DR → provider
Step 1:   Client → Gateway (forward) → Praxis Proxy (route + creds) → SE/DR (or Praxis TLS) → provider
End state: Client → Gateway (forward) → Praxis Proxy (route + creds + TLS/SNI/CA/egress) → provider
```

**Interim provider hop (Step 1), made concrete.** ServiceEntry and DestinationRule only apply to traffic leaving an Istio-managed workload, so during the interim the Praxis proxy pod is enrolled in the mesh with an Istio sidecar that intercepts outbound provider traffic. **TLS originates once, in Praxis** — the proxy's own upstream TLS config ([`cluster.rs`](https://github.com/praxis-proxy/praxis/blob/2212314b12a4536774f41ed4e1aec1619d403459/crates/tls/src/config/cluster.rs)) provides SNI and the CA; DestinationRule is not used for TLS origination on this path.

Sidecar interception alone does **not** enforce the provider allowlist. Under Istio's default `ALLOW_ANY`, the sidecar forwards to hosts with no ServiceEntry, so a misrouted request can still leave the pod with provider credentials attached. Before cutover, outbound provider traffic must fail closed: route it through an allowlisting egress path (ServiceEntry-backed egress gateway, or equivalent), and enforce that path with NetworkPolicy (DNS and control-plane traffic allowed separately). `outboundTrafficPolicy.mode: REGISTRY_ONLY` may be defense in depth; it is not the enforcement boundary. In the end state the sidecar / interim egress hop is removed and Praxis owns egress directly (see the CONNECT-proxy dependency).

### Who does what, and why

- The user writes the same `ExternalModel` / `ExternalProvider` as today. Spec needs no new required field for the first step.
- **`ai-gateway-controller`** resolves providers and Secrets, publishes resolved routing facts on `ExternalModel` status, and retains SE/DR until the Praxis egress requirement is met.
- **MaaS** is the sole writer of the tenant `Proxy`: it reads `ExternalModel` status, writes / removes the routing entries, rejects model-ID collisions between KServe and external models, generates auth and per-model limits that do not attach to a route, fills `MaaSModelRef` endpoint from the tenant ingress host, and continues to own subscriptions. `ai-gateway-controller` sets Ready on `ExternalModel` once MaaS reports the routing entry applied — no HTTPRoute condition.
- **`gateway-controller`** renders `Proxy` + `Policy`.
- The tenant Gateway remains the entry hostname clients already use; it forwards to the proxy.

### Proxy ownership

The Proxy API's lists are atomic: a listener's `filterChains` has no `listType` marker (atomic by default), and each chain's `filters` is `+listType=atomic` ([`proxy_types.go`](https://github.com/praxis-proxy/gateway-controller/blob/68d3ce01d9e981f94b5192b125f8eceb55e18bd0/api/proxy/v1alpha1/proxy_types.go)). Kubernetes treats an atomic list as one unit owned by one manager and replaced whole. Two controllers adding entries to the same chain — or even naming each other's chains in `filterChains` — cannot partition ownership via server-side apply; one writer's apply can drop or overwrite the other's entries.

The design therefore requires **one routing writer per tenant `Proxy`**, and that writer owns chain order for the whole listener.

**MaaS is the sole Proxy writer**, consistent with the KServe companion ADR, which already gives MaaS this role in the tenant namespace:

- `ai-gateway-controller` publishes resolved routing facts — providers, weights, paths, credential references, validation result — on `ExternalModel` status.
- MaaS reads that status and writes / removes the routing entries in the tenant `Proxy`, alongside its KServe routing writes.
- MaaS rejects a KServe model and an ExternalModel that claim the same client-facing model ID — a check only the single writer can own.
- MaaS reflects the applied entry on `MaaSModelRef` (endpoint from the tenant ingress host, Ready once the entry is applied), so `ai-gateway-controller` sets `ExternalModel` Ready from MaaS's report and the publish object no longer depends on route status. The report is tied to a **routing revision**: each published set of routing facts carries a revision or content identity, and Ready requires an applied report matching the *current* identity. A report for a superseded identity — for example when provider configuration changes from entry A to entry B while B is still being applied — does not satisfy Ready, so `ExternalModel` cannot report Ready for configuration the Proxy is not yet serving.

Rejected shapes: **`ai-gateway-controller` as sole Proxy writer** — inverts the companion ADR's writer for KServe models and would need MaaS to hand KServe routing over; **two SSA field owners in one Proxy** — unworkable given the atomic lists above. `ModelProvider` per ExternalModel (see Alternatives) remains a later step that would give each entry its own object and writer without changing this decision's user-facing outcome.

### Benefits

- **One dataplane per Praxis tenant** for model routing. External and in-cluster models share the Proxy story.
- **One writer per object, including the Proxy.** ExternalModel stays with ai-gateway-controller; Proxy scaffolding and MaaSModelRef stay with MaaS; rendered config stays with gateway-controller; exactly one controller writes routing into the tenant `Proxy`.
- **No dependency on the `ModelProvider` API** for the first cut.
- **Nothing changes in what users write.** Clients keep body-based model IDs.
- **Egress risk is bounded.** HTTPRoute can go away without pretending DestinationRule is optional.

### Requirement: Praxis owns TLS / SNI / CA / egress (before SE/DR removal)

Upstream TLS with SNI and a custom CA already exists in Praxis ([`cluster.rs`]). The gap is narrower than "Praxis has no TLS": what is missing is **egress through an HTTP CONNECT proxy** — the allow-listed, controlled egress path SE/DR provide today — tracked as [`praxis-proxy/praxis#1589`](https://github.com/praxis-proxy/praxis/issues/1589).

Today's DestinationRule sets at least:

- TLS mode `SIMPLE`
- SNI = provider hostname (host only, not host:port)
- optional `caCertificates`
- per-tenant isolation when multiple tenants share a Gateway namespace
- `exportTo` so the Gateway workload sees the policy

ServiceEntry makes the host `MESH_EXTERNAL` with DNS resolution. ExternalName Service gives the HTTPRoute a local backend name.

**Before** ai-gateway-controller stops creating SE/DR for Praxis tenants, Praxis upstream/egress config must provide the equivalent. Until then, keeping Istio transport for the provider hop while Praxis owns only model routing is an acceptable interim. Removing SE/DR without that capability is out of scope for the first cut.

## Changes in ai-gateway-controller

| Change | Today | Size |
| :---- | :---- | :---- |
| Stop creating `HTTPRoute` `external-model-*` for cut-over tenants only. | `modelHTTPRouteSet` builds Gateway API routes with provider header matches and sink rules. | Medium |
| Publish resolved routing facts (providers, weights, paths, credential refs, validation) on `ExternalModel` status for MaaS to consume — see "Proxy ownership". | Publishes ExtProc overlay; Envoy owns the hop after reselection. | Medium / New |
| Redefine Ready: routing facts (with revision) published after providers + credentials resolve; Ready set once MaaS reports the entry applied at the current revision; no HTTPRoute condition. | Ready ≈ networking resources created; `status.httpRouteName` set. | Small |
| Retire or replace `status.httpRouteName` (MaaS does not read it today anyway; route naming is hardcoded on the MaaS side). | CRD documents it for consumers. | Small, API change |
| Keep reference-only credential publication; map into Praxis credential config. | ExtProc file projection + `credential_inject`. | Medium |
| Preserve weighted multi-provider selection semantics in Praxis routing. | Weights on `ExternalProviderRef`; ExtProc `intelligent_route` + header reselection. | Medium — weighted selection on static candidates is **not available in Praxis today** (see Dependencies) |
| **Later:** stop creating Service / ServiceEntry / DestinationRule once Praxis TLS/SNI/CA/egress is real. | `applyTransport` owns the external HTTPS hop. | Medium — **blocked on Praxis** |
| Tenant gate stays: Praxis-selected Active `AITenant`; Gateway as ingress forward target, not HTTPRoute parent. | Reconcile no-ops / cleans up without Active Praxis tenant. | Small |

## Changes in MaaS

| Change | Today | Size |
| :---- | :---- | :---- |
| Read `ExternalModel` status and write / remove external-model routing entries in the tenant `Proxy` (sole Proxy writer — see "Proxy ownership"); report the applied entry with its revision so `ai-gateway-controller` can set Ready. | Does not read ExternalModel status. | Medium / New |
| Reject model-ID collisions between KServe models and ExternalModels. | No single writer owns the check. | New |
| Stop deriving `MaaSModelRef` endpoint / `httpRoute*` / Ready from the ExternalModel HTTPRoute; fill from the applied Proxy entry and tenant ingress host. | Fills status from route/Gateway; does not read ExternalModel status. | Medium |
| Take endpoint from the tenant ingress host (same as companion ADR). | Hostnames from route or Gateway listeners. | Medium, API change |
| Stop attaching per-model policies to ExternalModel HTTPRoutes. | Kuadrant on Gateway / HTTPRoute. | Medium |
| Generate per-model token limits and auth rules in the Praxis `Policy`. | Kuadrant policies. | New (shared with companion ADR) |
| Create `Proxy` + `Policy` for an `AITenant`, record Proxy on the tenant. | AITenant names Gateway only. | New (shared with companion ADR) |

## Changes in Praxis / gateway-controller

| Change | Today | Size |
| :---- | :---- | :---- |
| Render Proxy + Policy and deploy the proxy (shared dependency). | Draft / in flight on gateway-controller. | Shared with companion ADR |
| Model body routing + multi-provider weighted selection for ExternalModel IDs. | ExtProc `intelligent_route` overlay. | Body routing exists in proxy; **weighted selection works only in overlay mode today — static candidates cannot use weights** ([`intelligent_route` docs](https://github.com/praxis-proxy/ai/blob/a59c9270258a68c68f45419ef02d3f5a0a195fdd/docs/filters/intelligent_route.md)). Weighted static candidates are new upstream work |
| Credential injection from Secret references without secret bytes in config. | ExtProc `credential_inject`. | Exists in ExtProc; must exist in proxy |
| **Egress through an HTTP CONNECT proxy for third-party HTTPS providers** (blocks SE/DR removal; TLS/SNI/CA already supported). | Istio DestinationRule + ServiceEntry. | **New — tracked as [`praxis-proxy/praxis#1589`](https://github.com/praxis-proxy/praxis/issues/1589)** |

## Dependencies

- `gateway-controller` renders and deploys the proxy from a `Proxy` and a `Policy` ([`gateway-controller#14`](https://github.com/praxis-proxy/gateway-controller/issues/14)–[`#17`](https://github.com/praxis-proxy/gateway-controller/issues/17), draft [`#82`](https://github.com/praxis-proxy/gateway-controller/pull/82)).
- Policy engine: Kubernetes token auth ([`policy#213`](https://github.com/praxis-proxy/policy/issues/213), [`#214`](https://github.com/praxis-proxy/policy/issues/214)), per-model token limits ([`policy#352`](https://github.com/praxis-proxy/policy/issues/352), [`#308`](https://github.com/praxis-proxy/policy/issues/308)), shared limit counters ([`policy#127`](https://github.com/praxis-proxy/policy/issues/127)).
- Credential injection and weighted provider selection on the Praxis proxy path, not only on `praxis-extproc` attached to Istio. **Weighted selection with static (non-overlay) candidates does not exist in Praxis today** — upstream must add it before the ExtProc overlay can be dropped for multi-provider ExternalModels.
- **Praxis upstream TLS / SNI / CA:** already supported for upstream clusters ([`cluster.rs`](https://github.com/praxis-proxy/praxis/blob/2212314b12a4536774f41ed4e1aec1619d403459/crates/tls/src/config/cluster.rs)). What is missing is egress through an HTTP CONNECT proxy — tracked upstream as [`praxis-proxy/praxis#1589`](https://github.com/praxis-proxy/praxis/issues/1589). Required before SE/DR removal.
- Companion ADR migration stages: tenant cut-over is shared; ExternalModel entries are additional Proxy content for the same tenant.

## Open Questions

- **Shape of the `ExternalModel` status contract MaaS reads.** MaaS is the sole Proxy writer (decided — see "Proxy ownership"); what remains open is the exact status fields for resolved routing facts, the form of the routing revision / content identity, and how MaaS reports the applied entry and its revision back for Ready. Decided for the failed-validation case: Ready clears for the invalid revision; the last applied revision keeps serving until a valid successor is applied or the entry is removed — status must not imply the invalid revision is Ready from an older report alone.
- **Interim egress:** keep SE/DR while Praxis routes, or wait to cut ExternalModel traffic over until Praxis TLS is ready?
- **How the Gateway shows it is in front of Praxis.** Same open question as the companion ADR (GatewayClass vs annotation).
- **Secret projection vs proxy-native credential refs.** Today the only way to get a Secret into the proxy through a `Proxy` is `secretEnv` — Secret keys as env vars in the same namespace — so rotation requires a proxy restart. Hot rotation of provider credentials needs a native credential-reference mechanism in the Proxy API. Affects RBAC for the proxy ServiceAccount.
- **Price Tag / metering filters.** Confirm metering moves with the proxy so ExternalModel traffic stays metered after HTTPRoute removal.

## Alternatives

### Option 2: `ai-gateway-controller` creates a `ModelProvider` per ExternalModel

Instead of writing routing inline into the tenant `Proxy`, `ai-gateway-controller` creates a Praxis `ModelProvider` for each `ExternalModel`. `gateway-controller` renders routing from those objects. MaaS still owns the tenant `Proxy` / `Policy` scaffolding and `MaaSModelRef`.

This mirrors companion-ADR option 2 (MaaS creates `ModelProvider` for published KServe models), except the writer for external models is `ai-gateway-controller`, which already owns `ExternalModel` / `ExternalProvider`.

**Why it is not preferred for the first step:** the `ModelProvider` API is open with no milestone ([`gateway-controller#40`](https://github.com/praxis-proxy/gateway-controller/issues/40), [`#42`](https://github.com/praxis-proxy/gateway-controller/issues/42)). The preferred option reaches the same user-facing outcome with `Proxy` + `Policy` only, and can move to `ModelProvider` later without changing what users write.

**What does not change under this alternative:** HTTPRoute still goes away; `MaaSModelRef` still stops depending on route status; **Praxis must still own TLS / SNI / CA / egress before SE/DR can be removed.** `ModelProvider` only changes where the routing entry lives, not who originates HTTPS to the provider.

| Relationship | Proxy inline (single writer, see "Proxy ownership") | Alternative (ModelProvider) |
| :---- | :---- | :---- |
| Routing entry | Inline in tenant `Proxy`, written by MaaS from `ExternalModel` status | `ModelProvider` per ExternalModel, written by ai-gateway-controller |
| Depends on `ModelProvider` API | No | Yes |
| HTTPRoute | Removed | Removed |
| SE/DR until Praxis TLS/SNI/CA/egress | Kept | Kept |
| User-facing CRs | Unchanged | Unchanged |

## Security and Privacy Considerations

- Authentication and authorization move from Kuadrant-on-HTTPRoute to Praxis `Policy`, generated from the same MaaS auth and subscription objects. Tenant cut-over waits on Kubernetes token auth in the policy engine (same gate as the companion ADR).
- Credential references must stay reference-only in any overlay or Proxy / ModelProvider config. Secret bytes must not enter etcd via routing objects (already enforced for today's overlay).
- The proxy ServiceAccount needs carefully scoped access to provider credential Secrets (or a projection path owned by ai-gateway-controller).
- Egress to arbitrary external provider hostnames is a trust boundary. ServiceEntry alone is not that boundary under `ALLOW_ANY`; the interim requires a fail-closed egress path (see Step 1). Praxis needs an equivalent allow-listed upstream per ExternalProvider before SE/DR go away.
- No new client data is stored. The proxy reads the request body for the model name, as ExtProc does today.

## Risks and Considerations

- All models of a tenant (KServe-published and ExternalModel) may land in one `Proxy` object; every publish/unpublish is a write to the same object (same risk as the companion ADR).
- Proxy ownership is decided (MaaS sole writer, see "Proxy ownership"), but it adds a MaaS dependency to every ExternalModel publish: if MaaS is down, external models stop becoming Ready even though `ai-gateway-controller` has validated them.
- Treating SE/DR removal as implied by HTTPRoute removal underestimates the Praxis TLS/SNI/CA/egress gap.
- When an ExternalProvider Secret rotates, Praxis must pick up new credentials. Today the `Proxy` API only supports Secret keys as env vars (`secretEnv`), so rotation restarts proxy pods; hot rotation is an open Praxis dependency.
- During hybrid migration, a tenant must not split a single client-facing model ID across ExtProc-on-Envoy and Praxis Proxy.
- Routing config rolls that restart proxy pods (companion ADR risk) affect external and in-cluster models together.

## Stakeholder Impacts

| Group | Impacted? |
| :---- | :---- |
| AI Gateway (`ai-gateway-controller`) | Stops HTTPRoute for Praxis tenants; publishes routing facts on ExternalModel status for MaaS; keeps SE/DR until Praxis egress TLS exists; redefines Ready / status. |
| Models-as-a-Service (MaaS) | Becomes sole Proxy writer: reads ExternalModel status, writes routing entries, rejects model-ID collisions; stops filling `MaaSModelRef` from ExternalModel HTTPRoutes; owns Proxy / Policy scaffolding; shared cut-over with KServe path. |
| AI Gateway (Praxis) | Proxy must route ExternalModel IDs, inject credentials, select among weighted providers, and — before SE/DR removal — own TLS / SNI / CA / egress to third-party providers. |
| Dashboard / users | Keep writing ExternalModel / ExternalProvider; same hostnames and body model IDs. Per-model path URLs stop working. |
| Model Serving (KServe) | None directly; companion ADR only. |

## References

- Companion ADR: Routing MaaS-published KServe models through Praxis (Approved 2026-11-20)
- [praxis-proxy/gateway-controller design](https://github.com/praxis-proxy/gateway-controller/blob/f6a12554d9294130010ed16abea775657914bb6f/docs/design/api.md)
- ai-gateway-controller `DESIGN.md` (Phase 2 ExternalModel + praxis-extproc)
- ODH-ADR-MS-0005 — External Model / Praxis two-plane control-plane split
