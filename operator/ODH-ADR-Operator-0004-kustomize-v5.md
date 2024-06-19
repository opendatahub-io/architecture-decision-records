# Open Data Hub - Upgrade manifests to kustomize v5

|                |            |
| -------------- | ---------- |
| Date           | 06-May-2024 |
| Scope          | Open Data Hub|
| Status         | Draft |
| Authors        | [Yauheni Kaliuta](@ykaliuta) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This document describes points of synchronization in order to upgrade
manifests to be compliant to current (v5) kustomize version

## Why

Currently provided manifests are outdated and when operator deploys
them it complaints about usage of deprecated features. Deprecated
features will be dropped in the future.

## Goals

* Define operator requirements to components regarding provided
  manifests.

## Non-Goals

* No exact recipe how to convert old manifests to the new version.

## How

Current way of synchronization is a file `params.env`. The file can
reside anywere in the manifests directory. Component's code in the
operator responsible to provide the location. The file had special
meaning in older kustomize but with kustomize v5 it is possible to use
it with the same result:

- component's kustomize configuration should use values from
  `params.env` to generate correct manifests (currently it is used to
  generate a ConfigMap and use the values from it);

- operator in runtime updates values in the file according to the
  configuration and generates the manifests;

To update the values component's code in the operator provides a map
of `name->value` pairs where `name` is a variable name in `params.env`
to change and `value` is a *name* of environment variable which
*value* is used for `name` then.

Exception: it can additionally set `namespace` variable if requested.

- Names of the environment variables is a point of synchronization as
  well.

The proposal is to keep the interface is it is. It allows to avoid
extra synchronization and gives possibility for components to update
their manifests independently in their own pace.

So, the requirements from operator will be:
- provide `params.env` in the directory of deployed kustomize
  manifests which contains variables operator can change if required;
- use up to date kustomize configuration.

## Open Questions

Is the way of synchronization (params.env) good enough?

## Alternatives

Create another way to allow opertator change custom parameters.

## Security and Privacy Considerations

N/A

## Risks

* In case of desynchronization between component manifests and
  operator, operator can miss some important configuration of a
  component, like fail to update images.

## Stakeholder Impacts

| Group                  | Key Contacts                              | Impacted? |
|------------------------|-------------------------------------------|-----------|
| AI Explainablitiy      |                                           | Yes       |
| Data Science Pipelines | [Giulio Frasca](@gmfrasca)                | Yes       |
| Distributed Workloads  | [Francisco Arceo](@franciscojavierarceo ) | Yes       |
| Model Serving          | [Daniel Zonca](@danielezonca)             | Yes       |
| Model Registry         | [Yuan Tang](@terrytangyuan )              | Yes       |
| ODH Dashboard          | [Andrew Ballantyne](@andrewballantyne)    | Yes       |
| ODH Operator           | [Landon LaSmith](@LaVLas)                 | Yes       |
| Workbench              | [Harshad Reddy Nalla](@harshad16)         | Yes       |

## References

* https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/vars
* https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patchesjson6902
* https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patchesstrategicmerge
* https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/commonlabels
* https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/bases

## Reviews

| Reviewed by | Date | Notes |
|-------------|------|-------|
| name        | date | ?     |
