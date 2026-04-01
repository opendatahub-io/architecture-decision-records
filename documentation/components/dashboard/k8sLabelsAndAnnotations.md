[AcceleratorProfile]: ./README.md#acceleratorprofiles-deprecated

[`openshift.io/display-name`]: #openshiftiodisplay-name
[`openshift.io/description`]: #openshiftiodescription
[`opendatahub.io/recommended-accelerators`]: #opendatahubiorecommended-accelerators
[`opendatahub.io/accelerator-name`]: #opendatahubioaccelerator-name
[`opendatahub.io/sc-config`]: #opendatahubiosc-config

# Dashboard K8s Labels & Annotations

Dashboard has a reputation of using a lot of annotations and labels on various resources. This document should help serve to explain the use-cases behind each.

> Note: Not all resources shown in the Dashboard are K8s driven resources. For those that are not, this page does not have any impact on them.

> Note: This is not a comprehensive list of all labels & annotations used in OpenShift AI, just the ones managed and created by the Dashboard. Specific components may have ever-changing needs, so you should seek out those component's documentation for more information. 

* [Labels](#common-labels)
  * [`opendatahub.io/dashboard`](#opendatahubiodashboard)
* [Annotations](#common-annotations)
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]
  * [`opendatahub.io/recommended-accelerators`]
  * [`opendatahub.io/accelerator-name`]
  * [`opendatahub.io/sc-config`]
* [Specific Use-Cases](#specific-use-cases)
  * [DS Projects](#data-science-projects)
  * [Connection Types](#connection-types)
  * [Connections](#connections)
  * [ImageStreams](#imagestreams)
  * [Notebooks](#notebooks)
  * [ServingRuntime Templates](#servingruntime-templates)
  * [InferenceServices](#inferenceservices)
  * [Storage Classes](#storage-classes)
  * [HardwareProfiles](#hardwareprofiles)
  * [Model Registry](#model-registry)
  * [Model Catalog](#model-catalog)

## Common Labels

Common reused labels in the Dashboard. Key features of labels:

* Is able to be used as a filter in a k8s request
* Must be a restrictive k8s naming structure

### opendatahub.io/dashboard

The most common dashboard label. The initial goal here was to mark all things created by the Dashboard, so we could reverse lookup said resources. This has proven to be a bit over aggressive, adding friction in customers making use of external of the Dashboard flows work with Dashboard flows (eg. gitops).

This is a highly contentious label and will be seeing changes in the near future.

> Note: This concept is deprecated for DS Projects, it is adding no value and is adding confusion to the concept of ["what is a DS Project?"](./README.md#projects---openshift-console-vs-data-science-differences)

> Note: This concept is not entirely deprecated for some resources that have multiple uses, including those outside of OpenShift AI. But all OpenShift AI CRDs should not need this soon.

## Common Annotations

Common reused annotations in the Dashboard. Key features of annotations

* Can be a flexible field to be used for complex metadata or flexible usage of characters that are not K8s-safe (vs Labels)

### openshift.io/display-name

Used heavily by Dashboard UI flows to allow OpenShift AI users to craft a readable & flexible name. 

> Note: This is optional, so we fall back on the resource's k8s name. 

### openshift.io/description

Used almost as heavily as the display-name annotation. This allows for a description of the resource the user is creating. This usually is shown next to the display-name once the resource is created.

> Note: Some resources do not have the use for this, but that's more of an oversight than an intent.

> Note: This annotation is not required, nor tied to the use of the display-name annotation.

### opendatahub.io/recommended-accelerators

> Type: string array of [AcceleratorProfile] k8s names.

This annotation is what we use to suggest a recommended connection between a resource's usage by the user & an accelerator profile created on the cluster. This appears as a tag next to the accelerator dropdown item in the UI.

### opendatahub.io/accelerator-name

> Type: a string value of the [AcceleratorProfile] k8s name.

This annotation is what we use to relate back to an accelerator profile. This is metadata to help with reselection of the right accelerator profile in read & edit modes. This is needed as a way to convert back from the resource values used to the proper profile the user selected. We have a fallback for legacy support for Nvidia GPU, but everything else will fail to locate a profile and show an intermediary custom profile that cannot be mutated in edit modes.

If no accelerator was selected, this value should not appear.

### opendatahub.io/sc-config

> Type: object
```js
{
    displayName: string;
    isEnabled: boolean;
    isDefault: boolean;
    lastModified: string;
    description?: string;
}
```

This annotation is used as internal Dashboard metadata to describe, enable, and set which storage class is the default. This annotation does not affect Openshift default storage classes.

## Specific Use-Cases

### Data Science Projects

* Labels
  * `modelmesh-enabled` - required by Model Mesh to say the project is using model mesh configurations
    > Note: When this is `true`, the project is Model Mesh. When this is `false`, we key off it to say this project is KServe
* Annotations
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]

For the Project Sharing feature specifically:
* Label `opendatahub.io/project-sharing` is used to denote permissions crafted by Dashboard flows & thus show up in the Dashboard UI

### Connection Types

* Labels
  * `opendatahub.io/connection-type` - a value of `true` indicates that the `ConfigMap` represents a connection type
* Annotations
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]
  * `opendatahub.io/disabled` - a `true` or `false` value indicates whether the connection type is disabled
  * `opendatahub.io/username` - the name of the user who created the connection type

### Connections

* Labels
  * `opendatahub.io/managed` - Legacy value. Identifies data connections which are watched by the model mesh controller for the purpose of populating the model serving `storage-config`
* Annotations
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]
  * `opendatahub.io/connection-type` - Legacy fallback (pre-2.16). Used to identify S3-compatible data connections; `s3` is the only supported value. Still recognized as a fallback but superseded by `connection-type-ref`.
  * `opendatahub.io/connection-type-ref` - (2.16+) The primary annotation referencing the connection type schema used to create the connection (e.g., `s3`, `uri-v1`, `oci-v1`)
  * `opendatahub.io/connection-type-protocol` - (3.0+) Explicitly identifies the connection protocol for webhook validation and routing. Supported values: `s3`, `uri`, `oci`. Takes precedence over `connection-type-ref` when both are present.

### ImageStreams

> Note: Out-of-the-box variants of ImageStreams is a Workbench backed feature.

These are configured by the admin in the UI and are provided as out-of-the-box examples.

* General Annotations
  * [`opendatahub.io/recommended-accelerators`]
  * `opendatahub.io/notebook-python-dependencies` - the python dependencies that are included in the image to list to the user
  * `opendatahub.io/notebook-software` - the software that is included in the image to list to the user
* Annotations used primarily by the out-of-the-box images provided by the workbench component
  * (tag) `opendatahub.io/image-tag-outdated` - a `true` or `false` value to say if the image is present for lookup, but not intended for selection
  * (tag) `opendatahub.io/workbench-image-recommended` - the recommended tag to suggest to the user
  * `opendatahub.io/notebook-image-order` - a weighed value to help with organization of the images in display lists for the user
* Annotations used primarily by the Admin UI when created custom Notebook Images
  * `opendatahub.io/notebook-image-desc` - description provided by the user
  * `opendatahub.io/notebook-image-name` - a display name provided by the user
  * `opendatahub.io/notebook-image-url` - the original image value from the user before it's processed for the ImageStream

### Notebooks

> Note: This is a Workbench backed feature.

* Labels
  * `opendatahub.io/odh-managed` - Set to `'true'` to mark notebooks created and managed by the Dashboard. This label is actively set during notebook creation.
  * `opendatahub.io/user` - a translated username; the Dashboard k8s-ifies the user's username so we can compare or look up by user in the future
* Annotations
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]
  * `opendatahub.io/username` - the actual username (related to the Label `opendatahub.io/user`)
  * [`opendatahub.io/accelerator-name`]
  * `opendatahub.io/connections` - a comma-separated list of `<namespace>/<secret-name>` references connecting the workbench to Connection secrets. The namespace prefix allows the system to perform a SubjectAccessReview to verify the user has permission to access the Secret. The connection Secret must be in the same namespace as the Notebook.
  * `opendatahub.io/workbench-image-namespace` - This annotation is used to indicate the scope of a workbench image. If the workbench image is project-scoped, this annotation is added with the workbench image’s namespace. If it’s global-scoped, the annotation is omitted.
  * `opendatahub.io/hardware-profile-namespace` - This annotation is used to indicate the scope of a hardware profile. If the hardware profile is project-scoped, this annotation is added with the hardware profile’s namespace. If it’s global-scoped, the annotation is omitted.
  * `opendatahub.io/accelerator-profile-namespace` - This annotation is used to indicate the scope of a accelerator profile. If the accelerator profile is project-scoped, this annotation is added with the accelerator profile’s namespace. If it’s global-scoped, the annotation is omitted.

### ServingRuntime Templates

> Note: This is a Serving backed feature.

These are configured by the admin in the UI and are provided as out-of-the-box examples. These are stored as OpenShift Templates under the hood, but the admin only ever sees a ServingRuntime when configuring.

* Annotations (when configuring in the admin page)
  * [`openshift.io/display-name`]
  * `opendatahub.io/modelServingSupport` - (managed by the UI) a JSON array string of supported platforms (e.g., `'["single"]'`); options: `'single'`, `'multi'`. Note: ModelMesh (`'multi'`) is deprecated.
  * `opendatahub.io/apiProtocol` - (managed by the UI) the api protocols available; options (one of): 'REST', 'gRPC' 
  * `opendatahub.io/disable-gpu` - (optional, typed in) if the ServingRuntime should not be used with GPUs (aka accelerators)
  * [`opendatahub.io/recommended-accelerators`] - (optional, typed in)

* Annotations (when deploying in projects)
  * [`opendatahub.io/accelerator-name`]
  * `opendatahub.io/template-name` - the runtime used
  * `opendatahub.io/template-display-name` - the display name shown for the runtime
  * `opendatahub.io/serving-runtime-scope` - This annotation is used to identify whether a serving runtime template is project-scoped or global-scoped. 
  * `opendatahub.io/hardware-profile-namespace` -  This annotation is used to indicate the scope of a hardware profile. If the hardware profile is project-scoped, this annotation is added with the hardware profile’s namespace. If it’s global-scoped, the annotation is omitted.
  * `opendatahub.io/accelerator-profile-namespace` - This annotation is used to indicate the scope of a accelerator profile. If the accelerator profile is project-scoped, this annotation is added with the accelerator profile’s namespace. If it’s global-scoped, the annotation is omitted.

### InferenceServices

> Note: This is a Model Serving backed feature.

* Annotations
  * `opendatahub.io/connections` - a single connection reference (secret name, no namespace prefix needed) connecting the InferenceService to a Connection secret for model storage access. The connection Secret must be in the same namespace as the InferenceService.

### Storage Classes

* Annotations
  * [`opendatahub.io/sc-config`] - (managed by the UI) a JSON Blob of storage class metadata

### HardwareProfiles

> Note: HardwareProfiles are GA (`hardwareprofiles.infrastructure.opendatahub.io/v1`) and replace the deprecated AcceleratorProfiles CRD.

* Annotations
  * [`openshift.io/display-name`]
  * [`openshift.io/description`]

### Model Registry

* Labels
  * `opendatahub.io/rb-project-subject` - This label is used to distinguish RoleBindings with the group subject `system:serviceaccounts:{projectName}`, identifying them as specific to project service accounts. This allows us to use group RoleBindings separately for groups and projects, making sure they always appear in the view where they were created without relying on filtering by a string prefix.

  * `modelregistry.opendatahub.io/registered-model-id` and `modelregistry.opendatahub.io/model-version-id` - These labels identify InferenceServices deployed via the model registry UI and get the Model Registry Controller to sync the deployment. They are also used to filter InferenceServices when viewing the list of deployments for a specific model version.

  * `modelregistry.opendatahub.io/name` - This label provides a unique reference to InferenceServices deployed via a model registry. It ensures that models will be listed in the deployments tab of that specific registry, preventing incorrect listing across multiple registries with overlapping model IDs.  

### Model Catalog

Model catalog data is stored in ConfigMaps in the installation namespace (`opendatahub`):

* ConfigMaps
  * `model-catalog-sources` - The main managed ConfigMap containing catalog source and model metadata, defined by the dashboard's manifests. This is not editable by users.
  * `model-catalog-unmanaged-sources` - A secondary editable ConfigMap with an empty `sources` array, used to populate additional sources/models for demonstration purposes.
* Special Model Labels (within the `labels` array of a catalog model object)
  * `featured` - The first 4 models with this label appear in the Model Catalog section of the dashboard home page.
  * `lab-base` - Designates a model as an InstructLab starter model. Shown with special formatting (yellow color, "LAB starter" text) on the overview page.
  * `lab-teacher` - Designates a model as an InstructLab teacher model. Shown with special formatting (purple color, "LAB teacher" text) on the overview page.
  * `lab-judge` - Designates a model as an InstructLab judge model. Shown with special formatting (orange color, "LAB judge" text) on the overview page.
