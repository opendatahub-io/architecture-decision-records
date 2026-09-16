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

ModelExpress is enabled through the `DataScienceCluster` CR, but this ADR proposes that its operator be deployed and managed by the existing AI Gateway operator. AI Gateway is the only current consumer. ModelExpress remains a platform capability rather than a KServe sub-component, leaving room for future RL and other workloads to consume the metadata service without making them depend on KServe.

## Goals

* Default topology that maximizes cross-namespace deduplication with minimal setup.
* Opt-in namespace-scoped topology for tenants that need metadata isolation.
* Namespace-scoped CRDs throughout, so standard RBAC is the isolation mechanism.
* Deployment ownership aligned with the existing AI Gateway operator, without making ModelExpress a KServe sub-component.

## Non-Goals

* The metadata CR schemas themselves; those are owned upstream.
* Hard singleton enforcement; the default is a convention, not a CEL constraint.
* The `DataScienceCluster` component API surface for ModelExpress (fields, defaults); that follows the standard component onboarding process.

## How

The metadata service is deployed via a namespaced `ModelExpressServer` CR reconciled by the ModelExpress operator, using upstream's Kubernetes CRD metadata backend. All ModelExpress CRDs are namespace-scoped.

**Default: cluster singleton.** One `ModelExpressServer` in a shared system namespace (e.g. `modelexpress`). The service writes all metadata CRs into that namespace, so its RBAC never leaves it. Workloads cluster-wide use the service DNS endpoint published on `status.endpoint`. Admins own the instance; tenants need only network reachability and an auth allowlist entry.

**Opt-in: namespace-scoped instance.** A tenant creates a `ModelExpressServer` in its own namespace; metadata CRs stay there and namespace RBAC governs visibility. Workloads in that namespace point at the local endpoint by configuration; there is no automatic discovery or fallback between instances. The tenant forgoes cross-namespace deduplication by design. Both topologies compose on one cluster; instances share no state.

**ServiceAccount token auth.** The service authenticates callers by Kubernetes ServiceAccount (`spec.security` on the CR): clients present a projected token, the service validates it via TokenReview against configured `tokenAudiences`, and checks the caller against an `allowedServiceAccounts` list of `namespace:serviceAccount` pairs. This is the access boundary that makes the shared endpoint safe to expose cluster-wide, and on isolated instances it pins access to the tenant's own ServiceAccounts. Shared instances should run `mode: enforce`; the default is disabled.

**Install and ownership.** The existing AI Gateway operator deploys and manages the ModelExpress operator when ModelExpress is enabled through the `DataScienceCluster` configuration. The ModelExpress operator watches `ModelExpressServer` CRs in all namespaces. This follows the same ownership pattern as the `llm-d-batch-gateway` operator: related gateway capabilities are delivered by the AI Gateway operator even when the reconciled resources have their own API and lifecycle. The arrangement is an installation and lifecycle boundary, not a runtime dependency on KServe; it establishes a platform path for future RL and other workloads to consume ModelExpress.

## AI Gateway and KServe Integration

### AI Gateway operator deployment

The AI Gateway operator is the deployment owner for the ModelExpress operator. AI Gateway is the only current ModelExpress consumer; this ownership choice establishes the integration boundary for future consumers such as RL workloads. It keeps gateway-adjacent infrastructure under the operator that already manages the AI Gateway integration surface and avoids introducing another top-level operator installation path. The ModelExpress operator remains responsible for its own `ModelExpressServer` reconciliation, status, authentication configuration, and metadata CRs; the AI Gateway operator does not absorb those APIs or reconcile those resources directly.

ModelExpress is not nested under KServe. The AI Gateway operator may deploy the ModelExpress operator independently of KServe, and future ModelExpress consumers will not need to create KServe resources. This preserves a deployment model that can grow from the current AI Gateway integration to RL and other workload integrations.

The metadata service and KServe operate at different layers. The metadata service is cluster infrastructure: admin-provisioned, auth-configured, consumed by workloads across namespaces and orchestrators. KServe is the current deployment-level integration point for serving workloads: it templates the ModelExpress-related custom resources and workload configuration needed by an inference deployment. The AI Gateway operator still owns deployment of the ModelExpress operator, and KServe does not own its lifecycle.

### Metadata service deployment

The ModelExpress operator is deployed through the AI Gateway operator. KServe does not install or manage the ModelExpress operator, but it integrates at inference deployment time by templating the ModelExpress custom resources and workload configuration associated with the serving deployment. An admin creates the shared `ModelExpressServer` CR, configures the ServiceAccount allowlist, and the ModelExpress operator publishes the gRPC endpoint on `status.endpoint`. AI Gateway is the current consumer of that endpoint; future consumers such as RL workloads can integrate directly with ModelExpress.

### LLMISVC workload templating

For a `LLMInferenceService` targeting a ModelExpress-managed model, KServe templates the deployment-level custom resources and pod spec: it injects the metadata service endpoint (read from `ModelExpressServer` status) and a projected ServiceAccount token. This gives the engine's ModelExpress sidecar or init container what it needs to register, discover peers, and coordinate downloads. KServe owns this workload templating integration; the ModelExpress operator owns reconciliation of the ModelExpress custom resources. No manual pod spec editing is required.

### Future RL and other workloads as consumers

The metadata service is not inference-specific. The deployment boundary is intentionally designed so future RL training flows, GRPO actors, reward model servers, reference policy replicas, and other workloads can consume it on the same terms: same weights, same deduplication, same peer registration (see [ModelExpress: Distributing Model Artifacts at the Speed of Light](https://developer.nvidia.com/blog/modelexpress-distributing-model-artifacts-at-the-speed-of-light) for upstream discussion of the RL use case). These workloads may run under training orchestrators (TorchX, KubeFlow Training Operator), not KServe. A KServe-owned metadata service would force future RL pipelines to depend on a serving stack they do not use, or to run a separate metadata instance and lose cross-workload deduplication.

### Relationship to LocalModelCache

KServe's `LocalModelCache` (`serving.kserve.io/v1alpha1`) is pull-based: it names a `sourceModelUri`, provisions PVs via `LocalModelNodeGroup`, and downloads weights to node-local storage. ModelExpress replaces that pull with P2P coordination (GPU-to-GPU DMA over RDMA fabrics) and metadata-driven deduplication. These are parallel caching paths. `LocalModelCache` has no metadata service endpoint field and its download agent does not speak the ModelExpress protocol. Merging them would conflate admin-scoped infrastructure with tenant-scoped caching policy. A possible future convergence: ModelExpress already falls back through its loading chain to local disk when no P2P peer is available, so integration amounts to pointing ModelExpress at the same PV that `LocalModelCache` provisions. That change would live in KServe and would not alter the deployment model defined here.

## Alternatives

* **Cluster-scoped metadata CRDs with one mandatory instance.** Trivial discovery, but per-tenant visibility becomes impossible without admission-level filtering, and the service needs cluster-scoped write access. The isolation flow stops being implementable.
* **Namespace-scoped instances only.** Uniform and isolated, but N namespaces serving the same model means N downloads and N cache copies; the deduplication win disappears exactly where it matters (large shared foundation models).
* **Redis metadata backend.** Upstream supports Redis instead of CRs. It adds a stateful service to run and secure, and loses `kubectl` inspectability, watch semantics, and RBAC scoping; the CRD backend gives us the namespace model for free.
* **Sub-component of Model Serving/KServe.** Couples the current AI Gateway integration, and future RL or other workload integrations, to KServe despite there being no technical dependency.
* **Standalone top-level operator installation.** Preserves independence, but creates a separate installation and lifecycle path for a capability already adjacent to AI Gateway concerns. It also diverges from the `llm-d-batch-gateway` operator ownership pattern.

## Security and Privacy Considerations

Metadata is low sensitivity (model names, source types, cache locations), but model names alone can reveal what a tenant is working on; the namespace-scoped topology covers that case. In the shared topology, tenants cannot read the system namespace's CRs and reach the service only over gRPC gated by the ServiceAccount allowlist. Token validation results are cached (`cacheTtlSecs`, default 60s), so a revoked ServiceAccount retains access up to the TTL.

### Possible attacks

* **Weight poisoning via peer registration.** In P2P mode, replicas that hold a model are the cache: an allowlisted workload can advertise itself as a source and back the transfer with whatever memory it wants; receivers DMA it straight into GPU HBM. Nothing attests that the exposed memory matches the model it claims to be, so the attack does not need a full model swap. A single altered value (a NaN seeded into one tensor) silently degrades or sabotages every replica that loads from that peer. Upstream checksums (per-chunk CRCs, SHA-256-sealed artifact manifests) catch corruption in transit but not a malicious peer, because the peer authors its own manifest. This is by design: ModelExpress targets serving setups where peer replicas trust each other, and the allowlist is how that trust domain is drawn. The allowlist is therefore the trust boundary: admitting a ServiceAccount to a shared instance means trusting it to serve weights to all consumers. Admins should allowlist only workloads they would let publish models cluster-wide; tenants that cannot extend that trust belong on a namespace-scoped instance, which bounds the blast radius to the namespace.
* **Metadata CR tampering.** Write access to `ModelMetadata`/`ModelCacheEntry` CRs in the service's namespace allows redirecting peers to attacker-controlled sources without touching the service at all. Only the server's ServiceAccount needs write on those CRs; no tenant RBAC should reach the system namespace.
* **Cache poisoning.** Write access to a shared cache volume poisons checkpoints for every consumer of that instance. Cache volumes should be mounted only by the service's own pods.
* **Registration flooding.** An allowlisted caller can register bogus models to churn eviction or bloat the metadata namespace. Low impact (coordination degrades, serving does not), handled by revoking the offending ServiceAccount.

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

* ODH Platform / Operator: the AI Gateway operator gains ownership of the ModelExpress operator deployment and lifecycle; the ModelExpress operator remains responsible for ModelExpress APIs and reconciliation.
* AI Gateway: owns deployment of the metadata capability and is the only current ModelExpress consumer, without taking ownership of ModelExpress custom resources.
* Future workload integrations: RL and other workloads have a platform path to consume ModelExpress without requiring KServe.
* Model Serving: ModelExpress is intentionally not nested under KServe, leaving future integrations independent of the serving stack.

## References

* [ModelExpress upstream (ai-dynamo/modelexpress)](https://github.com/ai-dynamo/modelexpress)
* [opendatahub-io/modelexpress](https://github.com/opendatahub-io/modelexpress)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
