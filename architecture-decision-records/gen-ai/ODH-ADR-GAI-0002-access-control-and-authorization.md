# Access Control and Authorization Architecture

| Attribute      | Value      |
| -------------- | ---------- |
| Date           | 2025-12-02 |
| Scope          | Gen AI (gen-ai package) |
| Status         | Draft |
| Authors        | TBD |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-40682](https://issues.redhat.com/browse/RHOAIENG-40682) |

## What

This ADR documents the access control and authorization architecture for the gen-ai system, including Kubernetes RBAC integration, SubjectAccessReview (SAR) implementation, authorization middleware chain, permission requirements per endpoint, and the relationship between authentication and authorization. The system enforces namespace-scoped authorization to ensure users can only access resources they are permitted to use in a multi-tenant environment, leveraging OpenShift's native RBAC capabilities.

