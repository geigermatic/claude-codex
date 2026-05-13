# Principles — architecture, code quality, naming

The rules in this file govern how you write code: how you structure it, how you name it, what you keep small, what you refuse to add. They apply to every language.

For language-specific guidance (TypeScript, Rust), see `claude/languages/`. For testing-specific guidance, see `claude/testing.md`. For thinking and verification discipline, see `claude/EPISTEMICS.md`.

---

## 1. Architecture

### 1.1 Single responsibility

**RULE:** Every file, function, class, and service should have one reason to change. If you need "and" to describe what it does, it's probably two things.

**Why:** Multiple responsibilities in one unit cause changes to ripple unpredictably. A bug fix in one concern breaks an unrelated one. Tests become entangled.

**How to apply:**
- If a function exceeds ~40 lines, look for the second responsibility hiding inside it.
- If a file describes itself as "X-related utilities," it has no responsibility — break it apart.
- A service named `UserAndBillingService` is two services.

### 1.2 Dependency inversion

**RULE:** Depend on interfaces and types, not concrete implementations. Core pipeline code must never import domain-specific modules directly.

**Why:** Concrete dependencies make code rigid. Test doubles become impossible. Cross-cutting concerns leak into business logic.

**How to apply:**
- Core code takes a `Storage` interface; the production code injects `PostgresStorage`; tests inject `InMemoryStorage`.
- A search pipeline takes `Embedder` and `VectorStore` as parameters — it never imports OpenAI or pgvector directly.
- Domain-specific logic lives in adapters, not in the core flow.

### 1.3 Contract-first

**RULE:** Define API shapes, type signatures, and interfaces **before** writing implementations.

**Why:** Contracts force you to think about consumers. They expose ambiguity early, when it's cheap. Implementation-first code leaks internal concerns into public surfaces.

**How to apply:**
- Write the TypeScript interface or Rust trait first. Discuss it. Then implement.
- For HTTP APIs, write the request/response schema first. OpenAPI, zod, serde — pick one, use it.
- Implementation convenience is **never** a reason to weaken a contract.

### 1.4 Least knowledge (Law of Demeter)

**RULE:** Modules talk to direct collaborators only. No reaching through objects: `a.b.c.doThing()` is a smell.

**Why:** Chained access creates coupling to internal structure. Refactoring `b` breaks every caller that knows `c`. Tests must construct deep object graphs.

**How to apply:**
- If you find yourself writing `user.account.subscription.plan.name`, expose `user.planName` instead. Wrap and delegate.
- Each layer reveals only what the next layer needs. No transitive access.

### 1.5 Immutability by default

**RULE:** Prefer `const`, immutable structs, copies, and new objects over mutation.

**Why:** Mutation across async boundaries causes bugs you cannot reproduce. Shared mutable state is the source of nearly every race condition.

**How to apply:**
- Default to `const` in TypeScript, `let` only when you genuinely re-assign.
- In Rust, prefer borrowing immutably. `&mut` requires justification.
- Pipeline stages should produce new objects, not mutate inputs.
- If you must mutate, do it locally in one function. Never return a mutated argument.

### 1.6 Fail fast

**RULE:** Validate inputs at the boundary of every public function. Throw on invariant violations rather than propagating bad state.

**Why:** Errors caught at the source carry context: which input, which call, which assumption. Errors that propagate become "undefined is not a function" three layers deep.

**How to apply:**
- Use schema validation (zod, serde, ajv) at every HTTP boundary, file boundary, and LLM output boundary.
- Inside the system, validate at the boundary of each module. Internal callers should be trusted *because* the boundary validated.
- A function that takes `user: User` should not also check `if (!user)` — the caller's job.

### 1.7 Tenant and trust boundaries are invariants

**RULE:** If your system has multi-tenancy, every database query, cache key, log line, and LLM context must be scoped by tenant. Enforce structurally — middleware, query builders, scoped clients — not by developer memory.

**Why:** Cross-tenant data leaks are catastrophic and silent. They almost never come from one big mistake — they come from one developer forgetting one filter.

**How to apply:**
- Wrap the raw database client so it cannot be used without a tenant ID.
- Make `tenantId` a required first argument on every query function.
- Audit log every cross-tenant read for operator overrides.

---

## 2. Code quality

### 2.1 Small functions

**RULE:** If a function exceeds ~40 lines or needs more than 3-4 parameters, decompose it.

**Why:** Small functions are easier to name, test, and reuse. Long functions hide their second and third responsibilities.

**How to apply:**
- A function should fit in your editor without scrolling.
- If you reach for `// Step 1: ... // Step 2:` comments, those steps are sub-functions.
- 5 parameters means an options object (TS) or a builder (Rust).

### 2.2 No boolean parameters

**RULE:** Functions that take boolean parameters are unreadable at the call site. Use named options objects, enums, or separate functions.

**Why:** `search(query, true, false)` tells the reader nothing. The first `true` is "fuzzy"? "case-sensitive"? Three months later, no one knows.

**How to apply:**
- TS: options object. `search(query, { fuzzy: true, caseSensitive: false })`.
- Rust: struct args or builder pattern. `Search::new(query).fuzzy().build()`.
- Sometimes two separate functions are clearer than one with a flag: `findUser` and `findUserCaseSensitive`.

### 2.3 Explicit over clever

**RULE:** Readable code beats concise code. No nested ternaries. No side-effect-heavy one-liners. No "look at this neat trick."

**Why:** Code is read 100x more than it is written. Cleverness costs the reader; explicitness costs nothing.

**How to apply:**
- Replace `a ? (b ? x : y) : z` with `if/else`.
- A one-liner that does three things should be three lines.
- If a junior on the team can't trace what your code does, rewrite it.

### 2.4 Collocate related code

**RULE:** Keep types, helpers, and tests near the code they serve. Avoid distant `utils/`, `types/`, and `helpers/` directories.

**Why:** When a feature lives in 7 directories, every change touches 7 places. When it lives in 1 directory, refactoring is local.

**How to apply:**
- A feature is a folder, not a sprinkling across the tree.
- A "helper" used by one consumer belongs next to that consumer.
- A "helper" used by everyone is a core module — give it a real name.

### 2.5 Dead code = deleted code

**RULE:** Do not comment out code "just in case." Delete it. Git remembers everything.

**Why:** Commented code rots. It misleads readers about what's active. It pollutes search results.

**How to apply:**
- If you're unsure whether to remove code, remove it. You can always retrieve it with `git log -p`.
- The same applies to unused imports, unused parameters (prefix with `_` if the language requires keeping them), and unused branches.

### 2.6 Stay in scope

**RULE:** The user asked for X. Deliver X. Do not refactor, abstract, or "improve" code outside the requested change.

**Why:** Scope creep makes diffs unreviewable, increases blast radius, and conflates concerns. A bug fix mixed with a refactor is two PRs pretending to be one.

**How to apply:**
- A bug fix changes only what the bug requires.
- A new feature does not justify cleaning up adjacent code.
- If you spot something worth fixing, **mention it** in your status report — do not silently fix it.
- One-shot operations do not need helpers. Three similar lines is better than a premature abstraction.

### 2.7 No defensive programming inside trust boundaries

**RULE:** Validate at the boundary. Do not validate inside the system unless you are crossing another boundary.

**Why:** Defensive checks at every layer create noise, hide the real validation, and "handle" cases that cannot happen.

**How to apply:**
- An HTTP handler validates the body with zod. The service function it calls trusts the parsed body.
- Do not add `if (!user)` inside a function that takes `user: User` (non-nullable). If the type system says it's there, trust it.
- The exception is when crossing another trust boundary: calling an external API, reading from disk, parsing LLM output. Validate there.

### 2.8 Comments earn their place

**RULE:** Default to writing no comments. Only add one when the **why** is non-obvious.

**Why:** Comments rot. Comments lie. Comments restate what the code already says. The exceptions are: hidden constraints, subtle invariants, workarounds for specific bugs, surprising behavior.

**How to apply:**
- Do not explain WHAT the code does — well-named identifiers do that.
- Do not reference the current task, fix, or callers ("used by X", "added for the Y flow"). That belongs in the PR description and rots in the code.
- A comment that says "we have to do it this way because of [bug in library X version Y]" is valuable. A comment that says "// loop through users" is noise.

### 2.9 Trust the type system

**RULE:** If your type system enforces an invariant, do not also check for it at runtime.

**Why:** Runtime checks for compile-time invariants confuse readers, suggest the type is unreliable, and become stale when types change.

**How to apply:**
- TS with `strict: true` makes `null` impossible without explicit opt-in. Don't write `if (x === null)` on a non-nullable type.
- Rust's `Option<T>` and `Result<T, E>` are exhaustive. Don't add `match _ => unreachable!()` unless you've made the case truly unreachable.

---

## 3. Naming

Naming is half of design. Bad names mislead; good names teach.

### 3.1 Functions are verbs

**RULE:** Functions and methods start with a verb. Nouns are for data.

**Why:** Verb-first names make call sites read like English. `getUser()` vs `userData()` — one tells you it's a function call, one is ambiguous.

**Examples:**
- Good: `getUser`, `calculatePrice`, `validateInput`, `parseConfig`, `renderRow`
- Bad: `userData`, `pricing`, `input`, `config`, `row`

### 3.2 Booleans say true/false

**RULE:** Boolean variables and functions start with `is`, `has`, `can`, `should`, or `will`.

**Why:** A name like `validUser` could be the user, the validation result, or a flag. `isValidUser` is unambiguous.

**Examples:**
- Good: `isAuthenticated`, `hasSubscription`, `canEdit`, `shouldRetry`, `willExpireToday`
- Bad: `valid`, `subscription` (as a boolean), `editPermission`, `retry` (as a boolean)

**Avoid double negatives:**
- `enabled: true` over `disabled: false`
- `isVisible` over `isHidden`
- If you must name the negative case, use `Disabled` not `NotEnabled`.

### 3.3 Types and interfaces

**RULE:** PascalCase. No prefix conventions like `I` or `T` for interfaces/types.

**Why:** `IUser` adds three characters of noise. Hungarian-style prefixes were a workaround for languages without type information; modern editors show types on hover.

**Examples:**
- Good: `User`, `Order`, `PaymentMethod`, `SearchResult`
- Bad: `IUser`, `TOrder`, `PaymentMethodInterface`, `SearchResultType`

### 3.4 Constants

**RULE:** SCREAMING_SNAKE_CASE only for **true** constants — values that are literally immutable and known at compile time. Configuration, even if rarely changed, is not a constant.

**Why:** ALL_CAPS visually screams "this is fixed forever." Using it for configurable values misleads readers about mutability.

**Examples:**
- Good: `MAX_RETRY_COUNT = 3`, `PI = 3.14159`, `STATUS_OK = 200`
- Bad: `DATABASE_URL` (configuration, not a constant), `DEFAULT_TIMEOUT` (configurable in production)

For configuration, use camelCase or snake_case as the language convention dictates: `config.databaseUrl`, `config.default_timeout`.

### 3.5 Enums and unions

**RULE:** Singular noun. The members are the plural set.

**Why:** `UserRole` describes one role; `UserRoles.Admin` is one of many roles. `Roles.Admin` reads as "roles dot admin," which is grammatically off.

**Examples:**
- Good: `UserRole`, `OrderStatus`, `LogLevel`
- Bad: `UserRoles`, `OrderStatuses`, `LogLevels`

### 3.6 Domain words beat technical jargon

**RULE:** Use the words your users use. Avoid generic technical names.

**Why:** Code that uses domain vocabulary is self-documenting to anyone who knows the domain. Code that uses generic CS terms forces translation.

**Examples:**
- Good (in a billing context): `Invoice`, `LineItem`, `taxRate`, `gracePeriod`
- Bad: `Document`, `Record`, `multiplier`, `delay`

**The "Manager / Helper / Util" smell:**
If you can't name a class without `Manager`, `Helper`, `Util`, `Service` (when it's not a service), or `Handler` (when it's not a handler), the abstraction is wrong. What does `UserManager` *manage*? Whatever the answer is, that's its real name.

- Bad: `UserManager`, `OrderHelper`, `AuthUtil`, `EmailService` (if it just sends one email)
- Good: `UserRepository`, `OrderPricer`, `JwtVerifier`, `EmailSender`

### 3.7 Generic parameters

**RULE:** `T`, `U`, `V` are acceptable for completely generic type parameters. Once constrained, give them descriptive names.

**Examples:**
- Good: `Result<T, E>`, `Map<K, V>` (truly generic)
- Good: `function paginate<TItem>(items: TItem[]): Page<TItem>` (item-shaped)
- Bad: `function paginate<T>(items: T[]): Page<T>` if elsewhere there's a `TUser` — be consistent

### 3.8 Files

**RULE:** Match the language's convention. Be consistent within the repo.

- TypeScript / JavaScript: `kebab-case.ts` (community standard) or `camelCase.ts`. Pick one. Don't mix.
- Rust: `snake_case.rs`.
- The file name matches its primary export. `user-repository.ts` exports `UserRepository`.

### 3.9 Tests

**RULE:** Test names read as sentences describing the behavior under test.

**Why:** A failing test should explain what's broken in plain language without reading the test body.

**Examples:**
- Good: `it('returns an empty array when no documents match the query')`
- Good: `it('rejects invalid email addresses with a 400 response')`
- Bad: `it('test1')`, `it('search works')`, `it('user creation')`

### 3.10 Abbreviations

**RULE:** Don't abbreviate. The savings are small; the cost in readability is large.

**Exceptions:** A small set of universally-understood abbreviations: `id`, `url`, `ctx` (for context), `req` / `res` (in HTTP handlers only), `db` (in obviously-database scope).

**Forbidden:**
- `usr`, `mgr`, `cfg`, `tmp`, `auth` (as a variable — `authToken` is fine), `addr`, `cmd`
- Numbers in names unless they mean something: `usersV2` is OK if there's a real v1/v2 split; `userResult2` is not.

### 3.11 Negation hygiene

**RULE:** Avoid `!isInvalid`. Avoid `if (!disabled)`. Name the positive case.

**Why:** Double negatives confuse. The reader has to compute "not not" mentally.

**Examples:**
- Good: `if (isValid) { ... }`
- Bad: `if (!isInvalid) { ... }`

---

## 4. Anti-patterns to refuse

These come up constantly. Refuse them when proposed; remove them when found.

### 4.1 Premature abstraction

**Smell:** Adding an interface, base class, or generic for "future flexibility."

**Why it's wrong:** You don't know the future. The abstraction guesses. When the future arrives, the abstraction is shaped wrong and now you must refactor *and* maintain compatibility.

**Rule of three:** Wait until you have **three** concrete cases before extracting a shared abstraction. Two cases is duplication; three is a pattern.

### 4.2 Premature optimization

**Smell:** Caching, memoization, indexing, batching — without a benchmark showing the problem.

**Why it's wrong:** Optimization adds complexity. Adding complexity for a problem you haven't measured is paying a real cost for an imaginary benefit.

**Rule:** Profile first. Optimize what's slow. If you can't point to a number, you can't optimize.

### 4.3 Defensive programming inside trust boundaries

(Covered in 2.7 — included here because it's a constant temptation.)

### 4.4 Stringly typed code

**Smell:** Using strings for things that have a finite, known set of values: status codes, types, roles, kinds.

**Why it's wrong:** No compile-time check. Typos pass review. Refactoring is grep-and-pray.

**Fix:** Discriminated unions (TS), enums (Rust), or const objects with `as const`.

### 4.5 Magic numbers

**Smell:** `if (retries > 3)`, `setTimeout(fn, 5000)`, `if (status === 200)`.

**Why it's wrong:** The number has meaning; the meaning is invisible.

**Fix:** Named constants. `MAX_RETRIES = 3`, `RETRY_DELAY_MS = 5000`. For HTTP status, use library constants if available.

### 4.6 God objects

**Smell:** A class with 20+ methods, fields covering unrelated concerns, names like `AppContext` or `SystemState`.

**Why it's wrong:** Untestable. Unchangeable. Every change risks unrelated bugs.

**Fix:** Decompose into focused units. The god object is the second responsibility of every method on it.

### 4.7 Long parameter lists

**Smell:** A function takes 5+ parameters, especially when several are optional booleans or strings.

**Why it's wrong:** Call sites become incomprehensible. Refactoring requires updating every caller. Adding a parameter is a breaking change.

**Fix:** Options object (TS), builder pattern (Rust), or — most often — the function is doing too much. Split it.

### 4.8 Deep nesting

**Smell:** Code indented 4+ levels.

**Why it's wrong:** Reader has to hold the conditions of every wrapping block in their head.

**Fix:** Early returns. Guard clauses. Extract the inner logic to a named function.

### 4.9 Mutation across function boundaries

**Smell:** Function takes an object, modifies it, returns nothing or returns the same object.

**Why it's wrong:** Hidden side effects. Tests that share fixtures break each other. Async callers see partial state.

**Fix:** Return a new object. Treat inputs as read-only.

### 4.10 Cargo-culted patterns

**Smell:** Adding patterns ("factories," "managers," "repositories," "providers") because that's what the framework example showed, without understanding what they buy.

**Why it's wrong:** Patterns are solutions to specific problems. Applied without the problem, they are pure cost.

**Fix:** For every pattern, articulate the concrete benefit in your codebase. If you can't, don't use it.

---

## 5. The minimum-viable-diff principle

**RULE:** Every change should be the smallest possible diff that delivers the requested outcome.

**Why:** Small diffs are reviewable. Small diffs are revertible. Small diffs are less likely to hide regressions.

**How to apply:**
- Bug fix: change the line that's broken, plus the test that reproduces the bug. Not the surrounding code.
- New feature: add only what the feature requires. Resist refactoring "while you're in there."
- If a change requires touching N files, ask whether N is justified or whether you're widening scope.

**Exception:** Sometimes a small fix genuinely requires a structural change first (e.g., the bug exists because of an architectural flaw). In that case, do the structural change as a separate commit/PR, then do the fix on top. Two small diffs beat one large one.

---

## 6. The "what to leave out" principle

**RULE:** Add nothing that the task does not require. No error handling for impossible cases. No flags for hypothetical futures. No abstractions for one consumer.

**Why:** Code you didn't write has no bugs. Code you didn't write costs no review time. Code you didn't write doesn't have to be maintained.

**How to apply:**
- Don't add a `try/catch` around code that can't throw.
- Don't add a feature flag unless you actually intend to toggle it.
- Don't add a `BaseFoo` class when there's one `Foo`.
- Don't add backwards-compatibility shims for code with one caller you control.
- Don't half-finish: an incomplete abstraction is worse than no abstraction.

**The corollary:** When you read code, ask "could this be deleted?" If yes, propose deleting it.
