# Open Data Hub - Gateway API Authentication Architecture

|                |            |
| -------------- | ---------- |
| Date           | 2025-05-29 |
| Scope          | RHOAI Platform Authentication & Authorization |
| Status         | Approved |
| Authors        | [Lindani Phiri](@lphiri), [James Tanner](@jtanner) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHAISTRAT-927](https://issues.redhat.com/browse/RHAISTRAT-927), [RHAISTRAT-803](https://issues.redhat.com/browse/RHAISTRAT-803), [RHOAIENG-26418](https://issues.redhat.com/browse/RHOAIENG-26418) |
| Other docs:    | [BYOIDC Design](https://docs.google.com/document/d/1x5BZCbtUbSCe3OT3ojqF_Hn5QKbDS37EEPi1C834pf4/edit), [Migration Guide](https://github.com/jctanner/rhoai-hacking/blob/main/docs/KUBE-RBAC-PROXY.md) |

## What

This ADR documents the decision to adopt **Kubernetes Gateway API + kube-auth-proxy** as the unified authentication and authorization architecture for RHOAI 3.0, replacing the previous Route + oauth-proxy pattern and removing OpenShift Service Mesh as a platform dependency.

## Why

RHOAI faced three critical requirements: (1) support for external OIDC providers as OCP 4.19 adds external authentication, (2) inconsistent security implementations across components, and (3) Service Mesh complexity challenges. A working group (RHAISTRAT-927) evaluated alternatives and recommended focusing on BYOIDC as the architectural foundation.

## Goals

* Support both OpenShift OAuth and external OIDC providers with a single architecture
* Establish consistent authentication/authorization patterns across all RHOAI components
* Remove Service Mesh dependency to reduce operational complexity
* Implement end-to-end TLS encryption without requiring mTLS
* Enable single sign-on (SSO) across all RHOAI components
* Meet RH-SDL compliance requirements

## Non-Goals

* Implement Service Mesh features (advanced traffic management, mTLS between services)
* Maintain backward compatibility with RHOAI 2.x architecture
* Support OAuth-proxy sidecar pattern going forward

## How

The architecture consists of four components:

1. **Kubernetes Gateway API** - HTTPRoute CRs handle ingress routing to RHOAI services
2. **Istio Gateway (Envoy)** - Gateway implementation that processes HTTPRoutes and handles TLS termination
3. **kube-auth-proxy** - External authorization service supporting both OpenShift OAuth and external OIDC, queried via Envoy's ext_authz filter
4. **kube-rbac-proxy** - Authorization sidecar with TLS termination at the pod boundary

Traffic flows: `Client → Istio Gateway (Envoy) → [ext_authz check with kube-auth-proxy] → kube-rbac-proxy sidecar → Application`. The Envoy proxy uses an EnvoyFilter with ext_authz configuration to query the kube-auth-proxy service for authentication decisions before forwarding requests. kube-auth-proxy is NOT in the data path—it only validates credentials and returns allow/deny decisions to Envoy. TLS is enforced for every hop in the data path, terminating at the kube-rbac-proxy sidecar rather than implementing full mTLS (which would require Service Mesh).

All RHOAI components migrated from Route + oauth-proxy to HTTPRoute + kube-rbac-proxy for RHOAI 3.0. New components must follow this pattern: create HTTPRoute CRs referencing the platform Gateway and use kube-rbac-proxy for authorization.

Service Mesh was removed from RHOAI 3.0, including from KServe which previously required it.

## Alternatives

As part of RHAISTRAT-927 Milestone 2, five alternatives were evaluated via POCs:

**1. Service Mesh 2.x → 3.x Upgrade** (RHOAIENG-22474)
- Finding: Feasible but significant CRD changes, maintains Service Mesh complexity
- Rejected: Does not address external OIDC requirement, continues operational overhead

**2. Service Mesh 3.0 for authn/authz/mTLS**
- Finding: Provides mTLS but heavy resource overhead, difficult to automate as RHOAI building block
- Rejected: Complexity outweighs benefits, shared infrastructure control issues

**3. Gateway API for authn/authz** (RHOAIENG-24238)
- Finding: Gateway API specifications lack mature authentication support (April 2025 assessment)
- Decision: Use Gateway API for routing only, implement authentication separately

**4. oauth2-proxy for BYOIDC**
- Finding: Supports external OIDC but lacks OpenShift OAuth provider
- Rejected: Would require maintaining two separate auth solutions

**5. Reuse OCP console solutions**
- Finding: Not designed for reuse, unclear support model
- Rejected: Backlogged, not actively pursued

**Chosen**: Build kube-auth-proxy to support both OAuth and OIDC, use Gateway API for routing, implement TLS-to-pod without Service Mesh.

## Security and Privacy Considerations

**Security posture**: End-to-end TLS encryption for all external traffic, centralized authentication control, platform-enforced authorization via RBAC. TLS terminates at the kube-rbac-proxy sidecar; traffic within the pod to the application container is unencrypted HTTP.

**Limitation**: No mutual TLS between services (would require Service Mesh). Accepted as design decision - achieves encryption goals without Service Mesh overhead.

**Compliance**: Requires ProdSec validation (Owen Watkins) for RH-SDL compliance.

## Risks

**Technical**: New component (kube-auth-proxy) to maintain, no mTLS between services, upgrade complexity from 2.x to 3.0.

**Operational**: Loss of Service Mesh advanced features (A/B testing, canary deployments, rate limiting), kube-rbac-proxy configuration more complex than oauth-proxy CLI args.

**Mitigations**: kube-auth-proxy built on proven oauth-proxy codebase, comprehensive migration documentation provided, operator manages kube-rbac-proxy configuration via ConfigMaps, clear platform standards established for future components.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| ODH Platform         | Luca Burgazzoli, Lindani Phiri, James Tanner | May 2025 | Yes - core architecture change |
| All Component Teams  | Dashboard, Notebooks, KServe, DSP, DW, TrustyAI | May 2025 | Yes - migration to HTTPRoute + kube-rbac-proxy |
| QE | Scott Froberg, Andrej Smigala | May 2025 | Yes - new test plans, BYOIDC clusters |
| DevOps | Moulali Shikalwadi | May 2025 | Yes - Konflux onboarding for kube-auth-proxy |
| Docs | Breda McColgan, Em Turner | May 2025 | Yes - documentation rewrite |
| OpenShift | Derek Carr, Feng Pan, Marc Curry | May 2025 | Consulted - Gateway API alignment, OIDC integration |
| Product Security | Owen Watkins | May 2025 | Yes - RH-SDL validation required |
| Product Management | Jeff DeMoss | May 2025 | Yes - customer communication |

## References

**Working Group**:
- RHAISTRAT-927 - Working group formed Aug 2024, Milestone 3 recommendation (May 29, 2025): Focus on BYOIDC
- [ARB Proposal](https://docs.google.com/document/d/1WA7ewDxa8sEdiZ6yfoQdxLNzgLrloIVaR49iuo-VqlE/edit)

**Implementation**:
- RHAISTRAT-803 - BYOIDC feature (Oct 2024 - Nov 2025, closed with RHOAI 3.0 GA)
- RHAISTRAT-611 - Ongoing hardening for 3.3 GA
- RHOAIENG-30167 - Gateway API controller
- RHOAIENG-30171 - kube-auth-proxy (OAuth/OIDC handler)
- RHOAIENG-30168 - Component manifest migrations

**POC Evaluations**:
- RHOAIENG-22474 - Service Mesh 3.0 investigation
- RHOAIENG-24238 - Gateway API authentication POC

**Component Migrations**:
- RHOAIENG-32517 (Dashboard), RHOAIENG-32518 (Notebooks), RHOAIENG-34310 (Notebooks controller)
- RHOAIENG-32520 (KServe), RHOAIENG-34829 (TrustyAI)

**Documentation**:
- [BYOIDC Design Document](https://docs.google.com/document/d/1x5BZCbtUbSCe3OT3ojqF_Hn5QKbDS37EEPi1C834pf4/edit)
- [Migration Guide](https://github.com/jctanner/rhoai-hacking/blob/main/docs/KUBE-RBAC-PROXY.md)
- [ROSA BYOIDC Setup](https://docs.google.com/document/d/1tWM7UHhnA6BNeAJ25uqbjaiTIwc6y4U-KFUEILzXbdE/edit)

## Reviews

| Reviewed by                   | Date       | Approval | Notes |
| ----------------------------- | ---------  | -------- | ----- |
| Lindani Phiri                 |            |          | Working Group Leader |
| James Tanner                  |            |          | BYOIDC Lead |
| Luca Burgazzoli               |            |          | Platform |
| Derek Carr                    |            |          | OpenShift |
| Feng Pan                      |            |          | OpenShift |
| Jeff DeMoss                   |            |          | Product Management |
| Owen Watkins                  |            |          | Product Security - RH-SDL validation |
| Marc Curry                    |            |          | OCP Networking |
