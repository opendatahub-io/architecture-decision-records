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
* Configure `autox-core` as a Module Federation shared singleton to prevent duplicate bundle loading
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

To prevent the shared layer from accumulating product-specific logic, we enforce boundaries through automated controls and manual review:

#### Automated Enforcement (CI Gates)

**Architecture Unit Tests** (Go layer):
* Use `go-arch` or custom import analysis to assert that packages under `autox-core/services/*` do not import from `automl/*` or `autorag/*`
* Tests fail if any code in `autox-core` imports product-specific packages
* Example test pattern:
  ```go
  // autox-core/services/arch_test.go
  func TestNoCrossProductImports(t *testing.T) {
      packages := analysis.LoadPackages("./...")
      for _, pkg := range packages {
          if strings.HasPrefix(pkg.Path, "autox-core/services/") {
              for _, imp := range pkg.Imports {
                  if strings.Contains(imp, "automl/") || strings.Contains(imp, "autorag/") {
                      t.Errorf("autox-core package %s imports product-specific package %s", pkg.Path, imp)
                  }
              }
          }
      }
  }
  ```

**Linting Rules** (Go layer):
* Custom linter or `golangci-lint` configuration to forbid imports from `automl/*` and `autorag/*` into `autox-core`
* Configured as error-level (not warning) to block merges
* Example `.golangci.yml` rule:
  ```yaml
  linters-settings:
    depguard:
      rules:
        autox-core-isolation:
          files:
            - "**/autox-core/**/*.go"
          deny:
            - pkg: "automl"
              desc: "autox-core must not import product-specific automl packages"
            - pkg: "autorag"
              desc: "autox-core must not import product-specific autorag packages"
  ```

**TypeScript/UI Linting** (UI layer):
* ESLint rule to forbid imports from `@odh-dashboard/automl` or `@odh-dashboard/autorag` into `autox-core/ui`
* Example ESLint configuration:
  ```js
  {
    "rules": {
      "no-restricted-imports": ["error", {
        "patterns": [{
          "group": ["@odh-dashboard/automl*", "@odh-dashboard/autorag*"],
          "message": "autox-core/ui must not import product-specific packages"
        }]
      }]
    },
    "overrides": [{
      "files": ["packages/autox-core/ui/**/*.ts", "packages/autox-core/ui/**/*.tsx"],
      "rules": {
        "no-restricted-imports": ["error", {
          "patterns": ["@odh-dashboard/automl*", "@odh-dashboard/autorag*"]
        }]
      }
    }]
  }
  ```

**CI/CD Integration**:
* Architecture tests and linting rules run as required CI checks on all PRs
* PRs that violate boundary rules are blocked from merging
* Test failures surface with clear error messages identifying the violating import

#### Manual Review Guidelines

* Code reviews must verify that additions to `autox-core` are genuinely shared
* Product-specific behavior must live in the product packages, not in `autox-core`
* The shared library should expose extension points (configuration, hooks, adapters) rather than conditional logic
* Regular refactoring reviews to identify and extract incorrectly placed product-specific code

### Extension Patterns

`autox-core` remains identity-agnostic by exposing extension points that products compose. Prefer these patterns over conditional logic:

#### Pattern 1: Configuration Objects

Shared services accept configuration objects that products control:

```go
// autox-core/services/pipelines/service.go
type PipelinesConfig struct {
    CacheTTL          time.Duration
    EnableAutoRefresh bool
    DefaultNamespace  string
    // Products extend with their own config wrapper
}

type PipelinesService struct {
    config PipelinesConfig
    client PipelinesClient
    cache  *lru.Cache
}

func NewPipelinesService(config PipelinesConfig, client PipelinesClient) *PipelinesService {
    return &PipelinesService{
        config: config,
        client: client,
        cache:  lru.New(config.CacheTTL),
    }
}
```

**Product usage** (AutoML):
```go
// packages/automl/bff/internal/services/pipelines.go
type AutoMLPipelinesConfig struct {
    autoxcore.PipelinesConfig
    EnableExperimentTracking bool // AutoML-specific
}

func NewAutoMLPipelinesService(cfg AutoMLPipelinesConfig) *AutoMLPipelinesService {
    coreService := autoxcore.NewPipelinesService(cfg.PipelinesConfig, client)
    return &AutoMLPipelinesService{
        PipelinesService: coreService,
        trackExperiments: cfg.EnableExperimentTracking,
    }
}
```

#### Pattern 2: Service Composition for Simple Operations

For simple operations with no product-specific logic, products call autox-core services directly from handlers:

```go
// autox-core/services/pipelines/service.go
type PipelinesService interface {
    ListPipelineRuns(ctx context.Context, httpClient *http.Client, baseURL string, req ListPipelineRunsRequest) ([]*PipelineRun, string, error)
    GetPipelineRun(ctx context.Context, httpClient *http.Client, baseURL string, req GetPipelineRunRequest) (*PipelineRun, error)
    DeletePipelineRun(ctx context.Context, httpClient *http.Client, baseURL string, req DeletePipelineRunRequest) error
}

type ListPipelineRunsRequest struct {
    Identity          *kubernetes.RequestIdentity
    Namespace         string
    PipelineVersionID string
    PageSize          int32
    PageToken         string
}
```

**Product usage** (AutoML) - thin handler delegates directly to autox-core:
```go
// packages/automl/bff/internal/api/pipeline_runs_handler.go
func (app *App) ListPipelineRunsHandler(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
    ctx := r.Context()
    
    // Extract entities from context (added by middleware)
    identity := ctx.Value(constants.RequestIdentityKey).(*k8s.RequestIdentity)
    namespace := ctx.Value(constants.NamespaceKey).(string)
    dspa := ctx.Value(constants.DSPAKey).(*pipelines.DSPA)
    discoveredPipeline := ctx.Value(constants.DiscoveredPipelineKey).(*pipelines.DiscoveredPipeline)
    
    // Parse HTTP parameters
    pageSize := parseInt32(r.URL.Query().Get("pageSize"), 10)
    pageToken := r.URL.Query().Get("pageToken")
    
    // Call autox-core directly (no product-specific logic needed)
    runs, nextToken, err := app.pipelinesService.ListPipelineRuns(
        ctx,
        app.httpClient,
        dspa.BaseURL,
        pipelines.ListPipelineRunsRequest{
            Identity:          identity,
            Namespace:         namespace,
            PipelineVersionID: discoveredPipeline.VersionID,
            PageSize:          pageSize,
            PageToken:         pageToken,
        },
    )
    if err != nil {
        app.handleError(w, err)
        return
    }
    
    // Map to API response and return
    app.WriteJSON(w, http.StatusOK, PipelineRunsResponse{
        Runs:          mapPipelineRunsToAPI(runs),
        NextPageToken: nextToken,
    }, nil)
}
```

#### Pattern 3: Product Domain Services for Complex Orchestration

For complex operations requiring product-specific orchestration, products create domain services that compose autox-core primitives:

```
packages/automl/bff/internal/services/
└── automl_pipeline_service.go    # AutoML-specific orchestration

packages/autorag/bff/internal/services/
└── autorag_pipeline_service.go   # AutoRAG-specific orchestration
```

**Product domain service** (AutoML):
```go
// packages/automl/bff/internal/services/automl_pipeline_service.go
type AutoMLPipelineService struct {
    pipelinesService pipelines.PipelinesService  // autox-core service
    storageService   storage.StorageService      // autox-core service
    logger           *slog.Logger
}

// AutoML-specific request with domain-specific fields
type CreateAutoMLPipelineRunRequest struct {
    Identity           *kubernetes.RequestIdentity
    Namespace          string
    DSPA               *pipelines.DSPA
    DiscoveredPipeline *pipelines.DiscoveredPipeline
    
    // AutoML-specific fields
    DatasetPath        string
    TargetColumn       string
    ProblemType        string  // classification, regression, time-series
    MaxTrials          int
    TrainingBudget     string
    S3Credentials      *storage.S3Credentials
}

func (s *AutoMLPipelineService) CreateAutoMLPipelineRun(
    ctx context.Context,
    httpClient *http.Client,
    s3Client *storage.S3Client,
    req CreateAutoMLPipelineRunRequest,
) (*pipelines.PipelineRun, error) {
    // 1. Product-specific validation
    if err := s.validateAutoMLParams(req); err != nil {
        return nil, err
    }
    
    // 2. Product-specific pre-checks (using autox-core storage service)
    exists, err := s.storageService.ObjectExists(ctx, s3Client, storage.ObjectExistsRequest{
        Bucket: extractBucket(req.DatasetPath),
        Key:    extractKey(req.DatasetPath),
    })
    if err != nil || !exists {
        return nil, errors.New("dataset not found")
    }
    
    // 3. Build product-specific pipeline parameters
    pipelineParams := map[string]string{
        "dataset_path":    req.DatasetPath,
        "target_column":   req.TargetColumn,
        "problem_type":    req.ProblemType,
        "max_trials":      strconv.Itoa(req.MaxTrials),
        "training_budget": req.TrainingBudget,
    }
    
    // 4. Delegate core operation to autox-core service
    run, err := s.pipelinesService.CreatePipelineRun(
        ctx,
        httpClient,
        req.DSPA.BaseURL,
        pipelines.CreatePipelineRunRequest{
            Identity:     req.Identity,
            Namespace:    req.Namespace,
            PipelineID:   req.DiscoveredPipeline.PipelineID,
            VersionID:    req.DiscoveredPipeline.VersionID,
            Name:         generateAutoMLRunName(req),
            Parameters:   pipelineParams,
        },
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create AutoML pipeline run: %w", err)
    }
    
    // 5. Product-specific post-processing
    s.logger.Info("Created AutoML pipeline run",
        "run_id", run.ID,
        "namespace", req.Namespace,
        "problem_type", req.ProblemType,
    )
    
    return run, nil
}

func (s *AutoMLPipelineService) validateAutoMLParams(req CreateAutoMLPipelineRunRequest) error {
    // AutoML-specific validation logic
    validTypes := []string{"classification", "regression", "time-series"}
    if !contains(validTypes, req.ProblemType) {
        return &common.ValidationError{
            Field:   "problem_type",
            Message: "must be classification, regression, or time-series",
        }
    }
    if req.MaxTrials < 1 || req.MaxTrials > 100 {
        return &common.ValidationError{
            Field:   "max_trials",
            Message: "must be between 1 and 100",
        }
    }
    return nil
}
```

**Handler delegates to domain service**:
```go
// packages/automl/bff/internal/api/pipeline_runs_handler.go
func (app *App) CreatePipelineRunHandler(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
    ctx := r.Context()
    
    // Extract entities from context
    identity := ctx.Value(constants.RequestIdentityKey).(*k8s.RequestIdentity)
    namespace := ctx.Value(constants.NamespaceKey).(string)
    dspa := ctx.Value(constants.DSPAKey).(*pipelines.DSPA)
    discoveredPipeline := ctx.Value(constants.DiscoveredPipelineKey).(*pipelines.DiscoveredPipeline)
    
    // Parse HTTP request body
    var apiReq CreateAutoMLRunAPIRequest
    if err := json.NewDecoder(r.Body).Decode(&apiReq); err != nil {
        app.badRequestResponse(w, r, "invalid request body")
        return
    }
    
    // Delegate to AutoML domain service (owns orchestration)
    run, err := app.autoMLPipelineService.CreateAutoMLPipelineRun(
        ctx,
        app.httpClient,
        app.s3Client,
        services.CreateAutoMLPipelineRunRequest{
            Identity:           identity,
            Namespace:          namespace,
            DSPA:               dspa,
            DiscoveredPipeline: discoveredPipeline,
            DatasetPath:        apiReq.DatasetPath,
            TargetColumn:       apiReq.TargetColumn,
            ProblemType:        apiReq.ProblemType,
            MaxTrials:          apiReq.MaxTrials,
            TrainingBudget:     apiReq.TrainingBudget,
            S3Credentials:      app.s3Credentials,
        },
    )
    if err != nil {
        app.handleError(w, err)
        return
    }
    
    // Map to API response and return
    app.WriteJSON(w, http.StatusCreated, mapPipelineRunToAPI(run), nil)
}
```

**Key benefits of this approach**:
- Products own complex orchestration logic (no forced standardization)
- Full flexibility for product-specific validation, pre-checks, parameter building
- Clear separation: handlers (HTTP) → domain services (orchestration) → autox-core services (primitives)
- Easy to test product logic in isolation from HTTP layer
- No hook proliferation or hidden control flow

#### Pattern 4: UI Composition Primitives

UI layer provides composable hooks and components that products orchestrate:

```typescript
// autox-core/ui/hooks/usePipelinesList.ts
export const usePipelinesList = (namespace: string, options?: PipelinesListOptions) => {
  const [data, loaded, error, refresh] = useFetchState(
    React.useCallback(
      async () => {
        const response = await fetch(`/api/pipelines?namespace=${namespace}`);
        return response.json();
      },
      [namespace]
    ),
    []
  );
  
  return { pipelines: data, loaded, error, refresh };
};
```

**Product usage** (AutoML):
```typescript
// packages/automl/frontend/src/pages/PipelinesPage.tsx
const PipelinesPage: React.FC = () => {
  const { namespace } = useAutoMLContext();
  
  // Compose shared primitive with product-specific behavior
  const { pipelines, loaded, error, refresh } = usePipelinesList(namespace);
  const { experiments } = useExperimentTracking(pipelines); // AutoML-specific
  
  // Product-specific filtering/transformation
  const filteredPipelines = React.useMemo(
    () => pipelines.filter(p => p.type === 'automl'),
    [pipelines]
  );
  
  return (
    <PipelinesList
      pipelines={filteredPipelines}
      experiments={experiments} // AutoML-specific prop
      onRefresh={refresh}
    />
  );
};
```

#### Example Package Layouts

**BFF Extension Layout**:
```
packages/automl/bff/
├── internal/
│   ├── services/                    # Product domain services (orchestration)
│   │   ├── automl_pipeline_service.go
│   │   └── automl_storage_service.go
│   ├── config/                      # Product-specific configuration
│   │   └── config.go
│   └── api/                         # HTTP handlers (thin, delegate to services)
│       ├── app.go                   # Owns autox-core services + domain services
│       ├── pipelines_handler.go     # Simple ops → autox-core; complex → domain service
│       └── middleware.go
└── ...

packages/autorag/bff/
├── internal/
│   ├── services/                    # AutoRAG domain services (different orchestration)
│   │   ├── autorag_pipeline_service.go
│   │   └── autorag_retrieval_service.go
│   ├── config/
│   │   └── config.go
│   └── api/
│       ├── app.go
│       ├── pipelines_handler.go
│       └── middleware.go
└── ...

packages/autox-core/services/
├── pipelines/                       # Shared service primitives
│   ├── service.go                   # Core operations (list, create, delete)
│   ├── models.go                    # Domain models (PipelineRun, DSPA)
│   ├── client.go                    # HTTP client for KFP API
│   ├── discovery.go                 # DSPA/pipeline discovery with caching
│   └── validators.go                # Input validation
├── kubernetes/                      # K8s integration primitives
│   ├── service.go
│   ├── client.go
│   ├── factory.go                   # Unified factory (static vs token auth)
│   └── models.go
└── storage/                         # S3 integration primitives
    ├── service.go
    ├── client.go
    └── models.go
```

**UI Extension Layout**:
```
packages/automl/frontend/src/
├── pages/                 # Product-specific pages
│   └── PipelinesPage.tsx
├── features/              # Product-specific feature composition
│   └── experiments/       # AutoML-specific feature
│       ├── ExperimentsList.tsx
│       └── useExperimentTracking.ts
└── config/                # Product-specific configuration

packages/autox-core/ui/
├── hooks/                 # Shared composable hooks
│   ├── usePipelinesList.ts
│   └── useKubernetesClient.ts
├── components/            # Shared UI primitives
│   ├── PipelineCard.tsx
│   └── StatusBadge.tsx
└── utils/                 # Shared utilities
    ├── validation.ts
    └── formatters.ts
```

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

### Alternative 2: Shared Platform API + Module Federation Remote UI Architecture

**Approach**: Introduce a separately deployed platform API service that handles cross-cutting concerns (auth, logging, telemetry, shared orchestration), while each product maintains its own BFF. UI components are shared via a new Module Federation remote that both products load at runtime.

**Pros**:
* Strong separation between platform and product concerns
* Enables reuse of UI components without duplication
* Preserves independent BFF evolution
* Scales well across multiple teams
* Encourages platform engineering maturity

**Cons**:
* High system complexity (API + new federated remote + runtime integration)
* More moving parts in CI/CD and deployment
* New Module Federation remote introduces runtime dependency coupling
* Harder local development experience
* Versioning and compatibility management overhead
* Additional dependency surface requiring teams to integrate platform API clients and configure remote loading

**Rejected because**: While this approach offers architectural flexibility, it introduces significant operational and integration complexity that is not justified by current scale. The overhead of maintaining a separate platform API service, creating and deploying a new Module Federation remote for shared UI, and coordinating versioning across multiple deployment units outweighs the benefits for two products. This approach should be reconsidered if ODH adds more similar products or if team boundaries require stronger runtime isolation.

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
