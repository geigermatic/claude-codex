---
id: "DOC-ERRORS"
title: "Error Catalog"
type: "reference"
domain: "operations"
applies_to: ["all"]
status: "current"
confidence: "high"
last_verified: "{YYYY-MM-DD}"
---

# Error Catalog

Operational reference mapping log messages and error codes to root causes and remediations.

## Contents

1. [How to use this catalog](#how-to-use-this-catalog)
2. [How to add an entry](#how-to-add-an-entry)
3. [Errors](#errors)
4. [Audit log](#audit-log)

---

## How to use this catalog

When you see an error in logs, monitoring, or a customer report:

1. Search this file for the error code (e.g., `ERR-042`) or a key phrase from the log message.
2. If found: follow the documented fix. Note your encounter in the audit log if it's novel.
3. If not found: troubleshoot from scratch, then **add an entry** so the next person doesn't start over.

This catalog is append-only. Entries are never deleted — only marked superseded.

---

## How to add an entry

When you fix an error that was hard to diagnose, add an entry. The format:

```markdown
### ERR-{NNN} — {Short symptom description}

**Symptom:** {What the operator/customer sees. Include the exact log line if applicable.}

**Cause:** {The root cause. Not the proximate cause — the underlying reason.}

**Fix:** {What to do. Step by step if non-trivial.}

**Prevention:** {How we'd catch this earlier next time. Link to a related principle if applicable.}

**Verified:** {YYYY-MM-DD} by {@user} in {incident or PR reference}
```

Append entries at the bottom. Sort by code, not by date. Code numbers are assigned sequentially — pick the next available.

If the same error code shows up in multiple places, give them distinct sub-codes: `ERR-042a`, `ERR-042b`.

---

## Errors

<!-- Example entries follow. Replace with your project's real errors. -->

### ERR-001 — Failed to acquire database connection within timeout

**Symptom:** Log line `pool.acquire timeout` after {N} seconds; HTTP requests return 503 to users.

**Cause:** Connection pool exhausted under concurrent load. Pool size was {old value}, concurrent request count peaked at {higher}.

**Fix:**
1. Increase pool size in `config/database.ts` (or wherever applicable).
2. Audit any sync-wait patterns (Job A waits on Job B on the same pool) — see `claude-codex/claude/testing.md` §11.
3. Add a metric alerting on pool wait time exceeding {threshold}.

**Prevention:** Apply the synchronous-wait concurrency check at PR review time (see `claude-codex/claude/testing.md` §11).

**Verified:** {YYYY-MM-DD} by {@user} in incident #234.

---

### ERR-002 — Migration applied but code references old column name

**Symptom:** Runtime error `column "X" does not exist`; affects all writes to the {table} table.

**Cause:** Migration renamed `old_col` to `new_col` and was applied before the code change rolling out. Deployment ordering violated the forward-compat rule.

**Fix:**
1. Either roll back the migration (if no data has been written to the new column).
2. Or hotfix the code to use the new column name and redeploy.

**Prevention:** Schema renames must follow the **expand-and-contract** pattern:
1. Migration A: add the new column, dual-write from code.
2. Backfill data from old → new.
3. Migration B: drop the old column **after** code stops referencing it.

Never rename in a single migration.

**Verified:** {YYYY-MM-DD} by {@user} in incident #199.

---

### ERR-003 — JWT verification fails after `JWT_SECRET` rotation

**Symptom:** All authenticated requests return 401 for {N} minutes after a secret rotation; users see "session expired."

**Cause:** Token cache held pre-rotation tokens; verification used the new secret. Tokens minted before rotation cannot validate.

**Fix:**
1. Document the rotation behavior: tokens minted before rotation will fail until they expire naturally.
2. For zero-downtime rotation: maintain a brief overlap window where both old and new secrets are valid for verification.

**Prevention:**
- Document the expected behavior so it's not a surprise.
- For controlled rotations: implement dual-secret verification with a sunset on the old secret.

**Verified:** {YYYY-MM-DD} by {@user} in scheduled rotation runbook v3.

---

<!-- Add new entries above this line. -->

---

## Audit log

| Date       | Reviewer  | Status    | Notes                                    |
|------------|-----------|-----------|------------------------------------------|
| {date}     | {@user}   | created   | Initial catalog with 3 example entries.  |
