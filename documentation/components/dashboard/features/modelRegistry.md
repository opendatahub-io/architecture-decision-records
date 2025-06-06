# Model Registry

> Introduced in 2.16 as Tech Preview

- [Introduction](#introduction)
  - [A Model Registry is a Metadata Service Only](#a-model-registry-is-a-metadata-service-only)
  - [Additional Resources](#additional-resources)
- [Terms](#terms)
- [Example Data](#example-data)
  - [ModelRegistry k8s CR](#modelregistry-k8s-cr)
  - [RegisteredModel (`/api/model_registry/v1alpha3/registered_models/1`)](#registeredmodel-apimodel_registryv1alpha3registered_models1)
  - [ModelVersion (`/api/model_registry/v1alpha3/model_versions/2`)](#modelversion-apimodel_registryv1alpha3model_versions2)
  - [ModelArtifact (`/api/model_registry/v1alpha3/model_artifacts/3`)](#modelartifact-apimodel_registryv1alpha3model_artifacts3)
- [Implementation Notes](#implementation-notes)
  - [RBAC](#rbac)
  - [Autofilling Model Locations from Connections](#autofilling-model-locations-from-connections)
  - [Usage of `proxyService` and Resolving the API URL via a Service Annotation](#usage-of-proxyservice-and-resolving-the-api-url-via-a-service-annotation)
  - [Workaround for Arbitrary Updates to Last Modified Timestamps](#workaround-for-arbitrary-updates-to-last-modified-timestamps)
- [Technical Debt](#technical-debt)
- [Integration with Other Features](#integration-with-other-features)
  - [Model Serving](#model-serving)
  - [Model Catalog](#model-catalog)
  - [Pipelines](#pipelines)

## Introduction

The Model Registry feature provides a metadata store for users to track, version and manage their AI models. Administrators can set up registries and configure which users can see and use them. Users with access to a registry can register a model by entering metadata including the model location, which is stored in a MySQL database pre-configured by an administrator. Users can then view the registered models/versions and deploy them via integration with the dashboard's Model Serving feature, or tune them via integration with the dashboard's InstructLab pipeline feature (for supported models).

### A Model Registry is a Metadata Service Only

It is important to understand what a model registry is not:

- **A model registry does not provide its own database**. When an administrator creates a model registry they must enter details of a MySQL database they have pre-provisioned for storing registry metadata. Creating a model registry deploys a service providing an API layer around an existing database.
- **A model registry does not store or broker secrets/credentials for accessing models**. Only non-sensitive model location details and related metadata are stored in the database, and each user must enter their own credentials when consuming a model/version.
- **A model registry does not directly provide model storage capabilities**. There is functionality on the roadmap for associating some admin-provisioned OCI storage with a registry (using Connection Types) and to facilitate running and tracking jobs for the user to copy models into that storage, but Model Registry shall not be the provider of or gateway for that storage.
- **A model registry does not exist in a user's project/namespace**. Registries are created by an administrator in one centralized namespace (`odh-model-registries`) and a registry can be used by any user who has been granted access to it regardless of the projects they can access. When a user initiates some function from the registry that requires a project (such as deploying a model or copying it to storage) they select a project at that time. Any necessary Connections are created in that project, not in the registries namespace.

### Additional Resources

- ODH documentation: [Overview of model registries](https://opendatahub.io/docs/working-with-model-registries/)
- Developer documentation: [Model registry logical model](https://github.com/kubeflow/model-registry/blob/main/docs/logical_model.md).

## Terms

- **Model Registry** -- A metadata store configured by the administrator on the Model Registry Settings page.
  - When the admin creates a registry, the dashboard creates a ModelRegistry k8s custom resource in the model registries namespace (default is `odh-model-registries`), and a Secret in that namespace containing the database password. The ModelRegistry CR's spec points to a preconfigured MySQL database and to the database password Secret. Upon creation of the ModelRegistry CR, a service is deployed which can be accessed via a Model Registry REST API. This is how the dashboard interacts with a model registry.
  - When creating a registry, the admin can opt to secure their database connection with SSL. They are asked to either provide a new certificate string or select an existing Secret or ConfigMap in the registries namespace which contains a certificate. If they provided a new one, a new ConfigMap is created for it along with the resources described above. The ModelRegistry CR spec includes a reference to the certificate Secret or ConfigMap if this option is chosen.
  - [The CRD for the ModelRegistry kind can be found here](https://github.com/opendatahub-io/model-registry-operator/blob/main/config/crd/bases/modelregistry.opendatahub.io_modelregistries.yaml). Note that there are two CRDs called ModelRegistry. Be sure to use the right one:
    - ✅ `modelregistry.opendatahub.io/v1alpha1/ModelRegistry`
    - ❌ `components.platform.opendatahub.io/v1alpha1/ModelRegistry`
  - The [Model Registry REST API Swagger documentation can be found here](https://editor.swagger.io/?url=https://raw.githubusercontent.com/opendatahub-io/model-registry/main/api/openapi/model-registry.yaml).
  - The Model Registry page of the dashboard includes a dropdown menu to select a registry the user has access to (see [RBAC](#rbac)) and they can browse models and versions in the selected registry.
- **Registration** -- Within the context of a selected registry, a user can register a model (which also registers the first version of that model) or register a new version of an existing model. Registering a version (either for an existing model or for a new model) also creates an artifact to store that version's location.
- **Registered Model** -- A container for model versions which includes name, description, state, labels and custom properties.
- **Model Version** -- A child object of a Registered Model which represents a specific version of that model. Also contains its own name, description, state, labels and custom properties. Note that the model location is not stored in the version but in an artifact.
- **Model Artifact** -- A child object of a Model Version which references the actual location of an image of that model. This location is stored as a URI string in the artifact object. This URI can be a generic https:// URI, an oci:// URI, or an s3:// URI (which contains additional URL parameters for accessing an S3 bucket that are not in a standard format -- see [Technical Debt](#technical-debt)).
  - **Note about dashboard handling of artifacts**: The data model provided by the API allows for multiple artifacts to be associated with a version. However, **_the dashboard assumes there will always be only one artifact for each version_**, and it presents the artifact's details to the user as if they are additional details of the version. The entire concept of artifacts is hidden from the dashboard user. This decision was made to simplify the user experience, and it is considered acceptable to continue propagating this assumption.
  - When creating an artifact for a registered version, the name of the version is also used for the name of the artifact.
- **Archiving/Restoring** -- All objects in the model registry API have a `state` property which can have the values `"LIVE"` or `"ARCHIVED"`. Everything is created by default with the state `"LIVE"`. Models and versions can both be archived or restored in the dashboard which changes this state property.
  - Archived models and versions are hidden behind a menu action and are read-only in the dashboard.
  - Archiving a model does not change the `state` of its versions, but if a model is archived its versions will appear archived in the dashboard regardless of their `state` property. Restoring the archived model will return the dashboard to showing its versions in their previous archived/live states.
  - There are two separate sets of routes and views for archived models/versions and live models/versions. They reuse most of the same components, but there is some duplication. See [Technical Debt](#technical-debt).
- **Custom Properties** -- Models, versions and artifacts all contain some common fields including name, description, and customProperties. customProperties is an object containing an arbitrary key/value store for additional metadata about one of these objects. These properties are displayed and editable in the dashboard under the Properties table of the model details page and the version details page.
  - The customProperties on the artifact object are not shown in the dashboard. The artifact is used only for model location storage and no additional metadata.
- **Labels** -- The dashboard provides the ability to create and edit labels on models and versions. However, **_there is no concept of labels in the API_**. Instead we reuse the customProperties object to store labels: if a property has an empty string value its key is considered a label and it is shown in the dashboard's Labels section instead of the Properties section.
  - This creates a potentially confusing restriction: **_labels and property keys share the same uniqueness space_**. If the user attempts to create a label with the same text as a property's key or create a property key that is the same as a label it causes a validation error.
  - Just like other customProperties there are no labels for artifacts in the dashboard. Only models and versions have labels.
- **Owner/Author** -- The `owner` field on a model or the `author` field on a version both store the username of the user who registered that model/version. They are different. 🤷
- **Model Source** -- Model artifacts have a collection of properties for referencing a resource associated with where the model was registered from. These properties are used to render links back to places like the Model Catalog or Pipelines which registered a model/version/artifact.
  - `modelSourceKind` - identifies the kind of source which determines what the rest of the properties mean. e.g. `kfp` for Kubeflow Pipelines or `catalog` for Model Catalog.
  - `modelSourceClass` - a generic subgroup under modelSourceKind, e.g. `pipelinerun`.
  - `modelSourceGroup` - a generic subgroup under modelSourceClass, e.g. `my-project`.
  - `modelSourceId` - a unique identifier for the source resource.
  - `modelSourceName` - a name for the source resource.
- **Deployments** -- On the version details page there is a tab for Deployments of that version. Deployments are not a model registry concept; this is a point of connection to the dashboard's Model Serving feature. See [Integration with Other Features](#integration-with-other-features).

## Example Data

### ModelRegistry k8s CR

```yaml
apiVersion: modelregistry.opendatahub.io/v1alpha1
kind: ModelRegistry
metadata:
  annotations:
    openshift.io/description: "Description here"
    openshift.io/display-name: "Test registry"
  name: test-registry
  namespace: odh-model-registries
spec:
  grpc:
    port: 9090
  mysql:
    database: modelregistry
    host: mysql.odh-model-registries.svc.cluster.local
    passwordSecret:
      key: database-password
      name: test-registry-db-xrm5p
    port: 3306
    skipDBCreation: false
    username: modelregistry
  oauthProxy:
    port: 8443
    routePort: 443
    serviceRoute: enabled
  rest:
    port: 8080
    serviceRoute: disabled
```

### RegisteredModel (`/api/model_registry/v1alpha3/registered_models/1`)

```json
{
  "createTimeSinceEpoch": "1739872651053",
  "customProperties": {
    "_lastModified": {
      "metadataType": "MetadataStringValue",
      "string_value": "2025-02-19T20:03:35.237Z"
    },
    "my-custom-property": {
      "metadataType": "MetadataStringValue",
      "string_value": "user-entered value from Properties table on the model details page"
    },
    "my-label": {
      "metadataType": "MetadataStringValue",
      "string_value": ""
    }
  },
  "id": "1",
  "lastUpdateTimeSinceEpoch": "1739995415613",
  "name": "My Registered Model",
  "owner": "clusteradminuser1",
  "state": "LIVE"
}
```

### ModelVersion (`/api/model_registry/v1alpha3/model_versions/2`)

```json
{
  "author": "clusteradminuser1",
  "createTimeSinceEpoch": "1739872652113",
  "customProperties": {
    "_lastModified": {
      "metadataType": "MetadataStringValue",
      "string_value": "2025-02-19T20:03:35.236Z"
    },
    "my-custom-property": {
      "metadataType": "MetadataStringValue",
      "string_value": "user-entered value from Properties table on the version details page"
    },
    "my-label": {
      "metadataType": "MetadataStringValue",
      "string_value": ""
    }
  },
  "id": "2",
  "lastUpdateTimeSinceEpoch": "1739995415770",
  "name": "Version 1",
  "registeredModelId": "1",
  "state": "LIVE"
}
```

### ModelArtifact (`/api/model_registry/v1alpha3/model_artifacts/3`)

```json
{
  "artifactType": "model-artifact",
  "createTimeSinceEpoch": "1748456664853",
  "customProperties": {},
  "id": "3",
  "lastUpdateTimeSinceEpoch": "1748456664853",
  "name": "Version 1",
  "state": "LIVE",
  "storageKey": "test",
  "uri": "s3://my-bucket/my-model-v1?endpoint=http%3A%2F%2Fs3.amazonaws.com&defaultRegion=us-east-1"
}
```

Inconsistency note: while a ModelVersion references its parent with a `registeredModelId`, there is no reference to the parent version in a ModelArtifact. Instead, the URL path is used to create and list artifacts associated with a version by sending POST or GET requests to e.g. `/api/model_registry/v1alpha3/model_versions/2/artifacts`

## Implementation Notes

### RBAC

- **Model Registry Permissions** -- The Model Registry settings page also allows the administrator to configure permissions for access to a model registry. This "manage permissions" view uses the same functionality as managing permissions for a Project: the admin can add users or groups, which creates RoleBindings to give those entities a role granting them access to see that registry and use its REST API. The role
  - **Project permissions** -- A project can also be added to a registry in the manage permissions view. This gives all service accounts in the project permission to see and use the registry.
- **Listing Model Registries via Services** -- The non-admin user does not have permission to view ModelRegistry CRs, but the dashboard needs to be able to list them in order to populate the registry selector. Each registry has an associated Service resource, and a user with access to a registry can view that Service. To list the registries a user can see, the dashboard makes a SelfServiceRulesRequest to get the names of Services in the registries namespace and then fetches each of those Services. The model registry operator automatically syncs some metadata from the ModelRegistry CR to the Service (such as the display name and description) so non-admins have access to those details.
- **Usage of Service Account** -- Non-cluster-admin users (including ODH admins) do not have permission to create or view resources in the registries namespace (except Services). Currently, admin operations on the Model Registry Settings page use the dashboard's service account via backend routes. This will change soon with the introduction of the BFF and modular architecture.
  - Currently, the writing and reading of Secrets and ConfigMaps associated with the ModelRegistry CR (as described above in [Terms](#terms) are handled by the backend route and performed by the service account. Requests to the backend pass along the database password or certificate information and the underlying resources are abstracted away.)

### Autofilling Model Locations from Connections

When registering a model or version, a user can choose to pre-fill the location of that model from a Connection in a project they have access to. This does not create a reference to that connection in the model registry. Instead, the non-sensitive details from that Connection (S3 endpoint/region/bucket, OCI URI, etc) are copied into the form fields used to construct the URI that will be stored in an artifact in the registry API.

However, the name of that connection is stored in the `storageKey` property of the artifact. This value is then used in the deploy model form as the default name if the user must create a new connection. See [Integration with Other Features](#integration-with-other-features) and [Technical Debt](#technical-debt).

### Usage of `proxyService` and Resolving the API URL via a Service Annotation

The dashboard integrates with the Model Registry API service through the `proxyService` mechanism in order to route requests through the dashboard pod and avoid CORS restriction issues. This usage is configured in [backend/src/routes/api/service/modelregistry/index.ts](https://github.com/opendatahub-io/odh-dashboard/blob/main/backend/src/routes/api/service/modelregistry/index.ts).

Unlike other `proxyService` usages which forward requests directly to the service's internal address, the MR proxy route fetches the `routing.opendatahub.io/external-address-rest` annotation on a registry's associated k8s Service to determine the external address of the Model Registry API and forwards requests to that address. This is a requirement for Service Mesh support, but it is no longer a strict requirement of the API since Service Mesh is no longer a required component for Model Registry. This annotation URL remains and still works without Service Mesh, and we keep the proxy implemented this way in the dashboard to support the corner case where a user might want to enable Service Mesh even though it is not required.

When running the dashboard backend locally for development this external annotation URL will not work, so in this mode we require port-forwarding directly to the service. This only works if Service Mesh is not being used in the cluster (which is now the default configuration).

### Workaround for Arbitrary Updates to Last Modified Timestamps

Each of the API objects (model, version, artifact) have their own `lastUpdateTimeSinceEpoch` value which automatically updates to the current time when that object is patched. We show this value in the models table and versions table as "Last modified".

When a user performs some operation that updates a child resource, logically we want to present to the user that its parent resources have updated. For example, editing a version should bump the "last modified" timestamp for both the version and its parent model. This also applies for certain operations that don't technically modify anything in the registry API: when you deploy a version, new information appears in the Deployments tab for that version in the dashboard, so we want to show that version and that model as updated and allow the user to find recent deployments by sorting models/versions by their last modified time.

The registry API does not support a "touch" operation that updates the `lastUpdateTimeSinceEpoch` on a resource without making any changes. In order to achieve this behavior we need to update something. Our workaround is to create our own `_lastModified` custom property which we set to the current time manually. This is a write-only operation; we still use `lastUpdateTimeSinceEpoch` as the source of truth, we just update it by changing our custom property.

## Technical Debt

- We have raised the above timestamp workaround as technical debt and requested the MR team create a way for us to "touch" or "bump" a resource without editing it, but the MLMD backend used by model registry does not allow for this so we are stuck with our workaround.
- The API responses for listing models and versions have a default limit and expect pagination parameters, but the dashboard does not currently integrate with the API's pagination. In order to request all items from a list, the dashboard passes a hard-coded `?pageSize=99999` parameter.
- The Model Registry API has a resource called `InferenceService` that we do not use. This is different from the k8s `InferenceService` used by Model Serving. There is some automation in the model registry operator to create/synchronize MR API InferenceService objects with the k8s InferenceServices that reference registered versions they were deployed from. If desired in the future the dashboard could use these to list deployments of a version instead of relying on the k8s InferenceServices and reusing Model Serving dashboard code.
- Model locations are stored as a single URI string in ModelArtifact objects. Other components such as Model Serving rely on model locations possibly being specified with multiple fields. For example, an S3 model has an endpoint, region, bucket and path that are needed as separate fields for Model Serving. For S3 models, we are using a non-standard URI format to combine these values into a single string and parse them out when they are needed as separate values (format: `s3://{bucket}/{path}?endpoint={endpoint}&defaultRegion={region}`). This is not desirable -- we should advocate for a way to store model locations as multiple distinct fields in the registry.
- Our routes and pages for model details, the versions list under a model and the version details are duplicated with separate paths/components of each view for live models and registered models. While there is some abstraction of the components used, we may want to consider refactoring to deduplicate this code more so that there is a single route/page for each view regardless of whether the model/version is archived.
- When the user uses "Autofill from connection" while registering a version, instead of only storing the connection name (`storageKey` in the artifact) we should have a more robust mechanism for referencing and possibly reusing the original Connection secret. Design discussions for this are in progress.

## Integration with Other Features

### Model Serving

Users can deploy models from the model registry using the "Deploy" button on a model version details page or the "Deploy" action in the kebab menu on a row of a model's version list page. Upon clicking Deploy, a wrapper component `DeployPrefilledModelModal` is rendered which gathers the data necessary to pre-fill the dashboard's existing Deploy Model form (part of the model serving feature). The user first selects a project and we check if that project has a serving platform enabled:

- If so, the rest of the form is rendered and pre-filling logic is triggered.
- If not, the user is directed to first go to the Projects page and choose a model serving platform for that project, then return to the model registry to try deploying again. The link to the Projects page includes extra URL parameters that reference where the user came from in model registry, and when those parameters are present the Projects page renders a "Navigate to Model Registry" link after a serving platform is selected.

Once a project with a serving platform is selected, the model serving feature's existing Deploy Model form is rendered as a child and an effect triggers that attempts to pre-fill the model location information in the form fields. Using the model location details parsed from the URI of an artifact in the model registry, the dashboard looks for existing Connections matching that location in the deployment project:

- If one matching connection is found it is pre-selected.
- If more than one matching connection is found, none are pre-selected but all are marked as "Recommended".
- If no matching connections are found, the "Create a connection" option is pre-selected and the model location details are pre-filled, leaving the user only to fill in their credentials.

The user can then submit the form to deploy the model, after which they are navigated to the Deployments tab under the version they came from in the registry. This Deployments tab reuses code from the model serving feature's "Model deployments" table, showing the same rows filtered to only show models associated with the registered version.

### Model Catalog

A model in the Model Catalog can be registered to a model registry. A similar registration form is rendered under the model catalog page which excludes model location fields and asks the user only for names and descriptions for the registered model and version.

When a model is registered from the catalog, `modelSource*` properties on the artifact are used to reference the catalog model it originated from. Those properties are then used to render a "Registered from catalog" link on the version details page which takes the user back to the original model page in Model Catalog.

### Pipelines

The Model Customization / InstructLab pipeline flow in the dashboard is initiated with the "LAB-tune" button on a registered version details page, which takes the user to the model customization form. That form creates a pipeline run which generates a new model image and registers it as a new version using the model registry's Python client. When the pipeline run registers a version this way it includes `modelSource*` properties on the artifact. These are then used to render a "Registered from pipeline run" link on the version details page which then takes the user to the pipeline run details page.
