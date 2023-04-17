# Open Data Hub Component Requirements

|                |            |
| -------------- | ---------- |
| Date           | 2023-04-11|
| Scope          | ODH Core component |
| Status         | Draft |
| Authors        | [Landon LaSmith](@LaVLaS) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This decision document is proposing requirements for any components to be included under the ODH Core(Tier0) and Tier 1 level

## Why

For any application or project to be considered as a component at the ODH Core (Tier 0) or Tier 1 level, we need to ensure that it has two or more opendatahub-io community members listed as maintainers that are committing to
* Full support for current and future releases of each component
* Timely upgrades to to keep the component up to date with the latest releases
* Full end to end testsuite to verify the latest functionality provided with each release
* Latest security updates for all dependencies to ensure that any software packages in use have been scanned for known vulnerabilties

We need to define some technical standards to ensure that each of the requirements above are met for every component

## Goals

Provide technical guidance for how to achieve all of the requirements of an ODH Core or Tier 1 component by specifying
* Designated component maintainers
* Standards for users to report bugs, request features and track updates
* Define how releases are made available to the ODH community and advertised for release
* Testing requirements for each component and how they should be made available
* Implementation for the latest security and vulnerability scanning of dependencies

## Non-Goals

* No definition for component integration and deployment as part of the ODH Core platform on kubernetes
* No definition or requirements for software language specific coding standards, package management or test framework
* No definition of hard requirements for defining a release strategy in git

## How

#### Component Maintainers

Maintainers of a component must be publicly visible and included as part of the git history of the code base.  The two supported file standards for project ownership are
* [OWNERS](https://www.kubernetes.dev/docs/guide/owners/) - Supported by the [Prow](https://docs.prow.k8s.io/docs/overview/) CI/CD system to automate the review and approval of Pull Requests
* [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) - Supported by GitHub CI/CD to use GitHub organization membership to designate maintainers of an individual repository

#### Issue Tracking

All issues (bugs and feature requests) must be accessible to the public and accessible for comment by any interested parties.  The full history of any issue must be preserved for the entirety of the component's life in opendatahub-io.  [GitHub Issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/about-issues) is the standard for all code repositories

In a future ADR, we will set a standard for the global labels that all github repositories in opendatahub-io will need to use for designating bugs, feature requests, community orgs and component name.

#### Component Release Requirements

All components must provide periodic stable release that will be made available as part of an ODH release.  The recommendation is that each component uses some form of [Semantic Versioning](https://semver.org/) for the release numbering scheme to ensure that each release version will align to a specific feature set.  This will allow any external components to verify integration functionality against a specific version to ensure compatibility.  Component maintainers are responsible for ensuring that major feature updates are versioned appropriately using git tags (or release branches) so that critical fixes can be applied without forcing users to accept major changes to just to resolve minor issues.

Each release should be publicized as a github [release](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository) with a highlevel summary of major changes in the release and a link to [changelog](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes) to show updates to the code base since the previous release

All component releases must provide up to date documentation that
1. Guides users through an end to end workflow of the component that will be published on [opendatahub.io](https://opendatahub.io)
1. Document the major configuration points that are of interest to users and developers to customize the application for their workflow

#### Testing Requirements

Each component is responsible for supporting an automated test suite that can verify that the application is working as intended with every update.  This testsuite should include unit tests to verify code functionality AND functional tests to verify that the running application or service is working correctly when deployed to a live environment.  The full scope of the testsuite should be updated with each release to ensure that the component is working when included as part of an Open Data Hub deployment

Automated CI/CD testing is required to verify that any updates to the code base are up to the component standards AND do not introduce changes that would break a working deployment. The CI/CD implementation will be solely managed by the individual component maintainers with the expectation that it is executed: 
* When a Pull Request is submitted for review
* Prior to a release being tagged

The two common options opendatahub-io for CI/CD automation are [GitHub Actions](https://github.com/features/actions) and [openshift-ci](https://github.com/openshift/release). The opendatahub-io github organization is onboarded to the [openshift-ci](https://github.com/openshift/release) system to provide CI/CD workflows that can run the application testsuites in a containerized environment.

#### Security and Vulnerability Scanning

All components must be updated regularly to ensure compatibility with the latest package updates and that any security issues are addressed in a timely manner. By default, [dependabot](https://docs.github.com/en/code-security/dependabot) is enabled on all repos under the [opendatahub-io](https://github.com/opendatahub-io) organization to ensure that Pull Requests are submitted automatically when known dendendency vulnerabilites are discovered in a repo codebase.  Any request to disable the dependabot service will only be considered *ONLY IF* the request is to use another service that can provide greater functionality and quality to scan for vulnerabilities.

## Open Questions

Optional section, hopefully removed before transitioning from Draft/Proposed to Accepted.

## Alternatives

We could allow each component repo to set their own standards for each the requirements but that has lead to a different set of quality standards between components

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Platform SIG                  | key contact name | date       | ? |
| MLOps SIG                     | key contact name | date       | ? |
| ODH Dashboard                 | key contact name | date       | ? |

## References

* [Tiered Components](http://opendatahub.io/docs/tiered-components.html) on opendatahub.io

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Vaishnavi Hire                | 2023-04-12 | ?     |
