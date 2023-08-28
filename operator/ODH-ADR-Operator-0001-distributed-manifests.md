# Open Data Hub - odh-manifests git repository transition

|                |            |
| -------------- | ---------- |
| Date           | 2023-08-28 |
| Scope          | |
| Status         | Draft |
| Authors        | [Wen Zhou](@zdtsw) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This document outlines a solution to transform the current setup of hosting manifests within a centralized `odh-manifests` git repository into separate repositories for each individual component.

## Why

The existing structure of having a singular `odh-manifests` git repository for hosting manifests across all components presents several drawbacks:

- Manifests Duplication: The presence of duplicated manifests can lead to confusion among community users.
- Synchronization Challenges: Ensuring timely updates from component repositories to `odh-manifests` becomes problematic.
- Scalability Concerns: Extending the scope of Open Data Hub to include new tier-0/1 components presents complexities, despite its current status as the solution for tier-0/1.

## Goals

- Reduced Human Error: Streamlining the release cycle to minimize human errors.
- Enhanced Product Quality: Increasing confidence in the overall product quality.
- Developer Support: Assisting developers in validating and troubleshooting changes within their respective domains.
- This approach aims to enhance the organization and efficiency of manifest management within Open Data Hub's ecosystem.

## Non-Goals

This ADR is specifically intendeded for tier-0/1 components, which are supported across all repositories within the opendatahub-io organization.

## How

To achieve this transition, a multi-step approach is proposed:

- Component Repositories: Each component's git repository will independently host its manifests. Progress towards this transition is tracked in the checklist below:

| Component               | Default Git Repo                                | Default Git branch/tag| Transition finished ? |
| ----------------------- | ----------------------------------------------- |-----------------      | -----------  |
| Platform                | opendatahub-io/opendatahub-operator             | main                  | Yes |
| Dashboard               | opendatahub-io/odh-dashboard                    | main                  | Yes |
| Data Science Pipelines  | opendatahub-io/data-science-pipelines-operator  | main                  | Yes |
| Kserve                  | opendatahub-io/kserve                           | release-v0.11         | No  |
| Modelmesh               | opendatahub-io/modelmesh-serving                | -                     | No  |
| Workbenches             | opendatahub-io/notebooks                        | main                  | Yes |
| Workbenches             | opendatahub-io/kubeflow                         | v1.7-branch           | Yes |
| Ray                     | opendatahub-io/distributed-workloads            | main                  | Yes |
| Codeflare               | opendatahub-io/distributed-workloads            | main                  | Yes |

- Operator Integration: Within the operator, a function will be introduced to fetch manifests from individual component repositories during image build processes.
- Archival of odh-manifests: During the image build process, the current `odh-manifests` git repository is downloaded into a tarball file, which is subsequently decompressed into the final operator image. This approach is currently in use. However, as part of this transition, the `odh-manifests` git repository will be archived into read-only mode to align with the evolving workflow.
- Development Mode API: A new API field in `DataScienceCluster` will be integrated into the operator for development mode, enabling the fetching of manifests from component repositories during runtime.

## Open Questions

- Quality Assurance for Manifests: the component team is responsible for ensuring the quality of its manifests. Any changes should undergo thorough verification before merging into the default branch, as it serves as a crucial integration point. ODH nightly build will be generated and subjected to essential verification steps, enabling a rapid feedback loop.
- Flexible Component Repository Proposal: the proposal is to use  the component repository within the "opendatahub-io" organization. However, if this doesn't align with most use cases, we are open to making it more configurable to accommodate different components.
- Verification Process Proposal: In the current workflow, changes made to the odh-manifests Git repository undergo comprehensive testing steps as a pre-merge CI process. However, a defined process for post-transition verification remains pending. This raises the question of whether verification should be conducted within the Operator CI system or if we should rely on the ODH nightly build solely for post-merge verification purposes. Further clarification and decision-making are needed in this regard.
- During the transition period, the operator continues to utilize the `odh-manifests` git repository to retrieve manifests for any components that are not yet ready to host manifests.

## Alternatives

N/A

## Security and Privacy Considerations

N/A

## Risks

There are a couple of potential risks associated with the proposed approach:

- Operator Logic Updates: The component teams will be required to update the logic within the operator to align with the newly updated manifests. This transition may demand additional effort and coordination to ensure a seamless integration.
- Non-Production Manifests in Runtime: As a consequence of the new setup, there's a possibility that users might unintentionally use non-production manifests to build image or during runtime. This could potentially lead to unexpected behavior or issues.
- Monitoring Maintenance: The platform team will take primary responsibility for maintaining any modifications made to downstream monitoring manifests.
- ODH Release Cycle: Each component must provide a valid release tag from its git repository for the release coordinator to update the configuration in the operator. Otherwise, it falls back to using the default branch or tag as specified in the table above.

Addressing these risks through careful planning and communication will be crucial to the success of the manifest repository transition.

## Stakeholder Impacts

| Group                   | Key Contacts                              | Impacted? |
| ----------------------- | ----------------------------------------- | --------- |
| Platform                | [Landon LaSmith](@LaVLas)                 |  Yes |
| Open Data Hub UI        | [Andrew Ballantyne](@andrewballantyne)    |  Yes |
| Data Science Pipelines  | [Giulio Frasca](@gmfrasca)                |  Yes |
| Model Serving           | [Daniel Zonca](@danielezonca)             |  Yes |
| Workbenches             | [Harshad Reddy Nalla](@harshad16)         |  Yes |
| Distributed Workloads   | [Anish Asthana](@anishasthana)            |  Yes |

## References

N/A

## Reviews

| Reviewed by                    | Date          | Notes |
| ------------------------------ | ------------  | ----- |
|[Humair Khan](@HumairAK)        | 2023-09-08    | ----- |
|[Anish Asthana](@anishasthana)  | 2023-09-10    | ----- |
|[Vaishnavi Hire](@VaishnaviHire)| 2023-09-11    | ----- |
|[Harshad Reddy Nalla](@harshad16)|2023-09-11    | ----- |
|[Joohoo Lee](@Jooho)            | 2023-09-20    | ----- |
