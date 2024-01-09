# Open Data Hub - Extend DataScienceCluster api to support per-Component tuning

|                |            |
| -------------- | ---------- |
| Date           | 2023-Dec-1  |
| Scope          | OpenDataHub operator |
| Status         | Draft |
| Authors        | [Ajay Jaganathan](@AjayJagan), [Yauheni Kaliuta](@ykaliuta) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [Tracker issue](https://github.com/opendatahub-io/opendatahub-operator/issues/441)|
| Other docs:    | none |

## What

This document explains about upgrading the DSC CRD to support customizing the replica count, images and modifying the resource limits/requests of individual components.

## Why

Current operator design doesnot support changing the replica count and the images are hard-coded in the manifests. A lot of teams and users have started requesting for providing support of manipulating these fields.
NOTE: The image update is targeted for dev purposes and will not be available in production. 

## Goals

* For each component(eg. codeflare, ray, kserve etc) provide a field `replicas` through which we can modify the the Deployment's replica count. If a component is deploying multiple deplyoments, then something like this is preferred:
  ```yaml
    spec:
  components:
     workbenches:
          notebookController:
               replicaCount: <>
          kfNotebookController:
                replicaCount: <>
  ```
* For each component(eg. codeflare, ray, kserve etc) provide an image field in the *devFlags* through which we can pass-in a custom image to be used by the controller.
* Certain components uses other images(for eg. datascience-pipelines-controller uses `IMAGE_API_SERVER`, `IMAGES_ARTIFACT`). We need to provide support to customize these images as well in the *devFlags*
* Control the resource limits and requests for individual components for more fine grained control.

## Non-Goals

* Convert *devFlags* type to `runtime.RawExtension` to make devFlags flexible. [ref comment](https://github.com/opendatahub-io/architecture-decision-records/pull/23#issuecomment-1847519245)
## How

The idea is to create a custom plugin for the kustomize where we use JSON patch to patch these updated values and then deploy the manifests.

## Open Questions

- Should we limit the max replica count to 2 in case of controllers?
- As per current requests, only Dashboard team require HPAs. So as an initial step should we start using HPAs only for Dashboard. As for other teams, we can(maybe for now) add replica count in the devFlags and then move it to the *base* when required?

## Alternatives

N/A

## Security and Privacy Considerations

N/A

## Risks

- Providing an invalid image value can break the respective deployment.
- Since we control the resource limits and requests, there is a possibility for over/under utilization for the resources.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| group or team name            | key contact name | date       | ? |


## References

* optional bulleted list

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
