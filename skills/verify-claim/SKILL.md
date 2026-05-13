---
name: verify-claim
description: Turn an assertion ("the API supports X", "this function does Y", "the file is at Z") into runnable evidence. Use when you, the user, or a doc/memory has made a claim that should be verified before acting on it.
---

# /verify-claim — turn claims into evidence

The single most expensive failure mode of AI-assisted coding is **confident wrongness**: someone (you, the user, a doc, a memory) asserts something is true, and the team acts on it, and it's not.

This skill is the antidote. Given a claim, you generate evidence (or anti-evidence). The output is: "Verified, here's how" — or "Wrong, here's what's actually true."

## When to use this skill

Invoke when:

- The user makes a load-bearing claim before acting on it ("the API endpoint is at `/v2/users`")
- A memory or note references a file, function, or behavior — and you're about to act on it
- A doc claims something the code may or may not still do
- You catch yourself thinking "I think this works..." — stop and verify

Use this skill **proactively** when stakes are high: before recommending a fix, before quoting a version, before naming a function in customer-facing chat.

## The protocol

### Step 1 — State the claim precisely

Ask the user (or write it yourself) in this format:

> **Claim:** {one sentence asserting something verifiable}

Examples:
- "The function `parseQuery` is defined in `src/search/parse.ts`."
- "The `/api/users` endpoint requires authentication."
- "The current version is 1.4.2."
- "The dark-mode flag defaults to off."
- "There are 12 connectors in the catalog."

A claim must be:
- **Specific** — naming a file, function, value, behavior
- **Falsifiable** — can be shown wrong with a tool call
- **Singular** — one thing at a time

If the claim is vague ("the search is broken"), refine it first. "Search returns duplicate results when the query contains quotes" is a claim; "search is broken" is a feeling.

### Step 2 — Identify the verification mechanism

Match the claim to a verification:

| Claim type | Verification |
|----|----|
| File exists at X | `Read X` or `ls X` |
| Function/symbol exists | `Grep` for the definition |
| Function signature is X | Read the source |
| Endpoint exists | `curl` the endpoint locally |
| Endpoint requires auth | `curl` without auth, check status |
| Variable has value V | Read the source, check assignments |
| Test exists for behavior | Grep test files for the assertion |
| Behavior produces output X | Run the relevant code or test |
| Doc claim matches code | Read both, compare |
| Memory says X is current | Read current state, compare |
| Library has method Y | Read types/docs |
| URL returns X | Fetch the URL (render in browser if it's an SPA) |

### Step 3 — Execute the verification

Run the tool. Read the output. Be precise about what you observe.

**Do not summarize prematurely.** If you grep and find 3 hits, report all 3 (with file:line). Don't say "yes, it's defined somewhere."

### Step 4 — Report

Report in one of three states:

#### Verified

> **Verified.** {Restate the claim with evidence.}
>
> Evidence: {tool output excerpt or file:line reference}.

#### Refuted

> **Refuted.** {What's actually true.}
>
> Claim: {original}
> Actual: {what you found}
> Evidence: {tool output excerpt or file:line reference}.

#### Inconclusive

> **Inconclusive.** {What blocked verification.}
>
> Reason: {e.g., function exists but behavior depends on runtime state; need to run the test to confirm}.
> Recommendation: {next step}.

### Step 5 — Recommend next action

If verified: proceed with the action that depended on the claim.

If refuted:
- **Update the source.** If a doc claimed it, update the doc. If a memory claimed it, update the memory. If the user assumed it, correct the assumption.
- **Re-plan.** What does the user need to do differently now?

If inconclusive: state what would resolve it (run the test, hit the endpoint, ask a human).

## Example conversations

### Verifying a file path before acting

> **User**: `/verify-claim` — the JWT verification logic is in `src/auth/jwt.ts`.
>
> **Assistant**: Checking.
>
> *(Read src/auth/jwt.ts — file exists, contains JWT verification logic.)*
>
> **Verified.** `src/auth/jwt.ts` exists and contains the JWT verification logic. The relevant function is `verifyJwt` at line 24.
>
> Evidence: `src/auth/jwt.ts:24` defines `export function verifyJwt(token: string, secret: string): JwtPayload`.

### Refuting a memory-claim

> **User**: `/verify-claim` — the current version is 1.4.2.
>
> **Assistant**: Checking package.json.
>
> *(Read package.json line 3.)*
>
> **Refuted.** The current version is 1.4.7, not 1.4.2.
>
> Evidence: `package.json:3` reads `"version": "1.4.7"`. The memory or note that said 1.4.2 is out of date.
>
> Recommendation: update wherever 1.4.2 was sourced from. If it was a memory entry, refresh it.

### Verifying an external URL with the SPA-200 trap

> **User**: `/verify-claim` — the privacy policy is reachable at `https://example.com/privacy`.
>
> **Assistant**: HTTP 200 alone can lie on SPAs — the server serves a homepage shell for unknown routes with a 200 status, and 404s are rendered client-side. I'll render the URL in a browser to confirm.
>
> *(Browser tool: navigate to URL, take screenshot, read page title and visible content.)*
>
> **Verified.** The URL renders a page titled "Privacy Policy" with substantive content (sections on data collection, retention, user rights). The page is real, not a fallback shell.
>
> Evidence: page title = "Privacy Policy"; visible content includes the heading "Data we collect" and three policy sections.

### Verifying an API claim

> **User**: `/verify-claim` — `/api/users` requires authentication.
>
> **Assistant**: Testing locally.
>
> *(Bash: `curl -i http://localhost:3001/api/users`)*
>
> **Verified.** Without an auth header, `/api/users` returns 401 Unauthorized.
>
> Evidence: response = `HTTP/1.1 401 Unauthorized`, body = `{"error":"missing or invalid token"}`.

## What to refuse

- **Vague claims.** "The system works" can't be verified. Refine first.
- **Multi-part claims.** "X and Y and Z" — verify each separately.
- **Claims about the future.** "The deploy will succeed" is a prediction, not a verifiable claim today. Verify intermediate steps (build succeeds, tests pass).
- **Claims requiring production access** if you're operating in dev. State the limit, recommend the right environment.

## When NOT to invoke

- Claims that are obvious from currently-visible context. If you literally just read the file, you don't need to "verify" it's there.
- Trivial claims with low blast radius. "The string says hello" — just read it once and move on.

This skill is for **load-bearing claims**: things you're about to act on, where being wrong costs real work.

## What "done" means for this skill

- The claim is verified, refuted, or marked inconclusive — with explicit evidence.
- If refuted, the wrong source (doc, memory, assumption) is identified and updated.
- The user has clear next steps.
