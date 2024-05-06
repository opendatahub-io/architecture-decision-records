# Edge Architecture

The AI Edge component is focused on extending AI workflows to Near and Far Edge environments through the use of
MLOps Pipelines, backed by [OpenShift Pipelines](https://docs.openshift.com/pipelines/1.14/about/op-release-notes.html), and integration with your [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) workflows to deliver and serve AI/ML models to Edge environments.  This AI Edge project is focused on exploring Edge workflows to understand where we can add OpenShift AI features to extend AI workflows from the core to the edge.

## Terminology
### Inference Container
* Container where the pre-trained model, its dependencies and serving runtime are packaged into the image during the build process with the expectation that
  model inferencing will be available in any environment where OCI container workflows are supported.  This allows for precise control over the security, stability and versions of a model and its dependencies when deploying into resource restricted environments where limited network connectivity and lack of external services require
  an inference services that can works across a wide range of environments.

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
* Fetch the model from a storage location (Object Storage, Git, Model Registry,...).
* Build the inference container image.
* Test the inferencing container in a staging environment to verify key functionality before integration with GitOps workflows.
* Update a GitOps repository with the latest build of the inference container image for deployment into the Near Edge environment.

### GitOps Workflow
As part of ODH integration with GitOps workflows, we are focused on supporting [OpenShift GitOps](https://docs.openshift.com/gitops/) workflows to provide guidance
and best practices for deploying intelligent applications to edge environments using Argo CD.  Our primary goal is to support automated workflows that can onboard the latest version of an [Inference Container image](#Inference-Container) build into your existing GitOps configuration ensuring that your deployments are current independent of the GitOps tooling deployed into your environment

### OpenTelemetry
Observability into the operation of the AI/ML models is supported OpenTelemetry features for collection and tracing of models operating in Edge environments with full support for forwarding metrics back to the Core or Near Edge cluster(s)

## Architecture Diagrams
* [High level AI Edge Architecture Diagram](odh-edge-architecture-high-level-architecture.png)
* [MLOps Pipelines Diagram](odh-edge-architecture-mlops-pipeline.png)
