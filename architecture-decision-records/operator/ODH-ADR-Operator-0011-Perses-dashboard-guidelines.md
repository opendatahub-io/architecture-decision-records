# Open Data Hub - ODH-ADR-Operator-0011 - Perses Dashboard Guidelines

|                |                                  |
| -------------- | -------------------------------- |
| Date           | 2025-12-05                       |
| Scope          | RHOAI Platform                   |
| Status         | Draft                            |
| Authors        | [Dayakar Maruboena](@dayakar349) |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        | RHOAIENG-25596                   |
| Other docs:    | None                             |

## What

This ADR defines guidelines for creating Perses dashboard specifications for Red Hat OpenShift AI (RHOAI) components. It establishes naming conventions for datasources, dashboard creation workflows using Kubernetes Custom Resources, and provides an overview of dashboard deployment patterns managed by the OpenDataHub operator.

## Why

OpenShift AI component teams need standardized guidance for creating dashboard specifications to visualize component metrics. Perses provides a modern, cloud-native dashboard framework that integrates with RHOAI's observability infrastructure through Kubernetes Custom Resource Definitions (CRDs). Without standardized guidelines, teams face inconsistent dashboard implementations, naming conflicts, and integration challenges with the centralized monitoring stack.

## Goals

- Establish naming conventions for Prometheus and Tempo datasources used in Perses dashboards
- Define standard workflows for creating Perses dashboard specifications as Kubernetes Custom Resources
- Document the declarative dashboard deployment pattern managed by the OpenDataHub operator
- Enable component teams to independently create dashboard specifications
- Ensure consistency across RHOAI dashboard implementations

## Non-Goals

- Define specific metrics that components should expose
- Document dashboard UI integration and embedding patterns
- Implement or maintain centralized dashboard infrastructure
- Support dashboard frameworks other than Perses
- Provide migration paths from existing Grafana/Prometheus dashboards to Perses

## How

### Recommended Approach: Declarative Dashboard Specifications

**We recommend creating Perses dashboards using Kubernetes Custom Resources (CRs)** as the primary method for dashboard deployment in RHOAI.

This approach is preferred because:

- **Declarative Management**: Dashboards are managed as Kubernetes resources with full lifecycle support
- **Operator Integration**: The OpenDataHub operator automatically deploys and manages dashboard resources
- **Version Control**: Dashboard specifications can be stored in Git alongside component code
- **Consistency**: Ensures uniform deployment patterns across all RHOAI components

### Implementation Details

#### 1. Datasource Naming Conventions

The OpenDataHub operator creates the following standard datasources that dashboards should reference:

| Datasource Name                      | Type       | Purpose                        |
|--------------------------------------|------------|--------------------------------|
| `data-science-prometheus-datasource` | Prometheus | RHOAI platform metrics         |
| `tempo-datasource`                   | Tempo      | Distributed tracing data       |

**Naming Guidelines:**

- Use lowercase with hyphens as separators
- Include descriptive identifiers in datasource names
- Use `-datasource` suffix for datasource names

**Note**: The datasource names listed above match the OpenDataHub operator's implementation and are managed by the operator. Component teams should reference these exact datasource names in their dashboard specifications.

#### 2. Dashboard Specification Structure

Component teams create dashboards as `PersesDashboard` Custom Resources in the OpenDataHub operator's monitoring resources directory: `internal/controller/services/monitoring/resources/`.

**Dashboard Naming Convention:**
- Template files: `perses-{component}-dashboard.tmpl.yaml`
- Dashboard metadata name: `data-science-{component}-{purpose}`
- Example: `perses-tempo-dashboard.tmpl.yaml` creates `data-science-tempo-traces` dashboard

**Reference Template - Tempo Tracing Dashboard:**

Component teams can use the following real-world example from the OpenDataHub operator as a starting template for their own dashboards:

```yaml
apiVersion: perses.dev/v1alpha1
kind: PersesDashboard
metadata:
  name: data-science-tempo-traces
  namespace: {{.Namespace}}
spec:
  display:
    name: "RHOAI Distributed Tracing"
  duration: "1h"
  layouts:
    - kind: Grid
      spec:
        display:
          title: "Trace Visualization"
        items:
          - x: 0
            y: 0
            width: 24
            height: 12
            content:
              $ref: "#/spec/panels/traces"
  panels:
    traces:
      kind: Panel
      spec:
        display:
          name: "Distributed Traces"
        plugin:
          kind: TraceTable
          spec: {}
        queries:
          - kind: TraceQuery
            spec:
              plugin:
                kind: TempoTraceQuery
                spec:
                  query: "{}"
                  datasource:
                    kind: TempoDatasource
                    name: tempo-datasource
```

**Key Requirements:**

- **Location**: Dashboard templates must be placed in `internal/controller/services/monitoring/resources/`
- **File naming**: Use `perses-{component}-dashboard.tmpl.yaml` pattern
- **Namespace templating**: Use `{{.Namespace}}` for dynamic namespace injection
- **Datasource references**: Must reference operator-managed datasources (`data-science-prometheus-datasource` or `tempo-datasource`)
- **Panel structure**: Define panels in `spec.panels` and reference them in `spec.layouts` using `$ref`

#### 3. Dashboard Deployment Pattern

The OpenDataHub operator manages dashboard deployment through the Monitoring service configuration.

**Deployment Flow:**

1. **Component Team**: Creates dashboard specification as `PersesDashboard` CR
2. **Operator**: Validates required Perses CRDs exist (`PersesDashboard`, `PersesDatasource`, `Perses`)
3. **Operator**: Deploys dashboard resources to the monitoring namespace
4. **Perses Platform**: Renders dashboards based on deployed specifications
5. **Dashboard UI**: Integrates Perses dashboards (implementation covered in RHAISTRAT-113)

**Operator Validation:**

The operator performs atomic deployment validation:
- Checks for Perses CRD availability
- Validates datasource references exist
- Ensures monitoring configuration is enabled
- Sets appropriate status conditions on failures

### Architecture Overview

The Perses dashboard architecture in RHOAI consists of:

1. **Component Level**: Teams create `PersesDashboard` and `PersesDatasource` CRs
2. **Operator Level**: OpenDataHub operator deploys and manages Perses resources
3. **Platform Level**: Perses platform renders dashboards from CR specifications
4. **Data Layer**: Datasources query Prometheus/Tempo for metrics and traces
5. **UI Integration**: Dashboard UI displays Perses dashboards (RHAISTRAT-113)

```text
[Component Dashboard CR] → [ODH Operator] → [Perses Platform] → [Datasource (Prometheus/Tempo)]
                                                    ↓
                                           [Dashboard UI]
```

## Open Questions

None at this time.

## Alternatives

N/A

## Security and Privacy Considerations

- **Datasource Authentication**: Datasource proxies must use proper authentication for Prometheus/Tempo access
- **Network Policies**: Dashboard components must respect NetworkPolicy restrictions
- **RBAC Integration**: Dashboard visibility must align with Kubernetes RBAC policies
- **Sensitive Metrics**: Components must not expose sensitive data (credentials, PII) in metric labels
- **Resource Access**: Perses CRs should be created in appropriate namespaces with proper ownership

## Risks

- **CRD Dependency**: Requires Perses operator to be installed and healthy
- **Operator Coupling**: Dashboard deployment depends on OpenDataHub operator functionality
- **API Changes**: Perses API changes may require dashboard specification updates
- **Dashboard Discovery**: Mechanism for dashboard UI to discover available dashboards requires coordination (RHAISTRAT-113)
- **Version Compatibility**: Perses version must be compatible with dashboard specifications

## Stakeholder Impacts

| Group                        | Impact                                                        |
| ---------------------------- | ------------------------------------------------------------- |
| Component Development Teams  | Must create PersesDashboard CRs following naming conventions  |
| Platform Operations          | Manage Perses operator and monitoring infrastructure          |
| Dashboard UI Team            | Integrate Perses platform into RHOAI dashboard                |

## References

- [Perses Documentation](https://perses.dev)
- [Perses Kubernetes CRDs](https://github.com/perses/perses-operator)
- [ODH-ADR-Operator-0009: Observability and Tracing Strategy](ODH-ADR-Operator-0009-observability-tracing-strategy.md)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
