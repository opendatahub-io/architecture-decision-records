# Open Data Hub - Connection API and Connection Type Protocol

|                |            |
| -------------- | ---------- |
| Date           | 2025-10-23 |
| Scope          | Open Data Hub Operator |
| Status         | Proposed |
| Authors        | [Wen Zhou](@zdtsw) |
| Supersedes     | Data Connections (pre-2.16) |
| Superseded by: | N/A |
| Tickets        | [RHOAIENG-33699](https://issues.redhat.com/browse/RHOAIENG-33699) |
| Other docs:    | [Dashboard Connection Documentation](../../../documentation/components/dashboard/features) |

## What

This ADR defines the Connection API annotations and protocol handling introduced in OpenShift AI 2.16 and redefined in version 3.0. The API enables flexible connection management to external data sources and services through Secret-based resources with standardized annotations.

The 3.0 release introduces the `opendatahub.io/connection-type-protocol` annotation to support protocol-based connection validation and routing across components.

## Why

Prior to 2.16, OpenShift AI only supported S3-compatible "Data Connections" with a fixed schema and single annotation (`opendatahub.io/connection-type: s3`). This limitation prevented proper validation and routing for different connection protocols.

The Connection API annotations were introduced in 2.16 to:
- Support multiple connection protocols (initially S3 and URI, with OCI registries added later)
- Enable proper referencing of connection schemas
- Provide a consistent interface across components
- Enable future protocol extensibility without API changes

The 3.0 redefinition adds the `opendatahub.io/connection-type-protocol` annotation to enable webhook validation and protocol-specific handling for multiple protocols (s3, uri, oci) across components, addressing limitations discovered in production usage.

## Goals

- Define clear annotation semantics for connection protocols
- Enable protocol-based validation in operator webhooks
- Support protocol-specific routing across components
- Maintain backward compatibility with pre-2.16 Data Connections
- Enable custom protocol definitions

## Non-Goals

- Define Connection Type schema or management (Dashboard responsibility)
- Provide runtime secret rotation or dynamic credential management
- Implement connection pooling or lifecycle management
- Change the underlying storage mechanism (Connections remain as Kubernetes Secrets)

## How

### Connection Secret Structure

Connections are stored as Secrets in user project namespaces with standardized annotations for protocol identification and validation.

**Example:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-s3-connection
  namespace: <user-project>
  annotations:
    opendatahub.io/connection-type-protocol: "s3"  # NEW in 3.0
type: Opaque
data:
  AWS_S3_ENDPOINT: <base64-encoded>
  AWS_S3_BUCKET: <base64-encoded>
  AWS_ACCESS_KEY_ID: <base64-encoded>
  AWS_SECRET_ACCESS_KEY: <base64-encoded>
```

### Connection Type Protocol Annotation

The `opendatahub.io/connection-type-protocol` annotation enables:

1. **Webhook Validation**: Operator webhooks validate that Connections conform to protocol-specific requirements
2. **Protocol Routing**: Components select appropriate handlers based on protocol
3. **UI Filtering**: Dashboard filters available connections based on context

**Supported Protocols:**
- `s3`: S3-compatible object storage
- `uri`: Public HTTP/HTTPS URIs
- `oci`: OCI-compliant container registries
- Custom protocols can be defined as needed

### Webhook Integration

The operator's serving webhook validates connections based on the protocol annotation with precedence handling. The webhook determines the protocol by checking annotations in order of precedence (protocol → ref → legacy), then applies protocol-specific validation rules to ensure the connection Secret contains the required fields for that protocol type.

### Consumption Patterns

Components consume connections based on their specific requirements. The protocol annotation allows components to identify and validate appropriate connections for their use case. Protocol-specific integration logic is implemented by each component according to their needs.

### Security and Access Control

**Namespace Isolation:**
Connections are Kubernetes Secrets stored within user project namespaces. Cross-namespace access is **not supported** - connections can only be consumed by resources within the same namespace where the connection Secret exists.

**RBAC Requirements:**
- Users must have appropriate RBAC permissions to create, read, update, or delete Secrets in their project namespace
- Components consuming connections must have ServiceAccount permissions to read Secrets in the namespace
- If a user lacks permission to access a connection Secret, the consuming resource (workbench, model serving, etc.) will fail to start or function

**Validation Scope:**
The operator's webhook validation is **advisory, not restrictive**:
- Webhooks validate protocol-specific fields when the `opendatahub.io/connection-type-protocol` annotation is present
- If a user creates a connection Secret without proper annotations or with invalid fields, the Secret creation is **not blocked**
- Invalid connections will be discovered at runtime when components attempt to consume them
- This design allows for flexibility in connection creation while still providing validation guidance

### Annotation Evolution and Precedence

**Pre-2.16 (Legacy Data Connections):**
- `opendatahub.io/connection-type: s3` - Legacy annotation for S3 data connections

**2.16 (Connection API):**
- `opendatahub.io/connection-type-ref` - References the connection schema name (e.g., "s3", "uri")

**3.0 (Protocol Support):**
- `opendatahub.io/connection-type-protocol` - Explicitly identifies the connection protocol (s3, uri, oci) for validation and routing

**Annotation Precedence:**
When both annotations are present, the operator uses the following precedence for protocol determination:
1. `opendatahub.io/connection-type-protocol` (highest priority - explicit protocol, 3.0+)
2. `opendatahub.io/connection-type-ref` (backward compatibility fallback, 2.16+)
3. `opendatahub.io/connection-type` (legacy pre-2.16 support)

This precedence ensures that new connections explicitly define their protocol while maintaining backward compatibility with existing 2.16+ connections that only have `connection-type-ref`.

## Open Questions

N/A

## Responsibility

- **ODH Platform Team**: Webhook validation for protocol annotation, annotation standard maintenance
- **Component Teams** (Model Serving, Workbenches, etc.): Protocol-based integration and consumption of connections within their respective components
- **Dashboard Team**: Connection schema management (Connection Types), UI implementation

## Stakeholder Impacts

| Group              | Key Contacts     | Date       | Impacted? |
| ------------------ | ---------------- | ---------- | --------- |
| ODH Platform Team  | [Carl Kyrillos](@carlkyrillos) | 2025/10/23 | YES |
| Model Serving Team | [Edgar Hernandez](@israelity) | 2025/10/23 | YES |
| Dashboard Team     | [Andrew Ballantyne](@andrewballantyne) | 2025/10/23 | YES |
| Workbenches Team   | [Harshad Reddy Nalla](@harshad16) | 2025/10/23 | YES |

## References

- [Connection Labels & Annotations - Dashboard Docs](../../../documentation/components/dashboard/k8sLabelsAndAnnotations.md#connections)
- [Webhook Implementation](https://github.com/opendatahub-io/opendatahub-operator/tree/main/internal/webhook/serving)
- [Dashboard Storage Documentation](../../../documentation/components/dashboard/dashboardStorage.md)
