|                |            |
| -------------- | ---------- |
| Date           | 05-DEC-2025 |
| Scope          | |
| Status         | Approved |
| Authors        | Gordon Sim (@grs) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

This ADR concerns a decision on the appropriate API for managing the
configuration of MCP (Model Context Protocol) servers and the
intermediaries through which they will be accessed to control and
observe interactions.

## Why

Platform owners want to allow users of the platform to select the MCP
servers they wish to run, while ensuring that they are configured
appropriately. This would be best achieved by an API that reflects
these two distinct roles, allowing platform owners (or those to whom
they delegate the task) to pre-define the configuration they wish to
control, while letting users of the platform select which of the
approved MCP servers they wish to deploy.

## Goals

* A declarative API for managing the deployment and configuration of
  MCP servers and the intermediaries through which they are accessed.

## Non-Goals

* Selection of the intermediary to use or details of the
  implementation, which is covered by a separate ADR
  [ODH-ADR-MCP-0001].
 
## How

Two new CRDs will be introduced. One will be namespace-scoped and
named MCPServerRequest (or similar). This will be used by platform
users to indicate that they want to run a particular approved MCP
server in that namespace. The other will be cluster-scoped and named
MCPServerPolicy (or similar). It will be used by platform owners to
define an allowed MCP server and the configuration of that server and
the intermediary through which it will be accessed.

A controller will process a given MCPServerRequest for which there is
a matching MCPServerPolicy defined that applies to the relevant
namespace, and create CRs necessary to realise the desired MCP
server. E.g. in the case of using ToolHive, an MCPServer CR from the
ToolHive API would be created.

It should be noted that this mechanism does not by itself prevent
undesired use of MCP. Any user with the ability create pods can run
any image and configuration the platform allows. Either that ability
must be restricted to those that are trusted or some other means of
restricting images used on the cluster for any purpose needs to be in
place. The mechanism proposed here is intended to allow the policy to
be defined and make it easy to follow that, but is not intended to
thwart malicious use.

## Open Questions

N/A

## Alternatives

An alternative would be to use the ToolHive API directly. However, as
that API does not separate the configuration that should be centrally
defined from anything that can be left to the requesting user to
decide. This approach doesn't therefore really allow for a
self-service model with predefined configuration.

## Security and Privacy Considerations

N/A

## Risks

There is a risk that platform operators with established container
governance and RBAC mechanisms may prefer to continue using those
familiar tools rather than adopting the abstractions proposed here,
limiting the perceived value and adoption of this capability. Offering
a concrete, if tentative, solution is a way to open up those
conversations however.

There is also a relatively high risk that the proposed API would not
see wide adoption outside OpenShift AI. However, at this time there
isn't any real alternative that supports the self-service model. As
the ecosystem evolves, something new may emerge or some existing
solution may gain significant traction and alter the way requirements
are viewed. At such time this decision would need to be re-evaluated,
including perhaps defining some migration path.

However the burden of maintaining this layer is relatively small. It
does also allow different realisations of the requested MCP servers,
which could for example allow some MCP servers to be secured using
ToolHive and others using kagenti/mcp-gateway, either to support both
or to support migration from one to the other.

There is also a risk that MCP as a protocol fades in relevance over
time.

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
|                               |                  |            | ?         |


## References


## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
