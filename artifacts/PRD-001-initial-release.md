---
id: "PRD-001"
title: "claude-codex v0.1.0 — initial release"
type: "PRD"
domain: "product-strategy"
applies_to: ["v0.1.0", "v0.1.1"]
status: "current"
confidence: "high"
last_verified: "2026-05-13"
implemented: true
target_version: "v0.1.0"
---

# claude-codex v0.1.0 — initial release

> Build a public, opinionated, MIT-licensed discipline kit that turns AI-assisted coding into engineering practice. Ship a complete v0.1.0 with principles, interactive skills, and templates — modular, tool-agnostic, and dogfooded against its own rules during construction.

> **Note on form:** This is a **retroactive PRD**. It documents what was built in v0.1.0 (and the v0.1.1 patch), why, and what was deferred. Future contributors should treat this as the **founding design document** — the artifact that captures original intent. Forward planning for v0.2.0 and beyond lives in [PRD-002](PRD-002-adoption-improvements.md).

## Contents

1. [Problem](#problem)
2. [Current State (before this work)](#current-state-before-this-work)
3. [Design](#design)
4. [Phases (as delivered)](#phases-as-delivered)
5. [Files Touched](#files-touched)
6. [Metrics](#metrics)
7. [Risks (encountered and resolved)](#risks-encountered-and-resolved)
8. [Manual UI Test Checklist (as-launched verification)](#manual-ui-test-checklist-as-launched-verification)
9. [Audit log](#audit-log)

---

## Problem

AI-assisted coding has predictable failure modes that erode trust and accumulate technical debt:

1. **Confident wrongness.** Agents claim fixes work without running tests, invent APIs that don't exist, edit files they didn't read, and report success on incomplete work.
2. **Silent scope creep.** A one-line fix grows into a three-file refactor. Adjacent code gets "improved." Dead code is preserved "just in case."
3. **Skipped discipline.** No failing test first. No version bump. No lint. No documentation update. The agent ships.
4. **Generic answers.** "It depends" with no enumeration. No tradeoff articulation. No calibration of confidence ("I verified X" vs "I assume Y").

These failure modes are not model limitations — they are **operating-procedure gaps**. The same model, operating under different rules, produces dramatically different work. But there was no portable, opinionated discipline kit that engineers and tech PMs could drop into their projects to establish those rules.

What existed:

- **Scattered best practices** in books, blog posts, and individual team wikis — none in a shippable form.
- **Agent-specific config files** (Claude Code's `CLAUDE.md`, Cursor's `.cursorrules`, Aider's `.aider.conf.yml`) with no opinionated starter content.
- **Prompt engineering guides** that taught prompt construction but not engineering discipline.

Nothing combined: prescriptive principles + interactive workflows + reusable templates + an explicit "verify, don't guess" epistemic stance.

**Why now:** AI-assisted coding adoption is at an inflection. The next year of practice will codify habits that compound for the rest of the decade. A discipline kit shipped now — when teams are forming their AI usage habits — has higher leverage than the same kit shipped two years later, when bad habits have calcified.

---

## Current State (before this work)

A snapshot of the landscape v0.1.0 was launching into:

### What existed and worked

- Tool-specific config systems (`CLAUDE.md`, `.cursorrules`, etc.) provided the *mechanism* to load discipline rules into agents.
- Prompt-caching infrastructure (Anthropic's API and others) made it cheap to load long discipline files repeatedly.
- Public engineering practice was well-documented in textbooks, blog posts, and conference talks — the *content* of good practice was known.

### What was missing

- **Opinionated, ready-to-drop-in kits** combining principles + workflows + templates. Existing assets were either too generic (good practice in the abstract) or too narrow (a single template, a single rule list).
- **Skills/workflows that walked the agent through specific high-value SDLC moments** — writing a PRD, planning a TDD slice, fixing a bug test-first. Templates told you the shape; skills could enforce the protocol.
- **Adapters across agents.** Teams using Cursor, Aider, Cline, or Continue had no clear path to use the same discipline layer as Claude Code teams.
- **An epistemic layer addressing AI's specific failure mode** (confident wrongness). Engineering discipline books predate agents; they don't speak to "the agent guessed instead of verifying."

### Adjacent prior art

- LangChain templates, prompt-engineering guides, and a small number of "rules for AI coding" repos existed. None had the principles-skills-templates triad with this kit's depth or the dogfooding discipline (the kit treating itself as a real product, with PRDs, audit logs, manual UI test checklists).

This was the gap v0.1.0 set out to fill.

---

## Design

The kit's design rests on **three pillars** and several explicit cross-cutting decisions.

### The three pillars

1. **Principles** (`claude/*.md`) — modular, prescriptive rule files. Tool-agnostic. Each file is independently useful. A team can adopt one (`testing.md`) without adopting all.
2. **Skills** (`skills/*/`) — interactive workflows the agent invokes for specific high-value moments. Each skill is a structured protocol: ask clarifying questions, generate an artifact, refuse to skip steps.
3. **Templates** (`templates/*.md`) — battle-tested scaffolds for PRDs, error catalogs, doc-audit schemas, slice plans, and a starter `CLAUDE.md`.

A team can use principles without skills, or templates without principles, or all three together. Modularity is intentional — the highest-friction path is "all or nothing."

### Key decisions

#### Decision 1: prescriptive voice with explicit "Why" and "How to apply"

- **Choice:** Rules read as "MUST", "NEVER", "ALWAYS." Each non-obvious rule carries a **Why:** line (the reason behind the rule, often a class of incident) and a **How to apply:** line (when the rule actually kicks in).
- **Tradeoff:** Risks reading as dogma to engineers who reflexively dislike prescription.
- **Alternative considered:** Softer, recommendation-shaped framing ("consider", "you might want to").
- **Why rejected:** Recommendations get ignored. Prescriptive rules with explicit reasoning give the agent (and the human) something to either follow or consciously override. Decision-making is sharper than vague-suggestion-following.

#### Decision 2: epistemic discipline as a top-level file, not a section

- **Choice:** `claude/EPISTEMICS.md` is a dedicated file covering verification, confidence-language hygiene, hypothesis-driven debugging, and self-review. It is the longest principle file and the one most often referenced.
- **Tradeoff:** Some content (e.g., self-review) overlaps with `principles.md` and `testing.md`.
- **Alternative considered:** Distribute the epistemic rules across other files where they apply.
- **Why rejected:** The single biggest failure mode of AI-assisted coding is *confident wrongness*. Treating its remedies as scattered sub-rules buries the most important content. Concentrating them in a marquee file signals priority.

#### Decision 3: tool-agnostic principles + Claude-specific skills

- **Choice:** Principle files reference "your agent's tools" generically. Skills assume Claude Code's `Skill` system, `TodoWrite`, `Agent` subagents.
- **Tradeoff:** Skills require porting work for users of other agents.
- **Alternative considered:** Make everything tool-agnostic (no skills system at all, or skills as plain prompts).
- **Why rejected:** Skills derive most of their value from being *invocable* (`/prd-author`) and from the agent's ability to maintain state, call tools, and emit structured output. A "tool-agnostic skill" is just a long prompt — a fraction of the value.
- **Mitigation:** [ADAPTERS.md](../ADAPTERS.md) maps Claude-specific constructs (TodoWrite, subagents, slash commands) to their equivalents in Cursor, Aider, Cline, Continue, and others.

#### Decision 4: markdown-only, no build step

- **Choice:** Every artifact is markdown. No JavaScript, no static-site generator, no package manager, no compile step. `git clone` is the full installation.
- **Tradeoff:** Visitors browse raw markdown on github.com — less polished than a rendered docs site.
- **Alternative considered:** GitHub Pages or a docs-site renderer.
- **Why rejected:** Maintenance burden disproportionate to launch benefit. GitHub renders markdown well enough. Defer rendered docs to a later phase if adoption signals justify.

#### Decision 5: MIT license, public from day one

- **Choice:** MIT license; repo public the moment it has substance.
- **Tradeoff:** No "internal" period to iterate without external eyes.
- **Alternative considered:** Apache 2.0, CC-BY 4.0, dual-license, or private-first.
- **Why rejected:** MIT has the highest adoption / lowest friction. Enterprise-acceptable. Anthropic uses it for its own Claude tooling. Public-from-day-one creates a forcing function for quality and prevents the "internal-only" trap (most "we'll open-source it later" projects never do).

#### Decision 6: dogfood the kit against itself during construction

- **Choice:** The kit was built using its own discipline. Verification before claims. Tradeoffs articulated. Principles applied to the principle files themselves.
- **Tradeoff:** Slower to build than vibe-coding the content would have been.
- **Alternative considered:** Speed-ship a draft, polish later.
- **Why rejected:** A discipline kit that wasn't built using its own discipline would have been a self-falsifying product. Dogfooding is the proof of concept. PRD-002 and PRD-001 (this document) are the visible evidence.

#### Decision 7: folder split — `artifacts/` for kit-internal, `docs/` for user-facing convention

- **Choice (v0.1.1 patch):** Kit's own planning outputs live in `artifacts/`. The kit's *recommendation to users* — that their project PRDs go in `docs/` — is preserved. Reader-facing kit content (case studies, FAQ, planned for v0.4.0) also lives in `docs/`.
- **Tradeoff:** Two locations for PRD-shaped files (artifacts for kit's own, docs for users' projects).
- **Alternative considered:** Single `docs/` directory mixing kit-internal and user-recommended content.
- **Why rejected:** `docs/` is a *teaching surface* — visitors copy that convention into their own projects. Mixing kit-internal planning artifacts into it muddies the example users emulate.

### Non-goals (intentional exclusions in v0.1.0)

- **No paid or premium tier.** MIT all the way.
- **No telemetry or usage tracking.** The kit doesn't phone home.
- **No API or programmatic interface.** Markdown files only.
- **No JavaScript runtime, build step, or package manager.**
- **No comprehensive language coverage.** Only TypeScript and Rust in v0.1.0; Python and Go deferred.
- **No PM-specific module.** Deferred to v0.3.0 in PRD-002.
- **No adoption infrastructure** (CONTRIBUTING.md, CHANGELOG.md, issue templates). Deferred to v0.2.0 in PRD-002.

---

## Phases (as delivered)

### Phase 1 — Conception and structural critique

**Delivered:** The structural design of the kit through iterative critique — what to include, what to exclude, what the file tree should look like, what the tone should be.

Key outputs of this phase: the three-pillar architecture (principles / skills / templates), the modular file structure, the decision to treat epistemic discipline as a marquee file, the prescriptive voice with **Why** / **How to apply** convention.

### Phase 2 — Repo bootstrap

**Delivered:**
- Public GitHub repo at `geigermatic/claude-codex`
- LICENSE (MIT), .gitignore, directory tree
- Initial README with positioning, structure overview, and quickstart
- GitHub topics for discoverability

### Phase 3 — Principles (the rules layer)

**Delivered nine principle files in `claude/`:**

- `EPISTEMICS.md` — verify-don't-guess, confidence-language, hypothesis-driven debugging
- `principles.md` — architecture, code quality, naming, anti-patterns
- `testing.md` — TDD, the pyramid, AAA, bug-fix protocol, concurrency check
- `security.md` — input validation, tenant isolation, secrets, supply chain, output encoding
- `token-economy.md` — efficient tool use, context discipline, cache awareness
- `git-workflow.md` — push safety, versioning, commit-message hygiene, CI-payload pitfalls
- `documentation.md` — PRDs, audit blocks, freshness rules, no-emoji standard
- `languages/typescript.md` — strict mode, discriminated unions, zod, Promise hygiene
- `languages/rust.md` — ownership, errors, async patterns, type-state, builder pattern

Each file is substantive (200-600 lines), self-contained, and tool-agnostic.

### Phase 4 — Interactive skills

**Delivered six skills in `skills/`:**

- `prd-author/` — interactive PRD authoring through the 7-section structure
- `tdd-slice/` — slice-based TDD planner with contract test → impl → e2e → flag-gated deploy
- `bug-fix-tdd/` — strict test-first bug fixing protocol
- `verify-claim/` — turn assertions into runnable evidence
- `pre-merge/` — 12-step preflight checklist before any push to shared branches
- `doc-audit/` — sweep for stale frontmatter, broken references, missing audit entries

Each skill has a `SKILL.md` with frontmatter (name, description), a detailed protocol, example conversations, and an explicit "what to refuse" section.

### Phase 5 — Templates

**Delivered five templates:**

- `CLAUDE.md.starter` — opinionated starter for a new project, with `{{placeholders}}`
- `PRD.md` — the 7-section template, frontmatter, audit log block
- `ERRORS.md` — starter error catalog with format and three example entries
- `doc-audit.md` — canonical frontmatter + audit-block schema
- `SLICE-WORKFLOW.md` — reference for slice-based TDD with worked example

### Phase 6 — Adapters

**Delivered:** [ADAPTERS.md](../ADAPTERS.md) — mapping table from Claude Code constructs (TodoWrite, subagents, slash commands, plan mode) to equivalents in Cursor, Aider, Cline, Continue, Codex CLI, and direct API users. Starter snippets for each.

### Phase 7 — Quality gates

**Delivered:**

- Proprietary-information audit (grep across all files; found and fixed one instance of a stakeholder name leaked from build context).
- Spelling consistency pass (v0.1.1 patch: unified "artefact" → "artifact" for US spelling).
- Folder split (v0.1.1 patch: `artifacts/` for kit-internal planning, `docs/` reserved for user-facing convention and future kit-reader content).
- Dogfooding artifacts: PRD-002 (forward planning) and PRD-001 (this retrospective).

---

## Files Touched

### Created in v0.1.0 (initial commit)

**Root:**
- `README.md`, `CLAUDE.md`, `ADAPTERS.md`, `LICENSE`, `.gitignore`

**Principles (`claude/`):**
- `EPISTEMICS.md`
- `principles.md`
- `testing.md`
- `security.md`
- `token-economy.md`
- `git-workflow.md`
- `documentation.md`
- `languages/typescript.md`
- `languages/rust.md`

**Skills (`skills/`):**
- `prd-author/SKILL.md`
- `tdd-slice/SKILL.md`
- `bug-fix-tdd/SKILL.md`
- `verify-claim/SKILL.md`
- `pre-merge/SKILL.md`
- `doc-audit/SKILL.md`

**Templates (`templates/`):**
- `CLAUDE.md.starter`
- `PRD.md`
- `ERRORS.md`
- `doc-audit.md`
- `SLICE-WORKFLOW.md`

**Total:** 25 files, ~6,600 lines.

### Added/modified in v0.1.1 patch

- Folder rename: `docs/` → `artifacts/` (for kit's own planning)
- New: `artifacts/PRD-002-adoption-improvements.md` (forward roadmap)
- New: `artifacts/PRD-001-initial-release.md` (this document)
- Modified: `claude/documentation.md` — added "4 frontmatter rules" pithy callout
- Modified: 5 files for `artefact` → `artifact` spelling consistency
- Modified: `claude/git-workflow.md` — replaced stakeholder name leak with generic team reference

**Cumulative as of v0.1.1:** 27 files, ~7,500 lines.

---

## Metrics

### Quantitative — what v0.1.0 set out to deliver

| Metric | Target | Delivered |
|----|----|----|
| Principle files with substantive content (>200 lines) | 9 | 9 ✓ |
| Skills with defined protocol + example | 6 | 6 ✓ |
| Templates usable as-is | 5 | 5 ✓ |
| Adapter mappings to non-Claude agents | ≥3 | 5 (Cursor, Aider, Cline, Continue, Codex CLI) ✓ |
| Languages with dedicated guides | 2 | 2 (TS, Rust) ✓ |
| Proprietary information leaks at launch | 0 | 0 (after audit + 1 fix) ✓ |
| License declared and visible | MIT | MIT ✓ |
| Publicly accessible | Yes | Yes ✓ |
| Repo topics configured | ≥5 | 10 ✓ |

### Qualitative — design proof points

- **Dogfooded against its own rules during construction.** This PRD and PRD-002 are the visible evidence — both follow the kit's PRD template, frontmatter schema, audit-log conventions.
- **Modular adoption verified.** Each principle file passes the "could a team adopt only this file?" check.
- **Tool portability documented.** ADAPTERS.md covers the five most common alternative agents.

### Anti-metrics — what we tracked to NOT happen

- Zero proprietary-source references at launch. Pre-launch grep found 1; fixed to a generic reference. Post-launch verification: clean.
- Zero broken cross-references between principle files. Spot-checked at launch; full verification deferred to a doc-audit pass.
- No tightly-coupled file structure. Each principle/skill/template can be deleted without breaking the rest.

### What was NOT measured at launch (and is therefore unknown)

- External adoption signals (stars, forks, PRs, mentions) — accumulate after launch; tracking responsibility shifts to PRD-002.
- Discoverability via GitHub search — depends on time and topic adoption; baseline TBD.
- Real-world impact on AI-coding output quality — requires longitudinal study or external case reports.

---

## Risks (encountered and resolved)

### R1: Proprietary information leakage — HIGH

**Risk:** Build context for the kit contained references to specific projects, services, vendors, and people. Without an audit, those references could leak into a public repo.

**Mitigation:**
- Pre-launch grep sweep across all 25 files for known proprietary terms (project names, service names, stakeholder names, vendor names, port numbers).
- Found one instance: a stakeholder first name used as an example in `claude/git-workflow.md`.
- Replaced with generic "the design team" reference; committed in v0.1.0 + 1.

**Status:** RESOLVED. Verified clean as of v0.1.1.

### R2: Scope creep across construction — HIGH

**Risk:** The temptation to add "one more file" or "one more language" during the build, delaying launch indefinitely and undermining the kit's own anti-scope-creep principle.

**Mitigation:**
- Hard cutoff at v0.1.0 contents (9 + 6 + 5 + adapters + license + readme).
- Deferred adoption infrastructure (CONTRIBUTING, CHANGELOG, issue templates) to v0.2.0 — formally captured in [PRD-002](PRD-002-adoption-improvements.md).
- Deferred Python, Go, and PM module to v0.3.0.

**Status:** RESOLVED. v0.1.0 shipped with disciplined scope; future expansion has an explicit phased plan.

### R3: Prescriptive voice triggers dogma-rejection in some engineers — MEDIUM

**Risk:** "MUST" / "NEVER" / "ALWAYS" framing may turn off engineers who reflexively distrust prescription, regardless of whether the rule is correct.

**Mitigation:**
- Each non-obvious rule includes **Why:** and **How to apply:** lines so the reasoning is visible.
- Persona paths and a "Why opinionated?" FAQ planned for v0.3.0 / v0.4.0 explicitly address dogma-resistance.
- ADAPTERS.md and modular structure mean a skeptic can adopt one file and reject the rest.

**Status:** ONGOING. Addressed in design; will be revisited as community feedback accumulates.

### R4: Spelling and convention drift across files — MEDIUM

**Risk:** The kit was built with mixed UK/US spelling ("artefact" vs "artifact") and could harbor other consistency drift.

**Mitigation:**
- Post-launch sweep found 12 instances of "artefact"; unified to US "artifact" in v0.1.1.
- Future drift addressed by `/doc-audit` skill and pre-merge step 5 (formatter / lint).

**Status:** RESOLVED for the spelling case; structural defense in place for future drift.

### R5: Single-persona focus — MEDIUM

**Risk:** v0.1.0 implicitly targets the individual-contributor engineer. Tech PMs and team leads, declared as a target audience, have no entry point speaking to their concerns.

**Mitigation:** Explicitly captured in [PRD-002](PRD-002-adoption-improvements.md) as Phase 1 (`START-HERE.md` with persona paths) and Phase 2 (`claude/pm-workflow.md`).

**Status:** DEFERRED to v0.2.0 / v0.3.0 with a clear plan.

### R6: Tonality assumes prior context — LOW

**Risk:** Some principle files reference patterns (e.g., "the synchronous-wait deadlock," "the SPA 200-OK trap") without telling the story. A reader without that context gets the rule but not the visceral reason.

**Mitigation:** Case studies and "from the trenches" stories planned for v0.4.0 in PRD-002 Phase 3.

**Status:** DEFERRED with a plan.

### Open questions surfaced during the build

- How aggressive should the kit be in cross-linking between principle files? (Current: light; PRD-002 may revisit.)
- Should there be a `bin/` or `scripts/` directory for installation helpers? (Current: no; `setup.sh` planned for v0.3.0 as a single root file.)
- How should the kit handle version pinning for users who want to depend on a specific release? (Current: git tags; documented in CLAUDE.md "Versioning of this kit" section.)

---

## Manual UI Test Checklist (as-launched verification)

The kit's "UI" is the GitHub repository surface — repo page, README rendering, file navigation, and the experience of cloning and using the kit. The following checks were performed at launch:

| # | Step | Expected result | Status |
|---|------|-----------------|--------|
| 1 | Visit github.com/geigermatic/claude-codex anonymous | Repo loads; description, topics, and README all render | PASS |
| 2 | Read README opening section | First-time visitor can answer "is this for me?" in < 1 minute | PASS |
| 3 | Verify license file present at repo root | `LICENSE` file with MIT text visible | PASS |
| 4 | Browse `claude/` directory | All 7 principle files + `languages/` subdirectory present | PASS |
| 5 | Open `claude/EPISTEMICS.md` | Substantive content (>200 lines) with verification rules | PASS |
| 6 | Browse `skills/` directory | All 6 skill folders present, each with a `SKILL.md` | PASS |
| 7 | Open `skills/prd-author/SKILL.md` | Frontmatter, protocol, example conversation, refuse list all present | PASS |
| 8 | Browse `templates/` directory | All 5 templates present | PASS |
| 9 | Open `templates/PRD.md` | Frontmatter schema, 7 sections, audit log block, Manual UI Test Checklist template | PASS |
| 10 | Open `ADAPTERS.md` | Mappings for Cursor, Aider, Cline, Continue, Codex CLI present | PASS |
| 11 | Verify no proprietary terms in any file | Grep across all files for the build-context project names, vendor names, and people names returns no matches | PASS (after audit + 1 fix) |
| 12 | Verify consistent spelling | `grep -r 'artefact' --include='*.md'` returns no matches | PASS (after v0.1.1 patch) |
| 13 | Clone repo locally; run `cp templates/CLAUDE.md.starter ./CLAUDE.md` | Starter file copies cleanly; placeholders visible for project-specific values | PASS |
| 14 | Open `artifacts/PRD-002-adoption-improvements.md` | Forward roadmap visible; frontmatter validates | PASS |
| 15 | Open `artifacts/PRD-001-initial-release.md` (this file) | Retrospective design doc visible; frontmatter validates | PASS |
| 16 | Verify GitHub topics configured | At least 5 relevant topics listed on repo page | PASS (10 topics) |
| 17 | Verify repo is public | Anonymous user can clone over HTTPS | PASS |

All 17 launch-verification checks pass as of v0.1.1.

---

## Audit log

| Date       | Reviewer     | Status    | Notes                                                                                    |
|------------|--------------|-----------|------------------------------------------------------------------------------------------|
| 2026-05-13 | @geigermatic | created   | Retroactive draft documenting v0.1.0 launch + v0.1.1 patch. Written using the kit's `templates/PRD.md`. |
