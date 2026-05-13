---
name: doc-audit
description: Sweep docs/ for stale frontmatter, drift between docs and code, and missing audit entries. Use to find documentation rot before it misleads someone.
---

# /doc-audit — documentation freshness sweep

Docs rot when nobody verifies them. This skill scans your `docs/` directory (or wherever your documentation lives) and flags every doc that's likely stale, drift-prone, or structurally incomplete.

This is not a comprehensive audit (that would take hours). It's a fast triage: find the worst offenders, surface them to the user, recommend actions.

## When to use this skill

Invoke when:

- The user is about to ship a feature and wants to verify docs aren't lying
- A new team member is being onboarded and needs trustworthy docs
- It's been a while since the docs were checked (quarterly maintenance)
- The user is investigating a doc-vs-reality discrepancy

If the user wants a deep audit of a specific document, use `/verify-claim` instead — that's targeted, this is broad.

## The protocol

### Step 1 — Identify the doc location

Ask if unclear:

> Where do docs live? `docs/`? Multiple places? Should I include READMEs in subdirectories?

Default: scan `docs/` recursively. Include `README.md` and `CONTRIBUTING.md` at the repo root if present.

### Step 2 — Enumerate the docs

List all markdown files in the doc location:

```bash
find docs/ -name '*.md' -type f
```

Report total count.

### Step 3 — Check frontmatter coverage

For each doc, check if it has YAML frontmatter at the top. The required fields are:

- `id`
- `title`
- `type`
- `status`
- `confidence`
- `last_verified`

PRDs additionally need:
- `implemented`
- `target_version` (optional)

Report:

```
Frontmatter coverage:
  Total docs: 47
  With complete frontmatter: 32
  With partial frontmatter: 9
  Without frontmatter: 6

Docs missing frontmatter:
  - docs/notes/random-thoughts.md
  - docs/old-readme.md
  - ...
```

### Step 4 — Check `last_verified` freshness

For each doc with complete frontmatter, compute days since `last_verified`.

Buckets:
- **Fresh** (< 30 days): probably reliable
- **Aging** (30-90 days): probably reliable, but worth a glance if touched
- **Stale** (90-180 days): treat claims as hypotheses
- **Rotten** (180+ days): probably wrong; needs verification or deprecation

Report counts in each bucket. List the rotten docs by name.

### Step 5 — Check status drift

For each doc:

- `status: draft` + `last_verified > 90 days ago` → probably abandoned; flag for cleanup
- `status: current` + `last_verified > 180 days ago` → claims currency without verification; flag
- `status: superseded` without a "see X" link → broken reference; flag

### Step 6 — Spot-check code references

For each doc that references specific files or functions (pattern: `` `src/...` `` or `file.ts:42`), pick a sample of 3-5 references per doc and verify they still exist:

- File path: does the file exist?
- Function name: does grep find it?
- Line number: is it approximately right? (Line numbers drift; don't require exact match.)

For docs with broken references, flag them. Note the specific broken references.

Skip this step if the user asks for a fast scan — it's the most expensive part.

### Step 7 — Check audit log presence

For each doc that's structurally non-trivial (PRD, architecture, runbook):

- Does it have an audit log block at the bottom?
- Does the audit log have at least one entry?
- Does the most recent entry match `last_verified` in frontmatter?

Flag docs that:
- Have no audit log block (and should)
- Have an audit log that hasn't been updated since `last_verified`

### Step 8 — Check for "implemented" drift on PRDs

For each PRD (`type: PRD`):

- Is `implemented` set?
- If `implemented: false`, has the work actually shipped? (Hard to verify automatically; flag for user check.)
- If `implemented: true`, does the body still describe future tense ("we will build...")? That's drift.

### Step 9 — Generate the report

Output a structured report:

```markdown
# Doc Audit — {date}

## Summary

- Total docs scanned: 47
- Fresh (< 30d): 12
- Aging (30-90d): 18
- Stale (90-180d): 11
- Rotten (180d+): 6

## Top priorities

1. **Rotten doc with unclear status**: `docs/PRD-search-v2.md`
   - last_verified: 240 days ago
   - status: current (questionable)
   - Recommendation: verify or mark superseded

2. **Broken code references**: `docs/ARCH-pipeline.md`
   - References `src/old-pipeline/main.ts` (file deleted in commit abc123)
   - Recommendation: update references or mark drift

3. **Missing frontmatter**: 6 docs
   - List below; recommend adding metadata or deciding they're informal scratch

## Detailed findings

{full table of each issue}

## Recommended actions

1. Pick the top 3 rotten docs. Either verify (and update last_verified) or mark deprecated.
2. Update the 6 docs without frontmatter — at minimum, add the required fields.
3. Fix broken code references in `ARCH-pipeline.md`.
```

### Step 10 — Offer to fix

Ask:

> Want me to draft the fixes for any of these? I can:
> - Update frontmatter on the docs missing it
> - Add audit log entries for docs you've verified
> - Mark superseded docs explicitly
>
> Pick what you want to address; I'll work through them.

## Example conversation

> **User**: `/doc-audit`
>
> **Assistant**: Scanning `docs/`.
>
> *(Bash: find docs/ -name '*.md' | wc -l)*
>
> Found 34 markdown files in `docs/`. Beginning audit.
>
> *(Reads each file's frontmatter or first 30 lines if no frontmatter detected)*
>
> ...
>
> # Doc Audit — 2026-05-13
>
> ## Summary
>
> - Total docs scanned: 34
> - Fresh: 8
> - Aging: 12
> - Stale: 9
> - Rotten: 5
>
> ## Top priorities
>
> 1. **Rotten**: `docs/PRD-legacy-auth.md` — last verified 11 months ago; status still "current"; references `src/auth/legacy.ts` which was deleted 6 months ago. Recommend: mark `status: superseded` with a link to `PRD-native-auth.md`.
>
> 2. **Broken refs**: `docs/runbook-deploy.md` — references the old CI workflow file path; that file was renamed. Step 4 of the runbook would fail today.
>
> 3. **Missing frontmatter**: 4 docs — `docs/scratch/`, `docs/old-meeting-notes.md`. These look like informal notes; recommend moving them to a `notes/` directory or deleting if obsolete.
>
> ## Recommendations
>
> 1. Tag `PRD-legacy-auth.md` as superseded.
> 2. Update `runbook-deploy.md` steps 4-6 with the new CI path.
> 3. Decide on the scratch docs.
>
> Want me to draft any of these fixes?
>
> **User**: Yes, draft the supersede note for PRD-legacy-auth.md.
>
> *(Edits the file: updates frontmatter status, adds audit log entry, adds a "See PRD-native-auth.md" pointer at the top.)*
>
> Done. Diff is small — `status: current` → `status: superseded`, added a 2-line "Superseded by..." note, and added an audit log entry dated today. Review?

## What to refuse

- **Fixing without showing the user.** Always show the proposed change before applying.
- **Comprehensive deep audits.** This skill is triage. For one-doc deep verification, use `/verify-claim`.
- **Blindly trusting frontmatter.** A doc that says `last_verified: 2026-05-13` may have been edited without anyone verifying. Note that frontmatter is a claim, not a proof.

## When to recommend deprecation over update

A doc is a candidate for deprecation (not update) when:

- It's been rotten for 12+ months with no engagement
- The thing it documents has been replaced
- Maintaining it accurately would take more effort than rewriting

Deprecation note format:

```markdown
> **DEPRECATED:** This document describes the legacy X system. It has been replaced by [Y](Y.md). The contents below are kept for historical context and should not be acted on.
```

Set `status: deprecated` in frontmatter. Add an audit log entry.

## What "done" means for this skill

- The full audit report has been generated.
- The top priorities are clearly identified.
- The user has decided which to address.
- Any fixes have been applied with user approval.
- Frontmatter and audit logs reflect the work done.
