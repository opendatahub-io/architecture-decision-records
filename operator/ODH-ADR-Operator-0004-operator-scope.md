# Open Data Hub - Operator Scope

|                |            |
| -------------- | ---------- |
| Date           | Sep 5th, 2023 |
| Scope          | Open Data Hub Operator |
| Status         | Draft |
| Authors        | [Edson Tirelli](@etirelli), [Vaishnavi Hire](@VaishnaviHire) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [Tracker issue](https://github.com/opendatahub-io/opendatahub-operator/issues/158) |
| Other docs:    | [TODO: add link to ODH Operator Design](http://) |

## What

OpenShift and Kubernetes operators can be Namespace scoped, Multi-namespace scoped or Cluster scoped. The choice of the 
scope has impacts both on required permissions as well as operator capabilities. 

The ODH Operator v2.x is a meta-operator that manages a number of resources and other operators in the OpenShift cluster. 
As such, the operator was implemented as a Cluster Scoped Operator. This ADR captures the reasoning and impact of the 
decision.

## Why

The ODH operator acts like a meta operator, installing and managing other operators and resources. In particular the operator manages the following types of resources, among possibly others:

- Namespaces
- Deployment
- Role
- ClusterRole
- RoleBindings
- ClusterRoleBindings
- ConfigMaps
- Secrets
- Service
- NetworkPolicy
- Route
  
The operator deploys the following Custom Resource Definitions:

- DSCInitialization
- DataScienceCluster
 
The operator (optionally) requires and/or manages the following dependent operators, among others:

- KServe
- ModelMesh
- Ray
- Codeflare
- ServiceMesh
- Serverless
- NFD 
- NVIDIA GPU
- openshift-pipelines

In order to properly manage these resources, the operator requires access and permissions across namespaces. In particular, the operator creates and/or manages the following namespace (the namespace name can be changed via configuration):

- opendatahub
  
The operator also leverages the platform’s [Owner References](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) capability that ensures all dependent resources are owned by the operator and their life cycles are tracked and managed cleanly. Kubernetes however restricts the use of owner references in a way that only cluster scoped resources can own other cluster scoped resources, leading to the requirement of the operator being cluster scoped. 

## Goals

* The operator needs to support management of resources across namespaces
* The operator must track all resources it creates and manages
* The operator must remove resources created by a managed custom resource when that custom resource is removed

## Non-Goals

* 
  
## How

The ODH operator v2.x is set to Cluster scope.

## Alternatives

We considered the use of a namespace scoped operator, but the following limitations were determinant to choose the Cluster scope instead:

### 1. Some dependent resources are cluster scoped 
Some of the resources/dependencies that OpenShift AI requires are cluster scoped (e.g. Serverless, ModelMesh, etc). Managing these resources/dependencies requires the operator to also be cluster scoped.
The operator also introduces two new CRDs (DSCInitialization) and (DataScienceCluster) that are cluster scoped. 

### 2. Tracking and management of dependent resources
One of the drivers for the design and development of the ODH v2 operator was the need to  increase the resilience and reliability of the tracking and management of dependent resources. In particular the operator should provide accurate, clear and reliable diagnostics in case of problems with dependencies, and be able to properly maintain a clean cluster in case of upgrades or uninstalls. 
The operator leverages platform capabilities for that task, like owner references. Namespaced operators cannot own resources outside their own namespace (as per Kubernetes design). Scoping the operator to a namespace would require us to stop using owner references and implement from scratch the management of resources, using a different mechanism like labels. That would be complex, brittle, and duplicate functionality available in the platform itself.

### 3. Contradictory requirements with other users and best practices
The use of namespaced operator would prevent us from meeting the requirements of other customers and users (e.g. automated install and upgrades of resources across namespaces). 

## Tradeoffs

The main drawback of using a cluster scoped operator is the impossibility of running multiple instances and/or versions of RHODS in the same cluster. Users that need multiple instances/versions of RHODS are required to use one cluster for each. For cases where cost is a concern, alternatives like Hypershift can be considered.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| ODH Platform Team             | Vaishnavi Hire   | 2023/09/05 | ? |

## References

* ODH Operator Design document

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| [Vaishnavi Hire](https://github.com/VaishnaviHire) | Sep 15, 2023 |       |
| [Wen Zhou](https://github.com/zdtsw) | Sep 15, 2023 |       |
| [Trevor Royer](https://github.com/strangiato) | Sep 15, 2023 |     |

