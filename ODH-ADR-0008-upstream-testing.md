# Open Data Hub - ODH-ADR-0008 - Open Data Hub Upstream Testing Terminology and Strategy

|                |                                                                                      |
| -------------- | ------------------------------------------------------------------------------------ |
| Date           | 2023-07-11                                                                           |
| Scope          |                                                                                      |
| Status         | Draft                                                                                |
| Authors        | [Giulio Frasca](@gmfrasca)                                                           |
| Supersedes     | N/A                                                                                  |
| Superseded by: | N/A                                                                                  |
| Tickets        | [#181](https://github.com/opendatahub-io/data-science-pipelines-operator/issues/181) |
| Other docs:    | none                       |

## What

This document addresses the strategy, nomenclature, and guidelines of testing on upstream changes in ODH

## Why

There is not a defined set of instructions or specifications for HOW testing should be performed on upstream changes, which leads to disjointed and confusing expectations on acceptance criteria for developers, testers, and stakeholders alike.

## Goals

The goals of this ADR are to generate:

* A defined appendix of test types, to be used across ODH projects
* A defined procedure for the tests that will be run at a set of given checkpoints
* A plan to migrate testing to our upstream project (ODH) rather than downstream (RHODS)


## Non-Goals

This ADR will **NOT** address these non-goals:
* Coverage Requirements of testing
* Implementation details of tests (technologies used, etc)

## How

There are two parts to this ADR, and therefore this section is split in two sub-sections as well: **Nomenclature** and **Strategy**

### Nomenclature
Before we can address Testing Strategy, we must first define the types/terms used in testing.  Pulling from several external resources (see [References](#references) section below), here is what we propose as our testing nomenclature, as well at the cost (time+effort) for running the given test class:

| Type/Term            | Description                                                                                     | Cost      |
|----------------------|-------------------------------------------------------------------------------------------------|-----------|
| Unit Testing         | White-box tests that execute small blocks of code and check for edge cases, error handling, etc | Very Low  |
| Functional Testing   | White- or Black-box tests that validate entire features against business requirements           | Low       |
| Integration Testing  | Black-box tests that ensure components integrate well together                                  | Medium    |
| End-to-End Testing   | Full suite of tests that ensures *project* can be installed, all features tested, and torn down | High      |
| Perf/Stress Testing  | Full system tests to validate functionality and usability at high load.                         | Very High |


In addition to the above, here are a list of test classes that we are missing or haven't implemented yet:
| Type/Term                       | Description                                                                          | Cost    |
|---------------------------------|--------------------------------------------------------------------------------------|---------|
| Sanity/Smoke/Regression Testing | Quick, basic tests to give some confidence that a change won't break other features  | Medium  |


### Testing Strategy

We have identified 5 key checkpoints, during which some or all tests should be run.  We are proposing the following strategy for each test type:

| Type              | Owner  | Development | Code PR | Manifest PR | Nightly | Pre-release |
|-------------------|--------|-------------|---------|-------------|---------|-------------|
| Unit Tests        | Dev    |      x      |    x    |      o      |    o    |      o      |
| Functional Tests  | Dev    |      x      |    x    |      o      |    o    |      o      |
| Smoke Tests*      | Dev/QE |      x      |    x    |      x      |    x    |      x      |
| Integration Tests | Dev/QE |      x      |         |      x      |    x    |      x      |
| End-to-End Test   | QE     |             |         |             |    x    |      x      |
| Perf/Stress Tests | QE     |             |         |             |         |      x      |

*Key: `x`: Required, `o`: Run, but only generates non-blocking warnings

\* as noted above, this does not yet exist and would need to be generated


**What this would look like in practice:**
Development writes and maintains any white-box tests and validates any code written against it.  These will be maintained within their appropriate component repositories and should run quickly (maximum runtime <5 minutes) so as to not cause any delays in feature development.  This likely will involve the specific testing frameworks of the programming languages in use, and would be run as part of a GitHub CI action.  These can also be run as part of the integration/build cycle, but should not block releases and would only generate warnings to be passed back to the development team.

Development and QE will partner in developing a set of smoke tests.  These will also run quickly and reside within the component repository.  Development can and should run these while developing features to ensure component sanity, but QE should also be able to run them without a deep understanding of the underlying code.  This likely will leverage simple scripts or top-level commands with basic configuration files, and will not involve building or deploying the code.  Runtime: Less than 5 minutes

Development and QE will also collaborate to generate a standardized methodology for running **Integration Tests**.  These are slower-running in nature and therefore should not block Code-centric PRs, but still should run for changes made to deployment artifacts (`odh-manifests`, for example).  This would use the ocp-ci framework that already exists for most components and remain in the CI actions for `odh-manifests`.

QE will maintain and continue adding/extending their End-to-end testing suite, which deploys and tests live installations of the entire ODH project.  Due to the more resource-intensive and time consuming nature of this test class, development will not run these tests, but will provide feedback and assistance when requested by QE.  Ownership and maintenance will reside within the QE organization, however.

Performance testing also should be run regularly, but is very expensive and likely not feasible or cost-effective to run nightly.  Therefore, this type of testing should be integrated and  run on a per-release cadence.

After following this strategy, with the bulk of the testing being performed on-demand and/or in a nightly cadence against ODH, we can achieve our goal of migrating testing to upstream.


## Open Questions

* Remove the 'End-to-End' Terminology: This is a proposal to remove the term 'end-to-end' from testing nomenclature. Reasoning being that the term introduces confusion as it has different meanings to different personas: Developers see end-to-end meaning a test of a simple single workflow of their component, integrators see it as installation-test-teardown cycle for *ALL* components in ODH, and the term is also loaded/magic in various CI platforms as well.  Instead, rename this to 'Acceptance' Testing.

* Build Testing:  It could prove to be valuable to have another class of testing that involves Build sanity, or in other words, the capability to ensure builds are not broken by a given change.  This may already be happening with nightly builds in the pipeline, but should we attach an official status/procedure in this doc around it?

## Alternatives

Alternatively, we can continue with the status quo.  This means the bulk of the testing would reside in a silo for QE which proved to be difficult to maintain and automate.  Because the bulk of the work is all framed within the concept of a RHODS release, migrating testing to upstream will prove very difficult.  Additionally, there is not a set standard for development testing which has shown to allow for less stringent testing practices, and doesn't provide a wide coverage against bug introduction

Another alternative is to run most or all tests for every code and manifest change without context.  The tradeoff with this strategy is that development velocity will greatly suffer, as we have found the slower/more expensive tests create significant hesitance and roadblocks when opening pull requests, and hinder the ability to commit small changes quickly (rather than submitting large, monolithic changesets all at once, due to the expensive nature of CI in this state)

Given these two alternatives, we found the middle ground, as outlined above (run fast/cheap tests in development, and heavy/expensive tests on periodic cadences), to be the most efficient.

## Risks

Not running **EVERY** test class on every PR inherently introduces the risk of introducing more bugs than if code changes just required the fast-running tests for merge.  However, the time and effort cost of doing such is likely to cause significant delays and hesitation in development and review cycles, which is the reasoning for this tradeoff.

## Stakeholder Impacts

| Group                         | Key Contacts                  | Date       | Impacted? |
| ----------------------------- | ----------------------------- | ---------- | --------- |
| RHODS QE                      | [Jorge Garcia](@jgarciao)     |            | yes       |


## References

* [IBM Definitions](https://www.ibm.com/topics/software-testing)
* [Atlassian Definitions](https://www.atlassian.com/continuous-delivery/software-testing/types-of-software-testing)


## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| TBD                           |            |       |
