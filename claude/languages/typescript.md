# TypeScript — strict mode, type safety, runtime validation

TypeScript without discipline is JavaScript with longer keystrokes. With discipline, it eliminates entire classes of bugs at compile time. This file covers the configuration, type-system patterns, and runtime-safety practices that make TS earn its weight.

For language-agnostic principles, see `claude/principles.md`. For testing in TS, see `claude/testing.md`.

---

## 1. Compiler configuration: strict mode is the floor

**RULE:** Every TypeScript project starts with `strict: true` in `tsconfig.json`. Then add the strictness flags that aren't in the default strict set.

### `tsconfig.json` minimum

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "useUnknownInCatchVariables": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": false,
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler"
  }
}
```

### Why each flag matters

- **strict**: bundles `noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, and others. The baseline.
- **noUncheckedIndexedAccess**: `arr[0]` becomes `T | undefined`. Forces you to handle the empty-array case.
- **exactOptionalPropertyTypes**: distinguishes `{ x?: T }` (optional) from `{ x: T | undefined }` (always present, may be undefined). Catches subtle bugs.
- **noImplicitOverride**: forces `override` keyword when subclasses override parents.
- **useUnknownInCatchVariables**: catch blocks get `unknown`, not `any`. Forces validation before use.
- **noImplicitReturns**: every code path returns. Catches forgotten returns.

### Don't relax these without a reason

Each flag prevents a real class of bug. If turning one off makes "too many errors appear," that's a signal — those errors are real bugs you've been blind to.

---

## 2. Avoid `any`

**RULE:** Never use `any`. If you don't know the type, use `unknown` and narrow.

### The problem with `any`

- It poisons the type system: `any` propagates through every expression it touches.
- It's a runtime trapdoor: the compiler stops checking, but the bugs are still there.
- It hides bad design: most uses of `any` exist because the developer didn't want to think about the actual type.

### `unknown` instead

`unknown` says "I don't know what this is, and I'll narrow before using it":

```ts
function parseInput(raw: unknown): User {
  if (typeof raw !== 'object' || raw === null) {
    throw new Error('Invalid input');
  }
  // raw is now object
  if (!('email' in raw) || typeof raw.email !== 'string') {
    throw new Error('Missing email');
  }
  // raw is now { email: string }
  return { email: raw.email };
}
```

Or, better, use a schema validator (see §5).

### The two acceptable uses of `any`

1. **Migrating from JavaScript:** legacy code that hasn't been typed yet, with `// TODO: type this` and a tracking issue.
2. **Library type definitions for genuinely-dynamic APIs:** but these should be wrapped immediately at the boundary.

If you find yourself writing `any` for any other reason, you have not understood the type you need.

---

## 3. Avoid `as` (type assertions)

**RULE:** Every `as` is an unchecked assumption. Use type guards or runtime validation instead.

### What `as` does

`as` tells the compiler "trust me, this is type T." It does no runtime check. If you're wrong, the bug surfaces at runtime, far from the assertion.

### When `as` is unavoidable

- Narrowing after `JSON.parse` (which returns `any`) — use a schema instead.
- DOM API results where TypeScript's lib types are loose (e.g., `event.target as HTMLInputElement`) — even here, prefer a runtime check.
- Const assertions: `as const` is fine (it's not a type assertion, it's a type-widening preventer).

### When `as` is wrong

- After `JSON.parse(body) as User` — you haven't validated anything. Use zod.
- Casting between unrelated types: `value as unknown as OtherType` — almost always indicates the design is wrong.
- Casting to satisfy a strict signature when you "know" the type is right — the compiler is right; trust it.

### Better alternatives

- **Type guards**: functions that return `value is User` and check at runtime.
- **Discriminated unions**: encode the shape in the type system.
- **Schema validation**: zod, valibot, ajv at the boundary.

---

## 4. Discriminated unions over optional fields

**RULE:** When a value can be in one of several distinct states, model it as a discriminated union, not an object with optional fields.

### The wrong way

```ts
type Result = {
  success: boolean;
  data?: User;
  error?: string;
};
```

This type permits invalid combinations: `{ success: true, error: 'oops' }`. The reader must guess which fields are set when.

### The right way

```ts
type Result =
  | { type: 'success'; data: User }
  | { type: 'error'; error: string };
```

Now the compiler enforces the shape. `result.data` is only accessible when `result.type === 'success'`. Exhaustiveness checking works:

```ts
function handle(result: Result) {
  switch (result.type) {
    case 'success': return result.data;
    case 'error':   return null;
    // If you add a third variant, the compiler reminds you to handle it.
  }
}
```

### When to use

- Async operation results
- Form states (idle, submitting, success, error)
- API responses with different shapes per status
- State machines

### Discriminator field name

Use `type` or `kind` consistently. Don't mix `status` in some and `type` in others.

---

## 5. Runtime validation at every boundary

**RULE:** Types are erased at runtime. At every system boundary, validate with a schema.

### Boundaries that need validation

- HTTP request bodies, query params, route params
- HTTP response bodies (from external APIs)
- Environment variables
- File contents (config, data)
- LLM output (treat as untrusted)
- Messages from queues, webhooks, websockets
- Database query results when the schema may have drifted

### Recommended: zod

```ts
import { z } from 'zod';

const Env = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().int().positive(),
  NODE_ENV: z.enum(['development', 'staging', 'production']),
});

export const env = Env.parse(process.env);
```

Now `env.PORT` is typed as `number`. Anything missing or malformed throws at startup, not at request time.

### Patterns

- **Parse at the boundary, trust inside.** Once `Env.parse` succeeds, every consumer trusts `env` is valid.
- **Infer types from schemas.** `type Env = z.infer<typeof EnvSchema>` keeps types and validation in sync.
- **Reject unknown fields when strictness matters.** `z.object({...}).strict()`.

### Don't validate inside trusted boundaries

Re-validating data that's already been validated at the boundary is noise. If your service function takes `user: User`, it should not also check `if (!user.email)`. The parser at the HTTP boundary already did that.

---

## 6. Branded types for domain IDs

**RULE:** Identifiers should be typed distinctly. Don't pass a `userId: string` where a `tenantId: string` is expected.

### The problem

```ts
function getUserPosts(userId: string, tenantId: string): Post[] { ... }

// Easy to swap by accident:
getUserPosts(tenantId, userId);  // compiles, ships, breaks
```

### The fix: branded types

```ts
type UserId = string & { readonly __brand: 'UserId' };
type TenantId = string & { readonly __brand: 'TenantId' };

// Constructor functions perform any validation:
function asUserId(s: string): UserId {
  if (!s.startsWith('user_')) throw new Error('Invalid UserId');
  return s as UserId;
}
```

Now `getUserPosts(tenantId, userId)` fails to compile.

### Tradeoff

Boilerplate at the boundary (you have to mint branded values). Worth it for IDs that flow widely and where mix-ups are catastrophic (tenant IDs, especially).

### Use sparingly

Brand the IDs that matter. Not every string needs to be branded — that's overkill. Tenant IDs, user IDs, currency types are common candidates.

---

## 7. Avoid TypeScript `enum`

**RULE:** Use string literal unions or `as const` objects. Avoid `enum`.

### Why

TypeScript `enum` has surprising runtime semantics:
- Numeric enums produce reverse-lookup tables (`Color[0] === 'Red'`).
- Both numeric and string enums emit JavaScript code (not erased).
- Const enums are erased, but break under some build configurations.
- Enums don't compose well with other types.

### Better: string literal unions

```ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

function log(level: LogLevel, message: string) { ... }

log('info', 'started');  // OK
log('verbose', 'x');     // compile error
```

Zero runtime cost. Composes with everything.

### Or: `as const` objects

```ts
const LogLevel = {
  Debug: 'debug',
  Info: 'info',
  Warn: 'warn',
  Error: 'error',
} as const;

type LogLevel = typeof LogLevel[keyof typeof LogLevel];
```

When you want both: a type AND a runtime object you can iterate.

---

## 8. `unknown` in catch blocks

**RULE:** With `useUnknownInCatchVariables: true`, catch variables are `unknown`. Narrow before using.

### Pattern

```ts
try {
  doSomething();
} catch (err) {
  if (err instanceof MyDomainError) {
    return { type: 'domainError', code: err.code };
  }
  if (err instanceof Error) {
    logger.error('Unexpected error', { message: err.message });
    throw err;
  }
  logger.error('Non-error thrown', { value: err });
  throw new Error('Unexpected non-error throw');
}
```

### Why

JavaScript lets you `throw 'a string'` or `throw 42`. Defensive code handles this. Without narrowing, you can't access `.message` without an `as`, and `as` is wrong here too.

---

## 9. Promise hygiene

### Always await

**RULE:** Every Promise must be awaited or explicitly marked as fire-and-forget.

```ts
async function handleRequest() {
  await saveUser(user);  // OK

  saveUser(user);  // BUG — error swallowed, may not complete before return
  void saveUser(user);  // explicit fire-and-forget; rare and intentional
}
```

Configure `@typescript-eslint/no-floating-promises` as a lint error.

### Don't mix async/await with .then()

Pick one style per function. Mixing makes the control flow hard to follow.

### Promise.all for parallel; Promise.allSettled for "all of them must run"

```ts
const [user, posts] = await Promise.all([
  fetchUser(id),
  fetchPosts(id),
]);
```

If either fails, the whole thing rejects. That's usually right.

If you want all of them to run regardless of failures:

```ts
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
```

### Avoid serial async

```ts
// BAD — N round trips
for (const id of ids) {
  const user = await fetchUser(id);
  // ...
}

// GOOD — 1 round trip with N parallel requests
const users = await Promise.all(ids.map(fetchUser));
```

---

## 10. Module structure

### Avoid barrel files

A "barrel" is an `index.ts` that re-exports from sibling modules:

```ts
// src/users/index.ts
export * from './repository';
export * from './service';
export * from './types';
```

### Why barrels are usually bad

- **Bundle bloat.** Importing `import { User } from './users'` may pull in the entire `users/` tree.
- **Tree-shaking confusion.** Bundlers handle barrels poorly with side-effecty modules.
- **Cyclic imports.** Barrels reach across siblings; cycles emerge.

### What to do instead

Import from specific files: `import { User } from './users/types'`.

If you want a "public API" for a folder, **explicitly** re-export only the public surface. Don't `export *`.

### ESM `.js` extensions on relative imports

When emitting ESM (`"type": "module"` in `package.json`), Node requires file extensions on relative imports — even from `.ts` source.

```ts
// In TS source:
import { foo } from './bar.js';  // YES — works at runtime
import { foo } from './bar';     // NO — Node fails to resolve
```

`tsx` and `ts-node` are forgiving; production Node is not. Smoke-test the built output before merging.

---

## 11. Inference vs explicit types

### Let TS infer for locals

```ts
const user = await fetchUser(id);  // type is inferred
```

Inference reads fine and stays correct when the source changes.

### Annotate for public APIs

Function parameters, return types of exported functions, types of exported variables.

```ts
export function calculatePrice(item: Item, qty: number): Price { ... }
```

Why: callers should see types in their editor without jumping to the implementation. Inferred return types can change silently when the body changes.

### The `satisfies` operator

For when you want type-checking against a contract but want to keep the narrower inferred type:

```ts
const config = {
  port: 3000,
  host: 'localhost',
} satisfies Config;

config.port;  // inferred as `3000`, not `number`
```

`as Config` would have widened the type. `satisfies Config` validates AND keeps the narrow type.

---

## 12. The `Record<K, V>` trap

`Record<string, User>` looks safe, but with `noUncheckedIndexedAccess`, reading is `User | undefined`. Many people skip the flag to "make this easier," then hit `Cannot read properties of undefined`.

### The fix

Keep the flag on. Handle the undefined case:

```ts
const usersByEmail: Record<string, User> = {};
const user = usersByEmail['a@b.com'];
if (!user) {
  throw new Error('User not found');
}
user.name;  // narrowed to User
```

Or use a `Map`:

```ts
const usersByEmail = new Map<string, User>();
const user = usersByEmail.get('a@b.com');
// same undefined handling, but it's idiomatic
```

---

## 13. Testing patterns

### Test framework: vitest

The de facto modern standard. Faster than jest, easier config, ESM-native.

### Don't share state across tests

(See `claude/testing.md` §6.) Reset state in `beforeEach`.

### Mocking

- **vi.mock** for module-level mocks. Use sparingly.
- **vi.spyOn** for individual functions. Restore in `afterEach`.
- **vi.useFakeTimers / vi.setSystemTime** for time-dependent code.

### Don't run tests in watch mode in CI scripts

`vitest` (no arg) defaults to watch. `vitest run` is one-shot. Confusing this leads to CI builds that hang forever.

### Helpful patterns

- **`it.each([...])`** for parameterized tests.
- **`expect.assertions(N)`** in async tests to ensure all assertions ran.

---

## 14. Common mistakes to refuse

### Stringly-typed status fields

```ts
// BAD
interface Order { status: string; }

// GOOD
type OrderStatus = 'pending' | 'paid' | 'shipped' | 'cancelled';
interface Order { status: OrderStatus; }
```

### Empty interfaces extending `Record<string, unknown>`

```ts
// BAD — defeats the purpose of typing
interface UserPayload extends Record<string, unknown> {}
```

If the payload is open-ended, use `unknown` directly and narrow.

### Type parameters that aren't used

```ts
// BAD — T is unused; this is just (a: string) => string
function identity<T>(a: string): string { return a; }
```

### Importing types alongside values when only types are needed

```ts
// BAD — value-side import; may pull in runtime code
import { User } from './users';

// GOOD when only the type is used
import type { User } from './users';
```

Use `import type` for type-only imports. Bundlers strip these reliably.

---

## 15. Build output: ESM vs CJS

Pick one. Don't ship both unless you're a library author with backward-compat obligations.

- **ESM** is the modern default. Tree-shakes well, ESM-native runtime in current Node.
- **CJS** persists in older codebases and some library ecosystems.

If you're starting fresh: ESM. `"type": "module"` in `package.json`, `target: "ES2022"`, `module: "ESNext"`, `moduleResolution: "Bundler"` or `"NodeNext"`.

If you're shipping a library: dual-emit (`tsup`, `unbuild`) is acceptable. Even then, prefer ESM as the primary surface.
