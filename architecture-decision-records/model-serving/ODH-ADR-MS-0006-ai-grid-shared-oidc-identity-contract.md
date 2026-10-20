# Open Data Hub - AI Grid Shared-OIDC Identity Contract

|                |            |
| -------------- | ---------- |
| Date           | 2026-10-19 |
| Scope          | model-serving, AI Grid identity contract |
| Status         | Draft |
| Authors        | Lindani Phiri, Pierangelo Di Pilato |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | RHAI-10604 |
| Other docs:    | none |

## What

Define the shared-OIDC identity contract for AI Grid users on MaaS. One OIDC issuer and one MaaS tenant serve multiple Grid tenants. Authorization uses organization-scoped OIDC claims and tenant-qualified group names. The gateway validates the broker-signed token, injects trusted identity headers, and creates no Kubernetes identity for tenant users.

## Why

AI Grid needs a bring-your-own-broker identity model that works across Grid tenants without creating a Kubernetes identity for each user. A shared MaaS tenant keeps the deployment simple, while organization-scoped claims preserve tenant isolation. OIDC is an explicit business requirement, and API keys remain the supported path for programmatic access after SSO sign-in.

This contract also keeps MaaS independent of a specific identity provider. Keycloak is a reference implementation, but any OIDC provider that emits the agreed claims can be used.

## Goals

* Use one shared OIDC issuer for all AI Grid users in a deployment.
* Use one shared MaaS tenant for all Grid tenants in that deployment.
* Treat the `iss` and `sub` claim pair as the stable user identity.
* Use the `https://redhat.com/maas/orgs` claim to carry organization membership and roles.
* Use tenant-qualified group names to prevent group-name collisions between Grid tenants.
* Authorize each request only in the organization that the request names.
* Validate OIDC tokens at the gateway before MaaS receives the request.
* Inject trusted identity headers only after the gateway validates the token.
* Strip client-supplied identity headers before the gateway injects trusted values.
* Preserve API-key authentication and existing SSO-based MaaS APIs.
* Create no Kubernetes identity for AI Grid tenant users.

## Non-Goals

* Deploy or federate the identity provider.
* Define the complete OIDC token shape beyond the claims MaaS consumes.
* Define the Envoy filter chain or `kube-auth-proxy` deployment.
* Define the consumer UI gateway layout.
* Implement the shared PostgreSQL isolation schema.
* Add per-tenant OIDC configuration.
* Replace API-key authentication for inference and tools.

## How

### Identity provider

AI Grid uses one shared OIDC issuer per deployment. The issuer signs JWT access tokens for all Grid tenants in that deployment. MaaS trusts that issuer and maps its claims to a MaaS user identity and one or more Grid organizations.

The JWT contains:

* `iss`: the OIDC issuer.
* `sub`: the stable subject identifier at that issuer.
* `aud`: the intended MaaS audience.
* `iat` and `exp`: the token issue and expiry times.
* `email`: an optional display and notification identity.
* `groups`: tenant-qualified group names.
* `https://redhat.com/maas/orgs`: organization memberships and roles.

The following example shows one valid token shape:

```json
{
  "iss": "https://identity.example.com/",
  "sub": "00u7a1b2c3d4e5f6g7h8",
  "aud": "maas-api",
  "iat": 1791288000,
  "exp": 1791291600,
  "email": "alex@grid-tenant-1.com",
  "groups": [
    "42c3e46f-2477-44d7-a85b-d3b43f6b31fa:ai-grid-admins",
    "42c3e46f-2477-44d7-a85b-d3b43f6b31fa:ai-grid-users"
  ],
  "https://redhat.com/maas/orgs": [
    {
      "organization_id": "42c3e46f-2477-44d7-a85b-d3b43f6b31fa",
      "organization_name": "coke",
      "roles": ["admins"]
    }
  ]
}
```

The `iss` and `sub` pair is the canonical user identity. Email is not a stable identity because a user can change it. Although the organization claim is an array, the MVP limits each user to one organization.

### Organization claim

Each entry in `https://redhat.com/maas/orgs` contains:

* `organization_id`: the stable Grid tenant identifier.
* `organization_name`: an optional display name.
* `roles`: the user's roles in that organization.

The authorization service checks that the requested organization appears in this claim. It then applies only the roles and groups for that organization. It does not apply groups or roles from one organization to another.

### Group claim

The `groups` claim uses tenant-qualified group names. JWT examples use `<organization-id>:<group>`, and MaaS CRD examples use `<organization-id>/<group>`. Both forms prevent two Grid tenants from colliding when they use the same local group label.

Group membership controls model and subscription access. The `admins` organization role grants additional administrative authority inside that organization only.

### Gateway validation and headers

The gateway validates the OIDC token before it forwards a request to MaaS. After validation, it injects trusted identity headers derived from the verified claims. Before injection, it removes any client-supplied copy of those headers. It also removes the `Authorization` header after validation so downstream services cannot receive the caller's bearer token.

MaaS trusts only gateway-injected identity headers. It does not trust identity headers supplied by a client. This keeps the trust boundary at the gateway and prevents a user from spoofing another identity.

This ADR defines the trust and meaning of the identity headers. RHAI-10605 defines the gateway layout that injects them.

### API access

SSO requests use the verified OIDC identity. Existing MaaS APIs remain usable with SSO-based access, subject to the organization-scoped authorization rules described in this ADR.

API keys remain the supported path for scripts and tools. A user signs in with SSO and creates an API key through the entitled subscription. API-key requests do not inherit SSO admin claims. Admin authority applies only to the SSO path.

### No Kubernetes identity

AI Grid tenant users do not receive a Kubernetes `User`, `Group`, `ServiceAccount`, `RoleBinding`, or `ClusterRoleBinding`. MaaS stores and authorizes these users through its own identity and organization model.

This keeps tenant identity independent of cluster RBAC and avoids creating Kubernetes objects for every external user.

## Alternatives

### Use per-tenant OIDC issuers and per-tenant MaaS tenants

This option gives each Grid tenant a separate issuer, MaaS tenant, and policy set. It provides strong isolation, but it duplicates deployment and identity configuration for every tenant. It also blocks the organization-based model that AI Grid requires.

### Create Kubernetes identities for tenant users

This option uses Kubernetes users, groups, and role bindings. It reuses Kubernetes RBAC, but it couples external tenant identity to the cluster. It also creates many Kubernetes objects and expands the cluster's security surface.

### Validate OIDC tokens in the BFF instead of the gateway

This option keeps token validation close to the UI, but it leaves API routes without a single enforcement point. It also makes identity behavior differ between the UI and API paths.

### Use API keys only

This option avoids OIDC entirely, but it does not meet the explicit OIDC business requirement. It also makes browser sign-in and organization-scoped administration harder.

## Security and Privacy Considerations

* The gateway accepts tokens only from the configured shared OIDC issuer.
* The gateway validates the token signature, audience, issuer, and expiry.
* The gateway strips client-supplied identity headers before it injects trusted values.
* The gateway strips the `Authorization` header after validation.
* Authorization applies groups and roles only within the requested organization.
* API-key requests do not inherit SSO admin claims.
* Tenant users do not receive Kubernetes identities.
* Email is optional and is not used as the stable user identifier.

## Risks

* A shared OIDC issuer increases the blast radius if its signing key is compromised.
* Incorrect claim mapping can grant access to the wrong organization.
* A failure to strip inbound identity headers can permit identity spoofing.
* Tenant-qualified group names must remain unique and stable.
* Existing MaaS APIs must preserve owner, API-key, and platform-admin behavior.

## Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| ----- | ------------ | ---- | --------- |
| MaaS and Gateway team |  |  | Yes |
| AI Grid product team |  |  | Yes |
| Identity platform team |  |  | Yes |
| Security and privacy reviewers |  |  | Yes |

**Notes:**

* The MaaS and Gateway team owns token validation, trusted-header injection, and claim-based authorization.
* The AI Grid product team owns the Grid tenant and organization model.
* The identity platform team owns the OIDC provider and claim mapping.
* Security reviewers must approve the header, session, and failure behavior.

## References

* [AI Gateway tenancy ADR](./ODH-ADR-MS-0003-ai-gateway-tenancy.md).
* [MaaS subscription architecture ADR](./ODH-ADR-MS-0003-maas-subscription-architecture.md).
* [RHAI-10604](https://redhat.atlassian.net/browse/RHAI-10604).

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
