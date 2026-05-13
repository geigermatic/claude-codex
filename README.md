# claude-codex

**A codex of engineering practices for Claude Code — TDD, verification, code quality, and SDLC discipline.**

Turn vibe coding into engineering. This kit captures the principles, skills, and templates that separate world-class AI-assisted software development from "ask the LLM and hope."

> Built for [Claude Code](https://claude.com/claude-code). Adaptable to Cursor, Aider, Cline, Continue, and other coding agents — see [ADAPTERS.md](ADAPTERS.md).

---

## Why this exists

Default LLM-assisted coding has predictable failure modes:

- The agent **guesses** instead of verifying. It invents APIs that don't exist, edits files it didn't read, claims fixes work without running the tests.
- The agent **expands scope** unprompted. A one-line bug fix grows into a refactor. Comments multiply. Dead code stays "just in case."
- The agent **skips the boring discipline**. No failing test first. No version bump. No lint. No documentation update. Ship it.
- The agent **gives generic answers**. No tradeoff articulation. No confidence calibration. No tying recommendations to the actual codebase in front of it.

This kit fixes those failure modes through three artifacts:

1. **Principles** — modular, opinionated rules that load into your agent's context (`CLAUDE.md` and the `claude/` directory).
2. **Skills** — interactive, invokable workflows that walk the agent and the user through high-value SDLC moments: writing a PRD, planning a TDD slice, fixing a bug test-first, reviewing before merge.
3. **Templates** — battle-tested structures for PRDs, error catalogues, doc audits, and starter `CLAUDE.md` files.

Used together, these turn an enthusiastic-but-undisciplined agent into something closer to a senior engineer with judgment.

---

## What's in the box

```
claude-codex/
├── CLAUDE.md                          # entry point — imports the principles
├── claude/
│   ├── EPISTEMICS.md                  # how the agent should THINK: verify, don't guess
│   ├── principles.md                  # architecture, code quality, naming
│   ├── testing.md                     # TDD, pyramid, hypothesis-driven debugging
│   ├── security.md                    # input validation, secrets, deps, tenant isolation
│   ├── token-economy.md               # efficient tool use, context discipline
│   ├── git-workflow.md                # safety rails, versioning, commit messages
│   ├── documentation.md               # PRDs, audit blocks, freshness
│   └── languages/
│       ├── typescript.md              # strict mode, discriminated unions, zod
│       └── rust.md                    # ownership, errors, async, type-state
├── skills/
│   ├── prd-author/                    # /prd-author — interactive PRD draft
│   ├── tdd-slice/                     # /tdd-slice — slice-based TDD planner
│   ├── bug-fix-tdd/                   # /bug-fix-tdd — failing test first
│   ├── verify-claim/                  # /verify-claim — turn assertions into evidence
│   ├── pre-merge/                     # /pre-merge — preflight + self-review
│   └── doc-audit/                     # /doc-audit — frontmatter + freshness sweep
├── templates/
│   ├── CLAUDE.md.starter              # opinionated CLAUDE.md for a new project
│   ├── PRD.md                         # 7-section PRD with Manual UI Test Checklist
│   ├── ERRORS.md                      # starter error catalog format
│   ├── doc-audit.md                   # frontmatter + audit block schema
│   └── SLICE-WORKFLOW.md              # slice-based TDD reference
├── ADAPTERS.md                        # use this kit with Cursor, Aider, Cline, Continue
├── LICENSE                            # MIT
└── README.md
```

---

## Quick start

### Option 1 — Drop it into a Claude Code project (5 minutes)

```bash
cd your-project/
git clone https://github.com/geigermatic/claude-codex.git .claude-codex
cp .claude-codex/templates/CLAUDE.md.starter ./CLAUDE.md
# Edit CLAUDE.md to add your project's specifics, then commit.
```

Claude Code auto-loads `CLAUDE.md` from your project root. The starter file imports the principles via `@.claude-codex/claude/...` references — your agent now operates with the full discipline layer active.

To use a skill: type `/prd-author`, `/tdd-slice`, `/bug-fix-tdd`, `/verify-claim`, `/pre-merge`, or `/doc-audit` in chat.

### Option 2 — Install globally for all your Claude Code projects

```bash
git clone https://github.com/geigermatic/claude-codex.git ~/.claude/codex
# Reference ~/.claude/codex/claude/*.md in your ~/.claude/CLAUDE.md (user-level)
```

### Option 3 — Use the principles with another agent

See [ADAPTERS.md](ADAPTERS.md) for mappings to Cursor (`.cursorrules`), Aider (`.aider.conf.yml`), Cline (`.clinerules`), and Continue (`config.json` system message). The principle files are tool-agnostic — they apply anywhere.

---

## What good looks like with this kit active

**Before:**
> Add a dark mode toggle.

→ Agent edits five files, introduces a `ThemeManager` class, adds three new dependencies, writes no tests, doesn't run the build, claims success.

**After:**
> Add a dark mode toggle.

→ Agent asks: "Slice this as (a) toggle only, no persistence, or (b) toggle + localStorage + system-preference detection?" Picks (a). Writes a failing test for the toggle component. Implements the minimum to pass. Runs the test. Asks before extending. Reports: *"Toggle works, persists for session only, tested. No localStorage yet — flag if you want it next."*

The difference is not the model. It's the operating procedure.

---

## Design choices

**Prescriptive, not suggestive.** Rules read as "MUST" / "NEVER" / "ALWAYS." Each non-obvious rule carries a **Why** and a **How to apply** — so the agent (and the human) can judge edge cases instead of blindly following.

**Modular over monolithic.** Each principle file is independently useful. Drop in `claude/testing.md` if all you want is TDD discipline. Drop in `claude/security.md` if all you want is input-validation rigour.

**Tool-agnostic principles, Claude-specific skills.** Principles work in any agent. Skills assume Claude Code's `Skill` system, `TodoWrite`, `Agent` subagents, and conversation flow. Other-agent users get the principles plus a porting guide.

**No project specifics.** This kit does not mention any specific framework, vendor, repo, or industry. The templates are scaffolding, not solutions.

**Lean enough to read, deep enough to matter.** Every file is substantive. There is no filler. There is also no skipping the awkward parts — verification, version bumping, manual UI test checklists — because those are exactly the parts that vibe coding skips.

---

## Contributing

This kit grew from real engineering scar tissue — incidents, near-misses, and the moments where "the agent just shipped something broken" turned into a written rule. If you have your own scar tissue worth codifying, open a PR.

Strong PRs include:
- The **principle or pattern** in the same prescriptive voice (rule + Why + How to apply).
- The **incident or class of incidents** that motivated it (in the PR description, not the file).
- A concrete **example** if the principle is non-obvious.

PRs that turn a single past mistake into a general rule are especially welcome.

---

## License

[MIT](LICENSE). Use, fork, adapt freely. Attribution appreciated but not required.

---

## Acknowledgements

Distilled from production engineering practice across Rust ingestion services, Node/TypeScript orchestration layers, React frontends, and LLM pipelines. The patterns here are the survivors — the ones that earned their place by preventing real outages, recovering real deploys, and producing real customer-grade software.

Built for [Claude Code](https://claude.com/claude-code) by Anthropic.
