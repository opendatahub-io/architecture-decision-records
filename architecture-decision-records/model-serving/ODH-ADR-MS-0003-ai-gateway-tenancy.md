# Open Data Hub - Architecture Decision Record template

<!-- copy and paste this template to start authoring your own ADR -->
<!-- for the Status of new ADRs, please use Approved, since it will be approved by the time it is merged -->
<!-- remove this comment block too -->

|                |            |
| -------------- | ---------- |
| Date           | insert date |
| Scope          | |
| Status         | Work in progress |
| Authors        | Marius Danciu, Lindani Phiri, Jamie Land |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This ADR aims to define the multi-tenancy aspects of MaaS being a fundamental part of the AIGateway.

## Why

Currently MaaS is a single tenant system where all subscriptions and auth policies are managed in a single namespace and the only isolation is achieved by Kube RBAC. In reality organizations need to segregate access, management, compute resources, traffic for different purposes like internal departments. 

## Goals

* Define the meaning of a tenant for AI-Gateway
* Define how tenants are managed
* Define how access and assets (Kube CRs, database records) are isolated per tenant.

## Non-Goals

* Define the dashboard behaviour and UX design aspects in the multi-tenant context
* Define a RHOAI platform tenancy system (although this could be a precursor)

## How

### Control plane

![proposed architecture](./images/ODH-ADR-MS-0003-ai-gateway-tenancy-img-1.png)  

In the above diagram we describe the main CR objects required today and in the multi-tenancy concept. 

#### Tenant lifecycle management - RHOAI Admin role

##### Automatic vs. manual

We are proposing here that the tenant lifecycle management is automatic in the sense that the AIGateway CR is the main starting point. From here maas-controller handles the necessary preparations.

##### Tenant creation

1. The RHOAI admin creates an AI-Gateway CR object in the tenancy namespace (tentatively named ai-tenants). The AI-Gateway CR can be named to something more generic such as AI-Tenant so it can serve as a platform level tenancy object not just for the AI-Gateway/MaaS system. So naming consolidation needs TBD. However, for now, this document will refer to this CR as the AI-Gateway CR.

2. At this point the existing maas-controller can manage the AI-Gateway CR and in the future this can be moved to a higher level platform controller if needed. To minimize the amounts of changes this CR should use a more generic API group such as `tenancy.opendatahub.io/v1alpha1` (name TBD). Upon CR creation maas-controller will:
    - Create the maas tenant admin namespace . 
    - Create the Gateway CR 
    - Create the default MaasConfig CR in the tenant admin namespace. This CR can further be managed by a tenant admin user (or whomever the client decided to grant RBAC permissions) to further configure maas in the context of this tenant:
        - set the default api-key expiration time
        - configure observability
        - ...
    - Create the maas-api service instance. See [Option 2](#option-2)

##### Tenant deletion

To delete a tenant the admin only needs to delete the AIGateway CR. The following steps are performed automatically by the maas-controller:

1. The Gateway CR is deleted. This automatically triggers the Envoy PODs deletion as well.
2. The tenant admin namespace is deleted and consequently all namespace scoped CRs get deleted in this process
3. The CRs from the user namespace are not deleted. 
4. KServe controller updates the HttpRoutes status to `not ready` or `conflicted`. The routes are not deleted. Also the LlmInferenceService CR is not deleted but the status is updated.
4. The maas-controller needs to update the MaasModelRef status accordingly. 
5. The ExternalProvider pointing to a deleted gateway should have the status updated similarly to the HttpRoute status.

##### Tenant suspend & resume

This is not in scope for now. However it should be discussed if this is a requirement or not at some point.

#### Deploying models - model deployer role

We are not opinionated which namespace the clients choose to use to manage models. Thus they might want to have separate namespaces per tenants or use the same namespace for multiple tenants. 

1. User creates a LlmInferenceService CR and points this CR to an AI-Gateway. We support a single gateway to be associated and this gateway needs to be an ai-gateway. This has been extensively discussed and attaching to multiple gateways some being ai-gateways others not, can lead to a series of implications that explodes the complexity. This may be revisited later on after we have the basic tenancy system proven. It is important to realize that there is a 1 to 1 association between a tenant and a gateway. We often casually simplify this by stating `tenant == ai-gateway`.

2. KServe controller creates the HttpRoute and the InferencePool CRs for this LlmInferenceService

3. Model is deployed by KServe.

4. User creates the MaasModelRef CR that points to the LlmInferenceService CR

While the model is deployed and attached to the correct ai-gateway it is still not accessible via the ai-gateway because there are no auth and rate-limit policies created. 

#### Making models accessible - tenant admin role

1. From the tenant admin namespace, the admin creates the MaasAuthPolicy and the MaasSubscription CRs. The maas-controller further creates the Kuadrant AuthPolicy and TokenRateLimit (per-user rate limits) CRs. 

Now the model can be used for inference and also for discoverability using the /v1/models endpoint exposed by maas-api.


### Data plane

Identifying the tenancy at the dataplane level is done by the host name. Thus the pattern is `{tenant-name}.{domain}`. For example a `research.redhat.com` would indicate a tenant named `research`. 


#### Maas-api 

The maas-api service exposes the following capabilities via REST APIs:
- model discovery via /v1/models
- api key management (CRUD) via /v1/apikeys
- subscriptions listing via /v1/subscriptions

We know that a tenant is essentially a separate gateway using the host as `{tenant}/{domain}`. Thus we want the above apis to be exposed via the tenant URLs. Assuming the domain `acme.com` here are some examples:
- https://research.acme.com/api/v1/models
- https://redteam.acme.com/api/v1/apikeys
- https://dev.acme.com/api/v1/subscriptions

where `research`, `redteam` and `dev` are different tenants. We have 2 options here:

##### OPTION 1
1. There is one maas-api kube service with its own HttpRoute
2. Upon a tenant creation this HttpRoute is updated by the controller to be attached the tenant gateway. In this case the maas-api HttpRoute would point to 3 separate gateways as per the examples above.

    - PROS 
        - single instance a bit less cluster footprint. 
    - CONS 
        - requires updates to the HttpRoute upon gateways creation and deletion and more reconciliation work
        - At dataplane level the tenant name must be extracted from the host name either by the proxy and injected as an X-Tenant header or parsed by the service itself.
        - Authentication and authorization becomes more difficult because each tenant may have a separate OIDC thus the bearer token validation in the context of a tenant is more complex.


##### OPTION 2
1. Upon a tenant creation a new maas-api kube service is created.
2. It is attached to an HttpRoute and the HttpRoute is only attached to one tenant gateway. This means that for each tenant there is a separate maas-api kube service and PODs deployment "minted" for this tenant.

    - PROS
        - Easier to manage. For tenant deletion simply delete the Kube service, deployment, the HttpRoute and the gateway. Upon tenant creation, simply create a new maas-api service instance, the HttpRoute and associate it with the tenant gateway.
        - Segregated traffic, no neighbour noise.
        - Potentially use different certificates for different tenants.
        - No need to discriminate the tenant per host names since the maas-api service itself is "minted" at deployment time with the tenant name (via config or env_var) and it can only operate in the context of that tenant.
        - Easier database segregation later on. Currently we are assuming that there is a single MaaS DB instance and we discriminate the tenant by a separate column. This is already agreed upon. However if a customer want separate physical databases for some tenants, they can easily do this so that maas-api service starts up by pointing to a different database. 
    - CONS
        - A bit larger memory footprint since there is a separate maas-api service and deployment for each tenant.

##### OPTION 3
1. There is one maas-api kube service with its own HttpRoute and its own Gateway (not an AI Gateway)

The endpoints above would become something like:
- https://maas.acme.com/api/v1/models
- https://maas.acme.com/api/v1/apikeys
- https://maas.acme.com/api/v1/subscriptions


    - PROS 
        - single instance a bit less cluster footprint. 
    - CONS 
        - Unaware of tenancy. 
            - GET /v1/models would return all the models I (as a logged in user) have access regardless to which tenant/gateways it is shared/attached to.
            - GET /v1/subscriptions would list all subscriptions regardless of tenancy. This can be a security risk.
            - POST /v1/apikeys - apikeys creation would lead to apikeys creation with not information about the tenant. 


Based on the above, OPTION 2 seems to be more compelling and this is what we propose. 

Option 3 seems risky and incomplete as creating apikeys would requite potentially an extra header to determine the tenant. This contradicts the above approach where the tenant name should be part of the cannonical host name. Also customers may be skeptical of having APIs that work outside the tenancy

In fact when a new Gateway CR (gateway.networking.k8s.io/v1) is created Istio creates separate Envoy proxy PODs. So most of the benefits of option 2 above applies at the proxy level as well. 

### Inference

Since all policies are attached to the routes and the routes are attached to the tenant ai-gateway the inference traffic becomes simpler and more isolated. 

### Database 

Currently MaaS manages apikeys in a Postgres database. In the multi tenancy context a schema change is needed in order to add a new column tenant or gateway so that each api-key is scoped for that particular tenant. 

### Migration

- Current ApiKeys do not point to any tenant however the current model-as-a-service namespace can become the default tenant. So for the existing api-keys the DB is updated to this tenant name
- on migration the system will create the AIGateway CR to match with the existing namespace and gateway.
- > Other considerations should be added here

The expectation is that all models deployed prior to the upgrade will continue to work. 

## Open Questions

- Carefully decide CR and namespace naming so that there is limited impact for the future.
- OIDC configuration should be at AI-Gateway CR level since this is a configuration that will be needed not just for MaaS. 

## Alternatives

We had extensive discussions around two fundamental options:

1. Single gateway multiple tenants
2. Multiple gateways - one gateway per tenant

After lots of debates we are proposing in this ADR option 2 as option 1 creates a lot more complexity with regards to policies and http-routes management.

## Observability

- Any tenant action for creation, update, deletion need to be tracked via OTEL traces or logs.
- Any API call that currently emits OTEL traces needs to include the tenant/gateway information.
- Any Prometheus metrics or OTEL logs (i.e. maas token usage metrics) needs to include the tenant/gateway information.

## Security and Privacy Considerations

Multi tenancy has a crucial impact in the overall system security by further isolating access, traffic and system resources. 

## Risks

Optional section. Talk about any risks here.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| group or team name            | key contact name | date       | ? |


## References

* optional bulleted list

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
