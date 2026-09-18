# RHOAI Module NetworkPolicy Platform Contract

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-09 |
| Scope          | Open Data Hub Operator and integrated RHOAI modules |
| Status         | Draft |
| Authors        | [Davide Bianchi](@davidebianchi) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-90747](https://issues.redhat.com/browse/RHOAIENG-90747) |
| Other docs:    | N/A |

## What

This ADR defines the NetworkPolicy platform contract between `opendatahub-operator`, RHOAI module operators, and components. It separates policies delivered with a module operator bundle from policies created for module-managed operands, and assigns one lifecycle manager to each class.

The contract defines required policy manifests, selector ownership, reconciliation behavior, RBAC, validation, and the boundary of platform support. The terms MUST, MUST NOT, SHOULD, and MAY are normative.

## Why

RHOAI modules need predictable, least-privilege network isolation. Module operators and their operands have different deployment lifecycles, so a single NetworkPolicy management model creates ambiguity and can cause competing reconcilers or premature cleanup.

The platform must guarantee that module-bundle policies are deployed and cleaned up with the operator bundle. Each module operator must remain responsible for policies covering its own operands, where it has the required lifecycle and endpoint context.

This contract also establishes a baseline for complying with applicable OCP 5 NetworkPolicy requirements and network-security guidance, while remaining usable on generic Kubernetes and other supported OpenShift releases.

## Goals

- Define a clear ownership and lifecycle boundary for module-bundle and operand policies.
- Require least-privilege ingress for module operators and exposed operands.
- Ensure operand policies are declaratively reconciled and recreated after deletion.
- Make NetworkPolicy RBAC part of every module's deployment contract.
- Define bundle and runtime validation responsibilities.

## Non-Goals

- Define one identical ingress rule set for every module.
- Require `opendatahub-operator` to create or reconcile operand policies.
- Permit allow-all policies as a substitute for an explicit traffic design.
- Require or prescribe egress isolation for every component.
- Define a shared API or Custom Resource for NetworkPolicy configuration.
- Define the multi-tenancy security model.
- Define controls for secondary networks, host networking, or application-layer authorization.
- Define or modify labels on workloads not owned by the component.

## How

### Policy classes and ownership

The contract defines two policy classes:

| Policy class | Target | Accountable team | Declared by | Lifecycle manager |
| --- | --- | --- | --- | --- |
| Module-bundle policy | Module operator pods, including metrics or webhook endpoints | Component team | Module Helm chart or Kustomize bundle | `opendatahub-operator` through normal module deployment and cleanup |
| Operand policy | Module-managed operand pods and services | Component team | Module or component controller reconciliation | Responsible module or component controller |

Accountability remains with the component team in both cases. Lifecycle management describes which controller applies and removes the Kubernetes object; it does not transfer policy design responsibility.

### Module-bundle policies

Every module Helm chart or Kustomize bundle MUST contain at least one `networking.k8s.io/v1` `NetworkPolicy` for its module operator. The policy MUST:

- Be rendered by the same Helm chart or Kustomize source as the module operator.
- Render into the module operator namespace.
- Select module operator pods using specific, stable labels.
- Declare explicit `policyTypes`.
- Declare explicit ingress sources and ports for every allowed path.
- Use specific pod selectors for operator allow policies; a documented namespace-wide default-deny policy is permitted.
- Be tested in the module's supported deployment configuration.

If the module operator accepts no ingress traffic, its policy MUST declare `policyTypes: [Ingress]` with no ingress rules. Multiple policies are allowed, for example separate policies for metrics and webhook traffic.

`opendatahub-operator` installs bundle policies with the module operator resources, manages their rendered state, and removes them during module-resource cleanup. The module operator MUST NOT reconcile the same bundle policy. It may read or watch the object when required, but must not compete with the platform field manager or assign a second controller owner.

### Operand policies

Every module-managed workload that accepts traffic through a Service, webhook, sidecar, user-facing endpoint, or documented pod-to-pod listener MUST be selected by an operand NetworkPolicy controlling that ingress. A module MUST NOT rely only on a default namespace policy.

Ingress policies MUST restrict the following endpoint types to their documented peers and ports:

| Endpoint | Required ports |
| --- | --- |
| Webhook | Webhook port only |
| Metrics | Explicit metrics port(s) only |
| Gateway-routed API or user endpoint | Advertised endpoint port(s) only |
| Internal API | Declared endpoint port(s) only |

Component teams MUST define and document the concrete target selectors, allowed source peers, and ports for each exposed endpoint. The target `spec.podSelector` MUST use stable labels applied to workloads owned by the component. Source selectors for pod-based callers MUST use stable labels owned by the calling service. A namespace-only peer is permitted when that namespace is dedicated to the approved caller and the broader access is documented. A CIDR peer is permitted only when the component verifies that it represents the actual traffic source in its supported deployment configuration. This ADR does not define a shared mapping for gateway or API-server peers. Placeholders and unresolved source identities are non-compliant.

Webhook policies MUST permit admission traffic only on the webhook port and use the documented, verified representation of the Kubernetes API-server path. Controllers MUST NOT derive selectors from currently running pods or add labels to workloads they do not own.

Static operand policies placed only in a platform Kustomize overlay or Helm chart do not satisfy this contract. Modules using [ODH Platform Utilities](https://github.com/opendatahub-io/odh-platform-utilities) SHOULD use its deploy, dynamic ownership, and garbage-collection actions to implement the reconciliation and lifecycle requirements below.

The following controller-runtime snippets are illustrative. They explain the required ownership and watch semantics but do not prescribe controller setup when the shared framework provides equivalent behavior. For example, a controller responsible for the operand lifecycle could register an ownership watch with:

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&Operand{}).
    Owns(&networkingv1.NetworkPolicy{}).
    Complete(r)
```

`Owns` configures the watch but does not establish the Kubernetes owner reference. When the operand can be a valid owner of the namespaced policy, equivalent low-level controller-runtime code sets that reference before creating or applying the policy and propagates any error:

```go
if err := controllerutil.SetControllerReference(operand, policy, r.Scheme); err != nil {
    return ctrl.Result{}, err
}
```

If a valid owner reference cannot be used, such as when owner and policy scopes differ, the controller MUST use an explicit watch mapping and MUST delete the policy during operand cleanup. In either case, policy deletion MUST enqueue reconciliation.

During normal operand reconciliation, the responsible module or component controller MUST:

- Create policies.
- Recreate policies after accidental deletion.
- Update policies when desired configuration changes.
- Delete policies when the owning operand is removed or disabled.
- Use declarative apply semantics so manual changes do not permanently diverge from desired state.

The operand policy lifecycle MUST follow the operand lifecycle. The platform MUST NOT create, update, or garbage-collect operand policies.

> **Note:** This contract covers traffic on the primary Kubernetes pod network.
> Components using secondary network attachments, such as SR-IOV,
> RDMA/RoCE, or Ray Clusters configured on a secondary interface, MUST document and test
> applicable network-specific controls. Standard `NetworkPolicy` coverage MUST
> NOT be assumed for that traffic.

### User-facing operands

User-facing operands are included in the operand-policy contract. The fact that a user creates the operand does not make the user responsible for its NetworkPolicy.

The component controller responsible for integrating the operand's generated workloads and platform networking MUST create and reconcile their NetworkPolicies. This may be the workload controller itself, a dedicated component controller, or a companion platform-integration controller. That controller owns the policy's desired state even when another controller creates the generated Pods, Deployments, StatefulSets, or Services.

The responsible policy controller MUST have NetworkPolicy RBAC, declare ownership or an equivalent watch relationship, and make policy cleanup follow the user-facing operand lifecycle. If an upstream workload controller cannot reconcile the required policy, the RHOAI component integration controller MUST provide that reconciliation. `opendatahub-operator` MUST NOT create or reconcile operand policies.

User-facing ingress policies MUST cover endpoint paths exposed by the operand, including approved user or gateway ingress, component-to-operand traffic, and monitoring. A policy MAY select all Pods generated by one operand, including its replicas, but policies MUST NOT be shared across independently managed operands. A namespace-wide allow policy is not sufficient. For module-to-module traffic, the destination module owns ingress; the caller may manage optional egress, but neither module manages the other's policies.

### Policy content requirements

Every component allow policy MUST target a specific pod set:

- `spec.podSelector` MUST contain meaningful labels.
- `podSelector: {}` MUST NOT be used for a component allow policy. It MAY be used for a documented namespace-wide default-deny policy.
- Stable `app.kubernetes.io/*` labels or an equivalent module-owned label contract SHOULD be used.

Ingress allow rules MUST be explicit. Each ingress allow rule MUST contain a non-empty `from` list with specific NetworkPolicy peers and a non-empty `ports` list. Pod-based callers SHOULD be restricted by both namespace and pod selectors. A namespace-wide default-deny policy MAY use `ingress: []`. These patterns are prohibited for component allow policies:

```yaml
ingress:
- {}
```

```yaml
from: []
```

Omitting `from` or `ports`, using an empty list, or otherwise allowing unrestricted ingress is non-compliant for an ingress allow rule.

### Optional egress restrictions

This ADR does not require every component to restrict egress. A component that does not manage egress MUST omit `Egress` from the `policyTypes` of its policies; it MUST NOT add an allow-all egress rule.

When a component manages egress, its egress policy MUST select the intended workloads with meaningful, stable labels and include `Egress` in `policyTypes`. Egress allow rules MUST contain non-empty `to` and `ports` lists and allow only documented destinations. An egress default-deny policy MAY use `egress: []`. In-cluster destinations SHOULD use stable namespace and pod selectors, while external destinations with stable address ranges MAY use the narrowest applicable CIDRs. Standard Kubernetes NetworkPolicy cannot select DNS names or reliably represent dynamically addressed services. Components that depend on such services, including external object storage or managed databases, MAY leave egress unmanaged or use separately governed network-specific controls outside this contract.

### Controller RBAC and configuration

The module or component controller's ClusterRole or namespace-scoped Role MUST include the NetworkPolicy permissions required for its operand-policy scope. The `opendatahub-operator` ClusterRole or namespace-scoped Role MUST include the NetworkPolicy permissions required for its module-bundle policy scope:

```yaml
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
```

Permissions MUST be present in each controller's rendered Helm or Kustomize manifests. Platform RBAC does not replace controller RBAC. Namespace-scoped permissions SHOULD be used when policies are confined to one namespace. Each controller using cluster-scoped permissions MUST define its NetworkPolicy authority set and reject reconciliation outside it. Runtime validation for each controller MUST use its actual ServiceAccount and cover an authorized namespace and a namespace outside that authority set.

Modules MAY expose policy configuration through a resource appropriate to their use case. This ADR does not prescribe a shared Custom Resource or schema. Configured ingress or egress rules MUST be additive to the controller-owned policy baseline and meet the selector, peer, and port requirements in this ADR. A NetworkPolicy not reconciled by the responsible controller is an environmental override: it neither replaces the required component policy nor becomes part of its lifecycle.

Required ingress policy protection MUST remain enabled throughout the operand lifecycle. Components MUST NOT offer configuration that deletes a required ingress policy or stops its reconciliation.

### Lifecycle semantics

Module-bundle policies follow the module operator resource lifecycle:

1. The platform renders and applies the policy with module operator resources.
2. Platform field management and Server-Side Apply manage rendered policy changes.
3. When a module is disabled, the platform requests removal of the module CR and keeps module operator resources available until removal completes.
4. The responsible module or component controller MUST complete operand-policy cleanup or verify garbage collection before module CR removal completes.
5. After the module CR is fully removed, the platform removes rendered module resources, including bundle policies.

Operand policies follow the module CR and operand lifecycle:

1. The responsible module or component controller creates them and establishes an owner reference or explicit lifecycle mapping for the operand.
2. The responsible controller recreates them after deletion.
3. Removing an operand removes its policies through controller cleanup or verified garbage collection.
4. The platform does not delete them by rendering the module bundle.

The two lifecycle paths MUST NOT target the same NetworkPolicy object.

### Validation contract

Validation has separate bundle and runtime gates.

#### Bundle validation in `opendatahub-operator`

The platform module-manifest compliance test MUST render every registered module Helm/Kustomize source and verify:

- Every rendered module operator workload is selected by at least one NetworkPolicy using a non-empty `podSelector` specific to that workload.
- Each policy uses `networking.k8s.io/v1` and kind `NetworkPolicy`.
- Policy namespace matches the rendered module operator namespace.
- Each bundle allow-policy `podSelector` matches its intended rendered workload and no unrelated workload. A documented namespace-wide default-deny policy MAY use an empty selector.
- `policyTypes` is explicit and includes `Ingress` for endpoint-protecting policies.
- Every allowed ingress rule has explicit sources and ports; a policy with no ingress rules is valid when the selected operator requires no ingress.
- No empty selector on a bundle allow policy or allow-all ingress pattern is present; a documented namespace-wide default-deny policy is valid.
- Rendered RBAC grants the platform operator's ServiceAccount required NetworkPolicy verbs for module-bundle policies through the expected Role/RoleBinding or ClusterRole/ClusterRoleBinding and matches the controller's declared namespace authority set.

This test validates module-bundle policies only. It MUST NOT claim to validate operand coverage or module-controller ownership.

#### Runtime validation by component teams

Component teams own runtime validation for their traffic paths and policy lifecycle. Module or component integration tests MUST verify operand policies:

- Every documented endpoint has corresponding policy coverage.
- Test evidence identifies each workload selector, Service and port, exposure path, allowed peer, and owning NetworkPolicy. Generated and user-created workloads are checked at runtime, not only from static manifests.
- Allow-policy target selectors match the intended workloads and no unrelated pods. A documented namespace-wide default-deny policy MAY select every pod in its namespace.
- Source peers and ports match the documented callers and endpoints.
- Deleting a policy causes the responsible controller to recreate it.
- Failure to create or restore a required policy reports the component as not ready or degraded in both its component CR and the DSC component status; the status clears after recovery.
- Removing an operand leaves no policy created for that operand after reconciliation and cleanup complete.
- Manual policy changes are reconciled to desired state.
- When a component manages egress, egress peers and ports match its documented destinations.

Where NetworkPolicy enforcement is available, component integration tests MUST also verify that unauthorized cross-namespace ingress is denied and approved module, gateway, monitoring, and webhook traffic continues to work. Components managing egress MUST also verify that documented egress succeeds and an unapproved destination is denied.

Where NetworkPolicy enforcement is available, traffic-behavior tests MUST evaluate every NetworkPolicy that selects the tested pods, regardless of lifecycle owner, because NetworkPolicies are additive. Environmental overrides MUST be recorded in the test evidence. A deny-all baseline is a preferred test strategy, not a required object; every such qualification MUST include positive and negative traffic tests.

### Platform support boundary

The platform supports this contract by:

- Rendering and applying module-bundle policies with module operator resources.
- Validating bundle-policy presence and shape during module compliance checks.
- Preserving NetworkPolicy resources through Helm/Kustomize rendering and deployment.
- Passing platform namespace configuration required by module policies.
- Cleaning up bundle policies with other rendered module resources.

It does not:

- Create operand policies on behalf of modules.
- Add platform ownership to operand policies.
- Use module garbage collection to delete operand policies.
- Treat one bundle policy as sufficient operand-policy compliance.

## Alternatives

### Platform manages all NetworkPolicies

Have `opendatahub-operator` create and reconcile both module-bundle and operand policies.

This centralizes validation and lifecycle handling, but the platform operator lacks complete knowledge of module-created operands, generated endpoints, and module-specific configuration. It also expands platform RBAC and creates coupling between platform and component release cycles.

### Module operators manage all NetworkPolicies

Have each module operator also manage the policy for its own operator pods.

This gives one controller all policy context for a module, but conflicts with the platform's bundle deployment and cleanup lifecycle. It can produce competing field managers and leave operator pods unprotected when the module operator is unavailable.

### Static operand policies in the module bundle

Ship operand policies with the module Helm or Kustomize source and let `opendatahub-operator` apply them.

This is simple for fixed workloads but cannot reliably follow user-created operands, generated namespaces, endpoint configuration, or operand deletion. It also gives the platform deployer lifecycle responsibility for resources whose desired state is known only to the component controller.

### Namespace-wide allow policies

Use broad namespace-level policies and rely on service authentication for finer control.

This reduces component effort but violates least privilege, permits unintended cross-component traffic, and is insufficient as a network isolation contract.

## Security and Privacy Considerations

- NetworkPolicy is defense in depth; it does not replace application authentication, authorization, or encryption.
- Explicit selectors, sources, and ports reduce unintended exposure and cross-namespace access.
- Policy changes can deny required control-plane, monitoring, or webhook traffic; component tests MUST cover approved traffic paths.
- Components that manage egress use explicit `Egress` rules for only their documented destinations and ports.
- NetworkPolicies are additive. A broad policy selecting the same pods can expand allowed traffic, so validation MUST evaluate the complete installed policy set.
- Compliance demonstrates component-level NetworkPolicy ownership and tested L3/L4 restrictions where NetworkPolicy enforcement is available. It does not demonstrate complete tenant isolation, application authorization, data partitioning, or encryption.
- This contract does not introduce collection or persistence of application data.

## Risks

- Incorrect selectors or platform namespace assumptions can break module availability.
- A cluster can use a CNI that does not support NetworkPolicy. Kubernetes may accept policy objects without enforcing isolation. Because the platform supports both OCP and generic Kubernetes, CNI capabilities cannot be assumed uniformly.
- Hostname-only or dynamically addressed egress destinations cannot be represented precisely by standard NetworkPolicy.
- Components that do not manage egress can make unrestricted outbound connections unless another policy or network control restricts them.
- An unrelated broad NetworkPolicy can weaken isolation because allowed traffic is the union of all policies selecting a pod.
- A module that omits operand reconciliation can appear compliant at bundle-render time while leaving runtime endpoints exposed.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| --- | --- | --- | --- |
| ODH Platform / Operator | | 2026-09-09 | Yes |
| All module and component teams | | 2026-09-09 | Yes |
| Security and release engineering | | 2026-09-09 | Yes |

Component teams must add bundle policies, operand reconciliation, RBAC, and runtime tests. The platform team must run bundle compliance tests, and preserve the lifecycle boundary.

## References

- [RHOAIENG-90747](https://issues.redhat.com/browse/RHOAIENG-90747)
- [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes NetworkPolicy API](https://kubernetes.io/docs/reference/kubernetes-api/networking/network-policy-v1/)
- [ODH Platform Utilities](https://github.com/opendatahub-io/odh-platform-utilities)

## Reviews

| Reviewed by | Date | Notes |
| --- | --- | --- |
| | | |
