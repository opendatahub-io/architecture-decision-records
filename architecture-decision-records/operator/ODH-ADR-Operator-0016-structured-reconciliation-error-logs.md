# ODH-ADR-Operator-0015 Structured reconciliation error logs

|                | |
| -------------- | --- |
| Date           | 2026-09-10 |
| Scope          | Open Data Hub operator (manager and cloudmanager) |
| Status          | Draft — publish to [architecture-decision-records/operator](https://github.com/opendatahub-io/architecture-decision-records/tree/main/architecture-decision-records/operator) |
| Authors        | Heorhii Churhuliia |
| Supersedes     | N/A |
| Superseded by  | N/A |
| Tickets        | [RHAISTRAT-2417](https://redhat.atlassian.net/browse/RHAISTRAT-2417), [RHAI-411](https://redhat.atlassian.net/browse/RHAI-411), [RHAI-529](https://redhat.atlassian.net/browse/RHAI-529) |
| Other docs     | [Contribution guideline](../structured-logging.md) |

This file is the in-repo copy of the convention specification. The
organization ADR repository is the canonical publication target for external
operators (Loki Operator, Tempo Operator). Adoption by those operators is
voluntary and is not a delivery dependency (platform policy: RHOAI does not
install or control external operator dependencies).

## What

Standardize rhods-operator / Open Data Hub operator reconciliation **error** logs
on structured key-value pairs: `name`, `namespace`, `resourceKind` (camelCase).

Contributor-facing rules and before/after examples live in
[structured-logging.md](../structured-logging.md). This ADR records the decision
and the three problems it solves.

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
* Enforce with `cmd/loglint` (see contribution guideline).

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
