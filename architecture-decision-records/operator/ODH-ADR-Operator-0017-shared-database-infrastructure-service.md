# Open Data Hub - Shared Database Infrastructure Service


|                |                                                                                                                                                                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Date           | August 6, 2026 (revised September 21, 2026)                                                                                                                                                                                                                        |
| Scope          | Operator / Platform Infrastructure                                                                                                                                                                                                                                 |
| Status         | Draft                                                                                                                                                                                                                                                              |
| Authors        | [Luca Burgazzoli](@lburgazzoli)                                                                                                                                                                                                                                    |
| Supersedes     | N/A                                                                                                                                                                                                                                                                |
| Superseded by: | N/A                                                                                                                                                                                                                                                                |
| Tickets        | [RHAIRFE-1141](https://redhat.atlassian.net/browse/RHAIRFE-1141), [RHAISTRAT-176](https://redhat.atlassian.net/browse/RHAISTRAT-176), [RHAISTRAT-2656](https://redhat.atlassian.net/browse/RHAISTRAT-2656), [RHOAIENG-94723](https://redhat.atlassian.net/browse/RHOAIENG-94723) |
| Other docs:    | [POC: opendatahub-db-operator](https://github.com/lburgazzoli/opendatahub-module-operator/tree/db-service/modules/opendatahub-db-operator), [Standardized Data Backbone proposal](https://docs.google.com/document/d/1oxlF_pwZG3sPkaogFBV_f1T9UOjW-uZsDswj5BDUMiQ) |


## What

Introduce a shared database infrastructure service that gives OpenShift AI components a declarative, Kubernetes-native way to request PostgreSQL access. The service models database provisioning after the `PersistentVolume` / `PersistentVolumeClaim` pattern:

- Platform administrators configure **supply** through `DatabaseProvider` resources
- Components express **demand** through `SchemaClaim` or `DatabaseClaim` resources

The operator binds the two and delivers connection credentials via a `Secret` in the claim's namespace.

## Why

A number of components in OpenShift AI require relational database access (Model Registry, Data Science Pipelines, TrustyAI, MLflow, and others). Today each component brings its own database engine preference, connection configuration mechanism, secret naming convention, and lifecycle management approach. This fragmentation creates three concrete problems:

1. **Installation and configuration friction.** A significant portion of support cases are traced to storage and database prerequisite setup ([RHAIRFE-1141](https://redhat.atlassian.net/browse/RHAIRFE-1141)). Platform administrators must independently configure credentials, endpoints, and connection parameters for every component that needs a database.
2. **Operational complexity at scale.** A single OpenShift AI installation can accumulate numerous small, independent database instances with no shared observability, backup strategy, or credential rotation policy. Understanding the system as a whole becomes difficult when each component manages its own database lifecycle.
3. **No out-of-the-box defaults.** There are no sensible defaults that let administrators get a working platform without manually configuring every database dependency upfront. Components that could share a single PostgreSQL instance instead each require their own from-scratch setup.

A unified database service addresses all three by giving administrators a common configuration surface for components that opt in, giving components a standard API, and providing an optional platform-managed convenience backend that works out of the box, while preserving the ability for administrators to configure independent databases for specific components and services when needed.

## Goals

- Provide a declarative Kubernetes API for components to request database access without managing database lifecycle themselves
- Model supply and demand after the `PersistentVolume` / `PersistentVolumeClaim` pattern so the concepts are familiar to Kubernetes administrators
- Support both administrator-managed PostgreSQL instances (External) and a platform-managed convenience backend (Internal)
- Deliver connection credentials as standard Kubernetes `Secret` resources in the component's namespace; consumers decide how to consume them (mount, env vars, API read, or otherwise)
- **Publish a single, fixed credential contract as discrete fields**, so that every component configures database access the same way rather than the platform accommodating each component's existing shape
- Provide a default database provider with the stable name `rhai-db` so components can reference a well-known default. It is created automatically, is usable without administrator configuration, and is deliberately basic
- Let a component declare the PostgreSQL extensions it requires on its claim, in one provider-type-agnostic field. The platform installs them where it owns the instance, and verifies and reports where it does not
- Keep shared database adoption optional: components must retain the ability to configure a specific, independent database when the platform service is not suitable or not enabled
- Allow components to select among multiple configured providers when an administrator provisions more than the default
- Provide documentation covering API-level migration: how to reconfigure existing components to consume the shared service API instead of their current per-component database configuration. This includes guidance on creating the appropriate claims, mapping existing credentials, and validating connectivity. Data migration (moving data between database instances) is a component-level responsibility independent of this service: each component or module should provide its own data migration guidance, as this is a general operational requirement regardless of whether the shared service is in use
- Enable the platform to toggle the entire feature on or off through the existing module enablement mechanism

## Non-Goals

- Building a full database-as-a-service. The Internal backend does not include enterprise-grade capabilities (HA, automated backup/restore, performance tuning, multi-engine support); customers requiring these should use an External provider
- Supporting database engines other than PostgreSQL in the initial version
- Scheduled or user-initiated credential rotation workflows
- Integration with external secret management systems (e.g., HashiCorp Vault) for credential generation and storage
- Automatic data migration from existing per-component databases to the shared instance. This ADR covers API-level migration (making components use the shared service), not data movement between database instances
- **Installing PostgreSQL extensions into External providers.** The operator installs claim-declared extensions on the Internal provider, where it owns the instance and holds superuser. Against an External provider it verifies and reports only; it does not attempt privileged installation on infrastructure it does not own
- **Supporting arbitrary PostgreSQL extensions.** Installation on the Internal provider is limited to a platform-supported allow-list. Extensions outside it are not installed, even when the image happens to carry them
- **Schema migration orchestration.** Consuming components remain responsible for their own DDL and schema versioning
- Per-component database features that are unique to a single component and have no shared equivalent
- Enterprise-grade database capabilities (multi-cluster federation, advanced replication topologies, fine-grained performance tuning). These are covered by using an External provider pointed at an appropriately configured PostgreSQL deployment

## How

The service introduces four Custom Resource Definitions across two API groups. The following sections detail the resource model, its relationship to the existing Connection API, provider selection, provisioning flow, credential contract, supported PostgreSQL versions, TLS, the Internal backend, default provider, adoption model, drift recovery, one-way doors, and security considerations.

### Resource Model

The core abstraction separates supply (where databases live) from demand (what components need):

```mermaid
flowchart TB
    subgraph supply ["Supply (cluster-scoped)"]
        DP["DatabaseProvider"]
    end

    subgraph demand ["Demand (namespace-scoped)"]
        SC["SchemaClaim"]
        DC["DatabaseClaim"]
    end

    subgraph output ["Output (namespace-scoped)"]
        S1["Secret"]
        S2["Secret"]
    end

    DP --- SC
    DP --- DC
    SC --> S1
    DC --> S2
```

**DatabaseProvider** (cluster-scoped, `infrastructure.opendatahub.io/v1alpha1`) describes where claims should be provisioned. Two types are supported:

- **External**: points at an existing PostgreSQL instance managed by the administrator. The operator validates connectivity using an admin `Secret` and provisions claims against it, but does not manage the instance itself. The provider configuration controls whether the operator is allowed to create databases and schemas, so administrators can restrict provisioning to only what their policies permit.
- **Internal**: the platform deploys a single-instance PostgreSQL backend within the cluster as a convenience facility. It does not provide enterprise-grade capabilities (HA, automated backup/restore); customers requiring these should use an External provider.

```yaml
# External provider: administrator-managed PostgreSQL instance
apiVersion: infrastructure.opendatahub.io/v1alpha1
kind: DatabaseProvider
metadata:
  name: production-db
spec:
  type: External
  external:
    connectionSecretRef:
      name: production-db-admin
      namespace: redhat-ai-databases
    allowedOperations:
      - SchemaCreation
```

```yaml
# Internal backend: auto-created by the DatabaseService as default
apiVersion: infrastructure.opendatahub.io/v1alpha1
kind: DatabaseProvider
metadata:
  name: rhai-db
spec:
  type: Internal
  internal:
    namespace: redhat-ai-databases
    storage:
      size: 10Gi
```

**SchemaClaim** (namespace-scoped) requests a dedicated schema and user within a database. The claim can optionally specify a target database; if omitted, it uses the default database configured on the provider.

```yaml
apiVersion: infrastructure.opendatahub.io/v1alpha1
kind: SchemaClaim
metadata:
  name: model-registry
  namespace: redhat-ai-applications
spec:
  provider:
    name: rhai-db
  database: ml_platform
  secretName: model-registry-db
  access: ReadWrite
  deletionPolicy: Retain
```

**DatabaseClaim** (namespace-scoped) requests a dedicated database and user on the provider's PostgreSQL instance. If the database does not exist and the provider allows database creation, the operator creates it. If omitted, the default database configured on the provider is used.

```yaml
apiVersion: infrastructure.opendatahub.io/v1alpha1
kind: DatabaseClaim
metadata:
  name: analytics
  namespace: redhat-ai-applications
spec:
  provider:
    name: production-db
  database: analytics_prod
  access: ReadWrite
```

Both claim kinds accept a `spec.extensions` list naming the PostgreSQL extensions the component requires; see the Extensions section below for how it is satisfied on each provider type.

**DatabaseService** (`services.platform.opendatahub.io/v1alpha1`, cluster-scoped singleton) is the module enablement CR that lets the platform toggle the entire service on or off through the standard module lifecycle mechanism. It is also where the administrator configures whether a default `DatabaseProvider` is automatically created, and under what name. By default, the service creates an Internal backend named `rhai-db` when enabled.

```yaml
apiVersion: services.platform.opendatahub.io/v1alpha1
kind: DatabaseService
metadata:
  name: default-db-operator
spec:
  defaultProvider:
    managementState: Managed
    name: rhai-db
```

#### CRD Kind Naming

Kind names are immutable after release, so this is settled here rather than left to implementation.

**Decision: kind names are engine-neutral** — `DatabaseProvider`, `SchemaClaim`, `DatabaseClaim`, `DatabaseService`. Engine-specific alternatives (`PostgresSchemaClaim`) are rejected.

The engine is expressed in the spec, not in the kind. `DatabaseProvider.spec.engine` is an enum defaulted to `PostgreSQL`, and it is the discriminator that selects engine-specific behaviour and engine-specific `Secret` keys. Claims inherit the engine of the provider they bind to and do not restate it.

The rationale is that a kind-per-engine model multiplies the API surface for every engine added, forces consumers to change kind (not just configuration) to move between engines, and would require a second set of claim controllers. A spec discriminator keeps multi-engine support additive. This is consistent with the engine-neutrality this ADR already claims in its Risks section.

The cost is accepted and stated plainly: **a generic kind name over-promises while the service is PostgreSQL-only.** Any non-PostgreSQL value of `spec.engine` is rejected by CEL validation in this version, so the over-promise is visible at admission time rather than at runtime.

### Relationship to the Connection API

[ODH-ADR-Operator-0009](https://github.com/opendatahub-io/architecture-decision-records/blob/main/architecture-decision-records/operator/ODH-ADR-Operator-0009-connection-api.md) defines the Connection API: annotated `Secret` resources carrying credentials for external data sources, with an `opendatahub.io/connection-type-protocol` annotation driving validation and routing. It is normative for connection surfaces, and it explicitly permits custom protocols. [RHAISTRAT-176](https://redhat.atlassian.net/browse/RHAISTRAT-176) ("Connections 2.0") tracks its evolution. This section records why this service introduces new resources rather than reusing that API, and where it conforms to it instead.

**A new API group is warranted for the claim and provider resources.** The two surfaces solve disjoint problems:

| | Connection API (ODH-ADR-0009) | This service |
| --- | --- | --- |
| Origin of credentials | Supplied by a user or administrator who already has them | Generated by the operator during provisioning |
| Direction | Describes access to a resource that already exists | Requests that a resource be created |
| Lifecycle | None. ODH-ADR-0009 lists "connection pooling or lifecycle management" and "runtime secret rotation or dynamic credential management" as explicit Non-Goals | Binding, provisioning, drift recovery, retention policy, deletion |
| Supply side | None. There is no resource representing an available pool of connections | `DatabaseProvider`, with capability gating and selection |
| Schema ownership | Connection Types are Dashboard-owned | The credential contract is operator-owned and fixed by this ADR |
| Validation posture | Advisory. ODH-ADR-0009 states invalid connections are "not blocked" and are discovered at runtime | Authoritative. A claim that cannot be satisfied does not reach `Provisioned=True` |
| Current consumers | `Notebook`, `InferenceService`, `LLMInferenceService` | Platform components and operators, none of which are those workload kinds |

There is no declarative demand signal in the Connection API and nowhere to put one. Modelling a claim as an annotated `Secret` would mean encoding a request for provisioning inside the artifact that provisioning is supposed to produce.

#### Two artifacts, one contract

The distinction above is between *requesting provisioning* and *describing a connection*. Those are separate concerns, and this service deliberately separates them rather than assuming that every database a component uses was provisioned by this service.

- **A claim** (`SchemaClaim` / `DatabaseClaim`) is a request. It is a new kind because nothing in the Connection API can express demand.
- **A connection** is a description of how to reach a database that already exists. That artifact is **not new and this ADR does not invent one**: it is an ODH-ADR-0009 Connection `Secret` with protocol `postgres`, and it may be authored by hand, by GitOps, or by the Dashboard, exactly as ODH-ADR-0009 already provides for.

A claim, when fulfilled, **produces** a connection of exactly that kind. The credential contract defined in this ADR *is* the field schema of the `postgres` connection type.

The consequence is the important part, and it is a requirement rather than a convenience:

> **A consuming component accepts a `postgres` Connection `Secret`, and must not be able to tell whether it was produced by a claim or written by an administrator.**

This is what allows an administrator to point a component at a database this service never provisioned — a pre-existing corporate PostgreSQL, a cloud managed instance, an instance owned by another team — without the component needing a second configuration path, and without the platform needing admin credentials on that database. It is also what lets a component adopt the standard contract *before*, or entirely without, the provisioning side of this service being enabled.

**The output `Secret` therefore conforms to the Connection API,** carrying:

```yaml
metadata:
  annotations:
    opendatahub.io/connection-type-protocol: "postgres"
```

Registering the `postgres` protocol and its field expectations with the Connection API owners is a **prerequisite of this design, not an optional enhancement**, because it is the mechanism by which the hand-authored case and the provisioned case are the same artifact. It is tracked as delivery work.

The two surfaces also agree on namespace locality. ODH-ADR-0009 states cross-namespace Connection `Secret` consumption is not supported; this service writes the claim `Secret` into the claim's own namespace for the same reason. Every named consumer reads its database `Secret` from its own namespace, so this is the only workable placement regardless of which API describes it.

#### Databases the operator must not touch

An `External` `DatabaseProvider` points at a pre-existing instance, but the operator still provisions *into* it: it requires an admin `Secret` and creates a role per claim, even when `allowedOperations` forbids creating schemas or databases. There is no mode in which a claim binds to credentials that already exist and the operator creates nothing.

That mode matters for two situations that are known to occur: an administrator who will not delegate admin credentials on a corporate database, and a database whose roles are provisioned by a DBA out of band.

**Decision: those situations are served by a hand-authored Connection `Secret` in this version, and a bind-only claim mode is deferred.** The administrator authors a `postgres` Connection `Secret` directly in the consuming namespace. That works today, requires nothing at all from this service, and is precisely the case the paragraph above is built to support — the consuming component cannot tell the difference, so no component needs to change to accommodate it. What it forfeits is the central supply registry, the per-namespace materialization and the status reporting that `DatabaseProvider` provides.

A bind-only claim mode would recover those, and it is recorded in Future Considerations rather than built now: it is purely additive — a new value of an existing field, binding where today it provisions — so deferring it closes no door. It is related to but narrower than the rejected "Single Central Secret with Operator-Led Propagation" alternative, which discarded the demand signal and multi-provider support entirely; a bind-only claim would keep both. It overlaps the "alternative credential delegation models for External providers" item in Future Considerations and should be taken up with it.

### Provider Selection

Claims reference a provider by exact name or by label selector. When a selector matches multiple providers, the operator picks the one with the highest `db.infrastructure.opendatahub.io/selection-priority` annotation, breaking ties alphabetically. Once a selector-based claim binds to a provider, it keeps that provider as long as it still exists and matches. A newly appearing or higher-priority provider does not force rebinding.

### Provisioning Flow

The end-to-end flow from administrator setup to component consumption:

```mermaid
sequenceDiagram
    participant Admin as Platform Admin
    participant DP as DatabaseProvider
    participant Claim as SchemaClaim
    participant Op as DB Operator
    participant PG as PostgreSQL
    participant Secret as Credentials Secret
    participant Consumer as Consumer

    Admin->>DP: Create provider (External or Internal)
    Op->>PG: Validate connectivity or deploy instance
    Op->>PG: Read server_version_num
    Op->>DP: Set Reachable=True, record server version in status
    Note over Claim: Created by admin, component controller,<br/>or shipped as a default manifest
    Op->>DP: Resolve provider
    Op->>PG: CREATE DATABASE/SCHEMA + CREATE ROLE + GRANT
    Op->>PG: Install declared extensions (Internal) or verify them (External)
    Op->>Secret: Write credentials via SSA
    Op->>Claim: Set Provisioned=True

    Consumer->>Secret: Consume credentials (mount, env vars, etc.)
    Consumer->>PG: Connect using credentials
```

### Credential Contract

When a claim reaches `Provisioned=True`, the operator writes a `Secret` in the claim's namespace. **This is the hardest artifact in the design to change after release, because every adopting component encodes its key names.** It is therefore fixed here, key by key.

The `Secret` is of type `Opaque` and carries the `opendatahub.io/connection-type-protocol: "postgres"` annotation described above.

| Key         | Always present | Content                                                                 |
| ----------- | -------------- | ----------------------------------------------------------------------- |
| `host`      | yes            | Service DNS name (Internal) or external hostname                        |
| `port`      | yes            | PostgreSQL port (default 5432)                                          |
| `user`      | yes            | Generated role name, unique per claim                                   |
| `password`  | yes            | Generated password, **alphanumeric only** (see below)                   |
| `dbname`    | yes            | Database name                                                           |
| `sslmode`   | yes            | libpq SSL mode: `disable`, `require`, `verify-ca` or `verify-full`      |
| `schema`    | `SchemaClaim` only | Schema name                                                         |
| `ca.crt`    | when the provider presents a CA | PEM-encoded CA bundle for verifying the server certificate |

**These discrete fields are the whole contract. The service publishes no pre-composed connection string, DSN or URI.** The reasoning is below, and it is a deliberate reversal of an earlier draft of this section.

The `Secret` name defaults to the claim's name but can be overridden. The database service only writes the `Secret`; how consumers use it (volume mount, environment variables, direct API read, or any other mechanism) is entirely up to them. The service imposes no consumption machinery.

#### Key naming

Keys are **unprefixed**, following the convention established by CloudNativePG, whose connection `Secret` uses exactly `host`, `port`, `user`, `password` and `dbname` ([`pkg/specs/secrets.go`](https://github.com/cloudnative-pg/cloudnative-pg/blob/main/pkg/specs/secrets.go)). A `pg.`-prefixed variant was considered and rejected on two grounds:

1. **Dotted keys are not usable with `envFrom`.** The Kubernetes `EnvFromSource` contract requires keys to be a `C_IDENTIFIER`; keys containing `.` are silently skipped when a `Secret` is projected with `envFrom`, and the warning event that used to report this is unreliable on current versions. `pg.host` therefore cannot be consumed by the simplest and most common projection mechanism, while `host` can.
2. **It diverges from the ecosystem for no gain.** The prefix namespaces keys that already live in a single-purpose `Secret`.

`ca.crt` keeps its dot deliberately: it is consumed as a file, never as an environment variable, and `ca.crt` is the established Kubernetes convention for CA bundles (`kube-root-ca.crt`, service CA ConfigMaps, and CloudNativePG alike).

CloudNativePG additionally emits `username` as an alias for `user`, along with `pgpass`. Neither is required by any named consumer, and both are additive, so they are omitted here and can be introduced later without breaking the contract. CloudNativePG's `uri`, `jdbc-uri`, `fqdn-uri` and `fqdn-jdbc-uri` keys are omitted for the reasons in the next section.

#### No composed URI

Four of the named consumers today accept **only** a pre-composed URI or DSN — MaaS reads `DB_CONNECTION_URL`, eval-hub reads `db-url`, MLflow takes `MLFLOW_BACKEND_STORE_URI`, DataConnectHub reads a `url` field inside a TOML fragment. Only Model Registry (`host`/`port`/`username`/`database`) and Dashboard gen-ai (`PGVECTOR_*` environment variables) take discrete values.

An earlier draft of this section proposed publishing both shapes so that each consumer could read the one it already understood. **That is rejected.** The contract is discrete fields only, for four reasons:

1. **Two representations of one truth is a drift surface.** Even written atomically by one controller, a `uri` and its component keys are two encodings that must be kept in agreement forever, by every future change to either. A consumer that reads `host` and a consumer that parses `uri` must never be able to get different answers, and the only way to guarantee that is to have one source.
2. **Composition is cheap and parsing is not.** Building a URI from fields is a one-line operation in every language in the product. Recovering fields from a URI requires correct handling of percent-decoding, IPv6 bracketing, optional ports, empty passwords and query parameters. Publishing only the fields gives every consumer the cheap direction; publishing only the URI would force some of them into the expensive one.
3. **There is no single correct URI.** The syntax that carries a schema is driver-specific: libpq and drivers built on it take `options=-c search_path=...`, the PostgreSQL JDBC driver takes `currentSchema=...`, Npgsql takes a `Search Path` keyword in a non-URI connection string. This service does not constrain which driver a consumer uses, so it cannot compose a string that is correct for all of them. CloudNativePG's own answer to this is to emit four different URI keys, which is an admission that the URI is an ecosystem convenience rather than a contract. The platform should not ship a string whose correctness depends on a choice the platform does not make.
4. **The purpose of this ADR is to normalize, not to accommodate.** Publishing a URI because four consumers currently expect one preserves the fragmentation the service exists to remove, and locks the platform into serving whichever shapes happen to exist today. Components adapting to a common contract is the expected cost of standardization, not a failure of it.

The consequence is stated plainly: **every consumer that reads a single connection string must change to compose one from the published fields.** Composition from these fields is a few lines in any language, and each consumer can do it directly. A shared Go helper in `odh-platform-utilities` would spare most of them from writing it separately and is recommended, but this contract does not depend on one existing: the fields are the contract, and a consumer that composes its own string is fully conformant.

Two properties of the contract exist specifically to make consumer-side composition safe:

- **Generated passwords are alphanumeric** (`[A-Za-z0-9]`). Every character is an RFC 3986 unreserved character, so a naive `postgresql://user:password@host:port/dbname` interpolation is correct without percent-encoding. This is a **normative guarantee, not an implementation detail**: because composition now happens in consumers rather than in the operator, widening the password alphabet later would silently break naive composers. Any future change to password generation must preserve URL-safety.
- **`schema` is a first-class key.** A consumer holding a `SchemaClaim` must apply it — by setting `search_path` on connect, by schema-qualifying its DDL, or through whichever driver parameter its stack provides. This is the one place where ignoring a key is dangerous rather than merely limiting: a consumer that connects without applying `schema` connects *successfully* and then reads and writes the wrong schema. Adoption documentation must lead with this.

#### What the `Secret` does not carry

The `Secret` carries connection identity, credentials and transport security. It does not carry connection behaviour: pool sizes, connect and statement timeouts, `application_name`, client encoding, retry policy and read/write routing preferences are all **component configuration**, expressed wherever that component already expresses its configuration. They are not platform contract, they vary legitimately per consumer, and they are the category that a composed URI would have quietly invited into the contract as query parameters.

Anything that is genuinely a property of the *provider* rather than of a consumer's behaviour belongs on `DatabaseProvider` or on the claim, surfacing into the `Secret` as a new key — the route `sslmode` and `ca.crt` already took. New keys are additive and therefore safe; the one-way door is renaming or removing an existing key, not adding one.

Three such properties are recognized as absent today and are candidates for that route rather than gaps in this contract:

- **Client certificate authentication.** The contract assumes password authentication. An External provider requiring mTLS would need `tls.crt` and `tls.key` keys and a provider-side source for them. Not required by any named consumer today.
- **Multi-host and read-replica endpoints.** `host` is singular. A provider fronted by an HA cluster is expected to present a single DNS name or proxy endpoint. Consumers wanting a distinct read-only endpoint — MLflow has a `readReplicaBackendStoreUri` — are not served, and a second key or a second claim would be the mechanism.
- **Token-based authentication for managed cloud PostgreSQL.** AWS RDS IAM authentication and Azure Entra authentication issue short-lived tokens rather than passwords. Supporting them would mean a credential that expires on a timer, which directly contradicts this ADR's Non-Goal on rotation. Worth naming now precisely because it is the one plausible future requirement that reopens a closed decision.

### PostgreSQL Versions

**Decision: this service does not define, gate on, or enforce a PostgreSQL version range.** It reports the server version and leaves version requirements to the parties that actually hold them.

This reverses the obvious expectation, so the reasoning is given in full.

**The operator has no version floor of its own.** The SQL it executes to fulfil a claim is `CREATE ROLE`, `CREATE SCHEMA IF NOT EXISTS`, `CREATE DATABASE`, `GRANT`, `ALTER DEFAULT PRIVILEGES`, `ALTER ROLE` and `CREATE EXTENSION IF NOT EXISTS`. The newest construct in that set dates to PostgreSQL 9.3. The operator uses no predefined roles such as `pg_read_all_data` (14+), no `security_invoker` views (15+), and nothing else version-sensitive. There is therefore no version below which this service stops working, and a floor asserted by this ADR would be an invented constraint rather than a discovered one.

**Neither of the other two parties is served by a range in this API.**

- *The Internal backend* runs an image the platform selects. Its version is a property of that image and is stated in product documentation. There is nothing to negotiate and nothing to check.
- *An External provider* is a database the administrator already owns and whose version they already know. A `minVersion` field on a provider or a claim would ask them to declare a fact the operator can simply read, in service of a requirement they do not hold.

**Component version requirements are component business.** Only a component knows which SQL features it depends on, and only the component can act on the answer. Any client can determine the server version on connect — `SHOW server_version_num` needs no privileges — so a component with a genuine floor can check it and fail with a clear message. A platform-enforced range cannot do this better, because the platform does not know any component's requirement; it can only apply one number to all of them and be wrong in both directions. The floors observed in the product today illustrate the point: eval-hub declares 12 and later and MLflow 13 and later, both so far below any plausible deployment that enforcing them would never change an outcome.

**Prefer capability detection to version gating.** Where a claim has a hard requirement, it should state the capability rather than a version that implies it. The extensions mechanism below is the instance of this that exists: `pgvector` availability is what the Dashboard gen-ai consumer actually needs, and checking for the extension tests the requirement directly, where a version number would only be a proxy for it.

**What the service does instead:**

- The operator reads `server_version_num` during the provider reachability check and publishes it on `DatabaseProvider.status`. It is informational.
- Outside the range the product documents as tested, the provider reports a condition with a reason saying so. **This never blocks binding.** Refusing to connect to a customer's working database because it is older than a number in an ADR is a worse outcome than connecting and saying so.

**Where the supported range does belong.** RHAIRFE-1141's engineering response committed to defining and enforcing a supported PostgreSQL version range, and that commitment stands — but its subject is the product's component compatibility matrix, not this API. "Which PostgreSQL versions does OpenShift AI support?" is one question answered once for all components, owned by product management and QE, and enforced by what the release is tested against. Answering it inside the database service would scope a product-wide statement to one optional feature, and would leave components that never adopt this service uncovered by it. The value of that statement is unchanged; the place for it is release documentation.

#### Extensions

Extension availability decides whether a consumer can adopt at all: the Dashboard gen-ai consumer cannot use this service without pgvector. A component must therefore be able to state that requirement on its claim rather than discover it at runtime. Extensions are named as PostgreSQL names them, so pgvector is declared as `vector`.

**A claim declares the extensions it requires.** `spec.extensions` is a list of PostgreSQL extension names, and it is the same field regardless of provider type — a component states what it needs, not how the platform should get it.

```yaml
apiVersion: infrastructure.opendatahub.io/v1alpha1
kind: SchemaClaim
metadata:
  name: gen-ai-vectors
  namespace: redhat-ai-applications
spec:
  provider:
    name: rhai-db
  secretName: gen-ai-db
  extensions:
    - vector
```

**What the operator does with that list depends on who owns the instance**, and this asymmetry is deliberate:

- **Internal provider — install on demand, from a supported allow-list.** The operator owns the instance and holds superuser on it, so it runs `CREATE EXTENSION IF NOT EXISTS` for each declared extension, scoped to the claim's database. Installation is restricted to an allow-list of extensions the platform ships in its PostgreSQL image and is prepared to support. An extension outside that set is rejected at admission, so a component learns immediately rather than at provisioning time. The allow-list is a supportability boundary, not a technical one: extensions present in the image but not on the list are not installed, because shipping an extension and supporting it are different commitments.
- **External provider — verify and report, never install.** `CREATE EXTENSION` generally requires superuser, and the operator does not hold, and should not require, that level of privilege on infrastructure the administrator owns. The operator checks `pg_available_extensions` and `pg_extension`, and a claim whose required extensions are absent does not reach `Provisioned=True`; the condition names the missing extension so the administrator knows exactly what to install. Administrators install extensions on External providers themselves.

The consequence for a component is that `spec.extensions` is a portable statement of requirement. The same claim binds against either provider type; on the Internal provider it is satisfied automatically, on an External provider it either already holds or tells the administrator precisely what to do. A component never needs a provider-type-specific code path for extensions.

**This ADR fixes the mechanism, not the membership.** The allow-list is published with each release alongside the PostgreSQL image it corresponds to, because what the platform can support is a property of that image and of who owns its CVE exposure — both of which are delivery decisions that change on a release cadence, where this ADR does not. Naming a fixed set here would either freeze the list at whatever was true when this document was written, or be silently wrong the first time the image changed. What review should test is the mechanism: that a claim declares extensions, that the boundary is a supported set rather than whatever the image happens to carry, and that the rejection is at admission.

`vector` is the only extension a named consumer requires today, so it is the one the initial list must contain. `pg_trgm`, `uuid_ossp` and `pgcrypto` are installed by the proof of concept at first boot and are the natural remainder of the initial set, but that is evidence of intent rather than a support commitment and is for the image owners to confirm.

Adding an extension to the allow-list in a later version is additive and safe. Removing one is not, and the allow-list is therefore treated with the same care as the credential keys.

### TLS

TLS is part of the credential contract and is described here rather than left to implementation.

- For the Internal backend, the operator issues a server certificate through **cert-manager**. An `issuerRef` may be supplied; when it is not, the operator creates a provider-scoped self-signed issuer.
- Certificate state is reported through a `TLSConfiguration` condition on both `DatabaseProvider` and claim resources, and through `status.tls` on the provider.
- The resulting trust material reaches consumers through the `ca.crt` and `sslmode` keys of the claim `Secret`.
- For External providers, TLS is the administrator's responsibility. The operator propagates the CA supplied on the provider's admin `Secret`, and uses the provider's configured SSL mode.
- Default SSL mode is `require` for External providers and `disable` for the Internal backend's in-cluster traffic unless TLS is configured, in which case it follows the issued certificate.
- The `Secret` publishes CA *material*, not a path. A consumer using `verify-ca` or `verify-full` mounts `ca.crt` and points its driver's trust-root setting (`sslrootcert` for libpq) at wherever it mounted the file. The operator cannot supply that path, because it does not know the consumer's mount layout.

Three known gaps are recorded rather than designed away:

- The negotiated cipher suite does not currently follow the cluster's OpenShift TLS security profile.
- There is no defined degradation path when cert-manager is absent.
- **CA delivery shape varies by consumer.** Publishing `ca.crt` inside the claim `Secret` suits consumers that mount it, but several take their CA bundle from a `ConfigMap` instead — MLflow's `spec.caBundleConfigMap` accepts only a `ConfigMap`, and Data Science Pipelines merges everything into one. Those consumers need a copy of the CA in a `ConfigMap`, which this service does not currently produce. Whether the service emits a companion `ConfigMap` or each consumer arranges its own is a delivery decision; the `Secret` key is the contract either way.

All three are delivery concerns against this ADR, not open design questions.

### Internal Backend

The Internal backend is a convenience facility that the platform ships to reduce initial setup friction. It does not provide enterprise-grade capabilities such as HA or automated backup/restore. Customers requiring these capabilities should use an External provider.

When an Internal `DatabaseProvider` is created, the operator deploys a single-instance PostgreSQL backend within the cluster using a Red Hat supported image. The operator manages the full lifecycle of the backing resources and restricts network access to only namespaces with active provisioned claims.

The Internal backend is intentionally limited:

- Single instance, no HA, no automated backup/restore
- Designed for getting started, development, and experimentation

**Both provider types are in scope for the initial version, and neither is a stepping stone to the other.** The Internal provider exists so that the platform has a working default out of the box, which is the third of the three problems in "Why" and cannot be solved by an External provider. The External provider exists because the Internal one is explicitly not an enterprise database, and any customer with a real durability, HA or compliance requirement must be able to point the service at infrastructure that meets it. Cutting either one leaves the service unable to serve a population it claims to serve.

Because the operator owns this instance and holds superuser on it, it installs the extensions a claim declares, restricted to the allow-list in the Extensions section above. This is the one capability the Internal provider offers that an External provider cannot, and it is the reason `spec.extensions` on a claim is a portable statement of requirement rather than a provider-specific setting.

**The operator never reclaims the Internal backend's backing resources automatically.** An earlier draft allowed the provider's storage to be deleted after a configurable idle grace period once no claims referenced it, which suited ephemeral and experimentation use. That is removed: a timer that destroys a customer's data is not a behaviour this service should ship in its initial version, and "no claims reference this provider right now" is not evidence that the data in it is unwanted. An administrator who wants the backend gone deletes the `DatabaseProvider`. Reclaim may return later as an explicitly opted-in provider setting; it is recorded in Future Considerations.

This is distinct from a claim's `deletionPolicy`, which is unaffected and continues to govern what happens to that claim's own schema or database when the claim is deleted.

### Default Provider

When the `DatabaseService` CR is enabled (the default), the platform automatically creates an Internal backend named `rhai-db`. Components that do not need a specific provider can reference this well-known name as their default.

**The default provider is deliberately basic.** It is created with platform-chosen defaults for storage size, resources and TLS, and requires no administrator input to become usable — an administrator who does nothing still gets a working database on a fresh install. That is the whole point of it, and it is why the auto-creation is part of this design rather than a sample manifest: a default that must be configured before it works does not discharge the "no out-of-the-box defaults" problem in "Why". Administrators who need more than the defaults change them on the `DatabaseProvider`, or create an External provider and point components at it. The default provider name is configurable through the `DatabaseService` CR but is immutable once set: changing it after initial creation is not allowed, since components and claims already reference it. If an administrator chooses a name other than the default, they are responsible for ensuring that all component configurations reference the correct provider name.

### Component Adoption Model

Adoption is opt-in and incremental. A component that adopts the shared service:

1. Ships a `SchemaClaim` (or `DatabaseClaim`) referencing a provider by name or selector
2. Consumes the credentials `Secret` through standard Kubernetes mechanisms (volume mount, environment variables, or direct API read)

Additionally, the component must retain its existing configuration path for administrators who prefer an independent database. This is a firm requirement: the shared database service is an option, not a mandate. Every component must preserve the ability to be configured with a specific, independent database connection. This keeps the system flexible for environments where a shared database is not appropriate. This dual-path is a deliberate trade-off: during the incremental adoption period, administrators may need to configure some components through the shared service and others through their legacy paths. The configuration surface converges as more components adopt the shared service, but full convergence is not a prerequisite for the service to deliver value.

When an administrator configures more than one `DatabaseProvider`, components must have a way to select which provider to use, either by referencing a specific provider name or by using a label selector that matches the desired provider's capabilities.

### Drift Recovery

Claims self-heal when their managed resources drift:

- A missing credentials `Secret` triggers re-provisioning of the PostgreSQL role and a new `Secret` write (generating a new password in this case)
- A missing schema (`SchemaClaim`) triggers schema recreation
- A missing database (`DatabaseClaim`) triggers database recreation if the provider allows it
- A missing role triggers role recreation with a new password

Repairs are performed during normal periodic reconciliation. The operator never silently drops data. `SchemaClaim` with `deletionPolicy: Retain` (the default) drops only the role on claim deletion; the schema and its data persist.

Drift recovery is **not** credential rotation, and this ADR does not introduce rotation. A password changes only when database-side state has to be re-provisioned; there is no scheduled or on-demand rotation workflow. Rotation remains a Non-Goal, and delivery work that assumes otherwise should be cut back to match.

### One-Way Doors

Three decisions in this design cannot be reversed after release without breaking existing installations. They are called out explicitly so that review addresses them deliberately.

1. **Credential `Secret` key names.** Every adopting component encodes them. Changing a key after release breaks every adopter simultaneously. Adding keys is safe; renaming or removing is not. Note that the decision *not* to publish a composed URI keeps this door as narrow as it can be: a published URI would additionally freeze a composition **algorithm**, and an algorithm is a considerably worse thing to be unable to change than a name. Adding a URI key later remains possible; removing one would not have been.
2. **Resource scope per kind.** `DatabaseProvider` and `DatabaseService` are cluster-scoped; `SchemaClaim` and `DatabaseClaim` are namespace-scoped. Kubernetes does not support changing the scope of an existing CRD; doing so requires a new kind and a migration. The scoping follows the PV/PVC analogy and the requirement that claim `Secret`s land in the consumer's namespace.
3. **The default provider name `rhai-db`.** Configurable at first creation, immutable thereafter, and baked into component defaults that reference it.

CRD kind names are a fourth such door and are closed in the Resource Model section above.

The extension allow-list is a fifth of the same character, though a narrower one: adding an extension is safe, and removing one breaks every claim that declares it. It is not listed with the three above because a claim that names a removed extension fails loudly at admission rather than silently at runtime, but it should be extended with the same reluctance.

### Security and Tenant Isolation

- **Per-claim isolation**: each claim is provisioned with a unique PostgreSQL role and a dedicated password. The resulting credentials `Secret` exists only in the claim's namespace. A consumer in namespace A cannot access credentials generated for namespace B, and each role is scoped to only the resources (schema or database) requested by its claim.
- **Network isolation (Internal only)**: the operator dynamically configures a `NetworkPolicy` on the Internal backend's PostgreSQL `Pod`, allowing ingress only from namespaces with active provisioned claims. This list is recomputed on every reconcile. Namespace-level isolation is the minimum boundary provided by this service; additional security measures (e.g., pod-level or service-account-level controls) should be investigated if stricter isolation is required. For External providers, network isolation is the administrator's responsibility; this operator does not create `NetworkPolicy` resources for infrastructure it does not own.
- **No credentials in logs**: generated credentials must never be logged or cached. Because consumers compose their own connection strings, the composed string embeds the password and becomes a new place it can leak — into logs, into error messages on connection failure, and into `application_name`-style diagnostics. Consumers must redact it, and any shared composition helper should offer a separately-redactable form for logging.
- **SQL injection prevention**: all identifiers and literals interpolated into DDL must use proper quoting to prevent injection.
- **Standing DDL privileges**: several current consumers run unversioned DDL at every startup. On a shared provider this means the claim role retains schema-modification rights for the life of the claim. The `access` field on a claim bounds this, but components that can separate migration from runtime credentials should be encouraged to do so.

### Future Considerations

This ADR focuses on the initial implementation. The following items are recognized as valuable but deferred:

- **Automatic tracking migration**: an automated mechanism to discover existing per-component databases (those created ad-hoc by components before this service existed) and register them as `DatabaseProvider` + claim resources. This does not involve data migration; it ensures existing databases are tracked through the shared API so administrators have a unified view. This is a Day 2 operational improvement, not a prerequisite for initial adoption.
- **Credential rotation**: a scheduled or on-demand workflow for rotating claim credentials. The current design repairs missing credentials but does not offer proactive rotation.
- **External secret management integration**: support for delegating credential generation and storage to external systems such as HashiCorp Vault, enabling centralized secret lifecycle management and stronger security guarantees than in-operator password generation.
- **Alternative credential delegation models for External providers**: the initial design assumes the operator holds admin-level credentials to provision roles and schemas on External databases. In environments where this delegation is not acceptable, alternative models (e.g., pre-provisioned credentials supplied by DBAs, or a request-approval workflow) should be explored.
- **Bind-only claims**: a claim that binds to credentials which already exist and provisions nothing, for databases the operator must not modify. Deferred as described in "Databases the operator must not touch"; the case is served by a hand-authored Connection `Secret` in this version. Purely additive, and best taken up together with the delegation models above.
- **Idle reclaim of the Internal backend**: automatic deletion of an Internal provider's backing resources after a grace period with no referencing claims, removed from the initial version as described in "Internal Backend". If it returns it should be an explicitly opted-in provider setting rather than a consequence of a claim-level policy, since the resource it destroys is a customer's data.
- **Additional credential keys**: `username` as an alias for `user`, and `pgpass`, as emitted by CloudNativePG. Purely additive, and introduced when a consumer needs them. CloudNativePG's URI keys are excluded by the decision above and are not on this list.
- **Client certificate authentication, read-replica endpoints, and token-based authentication for managed cloud PostgreSQL**: the three recognized absences from the credential contract, each described in "What the `Secret` does not carry". The third is the one that would force a Non-Goal in this ADR to be reopened, since cloud IAM tokens expire.
- **Extension version and configuration control**: `spec.extensions` names extensions, not versions, and offers no way to pass `CREATE EXTENSION` options such as a target schema. Both are additive to the field's shape and neither is required by a named consumer today.
- **Non-PostgreSQL engines**: the initial implementation targets PostgreSQL exclusively, since all current internal service requirements are PostgreSQL-based. The `spec.engine` discriminator exists so that support can be added without new kinds. CRD kind naming is settled in this ADR and is not revisited by multi-engine work.

## Alternatives

### Reuse the Connection API (ODH-ADR-Operator-0009) for the whole surface

Model providers and claims as annotated `Secret` resources under the existing Connection API with a custom `postgres` protocol, rather than introducing new kinds. Rejected: the Connection API describes credentials that already exist and has no supply side, no demand signal, no binding, and no provisioning lifecycle — all of which are explicit Non-Goals in ODH-ADR-0009. A request to create a database cannot be expressed as the artifact that creating a database produces. The output `Secret` does conform to the Connection API, which captures the reusable part of the overlap.

### Publish a Composed URI Alongside the Discrete Keys

Emit a `uri` key in addition to `host`, `port`, `user`, `password` and `dbname`, as CloudNativePG does, so that the four consumers which currently read a single connection string can adopt without code changes. Rejected, and the reasoning is in the Credential Contract section above: it creates two representations of one truth that must be kept in agreement forever, it freezes a composition algorithm rather than a set of names, there is no single URI syntax that is correct across the drivers in use, and it entrenches the per-component fragmentation this service exists to eliminate. This is the most likely reviewer suggestion and it is pre-empted deliberately rather than left to be relitigated.

A narrower variant — an opt-in `spec.emitConnectionURI` on the claim — was also considered and rejected. It has every drawback above and adds a second contract shape that adopters must reason about.

### Single Central Secret with Operator-Led Propagation

The administrator provides a single `Secret` with PostgreSQL connection details, and the ODH Operator propagates those credentials to opted-in components. Simpler to implement, but provides no per-component isolation, no declarative demand signal, and no mechanism for multiple backends. The claim-based model subsumes this approach while retaining flexibility.

### Maintain the Per-Component Status Quo

Each component continues to manage its own database configuration independently. Requires no new infrastructure but preserves all the problems motivating this ADR: duplicated configuration, inconsistent credential management, no shared defaults, and growing support case volume.

### Standardize the Consumer Contract Only

Agree on a common `Secret` schema without building provisioning logic. Solves configuration inconsistency but not the provisioning burden; administrators still create every database, schema, role, and `Secret` manually.

### Full PostgreSQL Operator (CloudNativePG, Crunchy)

Depend on a mature PostgreSQL operator for both instance lifecycle and access management. Provides HA, backup/restore, and monitoring but introduces a heavyweight dependency. Most components only need "give me a schema and credentials" and do not need to reason about cluster topology or backup schedules. A further constraint is that the platform does not auto-install external operators, so a hard dependency would mean either bundling the operator — with the licensing, build, disconnected-install and CVE-ownership consequences that implies — or making the feature conditional on the customer having installed it. Customers who want a full PostgreSQL operator can still use one and expose it to this service through an External provider; the credential contract above deliberately matches CloudNativePG's key names so that this path is low-friction.

## Risks

- **Adoption requires cross-team coordination.**
  - *Rationale*: each component team must modify their controller to optionally consume the shared service API.
  - *Mitigation*: adoption is incremental and opt-in; components adopt at their own pace, and the existing independent database path is preserved.
- **The credential contract is fixed before every consumer has implemented against it.**
  - *Rationale*: key names cannot be changed after release, and most of the named consumers have not yet reviewed the schema.
  - *Mitigation*: the contract is deliberately minimal — discrete fields only, no composed forms — so there is less surface to get wrong, and every plausible extension is additive. Ratification requires review by at least two consumer teams, one of them a single-connection-string consumer.
- **Discrete-only means four of six named consumers must change code to adopt.**
  - *Rationale*: MaaS, eval-hub, DataConnectHub and MLflow all consume a single connection string today and have no field for discrete values. Requiring composition converts a configuration change into a code change for each, and for MLflow specifically converts what would have been a zero-change adoption (its `backendStoreUriFrom` accepts any `Secret` key) into a CRD change. This is the direct and intended cost of refusing to accommodate existing shapes.
  - *Mitigation*: adoption is opt-in and incremental, so no team is forced to absorb this on the platform's schedule, and composition is a few lines per consumer. A shared Go helper in `odh-platform-utilities` would reduce that further for the three of the four affected consumers that are Go, but it is a convenience and not the mitigation — the change is small enough that each team can absorb it unaided. The alternative — publishing a URI — buys short-term adoption by permanently encoding today's fragmentation into the platform contract.
- **The extension allow-list is a support commitment, and it is asymmetric between provider types.**
  - *Rationale*: every extension on the list is something the platform installs, ships in its image, and owns the CVE exposure for. The list is also the boundary of what the Internal provider can satisfy automatically, so a component whose requirement falls outside it cannot use the default provider at all — it must be pointed at an External provider where an administrator installs the extension by hand. Adding to the list is therefore not free, and refusing to add to it has an adoption cost.
  - *Mitigation*: the list is published per release rather than fixed by this ADR, and its initial contents need only cover what named consumers actually require — `vector`. Additions are additive and can be made in any later release; each one is a deliberate decision about what the product supports, taken by the same people who own the PostgreSQL image.
- **Extension installation gives the operator DDL reach it would not otherwise need.**
  - *Rationale*: `CREATE EXTENSION` runs as superuser on the Internal instance, and the set of extensions the operator will run it for is driven by a field a claim author controls.
  - *Mitigation*: the allow-list is enforced at admission, so the operator only ever names a value from a fixed set in the statement it executes. This privilege exists only against the Internal instance, which the operator already owns end to end; External providers are never subject to it.
- **Internal backend is single-instance with no HA.**
  - *Rationale*: a failure in the Internal backend's PostgreSQL pod affects all components using that provider.
  - *Mitigation*: the Internal backend does not provide HA or automated failover by design. Environments requiring these capabilities should use an External provider pointing at an HA-capable PostgreSQL deployment.
- **All components must support PostgreSQL.**
  - *Rationale*: the shared service targets PostgreSQL exclusively. Components that currently use a different database engine or rely on engine-specific features must add PostgreSQL support to adopt the shared service. Data Science Pipelines is MySQL-only today and cannot adopt without upstream Kubeflow Pipelines and MLMD work.
  - *Mitigation*: PostgreSQL is already the most common engine across OpenShift AI components. Components that cannot adopt PostgreSQL retain their independent database configuration path.
- **PostgreSQL-only scope may not cover all component needs.**
  - *Rationale*: some components may have requirements better served by a different engine.
  - *Mitigation*: components with genuinely distinct requirements keep their own database configuration; this service targets the common case, and the API naming is engine-neutral to allow future expansion.
- **Shared database creates a shared failure domain.**
  - *Rationale*: multiple components depending on the same database instance means a database outage affects multiple components simultaneously.
  - *Mitigation*: administrators who need fault isolation can configure multiple `DatabaseProvider` resources and assign components to separate backends.
- **Noisy-neighbor risk on shared instances.**
  - *Rationale*: one component's connection storms, long-running transactions, or vacuum pressure can degrade performance for all other components on the same provider.
  - *Mitigation*: resource isolation is a property of the PostgreSQL instance, not of this service. Administrators who need performance isolation should use separate providers.
- **Credential changes may cause transient consumer outages.**
  - *Rationale*: drift recovery can generate new passwords for claim credentials. Consuming components must detect the `Secret` change and re-establish connections.
  - *Mitigation*: this is a consumer-side responsibility documented in the credential contract.
- **Table-name collisions on shared providers.**
  - *Rationale*: some consumers create generically named, schema-unqualified tables. On a shared database these collide.
  - *Mitigation*: `SchemaClaim` exists for exactly this case, and the `schema` key tells the consumer where it must place its objects. Because the service does not compose the connection string, applying `schema` is the consumer's responsibility, and a consumer that ignores it will collide silently rather than fail loudly. Adoption documentation must treat `schema` as mandatory rather than optional, and any shared composition helper should do the same.
- **The operator requires broad cluster privileges.**
  - *Rationale*: cross-namespace `Secret` management, cluster-scoped provider reconciliation, and database admin credentials represent a meaningful RBAC surface.
  - *Mitigation*: RBAC is scoped per-verb using Kubebuilder markers, and the operator follows the platform's existing module RBAC conventions. Security review should evaluate the privilege surface as part of the module onboarding process.
- **Users may rely on the Internal backend without understanding it lacks enterprise-grade features.**
  - *Rationale*: the default provider is auto-created with a stable name and works out of the box, which makes it easy for users to normalize around it without switching to an External provider.
  - *Mitigation*: documentation and status reporting should clearly communicate that the Internal backend does not include HA, automated backup/restore, or other enterprise-grade capabilities. The platform may surface warnings when these features would be expected.
- **Success depends on cross-organizational alignment.**
  - *Rationale*: adoption requires coordination across platform engineering, individual component teams, and customer database administrators, each with different priorities and constraints.
  - *Mitigation*: the opt-in model and preserved independent configuration path reduce the urgency of full alignment. Components can adopt incrementally without requiring all stakeholders to agree upfront.

## Stakeholder Impacts


| Group                    | Key Contacts               | Date | Impacted? |
| ------------------------ | -------------------------- | ---- | --------- |
| Architects Team          | @opendatahub-io/architects |      | y         |
| Platform Team (Operator) |                            |      | y         |
| Model Registry           |                            |      | y         |
| Data Science Pipelines   |                            |      | y (blocked: MySQL-only) |
| TrustyAI                 |                            |      | y         |
| MLflow                   |                            |      | y (must compose; `backendStoreUriFrom` takes one key) |
| MaaS                     |                            |      | y (must compose; reads a single URI today) |
| eval-hub                 |                            |      | y (must compose; reads a single URI today) |
| DataConnectHub           |                            |      | y (must compose; reads a single URI today) |
| Dashboard (gen-ai)       |                            |      | y (declares `extensions: [vector]`; installed on Internal, verified on External) |


## References

- [POC: opendatahub-db-operator](https://github.com/lburgazzoli/opendatahub-module-operator/tree/db-service/modules/opendatahub-db-operator): proof of concept implementation with full CRD definitions, controller logic, and integration tests
- [ODH-ADR-Operator-0009: Connection API and Connection Type Protocol](https://github.com/opendatahub-io/architecture-decision-records/blob/main/architecture-decision-records/operator/ODH-ADR-Operator-0009-connection-api.md)
- [RHAIRFE-1141: Unified storage configuration for RHOAI components](https://redhat.atlassian.net/browse/RHAIRFE-1141)
- [RHAISTRAT-176: Connections 2.0](https://redhat.atlassian.net/browse/RHAISTRAT-176)
- [RHAISTRAT-2656: Unified storage configuration for RHOAI components](https://redhat.atlassian.net/browse/RHAISTRAT-2656)
- [Standardized Data Backbone proposal](https://docs.google.com/document/d/1oxlF_pwZG3sPkaogFBV_f1T9UOjW-uZsDswj5BDUMiQ)
- [CloudNativePG connection Secret contract](https://github.com/cloudnative-pg/cloudnative-pg/blob/main/pkg/specs/secrets.go)
- [Kubernetes PersistentVolumeClaim documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

## Reviews


| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
