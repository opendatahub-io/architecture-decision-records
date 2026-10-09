# ODH-ADR-Operator-0019 Structured reconciliation error logs

|                | |
| -------------- | --- |
| Date           | 2026-09-10 |
| Scope          | Open Data Hub operator (manager and cloudmanager) |
| Status          | Draft |
| Authors        | Heorhii Churhuliia |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHAISTRAT-2417](https://redhat.atlassian.net/browse/RHAISTRAT-2417), [RHAI-411](https://redhat.atlassian.net/browse/RHAI-411), [RHAI-529](https://redhat.atlassian.net/browse/RHAI-529) |
| Other docs     | Contributor convention embedded inline under [Convention](#convention) |

This ADR is the canonical convention specification, including for external
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

* Identity fields for reconciliation errors: `name` (always, except list-level
  failures that have no single object identity), `namespace`
  (namespaced only), and `resourceKind` — required whenever it is not already
  carried as `controllerKind` by the request logger, and skipped when it would
  duplicate `controllerKind` (see
  [When to set resourceKind](#when-to-set-resourcekind)).
* Stable, identity-free message strings.
* Cluster-scoped resources (`DataScienceCluster`, `DSCInitialization`) do not add
  `namespace` themselves; the request logger's `"namespace": ""` is expected.
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
| `name` | object name | always, except list-level failures with no single object (e.g. a watch-mapper list error) |
| `namespace` | object namespace | namespaced resources only |
| `resourceKind` | Kubernetes kind (`DataScienceCluster`, `Auth`, `Dashboard`, …) | when kind is not already implied (see [When to set resourceKind](#when-to-set-resourcekind)) |

Do not invent aliases for these keys (`Request.Name`, `resource_kind`, `ns`,
`DSCInitialization` as a namespace key). Extra keys for other objects
(`component`, `module`, `deployment` / `deploymentNamespace`,
`child` / `childNamespace`, `configmap`, `webhook`, `path`) are allowed and
encouraged.

controller-runtime already injects `name`, `namespace`, and `controllerKind` on
the logger taken from `logf.FromContext(ctx)` during `Reconcile`. Do not
duplicate those keys. Add `resourceKind` only when it is not the same as
`controllerKind` (watch mappers, or loggers that did not come from the request
context). `resourceKind` always describes the same object as `name`; a child
object gets its own key, never `resourceKind` (see
[Parent vs child identity](#parent-vs-child-identity)).

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

// After — identity taken over explicitly (logger without request context).
// Cluster-scoped: don't add namespace yourself; resourceKind is set here only
// because controllerKind is not inherited on this logger.
log.Error(err, "Failed to retrieve resource.",
    "resourceKind", "DSCInitialization", "name", req.Name)

// After — on the request logger (the common Reconcile case): name/namespace and
// controllerKind are already injected, so drop the custom keys entirely.
log.Error(err, "Failed to retrieve resource.")
```

**3. Missing `resourceKind`** — `name` and `namespace` are not enough to search
across controllers. Kind is implied for a dedicated component reconciler, but
not for the module controller, service controllers, watch mappers, or
cloudmanager, which each touch more than one kind.

```go
// Before — cannot filter "all Auth list failures" without parsing the message
log.Error(err, "Failed to get AuthList")

// After — watch mapper logs the watched kind, not the parent reconciler.
// A list failure has no single object, so "name" is omitted; "resourceKind"
// still identifies which kind the list was for.
log.Error(err, "Failed to get AuthList", "resourceKind", "Auth")
```

### When to set resourceKind

| Situation | `resourceKind` |
| --- | --- |
| Dedicated component / DSC / DSCI `Reconcile` using `logf.FromContext(ctx)` | Skip. It would duplicate the `controllerKind` the request logger already carries. Add it only when the call does not go through that logger. |
| Service controllers, module controller / handlers, cloudmanager | Required. Kind is not implied by the controller name. |
| Watch mappers and list handlers | Required. Set it to the watched kind (`Auth`, `GatewayConfig`), not the parent. |
| Child object of the reconciled CR (component, module, Deployment, ConfigMap, cloudmanager GC/cleanup target) | Leave `resourceKind` describing the parent (usually dropped because it equals `controllerKind`). Never put the child's kind in `resourceKind`. Give the child its own dedicated key (see [Parent vs child identity](#parent-vs-child-identity)). |

### Parent vs child identity

**`resourceKind` always describes the same object as `name`** — the reconciled
(parent) object carried by the request logger. `name`, `namespace`, and
`resourceKind` are one set and must never point at different objects.

On a `Reconcile` path the request logger already carries the parent's `name` /
`namespace` / `controllerKind`, so `resourceKind` equals `controllerKind` and is
dropped as a duplicate. Reconcile code therefore touches *other* objects without
ever reassigning the reserved keys:

* A child or secondary object gets its **own dedicated key**, never
  `resourceKind`: `component`, `module`, `deployment` / `deploymentNamespace`,
  `child` / `childNamespace` (cloudmanager GC/cleanup), `configmap`, `webhook`,
  `path`.
* Do not log the child's kind in `resourceKind` next to the parent's `name`, and
  do not reuse `name` / `namespace` for the child.

```go
// Child component CR — parent identity stays on the context logger;
// the child gets its own key. resourceKind is NOT set to the child's kind.
log.Error(err, "failed to delete component CR", "component", handler.GetName())

// Child Deployment — do not steal the reserved "name" / "namespace" keys
log.Error(err, "failed to inject env vars into Deployment",
    "deployment", deploy.GetName(), "deploymentNamespace", deploy.GetNamespace())

// Cloudmanager GC — the deleted child uses child / childNamespace
log.Error(err, "cleanup delete failed",
    "child", obj.GetName(), "childNamespace", obj.GetNamespace())
```

If a log line is genuinely *about* the child (not the reconciled parent), log the
child as the full set — `name` / `namespace` / `resourceKind` all describing the
child — rather than mixing parent and child across those keys.

### Scoping

Cluster-scoped resources (`DataScienceCluster`, `DSCInitialization`) do not add
`namespace` themselves — never pass `req.Namespace` explicitly, as it is empty or
misleading. Note that on a `Reconcile` path controller-runtime's request logger
always injects `namespace` from the request, so cluster-scoped errors logged
through `logf.FromContext(ctx)` still carry `"namespace": ""`; that empty value is
expected and should not be "fixed" by dropping the request logger. When logging
through that request logger, only `name` is explicit — `resourceKind` is skipped
because it would duplicate the inherited `controllerKind` (per
[When to set resourceKind](#when-to-set-resourcekind)); set `resourceKind`
explicitly only when the request logger is unavailable (for example a watch
mapper or a logger that did not come from the request context). Namespaced
resources (component CRs, module CRs, service CRs, cloudmanager CRs) must include
`namespace` when identity is logged explicitly.

### CI enforcement

`make lint` runs `go test ./cmd/loglint/…` and `go run ./cmd/loglint ./…`. To
scan controller packages locally (will fail on call sites not yet converted by
RHAI-525–528):

```sh
go run ./cmd/loglint ./internal/controller/... ./pkg/controller/...
```

The analyzer flags:

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
