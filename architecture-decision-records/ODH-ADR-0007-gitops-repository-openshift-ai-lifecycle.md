# Open Data Hub - GitOps Repository for OpenShift AI Lifecycle Management

|                |            |
| -------------- | ---------- |
| Date           | December 11, 2025 |
| Scope          | OpenShift AI Dependencies and Lifecycle Management |
| Status         | Approved |
| Authors        | [Luca Burgazzoli](@lburgazzoli) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-551](https://issues.redhat.com/browse/RHAISTRAT-551) |
| Other docs:    | none |

## What

Establish a dedicated GitOps repository to manage the complete lifecycle of Red Hat OpenShift AI and its dependencies using Kustomize and Helm.

## Why

One of the challenges in OpenShift AI deployment resides in handling external dependencies (e.g., Kuadrant, Kueue, LeaderWorkerSet) that require cluster-level permissions. 
Traditionally, OpenShift AI relied on a "Managed vs. Unmanaged" paradigm: while OpenShift Admins were expected to install these dependencies, the ODH Operator offered an option to configure them or leave them to the Admin. 
This ambiguity led to significant confusion and clear ownership problems.

We are moving away from this mixed model. 
Instead, we will provide a GitOps repository and Helm charts to facilitate the initial bootstrapping of the cluster. 
Fundamentally, this shifts the paradigm such that after this initial setup, full responsibility for managing dependencies and their configurations resides exclusively with the OpenShift Administrator.

## Goals

* Provide a "single source of truth" for the OpenShift AI stack and its dependencies
* Automate complex prerequisites and dependency management
* Enable modular, layered deployments (granular, grouped, or full installation)
* Support phased adoption starting with external dependencies
* Align versioning with OpenShift AI releases

## Non-Goals

* Replacing administrator ownership of production configurations
* Providing formal versioned releases of the GitOps repository

## How

**Technology**: The repository will be Kustomize and Helm based. It will utilize Kustomize for modular, layered deployments (granular, grouped, or full installation) and Helm charts as a complementary method for packaging and distribution.

**Methodology**: Dependency configurations will be treated as administrator-owned resources that Red Hat helps bootstrap during initial installation ("Day 1"). After this setup, administrators own all ongoing operations and maintenance ("Day 2").

**Phasing**:

* Phase 1: External dependencies (Jobset, Kueue, Kuadrant, etc.)
* Phase 2: Full OpenShift AI installation (DataScienceCluster, DSCInitialization, and component configurations)

**Versioning**: No formal releases; branches will align with OpenShift AI versions (e.g., `release-3.0`).

## Alternatives

### Leverage OLM Dependencies

**Discarded**: OLM v0 only supports static dependencies, meaning it is an all-or-nothing approach with no flexibility for conditional installation. Additionally, OLM v1 will not have dependency support at all, making this a non-viable long-term solution.

### Use the ODH Operator as Orchestrator for OLM Dependencies

**Discarded**: While this approach would simplify Day 1 installation, it complicates Day 2 operations since the ownership of the dependencies is not clear. Furthermore, some dependencies are not exclusively used by OpenShift AI, which would make the ownership problem even more significant.

## Risks

* **Administrators assume full responsibility for maintaining and customizing resources after the initial deployment.**  
  *Mitigation*: The repository serves as reference configuration that administrators can use as a starting point for their production configurations.

* **Configuration drift may occur if administrators diverge significantly from the provided templates.**  
  *Mitigation*: Administrators can consult the OpenShift AI upgrade guides to understand changes between releases and use this repository as the reference configuration sample to reconcile drift.

* **Keeping the GitOps repository synchronized with OpenShift AI releases requires ongoing maintenance.**  
  *Mitigation*: Branch alignment with release versions (e.g., `release-3.0`) ensures clear compatibility mapping.

## Stakeholder Impacts

| Group                         | Key Contacts                   | Date       | Impacted? |
| ----------------------------- | ------------------------------ | ---------- | --------- |
| Architects Team               | @opendatahub-io/architects     |            | y         |

## References

* [GitOps Repo for OpenShift AI Dependencies](https://github.com/opendatahub-io/openshift-ai-gitops)
* [Design Documentation](https://docs.google.com/document/d/1j5VHIDKSVW09QoeFebiBiROW9UL6K0UrOIYoJ-5HEzs/edit)
* [RHAISTRAT-551](https://issues.redhat.com/browse/RHAISTRAT-551)

