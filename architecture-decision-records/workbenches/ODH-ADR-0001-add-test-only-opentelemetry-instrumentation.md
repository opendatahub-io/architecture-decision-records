# Open Data Hub - Add test-only OpenTelemetry tracing instrumentation


| Field          | Value                                                       |
| -------------- | ----------------------------------------------------------- |
| Date           | 2025-04-02                                                  |
| Scope          | Open Data Hub                                               |
| Status         | Draft                                                       |
| Authors        | [Jiri Danek](@jiridanek)                                    |
| Supersedes     | N/A                                                         |
| Superseded by  | N/A                                                         |
| Tickets        | N/A                                                         |
| Other docs     | N/A                                                         |

## What

Add OpenTelemetry tracing instrumentation to newly written and modified odh-notebook-controller code.

## Why

OpenTelemetry traces are a powerful observability signal that can assist with testing the product. 

Eventually, but not right now, we may even consider integrating with https://kubernetes.io/docs/concepts/cluster-administration/system-traces/ to trace across Kubernetes components.

## Goals

* Enable tracing only for testing (envtest testing) and not for production deployments.
* Allow writing tests that check for events in order to assert for otherwise invisible properties of the software. Such as whether a specific error condition (that is transparently retried later) has been hit.

## Non-Goals

* Enabling OpenTracing in production deployments. This can be considered later, separately.

## How

* https://github.com/opendatahub-io/kubeflow/pull/570
* When a TraceProvider is not configured, the OpenTelemetry library in Go defaults to noop trace provider
* TraceProvider is to be only set when running tests, which ensures that there is no active tracing in production deployments.

## Alternatives

1. Writing a test that reads and parses debug log messages is feasible, but suboptimal when OpenTelemetry is also an option.

## Stakeholder Impacts

| Group                 | Key Contacts      | Date       | Impacted? |
|-----------------------|-------------------|------------|-----------|
| IDE Team              | @harshad16        | 2025-05-14       | yes|

## Reviews

| Reviewed by   | Date       | Notes |
|---------------|------------| ------|
| @harshad16  | 2025-05-14 | no additional notes |
