# Open Data Hub - Data Plane Layer

|                |            |
| -------------- | ---------- |
| Date           | 2025-12-10 |
| Scope          | Cross-component |
| Status         | Proposed |
| Authors        | [Jamie Land](https://github.com/jland-redhat) |
| Supersedes     | N/A |
| Superseded by: | N/A |
| Tickets        | |
| Other docs:    | none |

## What

A singular data source will be specified at the operator level that can then be used by different ODH components in order to simplify the onboarding experience for customers.

## Why

We already have at least two components—[Model as a Service](https://github.com/opendatahub-io/maas-billing/pull/300) and [Model Registry](https://github.com/opendatahub-io/model-registry-operator/pull/281/files)—that require the use of a database, with more potentially on the way. It makes sense to have a unified strategy for database configuration across the platform.

## Goals

* Provide a single location where database connection information for all ODH components is specified
* Define a clear onboarding process for new components that need to use this central database
* Simplify the customer experience when configuring database connectivity

## Non-Goals

* Deployment or ownership of customer data or databases
* Providing a managed database service

## How

Generally, we should not be creating a database for a user outside of creation for demo purposes. We need a mechanism by which components that require access to the central data source can access their specific database or schema.

## Open Questions

* How do we want users to specify the data connection information?
* Are we okay with components all using the same database, or should we require different databases for different components to maintain data separation?
* How can we store database credentials in a safe way that meets industry compliance standards?
* How should teams handle any loss of connection to the database (if that is something we want to consider)?

## Alternatives

We could have each component be responsible for its own data source configuration, but that may cause components to handle data sources in different ways. This would:

* Cause confusion amongst customers who need to configure each component separately
* Require teams to solve the same problems independently
* Result in an inconsistent user experience across the platform

## Security and Privacy Considerations

We will need a way to store and secure database credentials in order for our operator to function. We also need to understand what we can and cannot store in the database and ensure that is well documented. We don't expect to store PII, but that would be an example of something that could cause issues if we did.

## Risks

* Issues with the centralized database could bring down multiple components simultaneously
* Single point of failure for data persistence across the platform

## Stakeholder Impacts

| Group                         | Key Contacts     | Date       | Impacted? |
| ----------------------------- | ---------------- | ---------- | --------- |
| Model as a Service            | Jamie Land       | 2025-12-10 | Y |
| Model Registry                |                  |            | Y |

## References

* [Model as a Service - Database Connection PR](https://github.com/opendatahub-io/maas-billing/pull/300)
* [Model Registry - Database Connection PR](https://github.com/opendatahub-io/model-registry-operator/pull/281/files)

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------- | ----- |
|                               |            |       |
