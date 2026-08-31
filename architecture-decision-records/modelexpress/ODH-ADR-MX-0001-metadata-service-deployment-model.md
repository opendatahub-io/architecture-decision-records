# ODH-ADR-MX-0001: ModelExpress Metadata Service Deployment Model

|                |            |
| -------------- | ---------- |
| Date           | 2026-08-31 |
| Scope          | ModelExpress |
| Status         | Draft |
| Authors        | [Will Eaton](@wseaton) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | [ModelExpress upstream](https://github.com/ai-dynamo/modelexpress), [opendatahub-io/modelexpress](https://github.com/opendatahub-io/modelexpress) |

## What

How the ModelExpress metadata service is deployed on a cluster: how many instances run, where, and in which namespace its metadata CRs (`ModelMetadata`, `ModelCacheEntry`) live. The decision: a shared cluster singleton in a system namespace by default, with namespace-scoped instances as an opt-in isolation flow.

## Background

ModelExpress splits into a control plane and a data plane. The control plane is the metadata service this ADR covers: a gRPC service that tracks which models exist and where, so a new replica can find a peer that already holds the weights. The data plane moves the bytes, using [NIXL](https://github.com/ai-dynamo/nixl) as the transport library for GPU-to-GPU P2P transfer over RDMA-capable fabrics: InfiniBand, RoCE, EFA, or multi-node NVLink (MNNVL).

The hardware requirement applies only to the data plane. The metadata service runs on any cluster, and on nodes without an RDMA fabric ModelExpress falls back through its loading chain (ModelStreamer, GPUDirect Storage, the engine's native loader), still using the shared metadata and download coordination. The deployment model below is therefore independent of node hardware; the P2P fast path lights up where the fabric exists.

## Why

ModelExpress deduplicates model downloads and coordinates peer-to-peer weight transfer between inference workloads. Its metadata service tracks which models exist on the cluster and where, using namespaced CRs as its state store. The benefit grows with the number of workloads sharing one metadata view: two namespaces serving the same model only deduplicate if they share a metadata service. That argues for one shared instance. Some tenants, however, must not expose even model names across namespace boundaries, so an isolated topology has to exist too.

ModelExpress is enabled through the `DataScienceCluster` CR, and this ADR proposes it as a root-level component: a peer of Model Serving, not a sub-component of it, with no runtime dependency on KServe or any other component operator.

## Goals

* Default topology that maximizes cross-namespace deduplication with minimal setup.
* Opt-in namespace-scoped topology for tenants that need metadata isolation.
* Namespace-scoped CRDs throughout, so standard RBAC is the isolation mechanism.
* Root-level `DataScienceCluster` component with no dependency on other component operators.

## Non-Goals

* The metadata CR schemas themselves; those are owned upstream.
* Hard singleton enforcement; the default is a convention, not a CEL constraint.
* The `DataScienceCluster` component API surface for ModelExpress (fields, defaults); that follows the standard component onboarding process.

## How

The metadata service is deployed via a namespaced `ModelExpressServer` CR reconciled by the ModelExpress operator, using upstream's Kubernetes CRD metadata backend. All ModelExpress CRDs are namespace-scoped.

**Default: cluster singleton.** One `ModelExpressServer` in a shared system namespace (e.g. `modelexpress`). The service writes all metadata CRs into that namespace, so its RBAC never leaves it. Workloads cluster-wide use the service DNS endpoint published on `status.endpoint`. Admins own the instance; tenants need only network reachability and an auth allowlist entry.

**Opt-in: namespace-scoped instance.** A tenant creates a `ModelExpressServer` in its own namespace; metadata CRs stay there and namespace RBAC governs visibility. Workloads in that namespace point at the local endpoint by configuration; there is no automatic discovery or fallback between instances. The tenant forgoes cross-namespace deduplication by design. Both topologies compose on one cluster; instances share no state.

**ServiceAccount token auth.** The service authenticates callers by Kubernetes ServiceAccount (`spec.security` on the CR): clients present a projected token, the service validates it via TokenReview against configured `tokenAudiences`, and checks the caller against an `allowedServiceAccounts` list of `namespace:serviceAccount` pairs. This is the access boundary that makes the shared endpoint safe to expose cluster-wide, and on isolated instances it pins access to the tenant's own ServiceAccounts. Shared instances should run `mode: enforce`; the default is disabled.

**Install.** The ODH operator deploys the ModelExpress operator when the component is enabled in the `DataScienceCluster`. The operator watches `ModelExpressServer` CRs in all namespaces and requires no other component to be enabled.

## Alternatives

* **Cluster-scoped metadata CRDs with one mandatory instance.** Trivial discovery, but per-tenant visibility becomes impossible without admission-level filtering, and the service needs cluster-scoped write access. The isolation flow stops being implementable.
* **Namespace-scoped instances only.** Uniform and isolated, but N namespaces serving the same model means N downloads and N cache copies; the deduplication win disappears exactly where it matters (large shared foundation models).
* **Redis metadata backend.** Upstream supports Redis instead of CRs. It adds a stateful service to run and secure, and loses `kubectl` inspectability, watch semantics, and RBAC scoping; the CRD backend gives us the namespace model for free.
* **Sub-component of Model Serving.** Couples enablement to KServe with no technical dependency; non-KServe consumers (llm-d, raw deployments) would have to enable a serving stack they don't use.

## Security and Privacy Considerations

Metadata is low sensitivity (model names, source types, cache locations), but model names alone can reveal what a tenant is working on; the namespace-scoped topology covers that case. In the shared topology, tenants cannot read the system namespace's CRs and reach the service only over gRPC gated by the ServiceAccount allowlist. Token validation results are cached (`cacheTtlSecs`, default 60s), so a revoked ServiceAccount retains access up to the TTL.

## Risks

* The shared singleton is a coordination point. An outage degrades to independent downloads, not serving outages, and replicas are configurable on the CR.
* Nothing prevents two `ModelExpressServer` CRs in one namespace; one-per-namespace is documented as the supported configuration.
* The `ModelExpressServer` API group is a placeholder pending upstream donation; CRs will need migration when it changes, in either topology.

## Stakeholder Impacts

| Group                | Key Contacts | Date | Impacted? |
| -------------------- | ------------ | ---- | --------- |
| ModelExpress         | Will Eaton   | 2026-08-31 | Yes |
| ODH Platform / Operator | | | Yes |
| Model Serving (KServe) | | | Maybe |
| Dashboard            | | | No |

* ODH Platform / Operator: new root-level `DataScienceCluster` component; the ODH operator gains the component handler and manifests to deploy the ModelExpress operator.
* Model Serving: inference workloads are consumers of the endpoint; nothing here requires KServe changes.

## References

* [ModelExpress upstream (ai-dynamo/modelexpress)](https://github.com/ai-dynamo/modelexpress)
* [opendatahub-io/modelexpress](https://github.com/opendatahub-io/modelexpress)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
