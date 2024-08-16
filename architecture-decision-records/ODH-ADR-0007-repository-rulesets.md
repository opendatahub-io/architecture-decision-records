# Usage of GitHub rulesets in all opendatahub-io repositories

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | 2024-08-14 |
| Scope          | |
| Status         | Approved |
| Authors        | [Alex Corvin](@accorvin) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

We will require that all repositories in the opendatahub-io GitHub organization
use a GitHub [ruleset][rulesets] requiring a pull request before merging
to the default branch.

## Why

This change is being made in the context of broader changes to adopt best practices on
GitHub organization management and to adhere to the latest security recommendations
for code repositories. We want to enact a minimally restrictive protection strategy
that secures the code base while not detrimentally impacting teams.

## Goals

* Prevent code changes from being made without "proper oversight"
* Delegate responsibility to define what "proper oversight" means to
  individual code teams as much as possible

## Non-Goals

* N/A

## How

Maintainers or administrators for every repository in opendatahub-io will
define a [ruleset][rulesets] for their repository by following the
instructions [here](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository).

When configuring the ruleset:
  1. Selection the option to create a `New branch ruleset`
  2. The ruleset name can be anything. We recommend `Require a pull request for the default branch`
  3. For the target branch, select the `Include default branch` option
  2. Under branch rules, enable the option for `Require a pull request before merging`

All other options are optional and left to the discretion of code owners.

The opendatahub-io organization owners will be reponsible for periodically
auditing to ensure that all repositories are adhering to this policy and
will reach out to owners of any repositores not in compliance.

## Open Questions

N/A

## Alternatives

* Do not have a blanket policy of code protection. We feel that this is not a
  viable option due to recommended security best practices and requirements.
* Enforce a stricter set of rules than just requiring pull requests for default
  branches. We feel that this is overly prescriptive and individual teams should
  implement whatever works best for them.
* Use branch protection rules instead of rulsets. Rulesets are preferred as they are
  newer and more feature rich. It is also easier to write automation to audit
  adherence to policy with rulesets.

## Security and Privacy Considerations

* This change improves our overall security posture in opendatahub-io
* A security purist would argue that we should have a more rigid protection
  policy in place, but doing so would likely remove flexibility for
  individual teams.

## Risks

* We will need to regularly audit the codebase to ensure that this policy
  is being adhered to.
* With a minimal policy of requiring a pull request to merge code, a code
  owner could open a pull request that they then self-merge and get unintended
  code into the default codebase. We'll mitigate this risk through regular audits
  of team membership. Furthermore, we'll accept this risk as necessary to retain
  team autonomy (see [Security and Privacy Considerations](#security-and-privacy-considerations)).

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

* N/A

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |

We will not use this table for reviews. Instead, approval on the pull request
adding this ADR will be used as reviews.

[rulesets]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets