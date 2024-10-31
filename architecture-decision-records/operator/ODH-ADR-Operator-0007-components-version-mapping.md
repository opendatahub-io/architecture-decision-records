# Open Data Hub - Map component upstream versions to ODH releases

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2024-10-31                       |
| Scope          | Open Data Hub Operator           |
| Status         | Draft                         |
| Authors        | [Saravana Srinivasan](@sasriniv) |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        |                                  |
| Other docs:    | none                             |

## What

This document is intended to outline the design decisions made to map upstream versions of components that are supported by Data Science Cluster to ODH releases.

## Why

Customers are expecting to know the list of upstream components and their versions they are using while using the product. There were already several requests from the customers wanting to know the component versions during releases. 

## Goals

- To maintain a standard to capture the list of upstreams of the components along with their version and repository url that are supported by Data Science Cluster.
- Have it displayed in the ODH operator, precisely in Data Science Cluster's components status.

## How

- Expecting components team to create and maintain an "component_metadata.yaml" file in their repositories, in the same location as the manifests are being retrieved by the odh-operator, precisely in the root level. The yaml can contain release details of the upstreams in format specified below.
```
releases:
  - name:
    version:
    repositoryurl:
  - name:
    version:
    repositoryurl:
```
- Develop code logics on the ODH operator to read through the component_metadata.yaml file present in the same location as the manifests are being retrieved by the odh-operator, precisely in the root level.
- Update the component status section in Data Science Cluster with the information read from the yaml file

## Open Questions

- **Updating version details:** The component team is responsible for creating and maintaining the "component_metadata.yaml" file containing details of the upstreams and also, to promptly update the same when there are new additions to the upstreams.


## Alternatives

- Initially, discussion happened to maintain this version information in an .env file. But, with further discussions and understandings, certain components have got multiple upstreams with them. Maintaining and fetching multiple information from an env would be cumbersome, this pushed us to choose yaml file where we can group details of each of the upstream.

## Stakeholder Impacts

| Group                         | Key Contacts                                                | Date       | Impacted? |
| ------------------------- | --------------------------------------------------------------- | ---------- | --------- |
| ODH Platform Team         | @lburgazzoli @lphiri                                            |            | y         |
| Model Serving             | [Daniel Zonca](@danielezonca), [Edgar Hernández](@israel-hdez)  |            | y         |
| ODH Dashboard Team        | @andrewballantyne                                               |            | y         |
| IDE Team                  | @harshad16                                                      |            | y         |
| DS Pipelines Team         | @HumairAK @gmfrasca                                             |            | y         |
| Serving Team              | @Jooho                                                          |            | y         |
| TrustyAI Team             | @RobGeada                                                       |            | y         |
| Distributed Workloads     | @dimakis @anishasthana                                          |            | y         |

## References

RHOAISTRAT-327 Refinement Document

