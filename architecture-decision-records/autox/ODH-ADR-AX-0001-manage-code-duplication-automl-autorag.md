# Create `autox-core` Package to Address AutoML/AutoRAG Code Duplication

|                |            |
| -------------- | ---------- |
| Date           | 2026-04-24 |
| Scope          | AutoX (AutoML/AutoRAG) |
| Status         | Approved |
| Authors        | [daniduong](@daniduong) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-59342](https://redhat.atlassian.net/browse/RHOAIENG-59342) |
| Other docs:    | (see RHOAIENG-59342 for additional research/planning documents) |

## What

This ADR defines the strategy for managing code duplication between AutoML and AutoRAG packages, which share approximately 80% of their codebase while maintaining distinct product identities.

## Why

AutoML and AutoRAG are deployed as separate packages in a monorepo, each containing a UI layer and Backend-for-Frontend (BFF). While the packages are ~80% identical, they must maintain distinct identities including behavior, autonomy, and deployment independence. Without a clear strategy, we face either high maintenance burden from duplication or risk of inappropriate coupling that erodes product boundaries.

## Goals

* Maximize reuse of shared logic across UI and BFF layers
* Preserve clear product identity boundaries for AutoML and AutoRAG
* Maintain independent CI/CD pipelines per package
* Support gradual divergence as products evolve
* Keep future changes safe and predictable

## Non-Goals

* Merging AutoML and AutoRAG into a single deployable product
* Changing the existing deployment model
* Implementing runtime feature-flagging as the primary differentiation mechanism

## How

### BFF Layer

We will address the 80-90% code duplication in AutoML and AutoRAG BFF layers by creating `autox-core` as a shared services-only library.

**What gets shared**:
* Common Kubernetes, Pipelines, etc. logic extracted into reusable building blocks
* Clean service-layer functions with explicit request/response types (not an HTTP framework)
* Organized by feature domain (`kubernetes/`, `pipelines/`, etc.)

**What stays separate**:
* Each product maintains its own App structs and HTTP handlers
* Product-specific orchestration and business logic remain in AutoML/AutoRAG packages

**Architectural improvements**:
* Establish clean boundaries: handlers → services → clients
* Enable massive deduplication while preserving domain-specific customization flexibility

### UI Layer

We will address UI code duplication by creating `autox-core/ui` as a shared library of low-level composable primitives.

**What gets shared**:
* Low-level hooks and components that products compose into features
* Common utilities (validation helpers, formatters, domain logic)
* Organized by technical layer (components, hooks, layouts, utils, types) to support cross-feature composition

**What stays separate**:
* Each product maintains its own pages, routes, and navigation structure
* Product-specific feature composition and orchestration logic remain in AutoML/AutoRAG packages

**Architectural improvements**:
* Establish clear composition hierarchy: primitives → features → pages
* Module Federation singleton configuration ensures single runtime instance and optimized bundle size
* Avoid monolithic shared components with extensive prop variants in favor of composable primitives

### Local Development

* **BFF**: Go workspaces for seamless cross-package development
* **UI**: npm workspaces for seamless cross-package development
* No manual linking required

### Package Structure (tentative)

```
packages/autox-core/
├── services/
│   ├── kubernetes/             # Kubernetes integration
│   │   ├── models.go           # Domain models (Namespace, Secret, RequestIdentity)
│   │   ├── client.go           # K8s client interface
│   │   ├── factory.go          # Unified factory (handles static vs token auth)
│   │   ├── service.go          # KubernetesService (business logic)
│   │   ├── validators.go       # Input validation (namespace format, RBAC checks)
│   │   └── errors.go           # Custom errors (NotFoundError, ForbiddenError)
│   │
│   ├── pipelines/              # Kubeflow Pipelines integration
│   │   ├── models.go           # Domain models (PipelineRun, Pipeline, DSPA)
│   │   ├── client.go           # HTTP client for KFP API
│   │   ├── service.go          # PipelinesService (discovery, caching, CRUD)
│   │   ├── discovery.go        # DSPA discovery, pipeline discovery with LRU cache
│   │   ├── validators.go       # Request validation
│   │   └── errors.go           # Custom errors
│   │
│   ├── ...
│   │
│   └── common/                 # Shared utilities
│       ├── errors/             # Base error types and helpers
│       ├── validation/         # Generic validators (DNS-1123, URLs)
│       └── cache/              # LRU cache implementation
│
└── ui/
    ├── components/             # Shared UI primitives
    ├── hooks/                  # Shared hooks
    ├── layouts/                # Shared layouts
    ├── utils/                  # Validation, formatting, helpers
    └── types/                  # Shared TypeScript types
```

### Identity Layer

Each product package owns its identity through:
* **Configuration layer**: product-specific settings and feature configurations
* **Feature flags**: product-specific feature enablement
* **BFF orchestration/adapters**: product-specific API composition and orchestration logic

The shared `autox-core` library remains **identity-agnostic** and contains only reusable logic that applies to both products.

### Boundary Discipline

To prevent the shared layer from accumulating product-specific logic:
* Code reviews must verify that additions to `autox-core` are genuinely shared
* Product-specific behavior must live in the product packages, not in `autox-core`
* The shared library should expose extension points (configuration, hooks, adapters) rather than conditional logic
* Regular refactoring reviews to identify and extract incorrectly placed product-specific code

## Alternatives

### Alternative 1: Runtime Composition / Feature-Flag Driven Single Codebase

**Approach**: Deploy a single unified codebase where both products are controlled at runtime via feature flags, runtime configuration, and conditional UI/BFF behavior.

**Pros**:
* Zero duplication
* Fastest initial development velocity
* Easy to share fixes instantly across both products
* Simplifies shared logic by default

**Cons**:
* Weakens distinct identity requirement structurally
* Risk of hidden coupling via runtime branching
* Harder debugging due to conditional logic paths
* Deployment independence becomes artificial or reduced
* Can evolve into a flag-heavy monolith where product identities blur over time

**Rejected because**: This approach conflicts with the core requirement to maintain distinct product identities. Over time, runtime branching leads to tangled conditional logic that makes it difficult to diverge products safely. The structural separation provided by the recommended approach makes boundaries explicit and enforceable.

### Alternative 2: Shared Platform API + Module Federation UI Architecture

**Approach**: Introduce a separately deployed platform API service that handles cross-cutting concerns (auth, logging, telemetry, shared orchestration), while each product maintains its own BFF. UI components are shared via Module Federation with dynamically loaded modules.

**Pros**:
* Strong separation between platform and product concerns
* Enables reuse of UI components without duplication
* Preserves independent BFF evolution
* Scales well across multiple teams
* Encourages platform engineering maturity

**Cons**:
* High system complexity (API + federation layer + runtime integration)
* More moving parts in CI/CD and deployment
* Module Federation introduces runtime dependency coupling
* Harder local development experience
* Versioning and compatibility management overhead
* Additional dependency surface requiring teams to integrate platform API clients and configure federation remotes

**Rejected because**: While this approach offers architectural flexibility, it introduces significant operational and integration complexity that is not justified by current scale. The overhead of maintaining a separate platform API service, managing Module Federation runtime contracts, and coordinating versioning across multiple deployment units outweighs the benefits for two products. This approach should be reconsidered if ODH adds more similar products or if team boundaries require stronger runtime isolation.

### Comparison Summary

| Criterion | Shared Libraries (Chosen) | Runtime Composition | Platform API + Federation |
|-----------|---------------------------|---------------------|---------------------------|
| Code duplication | Low | Lowest | Low (UI only) |
| Identity separation | Strong | Weak | Strong |
| Deployment independence | High | Medium | High |
| System complexity | Medium | Low → Medium | High |
| Runtime risk | Low | Medium | Medium–High |
| Long-term maintainability | High | Medium | Medium |
| Organizational overhead | Low | Low | High |

## Security and Privacy Considerations

The `autox-core` shared library consolidates authorization and secret-handling logic common to both AutoML and AutoRAG. This reduces duplication of security-critical code, but requires explicit safeguards against shared-library failure modes.

### Authorization Composition Failure Modes

**Shared authorization code must prevent incorrect composition and cross-product privilege escalation:**

* **CWE-285 (Improper Authorization)** - Shared RBAC logic in `autox-core` must not allow product-specific policy bypasses. Example failure: AutoML grants access to a namespace based on role X, but AutoRAG incorrectly composes the same shared check with weaker preconditions, allowing unauthorized access.
* **CWE-863 (Incorrect Authorization)** - Shared token validation or SAR/SSAR checks must not introduce privilege escalation when products apply different authorization policies. Example failure: Shared code assumes namespace-scoped permissions, but one product incorrectly applies it to cluster-scoped resources.

**Requirements:**

1. **Authorization logic must be identity-agnostic**: Shared authorization code operates on generic permissions and roles, not product-specific policies. Product-specific authorization rules remain in product packages.
2. **Explicit composition contracts**: All shared authorization functions must document their assumptions (e.g., "assumes namespace-scoped access check") and validate inputs (e.g., reject cluster-scoped resource requests if only namespace-scoped checks are implemented).
3. **Mandatory product-policy tests**: Test suite must include scenarios exercising `autox-core` shared authz code with different product policies. Example: AutoML allows role X to create pipelines in namespace Y, AutoRAG denies role X in namespace Y — shared code must correctly enforce both policies.
4. **Security reviews apply to shared code**: Changes to authorization logic in `autox-core` affect both products. Reviews must verify no cross-product privilege escalation or policy bypass.

### Secret and Credential Handling

**Shared code handling tokens, secrets, and credentials must prevent leakage and enforce safe lifecycle:**

* **CWE-209 (Generation of Error Message Containing Sensitive Information)** - Error messages from `autox-core` must not expose tokens, secrets, Kubernetes API responses containing sensitive data, or internal system details.
* **CWE-532 (Insertion of Sensitive Information into Log File)** - Logs from `autox-core` must not contain tokens, secret material, user credentials, or unredacted Kubernetes resource content.

**Requirements:**

1. **Log redaction rules**:
   - Kubernetes tokens/bearer credentials: MUST be redacted before logging (replace with `[REDACTED]` or hash prefix for correlation).
   - Secret resource content: MUST NOT be logged. Log resource metadata only (name, namespace, type).
   - Pipeline API responses: MUST redact any embedded credentials or sensitive parameters before logging.
   - User identity headers: Log identity type (e.g., "token-based auth") but NEVER log raw token values.

2. **Non-retention and safe handling for secrets**:
   - Tokens/secrets used by Kubernetes/pipeline clients: MUST use short-lived credentials where possible. MUST NOT persist to disk or in-memory caches beyond request lifetime.
   - Secrets passed to pipelines: MUST be handled in-memory only during request processing. MUST NOT be stored in logs, debug output, or error messages.
   - Error handling: When validation or client operations fail, error messages MUST redact secret material. Example: "Invalid token" instead of "Invalid token: eyJhbGciOi...".

3. **Testing coverage**:
   - Unit tests must verify that error messages and logs from `autox-core` do NOT contain sensitive data when operations fail.
   - Integration tests must verify that shared Kubernetes/pipeline client code redacts tokens in logs and does not persist secret material.

### Deployment Independence

Each product is still deployed as separate container sets with independent security boundaries. Vulnerabilities in one product's deployment do not automatically affect the other. However, shared library vulnerabilities affect both products — all security patches to `autox-core` must be coordinated across AutoML and AutoRAG releases.

## Risks

**Medium risk: Shared layer accumulation of product-specific logic**

Without strict governance, the `autox-core` library may gradually accumulate product-specific logic disguised as shared utilities. This erosion of boundaries would undermine the benefits of structural separation.

**Mitigation**:
* Establish clear ownership and code review guidelines for `autox-core`
* Regular architecture reviews to identify boundary violations
* Document extension patterns that preserve identity-agnosticism
* Prefer configuration/composition over conditional logic in shared code

## Stakeholder Impacts

| Group                     | Key Contacts          | Date       | Impacted? |
| ------------------------- | --------------------- | ---------- | --------- |
| Dashboard Purple Team     | @daniduong, @chrjones | 2026-04-24 | Yes       |
| Dashboard Team (general)  | TBD                   | 2026-04-24 | Maybe     |
| Other ODH component teams | TBD                   | 2026-04-24 | Maybe     |

**Notes**:
* **Dashboard Purple Team**: Primary stakeholder. Owns AutoML and AutoRAG packages and will implement this refactoring. This decision directly affects their development workflow, release cadence, and maintenance burden.
* **Dashboard Team (general)**: Other dashboard developers may benefit from this pattern if they encounter similar duplication scenarios with future dashboard features or packages.
* **Other ODH component teams**: Teams building UI + BFF combinations in other ODH components (model-serving, workbenches, pipelines UI) may want to evaluate whether this pattern applies to their use cases. If a generic shared library emerges from this work, it could reduce duplication across the broader ODH dashboard ecosystem.

## References

* [RHOAIENG-59342](https://redhat.atlassian.net/browse/RHOAIENG-59342) - Jira ticket tracking this work

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------- | ------|
|                               |            |       |
