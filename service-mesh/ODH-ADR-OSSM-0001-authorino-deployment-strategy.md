# Automate Authorino Operator Deployment in OpenDataHub Operator

|                |            |
| -------------- | ---------- |
| Date           | January 16, 2024 |
| Scope          | Platform, Service Mesh |
| Status         | Draft |
| Authors        | [Bartosz Majsak](@bartoszmajsak) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHOAIENG-514](https://issues.redhat.com/browse/RHOAIENG-514) |
| Other docs     | None |

## What

To integrate [Authorino](https://github.com/Kuadrant/authorino) as an [external authorization provider in Service Mesh](https://istio.io/latest/docs/tasks/security/authorization/authz-custom/), it is necessary to install its [Operator](https://github.com/kuadrant/authorino-operator). This operator manages the Authorino instance, providing authorization services. Instead of deploying it via OLM and OperatorHub, embedding this process within the OpenDataHub operator is proposed.

## Why

The goal is to streamline the installation of Authorino for the following reasons:

  * Avoiding additional pre-requisite checks that could lead to OpenDataHub Operator reconcile failures.
  * Reducing user confusion by eliminating the need to manually install a component primarily for internal use.
  * Facilitating seamless management of productized images.

## Goals

* Improve the user experience by simplifying the installation and configuration of Service Mesh pre-requisites.

## Non-Goals

* Altering the distribution, artifacts, or installation processes for the upstream Kuadrant/Authorino operator.

## How

  * Embedding Authorino Controller deployment within the OpenDataHub deployment automation.
     * Incorporating specific tagged manifests from the Authorino Operator repository (e.g., [`v0.10.0/config`](https://github.com/Kuadrant/authorino-operator/tree/v0.10.0/config)) into the OpenDataHub Operator image. Bundling mechanism is already [in place](https://github.com/opendatahub-io/opendatahub-operator/blob/incubation/get_all_manifests.sh) and in use by other controllers/operators.
     * Automatically applying these manifests during the DSCI setup process.

## Open Questions

* How to manage cases where the Authorino Operator is pre-installed by the user?
  * Should `managementState: Unmanaged` be exposed for Authorino (sub)component?
  * As this is integral part of Service Mesh setup, does `managementState: Unmanaged` set on this level implies the same for Authorino deployment?
* What strategy should be employed for handling productized images? Is the `relatedImages` mechanism in CSV suitable?

## Alternatives

1. Deploying Authorino via OLM:
    1. This method imposes an additional step for cluster admins, who would need to manage subscriptions through the Operator Hub.
    2. There is a risk of automatic updates through OLM introducing incompatible versions or unforeseen issues.

## Stakeholder Impacts

| Group                 | Key Contacts   | Date           | Impacted? |
| --------------------- | -------------- | -------------- | --------- |
| ODH Operator          | Vaishnavi Hire | Jan 16, 2024   | Yes       |

## References

* A similar approach was adopted by the [CodeFlare Operator](https://github.com/opendatahub-io/architecture-decision-records/blob/main/distributed-workloads/ODH-ADR-DW-0001-determine-codeflare-deployment-strategy.md).

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| [Reviewer Name]               | [Date]     | [Notes] |
