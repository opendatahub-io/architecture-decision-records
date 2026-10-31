# Open Data Hub - AI Grid Shared-OIDC Identity Contract

|                |            |
| -------------- | ---------- |
| Date           | 2026-10-31 |
| Scope          | model-serving, AI Grid identity contract |
| Status         | Approved |
| Authors        | Lindani Phiri, Pierangelo Di Pilato |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | RHAI-10604 |
| Other docs:    | none |

## Revision history

* 2026-10-31: Flattened the identity claims. Replaced the
  `https://redhat.com/maas/orgs` array of objects with the flat claims `org_id`,
  `org_name`, and `roles`. Made the group claim plain on the wire. Clarified that the broker mints the organization identity.

## What

Define the shared-OIDC identity contract for AI Grid users on MaaS. One OIDC
issuer and one MaaS `AITenant` serve multiple AI Grid organizations. Authorization
uses flat, organization-scoped OIDC claims. The broker mints the organization
identity. The gateway validates the broker-signed token, injects trusted identity
headers, and creates no Kubernetes identity for organization users.

## Why

AI Grid needs a bring-your-own-broker identity model that works across AI Grid
organizations without creating a Kubernetes identity for each user. A shared MaaS
`AITenant` keeps the deployment simple, while organization-scoped claims preserve
organization isolation. OIDC is an explicit business requirement. OIDC builds on
existing support in MaaS. API keys remain the supported path for programmatic
access after SSO sign-in.

This contract also keeps MaaS independent of a specific identity provider. Keycloak
is the reference implementation, but any OIDC provider that emits the agreed claims
can be used.

## Goals

* Use the OIDC configuration of the single AI Grid `AITenant` and its `MaasTenantConfig`.
* Use one shared OIDC issuer for all AI Grid users in that `AITenant`.
* Use one shared MaaS `AITenant` for all AI Grid organizations in that `AITenant`.
* Treat the `iss` and `sub` claim pair as the stable user identity.
* Use the `org_id`, `org_name`, and `roles` claims to carry organization membership and roles.
* Use plain group names on the wire. Qualify them internally to `org_id/group_name` for comparison.
* Make the broker mint the organization identity. No upstream identity provider sets it.
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

AI Grid uses the OIDC configuration from its single `AITenant` and
`MaasTenantConfig`. That configuration names one shared OIDC issuer for all AI Grid
organizations in the `AITenant`. General MaaS deployments can configure a different
issuer for each `AITenant`. This ADR does not change that behavior.

The issuer signs OIDC ID tokens for all AI Grid organizations in the `AITenant`.
MaaS trusts that issuer and maps its claims to a MaaS user identity and one AI Grid
organization.

The OIDC ID token contains:

* `iss`: the OIDC issuer.
* `sub`: the stable subject identifier at that issuer.
* `aud`: the intended MaaS audience.
* `iat` and `exp`: the token issue and expiry times.
* `email`: an optional display and notification identity.
* `groups`: plain access group names.
* `org_id`: the AI Grid organization identifier.
* `org_name`: an optional organization display name.
* `roles`: the user's roles in the organization.

The following example shows one valid ID token shape:

```json
{
  "iss": "https://identity.example.com/",
  "sub": "00u7a1b2c3d4e5f6g7h8",
  "aud": "maas-api",
  "iat": 1791288000,
  "exp": 1791291600,
  "email": "alex@grid-org-1.com",
  "groups": ["ai-grid-admins", "ai-grid-users"],
  "org_id": "42c3e46f-2477-44d7-a85b-d3b43f6b31fa",
  "org_name": "acme",
  "roles": ["admins"]
}
```

The `iss` and `sub` pair is the canonical user identity. Email is not a stable
identity because a user can change it. The MVP limits each user to one
organization. The single `org_id` value makes this structural.

The broker mints `org_id` from the upstream identity provider binding. No upstream
identity provider sets `org_id`. This stops a tenant from faking an organization.

### Organization claim

Each token carries one organization:

* `org_id`: the stable AI Grid organization identifier.
* `org_name`: an optional display name.
* `roles`: the user's roles in that organization.

The authorization service resolves the requested organization from gateway-injected
context. It makes sure that the requested organization matches `org_id`. It then applies
only the roles and groups from this token. It does not apply groups or roles from
one organization to another.

The MVP defines one organization role: `admins`. Unknown roles do not grant
administrative authority. MaaS can retain unknown roles for forward compatibility,
but it must not use them to authorize operations.

Authorization is subscription-scoped. The service selects only subscriptions whose
organization ID matches `org_id` and whose subject groups match at least one group
from the token. Roles do not broaden inference eligibility.

### Group claim

The `groups` claim carries plain access group names, such as `ai-grid-users`. The
names are not organization-qualified on the wire.

For authorization, MaaS qualifies each group name internally to the form
`org_id/group_name` before it compares. For example, `ai-grid-users` in a token for
organization `42c3e46f-2477-44d7-a85b-d3b43f6b31fa` compares as
`42c3e46f-2477-44d7-a85b-d3b43f6b31fa/ai-grid-users`. The qualified form prevents
group-name collisions between AI Grid organizations.

Each group name is an opaque, case-sensitive, non-empty string. MaaS compares the
qualified string exactly and performs no trimming, case folding, or other
normalization. It rejects tokens that contain an empty group name or a duplicate
group name.

Group membership controls model and subscription access. The `admins` role grants
additional administrative authority inside that organization only.

In the MVP, an organization administrator can:

* List subscriptions they administer.
* Search and read API-key metadata for users in those subscriptions.
* Revoke API keys bound to those subscriptions.
* View observability for users and subscriptions they administer.

An organization administrator can not create an API key for another user, change
model access, or modify a subscription.

### Role assignment

For the MVP, roles come from the upstream identity provider. The broker maps an
upstream role or group claim into the `roles` claim. The mapping stays scoped to
the organization, because the broker still mints `org_id`. A customer can grant a
role only inside its own organization. In the MVP, an upstream provider that emits
no role or group claim can not grant the `admins` role.

Broker-managed roles are a future option. In that option, an organization
administrator records roles in the broker, through the MaaS portal or a broker
delegated-admin console. The broker then mints the `roles` claim from broker state.
This future option will support a provider that emits no roles.

### Gateway validation and headers

The gateway validates the OIDC token before it forwards a request to MaaS. After
validation, it injects trusted identity headers derived from the verified claims.
Before injection, it removes every client-supplied copy of those headers.

The identity-header contract is:

| Header | Source | Value |
| ------ | ------ | ----- |
| `x-maas-username` | `sub` | The stable subject identifier. |
| `x-maas-group` | `groups` | A JSON array of the user's groups. |
| `x-maas-orgid` | `org_id` | The organization ID. |
| `x-maas-roles` | `roles` | A JSON array of roles for the organization. |
| `x-forwarded-id-token` | Verified OIDC ID token | The verified token for the BFF and MaaS API. |

All values are exact strings. JSON array headers contain valid JSON. The gateway
does not trim, case-fold, or otherwise normalize values. It overwrites these
headers only after successful OIDC validation.

MaaS trusts only gateway-injected identity headers. It does not trust identity
headers supplied by a client. Gateway-to-MaaS and gateway-to-BFF traffic must use
mTLS, a restrictive `NetworkPolicy`, or an equivalent authenticated in-cluster
channel. Direct client access to MaaS and the BFF must be denied.

RHAI-10605 defines the gateway layout that injects these headers.

### API access

SSO requests use the verified OIDC identity. Existing MaaS APIs remain usable with
SSO-based access, subject to the organization-scoped authorization rules described
in this ADR.

For the browser SSO flow, the client presents the proxy session cookie. If the
request does not present an API key, the gateway validates the OIDC ID token.
The gateway removes the client-supplied `Authorization` header. The gateway forwards
the verified token through `x-forwarded-id-token`. The BFF calls the MaaS API
through the validating gateway.

API keys remain the supported path for scripts and tools. A user signs in with SSO
and creates an API key through the entitled subscription. The existing API-key
transport is `Authorization: Bearer sk-oai-*`. If the target IPP `ExternalModel`
uses `apiFormat: messages`, the gateway also accepts `x-api-key`. The gateway
accepts `x-api-key` in no other case.

Outside the UI path, an API-key request uses the existing
`Authorization: Bearer sk-oai-*` transport. If the target model uses
`apiFormat: messages`, the request can also use `x-api-key`. The gateway ignores
the proxy session cookie and SSO-derived identity headers. It rejects an invalid
API key and does not remove a recognized API-key `Authorization` header.

On the UI path, the proxy session cookie must be present. If an API key is also
present, the gateway validates both the session and the API key. If the API key is
supplied through `Authorization`, the gateway preserves and validates that header.
If the client supplies the API key through `x-api-key`, and the target model uses
`apiFormat: messages`, the gateway validates the key. API-key requests do not inherit SSO
admin claims. Admin authority applies only to the SSO path.

### No Kubernetes identity

AI Grid organization users do not receive a Kubernetes `User`, `Group`,
`ServiceAccount`, `RoleBinding`, or `ClusterRoleBinding`. MaaS stores and authorizes
these users through its own identity and organization model.

This keeps organization identity independent of cluster RBAC and avoids creating
Kubernetes objects for every external user.

A Kubernetes-backed identity, such as a platform admin, does not use the `roles`
claim. MaaS determines admin authority for a Kubernetes-backed identity on a
separate path. That path uses its Kubernetes groups.

## Alternatives

### Carry organization membership as an array of objects

The earlier version of this contract used a `https://redhat.com/maas/orgs` claim.
That claim held an array of objects, each with `organization_id`,
`organization_name`, and `roles`. The array allowed many organizations per token.
This version flattens the claim. The MVP limits each user to one organization. A
flat shape is also simpler for an OIDC provider to emit. The flat shape makes
one-organization-per-token structural.

### Use per-organization OIDC issuers and per-organization MaaS tenants

This option gives each AI Grid organization a separate issuer, MaaS `AITenant`, and
policy set. It provides strong isolation, but it duplicates deployment and identity
configuration for every organization. It also blocks the organization-based model
that AI Grid requires.

### Create Kubernetes identities for organization users

This option uses Kubernetes users, groups, and role bindings. It reuses Kubernetes
RBAC, but it couples external organization identity to the cluster. It also creates
many Kubernetes objects and expands the cluster's security surface.

### Validate OIDC tokens in the BFF instead of the gateway

This option keeps token validation close to the UI, but it leaves API routes
without a single enforcement point. It also makes identity behavior differ between
the UI and API paths.

### Use API keys only

This option avoids OIDC entirely, but it does not meet the explicit OIDC business
requirement. It also makes browser sign-in and organization-scoped administration
harder.

## Security and Privacy Considerations

* The gateway accepts tokens only from the configured shared OIDC issuer.
* The gateway validates the ID token signature, audience, issuer, and expiry.
* The broker mints `org_id` from the identity-provider binding, not from an upstream claim.
* The gateway strips client-supplied identity headers before it injects trusted values.
* The SSO flow strips client-supplied identity headers and non-API-key `Authorization` headers before it injects trusted values.
* Outside the UI path, the API-key flow uses `Authorization: Bearer sk-oai-*`. For an `apiFormat: messages` model, the flow can also use `x-api-key`.
* Outside the UI path, the API-key flow ignores the proxy session cookie and SSO-derived identity headers and preserves a recognized API-key `Authorization` header.
* If the proxy session cookie is present on the UI path, the API-key flow validates the API key.
* Gateway-to-MaaS and gateway-to-BFF traffic uses an authenticated in-cluster channel, and direct client access is denied.
* Authorization applies groups and roles only within the requested organization.
* Upstream-mapped roles affect only the organization that owns the identity provider.
* API-key requests do not inherit SSO admin claims.
* Organization users do not receive Kubernetes identities.
* Email is optional and is not used as the stable user identifier.

## Risks

* If the signing key of a shared OIDC issuer is compromised, the blast radius increases.
* Incorrect claim mapping can grant access to the wrong organization.
* A broker that assigns the organization from an upstream claim, instead of the identity-provider binding, lets a tenant fake an organization.
* A failure to strip inbound identity headers can permit identity spoofing.
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
