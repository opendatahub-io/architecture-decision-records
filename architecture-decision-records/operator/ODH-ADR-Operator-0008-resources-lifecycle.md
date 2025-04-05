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

This ADR defines how the opendatahub-operator manages the lifecycle of the Kubernetes resources it provisions using component-defined manifests and the opendatahub.io/managed annotation.

## Why

The operator provisions and reconciles resources defined by components. 
To support a variety of use cases—including user-customizable objects and create-only behavior—there needs to be a clear contract regarding how resources are managed over time, especially in the presence of manual modifications or evolving manifests.

## Goals

- Define consistent resource management semantics.
- Allow component developers to declare create-only resources.
- Allow end users to take ownership of specific resources post-deployment.
- Prevent unintended reconciliation or overwriting of user-managed resources.

## How

The operator evaluates the presence and value of the `opendatahub.io/managed` annotation on both the **rendered manifest** and the **live Kubernetes resource**:

| Annotation Location      | Value                        | Behavior                                                                 |
|--------------------------|------------------------------|--------------------------------------------------------------------------|
| **Manifest (Rendered)**  | `"false"`                    | Resource is created once, not reconciled afterward (create-only).        |
| **Manifest (Rendered)**  | _Missing_ or `"true"`        | Operator fully manages and reconciles the resource.                      |
| **Live Resource**        | `"false"`                    | Operator skips reconciliation and treats the object as user-owned.       |
| **Live Resource**        | _Missing_ or `"true"`        | Operator enforces manifest state and overwrites any manual modifications.|

### Additional Behavior

- Annotations defined in manifests are **not propagated** to the resulting Kubernetes objects to avoid misleading users into thinking they can control lifecycle via the live resource.
- **In all cases**, if the live resource is deleted from the cluster (either accidentally or manually), the operator will **recreate it** during the next reconciliation loop. This ensures declared state is always realized, regardless of whether the object is fully managed or marked as create-only.


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