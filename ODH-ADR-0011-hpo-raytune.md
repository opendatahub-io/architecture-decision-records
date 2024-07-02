# GitHub Label Standard for opendatahub-io organization

|                |            |
| -------------- | ---------- |
| Date           | 2023-04-07 |
| Scope          | |
| Status         | Draft |
| Authors        | [Dinakar Guniguntala](@dinogun), [Kusuma Chalasani](kusumachalasani), [Bharath Appali](bharathappali) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This document proposes the integration of [RayTune](https://docs.ray.io/en/latest/tune/index.html) Hyper Parameter Optimization (HPO) Library as the primary choice for HPO for OpenShift AI. Katib was an alternative that was explored and it appears that it needs more work before it can be integrated into OpenShift AI at this time. Katib integration will be covered by a separate ADR.


## Why

HyperParameter Tuning / Optimization is a critical capability for data scientists to get the best model performance and is needed in the OpenShift AI featureset.

The RayTune project is part of the overall [Ray](https://github.com/ray-project/ray) framework and requires minimal integration into OpenShift AI. RayTune supports a broad range of algorithms as well as integrates with other HPO tools such as Ax, BayesOpt and Optuna. RayTune has an active community, last release with RayTune updates was in Dec 2023.

RayTune is designed for distributed computing, enabling scalability across multiple nodes for computation. 

RayTune does not expect users to be kubernetes savvy which lowers the bar for entry and makes it more flexible from a data scientist perspective.

HPO results generated from RayTune do not have to adhere to any specific format and so make it reusable even outside the context of RayTune.

RayTune runs all its trials as part of the same namespace as it was invoked in and so adheres to the RBAC model enforced by OpenShift AI.


## Goals

* Provide a way for users to incorporate HPO into Openshift AI either independently or as part of a model training run
* The solution needs to be available by April 2024
* Evaluate RayTune and Katib specifically against the following
  * User experience
  * Presence / absence of any major functionality / feature
  * Compute resources needed for regular use
  * Support for Multi Tenancy
  * Support storing metadata using Kubeflow MLMD
  * Community


## Non-Goals

* Implementation of new features to fill in any gaps noticed

## How

RayTune, being already part of the Ray ecosystem, requires minimal integration into the current OpenShift AI framework. We will create example [code](https://github.com/kruize/hpo-poc/tree/main/demos) that details the steps needed by a Data Scientist to invoke a HPO run using RayTune inside OpenShift AI. The code will also have [samples](https://github.com/kruize/hpo-poc/blob/main/demos/raytune-oai-MR-gRPC-demo.ipynb) for storing HPO metadata into the OpenShift MLMD.
List of meta data types used by HPO Experiments:

| Name                  | Metadata Type | Description     | Existing / New |
| --------------------- | ------------- | --------------- | -------------- |
| odh.Dataset           | Artifact      | Represents a dataset used in the model training process. | New      |
| odh.ModelArtifact     | Artifact      | Represents a trained machine learning model.             | Existing |
| odh.Train             | Execution     | Tracks the execution of the model training process.      | New      |
| odh.Metrics           | Artifact      | Stores evaluation metrics generated during model training or inference.  | New  |
| odh.HPOConfig         | Artifact      | Captures the hyperparameter configuration for all trails of HPO experiments. | New  |
| odh.HPOTrial          | Context       | Provides information about individual trials conducted within an experiment, aiding in experiment management. | New   |
| odh.HPOExperiment     | Context       | Serves as a parent context for trials, facilitating the organiszation and comparison of experiments.          | New   |


## Alternatives

RayTune and Katib were evaluated for this proposal
1. RayTune supports a few additional algorithms that are not part of the Katib at this time. This includes Ax (Adaptive Experimentation), Dragonfly, BlendSearch, CFO (Cost Frugal Hyperparamater Optimization), HEBO (Heteroscedastic Evolutionary Bayesian Optimization), Nevergrad, ZOOpt, SigOpt and Repeater.
1. RayTune integrates seamlessly into OpenShift AI workflow as it is part of the Ray framework that is currently in use by OpenShift AI. It does not require any additional setup. HPO trials run as processes on Ray clusters.
1. Katib is part of the Kubeflow framework and requires atleast one additional operator, viz Katib Operator, to be installed and configured for it to work with OpenShift AI.
1. Additional operators such as the training operator may need to be installed to fully utilize all of the functionality offered by Katib (Eg. Parallelism)
1. Katib has specific result logging requirements that might not be useful outside of Katib.
1. Katib has better trial separation, it spawns a pod for every trial. Also Katib introduces a sidecar container for metrics collection for each pod.
1. Katib requires container images to be created that follow the Katib spec. Parameters from the trial runs can be parsed in one of three supported formats, viz, 1. Stdout, 2. prometheus, 3. file based. This also means that the users need to be Kubernetes / container savvy.

The table below captures resource usage for RayTune while running a typical Distributed PyTorch job
| Workers | CPU (Cores) | Memory (GiB) | Time Taken (mins) | Comments |
| ------- | ----------- | ------------ | ----------------- | -------- |
| 1       | 3           | 10.8         | 21                |          |
| 2       | 3.93        | 13.8         | 13                |          |
| 4       | 4.82        | 18.8         | 12                |          |


## Security and Privacy Considerations

Using RayTune does not need any additional work to support multi-tenancy. Since RayTune trials run as jobs in a worker pod within the same namespace, the OpenShift RBAC currently in effect on a given OpenShift AI Project is sufficient to isolate the pods from other OpenShift AI namespaces / projects.


## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |


## References

1. [Ray Tune: Hyperparameter Tuning](https://docs.ray.io/en/latest/tune/index.html)
2. [GitHub - kubeflow/katib: Repository for hyperparameter tuning](https://github.com/kubeflow/katib)
3. [An overview for Kubeflow Katib](https://www.kubeflow.org/docs/components/katib/overview/)
4. [Katib Logging Format](https://invisibl.io/kubeflow-automl-experimentation-katib-kubernetes-mlops/)
