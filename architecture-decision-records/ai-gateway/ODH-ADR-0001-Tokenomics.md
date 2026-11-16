# Open Data Hub - AI Gateway Tokenomics and Budget Service

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | insert date |
| Scope          | |
| Status         | Approved |
| Authors        | Marius Danciu |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This document proposes a new usage metering solution that is part of the RHOAI product. It allows customers to configure and manage costs and attribution and also visualize usage in the UI dashboards.

## Why

Usage metering and budget enforcement is a major feature of AI Gateway and a very frequent topic raised by customers.

## Goals

* Define the architecture for the Budget Service
* Define the Budget Service APIs
* Define end-to-end flows
* Define the high-level integration with third-party solutions like M360 or OpenMeter, but the actual integration details may be subject to different ADRs in later phases
* This is meant to be a usage metering solution as part of MaaS that works out of the box without bringing in new dependencies (such as Kafka) that customers need to install and manage. Customers don't need to perform extra installation steps in order to have usage metering and budget quota enforcement enabled.
* Budget quota enforcement is intentionally kept as a separate concept from rate limiting. Rate limiting is a QoS topic that for now remains managed in MaaSSubscription at control plane level and applied by Praxis at data plane level.
* Budget cost is abstracted away and referred to in this document as Budget Units (BU).

## Non-Goals

* This is not a billing service. We do not intend to manage aspects like:
    * multi currency
    * invoices
    * billing cycles
    * auditability and disputes
* This is not a generic competing solution for M360 or OpenMeter
* Usage for other resource types apart from tokens, e.g. GPUs. However, the design is flexible so we can address other types of resources in the future.
* The Budget Service storage does not represent the SoR for usage metering. The document will address this separation in the next sections.
* Pacing (spend-rate policies such as daily sub-limits derived from a monthly budget). The architecture supports it in the sidecar without changes to Praxis or the object model, but it is out of scope for the initial implementation.
* Multiple tenants per gateway. The initial implementation supports one tenant per gateway. Multi-tenant gateways may be considered in later releases.

## How


### Architecture

```mermaid
flowchart LR
 subgraph redis_system["Redis system"]
        redis[("Redis - counters")]
  end
 subgraph praxis_pod["Praxis Pod"]
    direction TB
        praxis["Praxis<br>(proxy)"]
        sidecar["Budget Sidecar<br>Container<br>(localhost)"]
        otel_collector["OTEL Collector"]
  end
 subgraph control_plane["Control Plane"]
        aigc["AI Gateway Controller"]
 end
 subgraph budget_system["Budget System"]
    direction TB
        postgres[("PostgreSQL")]
        budget_service["Budget Service"]
 end
 subgraph obs["Observability"]
        loki["Loki"]
 end

    aigc -- manages --> praxis_pod
    aigc -- manages --> budget_system
    praxis -- is allowed? --> sidecar
    sidecar -- check balance --> redis
    sidecar -- allow / deny --> praxis
    praxis -. OTEL Logs .-> otel_collector
    budget_service -- SQL --> postgres
    client["Client"] -- inference --> praxis

    otel_collector -- CloudEvents --> budget_service
    otel_collector -- OTEL Log --> loki
    budget_service -. change notifications .-> redis
    sidecar -- fetch config --> budget_service
    budget_service -- metering APIs --> byom["Bring Your Own<br>Metering"]
```

Notes:
- In the Praxis POD, AIGC also brings in the BSC (Budget Sidecar Container)
- Praxis filter remains a PEP (Policy Enforcement Point). All budget calculations happen in the budgeting system (sidecar container and/or budget service)
- Redis/Valkey is used for atomic balance counters and change notification Pub/Sub. The Budget Service publishes change notifications to Redis when configuration objects are updated; sidecars subscribe and fetch the changed objects from the Budget Service REST API. Configuration objects (policies, rates, budget definitions) are cached in the sidecar's memory, not stored in Redis.

#### Inference Flow (budget check)

```mermaid
sequenceDiagram
    participant Client
    participant Praxis
    participant Sidecar
    participant Redis
    participant OTELCollector as OTEL Collector
    participant BudgetService as Budget Service

    Client->>Praxis: inference request
    Praxis->>Sidecar: POST /v1/budget_check<br>(identity, resource)

    Note over Sidecar: Match policy & rate<br>from in-memory cache

    Sidecar->>Redis: GET balance
    Redis-->>Sidecar: current balance

    alt sufficient balance
        Sidecar-->>Praxis: allowed: true
        Praxis->>Praxis: dispatch to model
        Note over Praxis: model responds
        Praxis-->>Client: inference response
        Praxis-->>OTELCollector: OTEL Log (usage)
        OTELCollector->>BudgetService: CloudEvent (usage)
        BudgetService->>Redis: DECRBY balance (actual BU cost)
        OTELCollector-->>OTELCollector: forward to Loki
    else insufficient balance (hard enforcement)
        Sidecar-->>Praxis: allowed: false, reason: budget_exhausted
        Praxis-->>Client: 429 Too Many Requests
    else insufficient balance (soft/overage)
        Note over Sidecar: Apply overage logic
        Sidecar-->>Praxis: allowed: true, over_budget: true
        Note over Praxis: Set x-over-budget header,<br>dispatch to model
        Praxis-->>Client: inference response + x-over-budget
        Praxis-->>OTELCollector: OTEL Log (usage)
        OTELCollector->>BudgetService: CloudEvent (usage)
        BudgetService->>Redis: DECRBY balance (actual BU cost)
        Note over BudgetService: Balance may go negative<br>(overage tracked)
    end
```

#### Configuration Update Flow

```mermaid
sequenceDiagram
    participant Operator
    participant BudgetService as Budget Service
    participant Postgres
    participant Redis
    participant Sidecar1 as Sidecar (pod 1)
    participant Sidecar2 as Sidecar (pod 2)

    Note over Sidecar1,Sidecar2: Subscribed to Redis<br>Pub/Sub {tenant}:changes

    Operator->>BudgetService: PUT /api/v1/rates/{id}<br>(X-Tenant: acme-corp)
    BudgetService->>Postgres: persist updated rate
    Postgres-->>BudgetService: ok
    BudgetService->>Redis: PUBLISH acme-corp:changes<br>{type: "rate", id: "standard"}
    BudgetService-->>Operator: 200 OK

    Redis--)Sidecar1: notification: rate "standard" changed
    Redis--)Sidecar2: notification: rate "standard" changed

    par Sidecar 1 refreshes
        Sidecar1->>BudgetService: GET /api/v1/rates/standard<br>(X-Tenant: acme-corp)
        BudgetService-->>Sidecar1: updated rate
        Note over Sidecar1: Update in-memory cache
    and Sidecar 2 refreshes
        Sidecar2->>BudgetService: GET /api/v1/rates/standard<br>(X-Tenant: acme-corp)
        BudgetService-->>Sidecar2: updated rate
        Note over Sidecar2: Update in-memory cache
    end
```

#### Budget Service ↔ Sidecar Sync

The sidecar fetches configuration objects (policies, rates, budget definitions) from the Budget Service REST API and caches them in memory. Redis is used only for balance counters and lightweight change notifications — not for storing configuration objects.

Each gateway serves exactly one tenant and may have multiple Praxis pods. All sidecars for the same tenant share Redis balance counters, which makes Redis the natural coordination point for multi-replica consistency.

##### How state flows

1. An operator creates or updates a rate, budget, or policy via the Budget Service REST API
2. The Budget Service persists the change to the durable backend (Postgres by default) and publishes a change notification to Redis (e.g., via Pub/Sub: `{tenant}:changes` channel with the object type and ID)
3. Each sidecar subscribes to its tenant's change notifications. On receiving a notification, it fetches the updated object from the Budget Service REST API and refreshes its in-memory cache

On startup, the sidecar calls the Budget Service REST API to fetch all policies, rates, and budgets for its tenant and subscribes to Redis change notifications.

##### What lives where

| Store | Data | Role |
|---|---|---|
| Sidecar (in-memory) | Policies, rates, budget definitions | Cached configuration for admission decisions. Refreshed from the Budget Service API on startup and on change notifications |
| Redis | Balances, change notifications | Atomic counters for the hot path and lightweight Pub/Sub for cache invalidation |
| Budget Service (API) | Policies, rates, budget definitions, consumption | Authoritative source served from the durable backend. Sidecars fetch from here |

##### Consumption data

The sidecar does not report consumption back to the Budget Service. Consumption data reaches the Budget Service through the OTEL pipeline: Praxis emits an OTEL Log after each response, the OTEL Collector transforms it into a CloudEvent, and the Budget Service persists it to the durable backend (see [Consumption Emission](#consumption-emission)).

##### Failure Modes

| Scenario | Behavior |
|---|---|
| Redis unavailable | Sidecar serves decisions from its in-memory cache. Change notifications are missed — the sidecar continues with stale config until Redis recovers. Balance checks and deductions are unavailable — sidecar applies fail-open or fail-closed per operator configuration |
| Budget Service unavailable | Sidecar continues with its in-memory cache and Redis balances. Config changes are delayed until the Budget Service recovers. Consumption data continues to flow via the OTEL pipeline independently |
| Sidecar restarts | Sidecar fetches its tenant's full state from the Budget Service REST API and subscribes to Redis change notifications |


#### Multi-Tenancy

The Budget Service is a single shared instance serving all tenants (soft multi-tenancy). All data is tenant-scoped and every management API request must carry the `X-Tenant` header to identify which tenant's budgets, rates, or policies are being managed.

At the gateway level, each gateway serves exactly one tenant. The sidecar running in the gateway pod inherits this tenant scope — it fetches only that tenant's configuration from the Budget Service API and reads only that tenant's Redis keys. This simplifies the sidecar's local state and avoids cross-tenant data leakage.

| Component | Tenancy model |
|---|---|
| Budget Service (central) | Single instance, soft multi-tenancy. All API calls scoped by `X-Tenant` header |
| Gateway + Sidecar | One tenant per gateway. Sidecar receives only that tenant's budget state |
| Redis | Keys namespaced by tenant ID |
| Postgres | All tables include a tenant ID column |


### Budget object model

The budget domain is modeled with three objects: **Rate**, **Budget**, and **BudgetPolicy**.

#### Rate

Defines BU costs per usage unit for a given resource. Resource matching and cost keys are flexible key-value maps — new usage categories (images, tool calls, video seconds) require no schema change.

| Field | Type | Description |
|---|---|---|
| `name` | string | Unique identifier for this rate table |
| `items` | list | One or more rate items (see below) |

Each **RateItem** contains:

| Field | Type | Description |
|---|---|---|
| `resource` | map | Resource matching keys (e.g. `model`, `provider`, `region`) |
| `costs` | map | BU per usage unit (e.g. `input: 3`, `output: 15`, `cache_read: 1`) |

The resource and cost keys are dynamically set. Admins can provide any key-value pairs here. However, for end-to-end reasons the cost fields emitted by Praxis in the OTEL logs are shown in the [Consumption Emission](#consumption-emission) section.

#### Budget

Pure capacity definition — how much BU and when it resets. No subject, no enforcement, no rates.

| Field | Type | Description |
|---|---|---|
| `name` | string | Unique identifier for this budget |
| `limit` | integer | BU capacity for each period |
| `period.start` | timestamp | UTC anchor for period calculation |
| `period.duration` | duration | Fixed period length (e.g. `P1M`, `P3M`, `P7D`) |
| `renewal` | one of | Reset behavior at period boundary |

**Renewal** variants:

| Variant | Fields | Description |
|---|---|---|
| `drain` | — | Use it or lose it; balance resets to `limit` each period |
| `rollover` | `max_carry` (optional) | Unused BU carry forward, optionally capped |

#### BudgetPolicy

Binds a subject to a budget with a rate table and optional overage behavior.

| Field | Type | Description |
|---|---|---|
| `budget_ref` | string | References a Budget by name |
| `rate_ref` | string | References a Rate by name — controls BU pricing for this subject |
| `subject.kind` | one of | `user`, `group`, `organization`, or `tenant` |
| `subject.name` | string | Identity of the subject |
| `overage` | optional, one of | Behavior when the budget is exhausted (see below) |
| `deduct_overage` | optional, boolean | If true, any overage from the previous period is deducted from the next period's balance at renewal. Defaults to false |

If `overage` is absent, enforcement is **hard** — requests are rejected with 429 when the budget is exceeded.

**Overage** variants:

| Variant | Fields | Description |
|---|---|---|
| `soft` | `rate_ref` (optional) | Continue charging this budget past the limit, optionally at a different rate. Injects `x-over-budget: true` header upstream |
| `group` | `rate_ref` (optional) | Fall back to the group-level budget, optionally at a different rate |
| `organization` | `rate_ref` (optional) | Fall back to the organization-level budget, optionally at a different rate |
| `tenant` | `rate_ref` (optional) | Fall back to the tenant-level budget, optionally at a different rate |

**Ambiguity warning:** A user can belong to multiple groups and organizations. When overage is set to `group` or `organization`, the system may find multiple matching policies at that level. Since group and organization membership is only known at request time (via SelfTokenReview), this ambiguity cannot be validated at policy creation time. At request time, if multiple policies match, the sidecar applies the most restrictive one and emits a warning event so the operator can resolve it by creating a direct user policy.


#### Budget Units (BU) as the Accounting Abstraction

Budget Units are an important simplification from actual currency and reinforce the claim that this is not a billing system. Customers can attribute actual currency amounts per BU if necessary, but again this is not the SoR.

A budget becomes straightforward:

- Budget limit: 100,000 BU per 30-day period
- Each model has a BU rate table (potentially different per tenant/group)
- Consumption = tokens used * BU rate for that token type and model
- Remaining = limit - total consumption in BU


### Separation of Concerns: Rate Limiting vs Budget Enforcement

Rate limiting and budget enforcement answer fundamentally different questions:

| Concern | Question | Protects | Example |
|---|---|---|---|
| Rate limiting | Too fast? | Infrastructure from overload | 100 req/s max |
| Budget enforcement | Too much total? | Cost control | 1000 USD/month cap |


Rate limiting belongs in the proxy regardless of tokenomics — it protects backend model servers from burst traffic. Budget enforcement is a business concern that belongs in the external budget system.

OpenMeter, for comparison, handles entitlements (budget/quota enforcement) but not rate limiting. These are separate systems in practice.


### BU Cost Computation

The BU cost for a request is computed by matching usage keys against the rate table:

```
cost_bu = sum(usage[key] * rate[key]) for all keys in usage
```

Token usage is normalized into five types:

| Key | Meaning |
|---|---|
| `input` (I) | All input tokens, including cache reads and writes |
| `output` (O) | All output tokens, including reasoning |
| `cache_read` (R) | Input subset served from cache |
| `cache_write` (W) | Input subset written to cache |
| `reasoning` (Q) | Output subset used for reasoning |

The full consumption formula:

```
cost_bu = (I - R - W) * bu_input + R * bu_cache_read + W * bu_cache_write
        + (O - Q) * bu_output + Q * bu_reasoning
```

Different subjects can have different rate tables assigned via their BudgetPolicy `rate_ref`. The same model costs different subjects different BU amounts. This makes the rate table a unified pricing and discount mechanism — no secondary discount logic is needed.


### Sidecar Architecture

#### The Problem

If Praxis calls the Budget Service synchronously on every request, inference latency depends on that external call. If the Budget Service is slow or unavailable, Praxis blocks or fails.

#### Sidecar as the Budget System's Local Agent

Each Praxis replica calls a localhost sidecar container provided and managed by the budget system. The sidecar runs in the same pod as Praxis.

From Praxis's perspective, the contract is simple: call `localhost:port` with who and what model. Get back allow or deny. After the model responds, call again with the actual token usage for deduction. Whether the sidecar answers from local state or calls the Budget Service is invisible to Praxis.

#### Sidecar Responsibilities

The budget system owns the sidecar and controls all caching and sync behavior:

| Responsibility | Description |
|---|---|
| Allow locally | Match policies and rates from in-memory cache, check balances in Redis. Tenants with plenty of budget remaining get fast approvals |
| Deny locally | When the balance is clearly exhausted |
| Fail-open / fail-closed | Operator-configured behavior when Redis is unreachable |
| Cache refresh | Fetch config from Budget Service API on startup, subscribe to Redis Pub/Sub for change notifications, fetch updated objects on demand |
| No consumption reporting | Consumption data flows to the Budget Service via the OTEL pipeline (Praxis → OTEL Collector → CloudEvent), not from the sidecar |

#### What This Solves

**Fleet coherence.** All sidecars for the same tenant share Redis balance counters and receive the same Pub/Sub change notifications. Multiple Praxis replicas with their own sidecars are automatically coordinated — balances are shared atomically via Redis, and config changes propagate to all sidecars via the notification + fetch pattern.

**Availability.** The sidecar depends on Redis for balances and on the Budget Service API for config. A Budget Service outage blocks config refreshes and management API changes but does not affect admission decisions — the sidecar continues serving from its in-memory cache and Redis balances.

**Latency.** Localhost calls are sub-millisecond. The admission path stays fast regardless of the Budget Service's latency.

#### What Praxis Needs to Know

| Concern | Praxis responsibility |
|---|---|
| Who | Tenant identity, resolved from headers or authentication |
| What | Model and capabilities |
| Usage emission | After the model responds, emit an OTEL Log with actual token counts. The Budget Service handles balance deduction via the CloudEvent pipeline |
| Enforcement | Call the sidecar, honor the allow/deny decision, apply hard (429) or soft (`x-over-budget: true` header) behavior |

Everything else — BU rates, balances, policies, discounts, pacing, fleet coordination, caching strategy — lives in the budget system and its sidecar.


### Identity Resolution

The `identity` object passed in the budget check call contains the full identity context needed for policy matching: user, groups, organization, and tenant. Praxis is responsible for resolving this identity before calling the sidecar.

| Field | Source | Description |
|---|---|---|
| `user` | Authentication token / headers | The authenticated user identity |
| `groups` | SelfTokenReview or OIDC token claims | Groups the user belongs to. Only known at request time — not available at policy creation time |
| `organization` | SelfTokenReview or OIDC token claims | Organization the user belongs to. Only known at request time |
| `tenant` | Gateway configuration | The tenant served by this gateway. Fixed per gateway — not resolved per request |

Identity resolution happens entirely within Praxis, before the budget check call. The sidecar receives the resolved identity and uses it to match policies — it does not perform any authentication or identity lookup itself.

The fact that group and organization membership is only known at request time has implications for policy validation and ambiguity resolution (see [Validation Rules](#validation-rules)).


### Admission and Deduction Flow

The budget flow has two phases: check before dispatch, deduct after completion.

#### Phase 1: Check (before dispatch)

Praxis sends the sidecar: "User A, model X."

The sidecar:
1. Looks up User A's budget policy from its in-memory cache
2. Reads the current balance from Redis
3. If balance > 0: **allow**
4. If balance <= 0: **deny** (hard enforcement) or **allow with `over_budget: true`** (soft/overage logic)

The check is a read operation — no balance modification occurs at this stage.

#### Phase 2: Deduct (after model responds)

After the model responds, Praxis emits an OTEL Log with the actual token usage. The OTEL Collector transforms it into a CloudEvent and delivers it to the Budget Service.

The Budget Service:
1. Receives the CloudEvent with identity, resource, and actual token usage
2. Matches the resource (model) to the rate table, computes actual BU cost
3. Atomically decrements the Redis balance by the actual cost
4. Persists the consumption record to Postgres
5. If the balance goes negative, the overage is tracked in the consumption ledger

The sidecar is not involved in the deduction — the Budget Service owns all balance writes to Redis.

#### Why No Reservation System

This design intentionally avoids a reservation-based model (pre-decrementing an estimated cost before dispatch). The `max_completion_tokens` parameter is unreliable — clients may omit it, set it arbitrarily high, or the actual usage may diverge significantly from the estimate. A reservation system built on unreliable estimates introduces complexity (holds, settlements, releases, orphaned hold cleanup) without delivering accurate budget enforcement.

Instead, the system uses a simpler check-then-deduct approach: the sidecar checks the current balance (read-only), and the Budget Service deducts the actual cost after the response. Overage is detected and tracked post-request rather than prevented pre-request.

When the sidecar detects that a budget is over (balance <= 0 at check time), Praxis can strip the output stream back to the client and inject the `x-over-budget` header in the response, signaling to the caller that the budget has been exceeded.

#### Overage Tracking and Period Carry-Forward

When a deduction drives the Redis balance negative, the sidecar continues to operate — subsequent budget checks see a negative balance and apply hard or soft enforcement accordingly. The Budget Service tracks the cumulative overage for each budget from the consumption ledger in Postgres.

At period renewal, the Budget Service resets the Redis balance to the budget limit. If the BudgetPolicy has `deduct_overage` enabled, the Budget Service subtracts the previous period's overage from the new balance:

```
new_balance = limit - previous_period_overage   (if deduct_overage is true)
new_balance = limit                              (if deduct_overage is false or absent)
```

This ensures that overspend in one period is not silently forgiven — it reduces the next period's capacity. The `deduct_overage` field is optional on BudgetPolicy and defaults to false (overage is not carried forward).

#### Edge Cases

| Scenario | Behavior |
|---|---|
| Redis unavailable during check | Sidecar applies fail-open or fail-closed per operator configuration |
| Budget Service unavailable when CloudEvent arrives | OTEL Collector retries delivery. Redis balance update is delayed until the Budget Service processes the event |
| Concurrent requests pass check before deductions land | Expected behavior — the deduction is asynchronous (via OTEL pipeline → Budget Service → Redis). Multiple requests may be allowed before any deduction occurs. Overage is tracked and corrected at the next check or period renewal |
| Budget Service crashes after Postgres write, before Redis update | On recovery, the Budget Service reconciles the Redis balance from the Postgres consumption ledger |


### Storage Architecture: Redis + Postgres

Redis and Postgres serve different roles in the budget system:

| Layer | Store | Purpose |
|---|---|---|
| Sidecar (in-memory) | Memory | Cached policies, rates, and budget definitions. Fetched from the Budget Service API, refreshed via Redis change notifications |
| Hot path (shared) | Redis | Atomic balance counters and Pub/Sub change notifications |
| Durable (default backend) | Postgres | Authoritative ledger, budget/rate/policy definitions, consumption history, reconciliation |

The sidecar caches configuration objects (policies, rates, budget definitions) in memory, fetched from the Budget Service REST API. Redis handles only what requires shared atomic state — balance counters — plus lightweight Pub/Sub notifications for cache invalidation.

Postgres is the default durable backend, but it is not the only option. When the Budget Service integrates with an external platform such as M360 or OpenMeter (see [Bring Your Own Metering](#bring-your-own-metering)), the external system can serve as the durable store instead of — or alongside — Postgres. In that scenario, the sidecars are unaffected because they fetch configuration from the Budget Service API (not from Postgres directly) and use Redis only for balances.

Consumption data reaches the Budget Service via the OTEL pipeline, not from the sidecar directly. See the [Storage Schema](#storage-schema-high-level) section for details on the Postgres and Redis schemas.

Redis operations:

- **Check (sidecar, before dispatch):** `GET {tenant}:balance:{budget_id}` — read the current balance to make an admission decision. The sidecar never writes to balance keys.
- **Deduct (Budget Service, on CloudEvent receipt):** `DECRBY {tenant}:balance:{budget_id} <actual_bu_cost>` — atomically decrement by the actual BU cost. The balance may go negative (overage).
- **Reset (Budget Service, on period renewal):** `SET {tenant}:balance:{budget_id} <new_balance>` — set balance to `limit` (or `limit - previous_overage` if `deduct_overage` is enabled on the policy).


### Pacing

Budget systems are accumulators — they track total spend against a cap but have no opinion about the shape of consumption over time. A tenant with a 30-day budget and a 100 req/s rate limit could exhaust their entire monthly budget in minutes if each request is expensive.

Pacing sits between rate limiting and budget enforcement:

| Concern | Question | Example |
|---|---|---|
| Rate limiting | Too fast? | 100 req/s max |
| **Pacing** | **Too much too soon?** | **No more than 10,000 BU/day from a 100,000 BU/month budget** |
| Budget enforcement | Too much total? | 100,000 BU/month cap |

Pacing is a policy decision that varies by use case: batch processing may want to burst, customer-facing apps need steady availability. The sidecar is a natural place to enforce spend-rate policies — the Budget Service can configure the sidecar with daily or hourly sub-limits derived from the period budget. Pacing becomes a sidecar-local policy, not a Praxis concern.

Pacing is out of scope for the initial implementation, but the architecture supports it without changes to Praxis or the object model.


### Validation Rules

#### At policy creation time

| Rule | Description |
|---|---|
| One subject, one active policy | Reject duplicate policies for the same subject |
| `budget_ref` must resolve | Must reference an existing Budget |
| `rate_ref` must resolve | Must reference an existing Rate |
| Overage `rate_ref` must resolve | If present, must reference an existing Rate |
| Overage level constraints | Overage level must be higher than or equal to the subject kind. `soft` is valid for any subject kind. `group` is valid for `user`. `organization` is valid for `user` or `group`. `tenant` is valid for `user`, `group`, or `organization` |

#### At request time

| Rule | Description |
|---|---|
| Most-specific wins | A user with a direct policy takes precedence over any group, organization, or tenant policy |
| Ambiguous group/organization membership | A user in multiple groups or organizations that have policies results in the most restrictive policy being applied. A warning event is emitted so the operator can resolve the ambiguity by creating a direct user policy. Group and organization membership is only known at request time (i.e. via SelfTokenReview or from the OIDC token claims) and cannot be validated at policy creation time |


### API: REST

The Budget Service exposes a REST API for managing rates, budgets, and budget policies. All endpoints require the `X-Tenant` header.

> **Note:** The exact request and response payloads will be defined in a separate API specification document.

#### Rates

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/rates` | List all rates |
| `GET` | `/api/v1/rates/{id}` | Get a rate by name |
| `POST` | `/api/v1/rates` | Create a rate |
| `PUT` | `/api/v1/rates/{id}` | Update a rate |
| `DELETE` | `/api/v1/rates/{id}` | Delete a rate |

#### Budgets

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/budgets` | List all budgets |
| `GET` | `/api/v1/budgets/{id}` | Get a budget by name |
| `POST` | `/api/v1/budgets` | Create a budget |
| `PUT` | `/api/v1/budgets/{id}` | Update a budget |
| `DELETE` | `/api/v1/budgets/{id}` | Delete a budget |

#### Budget Policies

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/budget_policies` | List all policies |
| `GET` | `/api/v1/budget_policies/{id}` | Get a policy by ID |
| `POST` | `/api/v1/budget_policies` | Create a policy |
| `PUT` | `/api/v1/budget_policies/{id}` | Update a policy |
| `DELETE` | `/api/v1/budget_policies/{id}` | Delete a policy |


All these objects are scoped per tenant. A tenant is identified in the request via the `X-Tenant` header. These objects are scoped per tenant and not per AI Gateway organization, since the tenant admin manages costs, not the organization admin. At this point in time this is a choice for simplification reasons that may be relaxed later on. However, as per the previous sections, budget policies can be created per organization.

A tenant admin automatically has the RBAC permissions to manage these objects. From an RBAC point of view we need three separate virtual resources:
- rates
- budgets
- budget-policy 
The api group for these resources is `budget.aigateway.opendatahub.io`


### API: Sidecar

The sidecar exposes a local endpoint for Praxis to check budget availability. This is a hot-path call — localhost only, never exposed outside the pod. The sidecar is read-only for Redis balances; the Budget Service handles all balance writes via the CloudEvent pipeline.

#### Check Budget

```
POST /v1/budget_check
```

```json
{
  "identity": {
    "user": "alice",
    "groups": ["engineering", "ml-team"],
    "organization": "research",
    "tenant": "acme-corp"
  },
  "resource": {
    "model": "claude-sonnet-5-5",
    "provider": "anthropic"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `identity.user` | string | User identity (resolved by Praxis from auth/headers) |
| `identity.groups` | list of strings | Groups the user belongs to (from SelfTokenReview or OIDC claims) |
| `identity.organization` | string | Organization the user belongs to |
| `identity.tenant` | string | Tenant identity |
| `resource.model` | string | Model identifier |
| `resource.provider` | string | Model provider |

The check is a read-only operation — the sidecar reads the current Redis balance to make an admission decision but does not modify it. No tokens are estimated or reserved. The actual deduction happens after the model responds (see [Deduct](#deduct)).

Response (modeled after [OpenMeter's entitlement access API](https://openmeter.io/docs/api)):

```json
{
  "allowed": true,
  "over_budget": false,
  "budget_ref": "team-alpha-monthly",
  "balance": 85000,
  "usage": 15000,
  "overage": 0
}
```

| Field | Type | Description |
|---|---|---|
| `allowed` | boolean | Whether the request should proceed (equivalent to OpenMeter's `hasAccess`) |
| `over_budget` | boolean | Whether the budget is exceeded — Praxis uses this to set the `x-over-budget: true` header upstream |
| `budget_ref` | string | Which budget was matched |
| `balance` | integer | Current BU balance |
| `usage` | integer | Total BU consumed in the current period |
| `overage` | integer | BU consumed past the budget limit (0 if within budget) |

When `allowed` is `false`, the response includes a `reason` field:

```json
{
  "allowed": false,
  "over_budget": true,
  "budget_ref": "team-alpha-monthly",
  "balance": 0,
  "usage": 100000,
  "overage": 0,
  "reason": "budget_exhausted"
}
```

> **Note:** The exact payloads may change and will be defined in a separate API specification document.

### Consumption Emission

After a model responds, Praxis emits an OTEL Log record containing the resource identification and actual token usage. The OTEL Collector forwards this as a CloudEvent to the Budget Service. On receipt, the Budget Service computes the actual BU cost, atomically decrements the Redis balance, and persists the consumption record to Postgres. This is the sole path for balance deductions — the sidecar never writes to Redis balance keys.

The OTEL Log record carries the following attributes:

```json
{
  "Timestamp": "2026-11-16T14:32:05.123Z",
  "SeverityText": "INFO",
  "Body": "inference_usage",
  "Attributes": {
    "request_id": "req-abc123",
    "identity.user": "alice",
    "identity.groups": ["engineering", "ml-team"],
    "identity.organization": "research",
    "identity.tenant": "acme-corp",
    "resource.model": "claude-sonnet-5-5",
    "resource.provider": "anthropic",
    "resource.region": "us-east-1",
    "usage.input_tokens": 312,
    "usage.output_tokens": 87,
    "usage.cache_read_tokens": 20,
    "usage.cache_write_tokens": 5,
    "usage.reasoning_tokens": 30,
    "usage.total_tokens": 454
  },
  "Resource": {
    "service.name": "praxis",
    "service.instance.id": "praxis-pod-abc-0",
    "k8s.namespace.name": "ai-gateway"
  }
}
```

| Field | Description |
|---|---|
| `request_id` | Unique request identifier for correlation and deduplication |
| `identity.*` | Full identity context of the request |
| `resource.model` | Model identifier — matches rate table resource keys |
| `resource.provider` | Model provider |
| `resource.region` | Deployment region (if applicable) |
| `usage.input_tokens` | Total input tokens (I) |
| `usage.output_tokens` | Total output tokens (O) |
| `usage.cache_read_tokens` | Input tokens served from cache (R) |
| `usage.cache_write_tokens` | Input tokens written to cache (W) |
| `usage.reasoning_tokens` | Output tokens used for reasoning (Q) |
| `usage.total_tokens` | Sum of all tokens |

The `resource.*` fields are matched against rate tables to compute BU cost. The `usage.*` fields provide the actual token counts for the consumption formula:

```
cost_bu = (I - R - W) * bu_input + R * bu_cache_read + W * bu_cache_write
        + (O - Q) * bu_output + Q * bu_reasoning
```

This OTEL Log record is also forwarded to Loki for observability.

#### CloudEvent

The OTEL Collector transforms the Log record into a CloudEvent and sends it to the Budget Service via HTTP. The CloudEvent follows the [CloudEvents v1.0 specification](https://cloudevents.io/):

```json
{
  "specversion": "1.0",
  "id": "evt-7f3a2b1c-4d5e-6f78-9a0b-1c2d3e4f5678",
  "source": "praxis/praxis-pod-abc-0",
  "type": "io.opendatahub.aigateway.usage.v1",
  "time": "2026-11-16T14:32:05.123Z",
  "datacontenttype": "application/json",
  "data": {
    "request_id": "req-abc123",
    "identity": {
      "user": "alice",
      "groups": ["engineering", "ml-team"],
      "organization": "research",
      "tenant": "acme-corp"
    },
    "resource": {
      "model": "claude-sonnet-5-5",
      "provider": "anthropic",
      "region": "us-east-1"
    },
    "usage": {
      "input_tokens": 312,
      "output_tokens": 87,
      "cache_read_tokens": 20,
      "cache_write_tokens": 5,
      "reasoning_tokens": 30,
      "total_tokens": 454
    }
  }
}
```

| Field | Description |
|---|---|
| `specversion` | CloudEvents specification version |
| `id` | Unique event identifier (for deduplication) |
| `source` | Originating Praxis instance |
| `type` | Event type — `io.opendatahub.aigateway.usage.v1` |
| `time` | Timestamp of the inference completion |
| `data.request_id` | Unique request identifier for correlation and deduplication |
| `data.identity` | Full identity context from the OTEL Log |
| `data.resource` | Resource keys matching rate tables |
| `data.usage` | Actual token counts for BU cost computation |

The Budget Service persists the CloudEvent data to Postgres, updates the consumption ledger, and exposes it via the metering APIs for external systems (Bring Your Own Metering). The consumption records are also used to compute cumulative overage per budget for period carry-forward (see [Overage Tracking and Period Carry-Forward](#overage-tracking-and-period-carry-forward)).

### Storage Schema (high level)

#### Postgres

Postgres is the default durable backend and the authoritative source of truth in standalone deployments. When the Budget Service integrates with an external metering platform, Postgres may be replaced by the external system's storage — the Budget Service treats the durable backend as a pluggable layer behind Redis.

All tables include a `tenant_id` column for multi-tenancy.

| Table | Description |
|---|---|
| `rates` | Rate definitions — name, and the list of rate items (resource keys + cost keys). Stored as JSONB for flexibility |
| `budgets` | Budget definitions — name, BU limit, period (start + duration), renewal type (drain/rollover + max_carry) |
| `budget_policies` | Policy bindings — references a budget and a rate, subject (kind + name), overage configuration |
| `consumption` | Append-only ledger of consumption records — request ID, subject, budget ref, BU consumed, token breakdown, timestamp |
| `period_balances` | Current period state per budget — current balance, period start, period end, rollover carry, overage amount. Updated on deduction and period reset |

The `consumption` table is the durable record that feeds reporting, reconciliation, overage tracking, and the metering APIs. It is append-only — rows are never updated or deleted. The Budget Service computes the cumulative overage per budget from this ledger for period carry-forward.

**Important:** We use a hybrid approach where we keep JSONB structures in Postgres. Dynamic fields like rate keys or resource keys are represented as JSONB (natively supported by Postgres). Reference fields, `tenant_id`, timestamps, `limit`, etc. are represented as separate columns.


#### Redis/Valkey

Redis handles atomic shared state and change notifications. Configuration objects (policies, rates, budget definitions) are not stored in Redis — they are cached in sidecar memory and served by the Budget Service API. All keys are namespaced by tenant ID.

| Key pattern | Type | Description |
|---|---|---|
| `{tenant}:balance:{budget_id}` | integer | Current BU balance for a budget. Decremented atomically on deduction after each response. May go negative (overage) |
| `{tenant}:changes` | Pub/Sub channel | Change notifications published by the Budget Service when a policy, rate, or budget is created, updated, or deleted. Payload contains the object type and ID. Sidecars subscribe to this channel and fetch the changed object from the Budget Service API |

The balance key is the critical path — the sidecar reads it for admission checks and atomically decrements it on deduction.

**Note:** The low-level schema details are intentionally omitted from this document as they tend to change as the design and implementation evolve, risking this information becoming outdated. For this reason we aim to cover only the high-level aspects here.


### Bring Your Own Metering

The Budget Service is designed as a self-contained solution that works out of the box. However, customers who already operate an external metering or billing platform — such as OpenMeter, Monetize360, or a custom solution — can integrate it with the Budget Service.

#### Integration boundary

All external metering integration happens exclusively at the Budget Service level. The rest of the stack — Praxis, the sidecar, Redis, the OTEL pipeline — remains completely unchanged and unaware of any external backend. This is a deliberate architectural choice:

- **Praxis** continues to call the sidecar for admission decisions. It does not know whether consumption data is forwarded elsewhere.
- **The sidecar** continues to fetch config from the Budget Service API and use Redis for balances. Its sync protocol is unchanged.
- **The OTEL pipeline** continues to emit logs and CloudEvents to the Budget Service. No additional collectors or exporters are needed.

The Budget Service acts as the single integration point, translating its internal consumption data into the format expected by the external system.

#### Integration patterns

| Pattern | Description |
|---|---|
| Push (webhook/event) | The Budget Service forwards consumption events to an external system's ingestion endpoint (e.g. OpenMeter's event API or M360's usage API) as they are persisted |
| Pull (metering API) | The external system queries the Budget Service's metering APIs to retrieve consumption data on its own schedule |
| Backend replacement | The external system replaces Postgres as the durable store. The Budget Service writes budget objects and consumption records to the external system instead of Postgres. Redis remains the operational store — Praxis, sidecars, and the OTEL pipeline are unaffected |

#### What the external system receives

The Budget Service can expose consumption data: attempt ID, identity (user, groups, organization, tenant), resource keys (model, provider, region), token breakdown, BU cost, and timestamp. The external system can then apply its own pricing, aggregation, and billing logic.

#### What the external system does not control

Budget enforcement decisions (allow/deny) remain with the Budget Service and its sidecars. The external system is a consumer of usage data, not an authority on admission. If a customer wants the external system to influence budget limits, this is done by updating budgets and rates via the Budget Service's management API — not by bypassing the sidecar.

> **Note:** The specific integration details for each external platform (API mappings, authentication, retry policies) will be addressed in separate ADRs as those integrations are prioritized.


### Observability

The budget system produces observability data at multiple levels:

#### OTEL Logs (via Praxis)

Praxis emits an OTEL Log record after each inference response containing full identity, resource, and token usage. These logs are forwarded to Loki for observability and to the Budget Service (as CloudEvents) for durable persistence. See [Consumption Emission](#consumption-emission) for the exact log structure.

#### Metrics

The sidecar and Budget Service expose Prometheus metrics for monitoring:

| Metric | Source | Description |
|---|---|---|
| Budget utilization | Sidecar | Current balance as a percentage of the budget limit, per budget |
| Admission decisions | Sidecar | Counter of allow/deny decisions, labeled by tenant, subject, and reason (sufficient, soft overage, hard deny) |
| Deduction latency | Budget Service | Histogram of Redis balance deduction latency on CloudEvent processing |
| Overage events | Sidecar | Counter of requests that triggered overage logic |
| Overage amount | Budget Service | Cumulative BU overage per budget in the current period |
| Config refresh latency | Sidecar | Latency of config fetches from the Budget Service API |
| CloudEvent ingestion | Budget Service | Counter and latency of CloudEvent processing |

#### Alerts

Operators can configure alerts based on these metrics. Recommended alert conditions:

| Condition | Severity | Description |
|---|---|---|
| Budget utilization > 80% | Warning | Tenant approaching budget limit |
| Budget utilization > 95% | Critical | Tenant near exhaustion |
| Overage exceeds threshold | Warning | Budget overage growing — may indicate runaway consumption or misconfigured rates |
| Config refresh failures | Warning | Sidecar cannot reach Budget Service API — serving stale config |
| Ambiguous policy match | Warning | Multiple group/org policies matched for a user — operator should create a direct user policy |

#### UI Dashboards

The RHOAI UI can query the Budget Service metering APIs to display:

- Consumption over time (BU spent per period, per tenant/user/group)
- Budget utilization (remaining vs limit, with trend projection)
- Top consumers (users or models by BU consumption)
- Overage history (when and how often budgets were exceeded)


## Open Questions

| # | Question | Context |
|---|---|---|
| 1 | **Fail-open or fail-closed default?** | When Redis is unreachable, should the sidecar allow requests (fail-open) or reject them (fail-closed)? This is operator-configurable, but the default affects out-of-the-box behavior. Fail-open risks overspend; fail-closed risks blocking legitimate requests |
| 2 | **Maximum negative balance tolerance** | When deductions drive a balance negative, how far negative is acceptable before the system takes corrective action? Concurrent requests between check and deduction can cause overshoot. Unbounded negative balances could indicate a problem |
| 3 | **Redis persistence strategy** | Should Redis use AOF, RDB snapshots, or both? This affects how much balance data is lost on a Redis restart and how quickly it can be rebuilt from Postgres |
| 4 | **Pub/Sub delivery guarantees** | Redis Pub/Sub is fire-and-forget — if a sidecar misses a notification (e.g., during a brief disconnect), it will serve stale config. Should the sidecar also poll the Budget Service API periodically as a fallback? If so, at what interval? |
| 5 | **Budget period boundary handling** | When a budget period resets (e.g., monthly rollover), who resets the Redis balance counter — the Budget Service or the sidecar? The Budget Service is the natural owner since it has the authoritative period state in Postgres |


## Alternatives

### Adopt OpenMeter or another open-source metering solution

Solutions like OpenMeter provide entitlement management, usage metering, and billing integrations. However, adopting a third-party solution introduces several concerns:

- **Dependency footprint.** OpenMeter requires Kafka, ClickHouse, and its own control plane. Customers would need to install and maintain these additional components. The goal of this design is an end-to-end solution that works out of the box without requiring customers to manage extra infrastructure.
- **Feature surface.** Third-party solutions come with their own UIs, APIs, and features (billing integrations, plan management, invoicing) that overlap with or conflict with the RHOAI product experience. Hiding or removing these features requires forking the upstream project, which leads to ongoing maintenance debt as the upstream evolves.
- **Integration cost.** Adapting an external solution to fit the Praxis sidecar architecture, the OTEL-based consumption pipeline, and the multi-tenancy model would require significant glue code — potentially more effort than building the focused subset of functionality we actually need.

OpenMeter's entitlement model (feature-based access with boolean/metered/static grants) informed parts of this design, particularly the budget check API response format. The Budget Service can expose metering data via APIs for customers who want to integrate with OpenMeter or similar solutions downstream (Bring Your Own Metering), without requiring them as runtime dependencies.

### Synchronous budget checks against the central Budget Service

Instead of the sidecar architecture, Praxis could call the Budget Service directly on every request. This was rejected because:

- **Latency.** Every inference request would depend on a network round trip to the Budget Service. Under load or during Budget Service degradation, inference latency increases or requests fail.
- **Availability coupling.** A Budget Service outage would block all inference requests, making budget enforcement a single point of failure for the entire gateway.
- **Fleet coordination.** With multiple Praxis replicas, each calling the Budget Service independently, the central service becomes a contention point for atomic balance operations.

The sidecar model decouples the admission path from the central service. Localhost calls are sub-millisecond, and the sidecar continues serving decisions from local state during outages.

### Redis-only or Postgres-only storage

- **Redis-only:** Provides the required latency but lacks durability guarantees. A Redis restart or eviction loses balance state and consumption history. Rebuilding from scratch is not feasible without an authoritative ledger.
- **Postgres-only:** Provides durability but cannot meet the sub-millisecond latency requirement for the admission hot path. Even with connection pooling, atomic check-and-decrement under high concurrency would bottleneck on row-level locks.

The hybrid approach uses each store for what it does best: Redis for the latency-sensitive hot path and Postgres as the durable source of truth.


## Security and Privacy Considerations

### In-Cluster Communication

All communication between components happens exclusively within the Kubernetes cluster network over TLS:

- **Sidecar → Budget Service** (REST API calls to fetch config) — in-cluster only, never exposed publicly
- **Sidecar → Redis** (balance counters, Pub/Sub subscriptions) — in-cluster only
- **Budget Service → Redis** (balance updates, change notifications) — in-cluster only

The Praxis-to-sidecar communication is localhost only (same pod), so it does not traverse the network at all and requires no authentication. Both containers share the same pod security boundary — if one is compromised, the other is already accessible.

### Service-to-Service Authentication

The sidecar's REST API calls to the Budget Service (for fetching policies, rates, and budgets) are authenticated using Kubernetes ServiceAccounts and projected ServiceAccount tokens (bound tokens with audience and expiry). The sidecar presents its ServiceAccount token, and the Budget Service validates it against the Kubernetes TokenReview API.

Authorization is enforced via RBAC. A dedicated virtual resource `s2s` in the `budget.aigateway.opendatahub.io` API group controls which ServiceAccounts are allowed to fetch budget state from the Budget Service. The same API group also defines three virtual resources (`rates`, `budgets`, `budget-policy`) for management operations.

The AI Gateway Controller is responsible for creating the appropriate RoleBindings when it provisions a gateway and its sidecar.

### Tenant Isolation

All management API requests require the `X-Tenant` header. The Budget Service validates that the caller has RBAC permissions for the specified tenant's resources. A missing or unauthorized `X-Tenant` header results in a 403 response.

At the data plane level, each sidecar fetches only its own tenant's config from the Budget Service API (the `X-Tenant` header scopes the response) and reads only its tenant's Redis keys (see [Multi-Tenancy](#multi-tenancy)). Tenant isolation in Redis is enforced by key naming convention (`{tenant}:*`) and Pub/Sub channel naming (`{tenant}:changes`). Redis ACLs or separate Redis instances per tenant can provide stronger isolation if required.

### PII in Identity Fields

The `identity` object (user, groups, organization, tenant) flows through the OTEL Log, CloudEvent, and Postgres consumption records. These fields may contain personally identifiable information depending on the identity provider configuration. Operators should be aware that consumption records in Postgres retain identity context for reconciliation and reporting purposes. Data retention policies for the `consumption` table should align with the organization's privacy requirements.

### Redis Data Security

Redis holds balance counters and Pub/Sub change notifications. It does not store configuration objects, consumption records, or identity information. Redis should be configured with TLS for in-transit encryption and access restricted to the sidecar and Budget Service via network policies.

## Implementation notes

We recommend that the BudgetService and the BudgetSidecar to be implemented in Rust not Go. Tokenomics support is a critical part of the system so we need to optimize system resources in terms of memory foot print an CPU. This also avoids potential CPU spikes caused by a garbage collector.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Redis data loss** — Redis restart or crash loses balance counters | Medium | High | Redis persistence (AOF/RDB). Budget Service reconciliation rebuilds balances from Postgres consumption records |
| **Redis balance corruption** — Redis balance becomes incorrect due to a partial failure (e.g., Budget Service crashes after Postgres write but before Redis DECRBY) | Low | Medium | Budget Service can rebuild Redis balances from Postgres consumption records on demand |
| **Missed Pub/Sub notifications** — Sidecar misses a Redis change notification and serves stale config (wrong rates or policies) | Low | Medium | Optional periodic polling of the Budget Service API as a fallback. Sidecar restart triggers a full config fetch |
| **Concurrent request overshoot** — Multiple requests pass the balance check before deductions land via the CloudEvent pipeline, causing the balance to go significantly negative | Medium | Low | Expected behavior — deductions are asynchronous. Negative balances block subsequent hard admissions. Operators set budgets with margin. The `deduct_overage` policy option ensures overshoot is recovered in the next period |
| **Sidecar memory pressure** — Tenants with many policies, rates, or budget definitions cause high memory usage in the sidecar | Low | Low | Config objects are small (KV maps). Monitor sidecar memory usage. If needed, limit the number of objects per tenant |
| **Budget Service as single point for config** — Budget Service outage prevents config refresh for all sidecars | Medium | Low | Sidecars continue with in-memory cached config. Only new config changes are delayed. Admission decisions and Redis balances are unaffected |


## Stakeholder Impacts

| Group                         | Key Contacts         | Date       | Impacted? |
| ----------------------------- | -------------------- | ---------- | --------- |
| Tokenmics                     |   Marius Danciu      |            |     Y     |
| Praxis                        |   Shane Utt          |            |     Y     |
| Dashboard                     |   Andrew Ballantine  |            |     Y     |
| MaaS                          |   Jamie Lang         |            |    TBD    |



## References


## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
|                               |            |       |
