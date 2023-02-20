# Data Science Pipelines Multi-User Approach

|                |            |
| -------------- | ---------- |
| Date           | 2023-02-20 |
| Scope          | Data Science Pipelines Project, within Open Data Hub |
| Status         | Draft |
| Authors        | [Greg Sheremeta](@gregsheremeta) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This decision document is proposing an approach for integrating [Data Science Pipelines](https://github.com/opendatahub-io/data-science-pipelines) (upstream = [Kubeflow Pipelines](https://github.com/kubeflow/kfp-tekton)) in a multi-user environment.

## Why

Out of the box, [Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/) comes with [multi-user isolation](https://www.kubeflow.org/docs/components/pipelines/v1/overview/multi-user/) that utilizes the Kubeflow concept of [Profiles](https://www.kubeflow.org/docs/components/multi-tenancy/overview/). We’re not using Profiles in Open Data Hub. Additionally, Kubeflow Pipelines [technical implementation](https://www.kubeflow.org/docs/components/multi-tenancy/design/) for multi-user isolation requires the installation of Istio and Dex. Istio in particular is a heavy dependency that we’d rather not require either.

## Goals

* Time pressure – implement a working solution for multi-user isolation and have it production ready by the beginning of April 2023.
* Implement a solution that takes into account supportability, operability, and SLAs in a managed services environment. We prefer a solution that will page SREs less over one that could potentially page SREs more.
* Utilize well-known Kubernetes concepts (RBAC, namespaces) to implement multi-user isolation.
* Ensure that our solution doesn’t make it difficult to migrate to Kubeflow Pipelines V2 when it’s released.
* Ensure that our solution stays consistent enough with upstream such that we can rebase on upstream frequently and that we can eventually make meaningful contributions to upstream.

## Non-Goals

* No plans to make Data Science Pipelines concepts work in a cross-namespace fashion. If a user has access to namespaces Namespace1 and Namespace2, a Component in Namespace1 cannot work with a Component in Namespace2 to comprise a Pipeline.

## How

Instead of using Kubeflow Pipelines’ out of the box [Multi-user isolation](https://www.kubeflow.org/docs/components/pipelines/v1/overview/multi-user/), we propose to roll out multiple individual single-user Kubeflow Pipelines stack into multiple namespaces, one for each Data Science Project. We’ll create a new operator called [data-science-pipelines-operator](https://github.com/opendatahub-io/data-science-pipelines-operator) to roll out and maintain these multiple stacks. (The rest of the ADR will refer to this approach as the “multi-stack approach”.)

## Open Questions

The Data Science Pipelines stack requires access to object storage (default is Minio) and a relational database (default is MySQL). For both Open Data Hub running on-prem and as a managed service, it’s unclear whether we’re better off using one central multi-tenant database and object store, or whether each stack should have its own individual single-tenant database and object store. Our chosen design allows for either approach.

## Alternatives

1. Use the Kubeflow Pipelines [multi-user isolation](https://www.kubeflow.org/docs/components/pipelines/v1/overview/multi-user/) as it exists, out of the box, including Istio and Dex and Profiles.
    * This is less work upfront for us to implement, but Istio is a complex and heavy dependency that we don’t want to require. It’s possible the Open Data Hub will require Istio in the future, but we don’t want to force that inclusion now if we don’t have to. We also don’t have the concept of Profiles in Open Data Hub, and it would be difficult to shoehorn that concept in. We dismissed this option early on.
2. Use the Kubeflow Pipelines [multi-user isolation](https://www.kubeflow.org/docs/components/pipelines/v1/overview/multi-user/) as it exists, out of the box, but **remove** the requirement for Istio and Dex. Replace these authz components with an oauth proxy that sets the HTTP header that the Kubeflow Pipelines components use to authorize users.
    * We have a [proof of concept](https://github.com/HumairAK/odh-manifests/blob/rhods-auth/data-science-pipelines/base/auth/AUTH_NOTES.md) of this working, but after weighing the tradeoffs with the multi-stack approach, we decided that the multi-stack approach fits our goals and requirements better:
        * supportability, operability, and SLAs in a managed services environment – while there is more surface area to monitor and potentially more things that can break with multiple stacks running, the impact of one single-tenant stack breaking is much less than the impact of the only multi-user stack breaking. In other words, we believe that the multi-stack approach will make it easier for us to meet SLOs and SLAs in a managed environment.
        * Ensure that our solution stays consistent with upstream – we felt that utilizing the upstream multi-user solution but modifying it by removing Istio left us farther away from upstream than our multi-stack approach. In the multi-stack approach we’re deploying multiple single-tenant stacks and not modifying them – we’re wrapping them with an operator.
        * Time pressure – we were originally concerned that writing a custom operator to maintain the multi-stack approach would cause a risk to our deadline, but we were able to create a working proof of concept of our operator within a week. It is true, though, that our multi-stack approach requires us to maintain custom code (an operator written in go) whereas using Alternative 2 would have merely required us to maintain custom manifests.

## Security and Privacy Considerations

* Because the multi-stack approach relies on Kubernetes RBAC and namespaces, we inherit the security and privacy benefits around multi-user authorization inherent in Kubernetes itself.
* We still have an open question around the object storage and relational database requirements, and our decision there has security and privacy considerations. If we use a single shared database, we would specify separate databases within the provider/database instance for each stack, so users’ Data Science Project data would not be stored alongside each other in the same tables. This is in contrast to the single-stack solution, which would have one provider and one shared database within the provider, so users’ data would be stored within the same tables. If we use multiple databases (one per stack), then the data is a little more separated, but we don’t see this as advantageous over multiple databases within one provider/database instance.

## Risks

* The implementing team as a whole doesn’t have deep experience with building operators.

## Stakeholder Impacts


## References

## Reviews

## Accept / Reject

