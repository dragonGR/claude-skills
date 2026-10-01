---
name: python-engineering
description: Python services, workers and scripts: asyncio, exceptions, HTTP clients, Decimal money, datetimes, Pydantic, SQLAlchemy, subprocess, paths, dependencies and pytest. Load it before writing or reviewing any Python beyond a one-liner, including a quick review of a snippet, and when debugging stalls, hangs, leaks, flaky tests or wrong totals.
license: MIT
metadata:
  author: Alex Tsanis
---

# Python engineering

Python fails quietly. A default argument shared by every call, a task garbage-collected mid-flight, a float that is almost a price, a naive datetime that is almost UTC: each one runs, passes the happy-path test and corrupts data in production. Most of the entries below are code a reviewer reads past because it looks idiomatic.

## Before you judge or change anything

- Python version: `requires-python`, the CI matrix, the Docker base image. Several behaviors below changed in 3.11, 3.12 or 3.14. Do not use syntax or stdlib features newer than the declared minimum.
- Sync or async, and the server. FastAPI runs `async def` routes and dependencies on the event loop and plain `def` ones in a threadpool, so the same blocking call is harmless in one and stalls the worker in the other.
- Process model: worker count, threads per worker, whether the server imports the app before forking (gunicorn `--preload`, Celery prefork, `multiprocessing` with fork).
- Major versions of Pydantic (v1, v2, or v2 with `pydantic.v1` imports) and SQLAlchemy (1.x `Query` or 2.0 `select`). Advice for one is wrong for the other.
- For a claimed bug: the input or interleaving that triggers it, and whether a validator, type, lock or later check already stops it. If you cannot name the path, drop the claim.

## Failure catalogue

### Language traps

**Mutable default argument.** `def f(items=[])` or `opts={}`: the default is built once, at definition, and every call shares it. In a web worker or batch job, data from one customer shows up in the next. Default to `None` and create inside, or use an immutable default (`()`, `frozenset()`).

```python
# Before: the second caller's digest also contains the first caller's alerts
def build_digest(user: User, alerts: list[Alert] = []) -> list[Alert]:
    alerts.extend(pending_alerts(user))
    return alerts

# After
def build_digest(user: User, alerts: list[Alert] | None = None) -> list[Alert]:
    alerts = [] if alerts is None else list(alerts)
    alerts.extend(pending_alerts(user))
    return alerts
```

The same sharing happens with class attributes (`class Cart: items = []`). `@dataclass` rejects unhashable defaults (since 3.11, any unhashable value, not only `list`/`dict`/`set`), but it accepts an instance of an ordinary class, which is hashable by identity and therefore shared; use `field(default_factory=...)`. Pydantic deep-copies non-hashable defaults per instance, so `tags: list[str] = []` on a Pydantic model is fine and not a finding.

**Late-binding closures.** A lambda or nested function created in a loop looks up the loop variable when it runs, not when it was created, so every callback sees the last value. Bind at creation: `lambda amount, rate=rate: amount * rate`, `functools.partial(apply_rate, rate)`, or a factory function.

**Consumed iterators.** Generators, `map`, `filter`, `zip` and file objects can be read once. `if not any(gen): return` followed by `for x in gen` silently drops the element `any` consumed. A generator passed to two functions gives the second nothing. `bool(gen)` is always true, so `if gen:` is not an emptiness check. Materialize with `list()` when you read twice or test for emptiness.

**Mutating while iterating.** Removing from a list inside `for x in items` skips the element after each removal without any error. Changing a dict's size during iteration raises `RuntimeError`. Iterate over a copy or build a new collection.

**`assert` as a guard.** `python -O` strips asserts. `assert order.owner_id == user.id` is not an authorization check. Raise explicitly for anything guarding authority, money or data shape.

**Caches that leak or cross users.** `@lru_cache` on a method keeps every `self` alive until evicted. `@cache` (unbounded) keyed on request input grows until the worker is OOM-killed. A cached function whose result depends on the caller (tenant, locale, permissions) but whose arguments omit it serves one user's data to another.

### Errors

**Bare `except:` or `except BaseException`.** Catches `KeyboardInterrupt`, `SystemExit` and `asyncio.CancelledError`. A retry loop wrapped this way keeps retrying after shutdown or a timeout cancelled it, which is how deploys hang. `except Exception` does not catch `CancelledError` (a `BaseException` subclass since 3.8), so at a boundary it is the correct broad catch, not a finding.

**Swallowing at the wrong layer.** `except Exception: return None` or `pass` deep in the stack turns a failure into a plausible value: a missing price becomes zero, a failed write reports success. Catch the narrowest exception you can act on, where you can act on it. Broad catches belong at the request, job or task boundary, where they log with the traceback and decide the response or retry.

**Lost tracebacks.** `log.error(f"failed: {e}")` records one line and no stack. Inside `except`, use `log.exception(...)` or `exc_info=True`. When translating, `raise PaymentFailed(order_id) from err` keeps the cause; `from None` hides it and belongs only where the cause would leak something. Returning `str(err)` to a client leaks SQL, paths and hostnames: send a stable error code, log the detail.

**`except SomeError` around a `TaskGroup`.** A `TaskGroup` raises `ExceptionGroup`, so `except httpx.TimeoutException` around it never matches. Use `except*` (3.11+) or handle errors inside each task.

### Async

**Blocking the event loop.** `requests`, `time.sleep`, sync DB drivers (psycopg2, PyMySQL, a sync SQLAlchemy `Session`), `boto3`, password hashing, large file reads or `json.loads` of large bodies inside `async def` freeze every other request on that worker. The symptom is p99 spikes on unrelated endpoints and failing health checks under load. Use the async client (`httpx.AsyncClient`, asyncpg or psycopg 3 async, `AsyncSession`), `await asyncio.sleep`, or `await asyncio.to_thread(fn, ...)` for blocking calls with no async equivalent. CPU-bound work goes to a process pool. A FastAPI route that only calls blocking libraries can simply be `def`. `PYTHONASYNCIODEBUG=1` logs callbacks slower than 100 ms.

**Fire-and-forget tasks.** The loop keeps only weak references to tasks. `asyncio.create_task(send_receipt(order))` with the result discarded can be garbage-collected before it finishes, and its exception appears at most as "Task exception was never retrieved". Keep tasks in a set with `add_done_callback(tasks.discard)` and a callback that logs failures, cancel and await them on shutdown, or use a `TaskGroup` when the caller can wait. Work that must survive a crash or deploy goes to a durable queue, not a task.

**Swallowed cancellation.** `except asyncio.CancelledError: pass`, bare `except:` or `except BaseException` in a coroutine absorbs cancellation, which breaks `asyncio.timeout`, `TaskGroup` and graceful shutdown. Clean up in `finally`, or catch, clean up and re-raise.

**Misreading `gather`.** With the default `return_exceptions=False`, the first exception propagates to the awaiter and the other awaitables keep running unsupervised; cancelling the `gather` afterwards does not stop them. With `return_exceptions=True`, exceptions come back as list items and vanish unless every result is checked. Use a `TaskGroup` when siblings should be cancelled on first failure; use `gather(..., return_exceptions=True)` only when you inspect each result.

**Unbounded concurrency.** `gather(*(fetch(u) for u in urls))` over a data-sized list starts every request at once. With a pooled client (httpx allows 100 connections by default) the excess waits for a connection until the pool timeout fails it; without a pool limit it exhausts file descriptors. Either way it trips the remote side's rate limits.

```python
# Before: a 20,000-row import means 20,000 simultaneous requests
await asyncio.gather(*(enrich(http, row) for row in rows))

# After: the limit comes from config; enrich() handles its own per-row errors,
# because one failure inside a TaskGroup cancels the siblings
sem = asyncio.Semaphore(settings.enrich_concurrency)

async def enrich_bounded(row: ImportRow) -> None:
    async with sem:
        await enrich(http, row)

async with asyncio.TaskGroup() as tg:
    for row in rows:
        tg.create_task(enrich_bounded(row))
```

For inputs too large to hold as tasks, use a fixed pool of workers reading a bounded `asyncio.Queue` (`references/async.md`).

**No deadline.** An `await` on a network read with no timeout can wait forever while holding a semaphore slot or pool connection. Wrap with `async with asyncio.timeout(seconds)` (3.11+) or `asyncio.wait_for`; both cancel the inner work and raise the builtin `TimeoutError` on 3.11+ (`asyncio.TimeoutError` before). A timeout is an unknown outcome: the remote side may have applied the write.

**`threading.local` in async code.** Every coroutine on the loop thread shares it, so a request id, tenant or DB session stored there leaks between concurrent requests. Use a module-level `contextvars.ContextVar`.

**Touching asyncio from other threads.** Almost no asyncio object is thread-safe. From a thread, use `loop.call_soon_threadsafe` or `asyncio.run_coroutine_threadsafe`.

### HTTP and other external calls

**No timeout.** `requests` never times out unless `timeout=` is passed; a hung upstream holds a worker thread indefinitely. Its read timeout is the gap between bytes, not a total deadline. httpx defaults to 5 seconds of inactivity, which may be wrong in either direction. `subprocess.run` has no default timeout. Set connect and read timeouts from config on the client or every call.

**A new client per call.** Top-level `requests.post(...)`, `httpx.AsyncClient()` inside a handler or `boto3.client()` per request pays a TCP and TLS handshake each time, leaks sockets when not closed, and defeats keep-alive. One `requests.Session` or `httpx.Client` per process (in ASGI apps, per lifespan), closed on shutdown, passed in.

**Retrying non-idempotent calls.** A retry decorator or urllib3 `Retry` with `POST` in `allowed_methods` around a charge, transfer or email duplicates it when the first attempt succeeded and only the response was lost. Retry only idempotent requests or ones carrying an idempotency key the server honors; cap attempts, back off with jitter, and do not retry 4xx other than 408 and 429.

**`verify=False`.** Accepts any certificate, so any network position can read and alter the traffic. Point `verify` or `REQUESTS_CA_BUNDLE` at the right CA bundle instead.

### Numbers and time

**Float money.** Summed float prices drift by cents and reconciliation fails. Money is `Decimal` or integer minor units end to end: accept it from JSON as a string or integer, store `numeric` or `bigint`, never pass through `float`. A Pydantic field typed `float` for a price is the bug even if later code converts it.

**`Decimal(float)`.** `Decimal(19.99)` is `Decimal('19.989999999999998436805981327779591083526611328125')`; the damage happened before `Decimal` saw it. Construct from `str` or `int`. In money code, trap `decimal.FloatOperation` so `Decimal(3.14)` and `Decimal < float` raise instead of silently mixing.

**Implicit or leaked rounding context.** The default context is 28 significant digits with `ROUND_HALF_EVEN`, and nothing rounds until you `quantize`. Quantize at defined points with the rounding mode the business rule names, and allocate remainders so split amounts sum to the total. `getcontext()` returns a context object that is mutated in place and shared with later work on the same thread and with asyncio tasks that inherited it, so `getcontext().rounding = ...` in one request can change rounding for others; use `decimal.localcontext()` or pass `rounding=` to `quantize`. Builtin `round()` also rounds halves to even and works on the binary value: `round(2.675, 2) == 2.67`.

**Naive datetimes.** `datetime.now()` is naive local time. `datetime.utcnow()` and `utcfromtimestamp()` (deprecated since 3.12) return naive values that anything interpreting them (`.timestamp()`, `.astimezone()`) treats as local. `datetime.fromtimestamp(ts).replace(tzinfo=timezone.utc)` relabels local time as UTC without converting, off by the host's offset. Ordering naive against aware raises `TypeError`; `==` is simply always false, so a dedupe or cache check never matches. Use `datetime.now(timezone.utc)`, `datetime.fromtimestamp(ts, tz=timezone.utc)`, `zoneinfo` for named zones, and reject naive input at the boundary (Pydantic `AwareDatetime`).

**Wall clock for durations.** `time.time()` jumps with NTP corrections. Timeouts, rate limiters and latency measurements use `time.monotonic()` or `time.perf_counter()`.

### Security at the boundary

**Code-executing deserializers.** `pickle` and anything built on it (`shelve`, `joblib`, `pandas.read_pickle`), `yaml.load` with `Loader=yaml.Loader` or `UnsafeLoader`, `eval`/`exec` on input. A pickled value in Redis or on a shared volume is code execution for anyone who can write there. Use JSON, `yaml.safe_load`, or sign with `hmac` and verify before loading.

**Shell commands from strings.** `subprocess.run(f"convert {name} out.png", shell=True)`, `os.system`, `os.popen`. Pass an argument list with the default `shell=False`, put `--` before user-supplied positional arguments so `-rf` is not read as an option, and set `timeout=` and `check=True` (the default `check=False` ignores non-zero exits).

**SQL built with strings.** f-strings, `%`, `.format()` or `+` into `cursor.execute`, SQLAlchemy `text()`, `order_by(text(...))`. Bind values (`%s` or `?` per driver, `:name` with `text()`). Identifiers cannot be bound: map request values through an allowlist dict to fixed SQL fragments. An f-string that interpolates only a value selected from such an allowlist is safe; do not report it.

**Path traversal.** `os.path.join(base, name)` and `Path(base) / name` escape with `../` and, when `name` is absolute, discard `base` entirely. Resolve, then check containment; better, never use client-supplied names as paths and look files up by an id with an ownership check.

```python
def safe_child(base: Path, name: str) -> Path:
    root = base.resolve()
    target = (root / name).resolve()
    if not target.is_relative_to(root):
        raise PermissionError(name)
    return target
```

**Archive extraction.** Before 3.14, `tarfile` extraction trusts member paths and links by default. Pass `filter="data"` (3.12+, backported to some older security releases) and cap member count and total uncompressed size for any archive, zip included.

**Predictable temp paths.** `tempfile.mktemp()` or a fixed `/tmp/report.csv` lets another local process create or symlink the path first, and makes concurrent workers overwrite each other. Use `NamedTemporaryFile`, `mkstemp` (mode 0600) or `TemporaryDirectory`.

**Secrets in logs and reprs.** `log.debug("settings %s", settings)`, logging `request.headers`, a dataclass or model's default repr, or an exception carrying a connection URL writes credentials into log storage. Type secrets as `pydantic.SecretStr`, set `field(repr=False)` on dataclass secrets, log allowlisted fields. Credentials that reached logs are compromised and need rotating.

**Weak tokens and comparisons.** `random` is predictable; tokens and reset codes come from `secrets`. Compare MACs and tokens with `hmac.compare_digest`.

**Fail-open configuration.** `os.environ.get("JWT_SECRET", "dev-secret")`, `DEBUG = os.getenv("DEBUG", "true")`, `bool(os.getenv("VERIFY_TLS"))` (true for the string `"false"`). Security-relevant settings are required fields in a typed settings object that fails at startup when missing or malformed.

### Process and module structure

**Import-time side effects.** Connecting to databases, reading secrets, starting threads or calling APIs at import makes test collection slow and order-dependent, breaks tools that import for `--help`, and under a pre-fork server shares one socket across every worker. Create resources in a factory or lifespan hook called by the entry point.

**Connections inherited across fork.** Pools created before gunicorn `--preload`, Celery prefork or `multiprocessing` fork are copied into each child; two processes then share a socket and read each other's results. Create engines and clients after fork, or call SQLAlchemy `engine.dispose(close=False)` in the child initializer (1.4.33+).

**Circular imports.** Show up as `ImportError: cannot import name` or an `AttributeError` on a partially initialized module, often only for one entry point, so tests pass and production fails. Fix the dependency direction (move shared types down a layer); use `if TYPE_CHECKING:` for annotation-only cycles; a function-local import is the last resort.

**Per-process state in web workers.** A module-level dict used as a rate limiter, idempotency store, session store, lock or cache exists once per process: with four workers on three pods there are twelve copies, limits multiply, dedupe misses, everything resets on deploy. Shared state lives in the database or Redis. Module globals written per request also leak between users on the same worker.

**Thread-safety by GIL folklore.** The GIL makes single bytecodes atomic, not your operations. `counter += 1`, `if key not in d: d[key] = build()` and read-modify-write on shared objects race under FastAPI's threadpool, `ThreadPoolExecutor` or gunicorn threads. Free-threaded builds (available from 3.13) remove the incidental serialization entirely, and even there the docs recommend explicit locks over relying on built-in types' internal locking. Guard shared mutable state with `threading.Lock` or don't share it. A SQLAlchemy `Session` and most client objects with per-call state are single-thread.

### Pydantic and SQLAlchemy

Details and migration tables in `references/pydantic-sqlalchemy.md`.

**Pydantic v1 habits in v2.** `Optional[X]` without a default is now required; ints no longer coerce to `str`; `.dict()`, `parse_obj`, `@validator`, `class Config` and `orm_mode` are renamed or deprecated. Extra keys are ignored by default, so privileged input models need `ConfigDict(extra="forbid")`.

**Validated model, unvalidated data.** `model_copy(update=raw_dict)` and `model_construct()` skip validation. Validating a request with one model and then applying the raw dict (or `setattr` over `body.model_dump()` without `exclude_unset=True`) is mass assignment and null-overwrite. Apply only the fields of the validated model.

**Session shared across requests.** A module-level `Session(engine)` is shared by concurrent threadpool requests (not thread-safe), keeps a growing identity map of stale objects, and after one failed flush raises `PendingRollbackError` for every later request. One session per request or unit of work, closed in `finally`; one `AsyncSession` per task.

**Lazy-load N+1.** Relationships default to `lazy="select"`: `order.customer` inside a loop over 100 orders is 100 queries, and under `AsyncSession` the same access raises instead of loading. Eager-load with `selectinload` (collections) or `joinedload` (many-to-one), and set `lazy="raise"` or `raiseload("*")` so a new N+1 fails in tests.

### Dependencies and tests

**Unlocked installs.** `requests>=2` resolves differently per build, so a bad upstream release ships with no code change. Install from a lock file with hashes (`uv sync --locked`, pip-tools output installed with `--require-hashes`).

**Typosquats and index confusion.** A package name typed from memory can be a malicious lookalike; check the PyPI page, source repo and maintainers before adding. pip treats `--extra-index-url` as equal to the main index and takes the highest version, so a public package with your internal package's name wins. Resolve internal names only from the internal index.

**Test state leaking.** Module- or session-scoped fixtures returning mutable objects, `os.environ[...] =` without restore, an `lru_cache`d `get_settings()` built from the first test's environment, and committed DB rows make tests order-dependent. Patching `requests.post` does not affect a module that did `from requests import post`; patch where the name is looked up. Details in `references/testing.md`.

## Decision rules

- Sync or async: async only when the service mostly waits on many concurrent network calls and every library on the hot path has an async client. Otherwise sync with a threadpool. In async code, every blocking call goes through `to_thread` or is replaced.
- Running coroutines together: `TaskGroup` when all must succeed and one failure should cancel the rest. `gather(return_exceptions=True)` when each result is independent and you check every one. A bare `create_task` only through a tracked set with a done callback. A durable queue when the work must survive the process.
- Money: integer minor units when the currency exponent is fixed and amounts only add and subtract; `Decimal` with explicit `quantize` points when rates, percentages or proration are involved.
- Boundary types: Pydantic (or attrs with validators) where data crosses a trust boundary; plain dataclasses for internal values already validated.
- Retrying a call: yes if the operation is idempotent or keyed; otherwise record the attempt as pending and reconcile.
- Eager loading: `selectinload` for one-to-many and many-to-many, `joinedload` for many-to-one; `.unique()` on the result when joined-loading a collection.

## Review checklist

- Does any function, class attribute or dataclass field share a mutable default?
- Do closures created in loops bind the loop variable at creation?
- Is any generator or iterator read twice or tested with `any()`/`bool()` before the loop?
- Is every `except` narrow enough to act on, and does every broad one sit at a boundary and log with a traceback?
- Can any `except` in a coroutine catch `CancelledError` without re-raising?
- Inside `async def`, is there any sync HTTP, sleep, DB driver, file or CPU-heavy call?
- Is every created task referenced, supervised and cancelled on shutdown?
- Is concurrency over data-sized input bounded by a configured limit?
- Does every network call and subprocess have a timeout, and is a timeout treated as an unknown outcome?
- Are HTTP clients and DB engines created once per process and closed on shutdown?
- Is money free of `float` from parsing to storage, with explicit quantize points and rounding modes?
- Are all datetimes aware, created with an explicit `tz`, and is `fromtimestamp` given `tz=`?
- Is any untrusted input reaching `pickle`, `yaml.load`, `eval`, a shell, SQL text or a filesystem path?
- Do security settings fail at startup when missing, with no default secrets?
- Could any log line or repr include a token, password, cookie or connection string?
- Is any module-level state written per request or relied on as shared across workers?
- Does each request get its own DB session, with relationships eager-loaded in list endpoints?
- Is every request body applied through the validated model, never the raw dict?
- Is the dependency added real, maintained and pinned in the lock file?
- Do tests restore env, caches and globals, and patch names where they are looked up?

## References

- `references/async.md`: read when writing or reviewing asyncio code: supervised background tasks, bounded fan-out, timeouts, cancellation, blocking calls, shutdown.
- `references/boundaries.md`: read when code handles untrusted input or the outside world: HTTP clients, subprocess, SQL, file paths, archives, uploads, deserialization, settings and logging.
- `references/money-and-time.md`: read for any code computing money, rates, proration, rounding, timestamps, time zones or durations.
- `references/pydantic-sqlalchemy.md`: read when using Pydantic models or SQLAlchemy sessions, or migrating Pydantic v1 to v2 or SQLAlchemy 1.x to 2.0.
- `references/testing.md`: read when writing pytest fixtures or mocks, or chasing tests that pass alone and fail in the suite.

Related skills: security-engineering for authorization, SSRF and threat modelling; database-engineering for transactions, locking and migrations; backend-architecture for idempotency, outbox and reconciliation; test-engineering for test strategy; performance-benchmarking before optimizing.
