---
name: bug-fix-tdd
description: Fix a bug test-first. Reproduce as a failing test before writing the fix. Use when the user reports a bug or asks you to fix a defect.
---

# /bug-fix-tdd — test-first bug fixing

You are helping the user fix a bug using strict test-first discipline. The protocol is non-negotiable: reproduce the bug as a failing test, **then** fix it. No shortcuts.

## When to use this skill

Invoke when the user reports a bug or asks you to fix a defect. Indicators:

- "X is broken"
- "Y returns the wrong thing when Z"
- "Why does the dashboard show 0 sometimes?"
- "Customer reports they can't log in"

If the user wants to add a feature, use `/tdd-slice` or `/prd-author`. If the issue is a typo or trivial cosmetic fix, just fix it without invoking this skill.

## Why test-first

A fix written before the test "works" because the developer wrote it to work. The test (written after) "passes" because the fix already passes. There is no independent verification that the fix actually addresses the reported bug.

A test written first describes the buggy behavior. It fails because the bug exists. When the fix makes it pass, you have proof that the fix addresses the **reported** bug — not some adjacent thing the developer happened to fix.

Bonus: the test becomes a regression guard. Six months later, if someone refactors and reintroduces the bug, the test catches it.

## The protocol

### Step 1 — Understand the bug

Ask the user:

> Walk me through the bug. What did you (or the user) expect, and what actually happened?

Probe for:
- **Exact reproduction steps.** "Click X, enter Y, see Z."
- **Expected behavior.** What should have happened.
- **Actual behavior.** What did happen.
- **Frequency.** Always, sometimes, only under specific conditions?
- **First-noticed timing.** Did this work last week? Did a recent deploy correlate?
- **Affected scope.** One user? One tenant? Everyone?

If you can't articulate the bug in one sentence after this, **ask more questions**. Vague bugs produce vague fixes.

### Step 2 — Find the relevant code

Search the codebase to locate the function, route, or module where the bug lives. Use Grep / Glob / Read — be efficient (see `claude/token-economy.md`).

State your hypothesis:

> Based on the symptom, the bug is most likely in `{file}:{line}` because {reasoning}. I want to confirm before writing the test.

Read the code. Confirm or revise the hypothesis. If the bug is somewhere else, restate the hypothesis with the new location.

**Do not start fixing yet.** The next step is the test.

### Step 3 — Write the failing test

Write a test that **demonstrates the bug**.

The test name describes the buggy behavior in plain language:

```
it('returns the correct user when the email contains a + sign')
it('handles concurrent writes to the same key without losing data')
it('does not throw when the input array is empty')
```

The test sets up the exact reproduction conditions and asserts the **expected** behavior. It will fail today because the bug is present.

### Step 4 — Run the test. Confirm it fails.

Run the test. Read the output.

**REQUIRED**: confirm the test fails for the reason you expected — not for an unrelated reason (e.g., setup error, import error, wrong file path).

If the test passes, one of these is true:
- The test doesn't actually exercise the bug. Refine it.
- The bug doesn't reproduce in test conditions. Investigate why; the test may not be hitting the same code path as production.
- The bug isn't where you think. Revise the hypothesis.

Do not move on until the test reliably fails **for the reason you intended**.

### Step 5 — Form the fix hypothesis

State what you think is causing the bug and what change will fix it.

> "The bug is in `parseEmail` — it splits on the first `@` but ignores `+` aliases. Fixing it requires preserving the part before `@` including any `+`. I expect to change `parseEmail` and only `parseEmail`."

The hypothesis includes:
- The cause (why does the bug happen?)
- The fix (what's the smallest change that addresses the cause?)
- The blast radius (what else might be affected by the fix?)

### Step 6 — Apply the fix

Make the minimum change to address the hypothesis. Do not refactor adjacent code. Do not "improve" things you notice while you're there. Stay in scope.

### Step 7 — Run the test. Confirm it passes.

Run the failing test. It should now pass.

If it doesn't, your hypothesis was wrong. **Revert the fix.** Form a new hypothesis. Do not stack changes hoping one of them will make the test go green.

### Step 8 — Run the FULL test suite

Run all tests in the affected module (or the whole suite if it's fast enough). Other tests must not break.

If something else fails: your fix had unintended consequences. Either:
- The other test was wrong and was passing by accident (rare; investigate carefully)
- Your fix broke real behavior (revert and refine)

### Step 9 — Read the diff

Re-read your own diff as a stranger. Check:
- [ ] Did you change only what the fix requires?
- [ ] Did you leave debug logs in?
- [ ] Did you accidentally edit anything outside scope?
- [ ] Is the change minimal?

If the diff includes things unrelated to the fix, **split them**. The bug fix is one commit; the cleanup is another.

### Step 10 — Document the fix

For user-visible bugs or operationally-significant errors:

- Add an entry to `docs/ERRORS.md` (or your project's error catalog) with the symptom, cause, fix, and date.
- If the bug came from a documentation drift, update the documentation too.

### Step 11 — Report

Report to the user, in two sentences:

> Fixed. {Specific symptom} now {expected behavior}. Test added at {test file:line}; full suite passes. Ready for your review.

## Example conversation

> **User**: `/bug-fix-tdd` — search is returning duplicate results when the user adds quotes around a phrase.
>
> **Assistant**: Walk me through it. What's the search query that breaks it, and what shows up?
>
> **User**: Search for `"acme corp"` (with quotes) and you get the same vendor listed three times.
>
> **Assistant**: One question: does this happen for any quoted phrase, or only for some? E.g., does `"hello world"` also dupe?
>
> **User**: Tested both. Both dupe.
>
> **Assistant**: OK. Let me locate the search code.
>
> *(Greps for search; reads relevant file)*
>
> Hypothesis: `tokenize()` in `src/search/parse.ts:42` strips quotes but doesn't de-duplicate the resulting terms. A query like `"acme corp"` produces tokens `[acme, corp, "acme corp"]` — three tokens, three search hits, no merge.
>
> Want me to verify by reading the function, or shall I write the test first based on the symptom?
>
> **User**: Verify first.
>
> *(Reads parse.ts:42)*
>
> Confirmed. The function emits both the unquoted tokens and the original quoted phrase, then `searchByTokens()` returns one hit per token without merging.
>
> Now the failing test:
>
> ```ts
> it('returns each matching document once when the query contains a quoted phrase', () => {
>   const docs = [{ id: '1', text: 'acme corp does things' }];
>   const results = search('"acme corp"', docs);
>   expect(results).toEqual([{ id: '1' }]);
> });
> ```
>
> Running it. ... Fails. Got 3 results, expected 1. Reason matches the hypothesis.
>
> Fix plan: in `searchByTokens`, dedupe by document ID before returning. Minimal change.
>
> Apply the fix? Or do you want to see the diff first?
>
> *(Continues through to fix + verify + diff review.)*

## What to refuse

- **Fixing without a failing test first.** No exceptions. If the user is in a rush, the test is what makes the rush safe. Two minutes of test-writing saves hours of "did this actually fix anything?"
- **Stacking changes.** If your first fix doesn't make the test pass, revert and rethink. Don't add a second fix on top.
- **Skipping the full suite.** Bug fixes always run the full affected test suite. Always.
- **Generic fixes.** If you find yourself adding `try/catch` to make the error disappear without understanding the cause, stop. The error is the signal. Find the cause.

## Special: bugs that are hard to reproduce

For race conditions, environment-dependent failures, or production-only bugs:

1. **Try to reproduce locally first.** Even if it takes setting up specific conditions, this is usually faster than debugging blind.
2. **If you genuinely cannot reproduce locally**, find the closest deterministic test. E.g., for a race condition, simulate the concurrency with explicit timing.
3. **If no deterministic test is possible**, add observability (logs, metrics) that would confirm the fix in production. Document this in the PR — "this fix cannot be unit-tested; verification will happen via X dashboard after deploy."

This is a fallback. Most bugs CAN be reproduced. Push for the reproduction before settling for observability-only verification.

## What "done" means for this skill

- The failing test is written and was confirmed to fail for the right reason.
- The fix is applied.
- The test now passes.
- The full test suite passes.
- The diff is minimal and reviewed.
- Documentation is updated where relevant.
- The user has been told what changed in two sentences.
