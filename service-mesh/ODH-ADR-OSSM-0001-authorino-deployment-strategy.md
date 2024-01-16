# Automate Authorino Operator Deployment in OpenDataHub Operator


|                |            |
| -------------- | ---------- |
| Date           | January 16, 2024 |
| Scope          | Platform, Service Mesh |
| Status         | Draft |
| Authors        | [Bartosz Majsak](@bartoszmajsak) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | https://issues.redhat.com/browse/RHOAIENG-514 |
| Other docs:    | none |

## What

In order to leverage [Authorino](https://github.com/Kuadrant/authorino) as [external authorization provider in Service Mesh](https://istio.io/latest/docs/tasks/security/authorization/authz-custom/) there is a need to install Authorino's [Operator](https://github.com/kuadrant/authorino-operator) which will be responsible for managing authorization service (Authorino instance). Instead of using OLM and OperatorHub we can embedded this process into OpenDataHub operator.

## Why

To simplify installation process of Authorino. Adding yet another manual installation step can be problematic for a few reasons:

  * there needs to be another pre-requisite check which can lead to OpenDataHub Operator reconcile failures 
    * user might not be aware of the need for this component and surprised by seeing an error when they simply try to configure DSCI/DSC
  * asking a user to install a component which is for interal use is uncessary and confusing
  * it allows us to handle productized images seamlessly

## Goals

* Simplify installation and configuration process of Service Mesh pre-requisites.

## Non-Goals

* Changing distribution, artifacts and installation process for the upstream Kuadrant/Authorino operator.

## How

  * Part of OpenDataHub deployment automation process
     * bundling tagged manifests in the OpenDataHub Operator image (e.g. [`https://github.com/Kuadrant/authorino-operator/tree/v0.10.0/config`](https://github.com/Kuadrant/authorino-operator/tree/v0.10.0/config))
     * applying them as part of DSCI setup

## Open Questions

* Do we consider a case when Authorino Operator is pre-installed by the user, so `managementState: Unmanaged`? How do we want to handle this?
* How should productized images be handled? Through `relatedImages` mechanism in CSV?

## Alternatives

1. Use OLM for Authorino
    1. This adds yet another installation step for the cluster admin, as they need to manage subscription through Operator Hub.
    2. Automatic updates of the operator installed through OLM may result in incompatible versions and issues we have not anticipated when testing.


## Stakeholder Impacts


| Group                 | Key Contacts   | Date           | Impacted? |
| --------------------- | -------------- | -------------- | --------- |
| ODH Operator          | Vaishnavi Hire | Jan 16, 2024   | Yes       |


## References

* Similar approach has been taken by the [CodeFlare Operator](https://github.com/opendatahub-io/architecture-decision-records/blob/main/distributed-workloads/ODH-ADR-DW-0001-determine-codeflare-deployment-strategy.md).

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
