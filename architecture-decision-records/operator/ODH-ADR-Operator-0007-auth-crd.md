# Open Data Hub - Architecture Decision Record template

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | 22-10-2024 |
| Scope          | Open Data Hub Operator |
| Status         | In-review |
| Authors        | [Steven Tobin](@StevenTobin) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAOIENG-14807](https://issues.redhat.com/browse/RHOAIENG-14807)|
| Other docs:    | none |

## What

This document outlines the decision to implement an auth CRD in ODH.

## Why

There is a growing need for the ODH operator to provide centralized authentication and authorization services for the platform.  The near term needs are :

-  Add support for managing user groups. Currently this  is  managed by the Dashboard component and needs to be centralized.
-  Prepare for upcoming for changes to handle OCP Platform support for external OIDC authentication

Longer term,  this API and its controller will be enhanced to handle additional RBAC configuration currently handled in the Dashboard component, such as creating roles and role bindings.

Centralising auth concerns will simplify the architecture of auth features, security and ease of use of these features for users.


## Goals

* Define a new auth CRD.

## Non-Goals


## How

A new  API is added to the operator in the form of an auth CRD in the services api group. This will initially handle a list of adminGroups and allowedGroups (migrated from the groupsConfig field of the dashboardConfig) which the operator will reconcile for access to dashboard UIs and applying requisite openshift permissions. This is intended to be the canonical place for auth configuration for future auth initiatives.

The Auth CR is a singleton like the DSC and DSCi CRs.

An example of the CR:
```
apiVersion: services.opendatahub.io/v1alpha1
kind: Auth
metadata:
    name: odhAuth
spec:
    adminGroups: []
    allowedGroups: []
```

## Open Questions
* Migration Path:
    *   We will use [CEL](https://kubernetes.io/blog/2022/09/29/enforce-immutability-using-cel/#immutability-upon-object-creation) to make the groupsConfig field in the current OdhDashboardConfig CRD immutable. The operator will manage copying the content of the field over to the new CRD and the Auth CRD will be the new source of truth for adminGroups and allowedGroups. 

## Alternatives

* Continue to not have a central API for auth.

## Security and Privacy Considerations


## Risks


## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| ODH platform         | [Luca Burgazzoli](@lburgazzoli), [Chris Sams](@), [Lindani Phiri](@), [Steven Tobin](@StevenTobin) | 22-10-2024      | ? |


## References

* optional bulleted list

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
