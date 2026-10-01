---
name: rust-engineering
description: Rust: panics on untrusted input, overflow and `as` casts, Tokio (blocking, locks across await, select! cancellation, timeouts, spawned tasks), Drop, unsafe and FFI, serde, errors, Cargo features. Load it before writing or reviewing any Rust, even a short handler, and when chasing panics, hangs, deadlocks, lost tasks or UB.
license: MIT
metadata:
  author: Alex Tsanis
---

# Rust engineering

The compiler rules out data races and use-after-free in safe code, so the Rust bugs that reach production are the ones it allows on purpose: panics on input the author assumed was valid, arithmetic that wraps in release builds, `as` casts that truncate, locks held longer than the code suggests, futures dropped halfway through a side effect, and `unsafe` blocks whose invariants nobody wrote down. "It compiles" says nothing about any of these. Most entries below look idiomatic in a diff.

## Before you judge or change anything

- Read `Cargo.toml`: `edition`, `rust-version` (MSRV), `[lints]`, and `[profile.release]`. Release builds default to `overflow-checks = false`. With `panic = "abort"`, any panic in any task kills the whole process, which turns every reachable `unwrap` into a crash.
- Runtime: Tokio multi-thread or `current_thread`, and whether code runs under `spawn_local`. Several rules below differ (`block_in_place` panics on `current_thread`; `!Send` futures are legal under `spawn_local`).
- Check a std API against the MSRV before using it:

| API or feature | Stable since |
|---|---|
| `let`-`else` | 1.65 |
| `Duration::try_from_secs_f64` | 1.66 |
| `std::pin::pin!` | 1.68 |
| `OnceLock` | 1.70 |
| `[lints]` table in Cargo.toml | 1.74 |
| `async fn` and `-> impl Trait` in traits | 1.75 |
| `Mutex::clear_poison`, slice `first_chunk`/`split_first_chunk` | 1.77 |
| `LazyLock`, `LazyCell` | 1.80 |
| Edition 2024 (implies resolver 3, MSRV-aware) | 1.85 |
| `str::floor_char_boundary`, `ceil_char_boundary` | 1.91 |

- For a claimed bug, name the input or interleaving that reaches it and check that no parser, type, bound or earlier check already stops it. An `unwrap` on a constant regex or on a value checked two lines up is not a finding. If you cannot name the path, drop the claim.

## Failure catalogue

### Panics and arithmetic on untrusted data

**Panicking on input.** `.unwrap()` on a parse, `buf[4..len]`, `v[idx]`, `map[&key]`, `u32::from_be_bytes(buf[..4].try_into().unwrap())` on a short frame. Each is a remote crash or, under unwind, a failed request that also poisons any `std::sync::Mutex` held at the time (see poisoning below). Use `get`, `first_chunk`/`split_first_chunk` and `?` with a typed error. `unwrap` and `expect` stay only where an invariant makes failure impossible, and `expect` states that invariant. Server crates can enforce this with the Clippy restriction lints `unwrap_used`, `expect_used` and `indexing_slicing`.

**Wrapping arithmetic in release.** Overflow panics in dev and wraps silently in release. `balance + credit`, `unit_price * qty`, `count * size_of::<T>()` computed from input produce a small or negative-looking number instead of an error: an underflowed balance becomes enormous, a wrapped length passes a bounds check. Division by zero and `i64::MIN / -1` panic in every profile.

```rust
// Before: in release, a debit larger than the balance wraps to ~1.8e19
let new_balance = account.balance_minor - debit_minor;

// After
let new_balance = account
    .balance_minor
    .checked_sub(debit_minor)
    .ok_or(LedgerError::InsufficientFunds { account: account.id })?;
```

Use `checked_*` for money, lengths and offsets. `saturating_*` only where clamping is the correct business answer (a retry counter, a metric); for a balance it hides the bug. `wrapping_*` only for hashes, checksums and sequence numbers that are defined to wrap. Turning on `overflow-checks = true` in release is a backstop, not a substitute: a panic is still an outage.

**Lossy `as` casts.** `as` never fails. `-1i64 as u64` is `u64::MAX`; `300u16 as u8` is 44; `len as u32` on a 5 GiB buffer writes the wrong length prefix; `f64 as i64` saturates and maps NaN to 0; `u64 as f64` loses precision above 2^53. Use `u64::try_from(x)`, `u32::try_from(len)` and map the error. The Clippy lints `cast_possible_truncation`, `cast_sign_loss` and `cast_possible_wrap` (pedantic) and `as_conversions` (restriction) find them.

```rust
// Before: a refund row with amount_cents = -500 pays out 18446744073709551116
let payout = row.amount_cents as u64;

// After
let payout = u64::try_from(row.amount_cents)
    .map_err(|_| PayoutError::NegativeAmount { row: row.id })?;
```

**Allocation sized by the sender.** `Vec::with_capacity(n)` or `vec![0; n]` with `n` read from a header or length prefix. Past `isize::MAX` bytes it panics; below that, a failed allocation aborts the process and cannot be caught. `read_to_end` on a socket or upload has no ceiling at all. Cap declared lengths against a configured limit before allocating, and read through `reader.take(limit)`. Axum's `DefaultBodyLimit` (2 MB) covers `Bytes`, `String`, `Json` and `Form` extractors but not a handler that streams `Body` itself.

**Recursion on attacker-shaped data.** A recursive descent parser, a recursive `Drop`, `Debug` or tree walk over nested input overflows the stack, and stack overflow aborts. serde_json has a recursion limit by default; `disable_recursion_limit` removes it and its docs tell you to add `serde_stacker`. Bound depth or iterate with an explicit stack.

**Byte-indexing strings.** `&name[..MAX_LEN]` panics when `MAX_LEN` falls inside a multi-byte character, so a display name with an emoji or accented letter crashes the handler. `len()` counts bytes, not characters. Use `name.floor_char_boundary(max)` (1.91+) or `char_indices` below that MSRV. Clippy `string_slice` flags every string slice.

**Float comparisons and sorting.** `==` on computed floats fails on rounding; NaN is unequal to itself. `sort_by(|a, b| a.partial_cmp(b).unwrap())` panics on the first NaN, and a comparator that is not a total order may make `sort` panic. Use `f64::total_cmp`, reject non-finite values at the boundary, and never hold money in `f64` (integer minor units, or a decimal type with explicit rounding).

**Durations from input.** `Duration::from_secs_f64` panics on negative, NaN or overflowing values, and `Instant + Duration` panics on overflow. Use `Duration::try_from_secs_f64` and `Instant::checked_add` for anything a client or config file supplies.

### Async and Tokio

Patterns, cancellation-safety table and shutdown template: `references/async-tokio.md`.

**Blocking the runtime.** `std::fs`, `std::net`, `std::thread::sleep`, a sync DB or HTTP client, password hashing, compression or large serde work inside `async fn` stalls every task scheduled on that worker: timers fire late, health checks fail, p99 jumps on unrelated endpoints. Blocking I/O goes to `spawn_blocking`; sustained CPU work goes to rayon or a dedicated thread pool, bounded by a semaphore. `spawn_blocking` work cannot be aborted, and runtime shutdown waits for it. `Runtime::block_on` inside an async context panics.

**Lock guard held across `.await`.** A `std::sync::MutexGuard` is `!Send`, so holding it across an await in a spawned task fails to compile, and the usual "fix" is switching to `tokio::sync::Mutex` and keeping the guard across a network call. Now every request queues behind the slowest upstream response. Tokio's own docs prefer the std mutex for plain data: lock, copy or mutate, drop, then await. Clippy `await_holding_lock` catches the std case.

```rust
// Before: every quote request waits for whoever holds the lock to finish an HTTP call
let mut cache = state.rates.lock().await; // tokio::sync::Mutex
let rate = match cache.get(&pair) {
    Some(r) => *r,
    None => {
        let r = state.provider.fetch_rate(&pair).await?;
        cache.insert(pair.clone(), r);
        r
    }
};

// After: std::sync::Mutex, guard never crosses an await
let cached = state.rates.lock().unwrap_or_else(PoisonError::into_inner).get(&pair).copied();
let rate = match cached {
    Some(r) => r,
    None => {
        let r = state.provider.fetch_rate(&pair).await?;
        state.rates.lock().unwrap_or_else(PoisonError::into_inner).insert(pair.clone(), r);
        r
    }
};
```

Two concurrent misses both fetch; that is usually acceptable. When it is not (expensive or rate-limited upstream), use a per-key `OnceCell` or an owner task, not a global lock across I/O.

**Cancellation-unsafe futures in `select!`.** The losing branch is dropped at whatever await it was parked on. `read_exact`, `read_to_end`, `read_to_string` and `write_all` lose partially transferred data; `Mutex::lock`, `Semaphore::acquire` and `Notify::notified` lose their queue position. The expensive case is your own multi-step async fn: dropped after "send payment" and before "record payment", it leaves an unrecorded side effect.

```rust
// Before: shutdown drops process_payout between the provider call and the DB update
tokio::select! {
    _ = shutdown.cancelled() => return Ok(()),
    res = process_payout(&db, &provider, job) => res?,
}

// After: the side-effecting unit runs to completion; shutdown is checked between units
loop {
    let job = tokio::select! {
        biased;
        _ = shutdown.cancelled() => break,
        job = jobs.recv() => match job { // mpsc recv is cancel-safe
            Some(job) => job,
            None => break,
        },
    };
    process_payout(&db, &provider, job).await?;
}
```

If one unit can outlast the shutdown grace period, give its external call a timeout and record the attempt as pending before the call, so a restart reconciles it instead of repeating it.

Also: a `select!` loop that creates `sleep(period)` each iteration restarts the timer whenever another branch fires, so a heartbeat or overall deadline never triggers under steady traffic. Create the `interval` or deadline once, outside the loop (pin a `Sleep` and select on `&mut deadline`). `tokio::time::timeout` cancels by dropping too, so the same rules apply to anything it wraps.

**No timeout.** reqwest has no total, read or connect timeout by default and follows up to 10 redirects, and a raw `TcpStream` read waits forever. Check the default of every client and pool you use, and set timeouts from config. A timeout is an unknown outcome: the server may have applied the write. Retry only with an idempotency key or after reconciling. A timeout also cannot fire while the wrapped future runs CPU work without yielding.

**Unbounded queues.** `mpsc::unbounded_channel`, a growing `Vec` of pending work, or `tokio::spawn` per incoming message lets a fast producer grow memory until the OOM killer arrives. Use `mpsc::channel(capacity)` with capacity from config (it panics on 0): inside a pipeline `send().await` applies backpressure; at the edge, `try_send` and shed load with 429 or 503. A `broadcast` receiver that falls behind gets `RecvError::Lagged(n)` and has lost `n` messages; handle it rather than treating it as closed.

**Detached tasks and lost panics.** `tokio::spawn(work)` with the handle dropped detaches the task: its result and any panic (a `JoinError` nobody awaits) are gone, and runtime shutdown kills it mid-operation. Fan-out goes in a `JoinSet` (dropping it aborts every task) with each `join_next` result checked; long-lived background work goes under a `TaskTracker` with a `CancellationToken`; work that must survive a restart goes to a durable queue. `tokio::spawn` itself panics outside a runtime, for example from a `Drop` during shutdown or in a plain `#[test]`.

**Interval bursts.** `tokio::time::interval` completes its first tick immediately, and its default `MissedTickBehavior::Burst` fires every missed tick back to back after a stall or a slow iteration. A poller that calls a rate-limited API then sends a burst of calls. Set `MissedTickBehavior::Delay` or `Skip`.

**`Send` surprises.** An `Rc`, `RefCell` borrow, `MutexGuard` or other `!Send` value alive across an await makes the whole future `!Send`, and `tokio::spawn` or an axum handler rejects it with a long error. Drop the value in an inner block before the await. Never add `unsafe impl Send` to silence the error. `async fn` in a public trait cannot promise a `Send` future (the compiler warns); use `#[trait_variant::make]` or write `fn f(&self) -> impl Future<Output = T> + Send`. Traits with `async fn` are not dyn-compatible; `Box<dyn Trait>` needs boxed futures.

### Shared state, locks and Drop

**Match scrutinee keeps the lock.** Temporaries in a `match` scrutinee live until the end of the `match`:

```rust
// Deadlocks or panics: the first guard is still alive in the None arm
match cache.lock().unwrap().get(&key) {
    Some(v) => v.clone(),
    None => {
        let v = load(&key)?;
        cache.lock().unwrap().insert(key, v.clone());
        v
    }
}

// Fixed: the guard dies at the end of the let statement
let hit = cache.lock().unwrap().get(&key).cloned();
```

Edition 2021 `if let ... else` behaves the same way (the `else` runs with the guard alive); edition 2024 drops it before `else`. `match` keeps the guard in every edition. Re-locking a std `Mutex` or `RwLock` on the same thread may deadlock or panic. DashMap deadlocks when you call `insert`, `entry` or `remove` while holding a `Ref` into the same map.

**`let _ =` drops immediately.** `let _ = sem.acquire().await?;` releases the permit at the end of the statement, so the concurrency limit does nothing; the same goes for span guards, temp dirs and other RAII values. Bind with a name (`let _permit = ...`). rustc denies the `Mutex` case (`let_underscore_lock`) but not the others.

**Poisoning cascades.** A panic while a `std::sync::Mutex` guard is held poisons it, and every later `.lock().unwrap()` panics too: one bad request takes down every thread touching that state. Decide per lock. If a panic mid-update can leave the data inconsistent, treat poison as fatal and restart the process deliberately. If every mutation is a single assignment, recover with `.unwrap_or_else(PoisonError::into_inner)`. Keep critical sections free of `unwrap` and indexing. `tokio::sync::Mutex` does not poison, so a caught panic leaves half-updated data silently visible.

**`Arc<Mutex<T>>` as the architecture.** Shared mutable state threaded through every component brings contention, lock-ordering deadlocks and poisoning. Before adding one, ask who owns the data. Usually one task should own it and others send commands (an actor with `mpsc` in and `oneshot` replies), or ownership should move through the pipeline, or read-mostly config should be swapped whole behind a `watch` channel.

**Clone-heavy hot paths.** `.clone()` of `String`, `Vec` or large structs per request or per message, often added to satisfy the borrow checker. Borrow, share immutable data as `Arc<str>` or `Bytes`, return `Cow` where most inputs pass through unchanged. Measure before and after (performance-benchmarking).

**No async Drop.** `Drop` cannot await, so cleanup that needs I/O (flush, graceful close, releasing a remote lease, commit) must be an explicit `async fn close(self)` called on every path, with `Drop` as a best-effort fallback. Tokio's `BufWriter` discards its buffer on drop; std's `BufWriter` flushes on drop but ignores the error. Call `flush().await` (and `sync_all` when durability matters) and check the result. Dropping a Tokio `Runtime` inside async code panics; use `shutdown_background`.

**Drop order.** Locals drop in reverse declaration order; struct fields drop in declaration order. When one field depends on another (a client using a runtime, a handle into a C context, a file inside a temp dir), declare the dependent field first or tear down explicitly.

### Unsafe and FFI

Templates, FFI boundary patterns and Miri usage: `references/unsafe-ffi.md`.

**`unsafe` without a stated invariant.** Every `unsafe` block gets a `// SAFETY:` comment naming the precondition and where it is upheld; every `unsafe fn` gets a `# Safety` doc section. Clippy `undocumented_unsafe_blocks` enforces the comment. If you cannot write the comment, the block is probably unsound.

**Invalid values.** `transmute::<u8, bool>` or into an enum, `mem::zeroed()` for references, `NonNull`, `fn` pointers or enums without a zero variant, `MaybeUninit::uninit().assume_init()` (even for integers), `Vec::set_len` over uninitialized elements, `str::from_utf8_unchecked` on bytes from a socket. Each is undefined behavior the moment the value exists. Use `TryFrom` for enums, `from_le_bytes` for numbers, `from_utf8`, and `MaybeUninit` written fully before `assume_init`.

**Raw slices from C.** `slice::from_raw_parts(ptr, len)` requires `ptr` non-null and aligned even when `len == 0`, and C APIs routinely pass `(NULL, 0)`. Check null first and use `&[]`. The returned lifetime is unbounded; tie it to the owner.

**Panics across FFI.** A Rust panic reaching an `extern "C"` boundary aborts the host process; a C++ exception entering Rust through a non-unwind ABI is UB. Wrap every exported function body in `catch_unwind` and return an error code. Use `extern "C-unwind"` only when the other side is written to unwind.

**Mismatched allocators.** Memory from `CString::into_raw` or `Box::into_raw` must come back through `CString::from_raw` or `Box::from_raw`, never C `free()`. Export a paired `_free` function and document it in the header.

**`unsafe impl Send`/`Sync` by assertion.** `Send` claims the value may move between threads; `Sync` claims `&T` may be used from several threads at once. A C handle documented as "usable from any thread, one at a time" is `Send` and not `Sync`. Cite the library's threading contract in the SAFETY comment.

**`static mut`.** A reference to one that overlaps any write is UB, nothing checks that, and edition 2024 denies taking such references. Use atomics, `Mutex`, `OnceLock` or `LazyLock`.

### Serde, errors and logging

Parsing templates for frames, bodies, strings, paths and PATCH payloads: `references/untrusted-input.md`.

**Unknown fields accepted.** Serde ignores unknown keys by default in JSON, so `"requireMfa"` sent to a struct expecting `require_mfa` is dropped silently and the default applies. Privileged and configuration payloads get `#[serde(deny_unknown_fields)]`, which does not work together with `#[serde(flatten)]`.

**Fail-open defaults.** `#[serde(default)]` on a struct fills every missing field from `Default`: a PATCH body `{"session_ttl_secs": 900}` deserialized into the full policy struct turns `require_mfa` off. An absent `Option<T>` field becomes `None`, which code often reads as "no restriction". Model PATCH as a struct of `Option`s applied field by field; security booleans get no default or the restrictive one.

**Untagged enums choosing behavior.** `#[serde(untagged)]` returns the first variant that deserializes, so a permissive variant listed early captures payloads meant for a stricter one. Use internally or adjacently tagged enums when the variant decides what the server does.

**Errors that erase context.** `.map_err(|_| Error::Internal)`, `.ok()?`, `Result<T, String>`, `Box<dyn Error>` built from `format!`. The cause chain is gone and callers cannot match. Libraries expose `thiserror` enums with `#[source]` or `#[from]`; applications add `anyhow` `.context(...)` at each layer and log with `{:#}` or `{:?}` (plain `{}` prints only the outermost message). A blanket `impl From<sqlx::Error> for ApiError` that maps everything to 500 turns "not found" and "unique violation" into retryable server errors; classify at the call site that knows what the error means. Clients get a stable error code, never the inner message.

**Secrets in `Debug` and spans.** `#[derive(Debug)]` on a config or credentials struct prints the secret in every `{:?}` and panic message. `#[tracing::instrument]` records every argument, using `Debug` for non-primitive types, so a password or token parameter lands in the logs. Use `skip(...)` or `skip_all` plus explicit `fields(...)`, and wrap secrets in a type whose `Debug` redacts. Compare MACs and tokens in constant time (`subtle`).

**Hasher and iteration-order assumptions.** std `HashMap` uses randomly seeded SipHash-1-3 and resists HashDoS; a map keyed by user input is fine as is. Swapping in FxHash or another unkeyed fast hasher for attacker-chosen keys brings collision floods back. Iteration order is arbitrary and differs per process, so hashing, signing or caching a serialized `HashMap`, snapshot tests, and "take the first entry" logic break across instances. Use `BTreeMap`, sort, or `IndexMap` when order matters.

**Regex false alarm.** The `regex` crate searches in time linear in pattern size times input size, so nested quantifiers such as `^([a-z0-9]+[-.]?)*$` on user input are not ReDoS. Compiling a user-supplied pattern still needs `RegexBuilder::size_limit`, and `fancy-regex` backtracks.

### Cargo, features and supply chain

Lint config, CI commands and `deny.toml`: `references/cargo-and-supply-chain.md`.

**Untested feature combinations.** Cargo builds each dependency with the union of features requested anywhere in the build, and building several workspace members together unifies their dependencies' features. A crate that forgot to enable `tokio/fs` compiles in `cargo build --workspace` because another member enables it, then fails for `cargo build -p crate` or a downstream user. Features must be additive. `--all-features` alone proves little; run `cargo hack check --each-feature` (or `--feature-powerset --depth 2`), which checks each package separately and includes `--no-default-features`.

**Build scripts and proc macros run as you.** They execute at build time on developer machines and CI runners with whatever credentials are present. Read the `build.rs` of a new dependency, keep deploy secrets out of build jobs, restrict sources and build scripts with `cargo deny` (`[sources]`, `[bans.build] allow-build-scripts`), and check advisories with `cargo deny check advisories` or `cargo audit`. CI builds with `--locked`; `cargo install` ignores the package's lock file unless given `--locked`.

## Decision rules

- Integer operations on input: `checked_*` returning an error. `saturating_*` only when clamping is the right answer. `wrapping_*` only when wrapping is the definition. Conversions: `TryFrom`, never `as` on values that can be out of range.
- Money: integer minor units in a newtype whose arithmetic is checked; a decimal type when rates or division are involved. Never `f64`.
- `unwrap`/`expect`: only on invariants the code itself establishes (constant patterns, a value checked just above, a type that cannot be empty). Never on input, I/O, parsing, env or config in a server path.
- Mutex choice: std `Mutex` for data with short critical sections that never await; `tokio::sync::Mutex` only when the guard must span an await (serializing an I/O resource), with a timeout around the work; an owner task with channels when many tasks mutate the same state or the state is an I/O handle; `watch` or an atomic swap for read-mostly config.
- Blocking work: `spawn_blocking` for short blocking I/O; rayon or a dedicated pool bounded by a semaphore for CPU work; a dedicated `std::thread` for a long-lived blocking loop.
- Spawned work: `JoinSet` when the caller waits for the results; `TaskTracker` plus `CancellationToken` for background tasks that must drain on shutdown; a durable queue when the work must survive a crash or deploy.
- Channels: bounded with capacity from config; `send().await` inside a pipeline, `try_send` plus load shedding at the edge.
- Errors: `thiserror` enums in libraries and in modules whose callers branch on the error; `anyhow` with context in binaries at the top.
- Poisoned lock: recover with `into_inner` only when a panic cannot leave the data half-updated; otherwise fail the process.
- `async fn` in traits: native for static dispatch; boxed futures (the `async-trait` crate) when you need `dyn Trait`; `trait_variant` or explicit `+ Send` for public traits used on multi-threaded runtimes.

## Review checklist

- Can any input reach `unwrap`, `expect`, indexing, slicing or a panicking std function (`from_secs_f64`, `Instant + Duration`, division)?
- Is every arithmetic operation on amounts, lengths and offsets checked, and is every `as` on a value that can be out of range replaced with `TryFrom`?
- Is every allocation, read and recursion depth driven by input capped by a configured limit?
- Does any string slice use a byte index that could fall inside a character?
- Is any float compared with `==`, sorted with `partial_cmp().unwrap()`, or used for money?
- Inside `async fn`, is there any blocking I/O, sleep, sync client or heavy CPU work?
- Does any lock guard (std or Tokio) live across an `.await`, or inside a `match` or 2021 `if let` scrutinee that locks again?
- Is every `select!` branch cancel-safe, and does any branch drop a multi-step side effect halfway?
- Does every network call, pool acquire and external wait have a timeout from config, and is a timeout treated as an unknown outcome?
- Is every channel bounded, and are `broadcast` lag errors handled?
- Is every spawned task owned (`JoinSet`, `TaskTracker`, awaited handle), with panics and errors observed and shutdown propagated?
- Does any `let _ =` drop a permit, guard or span that should live to the end of scope?
- Is each lock's poisoning behavior a deliberate choice, with no panicking code inside critical sections?
- Is I/O cleanup explicit (flush, close, `sync_all`) rather than left to `Drop`?
- Does every `unsafe` block have a SAFETY comment that names a real, upheld invariant, and are FFI exports wrapped in `catch_unwind` with null and length checks?
- Do privileged serde types deny unknown fields, avoid fail-open defaults, and avoid `untagged` for behavior selection?
- Do errors keep their source chain, and do clients see only stable codes?
- Can any secret reach a `Debug` impl, a panic message or a `#[instrument]` span?
- Does anything depend on `HashMap` iteration order, or use an unkeyed hasher on attacker keys?
- Do CI jobs build each package and feature combination with `--locked`, run Clippy with warnings denied, and run `cargo deny` or `cargo audit`?

## References

- `references/async-tokio.md`: read when writing or reviewing Tokio code: select loops, cancellation safety, timeouts, task supervision, graceful shutdown, channels, blocking work, actors.
- `references/unsafe-ffi.md`: read before writing or reviewing `unsafe`, FFI exports or imports, raw pointers, `MaybeUninit`, or when running Miri and sanitizers.
- `references/untrusted-input.md`: read when parsing bytes, request bodies, serde payloads, strings, numbers, file paths or regexes that come from outside the process.
- `references/cargo-and-supply-chain.md`: read when setting up lints, CI, feature flags, MSRV, `cargo deny`, or adding a dependency with a build script or proc macro.

Related skills: security-engineering for authorization and threat modelling; backend-architecture for idempotency, retries and reconciliation; database-engineering for transactions and locking; test-engineering for test strategy and fuzzing plans; performance-benchmarking before optimizing.
