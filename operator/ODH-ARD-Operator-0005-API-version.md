# Open Data Hub - opendatahub.io API version guidence

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2024-04-29                       |
| Scope          |                                  |
| Status         | Draft                            |
| Authors        | [Wen Zhou](@zdtsw)               |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        | [Tracking Jira](https://issues.redhat.com/browse/RHOAIENG-6524)|
| Other docs:    | none                             |

## What

This document outlines the requirements for each individual component before General Availability (set to 'Managed' in DSC CR by default) requires API stability (as on Stable level)

## Why

This guidance aims to mitigate risks to tier-0 components from non-backwards compatible API changes for Beta level.

## Goals

- Ensure that all components have their API versions on Stable level.
- Enforce compliance with this guidance for all components.
- Make adherence to this guidance a prerequisite for downstream offerings.
- Minimize the need to fork upstream projects to integrate API into opendatahub.io group.

## Non-Goals

- This ADR is specifically intendeded for ODH tier-0 components and focus on the API in opendatahub.io group.
- For components with CRDs from their upstream projects, we aim to maintain Beta level versions when the component reaches tier-0. However, we will need to support all available versions for a certain period as part of the deprecation window.

## How

To capture API we have in ODH with the current version 2.11.0
| Component             | Kind                  | Group                             | Version | CRD
| --------------------- | --------------------- | --------------------------------- | -- | ----------------------------------------------------- |
| Platform              | DSCInitialization     | dscinitialization.opendatahub.io  | v1 | dscinitializations.dscinitialization.opendatahub.io   |
| Platform              | DataScienceCluster    | datasciencecluster.opendatahub.io | v1 | datascienceclusters.datasciencecluster.opendatahub.io |
| Platform              | FeatureTracker        | features.opendatahub.io           | v1 | featuretrackers.dscinitialization.opendatahub.io      |
| Dashboard             | AcceleratorProfile    | dashboard.opendatahub.io          | v1 | acceleratorprofiles.dashboard.opendatahub.io          |
| Dashboard             | OdhApplication        | dashboard.opendatahub.io          | v1 | odhapplications.dashboard.opendatahub.io              |
| Dashboard             | OdhDocument           | dashboard.opendatahub.io          | v1 | odhdocuments.dashboard.opendatahub.io                 |
| Dashboard             | OdhDashboardConfig    | opendatahub.io                    | v1alpha | odhdashboardconfigs.opendatahub.io               |
| DataSciecnePipeline   | DataSciencePipelinesApplication | datasciencepipelinesapplications.opendatahub.io | v1alpha1 | datasciencepipelinesapplications.datasciencepipelinesapplications.opendatahub.io |
| Model Registry        | ModelRegistry         | modelregistry.opendatahub.io      | v1alpha1 | modelregistry.opendatahub.io                    |
| TrustyAI              | TrustyAIService       | trustyai.opendatahub.io           | v1alpha1 | trustyaiservices.trustyai.opendatahub.io        |

## Open Questions

N/A

## Alternatives

N/A

## References

- ODH tiered component: https://opendatahub.io/docs/tiered-components
- Kubernetes API versioning: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions

## Stakeholder Impacts

| Group                 | Key Contacts                 | Impacted? |
|-----------------------|------------------------------|---|
| Platform              | @VaishnaviHire/@LaVLas       | ? |
| ODH Dashboard Team    | @andrewballantyne/@lucferbux | Yes |
| DS Pipelines Team     | @anishasthana/@HumairAK      | ? |
| TrustyAI Team         | @RobGeada/@ruivieira         | ? |
| Model Registry Team   | @dhirajsb                    | ? |

## Reviews

| Reviewed by   | Date       | Notes |
|---------------|------------| ------|
| XXX           | 2024-MM-DD | ?     |
