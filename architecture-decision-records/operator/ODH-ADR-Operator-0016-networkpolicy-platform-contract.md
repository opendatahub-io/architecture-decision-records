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
- Avoid empty selectors and allow-all ingress.
- Work in every supported overlay.

If the module operator accepts no ingress traffic, its policy MUST declare `policyTypes: [Ingress]` with no ingress rules. Multiple policies are allowed, for example separate policies for metrics and webhook traffic.

`opendatahub-operator` installs bundle policies with the module operator resources, manages their rendered state, and removes them during module-resource cleanup. The module operator MUST NOT reconcile the same bundle policy. It may read or watch the object when required, but must not compete with the platform field manager or assign a second controller owner.

### Operand policies

Every module-managed workload that accepts traffic through a Service, webhook, sidecar, user-facing endpoint, or documented pod-to-pod listener MUST be selected by an operand NetworkPolicy controlling that ingress. A module MUST NOT rely only on a default namespace policy.

Expected source restrictions include:

| Endpoint | Required allowed source | Required ports |
| --- | --- | --- |
| Webhook | Kubernetes API server | Webhook port only |
| Metrics | Monitoring collectors | Explicit metrics port(s) only |
| Gateway-routed API or user endpoint | Approved gateway pods | Advertised endpoint port(s) only |
| Internal API | Explicit component callers | Declared endpoint port(s) only |

Component teams MUST define and document the concrete target selectors, allowed source peers, and ports for each exposed endpoint. The target `spec.podSelector` MUST use stable labels applied to workloads owned by the component. Source selectors for pod-based callers MUST use stable labels owned by the calling service. Placeholders and unresolved source identities are non-compliant.

Webhook policies MUST permit Kubernetes API-server calls only on the webhook port. Controllers MUST NOT derive selectors from currently running pods or add labels to workloads they do not own.

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

The responsible policy controller MUST have NetworkPolicy RBAC, declare ownership or an equivalent watch relationship, and make policy cleanup follow the user-facing operand lifecycle. If an upstream workload controller cannot reconcile the required policy, the RHOAI component integration controller MUST provide that reconciliation. The platform operator MUST NOT become a second reconciler.

User-facing policies MUST cover endpoint paths exposed by the operand, including approved user or gateway ingress, component-to-operand traffic, monitoring, and required platform-service or DNS traffic. A shared policy is acceptable only when its selector and lifecycle remain specific to the supported operand set; a namespace-wide allow policy is not sufficient.

### Policy content requirements

Every operand policy MUST target a specific pod set:

- `spec.podSelector` MUST contain meaningful labels.
- `podSelector: {}` is prohibited.
- Stable `app.kubernetes.io/*` labels or an equivalent module-owned label contract SHOULD be used.

Ingress rules MUST be explicit. Each ingress rule MUST contain a non-empty `from` list with specific NetworkPolicy peers and a non-empty `ports` list. Pod-based callers SHOULD be restricted by both namespace and pod selectors. A namespace-only source is acceptable only when that namespace is dedicated to the approved caller and the broader access is documented. These patterns are prohibited:

```yaml
ingress:
- {}
```

```yaml
from: []
```

Omitting `from` or `ports`, using an empty list, or otherwise allowing unrestricted ingress is non-compliant.

Workloads handling credentials, tokens, object-storage access, or other sensitive data MUST restrict egress to known destinations. An egress policy MUST select those workloads with a meaningful, stable `podSelector`, include `Egress` in `policyTypes`, and define non-empty `to` and `ports` lists. A dedicated egress policy uses `policyTypes: [Egress]`.

The allowlist MUST contain only destinations required by the workload.

The complete installed policy set MUST permit no other egress for those workloads. In-cluster destinations SHOULD use stable namespace and pod selectors. Destinations represented by IP ranges MUST use the narrowest stable CIDRs available. Because Kubernetes NetworkPolicy cannot select destinations by DNS name, modules using hostname-only or dynamically addressed services MUST document that limitation and any supplemental control.

### Controller RBAC and configuration

The module or component controller ClusterRole or namespace-scoped Role MUST include the NetworkPolicy permissions required for its policy scope:

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

Permissions MUST be present in the controller's Helm chart or Kustomize manifests. Platform RBAC does not replace controller RBAC. Namespace-scoped permissions SHOULD be used when policies are confined to one namespace. Cluster-scoped permissions are appropriate only when a component intentionally manages policies across namespaces.

Modules MAY expose operand policy configuration through their module or operand CRD. A recommended shape is:

```yaml
spec:
  networkPolicy:
    ingress:
      enabled: true
```

For modules exposing services, policy creation MUST be enabled by default. If supported configuration disables policy creation, the deployment no longer complies with this contract. The module MUST document the reduced security posture, retain NetworkPolicy RBAC, and report the disabled protection in component status or another administrator-visible signal.

### Lifecycle semantics

Module-bundle policies follow the module operator resource lifecycle:

1. The platform renders and applies the policy with module operator resources.
2. Platform field management and Server-Side Apply manage rendered policy changes.
3. When a module is disabled, the module CR is removed first while the module operator remains available to clean up operands.
4. After the module CR is gone, the platform removes rendered module resources, including bundle policies.

Operand policies follow the module CR and operand lifecycle:

1. The responsible module or component controller creates them and establishes an owner reference or explicit lifecycle mapping.
2. The responsible controller recreates them after deletion.
3. Removing or disabling an operand removes its policies through controller cleanup or verified garbage collection.
4. The platform does not delete them by rendering the module bundle.

The two lifecycle paths MUST NOT target the same NetworkPolicy object.

### Validation contract

Validation has separate bundle and runtime gates.

#### Bundle validation in `opendatahub-operator`

The platform module-manifest compliance test MUST render every registered module Helm/Kustomize source and verify:

- Every rendered module operator workload is selected by at least one NetworkPolicy.
- Each policy uses `networking.k8s.io/v1` and kind `NetworkPolicy`.
- Policy namespace matches the rendered module operator namespace.
- Each `podSelector` matches its intended rendered workload and no unrelated workload.
- `policyTypes` is explicit and includes `Ingress` for endpoint-protecting policies.
- Every allowed ingress rule has explicit sources and ports; a policy with no ingress rules is valid when the selected operator requires no ingress.
- No empty selector or allow-all ingress pattern is present.

This test validates module-bundle policies only. It MUST NOT claim to validate operand coverage or module-controller ownership.

#### Runtime validation by component teams

Component teams own runtime validation for their traffic paths and policy lifecycle. Module or component integration tests MUST verify operand policies:

- Every documented endpoint has corresponding policy coverage.
- Target selectors match the intended workloads and no unrelated pods.
- Source peers and ports match the documented callers and endpoints.
- Deleting a policy causes the responsible controller to recreate it.
- Removing or disabling an operand leaves no policy created for that operand after reconciliation and cleanup complete.
- Manual policy changes are reconciled to desired state.

Where NetworkPolicy enforcement is available, component integration tests MUST also verify that unauthorized cross-namespace ingress is denied, approved module, gateway, monitoring, and webhook traffic continues to work, and sensitive-workload egress allows documented destinations while denying an unapproved destination.

Traffic-behavior tests MUST evaluate the complete installed policy set because multiple NetworkPolicies are additive. The preferred test uses a deny-all baseline and verifies only explicitly supported traffic paths.

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
- Policy changes can deny required control-plane, monitoring, webhook, or DNS traffic; component tests MUST cover approved traffic paths.
- Sensitive workloads MUST restrict egress with explicit `Egress` rules covering only required DNS, Kubernetes API, and documented service destinations.
- NetworkPolicies are additive. A broad policy selecting the same pods can expand allowed traffic, so validation MUST evaluate the complete installed policy set.
- Policy configuration that disables protection MUST be documented as a reduced security posture.
- This contract does not introduce collection or persistence of application data.

## Risks

- Incorrect selectors or platform namespace assumptions can break module availability.
- A cluster can use a CNI that does not support NetworkPolicy. Kubernetes may accept policy objects without enforcing isolation. Because the platform supports both OCP and generic Kubernetes, CNI capabilities cannot be assumed uniformly.
- Hostname-only or dynamically addressed egress destinations cannot be represented precisely by standard NetworkPolicy.
- An unrelated broad NetworkPolicy can weaken isolation because allowed traffic is the union of all policies selecting a pod.
- A module that omits operand reconciliation can appear compliant at bundle-render time while leaving runtime endpoints exposed.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| --- | --- | --- | --- |
| ODH Platform / Operator | | 2026-09-09 | Yes |
| Module and component teams | | 2026-09-09 | Yes |
| Model Serving | | 2026-09-09 | Yes |
| Data Science Pipelines | | 2026-09-09 | Yes |
| Distributed Workloads | | 2026-09-09 | Yes |
| Model Registry | | 2026-09-09 | Yes |
| Dashboard and workbench teams | | 2026-09-09 | Yes |
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
