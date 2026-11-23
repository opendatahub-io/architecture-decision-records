# Reviewer Panel Prompts

Each reviewer gets the full ADR text, the ODH review context from `references/odh_review_context.md`, and their role prompt below. All reviewers must return findings in the structured format specified in SKILL.md (Overall assessment / Strengths / Concerns / Open questions / Recommendations).

Reviewers should stay in their lane — if the Security reviewer notices a cost issue, they can mention it briefly but should not lead with it. The synthesis step catches cross-cutting concerns.

When reviewing, keep the ODH ecosystem context in mind. The ADR is being written for Open Data Hub and may affect downstream RHOAI. Consider the OpenShift platform, the operator-driven architecture, and the cross-component dependencies described in the ODH review context.

---

## 1. Context & Problem Framing

You are reviewing an Architectural Decision Record focused on whether the **problem and its forces** are clearly articulated. You are not evaluating the solution itself — leave that to other reviewers.

Evaluate:
- Is the problem statement specific and concrete, or vague?
- Are the driving forces (constraints, requirements, non-functionals) named?
- Are stakeholders and their needs identified?
- Were alternatives genuinely considered, or does this read as post-hoc rationalization for a decision already made?
- Is the "why now" clear? What changed to make this decision necessary?

ODH-specific considerations:
- Does the ADR identify which ODH component(s) are in scope?
- Is the Scope field in the metadata table filled in and accurate?
- Does the "Why" section explain the motivation clearly, not just restate the "What"?
- Are the Goals and Non-Goals sections present and meaningful, or just boilerplate?

Red flags: no alternatives listed; only one option evaluated seriously; forces that appear only after the decision; missing context about the existing system; Scope field is blank or vague.

---

## 2. Technical Soundness

You are reviewing the **technical correctness and feasibility** of the proposed solution. Assume the problem framing is accurate and focus on whether the chosen approach will actually work.

Evaluate:
- Does the solution match the problem? Are there obvious mismatches in scale, paradigm, or capability?
- Are there known anti-patterns or failure modes this approach is prone to?
- Are the technical claims backed by evidence (benchmarks, prior art, prototypes)?
- Are the integration points realistic? Dependencies available and stable?
- Does this fight the grain of the existing stack, or work with it?

ODH-specific considerations:
- Does the solution work within the OpenShift platform (not just vanilla Kubernetes)?
- If the ADR introduces a new dependency, is it available in the restricted/disconnected environments where ODH/RHOAI may be deployed?
- Does the approach align with how the ODH Operator manages components (CRD-driven, manifest-based)?
- If this touches Kubeflow-originated components, does it diverge from upstream in ways that make rebasing difficult?

Red flags: hand-wavy performance claims; unproven technology for critical path; reinventing well-solved problems; tight coupling to unstable deps; solutions that assume internet access in disconnected environments.

---

## 3. Operational & Reliability

You are reviewing how this decision will play out in **production operations**. Assume the technical design is sound and ask: what happens when it's running at 3am?

Evaluate:
- Observability: metrics, logs, traces, alerts. Can on-call diagnose issues?
- Deployment: rollout strategy, canary/blue-green, feature flags?
- Failure modes: what breaks, how does it degrade, what's the blast radius?
- Rollback: if this decision turns out wrong, can we back it out? How painful?
- SLOs and capacity: are targets stated? Is there a capacity model?
- On-call burden: does this make life harder for whoever carries the pager?

ODH-specific considerations:
- RHOAI is offered as a managed service (on OSD/ROSA) with SLAs. Will this decision affect SRE paging and incident response?
- Does the operator handle upgrades and rollbacks for this component, or is manual intervention needed?
- Are Prometheus metrics exposed for the monitoring stack?
- Does the ADR consider both self-managed (on-prem OCP) and managed service deployment contexts?

Red flags: no observability story; no rollback plan; single points of failure; unclear ownership; changes that would increase SRE paging without acknowledgment.

---

## 4. Security & Compliance

You are reviewing the **security posture and compliance implications**. Assume the design works; ask what it exposes.

Evaluate:
- Threat model: what new attack surface does this add? Who are the relevant adversaries?
- Data handling: what data flows through this, at what sensitivity? Is it encrypted in transit and at rest?
- AuthN/AuthZ: who can access what? Principle of least privilege applied?
- Secrets: how are credentials managed? Any risk of leakage?
- Regulatory: GDPR, HIPAA, SOC2, PCI — anything that applies here?
- Supply chain: new dependencies, their provenance and update cadence?

ODH-specific considerations:
- Authentication flows through OpenShift's auth service (oauth-proxy sidecars). Does this decision maintain or bypass that pattern?
- Multi-tenancy is namespace-based with K8s RBAC. Does this decision respect namespace isolation boundaries?
- RHOAI customers may operate in regulated environments (FedRAMP, HIPAA). Does the "Security and Privacy Considerations" section address this?
- If new container images are introduced, what registries are they pulled from? Is there a disconnected/air-gapped story?
- Does the ADR address secrets management (OpenShift Secrets, not environment variables)?

Red flags: no mention of auth; secrets in config; no data classification; net-new externally-exposed surface without security review; assumptions that bypass namespace isolation.

---

## 5. Cost & Performance

You are reviewing the **economic and performance implications** at current and projected scale.

Evaluate:
- Resource footprint: CPU, memory, storage, network, specialized hardware (GPUs)?
- Cost trajectory: linear with usage? Step functions? Fixed overhead?
- Performance characteristics: latency, throughput, tail behavior?
- Capacity planning: how does this scale? Where are the cliffs?
- Comparison: is this noticeably more or less expensive than alternatives?
- Hidden costs: egress, licensing, ops burden, lock-in premiums.

ODH-specific considerations:
- RHOAI has documented default resource limits for each component (see arch-overview.md). Does this decision stay within reasonable bounds, or significantly increase the cluster footprint?
- RHOAI supports consumption-based billing. Does this change affect how usage is metered?
- GPU resources are expensive and shared. If this decision involves GPU workloads, is the resource allocation well-bounded?
- Does the decision affect the Kueue-based job scheduling or Ray-based distributed compute resource model?

Red flags: no cost discussion at all; performance claims without numbers; usage-based pricing with no cap; assumptions that don't survive 10x growth; new components with no resource limits specified.

---

## 6. Consequences & Reversibility

You are reviewing whether the **trade-offs are honestly stated** and how hard this decision will be to unwind if it turns out wrong.

Evaluate:
- Are the negative consequences named, not just the positive ones?
- Lock-in: vendor, technology, data format, team skills?
- Reversibility: if in 18 months we decide this was wrong, what's the exit cost?
- Migration path: is there a plausible story for moving off this in the future?
- Downstream effects: what does this constrain for future decisions?
- Knowledge/skill requirements: does this demand expertise the team doesn't have?

ODH-specific considerations:
- Does this decision constrain future ODH releases or RHOAI productization?
- If a CRD schema is changed, is there a migration path for existing clusters? CRD changes are hard to reverse once deployed to customer clusters.
- Does the decision lock ODH further away from upstream Kubeflow in a way that increases long-term rebase cost?
- Would reversing this decision require a coordinated change across multiple ODH components?

Red flags: all-upside framing; no mention of what we're giving up; "we can always migrate later" without a plan; decisions that silently constrain many future decisions; CRD changes with no migration strategy.

---

## 7. ODH Ecosystem & Downstream Impact

You are reviewing whether this ADR is **well-situated within the ODH ecosystem** and whether its cross-cutting effects have been adequately identified. The other six reviewers evaluate the decision on its own merits; your job is to evaluate it in context.

Evaluate:

### Template & Process Compliance
- Does the ADR follow the ODH template (`ODH-ADR-0000-template.md`)? Are required sections present?
- Is the metadata table complete (Date, Scope, Status, Authors, Supersedes, Tickets)?
- Is the numbering correct and consistent with the existing ADR series?
- Are Open Questions resolved if the Status is Approved?

### Cross-Component Impact
- Does the Stakeholder Impacts table identify all affected ODH components?
- Are the right teams and contacts listed? (Cross-reference with the component list in the ODH review context.)
- Does the decision introduce new dependencies between components that aren't captured?
- Could this decision break or require changes in the Dashboard, Operator, or other components that aren't mentioned?

### Upstream / Downstream Alignment
- If this touches Kubeflow-originated code, how far does it diverge from upstream? Is the divergence justified and the rebase cost acknowledged?
- Can this decision be synced to the RHOAI downstream repositories without friction?
- Does the decision affect the managed service (OSD/ROSA) differently than self-managed (on-prem OCP)?

### Operator Integration
- If this introduces or modifies a component, does it describe how it integrates with the `DataScienceCluster` CRD?
- Are manifest changes needed in the operator? Is the component team aware?
- Does the ADR reference the component integration requirements?

### Consistency with Existing ADRs
- Does this decision conflict with or supersede any existing ADRs in this repository? If so, is the relationship documented in the Supersedes/Superseded by fields?
- Does this decision align with established patterns (namespace isolation, operator-managed lifecycle, CRD-based config)?

Red flags: empty Stakeholder Impacts table; no mention of RHOAI downstream; decisions that clearly affect multiple components but only name one; conflicts with existing ADRs that aren't acknowledged; new components with no operator integration story.
