# Model Catalog

> Introduced in 2.19 as Tech Preview

- [Introduction](#introduction)
  - [Differences between Model Catalog and Model Registry](#differences-between-model-catalog-and-model-registry)
  - [The Current Implementation Is Temporary](#the-current-implementation-is-temporary)
- [Terms](#terms)
- [Example Data](#example-data)
  - [`model-catalog-sources` ConfigMap (truncated)](#model-catalog-sources-configmap-truncated)
- [Implementation Notes](#implementation-notes)
  - [Catalog Models Have No Single Unique Identifier](#catalog-models-have-no-single-unique-identifier)
  - [Special Labels](#special-labels)
- [Integration with Other Features](#integration-with-other-features)
  - [Model Registry](#model-registry)
  - [Model Serving](#model-serving)
  - [Home Page](#home-page)

## Introduction

The Model Catalog feature provides a centralized list of AI models available for anyone with access to the dashboard to browse, register and deploy. Catalog models are grouped into multiple "sources" such as Red Hat and Third-Party. For each model, the catalog displays metadata such as the description, "model card" (longer readme markdown), version tag, labels/tasks, Model location (URI), license, provider and publication date.

### Differences between Model Catalog and Model Registry

- There is one centralized model catalog in a cluster. There can be multiple model registries in a cluster (although they all exist in the same centralized namespace).
- The model catalog needs no configuration before it can be used. Model registries need to be created/configured by an administrator.
- The model catalog has no access restrictions. Model registries can be separately restricted to specific users/groups/projects.
- The model catalog is read-only. Users can add (register) models to a model registry.

These are the differences as currently implemented. Some of this may change with future enhancements; for example, there is a desire to allow an admin to configure which models/sources a user has access to instead of the unrestricted access of the current implementation.

### The Current Implementation Is Temporary

The initial version of Model Catalog introduced in 2.19 and enhanced in 2.20 is built without a real backend API. Model metadata is stored in a ConfigMap on the cluster, which is [created automatically by the dashboard's manifests](https://github.com/opendatahub-io/odh-dashboard/tree/main/manifests/rhoai/shared/apps/model-catalog). No external sources are being accessed, we are hand-curating a set of models to be included with each release. This implementation is not sustainable and was only chosen to facilitate early demonstrations and user research. In the near future this ConfigMap approach will be replaced by the consumption of a new Model Catalog API provided alongside the Model Registry API. This will allow the dashboard to fetch and search/filter remote repositories of model metadata and present those models in the catalog.

For more details on the temporary ConfigMap implementation and how the model metadata is updated, see [this README in the model registry repo](https://github.com/opendatahub-io/model-registry/tree/main/model-catalog).

## Terms

- **Model Catalog** -- The feature and page where models can be discovered and then registered or deployed.
- **Model Catalog Source** -- A group of models presented in its own section of the model catalog page. Each `ModelCatalogSource` object has a `source` string property (the name of the source) and a `models` array property.
- **Catalog Model** -- A model in a catalog source. These are the elements of the `models` array in a catalog source object and have the type `CatalogModel`. They contain all the metadata about a model including name, provider, description, readme, logo, license, artifacts, etc.
- **Provider** -- A property on each model which is _**distinct from the name of that model's source (parent)**_. For example, we have a source called "Third-Party" and it includes models with the provider "Meta Llama".
- **Artifact** -- Similar to an artifact in Model Registry, this is a small object containing the URI of the actual model location. Unlike a model registry artifact this is not its own first-class object but instead just an element of the `artifacts` array property of a catalog model. An artifact also has a `tags` string array used as the model's version (see Version/Tag below).
  - **Note about dashboard handling of artifacts**: Just like in Model Registry, _**the dashboard assumes a model will always include only one artifact and that artifact will always include only one tag**_ which is its single definitive version. If there are more artifacts or more tags on the first artifact, they are ignored. The artifact's URI and first tag are presented as if they are top-level details of the model and the concept of artifacts is hidden from the dashboard user. This choice was convenient for the initial temporary ConfigMap implementation and may need to be revisited when we move to the new Model Catalog API.
- **Version/Tag** -- What the dashboard displays as the "Version" of a model in the catalog is the first tag of the first artifact of a model. In places where we need an identifier for a model, we pass around a singular `tag` value which is this same first tag from the first artifact (See [Catalog Models Have No Single Unique Identifier](#catalog-models-have-no-single-unique-identifier)).
- **Labels and Tasks** -- There are two arrays of strings that function as labels on a catalog model, `labels` and `tasks`. The `labels` array is more general-purpose, and the `tasks` array is specifically for listing tasks the model is intended for such as `"text-generation"`. In the dashboard, the model catalog overview page (where models appear as small cards) only shows the `tasks` for each model (with some exceptions, see [Special Labels](#special-labels)) and the model details page shows both `labels` and `tasks` combined in its "Labels" section. UX-wise, `tasks` function as a subset of "Labels".
- **Model Catalog Sources ConfigMap** -- The current (temporary) mechanism for storing the list of catalog sources and their model metadata. The data in the ConfigMap is formatted as a JSON object containing a `sources` array whose elements are model catalog source objects. As of 2.20, there are two ConfigMaps used to store catalog sources, both stored in the installation namespace (`opendatahub`):
  - `model-catalog-sources` - The main configmap whose contents are defined by the dashboard's manifests. This is a managed resource (it is not editable) and it is how we update and release model metadata in the catalog for now.
  - `model-catalog-unmanaged-sources` - A secondary configmap created with an empty `sources` array. This is not managed (it is editable) and it can be used to populate additional sources/models on a cluster for demonstration purposes. The dashboard fetches both ConfigMaps and appends their `sources` arrays together. Please note that this is not intended to be editable by a real user and was introduced only to facilitate Summit demos where we wanted to present models that were not yet ready for release.

## Example Data

### `model-catalog-sources` ConfigMap (truncated)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-catalog-sources
  labels:
    opendatahub.io/dashboard: "true"
data:
  modelCatalogSources: |
    {
      "sources": [
        {
          "source": "Red Hat",
          "models": [
            {
              "repository": "rhelai1",
              "name": "granite-7b-starter",
              "description": "Base model for customizing and fine-tuning",
              "longDescription": "A custom Red Hat base model instruct tuned only for phase 00, produced by IBM Research specifically for RHEL AI.",
              "license": "apache-2.0",
              "licenseLink": "https://www.apache.org/licenses/LICENSE-2.0.txt",
              "maturity": "General availability",
              "provider": "Red Hat",
              "logo": "data:image/png;base64,<TRUNCATED IMAGE DATA>",
              "labels": [
                "lab-base"
              ],
              "tasks": [
                "text-generation"
              ],
              "languages": [
                "en"
              ],
              "readme": "# Model Card for Granite-7b-starter [Paper](https://arxiv.org/abs/2403.01081) \n### Overview\nGranite-7b-starter is a starting student model built for InstructLab, based on Granite-7b-base. This model can be used to create LAB models via InstructLab.\n <TRUNCATED ADDITIONAL MODEL CARD MARKDOWN>",
              "createTimeSinceEpoch": 1739210683000,
              "lastUpdateTimeSinceEpoch": 1739210683000,
              "artifacts": [
                {
                  "protocol": "oci",
                  "tags": [
                    "1.4.0"
                  ],
                  "uri": "oci://registry.redhat.io/rhelai1/modelcar-granite-7b-starter:1.4.0"
                }
              ]
            },
            <TRUNCATED OTHER MODELS>
          ],
        },
        <TRUNCATED OTHER SOURCES>
      ]
    }
```

## Implementation Notes

### Catalog Models Have No Single Unique Identifier

Unfortunately, the catalog metadata we have does not provide a single unique ID we can use to identify a model. The dashboard requires some way to uniquely identify a model for lookup purposes, to use as params in a URL for that model's page, etc. To solve this, we use a combination of multiple properties of a catalog model when identifying it:

- The `source` the model object is contained in
- The `repository` and `name` properties of the model object
- The first tag (`tags` array element) of the first artifact (`artifacts` array element) within the model object

For example, the URL of the dashboard page for a specific catalog model is `/modelCatalog/{source}/{repository}/{name}/{tag}`. We have utilities for returning these values from a catalog model object and for looking up a catalog model given these values.

Note: If a model in the ConfigMap is missing any of these values (for example, there is no tag in the artifact of a model), **navigating to that model in the dashboard will not work** (the link is not clickable). This can happen if we are using the `model-catalog-unmanaged-sources` ConfigMap with incomplete data, such as when metadata is being gathered for models that are not yet published.

### Special Labels

Certain values of a catalog model's `labels` array have special functionality:

- `lab-base` -- Designates a model as an InstructLab starter model. It is given special formatting (yellow color, text replaced with "LAB starter" microcopy) and it is shown on the overview page along with the `tasks` (when normally `labels` are only shown on the details page).
- `lab-teacher` -- Designates a model as an InstructLab teacher model. Like `lab-base`, it has special formatting (purple color, "LAB teacher" microcopy) and is also shown on the overview page.
- `lab-judge` -- Designates a model as an InstructLab judge model. It also has special formatting (orange color, "LAB judge" microcopy) and is also shown on the overview page.
- `featured` -- The first 4 models with the `featured` label appear in the Model Catalog section of the dashboard home page.

## Integration with Other Features

### Model Registry

Catalog models can be registered to a model registry. On a catalog model details page, a user can click the Register button and see a special Register Model form within the context/breadcrumb of the model catalog page. This form reuses state hooks and form fields from the Model Registry's regular registration form, but adds a dropdown to select a registry (because the normal registration form is already in the context of a registry) and makes the model location read-only since it is provided by the model catalog. Upon registration, the user is taken to the same view as if they had registered the model from within the model registry page.

### Model Serving

Models can be deployed directly from the catalog without being registered first. This functionality is similar to how deployment from the model registry works - The same `DeployPrefilledModelModal` component is rendered and the same type of model location data is passed (in the case of model catalog, this is always an OCI URI).

When deploying from model catalog, however, no credentials are required (because models published in `registry.redhat.io` can use credentials provided by default by the cluster). In the model serving deploy form, there is logic to check if the model URI contains `registry.redhat.io` and bypass the "Existing connection / create connection" fields to instead use the URI directly without a Connection.

### Home Page

As mentioned in [Special Labels](#special-labels), the first 4 models with the `featured` label appear in the Model Catalog section of the dashboard home page. That page uses the same data fetching hooks provided by and used by the model catalog code.
