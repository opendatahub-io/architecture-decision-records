# ODH-ADR-Operator-0018: RHOAI platform ingress sharding

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-24 |
| Scope          | RHOAI Operator (platform ingress and network-zone segmentation) |
| Status         | Draft |
| Authors        | [Davide Bianchi](@davidebianchi), [Lindani Phiri](@lphiri) |
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
- Preserve existing default Gateway and HTTPRoute parent references when additional ingresses
  are configured.
- Allow administrators to expose supported platform endpoints through distinct ingress shards
  when administrator-managed IngressController selectors enforce the intended network-zone
  placement.
- Report unexpected IngressController admission so administrators can verify and adjust their
  intended network-zone placement.
- Keep browser authentication sessions and authentication availability independent per ingress.
- Let authorized namespace metadata writers assign namespaces to an ingress; have the notebook
  controller report invalid or unready assignments without falling back to the default ingress.
- Preserve correct ingress assignment and availability through configuration changes, upgrades,
  and rollback.

## Non-Goals

- Automatically enabling every route-producing component to consume additional ingresses.
  Model-serving (including KServe), Dashboard, and other producers continue using the default
  ingress and its Gateway until separately integrated with the assignment contract.
- Creating an additional GatewayClass for each ingress.
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

OpenShift IngressControllers select traffic at the Route layer. The existing managed Gateway,
Service, Route, and listeners remain the default ingress. For each additional ingress, the
operator creates a separate named Gateway using the existing GatewayClass and an HTTPS
listener on port 443. The provider provisions its Gateway Service; the operator creates a
reencrypt Route intended for the configured IngressController and targeting that Service's
HTTPS port, not the default Gateway Service. Clients use HTTPS port 443 externally. Each
existing default HTTPRoute has a `parentRefs` entry naming the default Gateway; additional
Gateways have different names. Omitting `sectionName` or `hostnames` can allow an HTTPRoute
to attach to multiple listeners of its named Gateway, but not to a different Gateway. Thus
existing default HTTPRoutes require no changes.

The operator rejects conflicting ingress names, hostnames, Route labels, or client IDs and
unsupported ingress/authentication combinations before creating or changing ingress resources.
It reports Gateway, Route, and authentication readiness per ingress. Configuration validation
must enforce proxy scaling bounds; per-ingress status reports Gateway provisioning failures if
provider capacity is exhausted. CRD schema/CEL rules cover what they can; a validating webhook
may be needed for constraints that cannot be expressed in the schema.

### Shard binding and availability

Cluster administrators configure IngressController domains and Route selectors, including
which controllers admit each shard Route. The RHOAI operator does not manage those controllers.
It creates and continues reconciling the Route even when no controller or multiple controllers
admit it, or when admission differs from the configured target or hostname domain. It compares
IngressController configuration and Route admission status with the intended placement and
reports discrepancies as warnings in per-ingress status; it does not remove or withhold the
Route because administrators may intentionally configure overlapping selectors. Administrators
remain responsible for deciding whether the observed placement meets their network policy.
Separate Gateways prevent HTTPRoute attachment to the wrong Gateway, but do not prevent an
IngressController from admitting the wrong OpenShift Route.

`GatewayConfig` reports listener, Route, authentication, and overall readiness per ingress.
Placement warnings are distinct from failures to program a listener, admit a Route, or make
authentication available: they do not by themselves block publication, assignment, or
readiness. Other ingresses remain independently reportable.

### Authentication boundary

The operator provides a separate `kube-auth-proxy`, OAuth/OIDC client identity, cookie Secret,
callback HTTPRoute, and readiness state for each additional ingress. It creates a new callback
HTTPRoute per additional Gateway, with a `parentRefs` entry naming that Gateway and a backend
reference to that ingress's proxy Service. The existing default callback HTTPRoute remains
unchanged on the default Gateway. Authorization on each Gateway directs requests to its proxy;
callbacks return through that Gateway without re-entering authorization. Authentication
filters must select only their intended Gateway workload so adding an ingress cannot redirect
default traffic to another proxy. All ingresses use the top-level `GatewayConfig`
authentication mode and provider settings. When the top-level authentication mode is OIDC, the
default ingress and every additional ingress MUST use unique client IDs and unique client-secret
Secret references across the complete ingress set. Reuse between the default ingress and an
additional ingress, or between additional ingresses, is invalid. The operator reports the
affected ingress configuration as invalid and does not reconcile its authentication resources.
Secret-reference uniqueness applies to the reference (`namespace/name`), not secret data values.
In OpenShift OAuth mode, the operator provisions a separate OAuthClient for each ingress.
Mixed modes and per-ingress issuer overrides are unsupported.

Browser sessions use encrypted cookies and remain separate across ingresses. Replicas serving
one ingress share its cookie Secret. Cluster-valid Kubernetes/OpenShift bearer tokens remain
accepted at every proxy when `enableK8sTokenValidation=true`; browser-session separation
does not imply bearer-token audience isolation.

### Component integration, assignment, and status

`GatewayConfig` is the source of ingress configuration and status. For each integrated
component, the RHOAI operator projects the ingress information that component needs: a
Gateway reference to attach HTTPRoutes and, when it constructs public URLs, the ingress
hostname. Components not integrated with additional ingresses continue using the default.

Workbenches is the first integration. The RHOAI operator projects additional Gateway
references and hostnames into the `Workbenches` CR: the notebook controller uses the Gateway
reference for Notebook HTTPRoutes and the hostname to construct workbench URLs. It retains
`spec.gatewayDomain` and existing HTTPRoutes for default assignments. No other component's
HTTPRoutes need changing for this integration.

Administrators intend to select an ingress for a namespace with the
`opendatahub.io/ingress-name` annotation. Kubernetes RBAC governs who can modify namespace
metadata; the annotation does not establish who made the assignment. Without it, components
use the default ingress. If the selected ingress does not exist, the component reports the
unresolved or unready assignment on affected resources; it does not fall back to default.

The notebook controller reports `IngressResolved=False` with reason `IngressNotFound` when
the selected ingress is missing.
Qualification must verify distinct Gateway Services; default Dashboard and authentication
callback HTTPRoutes must work through the default Gateway but not through additional Gateways,
and additional Notebook and callback HTTPRoutes only through their assigned Gateways.

### Lifecycle and ownership

The RHOAI operator manages the default Gateway without changing its route attachment contract,
and manages additional Gateways, Routes, authentication resources, and projections into
module CRs. The Gateway provider provisions the Service for each Gateway. Each component
operator manages its HTTPRoutes and assignment status. Administrator-managed IngressControllers,
external OIDC registrations, and supplied Secrets remain untouched.

On ingress removal, these controllers reconcile independently; cleanup order is not
guaranteed. The RHOAI operator removes that ingress's Gateway, Route, authentication resources,
and projection; component operators remove HTTPRoutes for missing ingresses and report
unresolved assignments without falling back to default. The default Gateway and HTTPRoutes
remain intact throughout.

When no additional ingresses are configured, namespaces without an ingress annotation continue
using the default ingress. Annotated namespaces still resolve the selected name and report an
error if it is missing.

## Open Questions

- What quantitative detection or availability target should apply when IngressController
  selector or domain configuration drifts?
- Does TLS certificate rotation across multiple Gateways require a coordinated rollout?
- Does the OpenShift Gateway provider keep data planes distinct across Gateways, and how are
  authentication filters scoped to each Gateway workload?
- What maximum Gateway count can each supported OpenShift provider and cluster topology
  sustain? Planned one-, two-, and four-Gateway topologies are qualification cases only; they
  do not define a supported maximum.

## Alternatives

### Keep only the default Gateway

This preserves the current architecture and avoids additional resources, but cannot route
workbench traffic through distinct IngressController shards. It does not meet the customer
requirement.

### Share one Gateway across ingresses

One Gateway with distinct internal listeners reduces Gateway and data-plane resources. Live
ROSA validation confirmed the reencrypt Route-to-Gateway bridge using unique backend ports
and listener-specific filters. However, a default HTTPRoute that references the whole Gateway
without `sectionName` can attach to every compatible listener. Existing default Dashboard and
authentication callback HTTPRoutes do not restrict their parent references to a listener; in
`OcpRoute` mode the Gateway listeners also have no hostnames to constrain attachment. Adding
listeners would therefore risk exposing default routes through additional ingress shards.
Fixing this requires migrating every default and future HTTPRoute producer to explicit
`sectionName`, coordinating cross-repository upgrades, and preventing unconstrained routes
from attaching later. This conflicts with preserving existing default Routes. Separate Gateway
parents avoid that migration and, if the provider provisions separate workloads, reduce the
shared data-plane failure domain. The extra Gateway, Service, TLS, filter, and authentication
lifecycle and resource cost is accepted. The public `additionalIngresses` API describes
ingresses rather than fixing their backing Gateway topology forever.

### Select shared-Gateway listeners with public-host SNI

This would avoid allocating an internal listener port per ingress in the shared-Gateway
alternative, but live ROSA validation showed that reencrypt Route termination does not
preserve usable public-host SNI for backend listener selection. It would also leave default
HTTPRoutes without listener-specific parent references.

### Share one authentication proxy across additional ingresses

A shared proxy reduces resource overhead, but couples callback routing, credentials, cookies,
and readiness. Independent proxies and client identities provide per-ingress browser-session
isolation and fault reporting; the additional resource cost is accepted.

## Security and Privacy Considerations

- IngressController selectors and domain configuration determine Route-layer placement. The
  operator reports admission discrepancies but does not enforce exclusive admission; a Route
  admitted by another controller may be exposed outside the intended network zone until the
  administrator corrects its selectors or domain configuration. These warnings are advisory,
  not a guarantee of network-zone isolation. Before using an additional ingress for network
  separation, administrators must configure exclusive selectors, check which controllers admit
  the Route, and monitor for drift. DNS configuration alone does not prevent exposure through
  an unintended controller.
- Namespace ingress assignment trusts any principal with effective `update` or `patch` access
  to core `namespaces`. The dashboard ServiceAccount, for example, currently has cluster-wide
  namespace `patch` permission. Neither the operator nor the notebook controller can tell from
  the annotation whether an administrator made the assignment. Administrators must review
  namespace-writing RBAC when relying on assignments for placement; the annotation itself
  is not an authorization boundary.
- Each ingress uses a distinct OAuth/OIDC client identity, cookie Secret, proxy, callback
  route, and NetworkPolicy. The separate clients and cookie Secrets keep browser sessions
  independent.
- Default HTTPRoutes reference only the default Gateway; additional-ingress HTTPRoutes and
  authentication callbacks must reference only their assigned Gateways. Verify on supported
  OpenShift versions whether Gateway Services and Envoy pods are separate, and whether
  NetworkPolicies select only the intended pods. If pods are shared, do not claim data-plane
  or NetworkPolicy isolation between ingresses. Gateways still share a GatewayClass, controller,
  and cluster dependencies; separate Gateway objects alone do not guarantee network-zone
  isolation.
- Cluster-valid bearer tokens remain accepted by each proxy when token validation is enabled.
  Document this limitation; do not describe browser-session isolation as full credential
  isolation.
- RHOAI does not manage customer IngressController or external identity-provider state.

## Risks

- **Gateway provider and configuration failures:** Each additional Gateway adds Service,
  data-plane, TLS, and filter lifecycle. Verify that each Route reaches its Gateway's Service
  and each authentication filter targets only its intended Gateway; scope updates and readiness
  per ingress. Shared data planes, GatewayClass/controller, or cluster dependencies can still
  affect multiple ingresses.
- **IngressController drift:** Selector or domain changes can expose a Route through an
  unintended controller or leave it unadmitted until the administrator corrects the change.
  Report unexpected admission as a warning using IngressController configuration and Route
  status; administrators must investigate and correct selector or domain drift. Warnings do
  not block exposure or guarantee network-zone isolation.
- **Gateway capacity:** Additional Gateways consume provider, Service, Envoy, and cluster
  resources. If Gateway provisioning fails because provider capacity is reached, report the
  failure in that ingress's status; keep other ingresses independently reportable. Supported
  Gateway maximum remains to be determined through provider and scale qualification.
- **Proxy resource pressure:** Each ingress adds an independently scaled auth proxy. Enforce
  HPA bounds and report authentication readiness when required replicas are unavailable.
- **Cross-repository version skew:** An incompatible Workbenches projection can break route
  assignment. Gate activation on compatible operand readiness and require ordered rollback;
  default HTTPRoutes must remain attached to the default Gateway throughout.
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

- AICP owns `GatewayConfig`, Gateway and Route reconciliation, authentication resources,
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
- [Gateway API ParentReference (`sectionName` and whole-Gateway attachment)](https://github.com/kubernetes-sigs/gateway-api/blob/main/apis/v1/shared_types.go)
- [Current Dashboard HTTPRoute (default Gateway parent reference)](https://github.com/opendatahub-io/odh-dashboard/blob/858d53dc6db3a62b4b5e326562ecf9928d10335b/manifests/base/httproute.yaml)
- [Current default authentication callback HTTPRoute](https://github.com/opendatahub-io/opendatahub-operator/blob/f2bbf2852006329badb18bae2e19a3dddc09df97/internal/controller/services/gateway/resources/kube-auth-proxy-httproute.tmpl.yaml)
- [OpenShift Route API (admission status and target port)](https://github.com/openshift/api/blob/master/route/v1/types.go)
- [OpenShift IngressController API (route and namespace selectors)](https://github.com/openshift/api/blob/master/operator/v1/types_ingresscontroller.go)

## Reviews

| Reviewed by | Date | Notes |
| :---- | :---- | :---- |
