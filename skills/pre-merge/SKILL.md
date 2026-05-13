---
name: pre-merge
description: Run the full pre-merge checklist — tests, lint, build, version bump, doc updates, self-review of the diff. Use before merging any feature branch to main/beta/staging.
---

# /pre-merge — the pre-merge checklist

Before any merge to a shared branch (main, beta, staging, production), this skill runs through the full checklist that prevents bad code from shipping.

This is not a quick scan. It's a deliberate, sequential verification of every check that matters. Cutting corners here is how broken code reaches production.

## When to use this skill

Invoke before merging to a shared branch. Especially:

- Before merging to `main` (which usually deploys to production)
- Before merging to `beta` / `staging` (which deploys to customer-facing test environments)
- Before opening a PR that will trigger CI

If you're committing to your own feature branch (no push to shared), this skill is overkill. Use the trivial pre-commit checks instead (lint, build, run the test you wrote).

## The checklist

This is a sequential checklist. Run each step. Report each as PASS or FAIL. **Do not skip a step**, even if "obviously fine."

### 1. Diff overview

Run `git diff main..HEAD --stat` (or the appropriate base branch). Report:

- Number of files changed
- Lines added / removed
- Files outside the expected scope (flag for review)

If the diff is much larger than expected for the work described, **stop and investigate**. Scope creep is a common bug source.

### 2. Diff content self-review

Run `git diff main..HEAD`. Re-read your own diff as if a stranger wrote it. Look for:

- [ ] Debug code: `console.log`, `dbg!`, `print`, `pp` — flag every one
- [ ] Commented-out code blocks — flag every one
- [ ] Added TODOs without tracking — flag every one
- [ ] Accidental edits in unrelated files
- [ ] Strings or names that callers may reference (renames, especially)
- [ ] Comments that lie (claim something the code doesn't do)
- [ ] Sensitive values (tokens, keys, secrets) — STOP if found
- [ ] Backup files (`.bak`, `.orig`, `~`) — should be gitignored, not committed

Report each finding. If anything sensitive is found, **stop immediately** and ask the user before continuing.

### 3. Tests

Run the test suite. The exact command depends on the project — common patterns:

- TypeScript: `npm run test:unit` or `npx vitest run` (NEVER `npm test` if it's watch mode)
- Rust: `cargo test`
- Python: `pytest` or `python -m pytest`
- Go: `go test ./...`

Report:
- PASS / FAIL
- Test count
- Time
- If FAIL: which tests failed, with file:line and one-line error

If any test fails, **stop**. Investigate. Either fix the test (if your change broke it legitimately) or fix the underlying code (if the test caught a real bug).

### 4. Lint

Run the linter on changed files. Examples:

- TypeScript: `npx eslint $(git diff main..HEAD --name-only | grep -E '\.(ts|tsx|js|jsx)$')`
- Rust: `cargo clippy --all-targets --all-features -- -D warnings`
- Python: `ruff check .` or `pylint <files>`

Report PASS / FAIL. If FAIL: which rule, which file:line.

**Important**: lint only changed files unless the linter is fast. Linting the whole repo on every check is wasteful. But ensure you're not skipping files that the merge will introduce changes to.

### 5. Formatter

Run the formatter check (not write).

- TypeScript: `npx prettier --check $(git diff main..HEAD --name-only | grep -E '\.(ts|tsx|js|jsx|md|json)$')`
- Rust: `cargo fmt --check`
- Python: `black --check .` or `ruff format --check`

Report PASS / FAIL. If FAIL: which files are formatted incorrectly.

**Especially important**: prettier with a too-narrow glob misses files merged in from other branches. Run the formatter against the **changed files**, including markdown.

### 6. Type checking

Run the type checker.

- TypeScript: `npx tsc --noEmit`
- Rust: `cargo check --all-targets`

Report PASS / FAIL.

### 7. Full build

**RULE: Do not skip this step.** `tsc --noEmit` and unit tests can pass while the production build fails (different config, different bundler, different env).

- TypeScript: `npm run build` (or the project's actual build command)
- Rust: `cargo build --release`
- Whatever your Docker / CI build does, run it

Report PASS / FAIL. If FAIL: the error message. Investigate before proceeding.

### 8. Version check

Run `cat package.json | grep version` (or the equivalent for your project). Compare to `git show main:package.json | grep version`.

Report:
- Current branch version: X.Y.Z
- Main version: A.B.C
- Bumped? YES / NO

If NO and this change ships (is not internal-only): **bump the version**. Use semver:
- Patch for bug fixes, doc-only changes, minor enhancements
- Minor for new features, significant refactors
- Major for breaking changes

Ask the user to confirm the bump level if it's ambiguous.

### 9. Documentation check

For changes that touch features, APIs, or behavior visible outside the code:

- Did you update `docs/` for any user-visible change?
- Did you update `ERRORS.md` if you added new error handling?
- Did you update the PRD's `implemented` flag if this completes PRD work?
- Did you add an audit log entry to any doc you edited?

Report list of docs that should have been updated but weren't. If any, flag for the user.

### 10. Manual UI verification (for user-facing changes)

If the change is user-visible:

- Have you tested it in a real browser?
- Have you tested the golden path?
- Have you tested edge cases (empty state, error state, mobile)?
- Have you verified the build runs against a production-like backend, not just dev?

If you (the assistant) cannot test the UI yourself, **state that explicitly**:

> "I cannot test the UI from here. Have you manually verified this in a browser? If not, recommend doing so before merging."

Do not skip this step. Type checking and tests do not verify UI behavior.

### 11. The diff-vs-PR-description check

If a PR has been opened:

- Does the PR description match what the diff actually does?
- Are screenshots / GIFs included for UI changes?
- Is the test plan in the PR description?

If not, flag for the user to update the PR before merging.

### 12. Summary

Output a final report:

```
Pre-merge check: {branch} → {target}

[PASS] 1. Diff overview (12 files, +340/-87 lines)
[PASS] 2. Diff self-review (no debug code, no leaks)
[PASS] 3. Tests (147 passed, 0 failed, 3.2s)
[PASS] 4. Lint (clean on 8 changed TS files)
[PASS] 5. Formatter (clean)
[PASS] 6. Type check (clean)
[PASS] 7. Build (succeeded in 18s)
[PASS] 8. Version (bumped 1.4.7 → 1.4.8)
[FAIL] 9. Documentation (docs/api.md describes old behavior — update needed)
[INFO] 10. Manual UI: cannot test from here — recommend you verify in a browser

Result: NOT READY TO MERGE — fix doc drift first.
```

If any step is FAIL, the result is **NOT READY**. Do not merge.

If all steps are PASS (and INFO items don't block), report **READY** but **DO NOT MERGE** without explicit user permission (see `claude/git-workflow.md`).

## Example conversation

> **User**: `/pre-merge`
>
> **Assistant**: Running pre-merge checks on `feat/dark-mode` → `main`.
>
> *(Bash: git diff main..HEAD --stat)*
>
> Step 1/12: 8 files changed, +210/-12 lines. Scope looks consistent with "add dark mode toggle." No unrelated files. PASS.
>
> *(Bash: git diff main..HEAD | head -500, reads diff)*
>
> Step 2/12: Diff self-review. Found one `console.log('theme:', theme)` in `src/theme/store.ts:23`. Flagging for removal.
>
> Want me to remove that console.log first, or keep going and you'll fix it?
>
> **User**: Remove it.
>
> *(Edit src/theme/store.ts — remove console.log)*
>
> Continuing.
>
> Step 3/12: Tests... *(npx vitest run)* ... 147 passed. PASS.
>
> *(Continues through all 12 steps)*
>
> ...
>
> Result: READY TO MERGE.
>
> Do you want to merge now? Reminder: this is a push to main, which deploys to production.

## What to refuse

- **Skipping any step.** Even if "obviously fine." The point of the checklist is discipline.
- **Auto-merging at the end.** Even if everything passes, the merge is the user's call. See `claude/git-workflow.md`.
- **Treating warnings as PASS.** Warnings are signal. If lint warns, decide whether to fix or document.
- **Running on the wrong base branch.** Confirm with the user which branch they're merging to before starting.

## When to abbreviate

For trivial changes (typo fix, doc-only edit), the full 12-step protocol is overkill. Run:

1. Diff overview (sanity check)
2. Diff content self-review
3. Tests (always)
4. Lint
5. Formatter
6. Build
7. Version (if applicable)

Skip the doc, UI, and PR-description checks unless they apply.

For changes that touch the build system, security, or auth, run the **full** protocol regardless of perceived size. These are the changes most likely to have subtle effects.

## What "done" means for this skill

- All applicable checklist steps have been run.
- Each result is reported with PASS / FAIL / INFO.
- Failures are addressed or documented.
- The user has a clear READY / NOT READY signal.
- The merge has NOT happened automatically — that requires explicit user permission.
