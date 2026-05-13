# Rust — ownership, errors, async, type-driven design

Rust rewards clear thinking about ownership and lifetimes. Fighting the borrow checker usually means the data model is wrong; listen to it.

This file covers the patterns that distinguish idiomatic Rust from "Java with semicolons": ownership-aware design, error handling as a contract, async without `Arc<Mutex>` reflex, and using the type system to make invalid states unrepresentable.

For language-agnostic principles, see `claude/principles.md`. For testing in Rust, see `claude/testing.md`.

---

## 1. Let ownership guide design

**RULE:** If you're fighting the borrow checker, the data model is wrong. Don't reach for `Clone` or `Arc<Mutex<_>>` as a reflex.

### What "fighting the borrow checker" means

- You're cloning to avoid a borrow conflict.
- You're wrapping in `Arc<Mutex<_>>` to share state.
- You're adding lifetimes to make a function "just work."
- You can't articulate **who owns** a piece of data.

These are symptoms. The cause is usually: the same data is being viewed as both owned and shared, or as both mutable and immutable, without a clear boundary.

### Patterns that resolve it

- **Pass by value** when ownership is transferred. The recipient takes responsibility.
- **Pass by `&T`** when you need read-only access for a defined scope.
- **Pass by `&mut T`** when you need exclusive write access for a defined scope.
- **Restructure the data** so that one owner is clear. A `Vec<User>` owned by `App` is cleaner than every component holding `Arc<RwLock<User>>`.
- **Channels** when threads need to coordinate. `tokio::sync::mpsc` or `crossbeam::channel` is often cleaner than shared mutable state.

### When `Arc` is actually right

- True shared ownership where lifetimes can't be statically tracked (e.g., a connection pool used by N tasks).
- Cross-thread sharing where channels don't fit.
- Reference-counted immutable data (e.g., `Arc<str>` for cheap clones of a string).

### When `Arc<Mutex<T>>` is rarely right

- Almost never the first move. Try channels first. Try restructuring first.
- If you need it, prefer `Arc<RwLock<T>>` if reads dominate.
- If you need it, document why (the comment proves you've thought about it).

---

## 2. The newtype pattern

**RULE:** Wrap primitive types in newtypes for domain identifiers and quantities.

### The problem with raw primitives

```rust
fn transfer(from: u64, to: u64, amount: u64) { ... }

transfer(account_a, amount, account_b);  // compiles, ships, breaks
```

### The fix

```rust
struct AccountId(u64);
struct Amount(u64);

fn transfer(from: AccountId, to: AccountId, amount: Amount) { ... }

transfer(account_a, amount, account_b);  // compile error
```

### Why this is zero-cost

The wrapper struct compiles to the same bytes as the inner type. There's no runtime overhead. You get compile-time safety for free.

### When to use it

- Domain identifiers (`UserId`, `OrderId`, `TenantId`)
- Quantities with units (`Cents`, `Milliseconds`, `Bytes`)
- Validated strings (`Email`, `Url`) — combine with constructor functions that validate

### What to add

- `#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]` for most ID-like types.
- A constructor that validates: `Email::new(s: &str) -> Result<Self, EmailError>`.
- Don't expose the inner type publicly unless you must.

---

## 3. Make invalid states unrepresentable

**RULE:** Use the type system to forbid invalid combinations. If the compiler accepts it, it must be valid.

### The type-state pattern

A type carries its state in its type parameter:

```rust
struct Connection<State> { /* ... */ _state: PhantomData<State> }
struct Disconnected;
struct Connected;
struct Authenticated;

impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connected> { ... }
}

impl Connection<Connected> {
    fn authenticate(self, creds: Credentials) -> Connection<Authenticated> { ... }
}

impl Connection<Authenticated> {
    fn fetch(&self, query: Query) -> Result<Data, Error> { ... }
}
```

Now `fetch` can only be called on an authenticated connection. The compiler enforces the protocol.

### Discriminated enums for variants

```rust
enum PaymentStatus {
    Pending { initiated_at: SystemTime },
    Completed { transaction_id: String, completed_at: SystemTime },
    Failed { reason: String, attempted_at: SystemTime },
}
```

Each variant carries only the data relevant to its state. Pattern matching is exhaustive — adding a variant breaks every match, forcing you to handle it.

### Avoid optional fields that imply state

```rust
// BAD — Pending must have no transaction_id; Completed must have one
struct Payment {
    status: String,
    transaction_id: Option<String>,
    failure_reason: Option<String>,
}
```

Refactor into a discriminated enum like above.

---

## 4. Error handling: `thiserror` for libraries, `anyhow` for applications

**RULE:** Libraries define typed errors with `thiserror`. Applications use `anyhow` for ergonomic propagation. Don't mix them.

### Libraries

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum UserError {
    #[error("user not found: {0}")]
    NotFound(UserId),

    #[error("invalid email: {0}")]
    InvalidEmail(String),

    #[error("database error")]
    Database(#[from] sqlx::Error),
}
```

- Callers can match on specific variants.
- The error type is part of your API contract.
- Sources are preserved via `#[from]` and `#[source]`.

### Applications

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let config = load_config().context("loading config")?;
    let db = connect(&config.database_url).context("connecting to database")?;
    serve(db).await.context("running server")?;
    Ok(())
}
```

- No need to enumerate every variant.
- `.context("doing X")` adds breadcrumbs to error messages.
- Sources chain automatically.

### The rule

If your code might be consumed by another binary, define typed errors. If your code is the binary, `anyhow` is fine.

### Never

- `unwrap()` in production paths. Use `expect("invariant: <why>")` if you genuinely have an unreachable case — the message documents what would have to go wrong.
- Bare `panic!` in libraries. Return `Result`.
- Errors that say nothing: `Err("error".to_string())` tells a debugger nothing. Be specific.

---

## 5. `Result` and `?`

The `?` operator propagates errors. Use it relentlessly.

```rust
fn process(path: &Path) -> Result<Summary, ProcessError> {
    let bytes = fs::read(path)?;
    let parsed = parse(&bytes)?;
    let validated = validate(parsed)?;
    Ok(summarize(validated))
}
```

### Don't write match for the common case

```rust
// VERBOSE
let parsed = match parse(&bytes) {
    Ok(p) => p,
    Err(e) => return Err(e.into()),
};

// IDIOMATIC
let parsed = parse(&bytes)?;
```

### Map errors when types don't match

```rust
let parsed = parse(&bytes).map_err(ProcessError::Parse)?;
```

Or use `#[from]` on your error enum to make the conversion implicit.

### Don't swallow errors

```rust
// BAD — silently ignores parse failures
let parsed = parse(&bytes).unwrap_or_default();
```

If the failure is genuinely OK to ignore, log it. If it's not OK to ignore, propagate.

---

## 6. `From` and `Into` for conversions

**RULE:** Use `From<T>` to convert between types. Never use `as` casts on integer types in domain code (overflow danger).

### Pattern

```rust
impl From<UserRow> for User {
    fn from(row: UserRow) -> Self {
        User {
            id: UserId(row.id),
            email: row.email,
            created_at: row.created_at,
        }
    }
}

// At the call site:
let user: User = row.into();
```

### Why not `as`?

`as` silently truncates: `1_000_000_u32 as u8 == 64`. For domain types, use `TryFrom`:

```rust
let small: u8 = u8::try_from(large_u32).map_err(|_| Error::TooLarge)?;
```

For loss-tolerant casts (e.g., `usize as u64`), `as` is acceptable. But always pause and think before using it.

---

## 7. Async without reflexive `Arc<Mutex>`

Async Rust is unforgiving with shared mutable state. The instinct to reach for `Arc<Mutex<_>>` is usually wrong.

### Patterns to try first

#### Pass owned data into spawned tasks

```rust
let data = compute_initial_state();
tokio::spawn(async move {
    process(data).await
});
```

The task owns its data. No sharing needed.

#### Channels for coordination

```rust
let (tx, mut rx) = tokio::sync::mpsc::channel(100);

tokio::spawn(async move {
    while let Some(msg) = rx.recv().await {
        handle(msg).await;
    }
});

tx.send(message).await?;
```

One task owns the state; others send messages.

#### `RwLock` if reads dominate

```rust
let cache = Arc::new(tokio::sync::RwLock::new(HashMap::new()));
```

`RwLock` allows many concurrent reads. Use only when channels don't fit.

### When `Mutex` is genuinely right

- A small, well-defined critical section.
- Reads and writes are balanced.
- The contention is rare or the held duration is short.

If you find yourself holding a lock across an `.await`, that's a sign to redesign. The lock blocks every other task that wants it.

### Send + Sync gotchas

Async tasks require `Send + 'static` bounds. If your error type isn't `Send`, async propagation fails confusingly. `thiserror`-derived errors are `Send + Sync` by default — keep them that way.

---

## 8. Traits: dyn vs generics

### Generics: zero-cost, monomorphized

```rust
fn process<S: Storage>(storage: S, item: Item) { ... }
```

The compiler generates a separate copy of `process` for each `S`. Fast at runtime, slow to compile, code bloat.

### `dyn Trait`: trait objects, dynamic dispatch

```rust
fn process(storage: Box<dyn Storage>, item: Item) { ... }
```

One copy of `process` for all `Storage` implementations. Smaller binary, slightly slower (vtable lookup), but in practice the cost is negligible.

### When to use which

- **Generics** for hot paths, library code, or when you want zero overhead and the caller is known.
- **`dyn`** when the type is determined at runtime (e.g., plugins, dynamically-loaded handlers), or when generic bloat is a problem.

In application code, `dyn` is almost always fine. In library code, generics are usually preferred. **Don't overthink it.**

### Object-safe traits

Not every trait can be used as `dyn`. Avoid generic methods on a trait you intend to use as a trait object:

```rust
// NOT object-safe
trait Storage {
    fn get<T: DeserializeOwned>(&self, key: &str) -> Option<T>;
}
```

If you need both, define a base trait that's object-safe and an extension trait with generic methods.

---

## 9. The builder pattern

For types with many optional construction parameters, use a builder.

```rust
#[derive(Default)]
struct ClientBuilder {
    timeout: Option<Duration>,
    base_url: Option<Url>,
    retry_count: Option<u32>,
}

impl ClientBuilder {
    pub fn timeout(mut self, t: Duration) -> Self {
        self.timeout = Some(t);
        self
    }

    pub fn base_url(mut self, u: Url) -> Self {
        self.base_url = Some(u);
        self
    }

    pub fn build(self) -> Result<Client, BuildError> {
        Ok(Client {
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
            base_url: self.base_url.ok_or(BuildError::MissingBaseUrl)?,
            retry_count: self.retry_count.unwrap_or(3),
        })
    }
}
```

### Usage

```rust
let client = ClientBuilder::default()
    .base_url(url)
    .timeout(Duration::from_secs(10))
    .build()?;
```

### When to use

- 3+ optional parameters.
- Construction with validation.
- When the type's fields shouldn't be publicly accessible.

### Crates that help

- `bon` — derive-based builder generation.
- `typed-builder` — compile-time required-field enforcement.

---

## 10. Cargo workspace organization

For projects with multiple crates:

```
my-project/
├── Cargo.toml          # workspace manifest
├── crates/
│   ├── api/            # HTTP layer
│   ├── core/           # business logic
│   ├── storage/        # database
│   └── types/          # shared types
```

### Workspace `Cargo.toml`

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

### Per-crate `Cargo.toml`

```toml
[dependencies]
serde = { workspace = true }
tokio = { workspace = true }
```

Centralizes version management. Avoids version drift.

### Crate naming

- Use the workspace as a prefix: `myproject-api`, `myproject-core`.
- Or use `crates.io`-style two-letter prefixes for shorter names.
- Don't name crates after generic technical concepts (`utils`, `common`, `lib`).

---

## 11. `default-features = false`

When depending on large crates, opt out of unused features:

```toml
[dependencies]
tokio = { version = "1", features = ["macros", "rt-multi-thread"], default-features = false }
```

Why:
- Faster compile times.
- Smaller binaries.
- Smaller attack surface.

Audit your dependencies' feature flags. Many crates pull in everything by default.

---

## 12. Testing patterns

### `#[cfg(test)]` module at the bottom of each file

```rust
pub fn parse_query(input: &str) -> Result<Query, ParseError> { ... }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_simple_query() { ... }

    #[test]
    fn rejects_empty_input() { ... }
}
```

Unit tests live next to the code. They can access private items (within the same file).

### Integration tests in `tests/`

```
my-crate/
├── src/
│   └── lib.rs
└── tests/
    ├── api.rs
    └── db.rs
```

Each file in `tests/` is a separate test binary. Tests run against the crate's public API.

### Doc tests

```rust
/// Parses a query string into a structured query.
///
/// ```
/// use mycrate::parse_query;
///
/// let q = parse_query("color:red").unwrap();
/// assert_eq!(q.terms[0].field, "color");
/// ```
pub fn parse_query(input: &str) -> Result<Query, ParseError> { ... }
```

Doc tests are real tests. They verify the example in the docs still compiles and runs. Use them on public API.

### What to test

(See `claude/testing.md` for the broader testing discipline.) Rust-specific:

- Property-based testing with `proptest` for parsers, encoders, and pure logic.
- `criterion` for benchmarks (when you have a number to optimize).

---

## 13. Clippy

**RULE:** `cargo clippy -- -D warnings` runs clean. No `#[allow(...)]` without a comment explaining why.

### Configuration

```toml
# .cargo/config.toml or Cargo.toml [lints]
[lints.clippy]
all = "warn"
pedantic = "warn"
```

Then in CI:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

### `#[allow]` rules

- Always include a comment: `#[allow(clippy::too_many_arguments)] // builder pattern not applicable here`.
- Prefer fixing the lint to allowing it.
- Project-wide `#[allow]` at the crate root is acceptable for genuinely-misfiring lints in your codebase. Document the reason.

---

## 14. `rustfmt`

**RULE:** `cargo fmt` runs clean before every commit. CI verifies.

Use the default settings. Bikeshedding over rustfmt config is a waste of time the team could spend on actual work.

If you want consistency on `imports_granularity` or `group_imports`, set it in `rustfmt.toml`. Otherwise, defaults.

---

## 15. Async runtime: tokio is the default

Pick **tokio** unless you have a specific reason not to. It's the de facto async runtime; most async libraries assume it.

### Common mistakes

- **Blocking inside async.** `std::fs::read` blocks the executor. Use `tokio::fs::read`.
- **Spawning unnecessarily.** Spawning a task has overhead. Use `tokio::join!` for parallel awaits within a single task.
- **`block_on` inside async.** Will deadlock the runtime. Just `.await`.
- **Forgetting `Send` bounds.** Async tasks that hold non-Send types can't be spawned. Common offender: `Rc` (use `Arc`), `RefCell` (use `Mutex`).

---

## 16. Common mistakes to refuse

### Reflexive cloning

```rust
// BAD
fn process(s: &str) -> String {
    let owned = s.to_string();
    let result = transform(owned.clone());
    result
}
```

Clone only when ownership transfer is needed. Often it isn't.

### `unwrap` in production paths

Production code does not `.unwrap()` on operations that can fail. Use `?` to propagate, or `.expect("invariant: <why>")` if you've proved it can't fail.

### Stringly-typed errors

```rust
// BAD
fn process() -> Result<(), String> {
    Err("something went wrong".to_string())
}

// GOOD
#[derive(Debug, Error)]
enum ProcessError { ... }
```

### Long match arms with duplicated logic

If multiple match arms do the same thing, the match isn't pulling its weight. Extract a function or refactor the data.

### `impl Trait` in return type when generics would be clearer

`impl Trait` is great when the type is genuinely complex (combinator chains, closures). For simple cases, the explicit type is more readable.
