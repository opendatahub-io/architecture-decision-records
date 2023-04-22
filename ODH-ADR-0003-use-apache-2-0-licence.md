# Open Data Hub - ODH-ADR-0003 - Open Data Hub default licence

|                |            |
| -------------- | ---------- |
| Date           | 11-April-2023
| Scope          | Open Data Hub default licence |
| Status         | Draft |
| Authors        | [Greg Sheremeta](@gregsheremeta) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | N/A |
| Other docs:    | none |

## What

This ADR captures our decision to license Open Data Hub under the Apache 2.0 license going forward.

## Why

Historically, Open Data Hub had standardized on using the GPLv3 license for all new code repositories. Over time, we lost track of the original reasons for selecting GPLv3. When the original decision was made, Open Data Hub was structured differently and was more focused on providing an open source reference architecture, so the reasons for the previous decision no longer apply.

The current engineers and project team members who build Open Data Hub are in the best place to select the best license for the project.

While GPLv3 and Apache 2.0 are both acceptable choices for Open Data Hub, we believe that selecting the Apache 2.0 license will better align Open Data Hub with other projects in the Machine Learning domain. We also believe that selecting the Apache 2.0 license will encourage more open source contributions to Open Data Hub.

We did an inventory of the licenses in use in the ODH-included projects and also some popular peer projects, and we found that 100% of those we looked at use a permissive licence, and the vast majority of those use Apache 2.0:

|                |            |
| -------------- | ---------- |
| Jupyter | permissive, 3-clause BSD |
| Kubeflow | permissive, Apache 2.0 |
| KFP-Tekton | permissive, Apache 2.0 |
| Tekton | permissive, Apache 2.0 |
| CodeFlare | permissive, Apache 2.0 |
| Ray | permissive, Apache 2.0 |
| Elyra | permissive, Apache 2.0 |
| Modelmesh | permissive, Apache 2.0 |
| Pytorch | permissive, Apache-like, mostly Apache 2.0 headers |
| Tensorflow | permissive, Apache 2.0 |
| Keras | permissive, Apache 2.0 |
| Spark | permissive, Apache 2.0 |
| AutoML | permissive, Apache 2.0 |
| Scikit-image | permissive, mostly 3-clause BSD |
| Scikit-learn | permissive, 3-clause BSD |
| Pandas | permissive, 3-clause BSD |
| MXNet | permissive, Apache 2.0 |

## Goals

* Recognize that the Machine Learning community uses permissive licenses (mostly Apache 2.0) as the de facto standard, and strive to align Open Data Hub to match that de facto standard.
* Capture the decision that Open Data Hub will be licensed under the Apache 2.0 license going forward.
* Relicense any existing Open Data Hub-specfic repositories that are currently GPLv3 (such as Data Science Pipelines Operator) to Apache 2.0.

## Non-Goals

* We are not changing the license of any repositories included in Open Data Hub that are direct copies or forks of some other repository outside of Open Data Hub. Those must retain their existing licenses.

## How

* Publish this ADR as a proposed ADR.
* Have a week of commentary period for the Open Data Hub Community to ask questions and provide feedback.
* Assuming there is no Community dissent, we will move this ADR to accepted, change the affected repositories, and announce the change as being completed.

## Open Questions

We're leaving this ADR as "proposed" for a period of time so that the Open Data Hub Community can comment.

## Alternatives

The primary alternative is to not do anything and leave the default license as GPLv3. However, this has caused some confusion recently because the Machine Learning community has mostly adopted Apache 2.0 as discussed above.

Another alternative is to use a permissive license other than Apache 2.0. However, also as stated above, our goal is to be consistent with the Machine Learning community, and the community has mostly adopted Apache 2.0.

## Security and Privacy Considerations

n/a

## Risks

n/a for technical risks.

If there have been any non-trivial contributions to Open Data Hub that were made with the author's understanding that they were contributing under GPLv3, we need to get their permission to change the license on their contributions. We're not currently aware of any such contributions where the auther would not approve the relicensing to Apache 2.0.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| group or team name            | key contact name | date       | ? |


## References

* n/a

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
