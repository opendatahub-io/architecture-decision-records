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

There is a growing need for the ODH operator to handle authentication for the platform. OIDC is not handled gracefully over the entire system, and there are use cases where individual components are handling auth individually which those components would like to move away from. Centralising auth concerns will simplify the architecture of auth features, security and ease of use of these features for users.

## Goals

* Define a new auth CRD.

## Non-Goals


## How

A new API is added to the operator in the form of an *DSCAuth* (Naming to be decided) CRD. This will initially handle a list of adminGroups which the operator will reconcile for access to dashboard UIs and applying requisite openshift permissions. This is intended to be the canonical place for auth configuration for future auth initiatives.

## Open Questions

* Naming of the new CRD.
    * DSCAuth
    * odhAuth
    

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