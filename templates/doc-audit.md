---
id: "DOC-TEMPLATE-AUDIT"
title: "Document Audit Template & Schema"
type: "reference"
domain: "documentation"
applies_to: ["all"]
status: "current"
confidence: "high"
last_verified: "{YYYY-MM-DD}"
---

# Document Audit Template & Schema

The single canonical reference for document structure, metadata, and quality standards. Every doc in `docs/` should conform to this template.

## Contents

1. [Required frontmatter](#required-frontmatter)
2. [Body structure](#body-structure)
3. [Audit log block](#audit-log-block)
4. [Status values](#status-values)
5. [Confidence values](#confidence-values)
6. [Doc types](#doc-types)
7. [What invalidates a doc](#what-invalidates-a-doc)
8. [Common mistakes](#common-mistakes)

---

## Required frontmatter

Every doc starts with YAML frontmatter:

```yaml
---
id: "DOC-042"                       # stable; never reuse
title: "Search Architecture"        # human-readable
type: "architecture"                # see Doc types below
domain: "search"                    # area of the codebase
applies_to: ["v2.x"]                # versions/components this describes
status: "current"                   # see Status values below
confidence: "high"                  # see Confidence values below
last_verified: "2026-05-13"         # ISO date; bumped on every audit
---
```

### Required vs optional

**Always required:**
- `id`
- `title`
- `type`
- `status`
- `confidence`
- `last_verified`

**Required for PRDs:**
- `implemented` (boolean)
- `target_version` (optional but recommended)

**Optional but useful:**
- `domain` (for filtering)
- `applies_to` (for version-specificity)
- `owner` (point of contact)

### Never delete frontmatter

Even partially. Audit tooling depends on it.

---

## Body structure

### TOC required for 4+ sections

Any doc with four or more sections needs a linked Table of Contents near the top, after the introductory paragraph.

GitHub-style anchor links:

```markdown
## Contents

1. [Problem](#problem)
2. [Current State](#current-state)
3. [Design](#design)
```

### PRD section order (required)

PRDs follow a standard order:

1. Problem
2. Current State
3. Design
4. Phases
5. Files Touched
6. Metrics
7. Risks
8. Manual UI Test Checklist (REQUIRED for user-facing PRDs)
9. Audit log

See `templates/PRD.md` for the full template.

### Manual UI Test Checklist (PRDs)

Required for any PRD whose work is user-visible. Numbered table:

| # | Step | Expected result |
|---|------|-----------------|
| 1 | Concrete action | Observable outcome |

Each row: a step the user (or QA) can perform and a result they can verify. "Looks correct" is not an expected result.

### No emojis

Engineering docs use text labels:
- DONE / TODO / WIP (not checkmarks)
- WARNING / DANGER / TIP (not warning symbols)
- LOW / MED / HIGH (not colored circles)

Why: search/grep reliability, screen readers, cross-platform rendering. Marketing copy and UI text can use emojis; engineering docs cannot.

---

## Audit log block

Every non-trivial doc ends with an audit log:

```markdown
---

## Audit log

| Date       | Reviewer | Status   | Notes                                 |
|------------|----------|----------|---------------------------------------|
| 2026-05-13 | @alice   | verified | All code references current.          |
| 2026-03-20 | @bob     | partial  | Updated §3; metrics in §6 still drift |
| 2026-01-15 | @alice   | created  | Initial draft.                        |
```

### When to add an entry

- **Creating the doc:** entry with status `created`.
- **Editing the body:** entry with status `verified` (or `partial` if not everything was checked).
- **Auditing without editing:** entry with status `verified` (or `drift` if found stale sections). Bump `last_verified` in frontmatter.
- **Marking superseded:** entry with status `superseded`, plus a link to the replacement.

Always update `last_verified` in frontmatter to match the audit log's most recent verifying entry.

---

## Status values

| Status | Meaning |
|--------|---------|
| `draft` | Work-in-progress; not yet ready for use |
| `current` | Reflects current system state; safe to act on |
| `partial` | Some sections verified, others may be stale (see notes) |
| `drift` | Known to be stale; sections marked in body |
| `superseded` | Replaced by another doc (link to it) |
| `deprecated` | Describes a removed/obsolete part of the system; kept for history |

---

## Confidence values

| Confidence | Meaning |
|-----------|---------|
| `high` | Recently verified; trusted; act on the claims |
| `medium` | Verified at some point but not recently; spot-check before acting |
| `low` | Untested claims; use as a hypothesis, not fact |

Confidence is independent of status. A `current` doc can be `low` confidence if it's never been audited. A `deprecated` doc can be `high` confidence if its historical accuracy is well-established.

---

## Doc types

| Type | Purpose |
|------|---------|
| `PRD` | Problem + design + implementation plan; precedes work |
| `architecture` | How a system or subsystem is structured (steady state) |
| `runbook` | Step-by-step operational procedure |
| `reference` | Lookup material (error catalog, glossary, API reference) |
| `guide` | How-to for a specific task |
| `decision` | Architecture Decision Record (ADR) — why we chose X over Y |
| `postmortem` | Incident analysis |
| `note` | Informal scratch / meeting notes (often no frontmatter) |

---

## What invalidates a doc

A doc is invalidated when:

- Code it references no longer exists (renamed, deleted, moved)
- Behavior it describes no longer matches reality
- A decision it documents has been reversed
- A constraint it cites is no longer in effect

When you discover invalidation while doing other work:

1. **Minimum:** mark `status: drift` in frontmatter; add an audit log entry noting the drift.
2. **Better:** fix the doc in the same PR as the code change.
3. **Best:** include "doc updates" as part of the work checklist for every non-trivial PR.

---

## Common mistakes

### Out-of-sync frontmatter

`last_verified: 2026-05-13` but the body was edited yesterday with no audit log entry. Either the editor verified (then add the entry) or didn't (then don't update `last_verified`).

### Missing audit log

Doc has frontmatter but no audit log block. Add one — even a single `created` entry is better than none.

### Frontmatter on informal notes

Scratch notes and meeting minutes don't need frontmatter. If a doc is informal, it can live in `docs/notes/` (or wherever) without metadata. If it grows up to be a reference, then add frontmatter.

### Emojis sneaking in

Especially in status labels. Use `DONE`, not the checkmark emoji. Catch via grep on doc PRs.

### "Last verified" set to a future date

Probably a typo. Catch via doc-audit tooling.

### PRDs with no Manual UI Test Checklist

For any user-visible PRD, this is required. Refuse to merge a PRD without it.

### Comprehensive but wrong

Long, polished docs that are subtly inaccurate are worse than short, accurate docs. Prefer terse + correct.

---

## Audit log

| Date       | Reviewer  | Status    | Notes                                          |
|------------|-----------|-----------|------------------------------------------------|
| {date}     | {@user}   | created   | Initial template.                              |
