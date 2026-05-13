---
id: "DOC-SLICE-WORKFLOW"
title: "Slice-Based TDD Workflow"
type: "reference"
domain: "engineering-process"
applies_to: ["all"]
status: "current"
confidence: "high"
last_verified: "{YYYY-MM-DD}"
---

# Slice-Based TDD with Flag-Gated Deploys

A reusable workflow for non-trivial customer-facing feature work, especially anything that proxies an external service or where revert speed matters more than peak velocity.

This document is reference material. The interactive planner is the `/tdd-slice` skill.

## Contents

1. [When to use this workflow](#when-to-use-this-workflow)
2. [The five patterns](#the-five-patterns)
3. [Worked example](#worked-example)
4. [Anti-patterns](#anti-patterns)
5. [Adapting to your stack](#adapting-to-your-stack)

---

## When to use this workflow

Use slice-based TDD when:

- The change is customer-facing.
- Customer trust is on the line (auth, billing, anything externally-visible).
- The work proxies or replaces a third-party service (where revert speed matters).
- The change spans UI + API + storage and could ship in pieces.

**Don't use it** for trivial fixes, internal-only changes, or single-file work — the overhead exceeds the benefit.

---

## The five patterns

### Pattern 1 — Slice by UI-testable outcome

Each slice produces a user-visible outcome that can be verified in a browser. Not "the database schema is updated" — "the page renders with new data."

A slice that doesn't deliver something a user can verify is not a slice. It's a step toward a slice.

**Good slices:**
- Slice 1: Show a read-only version of the page with hard-coded data.
- Slice 2: Wire up the read API; data now comes from the backend.
- Slice 3: Add the edit form (write path).

**Bad slices:**
- "Build the schema."
- "Add the migration."
- "Implement the storage layer."

These are tasks within slices, not slices themselves.

### Pattern 2 — Contract test before implementation

Before any implementation code, write the test that defines the slice's outcome. The test fails today because the slice hasn't been built.

- For HTTP: a test that makes a request and asserts the response shape.
- For pure logic: a function-call test asserting input → output.
- For UI: an end-to-end test (Playwright, Cypress) that drives the browser.

The test is the spec. It captures what the slice is supposed to do. Once it passes, the slice is done — by definition.

**Why this matters:** without the test up front, the slice's "done" criterion is fuzzy. With it, "done" is "the test passes." No debate.

### Pattern 3 — Pure-function-plus-binding split

For backend work, split each slice's logic into:

- **A pure function** (or pure module) that takes inputs, returns outputs, has no I/O.
- **A thin binding** (e.g., Express handler, AWS Lambda handler, message consumer) that parses input, calls the pure function, formats output.

```ts
// Pure logic — easy to test
export function calculateRefund(order: Order, reason: RefundReason): RefundResult {
  // ... no I/O, no DB, no HTTP, just logic
}

// Thin binding — easy to skip in tests
app.post('/orders/:id/refund', async (req, res) => {
  const order = await db.getOrder(req.params.id);
  const result = calculateRefund(order, req.body.reason);
  await db.saveRefund(result);
  res.json(result);
});
```

**Why:**
- Pure functions test in milliseconds with trivial setup.
- The binding has so little logic that bugs are obvious.
- You can move the function (rename, restructure) without touching the route.

### Pattern 4 — Playwright (or equivalent) e2e for user-visible flows

For any user-visible slice, an end-to-end test drives a real browser:

```ts
test('user can toggle dark mode', async ({ page }) => {
  await page.goto('/settings');
  await page.getByRole('button', { name: 'Dark mode' }).click();
  await expect(page.locator('html')).toHaveAttribute('data-theme', 'dark');
});
```

E2E tests are slower and flakier than unit tests, so use them only for the user-facing happy path of each slice — not for every edge case. Unit tests cover edge cases. E2E covers "did the wiring actually work."

### Pattern 5 — Flag-default-OFF production landing

Every slice ships behind a feature flag, default OFF, in production. The merge to main lands the code; the flag flip lands the feature.

```ts
if (await featureFlag('native-password-reset', { tenantId })) {
  // new path
} else {
  // old path
}
```

**Why:**
- Revert is one config change away — no redeploy.
- You can roll out by tenant: turn on for one customer to validate, then 10, then everyone.
- If something breaks, the blast radius is bounded by the flag.

**The flag default:** OFF. Always. Even on the day you intend to ship. Turn it on as a separate, deliberate operation.

**After stable rollout:** remove the flag. Flag debt accumulates if you don't. Six-month-old "transient" flags become permanent dead branches.

---

## Worked example

### Feature: native password reset (replacing a third-party identity provider)

#### Slice 1 — Render the reset page

**Outcome:** User navigates to `/reset-password` and sees a form with email input + submit button.

**Contract test:**
```ts
test('reset password page renders', async ({ page }) => {
  await page.goto('/reset-password');
  await expect(page.getByRole('heading', { name: 'Reset your password' })).toBeVisible();
  await expect(page.getByLabel('Email address')).toBeVisible();
});
```

**Implementation scope:**
- New route at `/reset-password`
- Component renders a form
- Submit button is disabled (no backend yet)

**Flag:** `native-password-reset`, default OFF.

**Manual verification:**
| # | Step | Expected result |
|---|------|-----------------|
| 1 | Navigate to /reset-password | Page renders form within 1s |
| 2 | Try to submit empty form | Submit button is disabled (no backend) |

**Rollback:** Flip flag to OFF (seconds).

**Estimate:** Half a day.

#### Slice 2 — Wire the reset request to the backend

**Outcome:** User submits the form; backend records the request; user sees a "check your email" confirmation.

**Contract test:**
```ts
test('reset password request creates a token', async ({ page, request }) => {
  await page.goto('/reset-password');
  await page.getByLabel('Email address').fill('user@example.com');
  await page.getByRole('button', { name: 'Send reset link' }).click();
  await expect(page.getByText('Check your email')).toBeVisible();

  // Backend assertion
  const tokens = await request.get('/internal/test/password-reset-tokens');
  expect(await tokens.json()).toContainEqual(expect.objectContaining({
    email: 'user@example.com',
  }));
});
```

**Implementation scope:**
- POST endpoint that creates a reset token (still behind the flag)
- Frontend wiring to call the endpoint
- Confirmation screen

**Flag:** same flag, still OFF in production.

**Manual verification:** {table with steps}

**Rollback:** Flip flag to OFF; existing tokens become orphans (acceptable).

**Estimate:** 1 day.

#### Slice 3 — Email delivery

**Outcome:** User receives an email with a working reset link.

(Continues similarly through remaining slices.)

#### Slice N — Flag flip and old-path removal

After all slices are stable in production for {N days}:

1. Default the flag to ON for all tenants.
2. Monitor for {N days}.
3. Remove the flag and the old code path.
4. Bump major version (this is a behavior change for any consumer relying on the old flow).

---

## Anti-patterns

### One giant slice

"I'll build the whole thing on a feature branch, then we'll merge it at the end."

**Why it's wrong:** No customer feedback until the end. Can't revert pieces. If anything breaks, the only undo is "revert the whole feature."

**Fix:** Cut into slices, even if you don't deploy them separately. Each slice is at least one PR.

### Horizontal slicing

"Slice 1 = schema. Slice 2 = API. Slice 3 = UI."

**Why it's wrong:** No slice delivers user-visible value until the last one. Nothing to verify, nothing to roll back independently.

**Fix:** Vertical slices that touch every layer for one small outcome.

### Implementation before contract test

"I'll write the code, then write tests at the end."

**Why it's wrong:** Tests written after the code tend to match the code, not the spec. They pass; they don't verify.

**Fix:** Failing test first. Always.

### No flag, "we'll be careful"

"This is a small change, we don't need a flag."

**Why it's wrong:** Small changes break things too. Without a flag, revert is a redeploy (minutes) instead of a flag flip (seconds). When a customer hits the bug, you want seconds.

**Fix:** Flag-gate anything user-visible. Especially small changes.

### Permanent flags

"We launched the feature; we'll remove the flag later."

**Why it's wrong:** "Later" never comes. Six months later, the flag is dead code that confuses every reader.

**Fix:** Plan flag removal as part of the slice plan. Schedule it.

---

## Adapting to your stack

This workflow is stack-agnostic, but the tools differ:

| Concern | TypeScript/Node | Rust | Python |
|---------|----|----|----|
| Pure logic test | vitest / jest | `#[test]` in cfg(test) | pytest |
| HTTP contract test | supertest / vitest | reqwest + integration tests | pytest + httpx |
| E2E browser test | Playwright | Playwright via JS test, or headless_chrome crate | Playwright via Python bindings |
| Feature flag | LaunchDarkly / GrowthBook / homegrown env var + DB | same | same |

The patterns are the same; the tools change.

---

## Audit log

| Date       | Reviewer  | Status    | Notes                |
|------------|-----------|-----------|----------------------|
| {date}     | {@user}   | created   | Initial reference.   |
