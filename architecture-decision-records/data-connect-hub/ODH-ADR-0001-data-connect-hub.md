# Open Data Hub - Data Connect Hub

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | insert date |
| Scope          | |
| Status         | Approved |
| Authors        | Marius Danciu, Lukasz Cmielowski |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This proposal focuses on a specific part of the overall data strategy, specifically on analytic data ingestions and connections metadata management.

## Why

Although in RHOAI today we do see that certain connection types are available (S3, OCI etc.) and these connections are materialized as Kube secrets that other services can mount and use, in reality there are services that need to support a variety of data stores to ingest data. One example is EvalHub where test data can live in various storages (S3, HDFS, NFS, relational databases etc.). To support such an ecosystem the EvalHub service would need to integrate and maintain the entire stack of client libraries with their dependencies. This can be very time consuming to replicate in every service that needs to read analytic data from various sources. Thus the Data Connect Hub proposes to be a middleware service that other services can use (a single dependency) that facilitates data ingestion from multiple data stores.

## Goals

* Define the data ingestion flow for tabular and unstructured data.
* Define Connections management
* Define the tenancy model
* Define access management


## Non-Goals

* OTLP
* ETL
* Document processing
* Higher level governance aspects (lineage, catalog, etc)
* MCP server. Not in scope initially but we envision a similar approach with EvalHub, where DCH Operator can also manage a DCH MCP server along with the DCH service instance. 


## How

### Architecture

#### Abbreviations

- DCH = Data Connect Hub

#### High level overview
![Fig 1](./images/ADR-0001-img1.png)


- `DCH Operator` 
    - Managed by OpenDataHub operator as a new DSC component. 
    - When a new `DataConnectService` CR is created, the operator creates the DCH Service instance in a dedicated `infra` namespace. This operator watches all DCH CRs from the tenant namespace; however, some aspects of the `DataConnection` CR need to be exposed to the end users prior to data reading, as the clients need to specify which connection to use for reading. Here we are proposing the CQRS (Command Query Responsibility Segregation) pattern where the operator watches the `DataConnection` CRs and syncs only the properties that are visible to the end users to the DCH service instance via an internal REST API. This information is stored in the internal Postgres DB. Therefore, when the end users perform listing for the available connections via the public REST API, we are not hitting the Kube API server. The end-user queries are segregated from the admin management of the CRs. Another reason for this proposal is to avoid re-implementing Kube informers in the service layer, as this is already done in the operator. So as a summary:
    - End-users can only read DataConnection information (only properties that are meant for end-users) via REST APIs.
    - Tenant admins manage the DataConnection CRs via the Kube APIs.


- `DCH Service instance` - the tenant admins do not have direct access to the actual service instance. The reason is to separate the infrastructure components from the tenant management ones. Another reason is that all DCH service instances live in the same namespace in order to share the same internal Postgres metadata store instance. The Postgres secret will also live in the `infra` namespace so this cannot be shared with the tenant admins. (A similar approach is adopted by MaaS). 

#### Multi tenancy

We propose to build the DCH solution with multi-tenancy in mind right from its inception. This is a critical aspect that we see growing in the OpenShift ecosystem for other services (MaaS, EvalHub, MLFlow etc.). The core ideas for multi-tenancy are:

##### Hard tenancy

![Fig 2](./images/ADR-0001-img3.png)


##### Soft tenancy

![Fig 3](./images/ADR-0001-img4.png)

- **Hard-tenancy** - Each tenant has its own DCH service instance that points to a dedicated gateway. This means that the Praxis proxy is also dedicated. This means:
  - Each tenant gets its own hostname / url
  - DCH service instance is dedicated to that tenant
  - Traffic is isolated per tenant (no neighbour noise)
  - Different TLS certificates managed via cert-manager can be used per tenant.
  - DataConnectService CR is created by the tenant in the tenant namespace. This triggers the creation of the DCH service instance in the DCH infra namespace
  - Implies the existence of a tenants discovery service since each tenant implies a different host name
  - Can support separate metadata store per tenant.

- **Soft tenancy** - There is a single DataConnectService CR in the opendatahub namespace and this is managed by the cluster admin not by the tenant admins. This implies:
  - There is a single DCH service instance (that scales horizontally of course)
  - There is a single HttpRoute and a single Gateway. 
  - There is a single Praxis instance for all tenants.
  - All tenants use the same hostname / URL
  - Tenants can no longer be distinguished by the hostname. Instead we will require the presence of the `x-tenant-id` header. This is similar to EvalHub and MLFlow approaches for multi-tenancy.
  - Tenant namespaces need to be labeled with `dch.opendatahub.io/tenant` so that DCH Operator knows which namespaces to monitor. These are the namespaces where the `DataConnection`, `ConnectionSubscription` CRs are managed.
  - Tenants are segregated by a tenant-id field in the metadata store (Postgres)


- **Tenant admin persona** - has access to the tenant namespace and manages `DataConnection`, `ConnectionSubscription` and `DataConnectService` (for hard tenancy case) CRs. These CRs are described later on in this document.
- **Data consumer persona** - this can list available connections that this identity has access to and perform data ingestion via provided APIs, SDK or CLI.

##### Note
For the initial release we will adopt the soft tenancy approach as this implies less development and does not require a separate tenants discovery service.


#### Service overview
![Fig 4](./images/ADR-0001-img2.png)

This diagram is a zoom-in from the above one. It describes the main internal components of the DCH service and how multiple data source types can be supported.

### APIs

The DCH service instance exposes the following API types:

- `REST` API for:
    - reading/listing Connections information
    - chunked or full data reading for unstructured datasets.
    - small sample data reading mainly for UI rendering and connection test purposes.
- `Arrow-Flight` - a gRPC protocol for efficient tabular data reading in columnar format. This is suitable for SQL datastores but also for vector databases. 
- `CR` APIs - for tenant admin management.

### CR Admin APIs

- `DataConnectService` - describes the DCH service instance and configurations for this tenant. OIDC, hardware resources etc.
- `DataConnection` - defines the connections metadata to a particular store:
    - admin properties (never shared with end users)
        - url + port
        - credentials secret_ref
    - user visible properties
        - name
        - description
        - data source name
        - ...
- `ConnectionSubscription` - defines which users/groups can access what Connections with what limits.
    - subjects list
    - connection_refs list
    - limits

- Therefore a connection C1 is only visible to a user U1 if and only if U1 and C1 are defined in a subscription. 
- Limits are initially defined as request limits; however, we expect that in time we can support more metrics such as data volume limits etc.


### CR examples

```yaml
apiVersion: dataconnect.opendatahub.io/v1alpha1
kind: DataConnection
metadata:
  name: my_eval
  namespace: ai_trust
spec:
  description: Connection to the test data S3 bucket
  type: s3
  format: jsonl
  properties:
    size: 200MB
      
  admin:
    url: https://s3.amazonaws.com/test-data/prompts.jsonl
    region: us-east-1
    secret_ref: aws_ai_trust_credentials
    bucket: test-data
    path: prompts.jsonl
```

**Notes**
- The admin section contains properties that are never exposed via the public REST API to the end users.
- The properties field contains arbitrary key-value pairs that can be discovered by other processes, e.g. size of the data, missing fields information, etc.


```yaml
apiVersion: dataconnect.opendatahub.io/v1alpha1
kind: ConnectionSubscription
metadata:
  name: evals
  namespace: ai_trust
spec:
  description: Subscription for evals
  subjects:
    # Users, groups or service-accounts can be specified here. 
  - kind: user
    name: jim
  - kind: user
    name: karen
  connections:
    - name: my_eval
  ## Optional
  requests_limits:
    - limit: 30
      window: 1m

```

**Notes**
- If a DataConnection is not attached to a ConnectionSubscription, it is not usable for data reading.
- If the subjects contains a user with name `*` this is an explicit statement that the connection referenced by this subscription is **PUBLIC** and any user can access it. Thus any tenant can define public connections.
- Upon creating a DataConnection, the DCH system automatically performs sanity checks and reflects this in the CR status object. Example:

```yaml
apiVersion: dataconnect.opendatahub.io/v1alpha1
kind: DataConnection
metadata:
  name: my_eval
  namespace: ai_trust
spec:
  description: Connection to the test data S3 bucket
  type: s3
  format: jsonl
  properties:
    size: 200MB

  admin:
    url: https://s3.amazonaws.com/test-data/prompts.jsonl
    region: us-east-1
    secret_ref: aws_ai_trust_credentials
    bucket: test-data
    path: prompts.jsonl
  
  status:
    phase: ready
    endpoint: https://dch.myorg.com/v1/data/connections/my_eval
    conditions:
    - type: Ready
      status: True
      lastTransitionTime: "2026-07-15T10:05:00Z"
      reason: "bound"
      message: "This connection is bound to a subscription."
    - type: Pending
      status: True
      lastTransitionTime: "2026-07-15T10:02:00Z"
      reason: "not_bound"
      message: "This connection is not yet bound to a subscription."
    - type: Connected
      status: True
      lastTransitionTime: "2026-07-15T10:00:00Z"
      reason: "Connected"
      message: "Successfully connected to the target storage."

```



#### Access control 

As per the above CRs definition, it becomes obvious that Subscriptions are the mechanism for describing authorization aspects. But this is not enough as this is how tenant admins define relationships. Here is what happens under the hood: 

1. Authentication - If OIDC is configured in the `DataConnectService` CR, the client is expected to send the JWT token in the request. This gets validated by Authorino policy.
2. Authorization - As `ConnectionSubscription` CRs are created, the DCH operator manages Kuadrant AuthPolicy with Rego rules for describing the API access to different API endpoints.
3. Rate limiting - Similarly to authorization, the DCH operator creates Limitador RateLimitPolicy CR defining the rate limits for API access for the current authenticated identity.

This is the access control proposal for production use in OpenDataHub and RHOAI. However, for upstream usage/adoption of the service, this should work e2e without Kuadrant dependencies. Thus, for the access control flow we propose a sidecar container approach where requests are validated against currently defined `ConnectionSubscription` CRs.


##### What happens under the hood

1. Tenant admin creates the DataConnection CR
2. Tenant admin creates the ConnectionSubscription CR
2. DCHO (DataConnectHubOperator) creates the Authorino AuthPolicy and Limitador RateLimit policy. This is needed for REST and ArrwoFlight API access. 
3. DCHO creates the Role and RoleBinding for accessing the specific DataConnection CR by the subjects mentioned in the ConnectionSubscription.


#### Data Ingestion

DCH service proposes data ingestion APIs via arrow flight, or HTTP API. This ensures that the client using this API (or SDK) does not need to install and maintain various DB drivers or 3rd party libraries and this also implies that the actual database credentials are never exposed to these clients. However, while there are use cases where such API abstraction is very useful, in other contexts using the 3rd party libraries is preferable. DCH is not opinionated on the approach that clients want to adopt. However, there are implications that require mentioning here. If a client needs to use 3rd party libraries to connect directly to various data stores, it means that these clients require the actual backend credentials to connect. DCH service APIs are not used. However, the DCH Operator that watches DataConnection CRs will automatically create a new kube secret (if the DataConnection CR is configured as such) with the name `{dataconnection CR name}-secret` and this secret entails the DataConnection CR metadata + the actual credentials secret information that the DataConnection CR is pointing to. Visually this looks like:

```
  DataConnection CR + credentials secret = New secret
```

DCH Operator will ensure that the new secret will only be accessible by the subjects specified in the ConnectionSubscription CR. In other words, it will create and maintain the Role and RoleBinding for this new secret. In other words, if an application (i.e. a notebook) wants to mount this new secret, it needs to use a user or a ServiceAccount that is specified in the ConnectionSubscription CR. Without these, the Role and RoleBindings will not exist. 

It is of course possible for a user that has access to create Role and RoleBindings in this namespace to manually create these resources even if the ServiceAccount is not in the ConnectionSubscription CR. However, this is an explicit manual action that an admin can do, and this circumvents DCH.

As per the above notes, the DataConnection CR needs to be configured in order to allow the auto creation of the new secret. This a CR would need to contain the `autoCreateSecret` flag:

```yaml 
apiVersion: dataconnect.opendatahub.io/v1alpha1
kind: DataConnection
metadata:
  name: my_eval
  namespace: ai_trust
spec:
  autoCreateSecret: true

```
The reaason is that in cases where this is not needed, the system won't create and maintain stale secrets.


#### Observability

- All actions for CRs management are tracked by the DCH Operator
- All data access requests from data consumers are tracked by DCH service as OTEL logs and metrics.
- OTEL Logs are managed by Loki as the logging solution for platform observability.
- Similarly to MaaS, the DCH operator can also manage Perses dashboards for user consumption and usage awareness.


#### SDK 

DCH team will create SDKs for integrating with the DCH service. Since data analytics are primarily needed in the Python ecosystem, the first SDK should be a Python SDK. 

#### CLI

A CLI tool can be handy for integrating into workflows that involve bash scripts. It could also be used in init-containers for downloading datasets prior to the process startup (i.e. training, evals etc.). It can be used as a SKILL for agentic workflows such as Claude.


### Software stack

- DCH Operator - written in GoLang
- DCH Service - written in Rust
    - Actix-Web as the http layer.
    - OpenDAL for unstructured data fetching from various sources.
    - arrow-rs - for Arrow, arrow-flight integration


### MVP proposal

- All above CRs supported
- DCH Operator deployable standalone, not integrated as an ODH component
- DCH Service management as described in this doc
- REST APIs for end-users connections listing.
- Arrow-Flight APIs for tabular data reading.
- REST APIs for unstructured data reading.
- Access control implemented in a sidecar container - no Kuadrant dependencies yet. This does not have to be throwaway code because this can be very useful for upstream adoption when Kuadrant is not available. 
- A few tabular data sources, tabular data formats (parquet, csv, json) and object store data sources support (to be agreed with PM)

## Open Questions

TBD

## Alternatives

We did look for Go-based open source solutions and while they exist, we observe a significant growth of Rust open source libraries and frameworks (OpenDAL, arrow-rs, DataFusion etc.). Combined with Rust benefits (zero-cost abstractions, no GC hence no GC spikes, and many others) these frameworks do offer low-level memory and IO optimizations and are difficult to ignore, especially when data transfer on the wire is very sensitive.
 

## Security and Privacy Considerations

By design we do separate credentials management and DataConnection CR. While the secrets themselves can be managed directly by admins for production use we do recommend the use of Vault so that the actual credentials secrets are managed by VSO.

Authentication and Authorization aspects are described above and together with tenants isolation by separate Gateways we offer a strong security foundation for both connections management and data access.

## Risks

Rust vs Go can be more challenging for developers that need to ramp up with Rust. However, with AI tools like Claude this can be mitigated significantly with both code and boilerplate generation but also a much faster learning curve for developers. 

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Data Connect Hub  | Monica Romila, Oronde Tucker, Marius Danciu, Lukasz Cmielowski   | date       | ? |


## References

* Actix-web - https://actix.rs/
* Arrow-rs - https://github.com/apache/arrow-rs 


## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
