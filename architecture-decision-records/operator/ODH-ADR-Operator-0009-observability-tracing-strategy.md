# Open Data Hub - ODH-ADR-Operator-0009 - Observability and Tracing Strategy

|                |                     |
| -------------- | ------------------- |
| Date           | 2025-08-05          |
| Scope          | Operator            |
| Status         | Draft               |
| Authors        | [den-rgb](@den-rgb) |
| Supersedes     | N/A                 |
| Superseded by: | N/A                 |
| Tickets        | RHOAIENG-25584      |
| Other docs:    | none                |

## What

This ADR defines the observability and distributed tracing strategy for OpenShift AI components, specifically addressing how traces are collected and forwarded to the OpenShift AI Tempo deployment for comprehensive monitoring and debugging capabilities.

## Why

OpenShift AI components need comprehensive observability through metrics, logs, and distributed tracing to enable effective monitoring, debugging, and performance analysis. Without a standardized tracing approach, teams face inconsistent instrumentation, difficult troubleshooting, and limited visibility into system behavior across component boundaries.

## Goals

- Establish a standardized approach for distributed tracing across all OpenShift AI components
- Provide predictable and consistent runtime environments for instrumented components
- Enable centralized trace collection and storage through Tempo integration
- Minimize overhead and complexity for component developers
- Support flexible storage backends (PV, S3, GCS) for trace data

## Non-Goals

- This ADR does not define metrics or logging strategies (only distributed tracing)
- This does not mandate specific OpenTelemetry language SDK implementations
- This does not define custom tracing instrumentation requirements beyond standard OpenTelemetry patterns
- This does not cover Perses integration for trace visualization

## How

### Recommended Approach: Native OpenTelemetry SDK Injection

**We recommend using "native" OpenTelemetry instrumentation via the `instrumentation.opentelemetry.io/inject-sdk: "true"` annotation** as the primary method for enabling distributed tracing in OpenShift AI components.

This approach is preferred because:

- **Predictable Runtime Environment**: Components using this annotation have a more predictable and consistent runtime environment
- **Better Performance**: Native SDK injection typically has lower overhead compared to auto-instrumentation
- **Greater Control**: Provides more control over instrumentation configuration and behavior
- **Stability**: More stable across different deployment scenarios and OpenShift versions

### Implementation Details

When enabling tracing for a component:

1. **Annotation-based Injection**: Component developers must manually add the following annotation to their deployment/pod specification to enable tracing:

   ```yaml
   metadata:
     annotations:
       instrumentation.opentelemetry.io/inject-sdk: "true"
   ```

2. **Instrumentation CR**: The OpenShift AI operator automatically creates an `Instrumentation` custom resource when tracing is enabled in the DSCInitialization CR:

   ```yaml
   apiVersion: opentelemetry.io/v1alpha1
   kind: Instrumentation
   metadata:
     name: {{.InstrumentationName}}
     namespace: {{.Namespace}}
   spec:
     exporter:
       endpoint: {{.OtlpEndpoint}}
     sampler:
       type: {{.SamplerType}}
       argument: "{{.SampleRatio}}"
   ```

3. **OpenTelemetry Collector**: The operator deploys an OpenTelemetry Collector that:

   - Receives traces via OTLP (gRPC and HTTP)
   - Processes traces with Kubernetes attributes and resource detection
   - Exports traces to the Tempo backend

4. **Tempo Integration**: Traces are automatically forwarded to the OpenShift AI Tempo deployment for storage and querying

### Configuration

Tracing is configured through the DSCInitialization CR:

```yaml
apiVersion: dscinitialization.opendatahub.io/v1
kind: DSCInitialization
metadata:
  name: default-dsci
spec:
  monitoring:
    traces:
      storage:
        backend: "pv" # or "s3", "gcs"
        size: "10Gi" # for PV backend
        # secret: "storage-credentials"  # required for s3/gcs
      sampleRatio: "0.1" # Sample 10% of traces (value between 0.0 and 1.0)
      tls:
        enabled: true # Enable TLS for Tempo gRPC connections
      exporters:
        # Optional: Define custom trace exporters for external observability tools
        otlphttp/jaeger:
          endpoint: http://jaeger-collector:4318/v1/traces
```

#### Configuration Fields

- **storage**: Defines the backend storage for trace data

  - `backend`: Storage type - "pv" (Persistent Volume), "s3" (AWS S3), or "gcs" (Google Cloud Storage)
  - `size`: Size allocation for PV backend
  - `secret`: Credentials secret name for S3/GCS backends

- **sampleRatio**: Sampling rate for traces (0.0 to 1.0)

  - Value of "0.1" means 10% of traces are sampled
  - Lower values reduce storage and processing overhead

- **tls**: TLS configuration for secure Tempo gRPC connections

  - `enabled`: When true, enables TLS for communication with Tempo

- **exporters**: Custom trace exporters for sending traces to external observability tools
  - Each key represents the exporter name (follows OpenTelemetry Collector naming conventions)
  - Values contain exporter-specific configuration following OpenTelemetry Collector format
  - Example: `otlphttp/jaeger` sends traces to a Jaeger collector via OTLP HTTP

### Architecture Overview

The OpenShift AI observability architecture consists of:

1. **Component Level**: Applications instrumented with OpenTelemetry SDK
2. **Collection Layer**: OpenTelemetry Collector for trace aggregation and processing
3. **Storage Layer**: Tempo for trace storage (supports PV, S3, GCS backends)
4. **Query Layer**: Integration with Perses for trace visualization

```
[Component with inject-sdk] → [OpenTelemetry Collector] → [Tempo] → [Perses]
```

## Open Questions

None at this time.

## Alternatives

### Language-Specific Auto-Instrumentation (Not Recommended)

While the OpenTelemetry Operator supports language-specific auto-instrumentation annotations, these are considered secondary options:

- `instrumentation.opentelemetry.io/inject-java: "true"` - Java auto-instrumentation
- `instrumentation.opentelemetry.io/inject-nodejs: "true"` - Node.js auto-instrumentation
- `instrumentation.opentelemetry.io/inject-python: "true"` - Python auto-instrumentation
- `instrumentation.opentelemetry.io/inject-go: "true"` - Go auto-instrumentation

**Note**: These language-specific annotations should only be used when the native SDK injection approach is not suitable for specific technical requirements.

### Manual SDK Configuration

Components could manually configure OpenTelemetry SDKs, but this approach increases complexity and reduces consistency across the platform.

## Security and Privacy Considerations

- Traces may contain sensitive information; appropriate sampling and filtering should be configured
- Network traffic between components and collectors should be secured
- Access to Grafana for trace viewing should be properly authenticated and authorized
- Storage backends should follow appropriate security practices (encryption at rest, access controls)

## Risks

- **Dependency on External Operators**: Requires OpenTelemetry and Tempo operators to be installed
- **Resource Overhead**: Additional resource consumption for collector and storage infrastructure
- **Debugging Complexity**: Issues with tracing infrastructure could impact observability across all components

## Stakeholder Impacts

| Group                       | Impact                                    |
| --------------------------- | ----------------------------------------- |
| Component Development Teams | Need to add annotations to enable tracing |
| Platform Operations         | Manage tracing infrastructure deployment  |

## References

- [OpenTelemetry Auto-Instrumentation](https://github.com/open-telemetry/opentelemetry-operator/tree/main?tab=readme-ov-file#opentelemetry-auto-instrumentation-injection)
- [Component Integration Guide](ODH-ADR-Operator-0003-component-integration.md)
- [OpenShift AI Monitoring Configuration Documentation](../../documentation/components/platform/README.md)

## Reviews

| Reviewed by | Date | Notes |
| ----------- | ---- | ----- |
