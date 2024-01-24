# Introducing namespace controller to manage Data Science Projects


|                |                                                 |
| -------------- |-------------------------------------------------|
| Date           | 2024-01-24                                      |
| Scope          |                                                 |
| Status         | Draft                                           |
| Authors        | [Vaishnavi Hire](@VaishnaviHire)                |
| Supersedes     | N/A                                             |
| Superseded by: | N/A                                             |
| Tickets        | https://issues.redhat.com/browse/RHOAIENG-2186  |
| Other docs:    | none                                            |

## What

This document outlines the decision to introduce a namespace controller in the ODH Operator.

## Why

Currently, the ODH Operator has no knowledge of the Data Science Projects(namespaces) managed by the ODH Dashboard.
Introduce a controller to manage Data Science Projects through backend. This will allow components to access DSProjects 
and add artifacts to project namespaces directly without creating then through the Dashboard.

## Goals

* ODH operator initialize and manage any resources that are required for ODH components.  This could include configmaps,
secrets, namespaced resources.
* ODH operator would be responsible for reconciling these resources to ensure that they are consistent with each release.


## Non-Goals

* This implementation does not include introduce a new CR, and will just watch Namespace resource.

## How

* Introduce a new controller in the operator that watches for namespaces identified as Data Science Projects.

## Alternatives

* Use ODH Dashboard to deploy resources in every Data Science Project required by ODH components. 


## Stakeholder Impacts

| Group     | Key Contacts     | Date       | Impacted? |
|-----------| ---------------- | ---------- | --------- |
| Platform  | key contact name | date       | ? |
| Dashboard | key contact name | date       | ? |

## References

* optional bulleted list

## Reviews

| Reviewed by                   | Date       | Notes |
| ----------------------------- | ---------  | ------|
| name                          | date       | ? |
