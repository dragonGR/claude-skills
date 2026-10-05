# asyncio patterns

Read this when writing or reviewing asyncio code: background tasks, fan-out, timeouts, cancellation, blocking calls and shutdown. Examples assume Python 3.11+ (`TaskGroup`, `asyncio.timeout`, `except*`).

## Supervised background tasks

A task you do not await needs an owner that holds a strong reference, logs its failure and cancels it on shutdown. The loop itself holds only weak references.

```python
import asyncio
import logging
from collections.abc import Coroutine

log = logging.getLogger(__name__)


class BackgroundTasks:
    def __init__(self) -> None:
        self._tasks: set[asyncio.Task[None]] = set()

    def spawn(self, coro: Coroutine[object, object, None], *, name: str) -> None:
        task = asyncio.create_task(coro, name=name)
        self._tasks.add(task)
        task.add_done_callback(self._finished)

    def _finished(self, task: asyncio.Task[None]) -> None:
        self._tasks.discard(task)
        if task.cancelled():
            return
        exc = task.exception()
        if exc is not None:
            log.error("background task %s failed", task.get_name(), exc_info=exc)

    async def shutdown(self, grace_seconds: float) -> None:
        if not self._tasks:
            return
        _, pending = await asyncio.wait(set(self._tasks), timeout=grace_seconds)
        for task in pending:
            task.cancel()
        await asyncio.gather(*pending, return_exceptions=True)
```

Create one instance in the application lifespan, call `shutdown` on exit with a grace period from config. This only protects work while the process lives. If losing the work on a crash or deploy is not acceptable (webhooks, emails, payments), write it to a durable queue or outbox table in the same transaction as the state change and let a worker deliver it.

## Bounded fan-out

Two shapes. Pick by input size.

A semaphore plus `TaskGroup` when the input fits in memory as tasks (thousands, not millions):

```python
async def refresh_all(
    http: httpx.AsyncClient,
    accounts: list[Account],
    limit: int,
) -> None:
    sem = asyncio.Semaphore(limit)

    async def one(account: Account) -> None:
        async with sem:
            try:
                await refresh_balance(http, account)
            except httpx.HTTPError:
                log.exception("refresh failed", extra={"account_id": account.id})

    async with asyncio.TaskGroup() as tg:
        for account in accounts:
            tg.create_task(one(account))
```

The per-item `try` matters: inside a `TaskGroup`, one unhandled exception cancels every sibling. Catch what a single item can fail with, and let everything else (programming errors) abort the batch.

A fixed worker pool reading a bounded queue when the input is a stream or very large. Memory stays at `workers * 2` pending items and the producer slows down when workers fall behind:

```python
async def process_stream(
    items: AsyncIterator[Item],
    handle: Callable[[Item], Awaitable[None]],
    workers: int,
) -> None:
    queue: asyncio.Queue[Item] = asyncio.Queue(maxsize=workers * 2)

    async def worker() -> None:
        while True:
            item = await queue.get()
            try:
                await handle(item)
            except Exception:
                log.exception("item failed", extra={"item_id": item.id})
            finally:
                queue.task_done()

    async with asyncio.TaskGroup() as tg:
        pool = [tg.create_task(worker()) for _ in range(workers)]
        async for item in items:
            await queue.put(item)
        await queue.join()
        for task in pool:
            task.cancel()
```

Cancelling the workers after `join()` is normal: a `TaskGroup` does not treat a child's `CancelledError` as a failure.

Size the limit from the downstream's capacity (its rate limit, your HTTP client's pool size, the DB pool), and keep it in config. A semaphore larger than the HTTP client's connection pool only moves the queueing into the pool, where the pool timeout fires instead.

## Timeouts and unknown outcomes

```python
try:
    async with asyncio.timeout(settings.provider_timeout_s):
        resp = await http.post(url, json=payload, headers={"Idempotency-Key": op_id})
except (TimeoutError, httpx.TransportError):
    await mark_pending(op_id)
    raise
```

`asyncio.timeout` cancels the body and raises the builtin `TimeoutError`. The client's own timeouts raise `httpx.TimeoutException`, which is a `TransportError` and not a `TimeoutError`, so catching only `TimeoutError` misses them. In both cases the request may have reached the provider. Record the operation as pending with its idempotency key and reconcile it (query the provider, or retry with the same key), never mark it failed and never retry without the key.

`asyncio.wait_for(aw, timeout)` does the same for a single awaitable. It waits for the cancelled task to actually finish, so total time can exceed the timeout when the inner code is slow to clean up.

## Cancellation

Cancellation arrives as `CancelledError` at the current `await`. Write coroutines so that is safe:

```python
# Before: shutdown and timeouts are absorbed; the loop keeps polling
async def wait_for_export(http, job_id):
    while True:
        try:
            status = (await http.get(f"/exports/{job_id}")).json()["status"]
            if status == "done":
                return
        except BaseException:
            log.warning("poll failed")
        await asyncio.sleep(settings.poll_interval_s)

# After: cancellation propagates; transport errors are retried up to a limit
async def wait_for_export(http: httpx.AsyncClient, job_id: str) -> None:
    failures = 0
    while True:
        try:
            resp = await http.get(f"/exports/{job_id}")
            resp.raise_for_status()
            if ExportStatus.model_validate_json(resp.content).status == "done":
                return
            failures = 0
        except httpx.HTTPError:
            failures += 1
            if failures >= settings.poll_max_failures:
                raise
        await asyncio.sleep(settings.poll_interval_s)
```

The caller bounds the total wait with `asyncio.timeout`, which only works because the loop lets `CancelledError` through.

Rules:

- Clean up in `finally` or `async with`. If you must catch `CancelledError` to do something specific, re-raise it.
- Do not catch `BaseException` in coroutines. `except Exception` is fine; it does not catch `CancelledError`.
- `asyncio.shield(aw)` keeps the inner awaitable running when the caller is cancelled, but the caller still gets `CancelledError`. Use it narrowly, for a short step that must finish once started (writing the result of a side effect that already happened), and still keep a reference to the shielded task.
- Code after an `await` may never run. Do not put "release the reservation" or "mark done" after an `await` without a `finally`.

## Blocking calls

Inside `async def`:

| Blocking | Replace with |
|---|---|
| `requests`, `urllib.request` | shared `httpx.AsyncClient` or `aiohttp.ClientSession` |
| `time.sleep` | `await asyncio.sleep` |
| psycopg2, PyMySQL, sync SQLAlchemy `Session` | asyncpg, psycopg 3 async, SQLAlchemy `AsyncSession` |
| `boto3` and other sync SDKs, password hashing, `open().read()` on large files | `await asyncio.to_thread(fn, *args)` |
| CPU-bound parsing, image work, crypto over large data | `loop.run_in_executor(process_pool, fn, *args)` |

`to_thread` uses the loop's default thread pool, which is small. A slow blocking call occupying every thread stalls every other `to_thread` in the process. Give heavy or slow blocking work its own bounded executor. Also, the thread keeps running after the awaiting coroutine is cancelled; cancellation does not interrupt it.

To find a stall: run with `PYTHONASYNCIODEBUG=1` (or `asyncio.run(main(), debug=True)`), which logs callbacks slower than 100 ms. `py-spy dump --pid <pid>` on a live worker shows what the loop thread is blocked on.

## Shared clients and lifespan

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    timeout = httpx.Timeout(settings.http_timeout_s, connect=settings.http_connect_timeout_s)
    app.state.http = httpx.AsyncClient(timeout=timeout)
    app.state.background = BackgroundTasks()
    try:
        yield
    finally:
        await app.state.background.shutdown(settings.shutdown_grace_s)
        await app.state.http.aclose()
        await engine.dispose()
```

Create clients inside the running loop (lifespan, not import time), close them after background work has drained.

## Request-scoped context

```python
from contextvars import ContextVar

request_id: ContextVar[str | None] = ContextVar("request_id", default=None)
```

Create context variables at module level, never inside functions or closures. Each asyncio task runs in a copy of the context current when it was created, so a value set in middleware is visible in tasks spawned by that request and not in other requests. `threading.local` offers no such isolation between coroutines.

## Calling into the loop from threads

From a thread (a sync callback, a thread pool worker, a signal-handling library), do not call loop or task methods directly. Use `loop.call_soon_threadsafe(callback, *args)` for callbacks and `asyncio.run_coroutine_threadsafe(coro, loop)` for coroutines; the latter returns a `concurrent.futures.Future` you can wait on with a timeout.
