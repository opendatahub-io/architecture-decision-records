# Open Data Hub - Map component upstream versions to ODH releases

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2024-10-31                       |
| Scope          | Open Data Hub Operator           |
| Status         | Approved                         |
| Authors        | [Saravana Srinivasan](@sasriniv) |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        |                                  |
| Other docs:    | none                             |

## What

This document is intended to outline the design decisions made to map upstream versions of components that are supported by Data Science Cluster to ODH releases.

## Why

Users are expecting to know the list of upstream components and their versions that are being shipped with the product. There were already several requests from users wanting to know the component versions during releases. 

## Goals

- To maintain a standard to capture the list of upstreams of the components along with their version and repository url that are supported by Data Science Cluster.
- Have it displayed in the Data Science Cluster's components status.

## How

- Component teams are expected to create and maintain a "component_metadata.yaml" file in their repositories, in the root of the directory from where the manifests are retrieved by the ODH operator at build time. The yaml can contain release details of the upstreams in the format specified below.
```
releases:
  - name:
    version:
    repositoryurl:
  - name:
    version:
    repositoryurl:
```
- Develop code logic on the ODH operator to read through the component_metadata.yaml file.
- Update the component status section in Data Science Cluster with the information read from the yaml file

## Open Questions

N/A

## Responsibility

- **Updating version details:** The component teams are responsible for creating and maintaining the "component_metadata.yaml" file containing details of the upstreams and also, to promptly update the same when there are new additions to the upstreams.


## Alternatives

- Initially, it was proposed to maintain this version information in an .env file. But, with further discussions and understanding, certain components have multiple upstreams with them. Maintaining and fetching information about multiple upstream releases from an env file would be cumbersome, this pushed us to choose a yaml file where we can group details of each of the upstreams.

## Stakeholder Impacts

| Group                     | Key Contacts                                                    | Date       | Impacted? |
| ------------------------- | --------------------------------------------------------------- | ---------- | --------- |
| ODH Platform Team         | @lburgazzoli @lphiri                                            |            | y         |
| Model Serving             | [Daniel Zonca](@danielezonca), [Edgar Hernández](@israel-hdez)  |            | y         |
| Model Serving Runtimes    | [Sean Pryor](@Xaenalt), [Vaibhav Jain](@vaibhavjainwiz)         |            | y         |
| ODH Dashboard Team        | @andrewballantyne                                               |            | y         |
| IDE Team                  | @harshad16                                                      |            | y         |
| DS Pipelines Team         | @HumairAK @gmfrasca                                             |            | y         |
| Serving Team              | @Jooho                                                          |            | y         |
| TrustyAI Team             | @RobGeada  @ruivieira                                           |            | y         |
| Distributed Workloads     | @dimakis @amsharma3                                             |            | y         |

## References

RHOAISTRAT-327 [Refinement Document](https://docs.google.com/document/d/1nbQB-uA48x79Ci3xrpMHGfdo3XjTR4xirtWk0we0kl8)
