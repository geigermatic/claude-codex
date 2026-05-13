---
id: "PRD-001"
title: "claude-codex adoption improvements"
type: "PRD"
domain: "product-strategy"
applies_to: ["v0.x"]
status: "draft"
confidence: "high"
last_verified: "2026-05-13"
implemented: false
target_version: "v0.2.0"
---

# claude-codex adoption improvements

> Lower the friction between "this kit exists" and "this kit is in my project." Ship adoption-critical artefacts (entry-point guide, visible skill output, GitHub polish, language breadth) so first-time visitors can evaluate and commit within 5 minutes.

This PRD is itself a dogfooding artefact: the first real document written using the kit's own `templates/PRD.md`. The audit log records its own creation via `/prd-author`.

## Contents

1. [Problem](#problem)
2. [Current State](#current-state)
3. [Design](#design)
4. [Phases](#phases)
5. [Files Touched](#files-touched)
6. [Metrics](#metrics)
7. [Risks](#risks)
8. [Manual UI Test Checklist](#manual-ui-test-checklist)
9. [Audit log](#audit-log)

---

## Problem

The kit launched at v0.1.0 with 25 files and ~6,600 lines of content. The content quality is high, but the **adoption path is unclear**. A first-time visitor faces three compounding friction points:

1. **No entry point for navigation.** The README maps the file tree but doesn't tell a visitor which file to read first. A reader skimming 25 markdown files has no obvious starting line.
2. **No visible proof.** Skills exist as protocols on the page, but no first-time visitor has seen output. The "before/after" framing in the README is asserted, not demonstrated. A skeptical engineer reading prescriptive rules without seeing the rules-in-action will discount them as opinion.
3. **No social or maintenance signals.** Zero stars, no `CONTRIBUTING.md`, no `CHANGELOG.md`, no issue templates. Adoption-critical visitors — especially tech leads and PMs evaluating whether to standardize their team on this — read these signals to gauge whether a project is worth the commitment.

Adjacent gaps compound the problem:

- **Single-persona framing.** The README and skills implicitly target the individual contributor engineer. Tech PMs and team leads, named as a target audience, have no entry point speaking to their concerns (rollout, team adoption, measuring discipline).
- **Language coverage gaps.** Only TypeScript and Rust covered. Python and Go each have substantial developer populations that find no language-specific guidance and may bounce.
- **Invocation friction.** The skills are usable but the mechanism to invoke them (`/skill-name` in Claude Code) is not explained for newcomers who haven't used skills before.

**Why now:** the kit is at its highest leverage moment — fresh launch, no installed base, no community expectations to manage. Improvements made now compound (better first impressions → more adoption → more contributions → better kit). Deferring lets first impressions calcify; the cohort that finds the kit in the first 90 days disproportionately shapes its reputation.

**Evidence:** internal critical review (2026-05-13) identified 8 distinct friction points; the persona walkthrough as a skeptical first-time visitor reproduced the navigation, proof, and signal gaps consistently.

---

## Current State

**Repository:** `github.com/geigermatic/claude-codex`
**Version:** v0.1.0 (initial commit + a one-line cleanup)
**License:** MIT
**Visibility:** Public; 10 GitHub topics configured for SEO

### Structure (verified 2026-05-13)

```
claude-codex/
├── CLAUDE.md
├── README.md
├── ADAPTERS.md
├── LICENSE
├── claude/
│   ├── EPISTEMICS.md, principles.md, testing.md, security.md,
│   ├── token-economy.md, git-workflow.md, documentation.md
│   └── languages/
│       └── typescript.md, rust.md
├── skills/
│   └── prd-author/, tdd-slice/, bug-fix-tdd/, verify-claim/, pre-merge/, doc-audit/
└── templates/
    └── CLAUDE.md.starter, PRD.md, ERRORS.md, doc-audit.md, SLICE-WORKFLOW.md
```

### What works

- README opens with the failure-modes framing target-audience engineers recognize from experience.
- One concrete before/after example (dark-mode toggle) demonstrates the value delta.
- Repo description, topics, and license signal credibility to a scanning visitor.
- Tool-agnostic principles + Claude-specific skills cleanly separated, supporting the "principles port, skills require porting" model in `ADAPTERS.md`.
- `ADAPTERS.md` addresses non-Claude-Code users — important for the kit's broader-audience aspiration.

### What's missing

- No `START-HERE.md`, `ESSENTIALS.md`, `CONTRIBUTING.md`, `CHANGELOG.md`
- No `.github/` directory (no issue templates, no PR template)
- No `docs/` directory (this PRD is the first occupant)
- No visible skill output (no transcript excerpts, no recordings)
- No Python or Go language guides
- No PM-specific workflow guide
- No `examples/` directory with real artefacts (PRDs, slice plans, audit logs)
- No badges in README signaling license, version, stars, build status

---

## Design

The adoption-improvement work breaks into **four tiers ordered by leverage**. Each tier ships as a discrete release so the kit gains visible version momentum and we can stop at any tier with a stable, useful artefact.

### Approach: stack the credibility before the depth

First-time visitors evaluate in seconds. **Tier 1 optimizes those seconds** — visible proof, clear entry point, maintenance signals. **Tier 2 broadens reach** through new audiences and lower setup friction. **Tier 3 adds depth and credibility** for skeptics. **Tier 4 distributes** through adapters and community surfaces.

This ordering is deliberate: a visitor who bounces at Tier 1 never benefits from Tier 2-4. Investing in depth before investing in adoption is the classic engineering trap.

### Key decisions

#### Decision 1: separate entry-point artefacts from README

- **Choice:** Add a `START-HERE.md` with persona paths rather than expanding the README.
- **Tradeoff:** Two files for the visitor to navigate. Mitigated by README explicitly linking *"New here? Start with [START-HERE.md](START-HERE.md)."*
- **Alternative considered:** put persona paths in the README itself.
- **Why rejected:** the README's job is positioning (*what is this, is it for me?*). The persona paths' job is navigation (*now that I'm interested, where do I go?*). Different jobs warrant different files. A README that tries to do both becomes long and dilutes both jobs.

#### Decision 2: ship a single-file lite variant

- **Choice:** Ship `ESSENTIALS.md` — ~200 lines containing the top 20 rules. Drop into any agent in 30 seconds.
- **Tradeoff:** Risk that visitors adopt only the lite variant and miss the depth. Mitigated by ending `ESSENTIALS.md` with *"This is the floor. The full kit at `claude/` and `skills/` builds the ceiling."*
- **Alternative considered:** require full-kit adoption.
- **Why rejected:** all-or-nothing is the highest-friction path. Letting visitors try the lite version converts more people to the full version than gating does. The conversion path is real; the gating cost is real.

#### Decision 3: maintain markdown-first; no app shell (yet)

- **Choice:** No GitHub Pages site, no rendered docs site, no React app in Tier 1.
- **Tradeoff:** Visitors browse raw markdown on github.com, which is less polished than a dedicated docs site.
- **Alternative considered:** ship GitHub Pages in Tier 1.
- **Why rejected:** maintenance burden disproportionate to launch benefit. GitHub renders markdown well enough for adoption. Defer to Tier 4 if adoption signals justify.

#### Decision 4: language guides as additive files, not changes to existing structure

- **Choice:** `claude/languages/python.md` and `claude/languages/go.md` are new files; existing TS/Rust files are not restructured.
- **Tradeoff:** Some duplication across language guides (e.g., naming conventions adapted per language).
- **Alternative considered:** extract a `claude/languages/_shared.md` for cross-language patterns.
- **Why rejected:** premature abstraction. Only 4 languages even after this work. The rule-of-three says wait for the 5th language before extracting shared structure. Per `claude/principles.md` §4.1.

#### Decision 5: PRDs and adoption artefacts live in `docs/`, not at root

- **Choice:** This PRD lives at `docs/PRD-adoption-improvements.md`. Future PRDs, case studies, and FAQs go in `docs/`.
- **Tradeoff:** One more directory layer.
- **Alternative considered:** PRDs at root.
- **Why rejected:** mirrors the convention the kit recommends to consumers (PRDs in `docs/`). Dogfooding our own template. Visitors who explore `docs/` see real artefacts using the kit's templates.

### Contract changes

No breaking changes to existing principle files or skills. New files added; existing files preserved.

**Version bump schedule:**
- `v0.2.0` after Phase 1 ships (introduces `START-HERE.md`, `docs/`, `.github/`)
- `v0.3.0` after Phase 2 ships (adds Python, Go, PM module, `ESSENTIALS.md`)
- `v0.4.0` after Phase 3 ships (case studies, FAQ, examples)
- `v0.5.0` or `v1.0.0` after Phase 4, depending on adoption signals

### Non-goals

- No rewrite of existing principle files (they're stable; resist scope creep)
- No paid or premium tier
- No telemetry or usage tracking
- No API or programmatic interface beyond markdown
- No dependency on external services
- No JavaScript framework, no build step, no package manager

---

## Phases

### Phase 1 (MVP) — Adoption critical path

**Outcome:** A first-time visitor can evaluate the kit in 5 minutes and commit to trying it. Maintenance signals visible.

**Scope:**
- `START-HERE.md` with 5 persona paths (Solo IC, Team Lead, Tech PM, Existing Project, Fresh Project)
- README revisions: 5 before/after vignettes; "5-minute test drive" section; badges
- Visible skill output: expand each `skills/*/SKILL.md` example-conversation section to include realistic transcript excerpts (2x current depth)
- `CONTRIBUTING.md` (how to submit a fix or improvement, what makes a strong PR)
- `CHANGELOG.md` with v0.1.0 and v0.2.0 entries
- `.github/ISSUE_TEMPLATE/bug.md`
- `.github/ISSUE_TEMPLATE/feature.md`
- `.github/ISSUE_TEMPLATE/improvement.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- This PRD (`docs/PRD-adoption-improvements.md`) committed alongside

**Effort:** ~1 day (AI-assisted)

**Ships as:** v0.2.0

### Phase 2 — Audience breadth

**Outcome:** Python and Go developers have language-specific guidance. PMs and tech leads have a workflow guide that speaks to them directly. Setup is one command.

**Scope:**
- `claude/languages/python.md` (~400 lines: strict mypy, runtime validation with pydantic, async patterns, packaging discipline)
- `claude/languages/go.md` (~400 lines: error handling idioms, interfaces, concurrency, dependency management)
- `claude/pm-workflow.md` (~300 lines): team coordination, onboarding engineers to the kit, measuring adoption, handling "I disagree with rule X" pushback, escalation patterns
- `ESSENTIALS.md` — single-file lite variant (top 20 rules, ~200 lines)
- `setup.sh` install script — drops `CLAUDE.md`, wires up skill paths, prompts for project-specific values
- Skill worked-example expansion: 2 additional fully-played-out conversations per skill (12 new transcripts total)

**Effort:** ~2 days

**Ships as:** v0.3.0

### Phase 3 — Depth and credibility

**Outcome:** Skeptics find evidence. Abstract principles have concrete worked examples.

**Scope:**
- `docs/case-studies.md` — 5-7 generalized incident stories ("The synchronous-wait deadlock," "The SPA 200-OK trap," "The migration rename that broke prod"). Each ends with the rule that would have prevented it.
- `docs/FAQ.md` — "Why opinionated?", "How do I disagree with a rule?", "What if my team won't adopt this?", "How does this compare to [other kit]?"
- `examples/PRD-example.md` — a realistic worked PRD using the template
- `examples/SLICE-PLAN-example.md` — realistic slice plan
- `examples/ERRORS-example.md` — realistic error catalog with 8-10 entries
- `examples/audit-log-example.md` — realistic doc with full audit log history
- Demo screen recording (asciinema or short video) of `/tdd-slice` running on a real change
- Stable semver tag releases for v0.1, v0.2, v0.3

**Effort:** ~1-2 days

**Ships as:** v0.4.0

### Phase 4 — Distribution

**Outcome:** Easy to use with non-Claude agents. Visible in community lists.

**Scope:**
- `adapters/cursor/.cursorrules` — pre-built ready-to-use Cursor configuration
- `adapters/aider/.aider.conf.yml` — same for Aider
- `adapters/continue/config.json.partial` — Continue snippet
- GitHub Pages landing at `geigermatic.github.io/claude-codex`
- Submissions to `awesome-claude-code`, `awesome-prompts`, `awesome-ai-coding` lists

**Effort:** ~0.5 day

**Ships as:** v0.5.0 (or v1.0.0 if adoption signals strong)

---

## Files Touched

### Phase 1

**New files:**
- `START-HERE.md`
- `CONTRIBUTING.md`
- `CHANGELOG.md`
- `.github/ISSUE_TEMPLATE/bug.md`
- `.github/ISSUE_TEMPLATE/feature.md`
- `.github/ISSUE_TEMPLATE/improvement.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `docs/PRD-adoption-improvements.md` (this file)

**Modified files:**
- `README.md` — vignettes, 5-min test drive, badges, START-HERE link
- `skills/prd-author/SKILL.md` — expanded example conversation
- `skills/tdd-slice/SKILL.md` — expanded example conversation
- `skills/bug-fix-tdd/SKILL.md` — expanded example conversation
- `skills/verify-claim/SKILL.md` — expanded example conversation
- `skills/pre-merge/SKILL.md` — expanded example conversation
- `skills/doc-audit/SKILL.md` — expanded example conversation

### Phase 2

**New files:**
- `claude/languages/python.md`
- `claude/languages/go.md`
- `claude/pm-workflow.md`
- `ESSENTIALS.md`
- `setup.sh`

**Modified files:**
- `CLAUDE.md` — add optional `@claude/pm-workflow.md` import
- `README.md` — note expanded language coverage
- Each `skills/*/SKILL.md` — additional example conversations

### Phase 3

**New files:**
- `docs/case-studies.md`
- `docs/FAQ.md`
- `examples/PRD-example.md`
- `examples/SLICE-PLAN-example.md`
- `examples/ERRORS-example.md`
- `examples/audit-log-example.md`

### Phase 4

**New files:**
- `adapters/cursor/.cursorrules`
- `adapters/aider/.aider.conf.yml`
- `adapters/continue/config.json.partial`
- `.github/workflows/pages.yml` (for GitHub Pages)

---

## Metrics

### Quantitative

| Metric | Target | Window | Measurement |
|----|----|----|----|
| GitHub stars | 100 | 90 days post-Tier-1 | GitHub repo insights |
| Forks | 25 | 90 days post-Tier-1 | GitHub repo insights |
| External PRs (non-author) | 5 | 90 days post-Tier-1 | PR list filtered by author |
| External issues filed | 10 | 90 days post-Tier-1 | Issues list |
| Fork-to-star ratio | ≥ 0.15 | 90 days post-Tier-1 | Computed |

### Qualitative

- **Discoverable**: appears on first page of GitHub search for "claude code best practices" within 60 days
- **Mentioned positively**: at least 3 community discussions (Reddit, Discord, X/Twitter) referencing the kit within 90 days
- **No critical adoption-blocker issues**: zero issues in the first 30 days of the form "I followed your instructions and nothing worked"

### Anti-metrics

- **Issues complaining about dogma > issues complaining about gaps**: signals tone is too prescriptive. Tier 3 FAQ addresses this.
- **Fork-to-star ratio < 0.10**: lots of curiosity, no adoption. Signals discoverability without conversion — content is interesting but not actionable.
- **Stars without engagement (no issues, no PRs)**: kit is bookmarked, not used. Sign that adoption infrastructure (Tier 1) is still insufficient.

---

## Risks

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | Scope creep across tiers — quality drops as we rush content | HIGH | Tier 1 is bounded; ship it as v0.2.0 before starting Tier 2. Treat each tier as its own discrete release with its own version bump. |
| 2 | Tonality friction — "MUST" / "NEVER" reads as dogma to some engineers | MED | Phase 3 FAQ addresses head-on with "Why opinionated?" and "How to disagree." Persona paths in Phase 1 reduce dogma-feel by speaking to each persona's needs. |
| 3 | Maintenance burden once issues accumulate | MED | `CONTRIBUTING.md` sets explicit expectations. Issue templates auto-triage. Keep repo surface area small. |
| 4 | Language guides drift from each other (different recommendations for the same principle across languages) | MED | Each language guide cross-references the core principle file. Reviewers check consistency on PRs that touch multiple guides. |
| 5 | Case studies (Tier 3) inadvertently leak identifiable details from anonymized incidents | LOW | Generalize aggressively — describe a *class* of incident, not a specific company/project. Use synthetic details where needed. |
| 6 | Setup script (Tier 2) fails on unusual environments | LOW | Script is dumb and idempotent (copies files only). Failure is recoverable; no destructive operations. Test on macOS + common Linux. |
| 7 | Adoption stalls despite Tier 1 — content quality is the bottleneck, not packaging | MED | Track anti-metrics; if fork-to-star is healthy but engagement is low, the kit's substance is the issue, not adoption infra. Pivot to content depth (skip Tier 2 distribution work). |

### Open questions

- Should `ESSENTIALS.md` live at repo root or under a `lite/` directory? (Lean: root, for discoverability.)
- Do we want a git-tag pinning mechanism so users can pin to v0.2.0 instead of tracking main? (Lean: yes, tag releases starting at Tier 1 ship.)
- How aggressive should PM-specific framing be — separate `pm/` directory or single file under `claude/`? (Lean: single file. Don't fragment until there are 3+ PM-specific docs.)
- Should we include analytics on GitHub Pages (Tier 4) to measure scroll/click? (Lean: no, to preserve the privacy-respecting feel of the kit.)

---

## Manual UI Test Checklist

The "UI" of this product is the GitHub repository page, the README rendered in browsers, and the navigation paths from there. This checklist exercises the visitor experience end-to-end.

| # | Step | Expected result |
|---|------|-----------------|
| 1 | Visit `github.com/geigermatic/claude-codex` as anonymous user | Description, topics, README load within 3s |
| 2 | Read README opening paragraph | Can answer "is this for me?" within 30 seconds |
| 3 | Scroll to badges block | License, version, stars badges all render correctly |
| 4 | Click "Start here" link in README | `START-HERE.md` loads; persona paths visible above the fold |
| 5 | Pick "Solo IC" persona path | Reach a working state in <5 minutes following the instructions |
| 6 | Pick "Tech PM" persona path | Path speaks to PM concerns (team rollout, measurement), not IC concerns |
| 7 | Try the README "5-minute test drive" section | Skill produces visible output as advertised |
| 8 | Read 2 before/after vignettes in README | Understand the value delta without other context |
| 9 | Click `CONTRIBUTING.md` | Know exactly what makes a strong PR; have a clear submission path |
| 10 | Open `New issue` page | Templates appear for bug / feature / improvement |
| 11 | Browse `skills/prd-author/SKILL.md` | At least 2 example conversation excerpts visible |
| 12 | Open `CHANGELOG.md` | v0.1.0 and v0.2.0 entries with dates and contents |
| 13 | (Phase 2) Visit `ESSENTIALS.md` | Single-file, readable in <5 minutes, top 20 rules clear |
| 14 | (Phase 2) Run `setup.sh` in a fresh project directory | `CLAUDE.md` and skill paths wired up; no destructive operations |
| 15 | (Phase 2) Find `claude/languages/python.md` | Python-specific rules present and consistent with TS/Rust counterparts |
| 16 | (Phase 3) Read one case study | Concrete incident → rule mapping is clear and credible |
| 17 | (Phase 3) Browse `examples/PRD-example.md` | Realistic, complete PRD that follows the template exactly |
| 18 | (Phase 4) Apply `adapters/cursor/.cursorrules` to a Cursor project | Cursor loads the rules; behavior visibly more disciplined |

Phase 1 is complete when items 1-12 pass. Each subsequent phase appends checks 13+.

---

## Audit log

| Date       | Reviewer     | Status   | Notes                                                           |
|------------|--------------|----------|-----------------------------------------------------------------|
| 2026-05-13 | @geigermatic | created  | Initial draft. Authored via the kit's `/prd-author` skill as a dogfooding artefact. |
