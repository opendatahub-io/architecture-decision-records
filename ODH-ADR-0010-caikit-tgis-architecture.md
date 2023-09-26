# Open Data Hub - ODH, Caikit, and TGIS architecture


|                |                                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Date           | 2023-Sept-13                                                                                                                 |
| Scope          | OpenDataHub and Caikit/TGIS integration architecture                                                                         |
| Status         | Accepted                                                                                                                     |
| Authors        | [Sean Pryor](@Xaenalt),                                                                                                      |
| Supersedes     | N/A                                                                                                                          |
| Superseded by: | N/A                                                                                                                          |
| Tickets        |                                                                                                                              |
| Other docs:    | https://lucid.app/lucidchart/06fbfa85-ac66-40f7-9e60-1aa1d1ae426b/edit?invitationId=inv_74fb2b71-c771-405e-909c-e813b7d65623 |

## What

This ADR describes the architecture of the joint IBM-RedHat integration of ODH and Caikit/TGIS into the AI stack.

## Why

Caikit and TGIS are two parts of the IBM software stack used for training and serving Large Language Models (LLMs). This stack allows ODH to have a stack that specificially addresses LLM use cases.

## Goals

* Integration Caikit/TGIS as runtime backends for KServe and CodeFlare/Ray.
* Open sourcing and integration of the Caikit API for clients.

## Non-Goals

## How

Users will have a few ways to interact with the software stack. Caikit will be used both as a backend software runtime, which is used by the Caikit SDK that users can code against to create their models. These models can be trained in Ray using the Caikit runtime stack as the training backend on the nodes. Caikit will also be integrated as a serving runtime under KServe. All of these components can be interacted with using the standard OpenShift APIs, creating CRs in OpenShift, etc. Additionally, Caikit will also expose an API that can run on the cluster, allowing for several convenience features such as moving a model between training and serving, as well as some tracking. These features will be implemeted in the same manner, creating CRs and calling OpenShift APIs.

## Open Questions

## Alternatives

* One alternative discussed was to have the Caikit API be the only method to interact with resources on the cluster. However, the downside to this approach is that it would severely limit the utility of Caikit attempting to require the community to use this rather than the familiar APIs of KServe/Ray. In this case, presenting them together allows users to pick and choose how to interact with the software stack, and doesn't lock out any of the important features.

## Security and Privacy Considerations

The sidecar approach, having Caikit and TGIS colocated in a pod allows for a narrowing of the security surface. This allows the shared volume to not be a point of security concern.

## Risks

## Stakeholder Impacts


| Group                             | Key Contacts                  | Date | Impacted? |
| ----------------------------------- | ------------------------------- | ------ | ----------- |
| RedHat Model Serving Team         | Sean Pryor, JooHo Lee         |      | Yes       |
| RedHat Distributed Workloads Team | Anish Asthana, Antonin Stefanutti                 |      | Yes       |
| IBM Caikit Team                   | Gabe Goodhart, Gaurav Kumbhat |      | Yes       |
| IBM TGIS Team                     | Nick Hill                     |      | No        |

References
* https://lucid.app/lucidchart/06fbfa85-ac66-40f7-9e60-1aa1d1ae426b/edit?invitationId=inv_74fb2b71-c771-405e-909c-e813b7d65623

## Reviews


| Reviewed by | Date | Notes |
| ------------- | ------ | ------- |
| name        | date | ?     |
