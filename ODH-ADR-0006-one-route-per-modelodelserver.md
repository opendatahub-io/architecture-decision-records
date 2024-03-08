# Open Data Hub - Architecture Decision Record template

<!-- copy and paste this template to start authoring your own ADR -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | 06/01/2023 |
| Scope          | Model Serving, ODH Dashboard|
| Status         | Draft |
| Authors        | [Vedant Mahabaleshwarkar](@VedantMahabaleshwarkar) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHODS-6216](https://issues.redhat.com/browse/RHODS-6216) [RHODS-5888](https://issues.redhat.com/browse/RHODS-5888) [ODH-Dashboard-1287](https://github.com/opendatahub-io/odh-dashboard/issues/1287)|
| Other docs:    | [proposal](https://docs.google.com/document/d/1PCL3wgkk_OSY9Km9vOt2kZ4t6lEtitj223dYea-Uuk4/edit#heading=h.w5e37xee1yqq) |

## What

This ADR captures the change from creating one route per model to creating one route per model server for accessing models. 
## Why

Modelmesh is designed by default to be able to refer all models within a model server using 1 route. Creating 1 route per model is not necessary with the way modelmesh is designed. Creating one route per model does not scale well as seen in scale tests ([jira](https://issues.redhat.com/browse/RHODS-6216)). The only reason we have one route per model right now is so that we can piggyback off of HAproxy metrics for that route to get some metrics for indivual models that modelmesh does not support yet. Once modelmesh has the [capabilty](https://github.com/kserve/modelmesh/pull/90) to output metrics for a particular model, we will not need individual routes anymore

## Goals

* Implement a solution that scales well so as not not cause Oauth proxy to crash due to a large number of routes ([jira](https://issues.redhat.com/browse/RHODS-6216))
* Implement a solution that is non disruptive to existing user workloads
* Document the change in behavior along with the appropriate release

## How

* Goal 1   
  * Creating 1 route per model server instead will greatly reduce the number of routes we end up creating. This should reduce the pressure on OauthProxy such that it does not crash. 
    * This will be confirmed by running the same scale test that revealed the issue to begin with
* Goal 2
  * Users with existing installs will already have models deployed that are bound to 1 model. If a user is using this route in a workload and we delete the route resource, that is a bad user experience. 
  * With that in mind, we will keep any existing routes untouched. These routes have OwnerDetection set to their InferenceService, so if the user deletes his model it will also delete the associated route. This behavior is consistent before and after this change. 
  * However, even for existing models -- the route that is displayed in ODH Dashboard will change to show the new "per model server" route with the path `/v2/models/<model-name>/infer` appended to it (indicating the sub-route to reach the model itself)

## Acceptance Criteria

| Changes                       | Link                                                                              | Status    |
| ----------------------------- | --------------------------------------------------------------------------------- | --------- |
| odh-model-controller          | (livebuild)[http://quay.io/vedantm/rhods-operator-live-catalog:1.28.0-oneroute]   | in-review |
| odh-dashboard                 |                                                                                   | ?         |
| scale test result             |                                                                                   | ?         |  

## Open Questions


## Reviews

| Group                         | Key Contacts      | Date       | Notes |
| ----------------------------- | ----------------- | ---------- | ----- |
| Open Data Hub UI Console team | Andrew Ballantyne | date       |       |
| Model Serving                 | Taneem Ibrahim    |            |       |
| Model Serving                 | Taneem Ibrahim    |            |       |


## References

* optional bulleted list

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Andrew Ballantyne             | date       | ?     |
| Taneem Ibrahim                | date       | ?     |
| Chris Chase                   | date       | ?     | 
| Daniele Zonca                 | date       | ?     | 
