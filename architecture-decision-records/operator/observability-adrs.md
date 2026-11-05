# Platform Observability ADR Index

This index collects the architecture decision records that define platform observability
contracts for Open Data Hub (ODH) and Red Hat OpenShift AI (RHOAI): how component metrics
are exposed and scraped, how recording rules and alerts reach the platform monitoring
stack, how traces are collected, and how dashboards and metric-based autoscaling consume
that data.

It is a discovery aid, not an ADR and not a release commitment. The status shown here is a
copy: the record itself is authoritative.

| Topic | ADR | Status | Author |
| --- | --- | --- | --- |
| Component metrics scraping and monitor enrollment | [ODH-ADR-Operator-0010 — RHOAI component metrics scraping guidelines](ODH-ADR-Operator-0010-Observability-component%20metrics%20scraping.md) | Proposed | [@dayakar349](https://github.com/dayakar349) |
| Perses dashboard guidelines | [ODH-ADR-Operator-0011 — Perses dashboard guidelines](ODH-ADR-Operator-0011-Perses-dashboard-guidelines.md) | Draft | [@dayakar349](https://github.com/dayakar349) |
| Observability and tracing strategy | [ODH-ADR-Operator-0009 — Observability and tracing strategy](ODH-ADR-Operator-0009-observability-tracing-strategy.md) | Draft | [@den-rgb](https://github.com/den-rgb) |
| Metrics-based autoscaling | [ODH-ADR-Operator-0011 — RHOAI component metrics-based autoscaling](ODH-ADR-Operator-0011-observability-metrics-autoscaling.md) | Draft | [@cgoodfred](https://github.com/cgoodfred) |
| Component recording rules and alerts on the Data Science MonitoringStack | [ODH-ADR-Operator-0020 — Component recording rules and alerts on the Data Science MonitoringStack](ODH-ADR-Operator-0020-component-recording-rules-on-monitoringstack.md) | Draft | [@asanzgom](https://github.com/asanzgom) |

## Reading the entries together

The records are complementary rather than alternatives:

- **0010** defines how component metrics are exposed and enrolled into the platform
  monitoring stack.
- **0020** defines where recording rules and alerts are delivered and evaluated, and
  depends on the enrollment model in 0010.
- **0011 (Perses)** defines how dashboards consume those metrics.
- **0011 (autoscaling)** defines how metric-based autoscaling consumes them.
- **0009** defines the tracing path, which is separate from the metrics pipeline.

## Duplicate ADR numbers

`ODH-ADR-Operator-0011` is used by two merged records, and several other numbers in
`operator/` are also duplicated from parallel reviews. Until the duplicates are resolved,
link these records by filename or full URL, not by number:

- `ODH-ADR-Operator-0011-Perses-dashboard-guidelines.md`
- `ODH-ADR-Operator-0011-observability-metrics-autoscaling.md`

Renumbering merged decisions is out of scope for index maintenance.

## Ownership and maintenance

**Maintainer: TBD.** The platform/architect owner responsible for this index still has to
be confirmed by [@opendatahub-io/architects](https://github.com/orgs/opendatahub-io/teams/architects);
see [issue #183](https://github.com/opendatahub-io/architecture-decision-records/issues/183).

Update this index when:

- an indexed ADR changes status (for example `Draft` to `Approved`), or is superseded;
- a new ADR establishes or changes a platform observability contract;
- a duplicate number is resolved.

When adding an entry:

1. Add a row to the table with the topic, filename link, current status, and author.
2. Keep the status identical to the ADR's metadata table — do not upgrade it here.
3. Do not renumber or move existing ADRs as part of the change.
