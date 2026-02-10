[Workbench component documentation]: ../workbenches
[AcceleratorProfiles]: ./README.md#acceleratorprofiles-deprecated
[HardwareProfiles]: ./README.md#hardwareprofiles

# Dashboard Storage Mechanisms

There are only two types of storages we have in the Dashboard. Local to the user's browser & on-cluster storage.

* [Browser Storage](#browser-storage)
* [On-Cluster Storage](#on-cluster-storage)
  * [Admin / Dashboard Configurations](#admin--dashboard-configurations)
  * [Non-Admin flows](#non-admin-flows)

## Browser Storage

User-specific choices are currently stored in the browser's storages (local & session storages).

Such as:
* "remember my choice" settings
    * Stop notebook on toggle modal not showing up
    * Remember to open Jupyter tile Notebooks in new tab without asking
* Active QuickStart
* Some technical infrastructure around detecting token expiry and auto-handling an auto logout

## On-Cluster Storage

### Admin / Dashboard Configurations

Features that impact all users or the Dashboard itself. These are only available to those considered admin.

* Cluster Settings
    * Such as:
        * Model serving platforms
        * PVC size (Notebook tile only)
        * Notebook pod tolerations
        * Telemetry
    * are stored in the [OdhDashboardConfig]
* Cluster Settings' Notebook Culler
    * This is configured by the Notebook Controller feature
    * Stored today as `notebook-controller-culler-config` ConfigMap in the deployment namespace (see [Workbench component documentation] for more information)
* Accelerator profiles (deprecated)
    * These are stored as [AcceleratorProfiles] -- deprecated, replaced by HardwareProfiles
* Hardware profiles
    * These are stored as [HardwareProfiles] CRDs (`hardwareprofiles.infrastructure.opendatahub.io/v1`). HardwareProfiles replace both AcceleratorProfiles and the deprecated `notebookSizes`/`modelServerSizes` fields on OdhDashboardConfig.
* Notebook images
    * These are stored as ImageStreams in the deployment namespace
* Serving Runtimes
    * Admin-managed (global) serving runtimes are stored as OpenShift Templates wrapping ServingRuntime CRs in the deployment namespace
    * Project-scoped serving runtimes are stored as direct ServingRuntime CRs in the user's project namespace
      > Note: OpenShift Templates was an idea of future feature expansion and are not executed as OpenShift Templates today.
* Connection Types
    * These are stored as ConfigMaps in the deployment namespace

### Non-Admin Flows

Flows that can be performed by any user, provided the [feature is enabled](./configuringDashboard.md#configuring-features-onoff).

These are all stored as K8s resources using OCP or OpenShift AI backing CRDs.

#### Connections

Connections is a concept created by the Dashboard to store information and enable users to connect to various data sources. This information is stored in a K8s secret. The data within these secrets conform to the schema defined within connection types. Connection types are predefined OOTB and can also be defined by an admin.

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: <user-inputted-value>
  namespace: <users-namespace>
  labels:
    opendatahub.io/dashboard: 'true'
    opendatahub.io/managed: 'true'
  annotations:
    opendatahub.io/connection-type-ref: s3
    openshift.io/display-name: <users-driven-display-name>
data:
  AWS_ACCESS_KEY_ID: <value>
  AWS_DEFAULT_REGION: <value>
  AWS_S3_BUCKET: <value>
  AWS_S3_ENDPOINT: <value>
  AWS_SECRET_ACCESS_KEY: <value>
type: Opaque
```

> Note: The `opendatahub.io/connection-type-ref` annotation (introduced in 2.16) is the primary annotation for identifying the connection type schema. The legacy `opendatahub.io/connection-type` annotation is still recognized as a fallback. Additionally, connections may include the `opendatahub.io/connection-type-protocol` annotation (3.0+) for protocol-based validation and routing.

> Note: The `aws-connection-` name prefix shown in older examples is a legacy naming convention from the original S3-only Data Connections. New connections created through the Connection Type system do not use this prefix.

See more information on the labels & annotations in the [Connection section of the K8s Labels & Annotations](./k8sLabelsAndAnnotations.md#connections)

See more information on how the Connection feature works by reading more in the [Feature Connections section](./features/connections.md).
