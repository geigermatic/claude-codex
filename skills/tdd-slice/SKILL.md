---
name: tdd-slice
description: Plan a feature as vertical slices with contract test → implementation → e2e → flag-gated deploy. Use when starting non-trivial customer-facing feature work, especially anything proxying an external service or requiring revert speed.
---

# /tdd-slice — slice-based TDD planner

You are helping the user break a feature into deployable vertical slices and plan each slice with contract-test → implementation → e2e → flag-gated rollout. This skill produces an execution plan, not code.

## When to use this skill

Invoke when the user is starting non-trivial customer-facing feature work. Especially valuable when:

- The feature proxies an external service (where revert speed matters)
- Customer trust is on the line
- The change touches multiple layers (UI, API, DB)
- You want to ship in pieces, not one big-bang merge

If the work is trivial (one-line fix, single-file change), skip this skill and just do the work.

## The five patterns this skill enforces

1. **Slice by UI-testable outcome** — every slice produces something a user could verify in the browser.
2. **Contract test before implementation** — write the failing test first; it defines the contract.
3. **Pure-function-plus-Express-binding split** — business logic in pure functions, HTTP plumbing thin and separate.
4. **Playwright (or equivalent) e2e for user-visible flows** — protects against regressions in the user-facing path.
5. **Flag-default-OFF production landing** — every slice ships behind a feature flag, default off; turn on by tenant for rollout.

## The protocol

### Step 1 — Capture the feature

Ask:

> What's the feature, in one sentence? What does the user gain when it ships?

Then ask:

> Is there a PRD? If yes, point me at it. If not, are you sure this is the right place to start — would `/prd-author` help first?

If no PRD exists and the feature is non-trivial, **recommend `/prd-author` first**. Slice planning is sharper when the design is settled.

### Step 2 — Identify the slices

Ask:

> What's the minimum slice that delivers something user-visible? Then what comes next?

Push the user toward **thin vertical slices**:
- Each slice touches every layer it needs (UI, API, DB) but only enough to deliver one user-facing outcome.
- Each slice is independently deployable.
- Each slice is small enough that the user could walk away after it ships and have something real.

Anti-pattern: "horizontal" slices ("first the schema, then the API, then the UI"). These don't deliver value until all three land — and they can't be reverted independently.

Good slice examples:
- "Slice 1: Show a read-only version of the page with hard-coded data, behind a flag."
- "Slice 2: Wire up the read API to replace hard-coded data."
- "Slice 3: Add the edit form (write API)."
- "Slice 4: Add validation."
- "Slice 5: Add the Terms-of-Service and Privacy links."

Each is shippable, each is small, each is a real user experience.

### Step 3 — Plan slice N

For each slice, the user works through this:

#### 3a — Define the user-testable outcome

> What does the user see/do at the end of this slice? Describe the single observable thing they can test.

#### 3b — Write the contract test FIRST

Generate the failing test that defines the slice's outcome.

- For HTTP: a request/response test (status, body shape, headers).
- For pure logic: a function-call test (input, expected output).
- For UI: a Playwright (or equivalent) e2e test that drives the browser.

Confirm with the user: this test should **fail** before the slice is implemented. That's the point.

#### 3c — Split pure logic from HTTP plumbing

For backend work, push the user toward:

- A **pure function** that takes inputs and returns outputs. No HTTP, no DB, no I/O.
- A **thin binding** (e.g., Express handler) that parses the request, calls the pure function, formats the response.

This separation makes testing trivial (pure functions test in milliseconds) and refactoring safe (move the function elsewhere without touching the route).

#### 3d — Choose the feature flag

> What flag name? Where does it live? Who can flip it?

Flag default: **OFF**. Always. Even on the day you intend to ship.

Recommend:
- `kebab-case-flag-name`
- Stored in a feature-flag service (LaunchDarkly, GrowthBook, or a homegrown one)
- Or, if your project doesn't have one, a simple env var + per-tenant override table

The flag stays in place until the feature has been stable in production long enough to remove it. Then remove the flag (flag debt is real).

#### 3e — Specify the manual UI verification step

Even with e2e tests, a human verifies before merging:

> What will you do in the browser to confirm this slice works? Step-by-step.

This becomes the Manual UI Test Checklist row for this slice.

#### 3f — Identify the rollback path

> If this slice breaks production, how do you turn it off? In how many seconds?

Acceptable answers:
- "Flip the flag" (seconds)
- "Revert the PR + redeploy" (minutes)

If neither, the slice is too big or insufficiently flag-gated.

### Step 4 — Generate the slice plan

Output a structured plan as markdown. Save to `docs/SLICE-PLAN-{feature-name}.md`:

```markdown
# Slice Plan: {Feature Name}

## Overview

{One paragraph: what we're building, why slices.}

## Slices

### Slice 1: {Outcome in user terms}

- **Contract test**: {description, file path}
- **Implementation scope**: {what changes}
- **Files touched**: {list}
- **Flag**: `{flag-name}` (default OFF)
- **Manual verification**:
  | # | Step | Expected result |
  |---|------|-----------------|
  | 1 | ... | ... |
- **Rollback**: {how to undo}
- **Estimate**: {effort}

### Slice 2: ...

...
```

### Step 5 — Implement (separate session, not this skill)

This skill produces the plan. Implementation is a separate effort. When the user is ready to start a slice:

- They invoke `/bug-fix-tdd` (if it's bug-fix-shaped) or just start working from the plan.
- The plan is the source of truth for what each slice delivers.
- Each slice is its own PR (or commit, if working solo).

## Example conversation

> **User**: `/tdd-slice` — I want to plan out a native password reset flow that currently proxies to a third-party identity provider.
>
> **Assistant**: One-sentence feature: replace the third-party password reset with a native flow. Users see and use our own page instead of being bounced off-site.
>
> Is there a PRD for this? If not, the slice plan will be sharper after `/prd-author`.
>
> **User**: Yes, PRD-Native-Password-Reset.md. Let's slice it.
>
> **Assistant**: Good. Quick read of the PRD... [reads it] ... I see Phases 1-5 are already defined. Want me to use those as the slices, or do you want to re-cut them?
>
> **User**: Use the phases as slices.
>
> **Assistant**: OK. Slice 1: render the reset page with a hard-coded form, no backend. User-testable: they can hit `/reset-password` and see the form. Behind flag `native-password-reset`, default off.
>
> Contract test before implementation: a Playwright test that visits `/reset-password` and asserts the form renders. It should fail today (the route doesn't exist).
>
> Sound right? Or would you split differently?
>
> **User**: That's right. Continue with slice 2.
>
> *(Continues through all slices, producing the plan.)*

## What to refuse

- **One-giant-slice plans.** If the user wants to do everything in one PR, push back. The point of slices is independent shippability.
- **Slices without flag gates** for customer-visible work. Ask "what's your rollback path?" — if the answer requires a deploy, the slice isn't flag-gated enough.
- **Plans without contract tests.** The contract test is the slice's spec. No spec = no plan.
- **Skipping manual verification steps.** Automated tests don't replace human-in-the-loop verification for user-facing flows. Especially after deploy.

## Special note: SPA "200 OK" trap

When the slice adds external URLs (Terms, Privacy, third-party docs), do not rely on HTTP 200 to verify the link works. SPAs (single-page apps) serve a homepage shell for unknown routes with a 200 status — the 404 is rendered client-side. **The only reliable verification is rendering the URL in a real browser.**

If your e2e tests use Playwright (or similar), this is automatic. If you're verifying manually, click the link and confirm the page that loads is what you expected.

## What "done" means for this skill

- The slice plan file is written to `docs/`.
- Each slice has a clear user-testable outcome, contract test, flag, manual verification, and rollback.
- The user has agreed to the slicing.
- Next step (start implementing Slice 1) is clear.
