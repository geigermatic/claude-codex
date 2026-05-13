---
id: "PRD-{stable-id}"
title: "{Title — what we're building}"
type: "PRD"
domain: "{area of codebase, e.g., search, billing, auth}"
applies_to: ["{version-or-component-this-applies-to}"]
status: "draft"
confidence: "high"
last_verified: "{YYYY-MM-DD}"
implemented: false
target_version: "{vX.Y.Z}"
---

# {Title}

> Brief summary in one sentence. What is being built and for whom.

## Contents

1. [Problem](#problem)
2. [Current State](#current-state)
3. [Design](#design)
4. [Phases](#phases)
5. [Files Touched](#files-touched)
6. [Metrics](#metrics)
7. [Risks](#risks)
8. [Manual UI Test Checklist](#manual-ui-test-checklist)

---

## Problem

{One paragraph stating what we're solving. Concrete pain, with evidence.}

**Who experiences this pain:** {users, operators, developers, customer segment}

**Evidence:**
- {Support tickets, e.g., "17 tickets in March mentioning X"}
- {Metrics, e.g., "5% conversion drop on the Y page since launch"}
- {Customer quotes, with attribution where possible}
- {Incidents, e.g., "Postmortem #42 traced to this gap"}

**Why now:** {What changed, what deadline, what opportunity}

---

## Current State

{How does the system work today? What's the failure mode without this change?}

**Affected code paths:**
- {file/module}: {what it does today}
- {file/module}: {what it does today}

**Current behavior, step by step:**

1. User does X
2. System does Y
3. Result is Z

**Where this fails:**

{The specific scenario where the current behavior is insufficient or wrong.}

---

## Design

{The proposed change.}

### Approach

{High-level description. Architecture diagrams if useful — use plain ASCII or link to an image.}

### Key decisions

#### Decision 1: {decision title}

- **Choice:** {what we're doing}
- **Tradeoff:** {what this costs}
- **Alternative considered:** {what we rejected}
- **Why rejected:** {reason}

#### Decision 2: ...

### Contract changes

{New endpoints, new fields, new types. List explicitly.}

- New endpoint: `POST /api/...`
- New field: `User.darkMode: boolean`
- Breaking change: {if any}

### Non-goals

{What we explicitly are NOT doing in this work, even though someone might expect us to.}

- {Thing we're not doing, and why}

---

## Phases

{Break the work into deployable units. Each phase delivers user-visible value or de-risks the next phase.}

### Phase 1 (MVP) — {phase title}

**Outcome:** {what users gain}

**Scope:**
- {What's included}
- {What's included}

**Files touched:**
- {file 1}
- {file 2}

**Effort:** {N days, rounded}

### Phase 2 — {phase title}

**Outcome:** {what users gain}

**Scope:**
- ...

**Effort:** {N days}

### Phase 3 — {phase title (if applicable)}

...

---

## Files Touched

{Forecast — not a perfect prediction. Reviewers use this to scope the review.}

### New files

- `path/to/new/file.ts` — {purpose}
- `path/to/new/test.ts` — {purpose}

### Modified files

- `path/to/existing/file.ts` — {what changes}
- `path/to/existing/other.ts` — {what changes}

### Deleted files

- `path/to/dead/file.ts` — {why}

---

## Metrics

{How will we know it worked?}

### Quantitative

- {Metric}: {target}
  - Measured via: {dashboard / log query / event}

### Qualitative

- {Outcome}: {how observed}

### Anti-metrics (regressions to watch for)

- {Metric that should NOT degrade}: {threshold}

---

## Risks

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | {risk} | HIGH | {how mitigated} |
| 2 | {risk} | MED | {how mitigated} |
| 3 | {risk} | LOW | {how mitigated} |

### Open questions

- {Question that must be answered before shipping}
- {Question that can be answered during phase 2}

---

## Manual UI Test Checklist

{For user-facing PRDs only. This is required, not optional.}

| # | Step | Expected result |
|---|------|-----------------|
| 1 | {Concrete user action} | {Observable outcome} |
| 2 | {Action} | {Outcome} |
| 3 | {Action} | {Outcome} |
| 4 | {Edge case: empty state} | {Outcome} |
| 5 | {Edge case: error state} | {Outcome} |
| 6 | {Mobile / responsive} | {Outcome} |

---

## Audit log

| Date       | Reviewer  | Status    | Notes               |
|------------|-----------|-----------|---------------------|
| {date}     | {@user}   | created   | Initial draft.      |
