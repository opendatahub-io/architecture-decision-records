# User Isolation and Multi-tenancy Architecture

| Attribute      | Value      |
| -------------- | ---------- |
| Date           | 2025-12-02 |
| Scope          | Gen AI (gen-ai package) |
| Status         | Draft |
| Authors        | TBD |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-40683](https://issues.redhat.com/browse/RHOAIENG-40683) |

## What

This ADR documents the user isolation and multi-tenancy architecture for the gen-ai system, including namespace-based isolation strategy, data separation mechanisms, hierarchical cache isolation (namespace → username → category → key), tenant resource management, and isolation guarantees. The system uses Kubernetes namespaces as the primary isolation boundary to ensure strict data separation between tenants while maintaining efficient resource utilization in a shared infrastructure environment.

