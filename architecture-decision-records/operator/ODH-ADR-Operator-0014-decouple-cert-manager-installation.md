# Decouple cert-manager Installation from the Cloud Controller Manager

|                |            |
| -------------- | ---------- |
| Date           | 2026-07-14 |
| Scope          | Operator |
| Status         | Draft |
| Authors        | [Davide Bianchi](@davidebianchi) |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHOAIENG-69977](https://redhat.atlassian.net/browse/RHOAIENG-69977) |
| Other docs     | [ODH-ADR-Operator-0013 — Extending RHAI to Generic Kubernetes](ODH-ADR-Operator-0013-extending-rhai-to-non-openshift-kubernetes.md) |

## What

Move cert-manager installation out of the Cloud Controller Manager (CCM). The CCM will continue to manage PKI resources (ClusterIssuers, root CA) but will no longer be responsible for deploying cert-manager itself. cert-manager installation becomes the responsibility of the deployment mechanism used for the target environment (e.g., Helm chart or manual installation).

## Why

The CCM currently both installs cert-manager and depends on it for PKI operations. This creates a logical dependency: the controller owns a component it depends on. While the current implementation handles this through careful sequencing (bootstrap no-ops when CRDs are absent, CRD watches trigger re-reconciliation), the coupling makes the controller's scope broader than necessary and creates ownership ambiguity.

Several factors make cert-manager a poor fit for controller-managed installation:

* **Commonly pre-installed.** cert-manager is a standard component in vanilla Kubernetes clusters. Many clusters already have it installed and managed independently.
* **Uninstall is unreliable.** The CCM tries to remove cert-manager on uninstall, but this is blocked by [CM-1019](https://issues.redhat.com/browse/CM-1019): the cert-manager-operator automatically recreates the `CertManager/cluster` CR after deletion, preventing the two-phase cleanup mechanism from completing. The CCM already works around this disabling the normal cleanup flow. This means the CCM does not fully own the cert-manager lifecycle, making the current ownership model incomplete.
* **Shared dependency.** cert-manager is used by multiple components (e.g., the CCM and the operator). Installing it from the CCM creates an implicit ordering constraint and tight coupling between components.
* **Installation speed.** Removing cert-manager installation from the CCM reduces its startup footprint and speeds up the overall installation process.
* **Enables deeper integration.** Decoupling cert-manager installation opens the path for other architectural changes, such as using the CCM as a module within the operator. This is harder when the CCM owns shared infrastructure dependencies.

## Goals

* Eliminate the logical dependency between the CCM and cert-manager
* Establish a clear boundary: the deployment mechanism installs cert-manager, the CCM manages PKI resources (ClusterIssuers, root CA)
* Support environments where cert-manager is already installed
* Define a migration path for existing deployments where the CCM currently owns cert-manager
* Deprecate the `certManager.managementPolicy` field from the CCM CR API

## Non-Goals

* Replacing cert-manager with a different certificate management solution
* Changing how the CCM manages PKI resources (ClusterIssuers, Certificates, root CA)
* Modifying cert-manager configuration or version requirements

## How

### Current architecture

The CCM deploys cert-manager through two layers:

| Layer | Content | Managed by |
| --- | --- | --- |
| Layer 1 (chart resources) | cert-manager-operator Deployment, RBAC, SA, CertManager CR, Namespaces, CRDs | CCM via `cert-manager-operator` Helm chart, deployed through SSA |
| Layer 2 (runtime operand) | cert-manager controller, webhook, cainjector Deployments, Services | cert-manager-operator at runtime, watching CertManager CR |

The CCM directly owns Layer 1 resources (ownerReferences to the CCM CR). Layer 2 resources are created by the cert-manager-operator and are not directly owned by the CCM.

The `certManager.managementPolicy` field (`Managed`/`Unmanaged`) already exists in the CCM CR API (each provider has its own CR, e.g., `AzureKubernetesEngine`, `CoreWeaveKubernetesEngine`). When set to `Unmanaged`, the cleanup mechanism deletes owned Layer 1 resources, while preserving:

* **Namespaces** (`cert-manager`, `cert-manager-operator`): protected as unremovable GVKs in GC
* **CRDs**: protected as unremovable in GC
* **Root CA Certificate CR**: created by the CCM's bootstrap action (not part of the cert-manager-operator chart), and protected as long-lived PKI infrastructure by the GC mechanism. Not deleted during cleanup.
* **Root CA Secret** (in `cert-manager` namespace): created by cert-manager in response to the Certificate CR. Has no ownerReference to the CCM CR, so it is not affected by cleanup.
* **ClusterIssuers**: cluster-scoped, protected as long-lived PKI infrastructure by the GC mechanism

### Boundary definition

The responsibility split between installation mechanism and controller after decoupling:

| Responsibility | Owner | Resources |
| --- | --- | --- |
| cert-manager installation (CRDs, Deployment, RBAC) | Deployment mechanism (e.g., Helm chart, manual) | cert-manager namespace, Deployment, CRDs |
| cert-manager lifecycle (upgrades, scaling) | Deployment mechanism | cert-manager Deployment, CRDs |
| PKI resources (root CA, ClusterIssuers, per-namespace and webhook certificates) | CCM | ClusterIssuer, Certificate, Secret |
| GC protection of PKI resources | CCM | GC-protected ClusterIssuers and root CA Certificate |
| CertManager monitoring condition | CCM | Status conditions on CCM CR |

### Configuration

After decoupling, cert-manager installation is no longer controlled by the CCM.

The `certManager.managementPolicy` field is deprecated. During the transition release, the field is still accepted but enforced as `Unmanaged` at runtime (any value other than `Unmanaged` is rejected with a validation error). The `certManager` field itself is retained for PKI-related configuration (e.g., certificate names, issuer references). Designing the PKI configuration surface is out of scope for this ADR.

On upgrade, the deployment mechanism should validate that the old CCM CR on the cluster has been migrated (cleanup completed) before deploying the new version.

### Helm chart integration

The Helm chart is the primary deployment mechanism for generic Kubernetes environments and takes ownership of cert-manager installation. The chart deploys the **cert-manager-operator**, which in turn deploys cert-manager itself (controller, webhook, cainjector). Users who already have cert-manager installed — either directly or through their own cert-manager-operator — can disable this with a chart flag. The chart pins a specific cert-manager version to ensure compatibility with the CCM's PKI expectations.

### CCM changes

The CCM will:

* **Remove** the cert-manager installation code entirely. The deployment mechanism should block upgrades until the migration is completed, and GC handles any remaining stale resources via generation mismatch
* **Deprecate** the `managementPolicy` field in the `AzureKubernetesEngine`, `CoreWeaveKubernetesEngine` and `AWSKubernetesEngine` CR API types. During the transition release the field is accepted but enforced as `Unmanaged`. The `certManager` field is retained for PKI configuration
* Retain all PKI resource management (ClusterIssuers, root CA Certificate, webhook Certificate, Secrets)
* Retain GC protection of bootstrap PKI resources (ClusterIssuers, root CA Certificate)
* Retain CRD monitoring and status conditions (including the existing degraded condition when cert-manager CRDs are not present)
* Reduce RBAC for cert-manager operator resources to read-only (monitoring only, no create/update/delete)
* Retain RBAC for cert-manager PKI resources (ClusterIssuers and Certificates)

The change applies uniformly to all CCM providers: AWS, Azure, and CoreWeave (CKS).

### Migration considerations

Existing deployments where the CCM owns cert-manager must complete cleanup on the old CCM version before upgrading.

During this transition, cert-manager is temporarily unavailable for new certificate issuance. PKI trust chain resources are preserved. Existing TLS connections continue working.

## Open Questions

None at this time.

## Alternatives

### Keep cert-manager installation in the CCM

Maintain the current architecture where the CCM installs and manages cert-manager.

* Pro: No migration effort, single component to manage
* Con: Logical dependency persists; lifecycle mismatch between cert-manager and CCM continues; CCM scope is broader than necessary

## Security and Privacy Considerations

* During migration, there must be no window where the root CA Secret is destroyed. The migration strategy preserves the `cert-manager` namespace (unremovable by GC), the root CA Certificate CR (GC-protected), and the root CA Secret (not owned by the CCM CR), ensuring the trust chain remains intact.
* During migration, existing TLS Secrets are preserved. Only new certificate issuance is paused during the transition window.

## Risks

* **Migration disruption**: During migration, cert-manager is temporarily unavailable. New certificate issuance is paused until the deployment mechanism re-installs cert-manager. Existing TLS Secrets and the root CA are preserved.
* **Existing automation**: Teams with automation that depends on the CCM installing cert-manager will need to update their workflows.
* **Upgrade requires migration**: Existing installs require a migration step on the old CCM version before upgrading. The deployment mechanism should validate that migration is complete before deploying the new version.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| --- | --- | --- | --- |
| AI Core Platform (Operator) | | 2026-06-30 | Yes |
| RHAII on XKS | | 2026-06-30 | Yes |

* **AI Core Platform (Operator)**: Directly impacted. CCM code changes (remove cert-manager installation code, reduce RBAC), deployment validation and migration flow, testing.
* **RHAII on XKS**: Directly impacted. Deployment validation and migration flow, testing.

## References

* [RHOAIENG-69977](https://redhat.atlassian.net/browse/RHOAIENG-69977)
* [CM-1019](https://issues.redhat.com/browse/CM-1019) — cert-manager-operator recreates CertManager/cluster CR after deletion

## Reviews

| Reviewed by | Date | Notes |
| --- | --- | --- |
| | | |
