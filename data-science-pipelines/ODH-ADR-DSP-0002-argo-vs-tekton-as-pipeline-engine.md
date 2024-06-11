# Argo Workflows vs. Tekton as pipeline engine for Data Science Pipelines (DSP)

|                |                                 |
| -------------- | ------------------------------- |
| Date           | 2023-10-06                      |
| Scope          |                                 |
| Status         | In Progress                     |
| Authors        | [Ricardo Martinelli](@rimolive) |
| Supersedes     | N/A                             |
| Superseded by: | N/A                             |
| Tickets        |                                 |
| Other docs:    | none                            |

## What

This document describes the Data Science Pipelines backend, which uses an external pipelines engine (currently Argo and Tekton are supported) to run pipelines. 

## Why

Kubeflow Pipelines depends on a pipeline engine to run its Data Science pipelines. In Kubeflow Pipelines v1, this implementation is dependent on the pipeline engine (with Argo Workflows as the reference implementation), thus leading the community to make a decision to fork the repo for other pipeline engine implementations like Tekton. With the v2 release, this backend is engine agnostic, as well there is a Driver implemented for that pipeline engine. Since the default backend is Argo, and the vast majority of Kubeflow Pipelines users use it as the backend, we must compare features and performance between Argo and Tekton to decide if moving with Tekton is still the right choice or we should adopt Argo as the pipeline backend.

Other important topics that needs evaluation are:
* With Argo being the default pipelines engine, the community will get their attention for bug fixed, testing, etc. on Argo as the pipelines engine.
* To have a feature-to-feature parity with Kubeflow Pipelines on Tekton, we need a skilled team about Tekton to develop these features. Currently, there is only 2 engineers from IBM to support that effort, plus some engineers at Red Hat focused only on Tekton.
* Time to Market: Kubeflow Pipelines v2 with Argo was released on July, but it was not released with Tekton support, because not all kfpv2 features were implemented on time.

## Goals

* Compare the 2 supported pipeline engines in Kubeflow Pipelines today: Argo and Tekton
* Describe the limitations between Argo and Tekton from the engineering pespective
* Provide a solution in which Argo and Tekton can coexist in case we decide for that option

## Non-goals

N/A

## How

Data Science Pipelines for phase 2 relies on Kubeflow Pipelines v2, which introduces the IR (Intermediate Representation) feature. With the new IR, Kubeflow Pipelines SDK translates the Python code to an agnostic representation that will be compiled in the backend to the desired pipeline engine during the pipeline run. In order to have this code compiled to a given pipeline engine in the backend, a Driver must be implemented to translate each run type to the corresponding implementation for the pipeline engine. A [diagram that explains the flow can explain in details](https://app.diagrams.net/#G1Vt42HC5ZvAggqF5WUuKQ1bm09BkcBPfM#%7B%22pageId%22%3A%22F-Q6uzJYFkfIFW8QHVqN%22%7D),

Because of the pipeline engine agnostic nature of KFP, there is no need to adopt Tekton as the engine and it is possible move to Argo, since it is the community choice. Argo, on the other hand, it is not backed by Red Hat and thus there is not a team of experts to help with specific implementation on top of it. The KFP community heavily relies on Argo as the pipeline engine because it was released with only that implementation officialy.

All the work to merge the kfp-tekton and the pipelines repo is currently [blocked by an issue that involves Go package dependencies](https://github.com/kubeflow/kfp-tekton/issues/1240), and there are other features yet to implement to make it feature-parity with the Argo pipeline engine. The speed to add new features in tekton is limited to the release of these features in Argo first, and the blocker is critical to move forward with other tasks. Currently, the ODH Data Science Pipelines engineering team has 8 members that can collaborate with the 2 IBM engineers to work on the implementation tasks, but the knowledge is still silo'd between the IBM engineers.

## Open Questions

With the upcoming Status-IR feature to be implemented in Kubeflow Pipelines, it will be possible to use [both Argo and Tekton with the same KFP deployment.](https://github.com/kubeflow/kfp-tekton/issues/1366). We need to figure out if with this new feature, when implemented, it is still required to choose between one or another pipeline engine to focus our development efforts or not.

## Risks

In case we adopt Argo as the backend, we have zero experience with the code and no influence in the community. For Tekton, we have a few core contributors, which brings us some references on who to ask if we need help, or if we need to report/prioritize bug fixes. 

## Reviews

| Reviewed by                   | Date            | Notes |
| ----------------------------- | --------------- | ------|