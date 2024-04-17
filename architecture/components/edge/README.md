# Edge Architecture

The AI Edge component is focused on extending AI workflows to Near and Far Edge environments through the use of
MLOps Pipelines and integration with [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) workflows to deliver and serve AI/ML models to Edge environments.

## Terminology
### Inference Container
* Container image where the pre-trained model, its dependencies and serving runtime are packaged into the image during the build proces with the expectation that
  model inferencing will be available in any environment where OCI container workflows are supported.

### Core Cluster
* The central OpenShift cluster that is used to manage the edge clusters.
* Can be an on-premise or cloud-based cluster.
* There are no resources or network constraints expected in the core cluster.

### Near Edge Cluster
* This is an edge environment to run and serve AI/ML inference workloads.
* The near edge environment is represented by separate OpenShift cluster(s) where AI/ML models can be configured and deployed using GitOps workflows
* The near edge environment is expected to have moderate yet constrained compute resources and network.
* Doesn't necessarily have any open ports for inbound connections.

## Components
### OpenShift Pipelines
OpenShift Pipelines provide the orchestration we use to:
* Fetch the model from a storage location(Object Storage, Git, Model Registry,...),
* Build the inference container
* Test the inferencing container in a staging environment that is representative of the final production environment where the inference server will be running
* Update a GitOps repository with the latest build of the inference container for deployment into the Near Edge environment

### GitOps Operator
As part of ODH integration with GitOps workflows, we are focused on supporting [OpenShift GitOps](https://docs.openshift.com/gitops/) workflows to provide guidance
and best practices for deploying intelligent applications to edge environments using Argo CD.

### Open Telemetry
Observability into the operation of the AI/ML models is supported OpenTelemetry features for collection and tracing of models operating in Edge environments with full support for forwarding metrics back to the Core or Near Edge cluster(s)

## Architecture Diagrams
* [High level AI Edge Architecture Diagram](odh-edge-architecture-high-level-architecture.png)
* [MLOps Pipelines Diagram](odh-edge-architecture-mlops-pipeline.png)
