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

Define the shared-OIDC identity contract for AI Grid users on MaaS. One OIDC issuer and one MaaS `AITenant` serve multiple AI Grid organizations. Authorization uses organization-scoped OIDC claims and organization-qualified group names. The gateway validates the broker-signed token, injects trusted identity headers, and creates no Kubernetes identity for organization users.

## Why

AI Grid needs a bring-your-own-broker identity model that works across AI Grid organizations without creating a Kubernetes identity for each user. A shared MaaS `AITenant` keeps the deployment simple, while organization-scoped claims preserve organization isolation. OIDC is an explicit business requirement, building on existing support in MaaS. API keys remain the supported path for programmatic access after SSO sign-in.

This contract also keeps MaaS independent of a specific identity provider. Keycloak is a reference implementation, but any OIDC provider that emits the agreed claims can be used.

## Goals

* Use the OIDC configuration of the single AI Grid `AITenant` and its `MaasTenantConfig`.
* Use one shared OIDC issuer for all AI Grid users in that `AITenant`.
* Use one shared MaaS `AITenant` for all AI Grid organizations in that `AITenant`.
* Treat the `iss` and `sub` claim pair as the stable user identity.
* Use the `https://redhat.com/maas/orgs` claim to carry organization membership and roles.
* Use organization-qualified group names to prevent group-name collisions between AI Grid organizations.
* Authorize each request only in the organization that the request names.
* Validate OIDC tokens at the gateway before MaaS receives the request.
* Inject trusted identity headers only after the gateway validates the token.
* Strip client-supplied identity headers before the gateway injects trusted values.
* Preserve API-key authentication and existing SSO-based MaaS APIs.
* Create no Kubernetes identity for AI Grid organization users.

## Non-Goals

* Deploy or federate the identity provider.
* Define the complete OIDC token shape beyond the claims MaaS consumes.
* Define the Envoy filter chain or `kube-auth-proxy` deployment.
* Define the consumer UI gateway layout.
* Implement the shared PostgreSQL isolation schema.
* Add per-organization OIDC configuration.
* Replace API-key authentication for inference and tools.

## How

### Identity provider

AI Grid uses the OIDC configuration from its single `AITenant` and `MaasTenantConfig`. That configuration names one shared OIDC issuer for all AI Grid organizations in the `AITenant`. General MaaS deployments can configure a different issuer for each `AITenant`; this ADR does not change that behavior.

The issuer signs JWT access tokens for all AI Grid organizations in the `AITenant`. MaaS trusts that issuer and maps its claims to a MaaS user identity and one or more AI Grid organizations.

The JWT contains:

* `iss`: the OIDC issuer.
* `sub`: the stable subject identifier at that issuer.
* `aud`: the intended MaaS audience.
* `iat` and `exp`: the token issue and expiry times.
* `email`: an optional display and notification identity.
* `groups`: organization-qualified group names.
* `https://redhat.com/maas/orgs`: organization memberships and roles.

The following example shows one valid token shape:

```json
{
  "iss": "https://identity.example.com/",
  "sub": "00u7a1b2c3d4e5f6g7h8",
  "aud": "maas-api",
  "iat": 1791288000,
  "exp": 1791291600,
  "email": "alex@grid-org-1.com",
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

* `organization_id`: the stable AI Grid organization identifier.
* `organization_name`: an optional display name.
* `roles`: the user's roles in that organization.

The authorization service resolves the requested organization from gateway-injected context. It checks that the requested organization appears in this claim. It then applies only the roles and groups for that organization. It does not apply groups or roles from one organization to another.

The MVP defines one organization role: `admins`. Unknown roles do not grant administrative authority. MaaS may retain unknown roles for forward compatibility, but it must not use them to authorize operations.

Authorization is subscription-scoped. The service selects only subscriptions whose organization ID matches the requested organization and whose subject groups match at least one organization-qualified group from the token. Roles do not broaden inference eligibility.

### Group claim

The `groups` claim uses organization-qualified group names. The canonical wire format is `<organization-id>/<group>`.

Both parts are opaque, case-sensitive, non-empty strings. Neither part may contain `/`. Consumers compare the complete string exactly and perform no trimming, case folding, or other normalization. They reject tokens that contain an empty part, an extra separator, or a duplicate organization-and-group pair.

Group membership controls model and subscription access. The `admins` organization role grants additional administrative authority inside that organization only.

In the MVP, an organization administrator may:

* List subscriptions they administer.
* Search and read API-key metadata for users in those subscriptions.
* Revoke API keys bound to those subscriptions.
* View observability for users and subscriptions they administer.

An organization administrator may not create an API key for another user, change model access, or modify a subscription.

### Gateway validation and headers

The gateway validates the OIDC token before it forwards a request to MaaS. After validation, it injects trusted identity headers derived from the verified claims. Before injection, it removes every client-supplied copy of those headers.

The identity-header contract is:

| Header | Source | Value |
| ------ | ------ | ----- |
| `x-maas-username` | `sub` | The stable subject identifier. |
| `x-maas-group` | `groups` | A JSON array of organization-qualified groups for the selected organization. |
| `x-maas-orgid` | `https://redhat.com/maas/orgs[].organization_id` | The selected organization ID. |
| `x-maas-roles` | `https://redhat.com/maas/orgs[].roles` | A JSON array of roles for the selected organization. |
| `x-forwarded-access-token` | Verified OIDC access token | The verified token for the BFF and MaaS API. |

All values are exact strings. JSON array headers contain valid JSON. The gateway does not trim, case-fold, or otherwise normalize values. It overwrites these headers only after successful OIDC validation.

MaaS trusts only gateway-injected identity headers. It does not trust identity headers supplied by a client. Gateway-to-MaaS and gateway-to-BFF traffic must use mTLS, a restrictive `NetworkPolicy`, or an equivalent authenticated in-cluster channel. Direct client access to MaaS and the BFF must be denied.

RHAI-10605 defines the gateway layout that injects these headers.

### API access

SSO requests use the verified OIDC identity. Existing MaaS APIs remain usable with SSO-based access, subject to the organization-scoped authorization rules described in this ADR.

For the browser SSO flow, the client presents the proxy session cookie. When the request does not also present an API key, the gateway validates the OIDC access token, removes the client-supplied `Authorization` header, and forwards the verified token through `x-forwarded-access-token`. The BFF calls the MaaS API through the validating gateway.

API keys remain the supported path for scripts and tools. A user signs in with SSO and creates an API key through the entitled subscription. The existing API-key transport is `Authorization: Bearer sk-oai-*`. The gateway also accepts `x-api-key` only when the target IPP `ExternalModel` uses `apiFormat: messages`. AI Grid does not enable `x-api-key` unconditionally.

Outside the UI path, an API-key request uses the existing `Authorization: Bearer sk-oai-*` transport. If the target model uses `apiFormat: messages`, the request may also use `x-api-key`. The gateway ignores the proxy session cookie and SSO-derived identity headers. It rejects an invalid API key and does not remove a recognized API-key `Authorization` header.

On the UI path, the proxy session cookie must be present. If an API key is also present, the gateway validates both the session and the API key. If the API key is supplied through `Authorization`, the gateway preserves and validates that header. If the API key is supplied through `x-api-key`, the gateway validates it only when the target model uses `apiFormat: messages`. API-key requests do not inherit SSO admin claims. Admin authority applies only to the SSO path.

### No Kubernetes identity

AI Grid organization users do not receive a Kubernetes `User`, `Group`, `ServiceAccount`, `RoleBinding`, or `ClusterRoleBinding`. MaaS stores and authorizes these users through its own identity and organization model.

This keeps organization identity independent of cluster RBAC and avoids creating Kubernetes objects for every external user.

## Alternatives

### Use per-organization OIDC issuers and per-organization MaaS tenants

This option gives each AI Grid organization a separate issuer, MaaS `AITenant`, and policy set. It provides strong isolation, but it duplicates deployment and identity configuration for every organization. It also blocks the organization-based model that AI Grid requires.

### Create Kubernetes identities for organization users

This option uses Kubernetes users, groups, and role bindings. It reuses Kubernetes RBAC, but it couples external organization identity to the cluster. It also creates many Kubernetes objects and expands the cluster's security surface.

### Validate OIDC tokens in the BFF instead of the gateway

This option keeps token validation close to the UI, but it leaves API routes without a single enforcement point. It also makes identity behavior differ between the UI and API paths.

### Use API keys only

This option avoids OIDC entirely, but it does not meet the explicit OIDC business requirement. It also makes browser sign-in and organization-scoped administration harder.

## Security and Privacy Considerations

* The gateway accepts tokens only from the configured shared OIDC issuer.
* The gateway validates the token signature, audience, issuer, and expiry.
* The gateway strips client-supplied identity headers before it injects trusted values.
* The SSO flow strips client-supplied identity headers and non-API-key `Authorization` headers before it injects trusted values.
* Outside the UI path, the API-key flow uses `Authorization: Bearer sk-oai-*` and may use `x-api-key` only when the target IPP `ExternalModel` uses `apiFormat: messages`.
* Outside the UI path, the API-key flow ignores the proxy session cookie and SSO-derived identity headers and preserves a recognized API-key `Authorization` header.
* On the UI path, the API-key flow validates the API key only when the proxy session cookie is also present.
* Gateway-to-MaaS and gateway-to-BFF traffic uses an authenticated in-cluster channel, and direct client access is denied.
* Authorization applies groups and roles only within the requested organization.
* API-key requests do not inherit SSO admin claims.
* Organization users do not receive Kubernetes identities.
* Email is optional and is not used as the stable user identifier.

## Risks

* A shared OIDC issuer increases the blast radius if its signing key is compromised.
* Incorrect claim mapping can grant access to the wrong organization.
* A failure to strip inbound identity headers can permit identity spoofing.
* Organization-qualified group names must remain unique and stable.
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
* The AI Grid product team owns the AI Grid organization model.
* The identity platform team owns the OIDC provider and claim mapping.
* Security reviewers must approve the header, session, and failure behavior.

## References

* [AI Gateway tenancy ADR](https://github.com/opendatahub-io/architecture-decision-records/blob/main/architecture-decision-records/model-serving/ODH-ADR-MS-0003-ai-gateway-tenancy.md).
* [RHAI-10604](https://redhat.atlassian.net/browse/RHAI-10604).

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
|             |      |       |
