---
name: odh-adr-review
description: Review an Open Data Hub Architectural Decision Record (ADR) using a team of seven specialist reviewer subagents and produce a consolidated report as both PDF and PPTX slide deck. Use this skill whenever the user asks to review, critique, audit, or get feedback on an ADR in the opendatahub-io/architecture-decision-records repository — whether the input is a Markdown file, a .docx document, or pasted text. Trigger even if the user does not explicitly say "ADR"; phrases like "review this architecture decision", "critique this design doc", or "run the reviewer panel on this" should also invoke this skill.
---

# ODH ADR Review Panel

This skill runs a panel of specialist reviewer subagents over an Open Data Hub Architectural Decision Record (ADR) and produces a consolidated report in two formats: a PDF document and a PPTX slide deck.

This is an ODH-specific adaptation of the generic [`adr-review`](https://github.com/opendatahub-io/ai-helpers/tree/main/helpers/skills/adr-review) skill. It adds a seventh reviewer focused on ODH ecosystem and downstream impact, injects ODH-specific context into all reviewers, and checks ADRs against the ODH template format.

## Why a panel of agents?

Architecture reviews benefit from multiple independent perspectives. A single reviewer tends to anchor on whichever concern they noticed first and under-weight the others. By running specialists in parallel, each focused on one dimension, we get broader, less-biased coverage — and the synthesis step surfaces tensions between perspectives (e.g. "the cheapest option is also the least reversible") that a single reviewer might flatten.

The seventh reviewer (ODH Ecosystem & Downstream Impact) is unique to this skill. It evaluates whether the ADR is well-situated within the ODH component ecosystem, follows the ODH template, correctly identifies cross-component stakeholders, and considers downstream RHOAI implications.

## Workflow

### Step 1 — Prompt for the ADR location and clarifying context

Always ask the user where the ADR is, even if there's a plausible file in context. Accepted inputs:

- A path to a Markdown file (`.md`, `.markdown`)
- A path to a Word document (`.docx`)
- A path to a directory containing multiple ADRs (review each one)

The default location for ADRs in this repository is the `architecture-decision-records/` directory, including its component subdirectories (e.g., `operator/`, `model-serving/`, `eval-hub/`).

If the user provides a `.docx`, extract the text first using the `document-skills:docx` skill or `python-docx`. If they provide Markdown, read it directly.

**Use the AskUserQuestion tool** to gather any clarifying context the reviewers will need. Ask in a single AskUserQuestion call with multiple questions rather than one-at-a-time. Tailor the questions to what is actually unclear after a quick skim of the ADR — don't ask boilerplate. Typical useful questions:

- **Audience & stakes** — "Who is this review for (author self-check, formal arch review, post-incident retro)?" This shapes tone and severity thresholds.
- **Scope** — "Are there dimensions you want the panel to emphasize or skip?" (e.g., "skip cost, we already costed it elsewhere")
- **Context not in the doc** — "Is there background the ADR assumes readers already know? Team size, existing stack, regulatory environment?"
- **Decision status** — "Is this a draft open to changes, or has the decision already been approved and you want a risk audit?"
- **Known concerns** — "Anything you're already worried about that you want the panel to focus on?"
- **Downstream impact** — "Does this decision specifically affect RHOAI managed service, self-managed, or both?"

Pass the answers into each reviewer subagent's prompt as a "Review context from the user" block so every reviewer sees the same framing.

### Step 2 — Read, normalize, and check template compliance

Read the file and extract the key sections. Before dispatching reviewers, check the ADR against the ODH template (`ODH-ADR-0000-template.md` in this repository). Specifically verify:

1. **Metadata table** — Date, Scope, Status, Authors, Supersedes, Superseded by, Tickets, Other docs
2. **Required sections** — What, Why, Goals, Non-Goals, How, Alternatives, Stakeholder Impacts, Reviews
3. **Optional but recommended** — Open Questions, Security and Privacy Considerations, Risks, References
4. **Numbering** — Does the ADR number follow the ODH-ADR-NNNN or ODH-ADR-Component-NNNN pattern?

Note any missing or empty sections. These observations feed into the reviewers (especially the ODH Ecosystem reviewer) as factual input, not as a verdict — a thin ADR may still represent a sound decision.

ADRs vary in adherence to the template. Don't refuse to review a non-conforming ADR; just note the gaps.

### Step 3 — Read ODH review context and dispatch the reviewer panel

First, read the ODH ecosystem context from `references/odh_review_context.md`. This context must be included in every reviewer's prompt.

Then spawn **seven reviewer subagents in parallel** using the Agent tool. Each gets:
- The full ADR text
- The ODH review context
- The template compliance observations from Step 2
- Its role-specific prompt from `references/reviewer_prompts.md`
- The "Review context from the user" block from Step 1

Launch all seven in a single message — do not serialize them.

The seven reviewers are:

1. **Context & Problem Framing** — Is the problem and its forces clearly articulated? Are the stakeholders and constraints named? Were alternatives genuinely considered?
2. **Technical Soundness** — Is the proposed solution technically correct and feasible? Does it match the problem? Does it work within the OpenShift/ODH platform?
3. **Operational & Reliability** — How will this be observed, deployed, and recovered? What are the failure modes in production? Does it consider managed service SLAs?
4. **Security & Compliance** — Threat model, data handling, authN/authZ, secrets, regulatory implications. Does it maintain namespace isolation?
5. **Cost & Performance** — Resource footprint, cost trajectory, latency/throughput, capacity planning within ODH's resource model.
6. **Consequences & Reversibility** — Are the trade-offs honestly stated? How hard is this to undo? Does it constrain future ODH/RHOAI releases?
7. **ODH Ecosystem & Downstream Impact** — Template compliance, cross-component impact, upstream/downstream alignment, operator integration, consistency with existing ADRs.

Each reviewer must return a structured finding in this shape:

```markdown
## [Reviewer Name]
**Overall assessment:** [Strong / Acceptable / Concerns / Blocking]
**Strengths:** [bullets]
**Concerns:** [bullets, each with the structure below]
**Open questions:** [bullets]
**Recommendations:** [bullets]
```

Each concern must include enough detail for an architect unfamiliar with the review to understand the issue and act on it. Use this structure per concern:

```markdown
- **[severity: low / med / high] [Short title]**
  _What:_ One or two sentences describing the specific gap or problem found in the ADR.
  _So what:_ Why this matters — the concrete consequence if the ADR is accepted without addressing it (e.g., implementer confusion, production risk, blocked downstream ADR).
  _Suggested fix:_ What the author should add, change, or clarify in the document to resolve it. Be specific enough that the author can act without re-reading the full review.
```

This level of detail applies to both the **PDF report** and **Step 3.5** (human-in-the-loop review). The user needs the full _What / So what / Suggested fix_ context to make informed agree/disagree decisions. Only the **PPTX** should use short-form bullets (title + severity) to stay scannable for meetings.

### Step 3.5 — Human-in-the-loop review of findings

Before synthesizing or generating any report, walk the user through the reviewers' findings and let them correct the record. This step exists because the reviewers are operating with limited context — the user often knows things (prior decisions, team dynamics, constraints that didn't make it into the doc) that would change whether a finding is valid.

Present the findings grouped by reviewer. For each reviewer, show:

- The overall assessment
- Each concern and recommendation as a discrete item

Then use the **AskUserQuestion tool** to collect the user's agree/disagree judgment. Batch the questions — don't spam one call per concern. A reasonable approach:

- One AskUserQuestion call per reviewer (7 total), with one question per non-trivial concern, offering options like "Agree", "Disagree — not a real issue", "Partially agree — needs rewording", "Already addressed elsewhere". Include a free-text option so the user can explain.
- Skip questions for concerns that are clearly low-severity and uncontroversial — respect the user's time.

Apply the user's corrections to the findings before synthesis:

- **Disagreed** concerns: remove them, and note in a "Reviewer corrections" sidebar that the user overrode the finding (with their reason if given). Keeping the audit trail matters — a disagreement is data, not an eraser.
- **Partially agreed** concerns: rewrite per the user's guidance.
- **Already addressed** concerns: move them to an "Addressed elsewhere" section rather than deleting, so reviewers of the report understand why they're absent.

The corrected findings — not the raw ones — are what feed into synthesis.

### Step 4 — Synthesize

Once all seven reviewers return, synthesize their findings into an overall report. The synthesis is not just concatenation — look for:

- **Agreements** — where multiple reviewers raised the same concern (stronger signal)
- **Tensions** — where reviewers disagree or where one dimension's "win" is another's "loss"
- **Gaps** — dimensions where nobody found much to say (may indicate the ADR didn't address it at all)
- **Top risks** — the 3-5 highest-severity items across the panel
- **Template compliance** — summarize how well the ADR adheres to the ODH template (from the Ecosystem reviewer's findings)
- **Downstream readiness** — is this ADR ready for RHOAI productization, or are there open concerns?
- **Overall recommendation** — Approve / Approve with changes / Needs revision / Reject, with a one-paragraph rationale

### Step 5 — Generate PDF and PPTX reports

Create a `adr-review/` subdirectory next to the input ADR file. Write outputs there:

- `adr-review/<adr-name>-review.pdf`
- `adr-review/<adr-name>-review.pptx`

**PDF structure** — use the `document-skills:pdf` skill (or `reportlab` directly if the skill isn't available):

1. Title page — ADR name, date of review, overall recommendation
2. Executive summary — 3-5 bullets, top risks, overall recommendation, template compliance summary
3. One section per reviewer with their full finding. Each concern must include the verbose _What / So what / Suggested fix_ detail so a reader can understand consequences and act on the finding without external context.
4. Synthesis section — agreements, tensions, gaps, downstream readiness

The PDF is the detailed, self-contained artifact. Err on the side of too much context per concern rather than too little — the reader may not have been in the review conversation.

**PPTX structure** — use the `document-skills:pptx` skill (or `python-pptx` directly if the skill isn't available). Follow the styling rules in `references/report_templates.md` (color palette, typography, slide layout) to produce a clean, professional deck — not plain white slides with default fonts:

1. Title slide — dark background, ADR name, date, verdict badge
2. Executive summary (overall assessment + top 3 risks)
3. One slide per reviewer (verdict badge + top 2 concerns with severity colors + top recommendation) — 7 slides total
4. Synthesis slide (agreements / tensions / gaps)
5. Next steps / recommendation slide

Keep slides scannable — bullets, not paragraphs. The PDF is the detailed artifact; the deck is for a 10-minute review meeting.

### Step 6 — Report back

Tell the user where the outputs landed and give a brief summary (overall recommendation + top risks + template compliance) inline so they don't have to open the files to get the gist.

## Handling edge cases

- **Directory of ADRs** — run the full workflow on each ADR and produce per-ADR outputs. Offer a roll-up report at the end if there are more than three.
- **Very short ADRs** — still run the full panel. A too-thin ADR is itself a finding (the Context & Framing reviewer and ODH Ecosystem reviewer will flag it).
- **Missing sections** — don't refuse to review. Note the gap and continue.
- **Non-ODH ADR format** (e.g., MADR, Nygard) — proceed anyway; the panel works on any architecture writeup. The Ecosystem reviewer will note that the ODH template wasn't followed.
- **ADRs in component subdirectories** — these are normal. The numbering pattern may be ODH-ADR-Component-NNNN rather than ODH-ADR-NNNN.

## Reference files

- `references/reviewer_prompts.md` — The full role prompt for each of the seven reviewer subagents. Read this when dispatching the panel.
- `references/odh_review_context.md` — ODH ecosystem context injected into every reviewer's prompt. Read this before dispatching.
- `references/report_templates.md` — PDF and PPTX structure templates with example content and styling rules.
