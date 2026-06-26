# Open Data Hub - Kubernetes Controller for MCP Server Registry Synchronization

|                |            |
| -------------- | ---------- |
| Date           | June 26, 2026 |
| Scope          | OpenShift AI — MCP Server Registry |
| Status         | Draft |
| Authors        | [Dan Kuc](@dkuc) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [MCP Registry RFC (mlflow/rfcs#0004)](https://github.com/mlflow/rfcs/tree/main/rfcs/0004-mcp-registry) |

## What

Introduce a Kubernetes controller that watches `MCPServer` custom resources (managed by the [MCP Lifecycle Operator](https://github.com/kubernetes-sigs/mcp-lifecycle-operator)) and automatically synchronizes their state into the MLflow MCP Server Registry. The controller will live in the [Model Registry Operator](https://github.com/opendatahub-io/model-registry-operator) repository.

## Why

With MLflow adopted as the mcp asset registry and the MCP Registry defined upstream ([RFC 0004](https://github.com/mlflow/rfcs/tree/main/rfcs/0004-mcp-registry)), there is a gap between MCP servers deployed on the cluster and their representation in the governed registry.

Without a controller, synchronization would fall to the ODH Dashboard or to users manually registering deployed MCP servers. This has several drawbacks:

* **Dashboard as bookkeeper** — the Dashboard would need to act on all mcp server deployment and deployment deletion events. This is control-plane work that belongs in a controller, not a UI backend.
* **Data accuracy** — a controller reacting to Kubernetes events provides stronger guarantees that the registry reflects the actual state of deployed MCP servers. Dashboard-driven sync is best-effort and can drift when users bypass the UI.
* **Consistency with existing patterns** — the Kubeflow Model Registry already uses this approach: the InferenceService controller watches KServe InferenceService CRs and synchronizes deployment state into the Model Registry. Applying the same pattern to MCP servers keeps the platform's operational model consistent.

## Goals

* Automatically register deployed MCPServer CRs in the MLflow MCP Server Registry, including server identity, version, capabilities, and the cluster-internal endpoint.
* Keep registry state in sync as MCPServer CRs are created, updated, or deleted.
* Authenticate to MLflow using Kubernetes-native RBAC — no separate credentials or admin accounts.

## Non-Goals

* Replacing the Dashboard's role in MCP server discovery or user-initiated registration — the controller handles the automated sync path; manual registration through the Dashboard remains valid.

## How

### Reconciliation

The controller uses the standard controller-runtime pattern:

1. **Watch** `MCPServer` CRs (`mcp.x-k8s.io/v1alpha1`) cluster-wide.
2. On each reconcile, read the CR's status — particularly `status.serverInfo` (name, version, capabilities from the MCP protocol handshake) and `status.address.url` (the cluster-internal endpoint).
3. **Create or update** the corresponding MCP server and version in the MLflow MCP Registry via its REST API, and create an access binding for the cluster-internal endpoint.
4. On deletion, **update the registry** to reflect the server is no longer deployed, then remove the controller's finalizer.

### Authentication

MLflow in OpenShift AI uses the `kubernetes-auth` application plugin, which authorizes every API request via Kubernetes SelfSubjectAccessReview against pseudo-resources in the `mlflow.kubeflow.org` API group.

The controller authenticates using its pod's projected ServiceAccount token:

* Sends `Authorization: Bearer <sa-token>` on each request to MLflow.
* Sends `X-MLFLOW-WORKSPACE: <namespace>` to scope the operation to the MCPServer CR's namespace.
* MLflow's `kubernetes-auth` plugin performs a SelfSubjectAccessReview to verify the controller's ServiceAccount has the required verbs on MCP registry pseudo-resources (e.g., `mcpservers` under `mlflow.kubeflow.org`) in the target namespace.

A ClusterRole grants the controller's ServiceAccount the necessary permissions on MCP registry pseudo-resources across all namespaces, following the same pattern as the `mlflow-integration` role.

### Dashboard integration

When the ODH Dashboard deploys an MCP server by creating an `MCPServer` CR, it must add metadata labels that the controller uses to associate the deployment with a governed registry entry. For example, labels identifying the registered MCP server name, version, and target workspace allow the controller to link the CR to the correct registry record rather than creating a new one. Without these labels, the controller treats the CR as an unmanaged deployment and may create a duplicate registry entry or skip it entirely.

The exact label keys will be defined during implementation, following the convention established by the InferenceService controller (e.g., `modelregistry.kubeflow.org/registered-model-id`).

### Repository placement

The controller will live in the **Model Registry Operator** (`opendatahub-io/model-registry-operator`). This operator already manages catalog and registry synchronization controllers and is maintained by the AI Hub team who will own this work.

## Alternatives

### Controller in the MLflow Operator

The MLflow Operator could host this controller since it already manages the MLflow deployment. However, the MLflow Operator's scope is deploying and operating the MLflow server itself — it does not make calls to the MLflow REST API today and does not watch non-MLflow CRDs. Adding registry sync responsibilities would expand its scope and couple MCP lifecycle concerns to MLflow deployment lifecycle. The Model Registry Operator is a better fit because it already owns this class of work.


## Security and Privacy Considerations

* The controller's ServiceAccount token is the sole credential — no static secrets or passwords are stored.
* Authorization is enforced by Kubernetes RBAC through MLflow's SelfSubjectAccessReview mechanism, scoped per namespace.
* The controller needs cluster-wide read access to `MCPServer` CRs and cluster-wide write access to MCP registry pseudo-resources in `mlflow.kubeflow.org`. This is consistent with the RBAC footprint of the existing InferenceService controller.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| AI Hub / Model Registry       |                  |            | Yes — owns the controller implementation |
| MLflow                        |                  |            | Yes — must define MCP registry pseudo-resources for RBAC |
| MCP Lifecycle Operator         |                  |            | No — no changes required to the operator or CRD |
| ODH Dashboard                 |                  |            | Yes — can rely on the controller for automated sync instead of implementing it |

## References

* [ODH-ADR-ML-0001: Consolidate AI Asset Registries on MLflow](ODH-ADR-ML-0001-consolidate-ai-asset-registries-on-mlflow.md)
* [MCP Registry RFC (mlflow/rfcs#0004)](https://github.com/mlflow/rfcs/tree/main/rfcs/0004-mcp-registry)
* [MCP Lifecycle Operator](https://github.com/kubernetes-sigs/mcp-lifecycle-operator)
* [InferenceService Controller (existing pattern)](https://github.com/kubeflow/hub/tree/main/pkg/inferenceservice-controller)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |