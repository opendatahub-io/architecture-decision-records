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

Users keep writing `ExternalModel` and `ExternalProvider`. `ai-gateway-controller` remains the control-plane owner of those objects. The per-model `HTTPRoute`, Istio `ServiceEntry` / `DestinationRule` provider hop, and Envoy route-reselection path go away. Routing (including multi-provider selection and credential injection) moves into the tenant's Praxis `Proxy` configuration. The `ModelProvider` API can be adopted later without a user-facing change.

## Why

The approved KServe / MaaS ADR removes the per-model HTTPRoute for in-cluster published models and has MaaS write routing into the tenant `Proxy`. External models still create an `HTTPRoute` (`external-model-<name>`) today and rely on Envoy to reselect that route after ExtProc chooses a provider. Leaving them on Gateway API while KServe models move to Praxis would split the tenant dataplane: one front door, two routing systems, and policies that still attach to routes for only half the models.

[`gateway-controller`](https://github.com/praxis-proxy/gateway-controller) is not an ingress controller. It configures a Praxis proxy behind the platform's existing ingress. A Gateway can still be the entry point and forward all traffic to Praxis. **The per-model HTTPRoute is removed.** Praxis routes on the `model` field of the request body, and ["no route is ever written"](https://github.com/praxis-proxy/gateway-controller/blob/f6a12554d9294130010ed16abea775657914bb6f/docs/design/api.md?plain=1#L311).

### Today: who does what, and why

- The user writes `ExternalModel` and `ExternalProvider` in the tenant namespace. `ExternalModel` names the client-facing model and binds providers (`targetModel`, `apiFormat`, `path`, weights, optional auth overrides).
- **`ai-gateway-controller`** watches those CRs (plus Secrets, Services, `AITenant`). For a Praxis-selected, Active tenant it:
  - Publishes a content-addressed **routing overlay** consumed by tenant-local ExtProc (`intelligent_route`, `credential_inject`).
  - Creates a per-model **HTTPRoute** (`external-model-<name>`) parented to the tenant Gateway from `AITenant.status.gatewayRef`.
  - Creates provider **Service** + Istio **ServiceEntry** / **DestinationRule** for the external HTTPS hop.
  - Writes `status.httpRouteName` so MaaS can attach policies without guessing names.
- **`praxis-extproc`** runs as ExtProc on the Gateway: reads the body model name, selects a provider, injects credentials from projected Secrets, sets selection headers, and clears the Envoy route cache so Envoy reselects an HTTPRoute rule.
- **Envoy / Istio** still owns the provider hop via the HTTPRoute backendRefs and SE/DR.
- **MaaS** attaches auth / rate-limit policies that today often target Gateway API objects (including per-model HTTPRoutes via `status.httpRouteName`).

### With Praxis: who does what, and why

- The platform ingress only forwards. The proxy is not tied to any Gateway API implementation for per-model routing.
- MaaS creates a `Proxy` and a `Policy` per tenant (same as the KServe ADR).
- **`ai-gateway-controller`** stops creating HTTPRoutes and Istio transport. It publishes what Praxis needs about each ExternalModel (backends, provider bindings, credential references, Ready) and writes those routing entries into the tenant dataplane configuration — preferred: into the tenant `Proxy` under a dedicated ownership boundary, or later as `ModelProvider` objects.
- **`gateway-controller`** renders the Praxis config from the `Proxy` and the `Policy`. Exactly one rendered config owner.
- Credential injection and multi-provider selection run **inside Praxis**, not as ExtProc + Envoy reselection.
- Nothing requires a per-model HTTPRoute for readiness or for policy attachment.

## Shortcomings in today's integration

Praxis ExtProc already participates, but the provider hop and policy attachment still depend on Gateway API. The items below have to be fixed before an ExternalModel can be served through Praxis the same way a published KServe model will be.

| Shortcoming | What it causes | Fix |
| :---- | :---- | :---- |
| Per-model routing is an HTTPRoute, and ExtProc only *chooses* — Envoy *routes*. | Removing the route breaks provider selection even when the overlay is correct. Route reselection is an Istio/Envoy pattern, not a Praxis one. | Praxis owns model → provider routing end-to-end. Stop creating `external-model-*` HTTPRoutes. |
| Provider transport is Istio Service + ServiceEntry + DestinationRule. | External HTTPS, SNI, and egress policy live outside Praxis. A tenant cut over to Praxis still needs Istio for external models only. | Publish provider backends into Praxis config (cluster / upstream / egress story owned by the proxy). |
| `status.httpRouteName` is the contract MaaS uses to attach policies. | Without a route, MaaS has nowhere to hang per-model Kuadrant policies. Same class of problem as the KServe ADR. | Generate per-model limits and auth in the Praxis `Policy`. Rename or retire `httpRoute*` status fields. |
| Ready means "networking resources created" (route + transport), not "Praxis can serve". | A model can be Ready on Gateway API and invisible to the proxy, or the reverse during migration. | Ready means providers + credentials resolve and the routing entry was accepted into the Proxy (or equivalent), independent of HTTPRoute status. |
| Tenant identity and Gateway binding still gate ExternalModel reconcile. | Correct for today's ExtProc-on-Gateway install; over-coupled once the Gateway only forwards to Praxis. | Keep the tenant Gateway as entry (ADR non-goal to remove it). Resolve the tenant from `AITenant` / `MaasTenantConfig` as today; point the tenant at a `Proxy` the same way the KServe ADR does. |
| Credential material is projected for ExtProc file reads. | Works for ExtProc; Praxis needs an explicit secret / credential reference model in proxy config without embedding secret bytes in overlays (already a rule today). | Keep reference-only credentials; map them into Praxis credential_inject (or successor) configuration. |

## Goals

- An ExternalModel in a Praxis tenant is served through Praxis, not through a per-model HTTPRoute.
- Users keep writing `ExternalModel` / `ExternalProvider`. Clients keep the hostname, the request URL with the model in the body, and the model ID (`spec.modelName` or `metadata.name`).
- One writer per object. `ai-gateway-controller` owns ExternalModel control plane; `gateway-controller` owns rendered proxy config; MaaS owns tenant `Proxy` / `Policy` scaffolding and subscription-derived policy rules.
- Align with the KServe ADR: same tenant cut-over unit, same ingress-forward model, same "ModelProvider can come later" escape hatch.
- A tenant can move in stages and be rolled back. Preparation can start before every Praxis dependency is ready.

## Non-Goals

- The move to Praxis itself. It is the starting point and is decided by the platform / companion ADR.
- Changing the user-facing ExternalModel / ExternalProvider APIs beyond status fields that name HTTPRoutes.
- Defining the `ModelProvider` API. It can be adopted later without a user-facing change.
- An ingress without Gateway API. The tenant Gateway stays in front of the proxy.
- Per-model path URLs of the form `/<namespace>/<name>/v1/...` through Praxis (same non-goal as the companion ADR).
- Moving a single model. The unit of the cut-over is the tenant.
- Replacing MaaS ownership of auth subscriptions / API keys; only the attachment point moves from HTTPRoute to Praxis `Policy`.
- In-cluster KServe / `LLMInferenceService` routing (covered by the companion ADR).

## How

We propose to keep `ExternalModel` and `ExternalProvider` as the objects a user writes. `ai-gateway-controller` already knows the backends, provider bindings, weights, paths, and credential references. It stops creating Gateway API / Istio transport for Praxis-selected tenants and instead writes a routing entry for each Ready ExternalModel into the tenant's Praxis configuration next to the policy slot — preferred: **inline in the tenant `Proxy`**, with field ownership that does not fight MaaS's KServe routing writes. `gateway-controller` renders it. No `ModelProvider` is required for the first step.

Credential injection and multi-provider weighted selection move from ExtProc-on-Envoy into Praxis filters. The content-addressed overlay can remain as an internal distribution mechanism during migration, then collapse into whatever `gateway-controller` already mounts or hot-reloads for routing.

```
Today:   Client → Gateway/Envoy → ExtProc (choose + creds) → HTTPRoute reselection → SE/DR → provider
Target:  Client → Gateway (forward) → Praxis Proxy (route on model + creds + provider) → provider
```

### Who does what, and why

- The user writes the same `ExternalModel` / `ExternalProvider` as today. Spec needs no new required field for the first step.
- **`ai-gateway-controller`** resolves providers and Secrets, publishes Ready without an HTTPRoute, and writes / removes routing entries for each ExternalModel in the tenant Proxy (or creates `ModelProvider`s later). It deletes legacy HTTPRoute / Service / SE / DR for tenants on the Praxis path.
- **MaaS** creates the tenant `Proxy` and `Policy`, generates auth and per-model limits that do not attach to a route, and continues to own subscriptions. It stops requiring `status.httpRouteName` for ExternalModels.
- **`gateway-controller`** renders `Proxy` + `Policy`. It does not need the `ModelProvider` type for the first step.
- The tenant Gateway remains the entry hostname clients already use; it forwards to the proxy.

### Benefits

- **One dataplane per Praxis tenant.** External and in-cluster models share the same Proxy routing story.
- **One writer per object.** ExternalModel stays with ai-gateway-controller; Proxy scaffolding stays with MaaS; rendered config stays with gateway-controller.
- **No dependency on the `ModelProvider` API** for the first cut. Same rationale as the companion ADR.
- **Nothing changes in what users write.** Clients keep body-based model IDs.
- **Adopting `ModelProvider` later** needs no change for users: move the routing entry from inline Proxy fields to a provider object without changing ExternalModel spec.

## Changes in ai-gateway-controller

| Change | Today | Size |
| :---- | :---- | :---- |
| Stop creating `HTTPRoute` `external-model-*` for Praxis-selected tenants. | `modelHTTPRouteSet` always builds Gateway API routes with provider header matches and sink rules. | Medium |
| Stop creating provider Service / ServiceEntry / DestinationRule for those tenants. | `applyTransport` owns the external HTTPS hop. | Medium |
| Write routing entries into the tenant `Proxy` (SSA / field owner), or a temporary bridge ConfigMap that gateway-controller already understands. | Publishes ExtProc overlay only; Envoy owns the hop. | Medium / New |
| Redefine Ready: providers + credentials resolve and routing entry was applied; no HTTPRoute condition. | Ready ≈ networking resources created; `status.httpRouteName` set. | Small |
| Retire or replace `status.httpRouteName` with a Praxis-facing status (e.g. proxy ref, overlay/revision digest kept). | Consumers attach Kuadrant policies via the route name. | Small, API change |
| Keep reference-only credential publication; map into Praxis credential config. | ExtProc file projection + `credential_inject`. | Medium |
| Preserve weighted multi-provider selection semantics in Praxis routing. | Weights on `ExternalProviderRef`; ExtProc `intelligent_route` + header reselection. | Medium |
| Tenant gate stays: Praxis-selected Active `AITenant`; Gateway only needed as ingress forward target, not as HTTPRoute parent. | Reconcile no-ops / cleans up without Active Praxis tenant. | Small |

## Changes in MaaS

| Change | Today | Size |
| :---- | :---- | :---- |
| Stop attaching per-model policies to ExternalModel HTTPRoutes. | Uses `status.httpRouteName` (and similar patterns for KServe routes). | Medium |
| Generate per-model token limits and auth rules in the Praxis `Policy`. | Kuadrant policies on Gateway / HTTPRoute. | New (shared with companion ADR) |
| Create `Proxy` + `Policy` for an `AITenant`, record Proxy on the tenant. | AITenant names Gateway only. | New (shared with companion ADR) |
| Allow ExternalModel Ready without an HTTPRoute when the tenant is on Praxis. | Pending / blocked patterns that wait on routes. | Small |

## Changes in Praxis / gateway-controller

| Change | Today | Size |
| :---- | :---- | :---- |
| Route external providers from Proxy config (host, port, TLS/SNI, path rewrite). | External hop is Istio SE/DR behind ExtProc. | New |
| Multi-provider weighted selection + model body routing for ExternalModel IDs. | ExtProc `intelligent_route` overlay. | Exists in ExtProc; must exist in proxy |
| Credential injection from Secret references without secret bytes in config. | ExtProc `credential_inject`. | Exists in ExtProc; must exist in proxy |
| Render Proxy + Policy and deploy the proxy (shared dependency). | Draft / in flight on gateway-controller. | Shared with companion ADR |

## Dependencies

Delivering this decision depends on the same Praxis control-plane work as the companion ADR, plus ExternalModel-specific dataplane capabilities.

- `gateway-controller` renders and deploys the proxy from a `Proxy` and a `Policy` ([`gateway-controller#14`](https://github.com/praxis-proxy/gateway-controller/issues/14)–[`#17`](https://github.com/praxis-proxy/gateway-controller/issues/17), draft [`#82`](https://github.com/praxis-proxy/gateway-controller/pull/82)).
- Policy engine: Kubernetes token auth ([`policy#213`](https://github.com/praxis-proxy/policy/issues/213), [`#214`](https://github.com/praxis-proxy/policy/issues/214)), per-model token limits ([`policy#352`](https://github.com/praxis-proxy/policy/issues/352), [`#308`](https://github.com/praxis-proxy/policy/issues/308)), shared limit counters ([`policy#127`](https://github.com/praxis-proxy/policy/issues/127)).
- External HTTPS / egress and TLS to third-party providers must be expressible in Praxis upstream config (no open issue cited here — track explicitly).
- Credential injection and weighted provider selection must be available on the Praxis proxy path, not only on `praxis-extproc` attached to Istio.
- Companion ADR migration stages: tenant cut-over is shared; ExternalModel entries are additional Proxy content for the same tenant.

## Open Questions

- **Who writes ExternalModel routing into the `Proxy`?** Preferred here: `ai-gateway-controller`, with a distinct SSA field owner from MaaS's KServe routing writes, so each controller owns its entries in one object. Alternative: MaaS is the sole Proxy writer and reads a status contract from ExternalModel (mirrors the companion ADR more closely, but moves external-model knowledge into MaaS).
- **Bridge during migration.** Keep publishing the ExtProc overlay while HTTPRoutes still exist for IPP / hybrid tenants; drop overlay-only distribution once the proxy hot-reloads Proxy-derived config.
- **How the Gateway shows it is in front of Praxis.** Same open question as the companion ADR (GatewayClass vs annotation). ExternalModel reconcile needs the same signal to skip HTTPRoute creation.
- **Secret projection vs proxy-native credential refs.** Whether Praxis mounts credential files, uses SDS, or reads Kubernetes Secrets directly affects RBAC for the proxy ServiceAccount.
- **Price Tag / metering filters.** External metering today hangs off ExtProc config; confirm the filter moves with the proxy so ExternalModel traffic stays metered after HTTPRoute removal.

## Security and Privacy Considerations

- Authentication and authorization move from Kuadrant-on-HTTPRoute to Praxis `Policy`, generated from the same MaaS auth and subscription objects. Tenant cut-over waits on Kubernetes token auth in the policy engine (same gate as the companion ADR).
- Credential references must stay reference-only in any overlay or Proxy config. Secret bytes must not enter etcd via routing objects (already enforced for today's overlay).
- The proxy ServiceAccount needs carefully scoped access to provider credential Secrets (or a projection path owned by ai-gateway-controller).
- Egress to arbitrary external provider hostnames is a trust boundary. Today Istio SE makes that explicit; Praxis needs an equivalent allow-listed upstream definition per ExternalProvider.
- No new client data is stored. The proxy reads the request body for the model name, as ExtProc does today.

## Risks and Considerations

- All models of a tenant (KServe-published and ExternalModel) may land in one `Proxy` object; every publish/unpublish is a write to the same object (same risk as the companion ADR).
- SSA / field-ownership conflicts if both MaaS and ai-gateway-controller write the Proxy without a clear partition.
- When an ExternalProvider Secret rotates, Praxis must pick up new credentials without requiring an HTTPRoute reconcile.
- During hybrid migration, a tenant must not split a single client-facing model ID across ExtProc-on-Envoy and Praxis Proxy.
- Routing config rolls that restart proxy pods (companion ADR risk) affect external and in-cluster models together.

## Stakeholder Impacts

| Group | Impacted? |
| :---- | :---- |
| AI Gateway (`ai-gateway-controller`) | Stops HTTPRoute / SE / DR for Praxis tenants; writes Proxy routing (or ModelProvider later); redefines Ready / status. |
| Models-as-a-Service (MaaS) | Stops attaching policies to ExternalModel HTTPRoutes; owns Proxy / Policy scaffolding; shared cut-over with KServe path. |
| AI Gateway (Praxis) | Proxy must route to external HTTPS providers, inject credentials, and select among weighted providers. |
| Dashboard / users | Keep writing ExternalModel / ExternalProvider; same hostnames and body model IDs. Per-model path URLs stop working. |
| Model Serving (KServe) | None directly; companion ADR only. |

## References

- Companion ADR: Routing MaaS-published KServe models through Praxis (Approved 2026-11-20)
- [praxis-proxy/gateway-controller design](https://github.com/praxis-proxy/gateway-controller/blob/f6a12554d9294130010ed16abea775657914bb6f/docs/design/api.md)
- ai-gateway-controller `DESIGN.md` (Phase 2 ExternalModel + praxis-extproc)
- ODH-ADR-MS-0005 — External Model / Praxis two-plane control-plane split
