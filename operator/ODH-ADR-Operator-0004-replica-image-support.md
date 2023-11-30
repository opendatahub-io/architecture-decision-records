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

## Goals

* For each component(eg. codeflare, ray, kserve etc) provide a field `replicas` through which we can modify the the Deployment's replica count.
* For each component(eg. codeflare, ray, kserve etc) provide an image field through which we can pass-in a custom image to be used by the controller.
* Certain components uses other images(for eg. datascience-pipelines-controller uses `IMAGE_API_SERVER`, `IMAGES_ARTIFACT`). We need to provide support to customize these images as well.
* Control the resource limits and requests for individual components for more fine grained control.

## Non-Goals

* 
## How

The idea is to create a custom plugin for the kustomize where we use JSON patch to patch these updated values and then deploy the manifests.

## Open Questions

- What is the best way to implement this? Should we fetch the Deployment, read the env array, apply the changes then patch the changes.
- The requirement suggests to update the images in env, in this case JSON patch doesnot support selecting maps in an array using the key value. It allows us to select only using the array index(here env being the array). Will the env always have the same ordering of its values? If we choose the array index method, it is pretty fragile. Either some sort of strategic merge or implementing own 'plugin'?

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
