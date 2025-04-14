# Open Data Hub - Resource Lifecycle Management in opendatahub-operator

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2025-04-05                       |
| Scope          | Open Data Hub Operator           |
| Status         | TBD                              |
| Authors        | [Luca Burgazzoli](@burgazzoli)   |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        |                                  |
| Other docs:    | none                             |

## What

This ADR defines how the opendatahub-operator manages the lifecycle of the Kubernetes resources it provisions using component-defined manifests and the `opendatahub.io/managed` annotation.

## Why

The operator provisions and reconciles resources defined by components. 
To support a variety of use cases—including user-customizable objects and create-only behavior. There needs to be a clear contract regarding how resources are managed over time, especially in the presence of manual modifications or evolving manifests.

## Goals

- Define consistent resource management semantics.
- Allow component developers to declare create-only resources.
- Allow end users to take ownership of specific resources post-deployment.
- Prevent unintended reconciliation or overwriting of user-managed resources.

## Terminology
To ensure clarity throughout this document, we define the following terms:

- Kustomized Manifest: The YAML configuration that has passed through the kustomization process but has not yet been applied to the cluster. This represents the intended state of resources as defined by component/services manifests and processed by kustomize.
- Kubernetes Object: The actual resource that exists in the Kubernetes cluster after manifests have been applied. These are the live entities that the operator manages and users can interact with.

## How

The operator evaluates the presence and value of the `opendatahub.io/managed` annotation on both the **Kustomized Manifest** and the **Kubernetes Object**:

| Annotation Location      | Value                        | Behavior                                                                 |
|--------------------------|------------------------------|--------------------------------------------------------------------------|
| **Kustomized Manifest**  | `"false"`                    | Resource is created once, not reconciled afterward (create-only).        |
| **Kustomized Manifest**  | _Missing_ or `"true"`        | Operator fully manages and reconciles the resource.                      |
| **Kubernetes Object**    | `"false"`                    | Operator skips reconciliation and treats the object as user-owned.       |
| **Kubernetes Object**    | _Missing_ or `"true"`        | Operator enforces manifest state and overwrites any manual modifications.|

### Additional Behavior

- The `opendatahub.io/managed` annotation defined in Kustomized Manifests is **not propagated** to the resulting Kubernetes Objects to avoid misleading users into thinking they can control lifecycle via the cluster object.
- For Kustomized Manifests with `opendatahub.io/managed: "false"`, the operator does not set an owner reference on the created Kubernetes Objects. This means that these objects will remain in the cluster rather than being garbage-collected when the component/service the objects are part of get removed/disabled.
- **In all cases**, if the Kubernetes Object is deleted from the cluster (either accidentally or manually), the operator will **recreate it** during the next reconciliation loop. This ensures declared state is always realized, regardless of whether the object is fully managed or marked as create-only.


## Open Questions

N/A

## Responsibility

The ODH Platform team is responsible for implementing and maintaining the behavior described in this ADR within the `opendatahub-operator`.

## Alternatives

N/A

## Stakeholder Impacts

| Group                     | Key Contacts                                                    | Date       | Impacted? |
| ------------------------- | --------------------------------------------------------------- | ---------- | --------- |
| ODH Platform Team         | @lphiri                                                         |            | y         |
| Model Serving             |                                                                 |            | y         |
| Model Serving Runtimes    |                                                                 |            | y         |
| Model Registry            |                                                                 |            | y         |
| ODH Dashboard Team        | @andrewballantyne                                               |            | y         |
| IDE Team                  |                                                                 |            | y         |
| DS Pipelines Team         |                                                                 |            | y         |
| Serving Team              |                                                                 |            | y         |
| TrustyAI Team             |                                                                 |            | y         |
| Distributed Workloads     |                                                                 |            | y         |

## References