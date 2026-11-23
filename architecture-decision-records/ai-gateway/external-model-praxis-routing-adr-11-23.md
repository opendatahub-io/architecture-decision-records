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

The approved KServe / MaaS ADR removes the per-model HTTPRoute for in-cluster published models and has MaaS write routing into the tenant `Proxy`. External models still create an `HTTPRoute` (`external-model-<name>`) today and rely on Envoy to reselect that route after ExtProc chooses a provider. Leaving them on Gateway API while KServe models move to Praxis would split the tenant dataplane: one front door, two routing systems, and policies that still attach to routes for only half the models.

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
- **`ai-gateway-controller`** stops creating per-model HTTPRoutes. It writes routing entries for each Ready ExternalModel into the tenant `Proxy` under a dedicated ownership boundary (or later as `ModelProvider` objects — see Alternatives).
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
| Ready on ExternalModel means "networking resources created" (route + transport). | Couples control-plane Ready to Gateway API objects that are going away. | Ready means providers + credentials resolve and the routing entry was accepted into the Proxy (or ModelProvider), independent of HTTPRoute status. |
| Tenant identity and Gateway binding still gate ExternalModel reconcile. | Correct for today's ExtProc-on-Gateway install; Gateway becomes forward-only under Praxis. | Keep the tenant Gateway as entry (ADR non-goal to remove it). Point the tenant at a `Proxy` the same way the KServe ADR does. |
| Credential material is projected for ExtProc file reads. | Works for ExtProc; Praxis needs an explicit secret / credential reference model without embedding secret bytes in config. | Keep reference-only credentials; map them into Praxis credential injection. |

## Goals

- An ExternalModel in a Praxis tenant is served through Praxis, not through a per-model HTTPRoute.
- Users keep writing `ExternalModel` / `ExternalProvider`. Clients keep the hostname, the request URL with the model in the body, and the model ID (`spec.modelName` or `metadata.name`).
- One writer per object. `ai-gateway-controller` owns ExternalModel control plane; `gateway-controller` owns rendered proxy config; MaaS owns tenant `Proxy` / `Policy` scaffolding, `MaaSModelRef`, and subscription-derived policy rules.
- Align with the KServe ADR: same tenant cut-over unit, same ingress-forward model, `ModelProvider` available later as an alternative shape.
- Treat **routing** (HTTPRoute) and **egress TLS** (SE/DR) as separable deliverables; do not remove SE/DR until Praxis owns TLS / SNI / CA / egress.
- A tenant can move in stages and be rolled back.

## Non-Goals

- The move to Praxis itself. It is the starting point and is decided by the platform / companion ADR.
- Changing the user-facing ExternalModel / ExternalProvider APIs beyond status fields that name HTTPRoutes.
- Defining the `ModelProvider` API (see Alternatives).
- Inventing a KServe-like status contract on ExternalModel for MaaS to discover backends — those facts are already on ExternalModel / ExternalProvider spec; the broken contract is `MaaSModelRef` ↔ HTTPRoute.
- An ingress without Gateway API. The tenant Gateway stays in front of the proxy.
- Per-model path URLs of the form `/<namespace>/<name>/v1/...` through Praxis (same non-goal as the companion ADR).
- Moving a single model. The unit of the cut-over is the tenant.
- Replacing MaaS ownership of auth subscriptions / API keys; only the attachment point moves from HTTPRoute to Praxis `Policy`.
- In-cluster KServe / `LLMInferenceService` routing (covered by the companion ADR).

## How

We propose to keep `ExternalModel` and `ExternalProvider` as the objects a user writes. `ai-gateway-controller` already knows the backends, provider bindings, weights, paths, and credential references from those specs. For Praxis-selected tenants it:

1. **Stops creating** `HTTPRoute` `external-model-*` (and Envoy reselection that depends on them).
2. **Writes** a routing entry for each Ready ExternalModel into the tenant `Proxy`, with SSA field ownership that does not fight MaaS's KServe routing writes.
3. **Keeps** Istio Service / ServiceEntry / DestinationRule until Praxis can express the same TLS / SNI / CA / egress semantics on the provider hop — then removes them in a follow-on.
4. Moves credential injection and multi-provider selection from ExtProc-on-Envoy into Praxis filters.

`gateway-controller` renders `Proxy` + `Policy`. No `ModelProvider` is required for the first step.

```
Today:    Client → Gateway/Envoy → ExtProc (choose + creds) → HTTPRoute reselection → SE/DR → provider
Step 1:   Client → Gateway (forward) → Praxis Proxy (route + creds) → SE/DR (or Praxis TLS) → provider
End state: Client → Gateway (forward) → Praxis Proxy (route + creds + TLS/SNI/CA/egress) → provider
```

### Who does what, and why

- The user writes the same `ExternalModel` / `ExternalProvider` as today. Spec needs no new required field for the first step.
- **`ai-gateway-controller`** resolves providers and Secrets, publishes Ready without an HTTPRoute, and writes / removes Proxy routing entries. It retains SE/DR until the Praxis egress requirement is met.
- **MaaS** creates the tenant `Proxy` and `Policy`, generates auth and per-model limits that do not attach to a route, fills `MaaSModelRef` endpoint from the tenant ingress host, and continues to own subscriptions.
- **`gateway-controller`** renders `Proxy` + `Policy`.
- The tenant Gateway remains the entry hostname clients already use; it forwards to the proxy.

### Benefits

- **One dataplane per Praxis tenant** for model routing. External and in-cluster models share the Proxy story.
- **One writer per object.** ExternalModel stays with ai-gateway-controller; Proxy scaffolding and MaaSModelRef stay with MaaS; rendered config stays with gateway-controller.
- **No dependency on the `ModelProvider` API** for the first cut.
- **Nothing changes in what users write.** Clients keep body-based model IDs.
- **Egress risk is bounded.** HTTPRoute can go away without pretending DestinationRule is optional.

### Requirement: Praxis owns TLS / SNI / CA / egress (before SE/DR removal)

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
| Stop creating `HTTPRoute` `external-model-*` for Praxis-selected tenants. | `modelHTTPRouteSet` builds Gateway API routes with provider header matches and sink rules. | Medium |
| Write routing entries into the tenant `Proxy` (SSA / field owner). | Publishes ExtProc overlay; Envoy owns the hop after reselection. | Medium / New |
| Redefine Ready: providers + credentials resolve and routing entry was applied; no HTTPRoute condition. | Ready ≈ networking resources created; `status.httpRouteName` set. | Small |
| Retire or replace `status.httpRouteName` (MaaS does not read it today anyway; route naming is hardcoded on the MaaS side). | CRD documents it for consumers. | Small, API change |
| Keep reference-only credential publication; map into Praxis credential config. | ExtProc file projection + `credential_inject`. | Medium |
| Preserve weighted multi-provider selection semantics in Praxis routing. | Weights on `ExternalProviderRef`; ExtProc `intelligent_route` + header reselection. | Medium |
| **Later:** stop creating Service / ServiceEntry / DestinationRule once Praxis TLS/SNI/CA/egress is real. | `applyTransport` owns the external HTTPS hop. | Medium — **blocked on Praxis** |
| Tenant gate stays: Praxis-selected Active `AITenant`; Gateway as ingress forward target, not HTTPRoute parent. | Reconcile no-ops / cleans up without Active Praxis tenant. | Small |

## Changes in MaaS

| Change | Today | Size |
| :---- | :---- | :---- |
| Stop deriving `MaaSModelRef` endpoint / `httpRoute*` / Ready from the ExternalModel HTTPRoute. | Fills status from route/Gateway; does not read ExternalModel status. | Medium |
| Take endpoint from the tenant ingress host (same as companion ADR). | Hostnames from route or Gateway listeners. | Medium, API change |
| Stop attaching per-model policies to ExternalModel HTTPRoutes. | Kuadrant on Gateway / HTTPRoute. | Medium |
| Generate per-model token limits and auth rules in the Praxis `Policy`. | Kuadrant policies. | New (shared with companion ADR) |
| Create `Proxy` + `Policy` for an `AITenant`, record Proxy on the tenant. | AITenant names Gateway only. | New (shared with companion ADR) |

## Changes in Praxis / gateway-controller

| Change | Today | Size |
| :---- | :---- | :---- |
| Render Proxy + Policy and deploy the proxy (shared dependency). | Draft / in flight on gateway-controller. | Shared with companion ADR |
| Model body routing + multi-provider weighted selection for ExternalModel IDs. | ExtProc `intelligent_route` overlay. | Exists in ExtProc; must exist in proxy |
| Credential injection from Secret references without secret bytes in config. | ExtProc `credential_inject`. | Exists in ExtProc; must exist in proxy |
| **TLS / SNI / CA / egress for third-party HTTPS providers** (blocks SE/DR removal). | Istio DestinationRule + ServiceEntry. | **New — large; must be tracked** |

## Dependencies

- `gateway-controller` renders and deploys the proxy from a `Proxy` and a `Policy` ([`gateway-controller#14`](https://github.com/praxis-proxy/gateway-controller/issues/14)–[`#17`](https://github.com/praxis-proxy/gateway-controller/issues/17), draft [`#82`](https://github.com/praxis-proxy/gateway-controller/pull/82)).
- Policy engine: Kubernetes token auth ([`policy#213`](https://github.com/praxis-proxy/policy/issues/213), [`#214`](https://github.com/praxis-proxy/policy/issues/214)), per-model token limits ([`policy#352`](https://github.com/praxis-proxy/policy/issues/352), [`#308`](https://github.com/praxis-proxy/policy/issues/308)), shared limit counters ([`policy#127`](https://github.com/praxis-proxy/policy/issues/127)).
- Credential injection and weighted provider selection on the Praxis proxy path, not only on `praxis-extproc` attached to Istio.
- **Praxis upstream TLS / SNI / CA / egress** equivalent to today's DestinationRule + ServiceEntry — required before SE/DR removal; open work to track explicitly.
- Companion ADR migration stages: tenant cut-over is shared; ExternalModel entries are additional Proxy content for the same tenant.

## Open Questions

- **Who writes ExternalModel routing into the `Proxy`?** Preferred here: `ai-gateway-controller`, with a distinct SSA field owner from MaaS's KServe routing writes. Alternative shape: `ModelProvider` per ExternalModel (below).
- **Interim egress:** keep SE/DR while Praxis routes, or wait to cut ExternalModel traffic over until Praxis TLS is ready?
- **How the Gateway shows it is in front of Praxis.** Same open question as the companion ADR (GatewayClass vs annotation).
- **Secret projection vs proxy-native credential refs.** Affects RBAC for the proxy ServiceAccount.
- **Price Tag / metering filters.** Confirm metering moves with the proxy so ExternalModel traffic stays metered after HTTPRoute removal.

## Alternatives

### Option 2: `ai-gateway-controller` creates a `ModelProvider` per ExternalModel

Instead of writing routing inline into the tenant `Proxy`, `ai-gateway-controller` creates a Praxis `ModelProvider` for each `ExternalModel`. `gateway-controller` renders routing from those objects. MaaS still owns the tenant `Proxy` / `Policy` scaffolding and `MaaSModelRef`.

This mirrors companion-ADR option 2 (MaaS creates `ModelProvider` for published KServe models), except the writer for external models is `ai-gateway-controller`, which already owns `ExternalModel` / `ExternalProvider`.

**Why it is not preferred for the first step:** the `ModelProvider` API is open with no milestone ([`gateway-controller#40`](https://github.com/praxis-proxy/gateway-controller/issues/40), [`#42`](https://github.com/praxis-proxy/gateway-controller/issues/42)). The preferred option reaches the same user-facing outcome with `Proxy` + `Policy` only, and can move to `ModelProvider` later without changing what users write.

**What does not change under this alternative:** HTTPRoute still goes away; `MaaSModelRef` still stops depending on route status; **Praxis must still own TLS / SNI / CA / egress before SE/DR can be removed.** `ModelProvider` only changes where the routing entry lives, not who originates HTTPS to the provider.

| Relationship | Preferred (Proxy inline) | Alternative (ModelProvider) |
| :---- | :---- | :---- |
| Routing entry | Inline in tenant `Proxy`, written by ai-gateway-controller | `ModelProvider` per ExternalModel, written by ai-gateway-controller |
| Depends on `ModelProvider` API | No | Yes |
| HTTPRoute | Removed | Removed |
| SE/DR until Praxis TLS/SNI/CA/egress | Kept | Kept |
| User-facing CRs | Unchanged | Unchanged |

## Security and Privacy Considerations

- Authentication and authorization move from Kuadrant-on-HTTPRoute to Praxis `Policy`, generated from the same MaaS auth and subscription objects. Tenant cut-over waits on Kubernetes token auth in the policy engine (same gate as the companion ADR).
- Credential references must stay reference-only in any overlay or Proxy / ModelProvider config. Secret bytes must not enter etcd via routing objects (already enforced for today's overlay).
- The proxy ServiceAccount needs carefully scoped access to provider credential Secrets (or a projection path owned by ai-gateway-controller).
- Egress to arbitrary external provider hostnames is a trust boundary. Today ServiceEntry makes that explicit; Praxis needs an equivalent allow-listed upstream per ExternalProvider before SE/DR go away.
- No new client data is stored. The proxy reads the request body for the model name, as ExtProc does today.

## Risks and Considerations

- All models of a tenant (KServe-published and ExternalModel) may land in one `Proxy` object; every publish/unpublish is a write to the same object (same risk as the companion ADR).
- SSA / field-ownership conflicts if both MaaS and ai-gateway-controller write the Proxy without a clear partition.
- Treating SE/DR removal as implied by HTTPRoute removal underestimates the Praxis TLS/SNI/CA/egress gap.
- When an ExternalProvider Secret rotates, Praxis must pick up new credentials without requiring an HTTPRoute reconcile.
- During hybrid migration, a tenant must not split a single client-facing model ID across ExtProc-on-Envoy and Praxis Proxy.
- Routing config rolls that restart proxy pods (companion ADR risk) affect external and in-cluster models together.

## Stakeholder Impacts

| Group | Impacted? |
| :---- | :---- |
| AI Gateway (`ai-gateway-controller`) | Stops HTTPRoute for Praxis tenants; writes Proxy routing (or ModelProvider later); keeps SE/DR until Praxis egress TLS exists; redefines Ready / status. |
| Models-as-a-Service (MaaS) | Stops filling `MaaSModelRef` from ExternalModel HTTPRoutes; owns Proxy / Policy scaffolding; shared cut-over with KServe path. |
| AI Gateway (Praxis) | Proxy must route ExternalModel IDs, inject credentials, select among weighted providers, and — before SE/DR removal — own TLS / SNI / CA / egress to third-party providers. |
| Dashboard / users | Keep writing ExternalModel / ExternalProvider; same hostnames and body model IDs. Per-model path URLs stop working. |
| Model Serving (KServe) | None directly; companion ADR only. |

## References

- Companion ADR: Routing MaaS-published KServe models through Praxis (Approved 2026-11-20)
- [praxis-proxy/gateway-controller design](https://github.com/praxis-proxy/gateway-controller/blob/f6a12554d9294130010ed16abea775657914bb6f/docs/design/api.md)
- ai-gateway-controller `DESIGN.md` (Phase 2 ExternalModel + praxis-extproc)
- ODH-ADR-MS-0005 — External Model / Praxis two-plane control-plane split
