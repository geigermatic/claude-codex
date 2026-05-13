# Git Workflow — safety, versioning, commit messages

Git operations are not local. They affect shared branches, CI pipelines, deployments, and other engineers' work. This file covers what you must never do without permission, how to version your changes, and how to write commit messages that survive CI/CD pipelines that parse them.

For the pre-merge checklist itself, see `skills/pre-merge/`. For PR conventions, see `claude/documentation.md`.

---

## 1. Never push without permission

**RULE:** Every `git push` requires explicit user permission for the specific push. Past permissions do not transfer.

**Why:** Pushing to a shared branch triggers CI, deploys, and other people's pulls. A push that breaks `main` blocks the team. A push to `beta` may auto-deploy to a customer-facing environment. There is no undo for a deploy.

### What "explicit permission" means

- "Push it" / "ship it" / "let's push" — yes, for that push only.
- "Looks good" — no. Ambiguous. Ask: "Should I push?"
- "Approved earlier in this session" — no. Each push is its own ask.
- "The user said merge it once" — no. The next merge is a separate decision.

### What requires permission

- `git push` (any branch)
- `git push --force` or `git push --force-with-lease`
- `git merge` followed by a push
- `git rebase` of a branch that has been pushed
- Creating a PR (CI triggers, reviewers get pinged)

### What doesn't require permission

- Local commits
- Local branch creation
- Local stash / cherry-pick / merge to a local-only branch
- Reading: `git log`, `git status`, `git diff`

**The rule of thumb:** if it changes state outside your machine, ask.

---

## 2. Never merge to main without permission

**RULE:** Merging to your project's production branch (`main`, `master`, `prod`, or whatever you call it) is the highest-stakes git action. Always ask, even if tests pass, even if "the user said ship it" earlier.

**Why:** Merge-to-main typically triggers a production deploy. Production deploys affect customers. They are not reversible without another deploy.

### The pattern

1. Push your feature branch.
2. Open a PR (after asking — see §1).
3. Run full tech review locally: TSC, tests, lint, build, doc updates, version bump.
4. Report the state of the PR.
5. **Stop. Wait for "let's merge" / "merge it" from the user.**
6. Merge.

### Failure modes to avoid

- "Tests pass and the diff looks clean, so I'll merge." → No. The diff is one input; the decision is the user's.
- "The user is busy; I'll merge to save time." → No.
- "Urgency framing" ("beta is broken, we need to ship") — even then, ask. Especially then.
- "The user approved version N earlier" — that approval was for version N. N+1 is a new ask.

---

## 3. Never use destructive shortcuts

Destructive shortcuts are tempting when something goes wrong. They are almost never the right answer.

### Forbidden without explicit user approval

- `git push --force` to a shared branch
- `git reset --hard` when there are uncommitted changes
- `git clean -fd` (deletes untracked files)
- `git checkout .` / `git restore .` (discards working tree)
- `git branch -D` on someone else's branch
- Deleting `.gitignore`'d files that turn out to be in someone's working tree

### What to do instead

- **Conflicts on merge/rebase:** resolve them. Don't `--abort` and retry hoping for different conflicts.
- **Want to throw away local work:** stash it first (`git stash`), so you can recover if needed.
- **Accidentally committed to wrong branch:** create the correct branch from current HEAD, then reset the wrong branch back. Don't force-push to fix it.
- **Force push needed for legitimate reasons (rebased a feature branch):** use `--force-with-lease` and confirm with the user.

### "But I'm sure"

When you find yourself thinking "I'm sure this is fine to discard" — that's the moment to pause and ask. The 30 seconds to confirm is worth more than the work you might destroy.

---

## 4. Never bypass hooks

Pre-commit and pre-push hooks exist for a reason: lint, format, type-check, test. They are not arbitrary.

**RULE:** Do not use `--no-verify` (which skips hooks) unless the user has explicitly requested it.

### Why

- A failing hook is a signal. Fix the underlying issue.
- Skipping hooks merges broken code that CI will reject (or worse, deploy).
- Hooks are the team's first line of defense.

### When you might legitimately bypass

- A genuinely-broken hook (e.g., a tool is misconfigured locally) — but fix the hook, don't ship around it.
- An emergency where the underlying issue is being tracked separately — and the user explicitly says "skip the hook."

In every case: ask first. Document why in the commit message.

---

## 5. Commit messages: format

Follow **Conventional Commits**: `<type>(<scope>): <description>`.

### Types

- `feat` — new feature
- `fix` — bug fix
- `docs` — documentation only
- `refactor` — code change that neither fixes nor adds
- `test` — test-only change
- `chore` — tooling, dependencies, build
- `perf` — performance improvement

### Scope

The affected area: `feat(auth):`, `fix(api):`, `docs(readme):`. Use lowercase. Keep it short.

### Description

- Imperative mood: "add login button" not "added login button" or "adds login button."
- Lowercase first letter.
- No trailing period.
- Under 72 characters total for the subject line.

### Examples

- `feat(auth): add OAuth callback handler`
- `fix(search): handle empty query string`
- `docs(api): clarify pagination behavior`
- `refactor(db): extract connection pool into module`

---

## 6. Commit messages: body

Use the body when the change needs context beyond what fits in the subject.

### Format

- Blank line after the subject.
- Wrap at 72 characters.
- Explain **why**, not what. The diff shows what.
- Reference issues/PRs by ID where relevant.

### Example

```
fix(search): handle empty query string

Previously, an empty query produced a 500 because the parser
assumed at least one term. The fix returns an empty result set
with a 200, matching the documented behavior.

Resolves #1234.
```

### When to skip the body

- The subject is self-explanatory (typos, renames, one-line fixes).
- The diff is tiny and obvious.

When in doubt, write the body. Future-you (or a colleague) reading `git log` will thank you.

---

## 7. Commit messages: safety against CI parsing

Modern CI pipelines often **parse commit messages** to populate Slack notifications, release notes, JIRA tickets, or deploy logs. Some parsers choke on certain characters.

### RULE: avoid these in commit subject lines

- **Backticks** (`` ` ``) — can crash YAML payload generation in Slack workflows
- **Single quotes / double quotes** at the start or end — confuse some parsers
- **Newlines in subjects** — never; subjects are one line
- **Triple-quoted blocks** in subjects — same issue as quotes

### Why this matters

A commit that breaks the CI notification pipeline doesn't break the deploy — the deploy may have already succeeded. But the team doesn't know it succeeded because the notification never posted. You learn about the broken notification an hour later when someone asks "did v1.2.3 ship?"

This has cost real teams real build cycles. Defensive commit-message hygiene is cheap.

### Safe alternatives

- Instead of `` `loginButton` ``, write "the login button" or "LoginButton".
- Instead of `add "smart" search`, write `add smart search` or `add fuzzy search`.

### If you must reference code in a message

Put it in the body, not the subject. Bodies are less likely to be parsed by CI integrations.

---

## 8. Versioning

For projects with a user-visible version (`package.json`, `Cargo.toml`, or a `VERSION` file):

### When to bump

- **Patch (`0.0.x`):** bug fixes, doc-only changes that ship with code, minor enhancements.
- **Minor (`0.x.0`):** new features, significant refactors visible to users.
- **Major (`x.0.0`):** breaking changes to APIs, schemas, or behavior.

### Always bump before merging

If a change ships, it gets a version bump. The exception is internal-only changes (CI tweaks, repo settings) that don't ship.

**Rationale:** the version is how you verify a deploy landed. If you don't bump, you can't tell "did v1.2.3 reach prod?" from looking at the running app.

### Single source of truth

Pick one canonical location for the version (`package.json` `version` field is standard for Node, `Cargo.toml` for Rust). All other references derive from it.

### Display the version in the running app

A version string in the UI footer, sidebar, or status endpoint is the cheapest deploy-verification tool there is. Add it.

---

## 9. Commit titles lead with version when bumping

**RULE:** When a commit bumps the version, the commit title leads with the new version.

### Format

```
v1.2.3 fix(scope): description
```

Plain space separator between the version and the type prefix. No leading `v` colon, no `[`, no special markup.

### Example

```
v0.9.42 fix(auth): handle expired refresh tokens
```

### Why

`git log` becomes scannable: "what shipped in v0.9.42?" → grep for the version prefix. No need to cross-reference release notes.

### Apply only to version-bumping commits

Other commits keep plain Conventional Commits format (`fix(auth): ...`). The version prefix is the signal that "this commit shipped a version."

---

## 10. Pre-merge checklist

Before merging any feature branch, run through this list:

- [ ] Tests pass locally
- [ ] Build succeeds locally (not just type-check — full build)
- [ ] Lint passes on all changed files
- [ ] Formatter (prettier, rustfmt) clean
- [ ] Version bumped (if shipping)
- [ ] Documentation updated for any user-visible change
- [ ] No debug code left in (console.log, dbg!, print)
- [ ] No commented-out code
- [ ] No TODOs added without tracking
- [ ] Diff re-read as a stranger
- [ ] Commit messages clean

For the interactive version, invoke `/pre-merge`.

---

## 11. Branch naming

Pick a convention and stick to it. Common options:

- `feat/<short-description>` — features
- `fix/<short-description>` — bug fixes
- `chore/<short-description>` — tooling
- `docs/<short-description>` — docs-only

Short. Lowercase. Hyphens, not underscores or spaces. No trailing dates or initials.

### Avoid

- Personal-name branches (`alice/some-work`) — they don't survive team handoffs
- Long descriptions in the branch name — the PR title is the description
- Branches without a prefix — makes `git branch` listings unscannable

---

## 12. The PR description

Every PR has a description. The PR description is where the "what changed and why" lives.

### Structure

```markdown
## Summary
One or two sentences. What this PR does.

## Context
Why this change. What problem it solves. Link to issue/PRD if relevant.

## Changes
- Bullet list of the substantive changes
- One bullet per concern

## Test plan
- How you verified this works
- Any manual checks performed
- Edge cases considered

## Risks
- What could go wrong
- What you mitigated
- What you accepted

## Rollback
- How to revert if something breaks
```

Skip sections that don't apply. A typo fix doesn't need a Risks section.

### What goes in the description, not the commit

- The full reasoning for the change
- Screenshots / GIFs of UI changes
- Stakeholder context ("Sarah needs this for the Friday demo")
- Test evidence ("Verified by running X with input Y, got Z")

Commits explain *what* and *why-briefly*. PRs explain *why-in-detail* and *how to verify*.

---

## 13. Forks and external contributions

If your project accepts external PRs:

- Set up CI to run on PRs from forks (with appropriate safety — no secrets in PR-from-fork builds).
- Require approving review from a maintainer before merging.
- Have a `CONTRIBUTING.md` that sets expectations.

If you contribute to others' projects:

- Fork, branch from default, push to your fork, open PR.
- Read their contribution guide first.
- Small PRs get reviewed; large PRs sit.

---

## 14. The recovery playbook

When something goes wrong, follow this order:

1. **Stop making changes.** Don't try to "fix forward" if you don't understand the situation.
2. **Assess:** what is broken, who is affected, what's the blast radius?
3. **Communicate:** tell the team / user the state. Don't go dark.
4. **Roll back if reversible.** A revert PR + redeploy is usually faster than diagnosing a fix.
5. **Diagnose with the system in a safe state.** Don't debug on a fire.
6. **Fix, with a test.** A bug that broke prod gets a regression test.
7. **Post-mortem.** What allowed this? Add the rule that would have prevented it.

### What not to do during a fire

- Force-push.
- "Cherry-pick" hot fixes across branches without testing.
- Skip CI to ship faster.
- Delete logs / branches / state "to clean up."

The instinct in a crisis is to act fast. The discipline is to act *correctly* — which is usually slower than feels right.
