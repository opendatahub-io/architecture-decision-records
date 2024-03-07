# Open Data Hub - Configure component resources through DSC

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

This document outlines design decision to configure `resources` for an ODH component in DataScienceCluster CRD. 

## Why

ODH component deployments lack a mechanism for customizing resource limits and requests. Deployment resources are hard-coded in the manifests with no means to adjust them according to user requirements.
We need a mechanism for users to configure resources when available resources are limited.

## Goals

* Enable the configuration of resource limits and requests for individual components via a dedicated `customization` struct within the DataScienceCluster CRD. An example yaml is given below:
  ```yaml
    spec:
  components:
    kserve:
      managementState: Managed
      customization:
        - name: kserve-controller-manager
           resources:
             requests:
              cpu: 100m
              memory: 128Mi
             limits:
              cpu: 200m
              memory: 256Mi
        - name: odh-model-controller
          resources:
             requests:
              cpu: 100m
  ```
* Introducing a generic `customization` struct will allow us with the flexibility to introduce other parameters easily.

## Non-Goals

- This ADR will not define customization parameters other than `resources`.

## How

* Implementation for this will be done in two phases
  * Introduce `customization` only to **Kserve** component. This is to address resource utilization [issues](https://github.com/kserve/kserve/issues/3467) in kserve.
  * Replicate the functionality in other components that require customization.
  * Kustomize [plugins](https://github.com/kubernetes-sigs/kustomize/tree/master/plugin/builtin) can be used to patch resources once deployed.
  

## Alternatives

- N/A


## Stakeholder Impacts

| Group                   | Key Contacts                                                   | Impacted? |
| ----------------------- |----------------------------------------------------------------|--------|
| Platform                | [Landon LaSmith](@LaVLas), [Edson Tirelli](@etirelli)                   | Yes    |
| Model Serving           | [Daniel Zonca](@danielezonca), [Edgar Hernández](@israel-hdez) | Yes    |

## References

N/A

