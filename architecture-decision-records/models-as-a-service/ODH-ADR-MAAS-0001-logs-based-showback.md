# Open Data Hub - Architecture Decision Record

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-16 |
| Scope          | Models-as-a-Service (MaaS), Observability |
| Status         | Draft |
| Authors        | [Arik Hadas](@ahadas) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-1314](https://redhat.atlassian.net/browse/RHAISTRAT-1314) |
| Other docs:    | none |

## What

This ADR introduces logs-based showback for Models-as-a-Service (MaaS). Instead of deriving per-user, per-model usage (request counts, token counts) from Prometheus metrics, an EnvoyFilter is added to the AI Gateway to emit structured access logs carrying this usage data. MaaS deploys an OTEL collector that reads these logs and exports them to Loki, and a proxy that serves per-user usage information. This ADR also covers surfacing the resulting usage data through an administrator dashboard and a personal usage dashboard.

The feature is delivered in two steps, gated behind a feature flag in the MaaS Config: it ships as Tech Preview, disabled by default, in RHOAI 3.5, and is planned to become the default showback mechanism in RHOAI 3.6.

## Why

Visibility into consumption is essential when running AI workloads at scale: tenants and platform operators need to know who is using which models, and how much, in order to allocate cost and plan capacity.

MaaS previously relied on metrics-based showback (Prometheus counters/histograms labeled by tenant, user, and model). In practice this approach fell short in three ways:

- **Cardinality**: labeling metrics by the combination of user, subscription, and model produces a number of time series that grows with each new user, subscription, or model. At MaaS scale this cardinality became too high for Prometheus to handle reliably, which was the main driver for moving away from metrics.
- **Accuracy**: the reported usage numbers were inaccurate in some cases.
- **Retention**: Prometheus retention windows are too short to cover typical showback and billing cycles.

Emitting a per-call log line for each inference call, carrying the per-call detail needed (tenant, user, model, token counts), and using it as the source of truth for showback avoids all of these issues: aggregation happens after ingestion rather than through high-cardinality time series, so there is no cardinality limit on the number of user/subscription/model combinations that can be tracked.

## Goals

* Produce accurate per-tenant, per-user, per-model usage data (request counts, token counts) for showback.
* Retain usage data long enough to cover showback and billing cycles, independent of metrics retention limits.
* Preserve per-call granularity so usage can be aggregated and re-aggregated along different dimensions (time window, model, user) after the fact.
* Lay the groundwork for usage-based billing in the future.
* Surface the resulting usage data through two dashboards: an administrator dashboard covering all users, and a personal usage dashboard, available to every user (including administrators), scoped to their own usage.
* Ship the feature incrementally, behind a feature gate, starting as Tech Preview and later becoming the default showback mechanism.

## Non-Goals

* Replacing Prometheus metrics for operational monitoring and alerting. This ADR only changes the source of truth for showback, not general observability.
* Implementing billing or invoicing itself; this ADR only covers collecting and aggregating the usage data that a future billing system would consume.
* Deploying or managing Loki itself. In RHOAI 3.5, this ADR assumes Loki is already configured by the user in the RHOAI monitoring namespace. In RHOAI 3.6, the user is only expected to install the Loki Operator as a prerequisite; configuring Loki (via a LokiStack CR) is handled by the odh-observability module.
* Gathering usage from gateways other than the default `maas-gateway` in RHOAI 3.5; usage from all AITenant gateways is planned for RHOAI 3.6.
* Correlating usage with API keys in RHOAI 3.5; this is planned for RHOAI 3.6.

## How

### Architecture

* An **EnvoyFilter** is deployed on the AI Gateway to emit a structured, per-request log line containing tenant, user, model, and token usage information for every inference call.
* An **OTEL collector** reads these log lines from the EnvoyFilter and exports them to **Loki**, which is the log store backing showback. Which component deploys the OTEL collector differs between RHOAI 3.5 and 3.6; see the rollout plan below.
* MaaS deploys a **proxy** that queries Loki and serves per-user usage information, consumed by the personal usage dashboard.
* An **administrator dashboard** and a **personal usage dashboard** (available to every user, including administrators, scoped to their own usage) are built with **Perses** on top of this data.
* The whole feature is controlled by a **feature gate in the MaaS Config**.

### Rollout plan

The feature is delivered in two steps:

#### RHOAI 3.5 (Tech Preview)

* The feature gate in the MaaS Config is **disabled by default**; the feature is Tech Preview.
* Loki is assumed to be already configured by the user in the RHOAI monitoring namespace; MaaS does not deploy or manage it.
* Usage is gathered only from the default `maas-gateway`, not from gateways belonging to other AITenants.
* MaaS deploys the OTEL collector and the per-user usage proxy.
* Usage data is not correlated with API keys.

#### RHOAI 3.6 (planned)

* Usage is gathered from all gateways belonging to AITenants, not just the default gateway.
* Usage data is correlated with API keys.
* The feature is **enabled by default**. User identity information, however, is still not gathered by default; it is only collected once enabled in `MaaSTenantConfig`.
* This becomes the default showback mechanism: the metrics-based usage dashboard is no longer deployed by default.
* Deploying the OTEL collector is no longer MaaS's responsibility. Users are expected to install the **Loki Operator** themselves as a prerequisite; once usage logging is configured in the **DSCI**, the **odh-observability** module in the RHOAI operator configures Loki (via a `LokiStack` CR) and deploys the OTEL collector, rather than MaaS.

## Alternatives

### Metrics-based showback (previous approach)

Prometheus counters/histograms labeled by tenant, user, and model, scraped and aggregated by the monitoring stack.

**Why rejected**: labeling by the user+subscription+model combination produced too many time series as the number of users, subscriptions, and models grew, which was the primary reason for moving away from this approach. It was also inaccurate in some observed cases and retention was too short for showback/billing cycles.

### Traces-based showback

Derive usage from the OTEL traces already emitted across the Gateway → maas-api → model backend path.

**Why rejected**: traces carry substantially more context than showback needs and are designed for analyzing complex, cross-service request flows. The usage data needed for showback (tenant, user, model, tokens) is already available directly at the gateway log level, so adopting full distributed tracing for this purpose adds unnecessary overhead and complexity.

### Storage backends for usage data (alternatives to Loki)

* **ClickHouse**: ruled out due to lack of support from Red Hat.
* **PostgreSQL**: ruled out because Perses, used for the showback dashboards, doesn't support it as a data source.

## Security and Privacy Considerations

Access logs used for showback can contain user identity information. Starting RHOAI 3.6, gathering user identity information is opt-in and only happens once enabled in `MaaSTenantConfig`; it remains disabled by default. Access to Loki and to the derived showback data (via the proxy and the dashboards) should be scoped per tenant, consistent with the tenant isolation model established for MaaS, and log retention should follow applicable data retention requirements. Once usage is correlated with API keys (RHOAI 3.6), the same tenant-scoping constraints apply to that correlation.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Log volume growth increases storage cost | Medium | Medium | Apply retention policies tuned to showback/billing cycle needs |
| In RHOAI 3.5, showback depends on a user-managed Loki instance that MaaS doesn't own | Medium | Medium | Feature is Tech Preview and disabled by default in 3.5; document the Loki prerequisite clearly |
| Aggregation/query lag delays showback data availability | Low | Low | Run the OTEL collector and proxy on a schedule/cadence that matches showback reporting needs |

## Stakeholder Impacts

| Group                | Key Contacts | Date | Impacted? |
| -------------------- | ------------ | ---- | --------- |
| Platform/Observability team | TBD    | TBD  | Yes       |

* **Platform/Observability**: owns the EnvoyFilter on the AI Gateway, the OTEL collector and per-user usage proxy deployed by MaaS, and implements the administrator dashboard and the personal usage dashboard that consume the resulting data.

## References

* [RHAISTRAT-1314](https://redhat.atlassian.net/browse/RHAISTRAT-1314)
* [Feature refinement document](https://docs.google.com/document/d/1ivx4vPOAHXSMnSfO8n2mfGB0uGBn8QpJWTtIEcU15y4/edit?usp=sharing)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| |  |  |
