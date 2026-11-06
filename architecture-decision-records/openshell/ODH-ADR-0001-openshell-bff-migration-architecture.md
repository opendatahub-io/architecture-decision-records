# Architecture and Downstream Sync Strategy for Upstream Openshell Dashboard BFF

|                |            |
| -------------- | ---------- |
| Date           | 2026-09-03 |
| Scope          | Openshell Dashboard |
| Status         | Not Approved |
| Authors        | [Derek Xu](mailto:derxu@redhat.com) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

# Definitions

| Router/Multiplexer | Component that routes requests to a specified handler |
| :---- | :---- |
| Handler | Transport-related translator (e.g. HTTP) to business logic |
| Repository/Service | Business logic |
| Model | A representation of an object or concept that handlers and services abide by |
| Middleware | Functions run before or after handlers (e.g. authorization) |
| Client | Wrappers to talk to external APIs. These can be wrapped in a repository or separated into client factories and injected into context per request. |

# What

Proposal for changes to the Openshell Dashboard Go BFF in the upstream ([https://github.com/NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell)) and how downstream ([https://github.com/opendatahub-io/odh-dashboard](https://github.com/opendatahub-io/odh-dashboard)) will reuse and modify upstream if needed.

## Key Points

- Struct embedding and decoration as a code pattern to maintain interoperability between upstream and downstream BFFs  
- `Chi` ([GitHub \- go-chi/chi: lightweight, idiomatic and composable router for building Go HTTP services](https://github.com/go-chi/chi)) for easier routing and middleware handling  
- A revamped file structure including the separation of handlers into separate objects to accompany interface decoration
- Downstream will need to have modifications; reuse will involve directly using upstream's packages and wrapping them to support multi-gateway authentication flows which is not an upstream requirement.

## Description

This architecture proposes the use of struct embedding and decorator patterns in order to facilitate extensibility in downstream codebases. The proposal offers a foundation for code reuse with flexibility for consumers and it relies on three primary mechanics:

- Accept interfaces and return structs: Upstream will return structs with private fields while downstream will embed these and write interfaces to accept them.  
- Dependency Injection: Functions and systems are built to accept any interface-abiding structure. This allows downstream consumers to inject their own custom logic, clients, or services into the upstream workflow.  
- Decoration: Embedding interfaces or structs into structs to allow new types to reuse or modify existing types.  
- Dependency Overlay & Extensibility: Upstream code provides configuration options (e.g., overriding default HTTP clients or startup commands) and consumer-defined extensibility hooks (e.g., passing a parameter to inject a custom router).

# Why

As the openshell dashboard is planned to be integrated into the ODH dashboard, there will be the need to handle development on multiple streams. With that, code syncing becomes cumbersome especially if the process involves forking and rebasing. With the possibility that upstream and downstream requirements will diverge, it is paramount that some flexibility be ingrained into the codebase from the start to allow downstream users to have a healthy amount of code reuse while with an easy path to modify code without having to go through complicated rebases. This approach facilitates those needs and the proposed file structure also builds on top of this by abstracting every major component of the API stream (i.e. handlers, services, clients, etc.).

- The upstream team can build, iterate, and refactor their internal logic without fear of breaking downstream applications, as long as the core interface contracts remain intact.  
- Consumers aren't locked into rigid upstream choices and can import exposed packages, reuse the boilerplate that works for them, and completely override or modify the specific components they need to change.

The proposed code patterns and the use of `Chi` are common convention while also offering a complement to code decoration by promoting separation of concerns, dependency injection, least privilege, and an easier routing system. These additions would also help the project standalone with easier testing capabilities and less boilerplate.

While this proposal focuses primarily on the upstream BFF for the OpenShell Dashboard, the architecture facilitates downstream BFF development if it is required in the future by exposing API components.

# Goals

- Developing a file structure to properly organize Go BFF projects to allow for future extensibility and ease of development.  
- Develop infrastructure based around interfaces, struct embedding, and dependency injection in an upstream environment to allow downstream to reuse and modify upstream code with ease.  
- Propose code patterns for upstream that have minimal impact on future development while keeping extensibility for downstream.

# Non-Goals

- Creating a standard for new BFF projects (file structure and code patterns are not rules).  
- Defining all necessary functions of the new BFFs for Openshell.  
  - Only proposing a pattern for upstream and downstream synchronization with related examples.
- Defining the downstream BFF specification
  - Current plans involve reusing the upstream BFF for now

# How

There needs to be public (not in `/internal`) interfaces and models upstream.

- Public interfaces, models, etc. live in `/pkg`  

**Note**: Each subsection supports and gives examples of how interface decoration lives in each component of the API.

## File Pattern

#### API Layer Based: Separate functions of the API into packages

```
backend/
cmd/
 server/
  main.go
pkg/
handlers/
 sandbox_handler.go
services/
 service1/
service1.go 
middleware/
models/
repositories/
internal/
Reused code that is not meant to be imported directly by others. These would probably mostly be used within default service implementations. You could also put the actual default implementations in here and leave the constructors in pkg/. This depends on if you need those to be public (uppercase) for the entire upstream project but still unimportable for consumers
 config/
 etc.
```

### main.go

Only starts the server and listens for SIG\_TERM or other system interrupts to handle soft-termination (i.e. `server.Close()`).

- Instantiates a server object and starts the listening process  
- Can handle the configuration and env variable steps

### server.go

This file should be a simple Server constructor that loads configuration (or in `main.go`) and instantiates services, handlers, etc. [`main.go`](http://main.go) should create this server object and start listening.

- Instantiates services with specified dependencies (e.g. `NewService(dep1, dep2)`)  
- Handlers instantiated with services and any other required dependencies(e.g. `NewHandler(newService)`)  
- Clients or Repositories should be instantiated and passed to the relevant services.  
  - This includes components such as K8s or Openshell SDK.  
- Houses all routes grouped with `Chi`. Doubly serves as an API “spec” since all routes are noted in the file.

Example:

```go
package server

import (
    "net/http"
    "time"

    "github.com/go-chi/chi/v5"
)

type ServerConfig struct {
    Port         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    IdleTimeout  time.Duration
}

type Server struct {
    srv *http.Server
}

type Services struct {
    Gateway  services.Gateway
    Provider services.Provider
}

func NewServer(cfg ServerConfig, svcs Services) *Server {
    r := chi.NewRouter()

    r.Use(chimiddleware.RequestID)
    r.Use(chimiddleware.Logger)
    r.Use(chimiddleware.Recoverer)

    gatewayHandler := handlers.NewGatewayHandler(svcs.Gateway)
    sandboxHandler := handlers.NewSandboxHandler(svcs.Sandbox)

    r.Route("/api", func(r chi.Router) {
        r.Use(apiSpecificMiddleware)

        r.Get("/gateway/1", gatewayHandler.route1)
        r.Post("/gateway/2", gatewayHandler.route2)
        r.Get("/sandbox", sandboxHandler.route1)
    })

    httpServer := &http.Server{
        Addr:         cfg.Port,
        Handler:      r,
        ReadTimeout:  cfg.ReadTimeout,
        WriteTimeout: cfg.WriteTimeout,
        IdleTimeout:  cfg.IdleTimeout,
    }

    return &Server{srv: httpServer}
}
```

## Handlers

Handle any routing based code such as HTTP headers, Kubernetes labels, annotations, etc. The separation between service functions should be distinct.

Each handler should have a constructor and its own set of dependencies. Code *could* attach all handlers to a single instance such as `App` as well but separating handlers out lets different handlers have their own set of tools (least privilege) and makes unit testing easier. To share common data amongst all, have a higher-level struct with some base fields such as a logger and have every “lower-level” struct embed that higher level struct.

```go
// handlers/handler.go
type Handler struct {
  logger slog.Logger
}

// handlers/userhandler.go
type UserHandler struct {
  *Handler
  userService services.UserService // service interface
  // fields specific to UserHandler
}

func NewUserHandler(serviceParameter) *UserHandler {
  return &UserHandler{
    Handler: &Handler{logger: slog.Default()} ,
    userService: service
  }
}
// You can use userHandler.logger now (first class field)
```

### Another example

```go

func (h *GatewayHandler) CreateGateway(w http.ResponseWriter, r *http.Request) {
    // can be a model
    var req struct {
        Name string `json:"name"`
    }

    // validate model
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    // Pass r.Context() down into the service for request scoped variables
    provider, err := h.gatewayService.CreateProvider(r.Context(), req.Name)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(provider)
}
```

- We use the `net/http` context to (in a limited manner) store and retrieve request scoped variables from middleware and control for timeouts and cancellation so cancellations propagate downwards.

### Downstream Usage

Ideally, if a downstream BFF were to be created, that BFF would use `Chi` as well. Though, if `httprouter` use was required, downstream can leverage how both `Chi` and `httprouter` routing systems are built on top of stdlib's `net/http`. Due to this, handlers can be used together though with some friction requiring adapter functions. If this friction is unwanted, the handlers can be written separately but reuse of other components such as the service layer can be reused.

For example, if a `Chi` handler uses a router specific `chi.URLParam` as opposed to `net/http`'s `r.PathValue`, then the following adapter would be required:

```go
func ChiParamAdapter(next http.HandlerFunc) httprouter.Handle {
 return func(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
  chiCtx := chi.NewRouteContext()

  for _, p := range ps {
   chiCtx.URLParams.Add(p.Key, p.Value)
  }

  ctx := context.WithValue(r.Context(), chi.RouteCtxKey, chiCtx)
  next.ServeHTTP(w, r.WithContext(ctx))
 }
}
```

## Service / Business Logic

Where the business logic lives. This includes SDK calls, DB access, external service connections, etc.

Each service should define a public interface, constructor, and type (struct) while struct fields can be left private so as to not allow direct usage of struct fields and tighter coupling.

This pattern would look like:

```go
package services

import "context"

type Provider struct {
    ID   string
    Name string
}

type ProviderService interface {
    GetProvider(ctx context.Context, id string) (*Provider, error)
    CreateProvider(ctx context.Context, name string) (*Provider, error)
}

type DefaultProviderService struct {
    client OpenshellClient
}

func NewDefaultProviderService(sdk OpenshellClient) *DefaultProviderService {
    return &DefaultProviderService{
        client: sdk,
    }
}

// Default Implementations...

```

If downstream decides they want to modify functionality, they can opt for struct embedding and decoration or selectively rewriting certain functions. As long as these downstream structures satisfy upstream’s interfaces, upstream components can still be used.

As an example following from above:

```go
type DecoratedProviderService struct {
    // Embed the UPSTREAM INTERFACE 
    // This grants pass-through for GetProvider automatically.
    services.ProviderService 
    logger *slog.Logger
}

func NewDecoratedProviderService(base services.ProviderService) *DecoratedProviderService {
    return &DecoratedProviderService{
        ProviderService: base,
        logger:          slog.Default(),
    }
}

// We intercept CreateProvider, add our logic, and delegate the rest.
func (d *DecoratedProviderService) CreateProvider(ctx context.Context, name string) (*services.Provider, error) {
    d.logger.Info("Intercepted CreateProvider request downstream", "name", name)
    
    // Delegate to the embedded upstream interface
    provider, err := d.ProviderService.CreateProvider(ctx, name)
    if err != nil {
        return nil, err
    }

    // Downstream modification
    provider.Name = provider.Name + "-downstream-modified"
    return provider, nil
}
```

###

## Testing / Release / Branching Strategies

Reference: [https://go.dev/doc/modules/managing-dependencies](https://go.dev/doc/modules/managing-dependencies)

### Versioning

If there is no separate downstream BFF, downstream can pin to specific release images from upstream and reuse the executables built from those releases.

If a separate BFF is required, Go has a versioning system (go.mod) already so downstream can pin specific versions or commit SHAS it needs and prevent breakage if upstream updates.

Upstream needs to tag specific commits and downstream can pin to a SHA and version combination. Downstream can pin to a SHA regardless of if upstream has pinned versions.

**Upstream**

```shell
git add .
git commit -m "Add JSON logger module"

# Tag the commit with a SemVer version
git tag v1.0.0

# Push the tag to the remote repository
git push origin v1.0.0
```

**Downstream**

```
go get github.com/company/core-system@v1.0.0

# populated in go.mod
require github.com/company/core-system v1.0.0
```

`go.sum` is there to pin dependencies to a SHA to prevent tampering with version-commit switches.

Follow similar versioning practices outlined by Go: [https://go.dev/doc/modules/release-workflow](https://go.dev/doc/modules/release-workflow)
Release process: [https://go.dev/doc/modules/publishing](https://go.dev/doc/modules/publishing)

### Upgrading and Syncing

If no downstream BFF is required, the image version can be bumped to use a new release image from upstream. Contract tests can live in downstream to ensure that there are no breaking changes.

For a separate downstream BFF, version pinning within the `go.mod` file can be used to keep breaking changes from making their way into downstream automatically. Upgrading and managing changes across streams will be a manual process for now. CI to validate compatibility and perform version bumps is a future plan.

Upgrading is as simple as fetching another version or commit SHA.

```
go get github.com/profile/some_module@af044c0995fe
```

If the two streams diverge, downstream will have the ability to wrap and add or modify routes as needed. Certain handlers or services can be swapped out as long as their replacement conforms to their interfaces. If we need to fork the upstream BFF due to large divergences, we can use `replace` directives to reroute all dependencies. Ideally, this would be used as a temporary measure before getting changes merged into upstream.

go.mod

```
replace github.com/NVIDIA/OpenShell/bff => github.com/YourOrg/OpenShell-fork/bff v0.0.0-your-commit-hash
```

### Testing

Testing stays the same as other BFFs in the ODH Dashboard and with what is currently in the existing BFF. Each handler, service, etc. should get a `*_test.go` file. For e2e tests, the BFF can be deployed as a container and attached to a running Openshell instance either locally or on a cluster. This can be done via orchestration with a Makefile and by injecting different dependencies into the test files depending on the environment.

For managing different streams, contract tests that ensure that any routes required by downstream match the request and responses that the upstream BFF will be created to ensure updates will continue to work with the frontend in ODH.

## Auth, rate limits, and process management

To consolidate these requirements, the use of `context` becomes a natural choice. It is powerful in the sense that it, in Go’s words, “carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.” What this means is that we can use it to store authenticated data for a single request, manage timeouts, and propagate cancellations all the way down the call stack. It can also be used to track when child processes/threads exit.

The `net/http` package already has a context (`r.Context()`) when handlers implement the `ServeHTTP` interface with functions such as `func(w http.ResponseWriter, r *http.Request)`. Pass this context to service functions that take in a `context.Context` parameter and use details in business logic.

In the case of K8s clients and multi-tenant flows, instead of extracting user details directly, follow the auth flow of using factory patterns and per-request clients here: [https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/docs/adr/0010-kubernetes-client-architecture.md](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/docs/adr/0010-kubernetes-client-architecture.md)

- Uses two K8s clients, 1 service account and 1 user client, per request  
- Cluster-wide SA creates user clients per HTTP request

### Middleware

Middleware functions can be used to extract and validate tokens (assuming JWT). Middleware will then attach relevant information to the request (`r.context`) to use for functions later in the pipeline. Middleware might attach to context:

- Clients (K8s clients, Openshell SDK, etc.)  
- Tokens  
- Identity information for clients

```go
package middleware
// ...
func RequireAuth(next http.Handler) http.Handler {
 return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
  token := r.Header.Get("Authorization")
  if token != "Bearer secret-token" { // Replace with actual JWT logic
   http.Error(w, "Unauthorized", http.StatusUnauthorized)
   return
  }

  // If authorized, pass execution to the next handler in the chain
  next.ServeHTTP(w, r)
 })
}

```

With the help of `Chi`, middleware can be automatically attached to all subroutes like so:

```go
r := chi.NewRouter()
r.Use(middleware.Logger) // router-wide middleware
r.Route("/api", func(r chi.Router) {
r.Use(apiSpecificMiddleware) // /api specific middleware

 r.Route("/gateway", ...gateway subroutes)
 r.Route("/sandbox", ...sandbox subroutes)
})
```

# Alternatives

### A God App Object File Structure

This is where all handlers are attached to a single structure such as `App`.

| Pros | Cons |
| :---- | :---- |
| Simple, easy imports | God object holds too many dependencies Separate handlers have access to each others’ services Unit testing is difficult since the App has to be instantiated as a whole each time. |

### Git Rebasing/Forking

| Pros | Cons |
| :---- | :---- |
| Most flexibility, no boilerplate for upstream | Every update introduces possibility for new conflicts Diverging codebases becomes difficult to maintain No code contracts |

### Reusing upstream BFF (option being used for now)

| Pros | Cons |
| :---- | :---- |
| Easy and less to maintain | No custom logic and required to keep unwanted features in downstream. This is what is currently being used until the streams diverge too much. |

# Other considerations

## Client factories

Reference: [https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/docs/adr/0006-factory-pattern-client-management.md](https://github.com/opendatahub-io/odh-dashboard/blob/main/packages/gen-ai/docs/adr/0006-factory-pattern-client-management.md)

There will be a need to have another layer below business logic to instantiate clients. The above ADR defines a method to create real and mock clients based on interfaces. These clients are created with an authenticated user identity and injected into context during a request.

- This creates more interfaces and files to maintain but drastically reduces overhead when testing and creates a clear separation between business logic and infrastructure.

##

# Stakeholder Impacts

| Group | Key Contacts | Date | Impacted? |
| :---- | :---- | :---- | :---- |
| Openshell Dashboard | [Gage Krumbach](mailto:gkrumbac@redhat.com) [Derek Xu](mailto:derxu@redhat.com) [Daniel Reed](mailto:dareed@redhat.com) | | Yes |
| AgentOps (RHOAI Dashboard BFF) Razzmatazz | [Gage Krumbach](mailto:gkrumbac@redhat.com) [Derek Xu](mailto:derxu@redhat.com) [Daniel Reed](mailto:dareed@redhat.com) [Arsheen Taj Syed](mailto:arsyed@redhat.com) | <https://github.com/opendatahub-io/architecture-decision-records> | Yes |
| Openshell | | | No |

# Approvals

| [Bob Gregor](mailto:bgregor@redhat.com) | Not approved. Yet. Tag me back async with a comment/task, DM on slack when/if I am blocking the team. Thank you\! |
| :---- | :---- |
| [Gage Krumbach](mailto:gkrumbac@redhat.com) | |
| [Juntao Wang](mailto:juntao@redhat.com) | |
