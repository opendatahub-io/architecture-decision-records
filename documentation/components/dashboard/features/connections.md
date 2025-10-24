# Connections

> Introduced in 2.16 -- Encompasses & replaces Data Connections

> **Note**: This document focuses on Dashboard-specific implementation details and Connection Types UI/UX. For the official Connection API definition, including annotations and protocol specifications, see [ODH-ADR-Operator-0009-connection-api.md](../../../../architecture-decision-records/operator/ODH-ADR-Operator-0009-connection-api.md).

* [Introduction](#introduction)
* [Connection Types & Corresponding Connections](#connection-types--corresponding-connections)
   * [How Editing Impacts Things](#how-editing-impacts-things)
* [Out of the Box (ootb) Offerings](#out-of-the-box-ootb-offerings)
* [Connectivity](#connectivity)
  * [Workbench Connections](#workbench-connections)
  * [Model Serving Connections](#model-serving-connections)

## Introduction

There are three terms we are working with here:

* **Data Connections** -- The "old" (no longer used) term. This indicated the S3-compatible Data Connection that we had prior to 2.16; moving forward these are _Connections_ based on the S3 _Connection Type_ (which comes [ootb](#out-of-the-box-ootb-offerings))
* **Connection Types** -- These are like templates for new _Connections_. These are crafted & managed by the RHOAI admins and stored inside the deployment namespace; some come [ootb](#out-of-the-box-ootb-offerings)
* **Connections** -- These are the instances inside a Data Science Project that can be connected to Workbenches & Model Serving Models, they are always based off a _Connection Type_

## Connection Types & Corresponding Connections

Connection Types are a form-driven way of adding a structured object that details the fields and structure of a Connection interface. These provide the RHOAI admin with some flexibility to how to structure Connections for their users.

Connection Types are a configmap in the deployment namespace. They are managed in the Admin Settings page. There is a preview button to see what the Connection will look like built into the form builder and should allow good coordination on how it will work for users.

Connections are project-based and always built off of one of the Connection Types that are accessible (created and enabled) to the user at time of creation. They are saved as Secrets inside the project.

Connection types are of this structure:
```typescript
type ConnectionTypeConfigMap = K8sConfigMapResource & {
  metadata: {
    annotations?: DisplayNameAnnotations & {
      'opendatahub.io/disabled'?: 'true' | 'false';
      'opendatahub.io/username'?: string;
    };
    labels: DashboardLabels & {
      'opendatahub.io/connection-type': 'true';
    };
  };
  data?: {
    category?: string;
    // JSON array of ConnectionTypeFields
    fields?: string;
  };
};
```

Each `ConnectionTypeField` is a configuration of a type of field. Read more about the [Dashboard Labels & Annotations over here](../k8sLabelsAndAnnotations.md).

The supported field types today are:
* BooleanField
* DropdownField
* FileField
* HiddenField
* NumericField
* SectionField
* ShortTextField
* TextField
* UriField

> Note: Each of these have fields to configure read-only, required, and varying configurations based on the type. There is quite a bit of variability here, so the details can be added if that kind of granularity is needed.

### How Editing Impacts Things

Editing an existing connection attempts to re-present the same look and feel at the time of creation with a few exceptions:

* If the Connection Type has since been modified
  * & new fields added
    * Then the new fields will be accessible
    * There should be limited impact, with exceptions
      * If the new fields are required, they will prevent users from resaving changes in their Connections until they are updated with new values
      * If the new fields have defaults, it will require the users to edit and resave the Connections
  * & existing fields removed
    * If they were not used, it will be pretty seamless to the form experience
    * If they were used, the field will be marked with very little information [1] & will just be the environment variable name to its value
  * & changes an existing field's type
    * The field should remain as-is in the old type/value until otherwise modified
* If the Connection Type has since been deleted
  * The Connection edit screen will have very little information [1] and will entirely just be a listing of environment variable names to their value

> [1] All the metadata comes from the Connection Type not the Connection itself; metadata such as: section information, the display name of the field, defaults, readonly, required, etc

It is worth noting that if the Connection Type is just disabled, we will still pull the configuration details during edits, but it cannot be used in future creations until it is re-enabled. This gives an avenue to use this functionality as a way to version existing Connection Types without encountering issues with existing Connections.

## Out of the Box (ootb) Offerings

Admins can disable this offering and/or duplicate it to provide defaults or read-only aspects to help their users with some information.

> Note: It is important to note that since the Connection Types are stored as ConfigMaps, passwords and other credential information are exposed in plain text if stored in the Connection Type. We do not recommend storing this kind of information in Connection Types at this time.

#### S3 compatible object storage - v1
To help with existing usages before the upgrade 2.16, we naturally continue to have support for S3. 

#### URI - v1
We also provide a URI ootb variant to help with connecting public [2] URL models to model serving.

> [2] At this time, private connections are not supported

#### OCI compliant registry - v1
With the OCI compliant registry ootb connection type, users are able to connect to a private container registry by providing a pull secret. Due to the presence of the `.dockerconfigjson` env variable, the created connection becomes a secret of type `kubernetes.io/dockerconfigjson` which can be used the same way traditional pull secrets are used in kubernetes, with the additional fields from the connection type.

To connect to a public container registry, a user can use the "URI - v1" connection type and provide the URI to the image tag and prepending it with `oci://`

## Connectivity

Connections don't do a lot by themselves; they effectively store configurations about how to connect to another source [3]. They reside inside projects and can connect to a few other resources that share the same project. Each interacts slightly different, so lets cover what those scenario details are.

> [3] Technically speaking, a Connection Type can template any type of information which does not need to reflect relationships with another storage. They can store reusable variable values so you can share them with multiple Workbenches -- [more details below](#workbench-connections).

### Workbench Connections

Workbenches are by far the most flexible of Connection consumers. All Connections are connected via `envFrom` (see below for an example), which injects all the keys of the Secret as environment variables. Consumption of the data can be done through the Workbenches' standard access to the environment variables (in Python that's `os.environ["ENV_NAME_HERE"]`).

Under the hood -- the connectivity between a Connection and a Workbench exists as such:

```yaml
apiVersion: kubeflow.org/v1
kind: Notebook
metadata:
  name: example-workbench
  # ...other properties
spec:
  template:
    spec:
      # ...other properties
      containers:
        - name: the-notebook-container
          # ...other properties
          envFrom:
            - secretRef:
                name: my-s3-connection
            - secretRef:
                name: my-uri-connection
```

The `my-s3-connection` (using the ootb S3 Connection Type) & `my-uri-connection` (using the ootb URI Connection Type) are connected via the `envFrom` section on the notebook container. Since all Connections are secrets & are injected the same way as environment variables, it will always be mounted from `envFrom.secretRef.name` for each Connection irrespective of their structure.

> Note: It is important to note that since they are injected as environment variables, two Connections sharing the same variable will clobber each other and the "last one" wins. The UI will note this concern when you have two Connections overlapping.

### Model Serving Connections

> Note: Due to the complexities of how Connections integrate with Model Serving, limited use-cases are available to Model Serving.

Essentially we only have support for these types:
* [S3-compatible](#s3-compatible-connection)
* [URI](#uri-connection)
* [OCI model cars](#oci-model-cars-connection)

> Note: At this time there is not much else that can be done as it requires specific integration logic in order to connect a specific set of fields from the Connection to align it with the implementation of the Serving feature (KServe, Model Mesh, etc).

#### S3-compatible Connection

> Pulling a model from an S3-compatible Bucket

Like _Data Connections_ in the previous world, these operate identically through the storage property.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: model-example-using-s3
  # ...other properties
spec:
  predictor:
    # ...other properties
    model:
      # ...other properties
      storage:
        key: my-s3-connection
        path: the/path/in/my/bucket
```

The `storage.key` is the Connection Secret. Note the `path` value is still used to qualify where in your S3 Connection bucket the model will be.

#### URI Connection

> Pulling a model from a public URI

A new feature with the initial release of the Connection Types.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: model-example-using-uri
  # ...other properties
spec:
  predictor:
    # ...other properties
    model:
      # ...other properties
      storageUri: 'https://the-url-to-my-model.com/path'
```

The `storageUri` path is queried for the model and installed into the pod that is associated to your deployment.

> Note: The `storageUri` field is an overloaded one in the KServe documentation and can have wider implications for usage. Anything that 

#### OCI Model Cars Connection

> Pulling a model from an authenticated OCI container registry

OCI is only supported on KServe single model serving deployments. Additionally, the image must be in a Modelcar[^Modelcar] format specified by KServe.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: model-example-using-oci
  # ...other properties
spec:
  predictor:
    imagePullSecrets:
      - name: oci-connection
    # ...other properties
    model:
      # ...other properties
      storageUri: 'oci://quay.io/someregistry/image:tag'
```

The `imagePullSecrets` points to the OCI connection.

The `storageUri` path starts with `oci://` and points to an image.

[^Modelcar]: https://kserve.github.io/website/latest/modelserving/storage/oci/#prepare-an-oci-image-with-model-data