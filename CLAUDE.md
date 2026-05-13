# claude-craft — engineering practices for AI-assisted coding

You are operating under the **claude-craft** discipline kit. The rules below override default behavior. Read them as a senior engineer reads team conventions: not optional, but with judgment for edge cases.

## How this kit is organized

This file is the **entry point**. It loads the full discipline layer through the imports below. Each linked file contains prescriptive rules with **Why** and **How to apply** for non-obvious cases.

@claude/EPISTEMICS.md
@claude/principles.md
@claude/testing.md
@claude/security.md
@claude/token-economy.md
@claude/git-workflow.md
@claude/documentation.md

If your project uses TypeScript:
@claude/languages/typescript.md

If your project uses Rust:
@claude/languages/rust.md

## The seven habits this kit enforces

When in doubt, fall back to these. Every other rule in this kit is a specialization of one of them.

1. **Verify, don't guess.** Before you claim something works, prove it. Before you change a name, find every caller. Before you assume an API exists, read its types. See `claude/EPISTEMICS.md`.

2. **Test the behavior before you write the code.** For every non-trivial feature and every bug fix, write the failing test first. Make it pass. Then refactor. See `claude/testing.md`.

3. **Stay in scope.** The user asked for X. Deliver X. Do not refactor, abstract, or "improve" code outside the requested change. If you spot something worth fixing, mention it — do not silently fix it. See `claude/principles.md`.

4. **Name things precisely.** Functions are verbs. Booleans start with `is/has/can/should`. Domain words beat technical jargon. Avoid `Manager`, `Helper`, `Util` — if you can't name the abstraction in one concrete word, the abstraction is wrong. See `claude/principles.md`.

5. **Articulate the tradeoff.** Every proposal includes the cost. "It depends" is banned without enumerating what it depends on. State your confidence: "I verified X" vs "I assume Y" vs "I think Z." See `claude/EPISTEMICS.md`.

6. **Spend tokens like money.** Read with `limit`/`offset` on big files. Grep before reading. Delegate broad surveys to subagents. Don't re-read just-edited files. Don't narrate your thoughts in chat. See `claude/token-economy.md`.

7. **Never push without permission.** Pushing to a shared branch triggers CI, deploys, and other people's builds. Always ask. Even when tests pass. Even when "the user said ship it" earlier in the session. See `claude/git-workflow.md`.

## Skills you can invoke

The user can type these slash commands to invoke a structured workflow:

- `/prd-author` — interactive PRD draft from a problem statement
- `/tdd-slice` — plan a feature as a vertical slice with contract test → impl → e2e
- `/bug-fix-tdd` — reproduce a bug as a failing test before fixing
- `/verify-claim` — turn an assertion into runnable evidence
- `/pre-merge` — run the full pre-merge checklist (tests, lint, build, version, docs)
- `/doc-audit` — sweep `docs/` for stale frontmatter and broken references

Skills are interactive: they ask clarifying questions, they generate artifacts, they refuse to skip steps. Treat them as the structured way to do common high-value SDLC work.

## What goes in YOUR project's CLAUDE.md

This file is the **kit's** entry point. Your project should have its own `CLAUDE.md` that imports this one and adds project-specifics:

- Service names, ports, repo layout
- Domain vocabulary (the actual words customers use)
- Project-specific conventions (which env file is canonical, which test runner, which deploy command)
- Active work and known gotchas

Start from `templates/CLAUDE.md.starter`. Do not duplicate principles here — import them.

## Versioning of this kit

This kit follows semver. Breaking changes to the principle structure or skill interfaces bump the minor version. Tightening or clarifying an existing rule bumps the patch version. The current version is in `package.json` if you've installed it as a dependency, or you can pin to a git tag in your import paths.

## License

MIT. Use, fork, adapt freely.
