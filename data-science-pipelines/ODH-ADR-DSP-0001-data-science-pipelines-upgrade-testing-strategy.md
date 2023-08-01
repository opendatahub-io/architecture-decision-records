# Upgrade Testing Process for Data Science Pipelines (DSP)

|                |                            |
| -------------- | -------------------------- |
| Date           | 2023-07-05                 |
| Scope          |                            |
| Status         | Accepted                   |
| Authors        | [Dharmit Dalvi](@DharmitD) |
| Supersedes     | N/A                        |
| Superseded by: | N/A                        |
| Tickets        |                            |
| Other docs:    | none                       |

## What

This document outlines the upgrade testing process for Data Science Pipelines (DSP). The process involves upgrading DSP from a latest released version to a version with the newest commit in main (or a tag version).

## Why

The upgrade testing process is crucial to ensure the successful upgrade of DSP versions and to identify any issues early on. By following a standardized testing approach, we can automate the process and provide continuous feedback on the stability and functionality of the upgraded DSP versions.

## Goals

* Regularly test the success of DSP version upgrades using the most recent commit or tag.
* Detect issues early and provide continuous feedback on the stability and functionality of the upgraded DSP versions.
* Incorporate upgrade testing into the regular release strategy to ensure smooth transitions between released and unreleased DSP versions.

## Non-Goals

* This ADR does not cover the implementation details such as tooling that ought to be used. Those aspects can be tailored according to specific requirements.

## How

The update testing strategy should account for all pre-requisites for deploying DSP, then perform an upgrade, and any follow up tests to confirm the upgrade was successful. The following outlines the design for such a process: 

1. Set up a Test Environment:
   - Provision a test Kubernetes cluster or an OpenShift cluster.
   - Ensure cluster admin privileges are available to perform the following steps.

2. Install Open Data Hub (ODH) Operator.

3. Deploy KfDef Core for the most recently released DSPO Version:
   - Use the KfDef manifest to deploy the core components of the most recently released DSP version.
   - Deploy a DSPA instance
   
4. Prepare for Upgrade Testing:
   - Determine the candidate version for the DSP upgrade. It could be a tag, branch or a commit.
   - Update the KfDef manifest for the candidate DSP version by configuring DSPO to point to the latest commit or a tag for that version.

5. Deploy KfDef Core for the candidate DSP Version:
   - Use the updated KfDef manifest to deploy the core components for the candidate DSPO version.
   - Deploy a DSPA instance
   
6. Run Upgrade Tests:
   - Execute tests specific to testing the success of the DSP version upgrade. Examples of test cases could include checking if resources such as DSPO and DSPA deployments, ServiceMonitors, etc., come up correctly.
   - Customize the tests according to specific requirements and use cases.

## Automation Considerations

   - Set up a workflow to automate the upgrade testing process.
   - Configure the workflow to trigger every night asynchronously using the most recent commit of the DSP upgrade branch.
   - Within the workflow, deploy the updated KfDef Core for the candidate DSP version and run the upgrade tests.
   - Collect the test results, including logs, error messages, and any relevant information.

## Open Questions

1. Impact on Running Pipelines:
   - Consider the impact of upgrading DSP on currently running pipelines.
   - Assess how the upgrade process might affect pipeline execution, ongoing workflows, and pipeline outcomes.
   - Plan for any necessary adjustments or mitigations to ensure the smooth functioning of ongoing pipelines during the upgrade process.
   - Efforts to address this question to be tracked in [this issue.](https://github.com/opendatahub-io/data-science-pipelines-operator/issues/217)

## Security and Privacy Considerations

No security and privacy considerations identified for the upgrade testing process.

## Risks

1. ODH Operator Redesign:
   - As per the input from the operator team, updating manifests will not be officially supported in the ODH Operator.
   - In development mode, users may have the option to update the manifest URI, which can enable testing of upgrades. However, this approach may not be officially supported and could introduce potential inconsistencies or issues.
   - There is a risk that relying on the manifest update capability in development mode may not align with the desired upgrade testing process or may not be reliable in a production environment.
   
2. The implementing team may face challenges in building and maintaining the automated upgrade testing process.

## Reviews

| Reviewed by                   | Date            | Notes |
| ----------------------------- | --------------- | ------|
| Achyut M.                     | July 19th, 2023 | --    |
| Greg S.                       | July 20th, 2023 | --    |
| Giulio F.                     | July 21st, 2023 | --    |
| Humair K.                     | July 18th, 2023 | --    |
