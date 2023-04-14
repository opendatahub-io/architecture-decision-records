# GitHub Label Standard for opendatahub-io organization

|                |            |
| -------------- | ---------- |
| Date           | 2023-04-14 |
| Scope          | |
| Status         | Accepted |
| Authors        | [Landon LaSmith](@lavlas) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This document will set the the organization standards for the core set of GitHub [Issue Labels](https://docs.github.com/en/issues/using-labels-and-milestones-to-track-work/managing-labels) that should be supported by every repository in the opendatahub-io organziation.

## Why

[opendatahub-io](https://github.com/opendatahub-io) needs a unified Issue workflow that can support common [queries](https://docs.github.com/en/issues/tracking-your-work-with-issues/filtering-and-searching-issues-and-pull-requests) for all Issue and Pull Request metadata from any repository in the opendatahub-io organization. To support this unified workflow, the required label names must match across all repositories.

## Goals

* Standardize an issue workflow that will be used to show the Issue lifecycle, ownership and category
* Define the label name standard that covers the universal states and metadata that is relevant to the entire opendatahub-io organization
  * Type: Bug, Feature, Documentation, tracker 
  * Status: To Do, In Progess, In Review, Closed 
  * Priority 
  * ODH Component: ODH Operator, Notebook Controller, Notebooks, Manifests, Data Science Pipelines, Model Serving, AI Explainability, ...
  * SIG: Platform, ML Ops, Developer Experience
  * Extra: Good First Issue
* Use the `opendatahub-community` repository that will become the centralized Issue triaging location for ODH Component owners to triage new issues and/or transfer to the appropriate component repository

## Non-Goals

* This ADR will only define the core set of labels that will be supported across all repositories in opendatahub-io.
  The labels outlined in this document will only be a subset of the available labels in any given repo and will not contain any labels that are isolatedj to an individual repostory workflow
* This is not a mandate that every component or SIG must use the centralized `opendatahub-community` repository to manage the lifecyle of issues relevant to their workflow

## How

Across all repositories in the opendatahub-io organization, we will create the labels below with the expectation that the workflows they outline will be universal across all opendatahub-io repositories

| Label                         | Description     |
| ----------------------------- | ---------------- |
| `tracker`             | Non-completable ticket; used for tracking work - akin to a Jira Epic |
| `untriaged`           | Indicates the newly created issue has not been triaged yet |
| `kind/bug`            | Indicates an unexpected problem or unintended behavior |
| `kind/enhancement`    | New functionality request (existing augments or new additions) |
| `kind/documentation`  | Improvements or additions to documentation |
| `kind/security`       | Indicates that this is a security issue that should be addressed |
| `needinfo`            | Further information is requested to unblock any progress on the issue |
| `priority/normal`     | An issue with the product; fix when possible                |
| `priority/blocker`    | Critical issue that needs to be fixed asap; blocks up coming releases                |
| `priority/low`        | An issue with the product that doesn't impact the user much or not at all (ie tech debt)    |
| `priority/high`       | Important issue that needs to be resolved asap. Releases should not have too many of these. |
| `good-first-issue`    | Good for newcomers |
| `odh-component/*`     | Name of the odh-component that owns this issue.  This should be used as the indicator for which component should own this issue if located in the centralized `opendatahub-community` repository |
| `sig/*`               | Name of the Special Interest Group in charge of this subject matter|
| `wg/*`                | Name of the Working Group that should be assigned to this issue |

Additional labels maybe reserved depending on certain bots or apps running in the organization.

It is assumed that all new issues will have the `untriaged` label until it is reviewed.  Once an issue is triaged, a `kind/*` and `priority/*` label should be added and the issue will follow the issue worklow outlined in the component repo where it is located.

## Alternatives

Alternative is to allow each SIG, WG, Maintainer, ... to use their own label naming system which would complicate attempts when querying or filtering on labels that have the same purpose but different name structure: `kind/bug` vs `kind::bug` vs `bug`

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Platform SIG                  |                  | 2023-05-31 |   yes     |
| MLOps SIG                     |                  | 2023-05-31 |   yes     |
| Developer Experience SIG      |                  | 2023-05-31 |   yes     |

## References

* Kubernetes Contributors Documentation
  * [Issue Triage Guidelines](https://www.kubernetes.dev/docs/guide/issue-triage/)
  * [Help Wanted and Good First Issue Labels](https://www.kubernetes.dev/docs/guide/help-wanted/#good-first-issue)

## Reviews
GitHub Approvals will function as reviews
