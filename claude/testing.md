# Testing — TDD, the pyramid, hypothesis-driven debugging

Tests are not paperwork. Tests are the proof that the code does what it claims, and the safety net that lets you refactor without fear. Code without tests is code you cannot change.

This file covers what to test, when to test it, how to name and structure tests, and the bug-fix and slice-based TDD workflows.

For verification discipline that surrounds testing (running the tests, reading the output), see `claude/EPISTEMICS.md`.

---

## 1. Test-first, always — for non-trivial work

**RULE:** For every non-trivial feature and every bug fix, write the failing test before writing the implementation.

**Why:** A test written after the code tends to test what the code does, not what the code should do. The test passes because the code passes. That is not a check; it is a tautology.

A test written first describes the desired behavior. The implementation must satisfy the test. The test is now a load-bearing check on real intent.

### What "non-trivial" means

- New behavior visible to a user, caller, or consumer
- Bug fixes (always — write the reproduction first)
- Refactors that change observable behavior (rare, by definition; if you have to, write characterization tests first)

### What is trivial

- Renaming a private variable
- Fixing a typo in a string
- Adjusting log levels
- Updating documentation
- Bumping a dependency version (the test suite is the test)

When in doubt: write the test. The cost is 5 minutes. The benefit compounds forever.

---

## 2. The bug-fix protocol

When a bug is reported, do not start by fixing it. Start by reproducing it as a failing test.

### Steps

1. **Read the bug report.** What is the symptom? What was expected?
2. **Write a failing test** that exhibits the symptom. The test name describes the buggy behavior in plain language.
3. **Run the test. Confirm it fails.** This proves the test actually tests the bug.
4. **Form a hypothesis** about the root cause. State it.
5. **Make the minimum change** to make the test pass.
6. **Run the test. Confirm it passes.**
7. **Run the full test suite.** No other tests should have broken.
8. **Read your diff.** Does the change make sense, or did you accidentally over-fix?

### What this prevents

- "Fixes" that don't actually fix the reported bug (you fixed something adjacent)
- Regressions where the bug returns six months later and no one notices (because there's no test guarding it)
- Over-fixes where you change three things and don't know which one mattered

---

## 3. The test pyramid

**RULE:** Many fast unit tests, fewer integration tests, rare end-to-end tests.

```
        /\
       /  \   E2E (few, slow, brittle)
      /----\
     /      \  Integration (some, medium)
    /--------\
   /          \ Unit (many, fast)
  /------------\
```

### Unit tests

- One module/function under test, no I/O, no network, no clock, no filesystem (except temp dirs).
- Run in milliseconds. The entire unit test suite for a file runs in under one second.
- Test doubles (mocks, fakes, stubs) for collaborators.
- The majority of your tests live here.

### Integration tests

- Multiple modules together, real I/O for the boundary being tested.
- Database tests use a real database (Postgres, SQLite — not mocks).
- HTTP tests hit a real local server.
- Slower (seconds), so used sparingly to cover boundaries between units.

### End-to-end tests

- Whole-system flows. Browser drives the UI; UI hits the API; API hits the database.
- Slowest, flakiest, most expensive to maintain.
- Reserved for the most critical user journeys: login, checkout, primary CRUD on the main entity.
- Not the place to cover edge cases — those belong in unit tests.

### Common mistake

Writing integration tests as if they were unit tests, because integration is "more realistic." It is also 100x slower. Most of your assertions belong in unit tests; integration only proves the wiring.

---

## 4. Arrange-Act-Assert

**RULE:** Every test has three clearly-separated phases:

1. **Arrange** — set up the state and inputs.
2. **Act** — exercise the code under test, one operation.
3. **Assert** — verify the outcome.

Write them with blank lines between or with comments if helpful. Do not mix the phases.

### Example

```ts
it('returns an empty array when no documents match the query', () => {
  // Arrange
  const store = new InMemoryDocumentStore();
  store.add({ id: '1', title: 'Cats' });
  store.add({ id: '2', title: 'Dogs' });

  // Act
  const results = store.search('birds');

  // Assert
  expect(results).toEqual([]);
});
```

### Why this matters

A test that arranges and asserts in the same line, or that has three "acts" interleaved with checks, is testing too many things. It will fail for unclear reasons. It will be expensive to update when the system changes.

**Rule:** One act per test. If you find yourself wanting two, write two tests.

---

## 5. Test behavior, not implementation

**RULE:** Tests assert outcomes (what), not mechanics (how). Refactoring internals must not break tests.

### Behavior tests

```ts
it('charges the customer the correct total', () => {
  const cart = new Cart();
  cart.add({ price: 10 });
  cart.add({ price: 20 });

  expect(cart.total()).toBe(30);
});
```

This test will survive any refactor of how the cart computes totals.

### Implementation tests (avoid)

```ts
it('calls priceCalculator.sum for each item', () => {
  const cart = new Cart();
  cart.add({ price: 10 });

  expect(priceCalculator.sum).toHaveBeenCalledWith([10]);
});
```

This test breaks the moment you decide to inline the calculation. It tests how, not what.

### When mocks are appropriate

- At trust boundaries: external APIs, slow I/O, things you don't own.
- Verifying that a side effect occurred (e.g., "did we send the email?").

### When mocks are inappropriate

- Inside your own module — use real collaborators.
- For functions that have no side effects — just call them.

---

## 6. Test isolation

**RULE:** Each test sets up its own state. Tests must pass in any order and individually.

**Why:** Shared state between tests creates false passes (test A leaves state that test B accidentally relies on) and confusing failures (the test fails only when test B runs, but you swear the bug is in test C).

**How to apply:**
- No shared module-level fixtures across tests.
- Reset databases / caches / file state in `beforeEach` or use a fresh instance per test.
- A test should pass if you delete every other test in the file.

**Smoke test for isolation:** Pick three tests at random, run only those, in reverse order. They should all pass.

---

## 7. Test naming

**RULE:** Test names read as sentences describing behavior under test.

### Format

```
it('<verb-phrase describing the outcome>')
it('<does X> when <condition>')
it('throws <error> if <invalid input>')
```

### Good

- `it('returns an empty array when no documents match the query')`
- `it('rejects invalid email addresses with a 400 response')`
- `it('retries up to 3 times on transient network errors')`
- `it('throws InvalidInputError if the email is missing the @ symbol')`

### Bad

- `it('test1')`
- `it('search works')`
- `it('user creation')` (noun, not a sentence)
- `it('should work')` (works in what way?)
- `it('returns correct result')` (what is correct?)

### Why this matters

When a test fails in CI, you read the name in a list of 200. A good name tells you what's broken without clicking through. A bad name forces you to read the body and the code under test.

---

## 8. The single-act rule, restated

If your test has more than one `expect(...)` covering different concerns, it's probably two tests pretending to be one.

### Acceptable cluster of assertions

```ts
it('returns a paginated response with correct metadata', () => {
  const result = paginate(items, { page: 2, perPage: 10 });

  expect(result.items).toHaveLength(10);
  expect(result.page).toBe(2);
  expect(result.totalPages).toBe(5);
});
```

All three assertions verify one outcome: the pagination shape is correct.

### Two tests pretending to be one

```ts
it('handles users correctly', () => {
  const user = createUser({ email: 'a@b.com' });
  expect(user.id).toBeDefined();

  user.email = 'c@d.com';
  expect(user.email).toBe('c@d.com');

  deleteUser(user.id);
  expect(getUser(user.id)).toBeNull();
});
```

Three acts. Three concerns. Split into three tests.

---

## 9. Hypothesis-driven debugging in tests

When a test fails unexpectedly, do not change the test until you understand the failure.

### The protocol

1. **Read the failure message.** Most test runners tell you what was expected, what was actual, and where.
2. **Form a hypothesis**: "I expected X. Actual is Y. The most likely cause is Z."
3. **Verify the hypothesis**: read the relevant code, add a single log line, run the test again.
4. **If the hypothesis was right**, fix the cause. If wrong, form a new hypothesis. Do NOT keep changing things.
5. **Once fixed**, ensure no other tests broke.

### Forbidden patterns

- Changing the test to match the actual output. The test is the spec. If the spec is wrong, that's a conversation, not a silent edit.
- Adding `.skip` or `.only` to bypass failures. Either fix the test or delete it deliberately.
- "Retrying" the test by re-running it. Flaky tests are bugs, not weather.

---

## 10. Flaky tests are bugs

**RULE:** A test that sometimes passes and sometimes fails is a bug, not a nuisance. Fix it or delete it.

**Why:** Flaky tests train the team to ignore failures. Once one test is flaky, every red CI run is "probably flaky." Real regressions hide in the noise.

### Causes of flakiness

- **Time:** `setTimeout`, `Date.now()`, races between async operations. Use fake timers or inject the clock.
- **Order dependence:** test A modifies global state that test B accidentally relies on. Fix isolation.
- **Real network calls:** intermittent failures from third parties. Mock them.
- **Real concurrency:** `Promise.race` results, parallel writes to shared state. Make the test deterministic.

### Recipe

1. Reproduce locally by running the test 100 times in a loop.
2. When it fails, capture the output.
3. Find the source of non-determinism. Eliminate it.
4. Re-run 100x to confirm.

If you can't fix it, delete it. A deleted flaky test is honest. A skipped flaky test is debt.

---

## 11. Concurrency and synchronous-wait patterns

**RULE:** When a feature involves waiting for child operations, parallel work, or coordinated state, verify that your concurrency limits exceed expected concurrent waits.

**Why:** Job queues with concurrency 1 deadlock when a worker waits on another worker. Connection pools with size N deadlock when N+1 simultaneous queries are made. These bugs do not appear in tests with a single user; they appear in production with two.

### Pre-merge checklist for sync-wait patterns

- [ ] Does this feature involve waiting on another job, request, or async operation?
- [ ] What is the maximum number of concurrent waits?
- [ ] Does the worker / pool / queue concurrency exceed that number?
- [ ] Is there a test that simulates N+1 concurrent operations and confirms no deadlock?

### Example

If your job queue has `concurrency: 1` and your new feature involves Job A waiting for Job B (where B also runs on the same queue), you have just designed a deadlock. The fix is `concurrency >= 2` plus a test that exercises the wait under load.

---

## 12. Test-data builders

For complex domain objects, use builders rather than inline literals.

### Without a builder

```ts
const user = {
  id: 'u-1',
  email: 'test@example.com',
  createdAt: new Date('2024-01-01'),
  role: 'member',
  preferences: {
    theme: 'light',
    notifications: true,
  },
  // ... 15 more fields
};
```

Tests become unreadable. A change to the User shape touches every test.

### With a builder

```ts
const user = aUser().withRole('admin').build();
```

The builder provides sane defaults; tests specify only what matters for their case. Schema changes touch one place.

### Builder rules

- The builder lives next to the type, not in a `test-utils/` directory.
- The default-constructed object is valid for "happy path" tests.
- Builder methods named `withX` set X explicitly.
- Reusable across the codebase, not duplicated per test file.

---

## 13. Coverage is a smoke alarm, not a target

**RULE:** Coverage is useful for finding untested code. It is dangerous as a target.

**Why:** When teams chase coverage percentages, they write low-value tests (testing getters and setters) to hit the number. Coverage goes up; quality goes down.

### How to use coverage

- Run it locally. Look for files with low coverage — they often contain risky, untested logic.
- Investigate why: did the writer skip it because it's hard to test, or because it's not worth testing?
- Use coverage gaps as a conversation starter, not a metric to hit.

### What to never do

- Set a CI gate at "85% coverage."
- Reject PRs for coverage drops without examining the actual code.
- Write tests purely to raise the number.

---

## 14. What to test, what to skip

### Always test

- Pure functions with non-trivial logic
- Public API of every module
- Bug fixes (the regression test)
- Boundary cases: empty input, max input, off-by-one
- Error paths: what happens when X throws?
- Security-sensitive code: input validation, authorization, tenant scoping

### Skip with confidence

- Trivial getters/setters (no logic)
- Framework code (test your code, not the framework)
- Wiring/configuration files
- Generated code

### Edge case to consider

- "I'd test it but it requires mocking 5 things." — that's a sign the code under test has too many dependencies. Test the design first.

---

## 15. The TDD rhythm: red, green, refactor

Once you're comfortable with test-first:

1. **Red:** write a failing test. The simplest test that fails.
2. **Green:** write the simplest implementation that makes the test pass. Hard-code if necessary.
3. **Refactor:** with the test as a safety net, improve the code. Tests stay green.
4. Repeat with a slightly more demanding test.

The discipline is in step 2: write the simplest thing that passes. Resist the urge to write a "complete" implementation in one go. Each cycle is small.

**Why this works:** every line of production code exists to satisfy a test. There is no untested code. The design emerges from the test cases.

**When this is overkill:** trivial changes, prototypes you'll throw away, exploratory spikes. Use judgment. The full TDD rhythm is for code you intend to ship.

---

## 16. Quality tests vs unit tests

For LLM-driven code (or anything with non-deterministic output), distinguish two test classes:

### Unit tests

Deterministic, fast, automated. Run on every commit, every PR, every CI build.

### Quality tests

Evaluate output quality against expected behavior. Often involve real LLM calls, real embeddings, real retrieval. Slow, expensive, sometimes flaky.

**Rules:**
- Quality tests run **on demand**, not on every commit.
- They run when you change a pipeline file (a prompt, a retrieval step, a confidence calculator).
- They produce a comparable score over time — you lock in baselines after confirmed improvements.
- They are not gates. They are signal.

Separate the two suites. Mark quality tests clearly. Never let quality tests block CI.

---

## 17. The minimum test you should never skip

For any new feature, before merging, ensure at least:

- [ ] One unit test for the happy path
- [ ] One unit test for the most likely failure case
- [ ] If user-facing: one manual UI check or e2e test

These three take 20 minutes. They prevent the most common regressions. They are the bare minimum that lets you ship without fear.

Less than this is not "moving fast." It is racking up debt.
