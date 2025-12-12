|                |            |
| -------------- | ---------- |
| Date           | 25-NOV-2025 |
| Scope          | |
| Status         | Approved |
| Authors        | Gordon Sim (@grs) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This ADR concerns the selection of a solution for managing the
deployment of MCP (Model Context Protocol) servers and the configuring
of intermediaries through which they will be accessed in order to
control and observe interactions.

## Why

Users are looking for ways to mitigate the inherent risks associated
with use of MCP. Providing a solution that helps them manage the
deployment of MCP servers and ensures that all traffic to and from
them goes through an intermediary - a proxy or gateway through which
use can be controlled and observed - helps with that.

## Goals

* A declarative solution for deployment of MCP servers and
  configuration of intermediated traffic to and from them.

## Non-Goals

* The solution selected here is not intended to impact the plans for
  MCP servers that are part of the OpenShift platform. Those will
  continue to use kagenti/mcp-gateway.
 
## How

ToolHive provides an MCP proxy that supports access control (including
token verification, token exchange, fine-grained access control over
tools), observability and the ability to limit (and even rename or
redescribe) tools, along with a CRD-based API for simple deployment
and configuration of MCP servers that are then automatically
configured to be accessed through an instance of this proxy.

This will form the basis of a solution to manage the use of MCP for
OpenShift users.

ToolHive also has support for registries serving the API as defined by
Anthropic's MCP registry. This ADR does not propose to use that
aspect.

## Open Questions

N/A

## Alternatives

### kagenti/mcp-gateway

The kagenti/mcp-gateway project was created to secure access to MCP servers
for controlling the OpenShift platform itself. Using that as an
intermediary for users' own MCP servers, and augmenting the solution
with a CRD-based API for automatic deployment of those servers and
configuration for access through the gateway, similar to the full
solution ToolHive provides, was considered.

However at present there is only one gateway per cluster and all tools
exposed through it are prefixed. That limitation is scheduled to be
addressed in mcp-gateway, but in the short term is not sufficiently
flexible. Installation of mcp-gateway also requires Istio and for
access control would require Kuadrant, whereas ToolHive is
self-contained which reduces short term risk.

### kagent-dev/kmcp

The kagent-dev/kmcp project is similar to ToolHive, offering a CRD
based API to deploy MCP servers and configure them to be accessed
through the agentgateway. At present however ToolHive seems to have
more features and more visibility.

## Security and Privacy Considerations

The robustness and correctness of the ToolHive MCP proxy can affect
security. The proxy image would be particularly sensitive to CVEs as
it would be relied upon for access control and all MCP traffic would
go through it.

## Risks

The MCP ecosystem is evolving rapidly. The kagenti/mcp-gateway project
will evolve. There will also be other work done to enable MCP support
in envoy as well as work ongoing in the Kubernetes community to
consider changes to the Gateway API necessary for AI-related use, such
as for proxying MCP requests. There is a risk that changes in the
ecosystem require this decision to be re-evaluated in the future.

The ToolHive project is also fast-moving and the vast majority of
contributions so far come from the founding company. Though Daniele
Martinoli and Roddie Kieley from Red Hat's Ecosystem Engineering have
been contributing to the project for some time now, there is a risk
that we struggle to get in any changes that we make and/or that the
project moves in directions that make it less suitable for our needs.

There is also uncertainty around the time and effort required to productize
the ToolHive components and integrate them into OpenShift AI, which could
impact delivery timelines.

There is also a risk that MCP as a protocol fades in relevance over
time.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
|                               |                  |            | ?         |


## References

* [ToolHive](https://github.com/stacklok/toolhive)
* [mcp-gateway](https://github.com/kagenti/mcp-gateway)
* [kmcp](https://github.com/kagent-dev/kmcp)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
