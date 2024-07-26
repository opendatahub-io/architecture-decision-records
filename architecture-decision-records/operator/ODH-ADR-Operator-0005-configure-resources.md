# Open Data Hub - Whitelist some component fields for user customizations

|                |                                                                          |
|----------------|--------------------------------------------------------------------------|
| Date           | 2024-03-07                                                               |
| Scope          |                                                                          |
| Status         | Draft                                                                    |
| Authors        | [Vaishnavi Hire](@VaishnaviHire)                                         |
| Supersedes     | https://github.com/opendatahub-io/architecture-decision-records/pull/23  |                         
| Superseded by: | N/A                                                                      |
| Tickets        |                                                                          |
| Other docs:    | none                                                                     |

## What

This document outlines design decision to introduce ability for user customizations of fields like resources and replicas.

## Why

ODH component deployments lack a mechanism for customizing resource limits, requests and deployment replicas. Deployment fields are hard-coded in the manifests with no means to adjust them according to user requirements.
We need a mechanism for users to configure resources when available resources are limited.

## Goals

* Enable the configuration of resource limits, requests and replicas for individual components. 
* Introduce an internal kustomize plugin that will whitelist fields like `replicas` and `resources`. This means any changes to these
fields will not be overwritten by the operator.
* We will not update or expose any fields

## Non-Goals

- This ADR will not define customization parameters other than `resources` and `replicas`.

## How

* Implementation for this will be done in two phases
  * Introduce customizations only in **Kserve** component. This is to address resource utilization [issues](https://github.com/kserve/kserve/issues/3467) in kserve.
  * Replicate the functionality in other components that require customization.
  * Kustomize [plugins](https://github.com/kubernetes-sigs/kustomize/tree/master/plugin/builtin) can be used to patch resources once deployed.
  

## Alternatives

- N/A


## Stakeholder Impacts

| Group                   | Key Contacts                                                   | Impacted? |
| ----------------------- |----------------------------------------------------------------|-----------|
| Platform                | [Landon LaSmith](@LaVLas), [Edson Tirelli](@etirelli)          | Yes       |
| Model Serving           | [Daniel Zonca](@danielezonca), [Edgar Hernández](@israel-hdez) | Yes       |
| ODH Dashboard Team    | @andrewballantyne                                              | ?         |
| IDE Team              | @harshad16                                                     | ?         |
| DS Pipelines Team     | @HumairAK                                                      | ?         |
| Serving Team          | @Jooho                                                         |  ?        |
| TrustyAI Team         | @RobGeada                                                      | ?         |
| Distributed Workloads | @dimakis                                                       | ?         |

## References

N/A

