# Open Data Hub - Make Trusted Bundle Configmap available 


|                |                                                             |
| -------------- |-------------------------------------------------------------|
| Date           | 2024-02-12                                                  |
| Scope          | Open Data Hub                                               |
| Status         | Draft                                                       |
| Authors        | [Landon LaSmith](@LaVLaS), [Vaishnavi Hire](@VaishnaviHire) |
| Supersedes     | N/A                                                         |
| Superseded by: | N/A                                                         |
| Tickets        |                                                             |
| Other docs:    | none                                                        |

## What

Add trusted-cabundle configmap to all non-openshift namespaces on ODH operator installation.

## Why

The first step to support self-signed certificates in ODH deployments is to make trusted-cabundle available in all ODH namespaces.

This allows ODH components to mount the certs as part of their deployment VolumeMounts.

## Goals

* Make trusted-cabundle configmap available in all non-openshift namespaces
* Users can opt-out of configmap injection by explicitly adding `security.opendatahub.io/inject-trusted-ca-bundle=false` annotation to a given namespace.

## Non-Goals

* Modification or management of OpenShift-specific or default namespaces concerning the trusted-cabundle configmap.
* Removal of injected trusted-cabundle configmap

## How

* We are introducing a controller that will be responsible for creating trusted-cabundle configmap in all new and existing non-openshift namespaces.
* For trusted-cabundle configmap, we are standardizing on `odh-trusted-ca-bundle` as the configmap name with a label of `app.kubernetes.io/part-of=opendatahub-operator`.
* A namespace is considered non-openshift if -
  * It doesn't start with `openshift-`
  * It doesn't start with `kube-`
  * It is not `openshift`
  * It is not `default`
* The configmap injection is triggered using an api field in DSCI `.spec.trustedCABundle.managementState`. When set to a `Managed` state this will inject the cert configmap in
all non-openshift namespaces. Users can opt-out of cert injection by setting the managementState to `Removed`.

## Alternatives

1. We are proposing a longer term solution that involves adding cert-cabundle configmap to only the namespaces that have
ODH resources. This approach is contingent upon the successful implementation of [DataScienceProjects controller ADR](https://github.com/opendatahub-io/architecture-decision-records/pull/25).


## Stakeholder Impacts

| Group                 | Key Contacts      | Date       | Impacted? |
|-----------------------|-------------------| ---------- | --------- |
| ODH Dashboard Team    | @andrewballantyne | date       | ? |
| IDE Team              | @harshad16        | date       | ? |
| DS Pipelines Team     | @HumairAK         | date       | ? |
| Serving Team          | @Jooho            | date       | ? |
| TrustyAI Team         | @RobGeada         | date       | ? |
| Docs Team             | Manuela Ansaldo   | date       | ? |
| Distributed Workloads | @anishasthana     | date       | ? |

## Reviews

| Reviewed by   | Date       | Notes |
|---------------|------------| ------|
| Edson Tirelli | 2024-02-12 | ? |
