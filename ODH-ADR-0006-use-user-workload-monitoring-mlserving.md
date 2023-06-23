# Open Data Hub - Architecture Decision Record template



|                |            |
| -------------- | ---------- |
| Date           | 2023-06-23 |
| Scope          | Model Serving|
| Status         | Draft |
| Authors        | [Vedant Mahabaleshwarkar](@VedantMahabaleshwarkar) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | TBD |
| Other docs:    | [Technical Design Document](https://docs.google.com/document/d/16R1dDOwZri7X_7cALuNK9lkLDF5cTJMKwHpSJRINAzA/edit) |

## What

This is a document intended to capture the decision to use [User Workload Monitoring](https://docs.openshift.com/container-platform/4.13/monitoring/enabling-monitoring-for-user-defined-projects.html) as the monitoring stack to generate metrics for mlserving
## Why

The decision to user User Workload Monitoring for Ml Serving metrics impacts multiple areas of work within RHODS. This ADR document acts as the high level proposal for the change as well as records the sign-off from all stakeholders. To summarize, the current monitoring stack for model serving fails to provide maintainable solutions to our monitoring requirements. Further details on the exact failures can be found in the technical design document for the solution. 

## Goals

* Use User Workload Monitoring as the monitoring stack for Model Serving within RHODS
* Minimize transition impact going from the current monitoring stack to UWM

## Non-Goals

* A Solution for long term metric retention is out of the scope for this ADR but will be covered in the technical design document for the solution. 
 
## How

The technical details surrounding this change require multiple considerations in terms of implementation, and it is not feasible to cover all the necessary details in an ADR document. The technical design document for this change captures the "How" behind this change. 

## Open Questions

1.  Enabling User Workload Monitoring requires `cluster-admin` permissions. This will the expectation that model serving metrics show up OOTB in RHODS when a model is deployed. 
We need documentation/UX alerts to let `rhods-admins` know that they need to enable UWM in collaboration with `cluster-admins` to enable the full functionality of model serving metrics once this change is implemented. What is the best way of ensuring a smooth UX during this transition? 


## Stakeholder Sign-Offs 

Approval on this ADR is counted as a sign-off

| Group                         | Key Contacts     | 
| ----------------------------- | ---------------- | 
| Model Serving            | Daniele Zonca | 
| ODH Dashboard | Andrew Ballantyne | 
| RHODS | Edson Tirelli | 
| QE | Jorge Garcia Oncins |  
