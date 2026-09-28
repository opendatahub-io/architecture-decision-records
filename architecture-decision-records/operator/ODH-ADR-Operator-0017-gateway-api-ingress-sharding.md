# ODH-ADR-Operator-0017: RHOAI platform ingress sharding

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-24 |
| Scope          | RHOAI Operator (platform ingress and network-zone segmentation) |
| Status         | Draft |
| Authors        | [Davide Bianchi](@davidebianchi) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-2272](https://redhat.atlassian.net/browse/RHAISTRAT-2272) |
| Other docs:    | [ODH-ADR-Operator-0012: Gateway API Authentication Architecture](ODH-ADR-Operator-0012-gateway-api-authentication-architecture.md) |

## What

Establish Ingress/Gateway API sharding as an RHOAI platform capability so supported platform
endpoints can be exposed through distinct OpenShift ingress shards and network zones, such as
separate network segments or dev/test/prod environments. This addresses the network-separation
gap created by RHOAI 3.x's single-domain, path-based exposure model.

## Why

RHOAI customers need platform endpoints to enter through distinct OpenShift IngressController
shards and network zones. RHOAI 3.x's single-domain, path-based exposure cannot express that
placement. Earlier manual Route relabeling was not durable because controllers reconcile Route
resources.

OpenShift selects a shard at the Route layer through IngressController `routeSelector`
configuration. The operator therefore needs a supported configuration contract for
administrator-managed ingress placement, with status that makes unexpected Route admission
visible while preserving the existing Gateway API authentication architecture and default
behavior.

## Goals

- Preserve existing single-ingress behavior when `additionalIngresses` is absent or empty.
- Allow administrators to expose supported platform endpoints through distinct ingress shards
  and network zones.
- Report unexpected IngressController admission so administrators can verify and adjust their
  intended network-zone placement.
- Keep browser authentication sessions and authentication availability independent per ingress.
- Let administrators assign namespaces to an ingress; have the notebook controller report invalid
  or unready assignments without falling back to the default ingress.
- Preserve correct ingress assignment and availability through configuration changes, upgrades,
  and rollback.

## Non-Goals

- Automatically enabling every route-producing component to consume additional ingresses.
  Model-serving, KServe, Dashboard, and other producers remain on the default listener until
  separately integrated with the assignment contract.
- Creating additional Gateway or GatewayClass resources in the initial implementation.
- Installing or modifying IngressControllers, the Gateway provider, or external identity
  providers. These remain administrator-managed.
- Supporting direct LoadBalancer exposure, a generic Kubernetes provider, or non-`OcpRoute`
  ingress modes.
- Per-workbench ingress selection, Dashboard UI changes, or new public DSC/DSCI fields.
- Redis/Valkey session storage or audience-bound bearer-token validation.

## How

### Configuration and traffic flow

The existing `GatewayConfig/default-gateway` remains the administration contract for the
default ingress and additional named ingresses. Administrators configure each ingress's
hostname, intended IngressController, and required authentication identity; the RHOAI
operator validates the configuration and reports per-ingress status. The `additionalIngresses`
API describes an ingress, not a Gateway topology: it can support a different backing
architecture later. An absent or empty `additionalIngresses` preserves the existing default
path. No new public DSC/DSCI API is required.

OpenShift IngressControllers select traffic at the Route layer. For each additional ingress,
the operator creates a Route intended for the configured IngressController and connects it to
the existing managed Gateway, which continues to provide Gateway API routing for platform
endpoints. The initial implementation uses reencrypt Routes and distinct internal listener
ports on the shared Gateway Service; clients still use HTTPS port 443 externally. This avoids
managing another Gateway and data plane per ingress. Gateway API limits a Gateway to 64
listeners: additional ingresses cannot exceed 64 minus the listeners already required by the
default ingress and other Gateway functions. The supported total may be lower if the provider
or cluster capacity imposes tighter limits.

The operator rejects conflicting ingress identities, hostnames, Route labels, client IDs, or
listener ports and unsupported ingress/authentication combinations before reconciling.
Internal listener ports must not collide with default, health, metrics, reserved, or sibling
ports. Configuration validation must enforce the Gateway listener budget and proxy scaling
bounds. CRD schema/CEL rules cover what they can; a validating webhook may be needed for
constraints that cannot be expressed in the schema. The initial configuration uses an
administrator-supplied `listenerPort`, but the `additionalIngresses` contract should not
require that port if a future Gateway topology no longer needs one.

### Shard binding and availability

Cluster administrators configure IngressController domains and Route selectors, including
which controllers admit each shard Route. The RHOAI operator does not manage those controllers.
It creates and continues reconciling the Route even when no controller or multiple controllers
admit it, or when admission differs from the configured target or hostname domain. It compares
IngressController configuration and Route admission status with the intended placement and
reports discrepancies as warnings in per-ingress status; it does not remove or withhold the
Route because administrators may intentionally configure overlapping selectors. Administrators
remain responsible for deciding whether the observed placement meets their network policy.

`GatewayConfig` reports listener, Route, authentication, and overall readiness per ingress.
Placement warnings are distinct from failures to program a listener, admit a Route, or make
authentication available: they do not by themselves block publication, assignment, or
readiness. Other ingresses remain independently reportable.

### Authentication boundary

The operator provides a separate `kube-auth-proxy`, OAuth/OIDC client identity, cookie Secret,
callback route, and readiness state for each additional ingress. Authorization on the shared
Gateway directs each ingress to its corresponding proxy; callbacks return to that ingress
without re-entering authorization. All ingresses use the top-level `GatewayConfig`
authentication mode and provider settings. In OIDC mode, additional ingresses use distinct
client IDs and client-secret references; in OpenShift OAuth mode, the operator provisions a
separate OAuthClient for each. Mixed modes and per-ingress issuer overrides are unsupported.

Browser sessions use encrypted cookies and remain separate across ingresses. Replicas serving
one ingress share its cookie Secret. Cluster-valid Kubernetes/OpenShift bearer tokens remain
accepted at every proxy when `enableK8sTokenValidation=true`; browser-session separation
does not imply bearer-token audience isolation.

### Component integration, assignment, and status

`GatewayConfig` is the source of ingress configuration and status. The RHOAI operator projects
the ingress information needed for routing into each integrated component CR. Each component
uses that data to expose its routes through the selected ingress and reports assignment
problems in its own status.

Cluster administrators select an ingress for a namespace with the `opendatahub.io/ingress-name`
annotation. Without it, components use the default ingress. If the selected ingress does not
exist, the component sets an unresolved-ingress status condition on affected resources; it
does not fall back to default.

Workbenches is the first integration: the RHOAI operator projects ingresses into the
`Workbenches` CR, retaining `spec.gatewayDomain` for the default. The notebook controller
uses the projection for Notebook routes and reports `IngressResolved=False` with reason
`IngressNotFound` when the selected ingress is missing.

### Lifecycle and ownership

The RHOAI operator manages ingress infrastructure and projections into component CRs. Each
component operator manages its routes and assignment status. Administrator-managed
IngressControllers, external OIDC registrations, and supplied Secrets remain untouched.

On ingress removal, these controllers reconcile independently; cleanup order is not
guaranteed. The RHOAI operator removes its ingress resources and projection; component
operators remove routes for missing ingresses and report unresolved assignments without
falling back to default.

When no additional ingresses are configured, namespaces without an ingress annotation continue
using the default ingress. Annotated namespaces still resolve the selected name and report an
error if it is missing.

## Open Questions

- What quantitative detection or availability target should apply when IngressController
  selector or domain configuration drifts?
- Does TLS certificate rotation across multiple listeners require a coordinated rollout?
- What selector complexity and cluster scale require qualification beyond the planned one-,
  two-, and four-listener test topologies? These topologies are test coverage, not the Gateway
  API limit.

## Alternatives

### Keep only the default listener

This preserves the current architecture and avoids additional resources, but cannot route
workbench traffic through distinct IngressController shards. It does not meet the customer
requirement.

### Create one Gateway per ingress

Multiple Gateways could provide separate data planes per ingress, reducing shared Envoy
failures and allowing stronger NetworkPolicy separation between ingress data planes and auth
proxies. They also require more Gateway, Envoy, TLS, and authentication lifecycle management.
The initial implementation reuses the existing Gateway with distinct listeners to reduce
delivery effort; live ROSA validation confirmed the Route-to-Gateway bridge. The public
`additionalIngresses` API names ingresses rather than Gateways so this topology can be
revisited without changing the ingress concept. Internal listener ports remain a current
implementation constraint, not the long-term ingress abstraction.

### Select listeners with public-host SNI

This avoids allocating an internal listener port per ingress, but live ROSA validation showed
that reencrypt Route termination does not preserve usable public-host SNI for backend listener
selection. Unique internal ports and listener-specific filters were validated instead.

### Share one authentication proxy across additional ingresses

A shared proxy reduces resource overhead, but couples callback routing, credentials, cookies,
and readiness. Independent proxies and client identities provide per-ingress browser-session
isolation and fault reporting; the additional resource cost is accepted.

## Security and Privacy Considerations

- IngressController selectors and domain configuration determine Route-layer placement. The
  operator reports admission discrepancies but does not enforce exclusive admission; a Route
  admitted by another controller may be exposed outside the intended network zone until the
  administrator corrects its selectors or domain configuration. These warnings are advisory,
  not a guarantee of network-zone isolation.
- Each ingress uses a distinct OAuth/OIDC client identity, cookie Secret, proxy, callback
  route, and NetworkPolicy. These isolate browser sessions and proxy access.
- All listeners share one Envoy workload. A shared Envoy, Gateway Service, certificate, or
  EnvoyFilter failure can affect every ingress; per-ingress NetworkPolicies cannot isolate
  listeners inside that pod.
- Cluster-valid bearer tokens remain accepted by each proxy when token validation is enabled.
  Document this limitation; do not describe browser-session isolation as full credential
  isolation.
- RHOAI does not manage customer IngressController or external identity-provider state.

## Risks

- **Shared Gateway failure domain:** Envoy, Gateway Service, TLS, or filter errors can affect
  every ingress. Generate and validate the complete filter set atomically, restore the last
  accepted configuration on rejection, and report per-ingress readiness.
- **IngressController drift:** Selector or domain changes can expose a Route through an
  unintended controller or leave it unadmitted until the administrator corrects the change.
  Report unexpected admission as a warning using IngressController configuration and Route
  status; document that warnings do not block exposure or guarantee network-zone isolation.
- **Gateway listener capacity:** Gateway API permits at most 64 listeners per Gateway, including
  existing listeners. Validate available slots before adding ingresses and report provider
  limits or capacity constraints that reduce the usable count.
- **Proxy resource pressure:** Each ingress adds an independently scaled auth proxy. Enforce
  HPA bounds and report authentication readiness when required replicas are unavailable.
- **Cross-repository version skew:** An incompatible Workbenches projection can break route
  assignment. Gate activation on compatible operand readiness and require ordered rollback.
- **Authentication prerequisites:** Missing credentials or provider configuration can leave
  an ingress unavailable. Report `AuthenticationUnavailable` and do not mark it ready.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| :---- | :---- | :---- | :---- |
| RHOAI Operator / AICP | Heimdall | 2026-09-24 | Yes |
| Workbenches / notebook controller | Pewter Scrum | 2026-09-24 | Yes |
| OpenShift networking administrators | Cluster administrators | 2026-09-24 | Yes |
| Security and identity providers | Platform Security, customer IdP administrators | 2026-09-24 | Yes |
| QE and documentation | QE, Documentation | 2026-09-24 | Yes |

Notes:

- AICP owns `GatewayConfig`, listener and Route reconciliation, authentication resources,
  validation, status, cleanup, and the Workbenches projection.
- The Workbenches team owns namespace assignment, HTTPRoute updates, Notebook status, RBAC,
  and controller packaging.
- Cluster administrators configure IngressController shards and external identity-provider
  clients. QE and Documentation qualify and explain the supported topology and prerequisites.

## References

- [RHAISTRAT-2272: RHOAI support for Ingress / Gateway API sharding](https://redhat.atlassian.net/browse/RHAISTRAT-2272)
- [RHAIRFE-953: Customer requirement](https://redhat.atlassian.net/browse/RHAIRFE-953)
- [RHOAIENG-35058: Related engineering ticket](https://redhat.atlassian.net/browse/RHOAIENG-35058)
- [ODH-ADR-Operator-0012: Gateway API Authentication Architecture](ODH-ADR-Operator-0012-gateway-api-authentication-architecture.md)
- [OpenShift Gateway API deployment topologies](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ingress_and_load_balancing/configuring-gateway-api#gateway-api-deployment-topologies_understand-gateway-api)
- [Gateway API Gateway CRD (`spec.listeners` maximum: 64)](https://github.com/kubernetes-sigs/gateway-api/blob/main/config/crd/standard/gateway.networking.k8s.io_gateways.yaml)

## Reviews

| Reviewed by | Date | Notes |
| :---- | :---- | :---- |
