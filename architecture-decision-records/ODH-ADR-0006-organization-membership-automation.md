# Codification of Open Data Hub GitHub organization membership

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | 2024-08-12 |
| Scope          | |
| Status         | Approved |
| Authors        | [Alex Corvin](@accorvin) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

We will use codify membership in the OpenDataHub-io GitHub organization using
[Peribolos](https://docs.prow.k8s.io/docs/components/cli-tools/peribolos/). We will
automate the process of applying this membership using GitHub actions.

## Why

This change is being made in the context of broader changes to ensure that the OpenDataHub code
base is secure and minimally vulnerable to malicious actors. As part of this effort, we plan to
reduce the set of organization owners to a very small set of individuals. We do not want this
change to result in a bottleneck for managing organization membership, and thus want to enable
individual teams to manage membership themselves.

## Goals

* Create an easy method to add and remove organizational members in a way that is self-service to teams
* Ensure that we can have a small set of organizational owners
* Implement a reliable process which won't have a high administrative burden

## Non-Goals

* We will not automate GitHub team membership or permissions on individual repositories.
  We want this automation to be as minimal as possible so that teams can continue using the
  GitHub native interface for as much as possible

## How

We will revive the [org-management](https://github.com/opendatahub-io/org-management) repository
which will contain a streamlined configuration file listing organization owners and members. We will
not use this file to list Organization teams, repositories, or teams' permissions on individual repositories.
Management of these items will continue to be manual.

We will create a new GitHub team in the opendatahub-io org called `Org Membership Maintainers` which
will include all managers and development leads. This team will be used in a CODEOWNERS file in the
org-management repo and have permission to approve and merge pull requests to the membership configuration file.

With this change in place, going forward we will use a pull request flow to add and remove
members to the organization. Individuals (either the individual requesting membership or somone acting
on their behalf) will open a pull request modifying the membership, which a member of the the
`Org Membership Maintainers` team will then approve.

## Open Questions

N/A

## Alternatives

* Continue with the current process of organization owners manually making membership changes - this
  is seen as not a viable alternative as our current set of org owners is seen as too large and therefore
  exposes us to security vulnerabilities
* Implement a custom organization role that enables only management or org members - this is hypothetically
  possible, but we have not tested this. Custom organization roles require GitHub enterprise which we
  do not currently have funding to adopt.

## Security and Privacy Considerations

* With this change we make the full set or organization owners and members public in the config file
* We will need to use a GitHub personal access tokken for us in the automation. This will need
  to be periodically renewed.

## Risks

* The automation will need to be maintained.
  * We expect the DevOps team to own this automation as part of their long term plans to automate
    membership in the Red-Hat-Data-Services org as required for Konflux.
* When we previously attempted to use Peribolos the automation was flaky and a constant point
  of frustration. To mitigate this:
    * We will use Peribolos only for managing organization membership, not teams or repository permissions.
      We feel that these latter items are more impactful to individual teams, and we'll leave manual control
      of these with the teams.
    * The previous implementation of Peribolos was not clearly communicated and therefore
      never fully adopted. This ADR is an attempt to communicate this more fully.

## Stakeholder Impacts

| Group                            | Key Contacts                            | Date        | Impacted? |
| -------------------------------- | ---------------------------------------- | ---------- | --------- |
| Architects Team                  | @opendatahub-io/architects               |            | y |
| Documentation Team               | @opendatahub-io/documentation            |            | y |
| Exploring Team                   | @opendatahub-io/exploring-team           |            | y |
| Model Serving Team               | @opendatahub-io/model-serving            |            | y |
| Training & Experimentation Team  | @opendatahub-io/training-experimentation |            | y |
| Platform Team                    | @opendatahub-io/platform                 |            | y |

## References

* [Peribolos](https://docs.prow.k8s.io/docs/components/cli-tools/peribolos/

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |

We will not use this table for reviews. Instead, approval on the pull request
adding this ADR will be used as reviews.