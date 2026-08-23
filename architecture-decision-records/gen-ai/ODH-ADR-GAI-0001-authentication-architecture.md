# Authentication Architecture

| Field          | Value      |
| -------------- | ---------- |
| Date           | 2025-12-02 |
| Scope          | Gen AI (gen-ai package) |
| Status         | Draft |
| Authors        | TBD |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-40681](https://issues.redhat.com/browse/RHOAIENG-40681) |

## What

This ADR documents the authentication architecture for the gen-ai system, including OAuth 2.0/OIDC integration with OpenShift, token extraction and validation mechanisms, authentication middleware chain, frontend authentication integration, and token lifecycle management. The gen-ai system operates as a Backend-for-Frontend (BFF) that integrates with OpenShift's authentication infrastructure to provide secure, token-based authentication for users accessing RAG (Retrieval Augmented Generation) and LLM interaction capabilities.

