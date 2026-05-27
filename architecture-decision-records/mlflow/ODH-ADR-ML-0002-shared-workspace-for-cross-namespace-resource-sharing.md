# Open Data Hub - Shared Workspace for Cross-Namespace Resource Sharing in MLflow

|                |            |
| -------------- | ---------- |
| Date           | May 24, 2026 |
| Scope          | OpenShift AI — MLflow, GenAI Studio, Dashboard |
| Status         | Draft |
| Authors        | [Edson Tirelli](@etirelli) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1750](https://redhat.atlassian.net/browse/RHAISTRAT-1750), [RHAISTRAT-1727](https://redhat.atlassian.net/browse/RHAISTRAT-1727), [RHAISTRAT-150](https://redhat.atlassian.net/browse/RHAISTRAT-150) |
| Other docs:    | [Shared MLflow Workspace Architecture Discussion — Meeting Notes (May 19, 2026)](https://docs.google.com/document/d/1nDW-hCC5wZXvmAHSBhRMFqQMNyNqsWWVju7UlQb8R7k/edit) |

## What

Introduce a "global workspace" mechanism in OpenShift AI that allows administrators to designate a single MLflow namespace as a shared resource namespace. Resources in this namespace — starting with prompts — are visible to all GenAI Studio users across the cluster, enabling centralized management and distribution of curated assets without breaking workspace-level isolation.

## Why

MLflow workspaces in OpenShift AI are scoped to Kubernetes namespaces, providing tenant isolation. This isolation is correct for user workloads but creates a gap when the platform needs to distribute shared resources — such as curated sample prompts, organizational prompt templates, or approved prompt libraries — across all users.

Without a sharing mechanism:

- **No centralized prompt management** — administrators cannot publish a curated set of prompts that all GenAI Studio users can discover and use. Each workspace is a silo.
- **Duplication and drift** — organizations must manually copy prompts into every workspace, leading to version inconsistency and governance gaps.
- **Sample prompt delivery blocked** — GenAI Studio cannot ship pre-built sample prompts to new users because there is no namespace from which to serve them. This gap was identified in RHAISTRAT-150 (Feb 2026) as a global namespace architectural gap blocking prompt delivery.
- **Promotion workflow undefined** — there is no governed path to "promote" a prompt from a user workspace to a shared location, or to distribute organization-approved prompts to all teams.

This ADR formalizes the architecture agreed upon on May 19, 2026, in the Shared MLflow Workspace Architecture Discussion (attendees: Eder Ignatowicz, Matt Prahl, Greg Krumbach, Juntao Wang, Edson Tirelli).

## Goals

* Enable administrators to designate a namespace as the global MLflow workspace from which shared resources are served to all GenAI Studio users.
* Maintain workspace-level isolation — users continue to work in their own namespaces; the global workspace is read-only for non-admin users.
* Provide a secure, RBAC-enforced mechanism for designating and managing the global workspace.
* Centralize the prompt API surface in the MLflow package so that all consuming components (GenAI Studio, future integrations) use a single Backend-for-Frontend (BFF) path.
* Design the initial implementation for a single global namespace while keeping the backend extensible to support multiple shared namespaces in the future.

## Non-Goals

* **Multi-namespace sharing in this iteration** — supporting multiple simultaneous global namespaces is deferred to a future iteration. The backend design accommodates this extension, but the initial scope is a single namespace.
* **Prompt promotion workflows** — the mechanism for promoting a prompt from a user workspace to the global workspace is not defined in this ADR. That workflow will be addressed separately.
* **Cross-cluster sharing** — sharing resources across multiple OpenShift clusters is out of scope. This ADR addresses single-cluster, cross-namespace sharing only.
* **Aggregate group permissions on MLflow CR** — managing group-level permissions on the MLflow Custom Resource was evaluated and removed from scope as unnecessary for the primary goal.
* **Assets beyond prompts** — while the architecture is designed to support sharing other resource types in the future, the initial implementation targets prompts only.

## How

### Architecture Overview

The global workspace architecture introduces three coordinated mechanisms: the dashboard as the administrative control plane (setting configuration and labeling namespaces), the Auth CR controller that provisions RBAC access in response to label changes, and a BFF path that serves shared resources to GenAI Studio.

```
┌─────────────────────────────────────┐
│         Cluster Settings            │
│  ┌───────────────────────────────┐  │
│  │ Global MLflow Namespace: [▼]  │  │
│  │   namespace-shared-prompts    │  │
│  └───────────────────────────────┘  │
│         (Admin sets this)           │
└──────────────┬──────────────────────┘
               │ 1. Patches ODH Dashboard config CR
               │ 2. Labels namespace (gated by SSAR)
               │ 3. Removes label from previous namespace
               ▼
┌──────────────────────────────────┐
│   Auth CR Controller             │
│  (platform operator)             │
│  - Watches for label changes     │
│  - Creates/removes RoleBindings  │
│    (mlflow-view, mlflow-edit)    │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐        ┌────────────────────────┐
│   MLflow BFF (Backend for        │        │   User's Workspace     │
│          Frontend)               │        │   (namespace: user-a)  │
│  - Reads global namespace from   │◄──────►│   MLflow instance      │
│    dashboard config              │        │   (user's own prompts) │
│  - Fetches prompts using user's  │        └────────────────────────┘
│    token (RBAC-enforced)         │
│  - Returns combined view to      │
│    GenAI Studio                  │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│        GenAI Studio UI           │
│  - Displays shared prompts from  │
│    global namespace              │
│  - Displays user's own prompts   │
│    from their workspace          │
└──────────────────────────────────┘
```

### Key Design Decisions

**1. Dashboard Configuration as Source of Truth**

The OpenShift AI Dashboard configuration is the single source of truth for which namespace is the active global workspace. Administrators set this via a dedicated field in Cluster Settings. This approach was chosen over a purely label-based mechanism because:

- It provides a clear administrative control surface (a single settings field rather than distributed labels across namespaces).
- It allows the dashboard and GenAI Studio to know which namespace to query without scanning all namespaces for labels.

When an admin saves the global namespace setting, the dashboard applies a label to the designated namespace (e.g., `global-mlflow-workspace: 'mlflow'`) and removes it from the previous namespace. The label value is the MLflow CR name rather than a boolean, allowing the design to accommodate multiple MLflow instances in the future without rearchitecting the label scheme. The Auth CR controller watches for these label changes and provisions the corresponding RoleBindings. The dashboard configuration — not the label — is the source of truth that GenAI Studio reads to determine which namespace is active.

Administrators may also label namespaces directly via OpenShift CLI, console, or GitOps tooling, bypassing the dashboard UI entirely. The Auth CR controller responds to label changes regardless of how they are applied.

**2. Single Global Namespace in Dashboard (Initial Scope)**

The dashboard UI exposes a single global namespace in its initial implementation. This is a UI-level constraint, not an infrastructure restriction — the underlying MLflow layer does not limit the number of namespaces that can be labeled as global. Multiple namespaces may carry the `global-mlflow-workspace` label simultaneously (e.g., applied via CLI or GitOps), and the Auth CR controller will provision RoleBindings for all of them. The dashboard simply presents one namespace to GenAI Studio users in this iteration.

Extending the dashboard to support multiple shared namespaces in a future iteration requires:

- Changing the dashboard configuration field from a single value to a list.
- Updating the BFF to query multiple namespaces.
- Adding a namespace selector in the GenAI Studio UI.

The single-namespace UI constraint was chosen to reduce initial complexity while validating the architecture pattern. 

**3. Self-Subject Access Review (SSAR) as Security Gate**

The ability to designate a namespace as the global workspace is gated by a Self-Subject Access Review (SSAR) call on the backend. This ensures that:

- Only users with permission to patch the dashboard configuration can change the global namespace.
- Security validation happens on the server side, not just in the UI.
- The permission model is consistent with how other administrative operations are secured in OpenShift AI.

UI-level-only validation was explicitly rejected as insufficient — backend enforcement via SSAR is mandatory.

**4. Auth CR Controller Manages Role Bindings**

The Auth CR controller (part of the platform operator) already manages RoleBindings for dashboard-created RHOAI projects. This same controller is extended to handle the global workspace case: when it detects the `global-mlflow-workspace` label on a namespace, it creates the necessary RoleBindings: authenticated users receive `mlflow-view` access in the global namespace so they can discover and use shared prompts, while RHOAI administrators receive both `mlflow-view` and `mlflow-edit` for managing shared resources.

This approach was chosen over having the dashboard's service account manage RoleBindings directly because:

- It keeps broad permissions (creating RoleBindings across namespaces) contained to the controller rather than expanding the dashboard's service account privileges.
- It supports GitOps workflows — administrators can label namespaces via declarative configuration, and the controller reacts to label changes regardless of how they are applied.
- It is architecturally consistent with how RoleBindings for other RHOAI projects are already managed.

When the global namespace is changed, the dashboard removes the label from the previous namespace and labels the new one. The Auth CR controller detects both changes and creates RoleBindings in the new namespace and removes them from the previous one.

**5. Prompt API Centralization in MLflow Package**

The prompt loading API logic is migrated from the GenAI Studio package into the MLflow package. This means:

- All prompt API calls route through the MLflow Backend-for-Frontend (BFF).
- GenAI Studio and any future consuming components use the same API path.
- The BFF uses the requesting user's token to fetch prompts, ensuring RBAC enforcement — the user only sees prompts they have access to.

**6. User Token-Based Access (No ServiceAccount Bypass)**

GenAI Studio fetches prompts from the global namespace using the user's own token via the BFF. This was a deliberate security decision: using a ServiceAccount to fetch prompts on behalf of users would bypass per-user RBAC, potentially exposing prompts that a specific user should not have access to.

The user's token ensures that standard Kubernetes RBAC applies to all resource access, including shared resources.

### Namespace Lifecycle

| Event | System Behavior |
|-------|----------------|
| Admin sets global namespace in Cluster Settings | Dashboard patches config CR, labels the namespace (`global-mlflow-workspace: '<cr-name>'`); Auth CR controller detects the label and creates RoleBindings (`mlflow-view`, `mlflow-edit`) |
| Admin changes global namespace to a different one | Dashboard removes label from previous namespace and labels the new one; Auth CR controller removes RoleBindings from the previous namespace and creates them in the new one |
| Admin clears global namespace setting | No global namespace active; GenAI Studio shows only the user's workspace prompts |
| Global namespace is deleted | Dashboard config becomes stale; GenAI Studio gracefully degrades to showing only user workspace prompts |

### UI Integration

The existing prompts management page will be reused to manage the global namespace as well. UX support is needed for a faster, easier way to select the global namespace from the project selector in GenAI Studio. A dedicated section in Cluster Settings allows admins to configure which namespace is global. The global workspace may also be highlighted in the existing project dropdown — through grouping or labeling — so that administrators can identify it at a glance.

## Alternatives

### 1. Support Multiple Global Namespaces from the Start

Allow administrators to designate multiple namespaces as shared workspaces simultaneously, with GenAI Studio aggregating resources from all of them.

**Pros:**
- More flexible for organizations with multiple teams curating different prompt sets.
- Avoids a future migration from single to multiple namespace support.

**Cons:**
- Increases initial implementation complexity (BFF must query multiple namespaces, UI needs a namespace selector or aggregation logic).
- Raises UX questions that are not yet answered (how to present prompts from multiple sources, how to handle name collisions).
- Adds query overhead across the Kubernetes API layer.

**Decision: Deferred at the UI level.** The infrastructure (labels + Auth CR controller) already supports multiple global namespaces. The deferral is in the dashboard and GenAI Studio UI, which present a single namespace to reduce initial UX complexity. Expanding to multiple namespaces requires only UI and BFF changes, not infrastructure work.

### 2. Label-Only Approach (No Dashboard Configuration)

Use Kubernetes labels on namespaces as the sole mechanism for designating global workspaces, without a dedicated dashboard configuration field.

**Pros:**
- Simpler implementation — no dashboard configuration changes needed.
- Kubernetes-native — labels are a familiar mechanism.

**Cons:**
- No single source of truth — any user with namespace-edit permissions could add or remove the label, creating an inconsistent or conflicting state.
- Dashboard must scan all namespaces for the label, which is less efficient and harder to audit.
- No administrative control surface — there is no single place where an admin can see or change the global namespace setting.

**Decision: Rejected.** Dashboard configuration provides better administrative control and a single source of truth. Labels are still used as an implementation mechanism — applied by the dashboard when the admin saves the setting, and watched by the Auth CR controller for RoleBinding provisioning — but the dashboard config CR is authoritative.

### 3. ServiceAccount-Based Backend Fetch

Use a privileged ServiceAccount to fetch prompts from the global namespace on behalf of all users, rather than using each user's individual token.

**Pros:**
- Simpler token management — no need to pass user tokens through the BFF for global namespace access.
- Avoids the need for per-user RoleBindings in the global namespace.

**Cons:**
- Bypasses per-user RBAC — all users see all prompts regardless of their individual permissions.
- Creates a privilege escalation vector — the ServiceAccount has broader access than any individual user.
- Inconsistent with the security model used for all other resource access in OpenShift AI.

**Decision: Rejected.** User token-based access maintains RBAC consistency and avoids privilege escalation. The operational cost of creating RoleBindings (automated by the operator) is low relative to the security benefit.

### 4. No Shared Workspace (Copy Prompts Per Workspace)

Do not implement cross-namespace sharing. Instead, provide tooling to copy prompts from a source location into each user's workspace.

**Pros:**
- No cross-namespace access mechanism needed.
- Each workspace is fully self-contained — no dependency on a global namespace.

**Cons:**
- Prompt copies diverge immediately — no centralized updates, no version consistency.
- Does not scale — every new workspace needs prompts copied into it.
- No governance — there is no authoritative source for "these are the organization-approved prompts."
- Does not solve the sample prompt delivery problem that motivated RHAISTRAT-1750.

**Decision: Rejected.** Copying without a shared source creates governance fragmentation and version drift that defeats the purpose of centralized prompt management.

## Security and Privacy Considerations

- **SSAR enforcement is mandatory** — the ability to designate the global namespace is gated by a backend Self-Subject Access Review, not UI-level validation alone. This prevents unauthorized users from changing the global namespace configuration even if they can access the settings UI.
- **User token-based access** — all resource access goes through the user's own token, ensuring that Kubernetes RBAC is enforced consistently. No ServiceAccount bypass is used.
- **Controller-managed RoleBindings** — role bindings (`mlflow-view`, `mlflow-edit`) in the global namespace are created by the Auth CR controller (platform operator), not by the dashboard's service account or individual users. This keeps broad permissions contained to the controller and supports GitOps-driven configuration.
- **Read-only for non-admin users** — the global namespace is read-only for regular users. Write access (creating, editing, deleting shared prompts) is restricted to administrators with appropriate namespace permissions.

## Risks

- **Single namespace becomes a bottleneck** — if the organization needs multiple curated prompt sets (e.g., per team, per department), the single-namespace model may be insufficient. *Mitigation*: the architecture is designed for multi-namespace extension; signals to watch include requests for team-specific prompt curation or namespace-level RBAC granularity within shared resources.

- **Stale dashboard configuration** — if the global namespace is deleted or becomes unavailable, the dashboard configuration points to a non-existent namespace. *Mitigation*: the BFF and GenAI Studio must handle this gracefully — fall back to showing only user workspace prompts and surface a warning to administrators.

- **RoleBinding lifecycle edge cases** — when the global namespace changes, the dashboard removes the label from the previous namespace and the Auth CR controller removes the corresponding RoleBindings. However, if labels are applied outside the dashboard (e.g., via CLI or GitOps) without updating the dashboard config CR, the two sources may diverge. *Mitigation*: document that the dashboard config CR is the authoritative source; label-only changes without a corresponding config CR update may result in RoleBindings without dashboard awareness.

- **Dependency on MLflow BFF** — centralizing prompt API calls in the MLflow BFF creates a dependency. If the BFF is unavailable, GenAI Studio cannot load shared prompts. *Mitigation*: GenAI Studio should degrade gracefully, showing user workspace prompts even if the BFF for shared prompts is unreachable.

## Stakeholder Impacts

| Group                         | Key Contacts       | Date       | Impacted? |
| ----------------------------- | ------------------ | ---------- | --------- |
| MLflow Core                   | Edson Tirelli, Matt Prahl | | Yes |
| GenAI Studio                  | Juntao Wang        | | Yes |
| AI Core Dashboard             | Eder Ignatowicz    | | Yes |
| AI Core Platform / Operator   | Greg Krumbach      | | Yes |
| UXD                           | Nana               | | Yes |
| Product Management            | Peter Double       | | Yes |
| Architects Team               | @opendatahub-io/architects | | Yes |

## References

* [Shared MLflow Workspace Architecture Discussion — Meeting Notes (May 19, 2026)](https://docs.google.com/document/d/1nDW-hCC5wZXvmAHSBhRMFqQMNyNqsWWVju7UlQb8R7k/edit)
* [RHAISTRAT-1750](https://redhat.atlassian.net/browse/RHAISTRAT-1750) — Shared MLflow workspace strategy
* [RHAISTRAT-1727](https://redhat.atlassian.net/browse/RHAISTRAT-1727) — Related strategy ticket
* [RHAISTRAT-150](https://redhat.atlassian.net/browse/RHAISTRAT-150) — Prompt Engineering & Management — Embed MLFlow UI
* [RHAIRFE-706](https://redhat.atlassian.net/browse/RHAIRFE-706) — Centralized prompt authoring, versioning, and playground integration

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| Edson Tirelli                 | May 24th, 2026 |       |
|                               |            |       |
|                               |            |       |
