# Open Data Hub - ODH component Integration with DataScienceCluster

|                |                         |
| -------------- |-------------------------|
| Date           | 2024-01-25              |
| Scope          | OpenDataHub operator    |
| Status         | Draft                   |
| Authors        | [Vaibhav Jain](@vajain) |
| Supersedes     | N/A                     |
| Superseded by: | N/A                     |
| Tickets        |                         |
| Other docs:    | none                    |

## What
This ADR proposes the approach to provide better customization flexibility to all components of Open Data Hub.

## Why

As a part of on-going discussion on [ODH-ADR-Operator-0004-replica-image-support](https://github.com/opendatahub-io/architecture-decision-records/pull/23) ADR, First we start the discussion to define replica count for dashboard and then its scope expands to define Images and replicas for all sub-components.
I am seeing these types of requirements as starting of various customization which could be required for different components as we go forward.
Currently we are trying to define a generic structure which could be applied for all components in monolithic DataScienceCluster CRD but I feel these types of customization would require more flexibility at each sub-component level and generic structure would not be a solution for long term.

#### Example: Introduction of replicas field

Dashboard team has raised requirements to capture replicas value from users in DataScienceCluster CRD because Opendatahub-operator directly provision deployment resources for dashboard. Now if we follow a generic approach to define replicas value for sub-components then `replicas` field would be available at each component level.
Replicas field in Dashboard component makes complete sense but if user defines replicas values of ModelServing and Notebook component then it would be waste of resources because ModelServing and Notebook are operators, which works on leader/slave concept. Even if users spin up the 3 replicas for operator pod, at a time only one pod would be selected as leader and would entertain all requests. Load would not distribute among replicas.

## Goals

- **Increased Component Customization:** To provide customization flexibility at component level so that every component could define their own structure to capture user inputs.
- **Improved fault isolation:** If one component reconciler encounters a fault or failure because of customization, it doesn’t propagate across the entire system.
- **Quicker deployment time:** If a user made configuration changes in a specific component then only that component would reconcile.
- **Enhance team productivity:** Allow individual teams to concentrate on their component deployment and maintenance without being burdened by the complexities of the entire system.
- **Accelerate scalability:** New components could be onboard without causing any impact on existing functionality.

## Non-Goals

- Defining the configuration/parameters of each component 
- Define application/runtime specific configuration.

## How

I am proposing to maintain separate CRDs for each ODH component where every component team designs them as per their need.
Such as :

- DashboardConfiguration
- NotebookConfiguration
- ModelServingConfiguration

DataScienceCluster CRD would be responsible for enabling/disabling the sub-component whereas Component specific CRDs would be used to maintain custom configuration.

**For example:**

In Model Serving, KServe and ModelMesh controller have common dependency as `odh-model-controller` which should be pre-configured.
With current structure users are forced to capture/apply manifest of `odh-model-controller` twice which causes infinite reconciliation issues in DataScienceCluster reconciler.

Ideally such an issue should only cause to reconcile Model Serving configuration but because of a single reconciler, manifest of all components applied infinite times.

I am proposing to maintain something like below:

```
apiVersion: opendatahub.io/v1
kind: ModelServingConfiguration
metadata:
 name: default
spec:
 odh-model-controller:
 replica: 1
 image: quay.io/vajain/odh-model-controller:2.0
 manifests:
 contextDir: config
 sourcePath: ''
 uri: 'https://github.com/opendatahub-io/odh-model-controller/tarball/main'
 kserve:
 replica: 2
 managementState: Managed
 memoryLimit: 1Gi,
 cpuRequest: 100m,
 cpuLimit: 1
 modelmesh:
 managementState: Managed 
```
Every component would have its own CRD and reconciler.

## Open Questions

- How do we handle migration from current monolithic structure to independent CRD structure?
- Where component status would be maintained? In component CRD or In DataScienceCluster CRD?
- Each component could be independently managed by their own CRD. Do we need a DataScienceCluster CRD?

## Alternatives

Another alternative to having separate CRDs would be to be able to specify a reference to a ConfigMap for each component. It's a pattern that a number of operators use, they define their configuration API, but it's stored in a ConfigMap, not a custom resource. That configuration API often extends https://github.com/kubernetes/component-base.

The source of truth would be a Golang struct, exactly the same as for a CRD. Component configuration API / struct could directly be embedded into the DataScienceCluster API so that each component configuration is strongly-typed, while retaining modularity and that the DataScienceCluster CRD that's generated from the API gets all the documentation from the components.
With the ConfigMap option, the pattern I see is that you have a default configuration that contains all the parameters, commented out (thanks YAML), and documented. So users can see what's available as configuration parameters, and they have the documentation readily available.

## References
- [component-base](https://github.com/kubernetes/component-base)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------- | ------ |
| [name]                        | insert date |  |
