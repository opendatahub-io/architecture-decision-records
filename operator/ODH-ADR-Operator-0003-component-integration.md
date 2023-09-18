# Open Data Hub - ODH component Integration with DataScienceCluster

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2023-09-18                       |
| Scope          |                                  |
| Status         | Draft                            |
| Authors        | [Vaishnavi Hire](@VaishnaviHire) |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        |                                  |
| Other docs:    | none                             |

## What

This document outlines design decision to integrate ODH components with DataScienceCluster CRD. The document also defines the rationale behind fostering a close-knit integration between the individual components and the operator API.

## Why

The [KfDef](https://github.com/opendatahub-io/opendatahub-operator/blob/master/config/crd/bases/kfdef.apps.kubeflow.org_kfdefs.yaml) CRD, defined by the v1.x of ODH operator allowed any valid kustomize manifests to be deployed to the OpenShift cluster.
However, this design introduced following drawbacks:

- **Managing Resources:** Monitoring all resources for custom components became strenuous, coupled with the hindrance in utilizing owner references for a clean-up process.
- **Limited Customization:** The existing structure restricted the customization of components, offering limited accessibility to component-specific fields through the API.
- **Duplication of GitOps Workflow/ kustomize-build:** The KfDef CRD replicated the kustomize build functions, presenting no supplemental features post the deployment of the components.

## Goals

- **Increased Component Customization:** Allowing the [DataScienceCluster](https://github.com/opendatahub-io/opendatahub-operator/blob/main/config/crd/bases/datasciencecluster.opendatahub.io_datascienceclusters.yaml) API to expose every integrated component will accord users the flexibility to directly configure component-specific fields via the CRD, thereby expanding customization scope.
- **Improved Component Management:** As every component is tightly coupled with the operator, the controller is aware of the resources being deployed and only has the permissions to watch and manage those specific resources. This also allows operator to manage component lifecycle and upgrades.
- **Informed Approach:** The goal is to ensure that the operator has knowledge of the components being deployed, and make intelligent decisions based on that knowledge. 
## Non-Goals

- This ADR will not define transition of Tier 2 components into Tier 0/1.

## How

- To achieve this transition, any new component should be integrated with ODH Operator by following steps given [here](https://github.com/opendatahub-io/opendatahub-operator/blob/main/components/README.md). As a result update the DataScieceCluster API to expose new component fields.
- Ensure any components that are integrated follow the requirements for Tier 0/1 components.

## Open Questions

- **Quality Assurance for Components:** The component team is responsible for ensuring unit tests are added to any new component specific code and for update operator [e2e tests](https://github.com/opendatahub-io/opendatahub-operator/blob/main/tests/e2e/helper_test.go#L55) to include testing of the
                                    new component.


## Alternatives

- Any valid kustomize manifests that users want to deploy alongside ODH integrated components, can be deployed using kustomize build
  or GitOps workflow.

## References

N/A

