# Extract Catalog Into a Standalone Operator

|                |            |
| -------------- | ---------- |
| Date           | 2026-04-16 |
| Scope          | AI Hub |
| Status         | Approved |
| Authors        | [Paul Boyd](@pboyd) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

Extract the model catalog controller from `opendatahub-io/model-registry-operator` into a new standalone operator (`opendatahub-io/catalog-operator`) with its own `Catalog` CRD.

## Why

Once the Kubeflow model registry is replaced by the MLflow model registry, the only thing left for the `model-registry-operator` will be deploying the catalog. The operator's name should reflect what it does, not the component it no longer deploys.

We could add the CRD separately, but combining it with the operator re-implementation is simpler overall. We currently lack a practical way to configure the catalog's deployment options. We have avoided this so far, even when it would have been useful (for example, the PostgreSQL PVC size is hard-coded) but we'll need it eventually.

## Goals

* Create `opendatahub-io/catalog-operator` with a `Catalog` CRD under the `catalog.opendatahub.io` API group
* Expose resource requirements, database configuration, and security mode on the CRD spec
* Integrate the new operator into the ODH operator as a new DSC component (`catalog`)
* Remove all catalog code from `opendatahub-io/model-registry-operator`

## Non-Goals

* MLflow model registry integration timeline or design
* Catalog UI changes
* When to set `modelregistry.managementState` to `Removed` by default

## How

### New operator: `opendatahub-io/catalog-operator`

Scaffold a Kubebuilder project with a `Catalog` CRD (group `catalog.opendatahub.io`, version `v1alpha1`). Port the catalog controller, templates, and shared utilities from the model-registry-operator. Refactor the controller to reconcile a `Catalog` CR instead of running as a channel-triggered singleton.

Proposed CRD example:

```yaml
apiVersion: catalog.opendatahub.io/v1alpha1
kind: Catalog
metadata:
  name: catalog
  namespace: rhoai-model-registries
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
      storageClass: "the-storage-class" # Can be null to use the default.
```

The CR name will be required to be `catalog` in order to prevent multiple catalogs in a single namespace. The created resources will have the same names as they do now from the `model-registry-operator`.

Catalog sources remain ConfigMap-based. The labeled-ConfigMap discovery mechanism (`opendatahub.io/catalog-source=true`) carries over unchanged.

### ODH operator integration

Add a new DSC component `catalog` with `managementState: Managed | Removed`. The ODH operator will deploy a default `Catalog` CR to the `modelregistry` component's `registriesNamespace`.

The namespace configuration will be migrated in phases to avoid breaking the Dashboard or other API consumers:

1. **Initial release**: The `catalog` component has no namespace field of its own. It deploys the `Catalog` CR to the `modelregistry` component's `registriesNamespace`. The Dashboard continues to use `registriesNamespace` to locate catalog resources. This keeps the catalog in the same namespace it has always been in, with no Dashboard changes required.

2. **Follow-up release**: Add a `catalogNamespace` field to the `catalog` component. Default it from `registriesNamespace` so existing installs are unaffected. Update the Dashboard to read `catalogNamespace` for catalog requests and `registriesNamespace` for registry requests.

3. **Registry removal**: When the `modelregistry` component is removed, clean up `registriesNamespace` references in the Dashboard and operator code. The `catalogNamespace` field becomes the sole authority for the catalog's namespace.

### Model-registry-operator changes

Remove the catalog controller and related code from `model-registry-operator`. It will not actively delete catalog resources because the `catalog-operator` will take over reconciling the same resources.

### Migration

The migration requires no explicit steps, provided the catalog is removed (or disabled) during the same upgrade that rolls out `catalog-operator`. The default Catalog CR will cause the `catalog-operator` to operate on the same resources that `model-registry-operator` manages now. After the first reconcile, the resources which were previously owned by the `default-modelregistry` CR will have their `ownerReference`s changed to the new default Catalog CR.

Customers who upgrade directly from an older version (with `model-registry-operator`) to a newer version (with only `catalog-operator`) may have their catalog resources garbage collected before the new operator does the initial reconciliation. Nothing important will be permanently lost: user-managed ConfigMaps lack the `ownerReference` and are therefore preserved; the PostgreSQL PVC may be lost, but it will be rebuilt when the catalog pod start. However, there could be a several minutes where the catalog is offline while the catalog is re-provisioned from scratch (particularly the PostgreSQL PVC). This is not ideal, but it can be mitigated by not skipping major RHOAI releases and the catalog resources will be re-provisioned after only a few minutes during an upgrade.

## Alternatives

### Rename `model-registry-operator` and remove Model Registry

After the Kubeflow Model Registry has been deprecated, rename `model-registry-operator` to `catalog-operator`. After the deprecation period, remove the code to deploy the Kubeflow `model-registry`.

If we don't want to implement the Catalog CRD at the same time, this is probably the way to go. It would be faster to implement. But we will need the Catalog CRD eventually, and combining it with a necessary rename is the most straightforward way to add it.

A variation of this is to rename `model-registry-operator` to a more general name (e.g. `ai-hub-operator`). This has the same downsides as renaming to `catalog-operator`, but it would be reasonable if we anticipate adding additional components under the umbrella of AI Hub. However, this is speculative since we have no definite plans to extend AI Hub beyond registries and catalogs. And if we did extend the scope of AI Hub, we don't know if the requirements of that hypothetical extension would make sense inside a general AI Hub operator.

### Deploy the catalog from the `mlflow-operator`

The catalog has been deployed from `model-registry-operator`, and with `mlflow-operator` replacing it, one could argue that `mlflow-operator` should also deploy the catalog. However, `model-registry-operator` deployed the catalog because the two were based on the same upstream project and shared the same container image. This does not apply to `mlflow-operator`, and deploying Kubeflow Model Catalog from an MLflow-specific operator would be strange.

## Risks

* The `catalog-operator` will need to recreate the same catalog resources that `model-registry-operator` does now. A discrepancy risks leaving Kubernetes resources orphaned: still owned by the `default-modelregistry` CR but unmanaged by any controller. A discrepancy could also affect the exposed API endpoints which would affect API users or the Dashboard, we will need to verify that nothing changes.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Platform team                 |                  |            | Yes - new DSC component |
| AI Hub team                   |                  |            | Yes - new repo, new operator to own |
| Dashboard team                |                  |            | Yes - phase 2 (see below) |

### Dashboard

In the initial release, the catalog remains in the `registriesNamespace` and no Dashboard changes are required. In a follow-up release (phase 2), the Dashboard will need to read the new `catalogNamespace` field for catalog requests instead of `registriesNamespace`. This is a small change (read a different field name) and can be scheduled at the Dashboard team's convenience. The catalog will not move to a different namespace until the Dashboard is updated.

## References

* [Kubeflow to MLflow Model Registry Migration Plan](https://docs.google.com/document/d/1gV1Eg95K1eTueNUHqDJcKglisQrx0OYXCZ3eCHKxQMk/edit?usp=sharing)
* [Model Registry Operator](https://github.com/opendatahub-io/model-registry-operator)
* [Model Registry](https://github.com/opendatahub-io/model-registry)
* [ODH Operator](https://github.com/opendatahub-io/opendatahub-operator)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
