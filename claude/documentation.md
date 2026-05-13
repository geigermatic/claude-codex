# Documentation — PRDs, audit blocks, freshness

Documentation is part of the codebase. It rots if untouched. It misleads if wrong. It guides the team if it's accurate and current.

This file covers the structure of PRDs, the metadata that keeps docs verifiable, the doc audit pattern, and the rules that prevent silent drift between code and documentation.

For the doc-audit workflow itself, invoke `/doc-audit`. For PRD authoring, invoke `/prd-author`.

---

## 1. Docs are code

**RULE:** Documentation lives in the repo, in version control, reviewed with the same rigor as code.

### What this means

- A change that invalidates a doc claim must update the doc in the same PR.
- A new feature comes with documentation, not "we'll document it later."
- Docs go through code review. Reviewers verify the doc matches the change.
- Stale docs are bugs.

### What this does not mean

- Every internal function needs a doc page. Most code is self-documenting.
- Every PR needs a doc update. Many changes have no doc impact.
- Comprehensive coverage is the goal. **Accurate** coverage is the goal.

The bar: anyone reading the docs should be able to act on them without verifying against the code. If they can't, the doc has failed.

---

## The 4 frontmatter rules — pithy summary

The expanded explanations live in §5-7. If you remember nothing else, remember these four.

1. **Editing body:** Update `last_verified` to today. If accuracy changes, update `status` and `confidence`. Add an audit log entry.
2. **Creating new doc:** Follow the schema in `templates/doc-audit.md`. Add an audit block at the bottom.
3. **Code invalidates doc claims:** Update `status` to `drift` and note the drift in the audit block. At minimum, flag it — don't silently leave the doc claiming something the code no longer does.
4. **Never delete frontmatter.** Required fields: `id`, `title`, `type`, `applies_to`, `status`, `confidence`, `last_verified`. PRDs also need `implemented`.

These four rules cover most of the doc-maintenance discipline. Pre-merge (see `skills/pre-merge/` §9) verifies them.

---

## 2. PRD — Product Requirements Document

PRDs describe **what we're building and why**. They precede implementation. They are the artifact that aligns the team on scope, design, and acceptance criteria.

### When to write a PRD

- Non-trivial features (more than a day of work)
- Anything customer-visible
- Cross-team or cross-service work
- Anything that involves a tradeoff worth recording

### When NOT to write a PRD

- Bug fixes (write a one-paragraph issue description)
- Refactors with no user-visible impact (a PR description suffices)
- Trivial changes (don't waste anyone's time)

---

## 3. The PRD structure

A PRD has seven sections. **In this order.** Skipping or reordering is allowed only with deliberate reason.

### 1. Problem

- One paragraph stating what we're solving.
- Concrete user or system pain. Not "we should improve search" — "users report search misses obvious matches because we don't normalize whitespace."
- Evidence: support tickets, incidents, customer feedback, metrics.

### 2. Current State

- How does the system work today?
- What's the failure mode without this change?
- Code paths, modules, services affected.
- Reference files and functions explicitly (e.g., `src/search/parse.ts:42`).

### 3. Design

- The proposed change. Architecture diagrams if useful.
- Key decisions and the tradeoffs each represents.
- Explicitly mark alternatives considered and why rejected.
- Contract changes (new endpoints, new fields, new types) listed first.

### 4. Phases

- Break the work into deployable units (slices, milestones, iterations).
- Each phase delivers user-visible value or de-risks the next phase.
- Estimate effort per phase (in days, rounded — avoid false precision).
- Identify which phase is the MVP.

### 5. Files Touched

- A list of files that will be created, modified, or deleted.
- Forecast, not perfect prediction. Reviewers use this to scope the review.

### 6. Metrics

- How will we know it worked?
- Quantitative where possible: latency, conversion, error rate.
- Qualitative when necessary: "users no longer report X."
- Include the measurement mechanism (dashboard, log query, customer survey).

### 7. Risks

- What could go wrong?
- Severity (LOW / MED / HIGH).
- Mitigation per risk.
- Open questions that must be answered before shipping.

---

## 4. The Manual UI Test Checklist (REQUIRED for user-facing PRDs)

**RULE:** Every PRD for user-facing work includes a Manual UI Test Checklist. This is not optional. It is the project's test plan.

### Format

A numbered table:

| # | Step | Expected result |
|---|------|-----------------|
| 1 | Navigate to /settings | Settings page renders within 1s |
| 2 | Click "Dark mode toggle" | Toggle animates; theme switches |
| 3 | Reload the page | Theme persists across reload |
| 4 | Open the app in a second browser | Theme syncs across browsers (or doesn't, depending on scope) |

### Why this is required

- Forces the PRD author to think through the user journey before implementation
- Gives the implementer a concrete acceptance criterion
- Gives QA / the user a script to verify the change manually
- Catches "we forgot the empty state" and "what about mobile?" early

### Concrete steps, observable outcomes

- "It should work" is not a step. "Click X and see Y" is.
- "Looks good" is not an expected result. "Element appears within 1s" is.
- The check must be runnable by someone without context.

---

## 5. Frontmatter metadata

Every doc has YAML frontmatter at the top. This metadata is the foundation of the doc-audit workflow.

### Required fields for all docs

```yaml
---
id: "DOC-042"                       # stable identifier, never reused
title: "Search Architecture"        # the doc's human title
type: "architecture"                # PRD | architecture | runbook | reference | guide
domain: "search"                    # the area of the codebase
applies_to: ["v2.x"]                # versions of the system this describes
status: "current"                   # current | draft | superseded | deprecated
confidence: "high"                  # high | medium | low — how confident in accuracy
last_verified: "2026-05-13"         # ISO date; bumped when the doc is checked
---
```

### Additional fields for PRDs

```yaml
implemented: false                  # has the work in this PRD shipped?
target_version: "v2.5"              # which release will this ship in
```

### Why each field matters

- **id**: stable reference even if the title or location changes.
- **type**: filters and tooling.
- **applies_to**: a doc that describes v1.x is misleading for v2.x readers.
- **status**: distinguishes "this is the truth" from "this was the truth once."
- **confidence**: invites readers to verify before acting on low-confidence docs.
- **last_verified**: the freshness signal.
- **implemented**: prevents "we built that already" confusion.

### Never delete frontmatter

Even if some fields seem irrelevant. The audit tooling expects them.

---

## 6. The audit block

Each non-trivial doc ends with an audit block — a structured log of who verified the doc against reality, when, and what they found.

### Format

```markdown
---

## Audit log

| Date | Reviewer | Status | Notes |
|------|----------|--------|-------|
| 2026-05-13 | @alice | verified | All code references current. |
| 2026-03-20 | @bob   | partial  | Code paths in §3 updated; metrics in §6 stale. |
| 2026-01-15 | @alice | created  | Initial draft. |
```

### When to update the audit log

- **Creating a doc:** add the initial entry.
- **Editing the body:** update `last_verified` in frontmatter AND add an entry to the audit log.
- **Verifying a doc against current code (without editing):** add an audit log entry; bump `last_verified`.
- **Discovering the doc is stale:** add an entry with `status: drift`; flag specific sections in the notes.

### Status values

- **created** — initial entry
- **verified** — audited; everything checks out
- **partial** — some sections verified, others noted as drifted
- **drift** — known to be stale; sections noted in body
- **superseded** — replaced by another doc; link to it

---

## 7. The freshness rule

**RULE:** A doc with `last_verified` older than 90 days has unknown accuracy. Treat it as a hypothesis, not a fact.

### How to handle stale docs

- **If you're touching adjacent code:** spend 10 minutes verifying the doc. Update `last_verified` if accurate.
- **If you spot drift:** mark the section as drifted in the body, log an audit entry, file an issue (or fix it).
- **If a doc has been stale >180 days and nobody's verified it:** consider deprecating it. A doc nobody maintains is worse than no doc.

### What freshness does not require

- Verifying every doc every quarter "just in case."
- Reading every doc top-to-bottom for tiny changes.

Freshness is about **trust**, not perfection. A doc dated 30 days ago that's been edited recently is trustworthy. A doc dated 18 months ago is a coin flip.

---

## 8. The doc-code consistency rule

**RULE:** If a doc claims something specific about the code (a file path, a function name, a behavior), the doc must update when that claim becomes wrong.

### Examples of doc claims that must stay in sync

- "Function `parseQuery` lives in `src/search/parse.ts`" — if the file moves, update the doc.
- "The endpoint accepts JSON; field `q` is required" — if the schema changes, update the doc.
- "The retry count is 3" — if you change the constant, update the doc.

### Patterns that help

- **PR review checklist:** "Did this PR change any behavior described in `docs/`?" → grep the docs for relevant terms.
- **Audit blocks** make drift visible: a doc with `last_verified` from before your refactor is suspect.
- **Skinny docs:** the less prose the doc has, the less to keep in sync. Prefer linking to code/tests over duplicating their content.

---

## 9. No emojis in documentation

**RULE:** Documentation uses text labels, not emojis.

### Why

- Search/grep across docs becomes unreliable when meaning is encoded in pictograms.
- Screen readers may skip or mis-pronounce.
- Some terminals and rendering pipelines mangle emoji.
- Tone varies wildly across cultures and contexts.

### Replace these patterns

- DONE / TODO / WIP instead of ✅ / ❌ / 🚧
- WARNING / DANGER / TIP instead of ⚠️ / 🛑 / 💡
- LOW / MED / HIGH instead of 🟢 / 🟡 / 🔴

The information content is the same. The robustness is higher.

### Exception

User-facing UI copy, marketing material, and casual communication can use emojis. Engineering documentation cannot.

---

## 10. Tables of contents

**RULE:** Any doc with four or more sections has a linked Table of Contents near the top, after the introductory paragraph.

### Format

```markdown
## Contents

1. [Problem](#problem)
2. [Current State](#current-state)
3. [Design](#design)
4. [Phases](#phases)
5. [Files Touched](#files-touched)
6. [Metrics](#metrics)
7. [Risks](#risks)
```

### Anchor format

GitHub-style: lowercase, spaces → hyphens, punctuation removed.

### When TOCs hurt

- Two-section docs: the TOC is longer than the doc.
- Reference cards: the structure is meant to be read top-to-bottom.

Use judgment; default to including a TOC at 4+ sections.

---

## 11. Linking, not duplicating

**RULE:** When code and documentation cover the same thing, the code is the source of truth. The doc links to the code.

### Patterns

- For "what does this function do" — link to the function with a permalink (commit-pinned).
- For "what is the schema" — link to the schema file, not paste it.
- For "what are the error codes" — link to the `ERRORS.md` catalogue, not duplicate.

### Why

Duplication rots independently. The code changes; the doc doesn't. Now you have two truths.

### When to duplicate anyway

- The reader needs to scan a list quickly without leaving the page.
- The linked content is in a remote repo or unstable location.
- The link target may break (deleted file, renamed function).

When you do duplicate, **note the source of truth** and **note the audit date** so a future reader knows when to re-verify.

---

## 12. Error catalogue

`docs/ERRORS.md` is a special doc: an operational reference mapping log messages and error codes to root causes and remediations.

### Format

```markdown
## ERR-042 — Failed to acquire database connection

**Symptom:** Log line `failed to acquire connection within timeout`; users see "Service unavailable."

**Cause:** Connection pool exhausted under concurrent load.

**Fix:** Increase `pool.max` in config. Verify with the synchronous-wait-pattern check (see `claude/testing.md` §11).

**Verified:** 2026-05-10 by @alice in incident postmortem #234.
```

### Rules

- Every user-visible error pattern gets an entry.
- Every silent error you fix in production gets an entry (with the post-mortem reference).
- Entries are append-only — never delete, only mark superseded.

### Why this matters

- Newer team members can self-serve when they see a familiar error.
- Repeat incidents have a paper trail.
- On-call gets faster over time.

---

## 13. Runbooks

A **runbook** is a step-by-step guide for an operational task: how to deploy, how to rotate a secret, how to handle a specific incident type.

### Format

- Title with the trigger condition: "How to rotate the JWT secret"
- Prerequisites: what access, what credentials
- Steps: numbered, with exact commands
- Verification: how to know it worked
- Rollback: how to undo if it didn't

### Rules

- Every operational task that's been done twice gets a runbook.
- Runbooks are tested by running them: the first person to use a new runbook reports what was unclear.
- Frontmatter includes `last_verified` because runbooks decay fastest of any doc.

---

## 14. Avoid

- **Documentation as decoration.** "We have docs!" without anyone reading or maintaining them.
- **Comprehensive but wrong.** Better to have less doc that's accurate than more that misleads.
- **Internal-only-by-default.** Most docs benefit from being readable by anyone — including future-you.
- **Marketing voice in technical docs.** Engineering documentation is plain, factual, falsifiable.

---

## 15. The "could someone act on this?" test

Before merging a doc, ask: **if a new team member read this and acted on it, would they succeed?**

- Are all referenced files / endpoints / commands correct?
- Are all preconditions stated?
- Are the success criteria observable?

If the answer is "they'd succeed if they already knew X" — X is missing from the doc.
