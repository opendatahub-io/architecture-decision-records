# Open Data Hub - Determine CodeFlare Deployment Strategy

|                |                                                                                      |
| -------------- | ------------------------------------------------------------------------------------ |
| Date           | September 22, 2023                                                                   |
| Scope          | Distributed Workloads                                                                |
| Status         | Review                                                                               |
| Authors        | [Anish Asthana](@anishasthana)                                                       |
| Supersedes     | N/A                                                                                  |
| Superseded by: | N/A                                                                                  |
| Tickets        | [Tracking Issue](https://github.com/project-codeflare/codeflare-operator/issues/314) |
| Other docs:    | none                                                                                 |

## What

We will no longer use an OLM installed CodeFlare operator to provide distributed workloads capabilities in ODH.

## Why

The Operator of Operators pattern followed by the ODH has ODH responsible for deploying CRDs and controllers for components under its umbrella. CodeFlare has taken a different approach, where we have:

1. A published CodeFlare Operator (CFO) is available in community operator hub
    1. Users need to manually subscribe to this operator
2. ODH creates configurations for CFO as part of the Data Science Cluster custom resource
    1. If the CFO does not exist on the cluster, ODH Operator will expose a status requiring users to manually subscribe to the CFO.

The above flow results in confusion for users – we have seen multiple instances of folks not subscribing to the CFO and coming to community channels with “issues”. Additionally, this results in CodeFlare diverging from other ODH components.

## Goals

* Simplify CodeFlare usage experience for users

## Non-Goals

* Changing installation path for upstream CodeFlare project

## How

1. We will create a fork of the CodeFlare Operator repository in the ODH organization
   1. This fork will be kept in sync with upstream CodeFlare. There are no plans for code to diverge
   2. This repository will now serve as the home for CodeFlare Operator code, manifests, and CRDs in ODH.
      1. Having a fork allows us to have an ODH-controlled repository for new image builds and manifest references, allowing us to better version the CodeFlare stack in ODH.
2. CodeFlare CRDs will initially be included alongside the CFO manifests
   1. They will eventually be moved into the ODH Operator bundle.
   2. An implication of this is that ODH and community olm CFO can not be installed on the same cluster.
3. This fork will be synced with upstream every time there is an upstream release
4. As we also have a fork of KubeRay in the ODH organization, we can follow a similar process for KubeRay. Given this, we can probably delete all manifests from the distributed workloads repository.

This would simplify the user experience greatly as users simply need to enable CodeFlare in the DSC, with ODH taking care of the rest. The above flow also brings CodeFlare closer in line with other ODH components. From a testing perspective, all of our existing test cases will still be useful. The only ones that won’t carry over without updates are the existing olm upgrade tests in the upstream CFO repository.

Another benefit to the above approach is better controls around the versioning of CodeFlare. In the event of a CodeFlare community operator release, existing ODH users are not at risk of being auto-updated before the ODH changes are ready.

## Questions

1. Should we delete the distributed workloads repository altogether?
    1. This requires us to have a landing place for distributed workloads documentation.
    2. [GitHub Issue](https://github.com/red-hat-data-services/distributed-workloads/issues/25)

## Alternatives

1. Continue to use OLM for CodeFlare
    1. This is not a great user experience as users need to manually subscribe to dependent operators. The ODH operator is currently not planning to include manage of subscription to dependent operators
    2. This will result in us continuing to differ from other ODH components, which could result in other unforeseen issues popping up in the future.

## Stakeholder Impacts

| Group                 | Key Contacts   | Date          | Impacted? |
| --------------------- | -------------- | ------------- | --------- |
| Distributed Workloads | Anish Asthana  | Sept 22, 2023 | Yes       |
| ODH Operator          | Vaishnavi Hire | Oct 4, 2023   | Yes       |

## Reviews

| Reviewed by        | Date          | Approval | Notes                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------ | ------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Antonin Stefanutti | Sept 22, 2023 | Approved |                                                                                                                                                                                                                                                                                                                                                                                   |
| Dimitri Saridakis  | Sept 29, 2023 | Approved |                                                                                                                                                                                                                                                                                                                                                                                   |
| Karel Suta         | Sept 26, 2023 | Approved |                                                                                                                                                                                                                                                                                                                                                                                   |
| Jessica Forrester  | Oct 02, 2023  | Approved | If the team feels comfortable with the overhead of carrying the community operator and the slightly different install path downstream, then this is the right customer experience from my perspective. Whether the CRDs are included in the operator bundle needs to be settled, but I’d recommended sticking with whatever pattern the operator has already established for now. |
| Daniele Zonca      | Oct 02, 2023  | Approved | I think we need to revisit how we integrate components into ODH to limit the need of having a fork “just” to simplify user experience. But this requires work that is not in the scope of this ADR so I’m fine to proceed with this proposal assuming DW team is fine maintaining this “fork”                                                                                     |
| Greg Sheremata     | Sept 27, 2023 | Neutral  | +1 to not making users click extra buttons in operatorhub. I’m agnostic to the implementation details described here, and thus not explicitly marking my review as an “approval”.                                                                                                                                                                                                 |
| Edson Tirelli      | Sept 26, 2023 | Approved | Ideally this change is also coordinated with changes in the ODH operator, to support cases where the codeflare operator is pre-installed by the user (“managementState: unmanaged”). Also, please ensure the “fork” is only a snapshot for supporting/image building purposes and not an actual fork with diverging code.                                                         |
| Vaishnavi Hire     | Oct 4, 2023   | Approved | No additional notes                                                                                                                                                                                                                                                                                                                                                               |
