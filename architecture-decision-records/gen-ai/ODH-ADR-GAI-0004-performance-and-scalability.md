# Performance and Scalability Architecture

| Attribute      | Value      |
| -------------- | ---------- |
| Date           | 2025-12-02 |
| Scope          | Gen AI (gen-ai package) |
| Status         | Draft |
| Authors        | TBD |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-40684](https://issues.redhat.com/browse/RHOAIENG-40684) |

## What

This ADR documents the performance and scalability architecture for the gen-ai system, including caching strategy and implementation, HTTP client configuration and timeouts, connection management approach, stateless architecture benefits, and scalability considerations. The system implements an in-memory hierarchical cache, per-request client creation, and a stateless BFF design to support horizontal scaling while maintaining responsive performance for RAG operations and LLM interactions.

