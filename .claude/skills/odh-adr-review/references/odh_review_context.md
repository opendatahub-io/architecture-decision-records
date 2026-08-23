# Open Data Hub — Review Context

This document provides ODH-specific context that every reviewer must read before evaluating an ADR in this repository. It supplements each reviewer's domain-specific lens with knowledge of the ODH ecosystem, conventions, and constraints.

## What is Open Data Hub?

Open Data Hub (ODH) is an open-source AI/ML platform built on OpenShift. It is the upstream project for **Red Hat OpenShift AI (RHOAI)**, a commercially supported product. Architectural decisions made in ODH directly affect RHOAI downstream.

## ODH Component Ecosystem

ADRs in this repo may affect one or more of these components:

| Component | Purpose | Key Repos |
|-----------|---------|-----------|
| **ODH Operator** | Meta-operator that deploys and manages all components via `DataScienceCluster` CRD | opendatahub-io/opendatahub-operator |
| **Dashboard** | Unified web UI for all ODH features | opendatahub-io/odh-dashboard |
| **Workbenches / Notebooks** | JupyterLab-based data science IDEs (Kubeflow Notebook Controller + ODH Notebook Controller) | opendatahub-io/notebooks, opendatahub-io/kubeflow |
| **Model Serving** | KServe (single-model) and ModelMesh (multi-model) inference serving | opendatahub-io/kserve, opendatahub-io/modelmesh-serving |
| **Data Science Pipelines** | ML pipeline orchestration (based on Kubeflow Pipelines) | opendatahub-io/data-science-pipelines-operator |
| **Distributed Workloads** | Ray, CodeFlare, Kueue for distributed training and compute | opendatahub-io/distributed-workloads |
| **Model Registry** | Model metadata and version tracking | opendatahub-io/model-registry |
| **TrustyAI / Explainability** | Fairness metrics and model explainability | trustyai-explainability/trustyai-explainability |
| **EvalHub** | Unified evaluation service for orchestrating AI model evaluations across multiple frameworks (lm_evaluation_harness, ragas, garak, etc.). Managed by the TrustyAI operator. | opendatahub-io/eval-hub |
| **AutoML** | Automated ML model building and optimization for tabular data using AutoGluon, orchestrated via Kubeflow Pipelines. Integrates with Model Registry and KServe. | opendatahub-io/automl |
| **AutoRAG** | Automated RAG application optimization using ai4rag engine, orchestrated via Kubeflow Pipelines. Explores chunking, embedding, retrieval, and generation configurations. | opendatahub-io/autorag |
| **RAG and Vector DB** | Document ingestion, chunking, embedding, and vector database management for retrieval-augmented generation workflows. Provides the runtime infrastructure that AutoRAG optimizes against. |
| **Llama Stack** | LLM inference API layer providing unified access to language models and vector databases for AI workloads | opendatahub-io/llama-stack |
| **Feature Store** | Feast-based feature management for training and serving | opendatahub-io/feast |
| **Agent Ops** | Onboards agents and MCP servers to the platform. Facilitates configuration of sandboxes, workload identity using SPIFFE/SPIRE, credential management, authorization, and agent discovery. Implemented using OpenShell, Agent Operator, MCP Operator, Agent Sandbox, and MCP Gateway.
| **Model and Agent Observability** | Collects traces and metadata about models and agents. Visualizes traces, creates evaluation datasets, evaluates expected vs. actual behavior.  Uses MLflow.
| **Model as a Service** | Platform-managed LLM inference endpoints that provide shared, pre-deployed models to users without requiring them to provision their own model serving infrastructure. |

## Platform Assumptions

- **OpenShift, not vanilla Kubernetes.** ODH runs on OpenShift Container Platform (OCP), OpenShift Dedicated (OSD), and ROSA. Platform features like Routes, ServiceMesh, Serverless (Knative), OLM, and OpenShift authentication are available and commonly used.
- **Operator Lifecycle Manager (OLM)** manages operator installation and upgrades.
- **Namespace-based multi-tenancy with Kubernetes RBAC** is the standard isolation model. ODH does not use Kubeflow Profiles, Istio-based auth, or Dex for multi-user isolation.
- **CRD-driven API.** The `DataScienceCluster` CRD is the primary API surface. New components must integrate with the operator and expose configuration through this CRD.

## Upstream / Downstream Relationship

- **ODH is upstream; RHOAI is downstream.** Changes to ODH are synced to downstream Red Hat repositories (red-hat-data-services/*) for productization.
- **Kubeflow is further upstream** for several components (Notebooks, Pipelines). ODH forks and extends Kubeflow components. Decisions that diverge from upstream Kubeflow should justify the divergence and consider the ongoing rebase cost.
- ADRs should consider: *Can this decision be synced downstream to RHOAI without breaking the productization flow?*

## Release Cadence

RHOAI follows a 3-month release cycle:
- **EA1 (Early Access 1):** ~1 month after development begins
- **EA2 (Early Access 2):** ~1 month after EA1
- **GA (General Availability):** ~1 month after EA2

ADRs do not need to be implementable within a single release phase, but should be aware of this cadence when discussing rollout strategy.

## Cross-Component Dependencies

Decisions in one component frequently affect others. Common patterns:
- Model Serving changes often require Dashboard UI updates, Operator CRD changes, and ServiceMesh/Serverless configuration
- Pipeline changes may affect Workbench integrations
- Operator-level decisions (CRD schema, lifecycle management) affect all components
- Security and authentication changes ripple across Dashboard, Notebooks, and Model Serving

The **Stakeholder Impacts** table in the ODH template exists to capture these cross-component effects. Reviewers should flag when this table appears incomplete given the scope of the decision.

## ODH ADR Template Format

ADRs in this repository follow the template in `ODH-ADR-0000-template.md`. The expected sections are:

| Section | Required? | Notes |
|---------|-----------|-------|
| Metadata table (Date, Scope, Status, Authors, Supersedes, Tickets, Other docs) | Yes | Status should be "Approved" when merged |
| What | Yes | Brief description of the decision |
| Why | Yes | Motivation — why an ADR is needed |
| Goals | Yes | Bulleted list |
| Non-Goals | Yes | Bulleted list — what is explicitly out of scope |
| How | Yes | High-level approach |
| Open Questions | Optional | Should be resolved before status moves to Approved |
| Alternatives | Yes | Must document trade-offs of each option |
| Security and Privacy Considerations | Optional | But strongly encouraged |
| Risks | Optional | But strongly encouraged |
| Stakeholder Impacts | Yes | Table with Group, Key Contacts, Date, Impacted? |
| References | Optional | Supporting links |
| Reviews | Yes | Table tracking reviewer sign-offs |

This format differs from MADR and Nygard-style ADRs. Reviewers should evaluate completeness against this template, not against other ADR formats.

## ADR Governance

- ADRs are numbered sequentially (ODH-ADR-NNNN or ODH-ADR-Component-NNNN). Numbers are not reused.
- Superseded ADRs remain in the repo, marked with `Superseded by` in the metadata table.
- Component-scoped ADRs live in subdirectories (e.g., `operator/`, `model-serving/`, `eval-hub/`).
- ADRs are approved by merging the PR after collecting reviewer sign-offs in the Reviews table.
