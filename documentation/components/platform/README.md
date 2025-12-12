# Platform Architecture

Platform component is responsible for maintaining the core ODH Operator and establishing standards for component
deployments, monitoring, security and ecosystem integration.

## ODH Operator APIs

### DSCInitialization API

* [API Reference](https://github.com/opendatahub-io/opendatahub-operator/blob/incubation/docs/api-overview.md#dscinitializationopendatahubiov1)
* This CRD is responsible for defining config required by the ODH platform before the applications are deployed.
* This includes creation of applications and monitoring namespaces, component wide configurations like Authorization,
monitoring etc

### DataScienceCluster API

* [API Reference](https://github.com/opendatahub-io/opendatahub-operator/blob/incubation/docs/api-overview.md#datascienceclusteropendatahubiov1)
* This CRD will be created by the end user to enable various data science components.
* It is responsible for enabling support for Notebooks, DataSciencePipelinesApplication, InferenceService etc based on
  the configuration

## Platform Architecture Overview
![Platform Architecture Overview](./Platform%20Architecture%20Overview.png)

## Authorization in ServiceMesh
![Authorization in ServiceMesh](./Authorization%20in%20Service%20Mesh.png)
