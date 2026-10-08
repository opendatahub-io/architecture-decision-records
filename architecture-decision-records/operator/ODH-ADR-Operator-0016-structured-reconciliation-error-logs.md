# ODH-ADR-Operator-0016 Structured reconciliation error logs

|                | |
| -------------- | --- |
| Date           | 2026-09-10 |
| Scope          | Open Data Hub operator (manager and cloudmanager) |
| Status          | Draft — publish to [architecture-decision-records/operator](https://github.com/opendatahub-io/architecture-decision-records/tree/main/architecture-decision-records/operator) |
| Authors        | Heorhii Churhuliia |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHAISTRAT-2417](https://redhat.atlassian.net/browse/RHAISTRAT-2417), [RHAI-411](https://redhat.atlassian.net/browse/RHAI-411), [RHAI-529](https://redhat.atlassian.net/browse/RHAI-529) |
| Other docs     | Contributor convention embedded inline under [Convention](#convention) |

This file is the in-repo copy of the convention specification. The
organization ADR repository is the canonical publication target for external
operators (Loki Operator, Tempo Operator). Adoption by those operators is
voluntary and is not a delivery dependency (platform policy: RHOAI does not
install or control external operator dependencies).

## What

Standardize rhods-operator / Open Data Hub operator reconciliation **error** logs
on structured key-value pairs: `name`, `namespace`, `resourceKind` (camelCase).

This ADR records the decision and the three problems it solves. So external
operators can adopt the convention without a sign-in, the contributor-facing
rules and before/after examples are embedded inline under
[Convention](#convention) below.

## Why

Operators debug reconciliation failures by asking “which object, in which
namespace, of which kind?” Message strings such as
`failed to reconcile Dashboard 'example' in namespace 'redhat-ods-applications'`
are readable in a single line but cannot be filtered as fields in Loki or
CloudWatch.

Three distinct problems showed up in the same log stream:

1. **Message-embedded identity** — name and namespace interpolated into the
   message (the original audit estimated ~75% of `log.Error()` calls; a
   repository-wide reconcile-path audit found ~0% on current `log.Error()`
   sites; the convention still forbids it so it cannot return).
2. **Non-standard DSCI keys** — `Request.Name` and `DSCInitialization` as field
   names, so a query for `name=` skipped DSCI.
3. **Missing `resourceKind`** — framework loggers already emit `name` /
   `namespace` / `controllerKind` at Reconcile entry, but kind is not queryable
   as `resourceKind`, and several controllers are not 1:1 with a kind.

camelCase matches controller-runtime's existing keys (`name`, `namespace`,
`controllerKind`).

## Goals

* Required identity fields for reconciliation errors: `name`, `namespace`
  (namespaced only), `resourceKind`.
* Stable, identity-free message strings.
* Cluster-scoped resources (`DataScienceCluster`, `DSCInitialization`) omit
  `namespace`.
* Do not duplicate controller-runtime request-logger keys.
* Do not reuse `name` / `namespace` / `resourceKind` for child objects.
* CI analyzer (`cmd/loglint`) prevents regressions.
* Publish this spec for voluntary external adoption.

## Non-Goals

* Overhauling info/debug/warning logs or non-reconcile paths (webhooks,
  upgrade, bootstrap) in this change.
* Custom dashboards or alert rules (out of RFE scope).
* Mandating Loki Operator or Tempo Operator changes.
* Changing log collection pipeline configuration.

## How

* Use `logr` key-value arguments, not `fmt.Sprintf` identity in the message.
* Take the logger from `logf.FromContext(ctx)` on Reconcile paths.
* Add `resourceKind` when the logger is not the request logger, or when the
  logged kind differs from `controllerKind` (watch mappers, mixed-kind
  controllers).
* Enforce with `cmd/loglint` (see [CI enforcement](#ci-enforcement) below).

## Convention

Apply this when adding or changing `log.Error()`, `logf.FromContext(ctx).Error`,
`l.Error`, or `logger.Error` calls on a reconcile path (controller, action,
module handler, or cloudmanager). The convention exists so operators can query
Loki or CloudWatch by resource identity without parsing message text.

### Required fields

Reconciliation error logs identify the reconciled object with camelCase keys:

| Key | Value | When |
| --- | --- | --- |
| `name` | object name | always |
| `namespace` | object namespace | namespaced resources only |
| `resourceKind` | Kubernetes kind (`DataScienceCluster`, `Auth`, `Dashboard`, …) | when kind is not already implied (see [When to set resourceKind](#when-to-set-resourcekind)) |

Do not invent aliases for these keys (`Request.Name`, `resource_kind`, `ns`,
`DSCInitialization` as a namespace key). Extra keys for other objects
(`component`, `module`, `deployment`, `path`) are allowed and encouraged.

controller-runtime already injects `name`, `namespace`, and `controllerKind` on
the logger taken from `logf.FromContext(ctx)` during `Reconcile`. Do not
duplicate those keys. Add `resourceKind` only when it is not the same as
`controllerKind` (watch mappers, child objects, or loggers that did not come
from the request context).

### Three problems this convention fixes

**1. Message-embedded identity** — resource name or namespace in the message
string cannot be queried as a field.

```go
// Before — identity is trapped in the message
log.Error(err, "failed to reconcile Dashboard 'example' in namespace 'redhat-ods-applications'")

// After — identity is structured; message is stable
log.Error(err, "reconciliation failed",
    "name", dashboard.Name, "namespace", dashboard.Namespace, "resourceKind", "Dashboard")
```

If the logger already came from `logf.FromContext(ctx)` in `Reconcile`, the
`name` / `namespace` pairs above are redundant — keep the message identity-free
either way: `log.Error(err, "reconciliation failed")`.

**2. Non-standard key names (DSCI)** — the `DSCInitialization` controller used
`Request.Name` for name and `DSCInitialization` as a namespace key, so a query
for `name=` missed DSCI.

```go
// Before
log.Error(err, "Failed to retrieve DSCInitialization resource.",
    "DSCInitialization Request.Name", req.Name)

// After — cluster-scoped: omit namespace
log.Error(err, "Failed to retrieve resource.",
    "resourceKind", "DSCInitialization", "name", req.Name)
```

**3. Missing `resourceKind`** — `name` and `namespace` are not enough to search
across controllers. Kind is implied for a dedicated component reconciler, but
not for the module controller, service controllers, watch mappers, or
cloudmanager, which each touch more than one kind.

```go
// Before — cannot filter "all Auth list failures" without parsing the message
log.Error(err, "Failed to get AuthList")

// After — watch mapper logs the watched kind, not the parent reconciler
log.Error(err, "Failed to get AuthList", "resourceKind", "Auth")
```

### When to set resourceKind

| Situation | `resourceKind` |
| --- | --- |
| Dedicated component / DSC / DSCI `Reconcile` using `logf.FromContext(ctx)` | Optional. `controllerKind` already identifies the reconciler; add it when the call bypasses that logger or you want the field present for queries. |
| Service controllers, module controller / handlers, cloudmanager | Required. Kind is not implied by the controller name. |
| Watch mappers and list handlers | Required. Set it to the watched kind (`Auth`, `GatewayConfig`), not the parent. |
| Child object of the reconciled CR (component, module, Deployment) | Do not reuse `name` / `namespace` / `resourceKind`. Use a dedicated key; parent identity stays on the context logger. |

```go
// Parent identity comes from the context logger; child uses its own key
log.Error(err, "failed to delete component CR", "component", handler.GetName())

// Child Deployment — do not steal the reserved "name" / "namespace" keys
log.Error(err, "failed to inject env vars into Deployment",
    "deployment", deploy.GetName(), "deploymentNamespace", deploy.GetNamespace())
```

### Scoping

Cluster-scoped resources (`DataScienceCluster`, `DSCInitialization`) do not emit
`namespace` — omit the key rather than sending `req.Namespace`, which is empty or
misleading. `name` and `resourceKind` still apply. Namespaced resources
(component CRs, module CRs, service CRs, cloudmanager CRs) must include
`namespace` when identity is logged explicitly.

### CI enforcement

`make lint` runs `go test ./cmd/loglint/…` and `go run ./cmd/loglint ./…`. The
analyzer flags:

* `log.Error()` message strings that embed identity (`in namespace`,
  `namespace %s`, `Request.Namespace`, `name %s`, `named %s`, `Request.Name`, …).
* Non-standard identity keys (`Request.Name`, `Request.Namespace`,
  `resource_kind`, `ns` used as namespace).
* Explicit `"name"` without `"resourceKind"` (the call is taking over identity
  logging and must include kind).

Suppress a true false positive with `//nolint:odhlog` on the same line, and
explain why.

### Query examples

```logql
{app="opendatahub-operator"} |= "reconciliation failed" | json | name="default-dsc"
{app="opendatahub-operator"} | json | resourceKind="Auth"
```

After the DSCI migration, replace `| json | Request_Name="default-dsci"` (or the
exact encoded key your pipeline used) with `| json | name="default-dsci"`.

## Alternatives

1. **Keep message-embedded identity.** Rejected: it cannot be queried as a
   field and diverges from controller-runtime.
2. **Use Kubernetes `kind` / `metadata.name` as log keys.** Rejected: `kind` is
   easy to confuse with the object's `TypeMeta`, and controller-runtime already
   standardized on `name` / `namespace`.
3. **Only add `controllerKind` (already injected).** Rejected: watch mappers
   and mixed-kind controllers need the *resource* kind, which is not always
   the controller's kind.

## Stakeholder Impacts

| Group | Impacted? | Notes |
| --- | --- | --- |
| ODH Operator | Yes | Emit and document the fields; CI enforcement |
| Observability / Support | Yes | DSCI key rename can break queries that filter `Request.Name` |
| log-normalizer-operator | Only if field mappings special-case the old DSCI keys | |
| odh-observability | Only if alerts/queries match old keys or message text | |
| Loki Operator, Tempo Operator | Voluntary | Convention spec only |

## Reviews

| Reviewed by | Date | Notes |
| --- | --- | --- |
| | | |
