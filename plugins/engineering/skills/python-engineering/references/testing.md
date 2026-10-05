# pytest: isolation and mocks

Read this when writing pytest fixtures or mocks, or when tests pass alone and fail in the full suite, in a different order or under parallel runs. Test strategy in general is in the test-engineering skill.

## State that leaks between tests

The symptom is order dependence: a test passes with `pytest path::test_x`, fails in the full run, or the other way round. Running with random order (pytest-randomly) or in parallel (pytest-xdist) exposes it early.

| Leak | Fix |
|---|---|
| Module- or session-scoped fixture returning a mutable object that tests modify | Function scope for anything mutable; share only immutable or read-only resources at wider scope |
| `os.environ["X"] = ...` in a test or fixture | `monkeypatch.setenv` / `monkeypatch.delenv`, undone after the test |
| `@lru_cache` on `get_settings()` or a client factory, built from whichever test ran first | A fixture that calls `get_settings.cache_clear()` before and after, or inject settings instead of caching a global |
| Module-level singletons (registries, clients, in-memory stores) | Construct per test and inject; reset in an autouse fixture if the design cannot change yet |
| Rows committed by one test visible in the next | Each test in a transaction rolled back at teardown, or a fresh schema per worker |
| `mock.patch` started without `stop()`, or patching in `setup_module` | `with patch(...)` or the `mocker`/`monkeypatch` fixtures, which undo automatically |
| Frozen time or seeded random left in place | Inject a clock and an RNG; if patching, use a fixture that restores |

Transaction-per-test with SQLAlchemy 2.x:

```python
@pytest.fixture
def db(engine: Engine) -> Iterator[Session]:
    with engine.connect() as conn:
        outer = conn.begin()
        session = Session(bind=conn, join_transaction_mode="create_savepoint")
        try:
            yield session
        finally:
            session.close()
            outer.rollback()
```

With `join_transaction_mode="create_savepoint"`, code under test can call `session.commit()` and `rollback()` normally; everything is discarded when the outer transaction rolls back. This does not cover code that opens its own connections (background threads, a second engine), and it hides bugs that only appear across real commits, such as two concurrent transactions racing. Test those against a real database with separate connections.

On SQLite the stdlib driver's legacy transaction handling breaks the savepoint, so the test's rows are committed for real and leak into later tests. Create the engine with `connect_args={"autocommit": False}` (Python 3.12+), or run these tests on the production database engine.

## Patching the right name

`patch` replaces a name in one namespace. It must be the namespace where the code under test looks the name up.

```python
# billing/charge.py
from requests import post

def charge(...):
    return post(...)

# Wrong: charge() already holds its own reference to requests.post
with patch("requests.post") as fake: ...

# Right
with patch("billing.charge.post") as fake: ...
```

- Use `autospec=True` (or `create_autospec`) so a mock rejects calls that do not match the real signature. A bare `MagicMock` accepts anything, so tests stay green after the real function's arguments change.
- Assert on outcomes and on the arguments that matter (amount, idempotency key), not on the exact sequence of internal calls.
- Mock at process boundaries (HTTP, payment provider, clock), not your own modules. For HTTP, a transport-level fake (httpx `MockTransport`, the `responses` library for requests) exercises your real client code, timeouts and error handling.
- A mocked dependency never times out, returns malformed data or succeeds after reporting failure. Write those cases explicitly.

## Things tests commonly miss in Python code

- The second call: mutable default arguments and module-level caches only misbehave on the second invocation in the same process. Call the function twice with different inputs.
- Aware vs naive: construct test datetimes with `tzinfo`, and include a case on a non-UTC local zone (`monkeypatch.setenv("TZ", ...)` plus `time.tzset()` on Unix) for code that converts timestamps.
- Money: amounts whose float representation is inexact (`0.1`, `19.99`, `2.675`) and allocations that do not divide evenly.
- Async wiring: pytest-asyncio defaults to strict mode, where async tests need `@pytest.mark.asyncio` and async fixtures need `@pytest_asyncio.fixture` (or set `asyncio_mode = "auto"`). pytest 8.4 and later fail an async test that no plugin runs instead of skipping it, and pytest 9 errors on an unhandled async fixture. In pytest-asyncio 1.4 an async fixture's loop defaults to the fixture's own scope, while tests default to a per-function loop, so a session-scoped engine or client used from ordinary tests fails with "attached to a different loop" or "Event loop is closed". Give the tests the same `loop_scope` as the fixture (or set `asyncio_default_test_loop_scope`). pytest-asyncio 1.0 removed the `event_loop` fixture, so overriding it in an old `conftest.py` no longer does anything.
- Cancellation and timeouts in async code: cancel the task mid-operation and assert cleanup happened and no partial write remains.
- Concurrency: two threads or tasks hitting the same code path, with a barrier to force the interleaving, rather than hoping a loop of 100 iterations catches it.
