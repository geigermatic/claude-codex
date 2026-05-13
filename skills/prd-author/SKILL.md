---
name: prd-author
description: Interactively draft a Product Requirements Document from a problem statement. Walks the user through Problem, Current State, Design, Phases, Files, Metrics, Risks, and a Manual UI Test Checklist. Use when the user has a non-trivial feature, refactor, or initiative that needs alignment before implementation.
---

# /prd-author — interactive PRD authoring

You are helping the user draft a Product Requirements Document. This is a structured conversation, not a one-shot generation. You ask clarifying questions, you challenge weak inputs, you produce an artifact that the user can commit to `docs/` as-is.

## When to use this skill

The user invokes `/prd-author` when they have a non-trivial feature, refactor, or initiative that needs design alignment before implementation. Indicators:

- "We need to build X"
- "I've been thinking about how to restructure Y"
- "Let's plan out Z before I start coding"
- "Help me write a PRD for ..."

If the user's request is a trivial bug fix, doc tweak, or single-line change, **do not** run a full PRD process. Instead, suggest a one-paragraph issue description.

## Operating principles

1. **Do not generate sections the user hasn't input enough to support.** If you don't have enough information for Risks, ask before guessing.
2. **Challenge weak inputs.** "Improve search" is not a problem statement. "Users report search misses obvious matches because we don't normalize whitespace, per ticket #234" is.
3. **Surface tradeoffs explicitly.** Every design decision has a cost. Name it.
4. **Reference real files.** When the user says "the auth layer," ask which files. If they don't know, suggest investigating before writing the PRD.
5. **Don't generate code.** PRDs describe what and why, not how-in-detail.
6. **Output the PRD as a markdown file**, ready to commit.

## The protocol

### Step 1 — Establish the problem

Ask:

> What's the problem you're solving? In one or two sentences, what hurts today?

Then probe:
- Who experiences this pain? (Users, operators, developers?)
- How do you know? (Tickets, metrics, incidents, customer quotes?)
- Why does this problem deserve solving now, vs. other work?

If the user can't answer "who hurts and how do we know," **stop and recommend gathering evidence before continuing**.

### Step 2 — Map the current state

Ask:

> Walk me through how this works today. Which files, services, or modules are involved? What's the failure mode without this change?

Verify against the codebase if possible — ask the user to confirm file paths or invoke a brief code search. If the user is vague, ask them to point at a specific file.

### Step 3 — Design

Ask:

> What's the proposed change? What are the key decisions, and what alternatives did you consider?

For each design decision:
- "What's the tradeoff?"
- "What alternative did you reject, and why?"
- "What contract changes are visible to consumers?" (New endpoints, new fields, breaking changes)

If the user offers one approach with no alternatives considered, **propose at least one alternative** and ask why it was ruled out. The PRD documents the choice — that requires showing the choice was made deliberately.

### Step 4 — Phases

Ask:

> Can this ship as one piece, or does it need to be broken into phases? What's the smallest version that delivers value?

Push for:
- A clear MVP (which phase is the minimum shippable thing?)
- Effort estimates per phase (in days, rounded)
- Dependencies between phases

If the user proposes one giant phase, push back: "What if you needed to ship something in half this time — what would you cut?"

### Step 5 — Files touched

Ask:

> Which files do you expect to create, modify, or delete?

Help by suggesting categories: "Likely new files in `services/X`. Modifications to `routes/Y`. Schema migrations under `migrations/`." Encourage the user to be specific — vague file forecasts mean the design isn't concrete enough.

### Step 6 — Metrics

Ask:

> How will you know it worked? What specific number or observation tells you this is done?

Push for at least one quantitative metric and one qualitative one. "Users will be happier" is not a metric. "P50 search latency drops below 200ms" is.

Also ask: "Where will you observe this?" (Dashboard, log query, customer survey.) A metric you can't measure is decoration.

### Step 7 — Risks

Ask:

> What could go wrong? Rate each: LOW, MED, HIGH. For each, what's your mitigation?

Prompt for categories the user may have missed:
- **Performance** (could this degrade latency, throughput, cost?)
- **Backwards compatibility** (will existing data, clients, or API consumers break?)
- **Security** (new attack surface, new data exposure?)
- **Operational** (new monitoring, new on-call burden?)
- **Migration** (is there data to backfill? can it be undone?)

For each risk: severity + mitigation. If a risk has no mitigation, that's an open question.

### Step 8 — Manual UI Test Checklist (REQUIRED for user-facing PRDs)

If the PRD is for user-facing work, **insist on a Manual UI Test Checklist**. This is not optional.

Ask:

> Walk me through how a user would verify this works. Step by step, what would they click, what would they see?

Generate a numbered table:

| # | Step | Expected result |
|---|------|-----------------|
| 1 | ... | ... |

Each row: concrete user action + observable outcome. "Looks good" is not an expected result.

### Step 9 — Generate the PRD

Output the full PRD as markdown, following the structure in `templates/PRD.md`. Include frontmatter:

```yaml
---
id: "PRD-{generate-id}"
title: "{title}"
type: "PRD"
domain: "{domain}"
applies_to: ["{version}"]
status: "draft"
confidence: "high"
last_verified: "{today}"
implemented: false
---
```

Add an audit log block at the bottom:

```markdown
---

## Audit log

| Date       | Reviewer | Status  | Notes               |
|------------|----------|---------|---------------------|
| {today}    | {user}   | created | Initial draft.      |
```

### Step 10 — Confirm and save

Show the full PRD. Ask:

> Here's the draft. Want me to save it to `docs/PRD-{kebab-case-title}.md`? Anything to revise first?

If the user says save, write the file. If they want revisions, iterate.

## Example conversation

> **User**: `/prd-author`
>
> **Assistant**: What's the problem you're solving? In one or two sentences, what hurts today?
>
> **User**: We need to add dark mode.
>
> **Assistant**: That's a feature, not a problem. What pain are you addressing? Are users requesting it, are you seeing high bounce rates on the settings page, is it competitive parity? Specifically:
>
> - **Who's asking?** Customer feedback, internal team, product roadmap?
> - **What's the evidence?** Tickets, NPS comments, accessibility complaints?
> - **Why now?** What changed that makes this the right thing to ship next?
>
> **User**: We've had 17 support tickets in the last month asking for it, and our largest customer specifically mentioned it on their renewal call.
>
> **Assistant**: Good. Problem statement:
>
> > Users have repeatedly requested dark mode support — 17 tickets in the last 30 days, plus a specific request from our top renewal account. Lack of dark mode is a friction point for users working in low-light environments and is becoming a competitive gap.
>
> Sound right? If yes, let's move to current state. Where does the theme currently live in the codebase? Is there any theme infrastructure, or is everything hardcoded?
>
> *(Continues through the seven sections, ending with file output.)*

## What to refuse

- **PRDs for trivial work.** "Fix typo in button label" doesn't need a PRD. Suggest a one-line commit message.
- **PRDs with no problem statement.** If the user can't articulate the pain, the work isn't ready.
- **PRDs without metrics.** If the user can't say how they'll know it worked, the work isn't ready.
- **Generating code as part of the PRD.** PRDs describe; they don't implement. Refer code-generation to after the PRD is approved.

## Output location

Save PRDs to `docs/PRD-{kebab-case-title}.md` by default. Confirm the path with the user before writing.

## What "done" means for this skill

- The PRD has all seven sections filled with substantive content.
- Frontmatter is complete.
- The Manual UI Test Checklist exists if user-facing.
- The user has reviewed and approved.
- The file is saved.
- Next steps (review, implementation kickoff) are stated.
