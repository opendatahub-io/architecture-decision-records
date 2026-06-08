# AI Hub Operator: Modular Architecture for Catalog and Model Registry

|                |            |
| -------------- | ---------- |
| Date           | 2026-06-05 |
| Scope          | AI Hub |
| Status         | Draft |
| Authors        | [Paul Boyd](@pboyd) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | RHOAIENG-60945 |
| Other docs:    | [ODH-ADR-Operator-0006](../operator/ODH-ADR-Operator-0006-internal-api.md) |

## What

Rename `opendatahub-io/model-registry-operator` to `opendatahub-io/ai-hub-operator` and restructure it as a module operator under the platform's modular architecture. The single repo produces a single container image with three operator entrypoints:

- **AI Hub operator**: the module operator the platform deploys; reconciles an `AIHub` module CR
- **Catalog operator**: reconciles `Catalog` CRDs; the AI Hub operator creates a default `Catalog` CR
- **Model Registry operator**: reconciles `ModelRegistry` CRDs created directly by users (no change to user-facing behavior)

## Why

The platform operator is migrating from a monolithic architecture to a modular one (see ODH-ADR-Operator-0006 and RHOAIENG-60945). Under this model, each component team owns a module operator that the platform deploys and manages. The AI Hub team must provide such an operator.

Separately, once the Kubeflow model registry is replaced by the MLflow model registry, the `model-registry-operator` will no longer deploy its eponymous component. The operator name will be misleading, and the catalog will need its own CRD to expose configuration options that are currently hard-coded (for example, the PostgreSQL PVC size).

The application namespace has the same problem. Today the catalog and model registries are deployed to `rhoai-model-registries` (or `odh-model-registries`), a name that references a component that no longer describes the workloads running there. Because service URLs embed the namespace, changing it requires coordinating with API consumers--primarily the Dashboard. The modular architecture provides a clean way to handle this: the `AIHub` CR status can advertise the application namespace, giving the Dashboard a stable indirection point that is independent of the actual namespace name.

Combining both changes--adopting the module operator pattern and extracting the Catalog CRD--in a single rename-and-restructure avoids an intermediate step that would be immediately obsolete.

## Goals

* Rename the repo and produce a single container image with three operator entrypoints (AI Hub, Catalog, Model Registry)
* Define an `AIHub` module CRD for platform integration
* Define a `Catalog` CRD exposing resource requirements and database configuration
* AI Hub operator deploys and manages the Catalog and Model Registry operators, and creates a default `Catalog` CR
* Provide a `ModuleHandler` implementation and deployment manifests for the platform operator
* Preserve backward compatibility for the Dashboard team

## Non-Goals

* MLflow model registry integration timeline or design
* Catalog UI changes
* Multi-tenancy for catalogs or model registries
* Detailed internal reconciliation logic for any of the three operators

## How

### Repository and image architecture

The `model-registry-operator` repo is renamed (proposed name is `ai-hub-operator`). The repo produces a single container image. The three operators are built either as a single binary with three subcommands or as three separate binaries; the choice is deferred to implementation. Using a single image reduces build, CI, and release overhead.

### AIHub CRD and operator

The `AIHub` CRD is the module CR. The platform creates one instance and projects platform-level fields into its spec (application namespace, gateway domain, release metadata) following the module API contract defined by the platform team. The AI Hub operator reconciles the `AIHub` CR and:

1. Deploys a `Deployment` for the Catalog operator (using the same image with the catalog entrypoint)
2. Deploys a `Deployment` for the Model Registry operator (using the same image with the model-registry entrypoint)
3. Creates a default `Catalog` CR in the configured application namespace if one does not exist
4. Aggregates the health status of all managed resources into the `AIHub` CR status for the platform to read

The `AIHub` CRD spec will, at minimum, carry the application namespace (the namespace where model registries and the catalog are deployed).

### Catalog CRD and operator

The Catalog operator reconciles `Catalog` CRDs under the `aihub.opendatahub.io/v1alpha1` API group. This is a port and refactor of the catalog controller currently in `model-registry-operator`, changed to reconcile a CR instead of running as a channel-triggered singleton.

The AI Hub operator creates one default Catalog CR. The CR name must be `catalog` to prevent multiple catalogs per namespace, this will be enforced by a webhook in the catalog operator. Proposed spec:

```yaml
apiVersion: aihub.opendatahub.io/v1alpha1
kind: Catalog
metadata:
  name: catalog
  namespace: rhoai-model-registries # will change to rhoai-ai-hub (or similar) in a future release
spec:
  resources:
    catalog:
      requests: {cpu: "100m", memory: "256Mi"}
      limits: {cpu: "500m", memory: "512Mi"}
    postgres:
      requests: {cpu: "100m", memory: "256Mi"}
      limits: {cpu: "500m", memory: "512Mi"}
  database:
    pvc:
      size: "10Gi"
      storageClass: "the-storage-class" # omit to use the cluster default
```

The created Kubernetes resources will have the same names as they do now from `model-registry-operator`. Catalog sources remain ConfigMap-based; the labeled-ConfigMap discovery mechanism (`opendatahub.io/catalog-source=true`) carries over unchanged.

### Model Registry operator

The Model Registry operator continues to reconcile `ModelRegistry` CRDs. Users create `ModelRegistry` CRs directly and will continue to do so. The operator runs as a separate process (separate entrypoint) from the same image. Its behavior is unchanged from the current `model-registry-operator`.

### Platform integration

The platform operator deploys the AI Hub operator and creates the `AIHub` module CR. The AI Hub team provides:

1. **ModuleHandler** in the ODH operator repo: thin registration code (~30 lines using the `BaseHandler` helper) that registers the `AIHub` module
2. **Deployment manifests**: Kustomize overlays (or a Helm chart) for deploying the AI Hub operator, including the `AIHub` CRD, RBAC, `Deployment`, and `ServiceAccount`
3. **Webhook ownership**: the AI Hub team owns all admission webhooks related to AI Hub workloads

This replaces the current pattern where the `model-registry-operator` integrates as a DSC component (`modelregistry`) directly managed by the ODH operator.

### DSC migration

The existing `modelregistry` DSC component must transition to the `AIHub` module. Ideally this happens in a single release: the platform operator stops managing the `modelregistry` component and instead deploys the AI Hub operator and creates an `AIHub` CR. If a single-release swap is not feasible, the `modelregistry` DSC component and the `AIHub` module will coexist temporarily, with the `modelregistry` component set to `Removed` once the AI Hub operator is verified healthy.

The specific migration approach depends on the platform team's modular architecture rollout schedule and is an open question (see below).

### Dashboard compatibility and namespace rename

The `AIHub` CR status exposes the application namespace--the namespace where the catalog and model registries are deployed. The Dashboard reads the namespace from this status field rather than from the DSC component status or a hardcoded value.

This separates two concerns: *where to read the namespace* (always the `AIHub` CR status) and *what the namespace is* (a value that can change independently). The Dashboard changes once--to read from the `AIHub` CR status--and never needs to change again regardless of what the namespace is called.

1. **Initial release**: The `AIHub` CR status reports the existing namespace (`rhoai-model-registries` or `odh-model-registries`). The Dashboard is updated to read the application namespace from the `AIHub` CR status.
2. **Future rename**: When we are ready to rename the namespace (e.g., to `rhoai-ai-hub`), update the `AIHub` CR spec. The operator deploys to the new namespace, the status updates, and the Dashboard picks it up automatically. No coordinated cross-team release required.

### Migration

The transition from `model-registry-operator` to the AI Hub operator's Catalog operator requires no explicit user steps.

**Ownership transfer.** The Catalog operator uses an adopt-or-create strategy for every resource it manages (Deployments, Services, PVCs, etc.). On each reconcile, if the expected resource already exists, the operator patches its `ownerReferences` to point to the `Catalog` CR. If the resource does not exist, the operator creates it. This is idempotent — running it against already-adopted resources is a no-op.

When the old `default-modelregistry` CR is deleted, the garbage collector will delete dependent resources whose only `ownerReferences` point to it. The Catalog operator handles this through its adopt-or-create strategy: on its first reconcile it recreates any missing resources.

**Upgrade window.** During the transition there may be a brief window where neither operator is running (gap) or both are running (overlap):

- *Gap*: The garbage collector may delete resources whose `ownerReferences` point only to the deleted CR. The Catalog operator recreates them on its first reconcile. The main cost is PostgreSQL PVC reprovisioning time (up to several minutes of catalog downtime).
- *Overlap*: Both operators may briefly reconcile the same resources. Because both produce the same desired state, this is a convergent no-op. The old operator stops reconciling once its CR is deleted.

**Requirement:** the `model-registry-operator` must not delete managed resources (Deployments, Services, PVCs) when its CR is deleted. Destructive finalizer logic on the old CR would cause resource loss before the Catalog operator can adopt them.

**Skip-version upgrade risk:** if the catalog resources are garbage collected before the Catalog operator does its first reconcile, the PostgreSQL PVC may be lost and rebuilt (several minutes of catalog downtime). User-managed ConfigMaps are unaffected (no `ownerReference`). This risk is mitigated by not skipping major RHOAI releases.

## Open Questions

* **Repo name**: `ai-hub-operator` is the leading candidate. Needs a decision before the first PR.
* **AIHub CR spec**: The exact fields the platform projects into the `AIHub` CR must be agreed with the platform team.
* **Webhook inventory**: Which webhooks does the AI Hub team currently own, and which need to move from the platform operator to the AI Hub operator?
* **Application namespace in module status**: Does the modular architecture define a standard way for modules to expose their application namespace in CR status? If so, the Dashboard integration follows an established convention. If not, the AI Hub team and platform team need to agree on the field name.

## Alternatives

### Standalone `catalog-operator` (the prior revision of this ADR)

Extract the catalog controller into a new `opendatahub-io/catalog-operator` repo and integrate it directly as a new DSC component. This is simpler to implement, however it does not provide the module operator the platform now requires. A separate AI Hub module operator would still be needed to satisfy the modular architecture requirements, adding an extra repo and image for no practical benefit.

### Separate repos and images per operator

Maintain three separate repos (`ai-hub-operator`, `catalog-operator`, `model-registry-operator`) each producing their own image. This provides stronger isolation between operators but significantly increases build, CI, and release overhead. Since the same team owns all three and their release cycles are tightly coupled, the isolation benefit does not justify the overhead.

### Single-process multi-controller

Run all three controllers in a single process within a single deployment. Simpler operationally, but reduces failure isolation: a crash in one controller restarts all three. Separate entrypoints allow independent restarts and future independent scaling.

### Deploy the catalog from `mlflow-operator`

The catalog has been deployed from `model-registry-operator` because the two share the same upstream project and container image. This rationale does not apply to `mlflow-operator`. Deploying Kubeflow Model Catalog from an MLflow-specific operator would be architecturally incoherent.

## Risks

* **Modular architecture pattern is evolving**: ODH-ADR-Operator-0006 is still being implemented. The AI Hub team may need to adapt if the `ModuleHandler` interface changes.
* **Repo rename ripple effects**: renaming the repo breaks CI pipelines, image references, OLM bundle metadata, and any external documentation pointing to the old name. Needs a coordinated rollout.
* **CRD ownership coordination**: the `ModelRegistry` CRD is currently owned by `model-registry-operator`. Ownership must transfer cleanly to the renamed repo without disrupting existing `ModelRegistry` CRs.
* **Skip-version upgrades**: catalog resources may be garbage collected before the new operator reconciles, causing brief catalog downtime (see Migration section).
* **Dashboard breakage**: the Dashboard must be updated to read the application namespace from the `AIHub` CR status in the initial release. After that, namespace renames are transparent to the Dashboard.

## Stakeholder Impacts

| Group | Impacted? |
|---|---|
| Platform team | Yes - must accept AIHub as a module operator and define the module CR contract |
| AI Hub team | Yes - repo rename, new architecture, new build pipeline |
| Dashboard team | Yes - must read the application namespace from the `AIHub` CR status in the initial release; future namespace renames require no Dashboard changes |

### Dashboard

In the initial release, the Dashboard is updated to read the application namespace from the `AIHub` CR status. The namespace value itself does not change. Future namespace renames (e.g., from `rhoai-model-registries` to `rhoai-ai-hub`) are transparent to the Dashboard because it reads the namespace from the CR status rather than using a hardcoded value.

## References

* [ODH-ADR-Operator-0006: Internal API (modular architecture)](../operator/ODH-ADR-Operator-0006-internal-api.md)
* [RHOAIENG-60945: AI Hub onboarding to modular architecture](https://issues.redhat.com/browse/RHOAIENG-60945)
* [Modular Architecture Migration Guide](https://gitlab.cee.redhat.com/data-hub/odh-modularisation-docs/-/blob/main/Component%20to%20Module%20Migration%20Guide.md)
* [Module Handler Developer Guide](https://gitlab.cee.redhat.com/data-hub/odh-modularisation-docs/-/blob/main/Module%20Handler%20Developer%20Guide.md)
* [Kubeflow to MLflow Model Registry Migration Plan](https://docs.google.com/document/d/1gV1Eg95K1eTueNUHqDJcKglisQrx0OYXCZ3eCHKxQMk/edit?usp=sharing)
* [Model Registry Operator](https://github.com/opendatahub-io/model-registry-operator)
* [ODH Operator](https://github.com/opendatahub-io/opendatahub-operator)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
