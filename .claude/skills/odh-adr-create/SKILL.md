---
name: odh-adr-create
description: Create a new Open Data Hub Architecture Decision Record (ADR) following the ODH template and conventions. Use this skill whenever the user wants to create, draft, write, or start a new ADR in the opendatahub-io/architecture-decision-records repository. Trigger even if the user doesn't say "ADR" explicitly -- phrases like "I need to document this decision", "let's write up the architecture for this", "create a decision record", or "new ADR for model serving" should invoke this skill. Also trigger when the user says they want to propose a change, capture an architectural choice, or record a technical decision for ODH.
---

# ODH ADR Creator

This skill helps users draft new Architecture Decision Records for the
Open Data Hub project. It handles the ODH-specific template, numbering
conventions, component scoping, and ecosystem context so the user can
focus on the substance of their decision.

The real value is in rapid iteration. Get a complete draft in front of
the user quickly, then refine it together.

## Before you begin

Read these files from the repository to get current context:

1. **The ODH template**: `architecture-decision-records/ODH-ADR-0000-template.md`
   in this repo. This is the authoritative template; follow its
   structure exactly.
2. **The ADR README**: `architecture-decision-records/README.md` for
   governance rules and philosophy.
3. **ODH ecosystem context**:
   `.claude/skills/odh-adr-review/references/odh_review_context.md` for
   the component ecosystem, platform assumptions, upstream/downstream
   relationships, and cross-component dependency patterns. This helps
   you write ADRs that are well-situated in the ODH landscape.

Read all three at the start of every invocation. They are the source of
truth; anything stated below is just guidance for applying them.

## Workflow

### Step 1: Quick interview

Ask the user a small number of open-ended questions in a single
message. The goal is to collect enough context to produce a solid first
draft, not to exhaustively document every detail. Something like:

* **What decision are you documenting?** What's changing and what's the
  proposed approach?
* **What's driving this?** Why now, what problem does it solve, what are
  the constraints?
* **What alternatives did you consider?** Even rough notes help; you can
  flesh them out in the draft.
* **What's the scope?** Is this a global/org-level decision, or specific
  to a component (e.g., model-serving, operator, distributed-workloads)?

If the user has already provided substantial context in the conversation
before invoking this skill, don't re-ask what you already know. Adapt
your questions to fill in the gaps.

Keep it conversational. The user may answer in a few sentences or a few
paragraphs; either is fine. You'll fill in the rest with reasonable
defaults they can revise.

### Step 2: Determine numbering and placement

**Auto-detect the next ADR number** by scanning existing files:

* **Global/org-level ADRs** live in `architecture-decision-records/` and
  follow the pattern `ODH-ADR-NNNN-*.md`. Scan for the highest NNNN and
  increment by 1.
* **Component-scoped ADRs** live in subdirectories (e.g.,
  `architecture-decision-records/operator/`) and follow the pattern
  `ODH-ADR-ComponentCode-NNNN-*.md`. Scan the relevant subdirectory for
  the highest NNNN and increment by 1.

Component abbreviations used in this repo (scan the subdirectory to
confirm the convention if unsure):

| Subdirectory | Code |
| --- | --- |
| operator | Operator |
| model-serving | MS |
| data-science-pipelines | DSP |
| distributed-workloads | DW |
| model-registry | MR |
| eval-hub | EH |
| explainability | XAI |
| automl | (uses global numbering in subdirectory) |
| autorag | (uses global numbering in subdirectory) |

If the component doesn't have existing ADRs or uses an unfamiliar
convention, check the subdirectory contents and follow the pattern you
find. If truly ambiguous, ask the user.

**Generate the filename** from the ADR number and a kebab-case slug
derived from the title:
`ODH-ADR-NNNN-short-descriptive-title.md` or
`ODH-ADR-Code-NNNN-short-descriptive-title.md`

### Step 3: Generate the draft

Produce a complete ADR following the template structure. Fill in every
section with substantive content based on what the user told you. Where
you don't have enough information for a section, write a reasonable
placeholder that makes it obvious what the user should fill in, but
make it a real sentence, not just "[TODO]".

**Metadata table:**

* Date: today's date
* Scope: the component or "ODH" for global
* Status: Draft (the user is authoring, not merging yet)
* Authors: ask or infer from git config
* Supersedes: N/A unless the user said otherwise
* Superseded by: N/A
* Tickets: include if the user mentioned a JIRA/GitHub issue
* Other docs: include if the user referenced related documents

**Required sections (fill all of these):**

* **What**: Concise summary of the decision
* **Why**: Motivation, driving forces, what changed
* **Goals**: Bulleted list extracted from the user's description
* **Non-Goals**: Bulleted list; infer reasonable non-goals from the
  scope, or write placeholders the user can refine
* **How**: The proposed approach in enough detail to understand the
  decision
* **Alternatives**: Document each alternative the user mentioned with
  trade-offs. If they only mentioned one alternative, note that and
  suggest they may want to add more.
* **Stakeholder Impacts**: Pre-populate the table with ODH components
  that are likely affected based on the ecosystem context you read. Use
  your understanding of cross-component dependencies to suggest groups
  the user might not have thought of. Keep the table compact (Yes, No,
  Maybe in the Impacted? column) and add a bulleted notes list below
  with explanations for each impacted group.
* **Reviews**: Empty table ready for reviewers

**Optional sections (include when relevant):**

* **Open Questions**: Include if there are unresolved points from the
  interview
* **Security and Privacy Considerations**: Include if the decision
  touches authentication, authorization, data handling, or multi-tenancy
* **Risks**: Include if you can identify meaningful risks
* **References**: Include if the user mentioned external docs, upstream
  issues, or prior art

**Writing style guidance:**

* Be direct and specific, not vague or hedging
* Use the active voice
* Use short, clear sentences. Avoid run-on sentences.
* Prefer periods, colons, or semicolons over em-dashes
* Name specific components, CRDs, and APIs rather than speaking in
  abstractions
* When discussing ODH ecosystem impacts, reference the actual components
  from the ecosystem table
* Keep sections proportional; a simple decision doesn't need a 500-word
  How section

### Step 4: Write the file and iterate

Write the draft to the determined file path. Then tell the user:

1. Where the file was created
2. A brief summary of what you filled in vs. what needs their attention
3. That you're ready to iterate; they can point out what to change, add,
   or remove
4. That when the ADR is ready, they'll typically want to create a branch
   and open a PR for review

Iterate as many rounds as the user wants. Each round, apply their
feedback and rewrite the file. The goal is a PR-ready ADR, not a
perfect first draft.

## Things to keep in mind

* **ODH is upstream for RHOAI.** Decisions should consider whether they
  can be synced downstream without breaking productization. If relevant,
  note this in the ADR.
* **OpenShift, not vanilla Kubernetes.** ADRs can and should reference
  OpenShift-specific features (Routes, OLM, ServiceMesh, oauth-proxy,
  etc.) where relevant.
* **Cross-component ripple effects are common.** Model Serving changes
  often require Dashboard UI updates and Operator CRD changes. Pipeline
  changes may affect Workbench integrations. Use the Stakeholder Impacts
  table to surface these.
* **The DataScienceCluster CRD is the primary API surface.** If the
  decision introduces a new component or changes how a component is
  configured, note whether it affects the CRD.
* **Namespace-based multi-tenancy with Kubernetes RBAC** is the
  isolation model. Don't assume Kubeflow Profiles or Istio-based auth.
