# Tokio patterns

Read this when writing or reviewing Tokio code that races futures, spawns tasks, talks to the network, shares state between tasks or has to shut down cleanly.

## Cancellation safety

A future passed to `select!` or `timeout` that does not finish is dropped at the await it was parked on. The question for each branch is what happens to work already done. From the Tokio docs:

| Cancel-safe | Not cancel-safe (loses data) | Loses queue position only |
|---|---|---|
| `mpsc::Receiver::recv`, `UnboundedReceiver::recv` | `AsyncReadExt::read_exact` | `sync::Mutex::lock` |
| `broadcast::Receiver::recv` | `AsyncReadExt::read_to_end` | `sync::RwLock::read`, `write` |
| `watch::Receiver::changed` | `AsyncReadExt::read_to_string` | `Semaphore::acquire` |
| `TcpListener::accept`, `UnixListener::accept` | `AsyncWriteExt::write_all` | `Notify::notified` |
| `AsyncReadExt::read`, `read_buf` | | |
| `AsyncWriteExt::write`, `write_buf` | | |
| `StreamExt::next` (tokio-stream and futures) | | |

Your own `async fn` is cancel-safe only if every await in it is, and only if dropping it between two awaits leaves no half-applied state. An async fn that calls a payment provider and then writes the result to the database is not cancel-safe no matter what primitives it uses.

Three ways to make a multi-step operation survive cancellation:

1. Do not race it. Check for shutdown between units of work, not during one (the loop in SKILL.md).
2. Spawn it, so dropping the select branch drops only the `JoinHandle`, and the task finishes on its own. Track it with a `TaskTracker` so shutdown waits for it.
3. Make it restartable: persist "attempt pending" with an idempotency key before the external call, and have a reconciler finish or retry pending attempts after a crash. This is the only option that also survives process death.

## Loops with timers

A `sleep` created inside the loop restarts every iteration. For an idle timeout that is what you want; for a heartbeat or an overall deadline it means the timer never fires while messages keep arriving.

```rust
// Before: under steady traffic, no ping is ever sent and the peer drops us
loop {
    tokio::select! {
        _ = tokio::time::sleep(cfg.heartbeat_every) => conn.send(Frame::Ping).await?,
        msg = inbound.recv() => match msg {
            Some(msg) => handle(msg).await?,
            None => return Ok(()),
        },
    }
}

// After: the timer lives outside the loop; Interval::tick is cancel-safe
let mut heartbeat = tokio::time::interval(cfg.heartbeat_every);
heartbeat.set_missed_tick_behavior(MissedTickBehavior::Delay);
let session_deadline = tokio::time::sleep(cfg.max_session);
tokio::pin!(session_deadline);

loop {
    tokio::select! {
        _ = &mut session_deadline => return Err(SessionError::Expired),
        _ = heartbeat.tick() => conn.send(Frame::Ping).await?,
        msg = inbound.recv() => match msg {
            Some(msg) => handle(msg).await?,
            None => return Ok(()),
        },
    }
}
```

`interval` completes its first tick immediately; call `tick().await` once before the loop if the first action should wait a full period. The default `MissedTickBehavior::Burst` catches up on missed ticks as fast as possible; `Delay` restarts the schedule from now, `Skip` jumps to the next aligned tick. Pollers and anything that calls a rate-limited upstream want `Delay` or `Skip`.

For an idle timeout, keep one pinned `Sleep` and call `reset(new_deadline)` on it when activity arrives.

## Supervised fan-out

```rust
pub async fn notify_all(
    http: reqwest::Client,
    targets: Vec<Target>,
    max_in_flight: usize,
) -> Vec<(TargetId, Result<(), NotifyError>)> {
    let permits = Arc::new(Semaphore::new(max_in_flight));
    let mut set = JoinSet::new();

    for target in targets {
        let permit = Arc::clone(&permits)
            .acquire_owned()
            .await
            .expect("semaphore is owned here and never closed");
        let http = http.clone();
        set.spawn(async move {
            let _permit = permit;
            let result = notify(&http, &target).await;
            (target.id, result)
        });
    }

    let mut results = Vec::new();
    while let Some(joined) = set.join_next().await {
        match joined {
            Ok(outcome) => results.push(outcome),
            Err(err) if err.is_panic() => std::panic::resume_unwind(err.into_panic()),
            Err(err) => tracing::warn!(error = %err, "notify task cancelled"),
        }
    }
    results
}
```

Points a reviewer checks:

- The permit is acquired before `spawn`, so at most `max_in_flight` tasks exist, not merely run.
- `_permit` is bound by name inside the task. `let _ = permit;` would release it immediately.
- Every `join_next` result is inspected. Re-raising a panic is one choice; logging it with the target id (carry the id outside the task) is another. Silently skipping it is not.
- If this function's future is dropped, the `JoinSet` drops and aborts the remaining tasks. That is right for request-scoped work. For work that must finish regardless, use a `TaskTracker` or a queue.

## Graceful shutdown

```rust
let shutdown = CancellationToken::new();
let tracker = TaskTracker::new();

for worker_id in 0..cfg.worker_count {
    let shutdown = shutdown.clone();
    let deps = deps.clone();
    tracker.spawn(async move {
        if let Err(err) = run_worker(worker_id, deps, shutdown).await {
            tracing::error!(worker_id, error = ?err, "worker exited with error");
        }
    });
}

tokio::signal::ctrl_c().await?;
shutdown.cancel();
tracker.close();
if tokio::time::timeout(cfg.shutdown_grace, tracker.wait()).await.is_err() {
    tracing::warn!("workers still running after the shutdown grace period");
}
```

`TaskTracker::wait` returns once the tracker is closed and empty. Unlike `JoinSet`, dropping a tracker does not abort its tasks, and it frees task outputs as tasks exit, so it suits long-lived background work. It does not report panics, so each task logs its own errors as above. In production also listen for SIGTERM (`tokio::signal::unix::signal(SignalKind::terminate())`), which is what container orchestrators send.

Inside `run_worker`, check the token between units of work with `biased;` so shutdown wins over new work, and never race the token against a half-finished side effect.

`spawn_blocking` work ignores cancellation and shutdown waits for it. Long blocking loops need their own stop flag (an `AtomicBool` or the token's `is_cancelled()`), checked each iteration.

## Replacing `Arc<Mutex<HashMap>>` with an owner task

```rust
enum Command {
    Reserve {
        sku: Sku,
        qty: u32,
        reply: oneshot::Sender<Result<ReservationId, ReserveError>>,
    },
    Release {
        id: ReservationId,
    },
}

async fn inventory_owner(mut commands: mpsc::Receiver<Command>, mut stock: Stock) {
    while let Some(cmd) = commands.recv().await {
        match cmd {
            Command::Reserve { sku, qty, reply } => {
                // The caller may have timed out and dropped its receiver; the reservation
                // still expires through Stock's TTL, so there is nothing else to undo here.
                let _ = reply.send(stock.reserve(sku, qty));
            }
            Command::Release { id } => stock.release(id),
        }
    }
}

#[derive(Clone)]
pub struct Inventory {
    commands: mpsc::Sender<Command>,
    reply_timeout: Duration,
}

impl Inventory {
    pub async fn reserve(&self, sku: Sku, qty: u32) -> Result<ReservationId, ReserveError> {
        let (reply, response) = oneshot::channel();
        self.commands
            .send(Command::Reserve { sku, qty, reply })
            .await
            .map_err(|_| ReserveError::Unavailable)?;
        tokio::time::timeout(self.reply_timeout, response)
            .await
            .map_err(|_| ReserveError::Timeout)?
            .map_err(|_| ReserveError::Unavailable)?
    }
}
```

The owner serializes all mutations without a lock, so there is no lock ordering, no poisoning and no guard across an await. Its rules: the command channel is bounded; the owner never does slow I/O inline (it would become the bottleneck); `ReserveError::Timeout` is an unknown outcome, because the owner may have processed the command after the caller gave up, which is why reservations carry an expiry; and if the owner can panic, something must notice and restart it, since every later `send` fails with the channel closed.

Use the owner-task pattern when several tasks mutate the same state or the state is an I/O resource. Keep a std `Mutex` when the state is small, critical sections are short and nothing awaits while holding it.

## Blocking and CPU work

```rust
// Password hashing is deliberately slow CPU work; on a runtime worker it stalls every
// task on that thread. The semaphore keeps a login burst from spawning unbounded threads.
let _permit = state.hash_permits.acquire().await?;
let password = password.clone();
let params = state.hash_params.clone();
let hash = tokio::task::spawn_blocking(move || hash_password(&password, &params)).await??;
```

The first `?` handles `JoinError` (a panic or cancellation in the closure), the second the hashing error; give your error type `#[from] JoinError` rather than stringifying it. For sustained parallel CPU work (image processing, large batch transforms) use rayon and hand results back through a `oneshot`. `block_in_place` exists for the multi-threaded runtime only and panics on `current_thread`.

On a `current_thread` runtime, or under `spawn_local`, a std `MutexGuard` held across an await compiles (the future need not be `Send`), and a second task locking the same mutex blocks the only thread: a deadlock with no panic and no log line.

## HTTP clients and timeouts

```rust
let http = reqwest::Client::builder()
    .connect_timeout(cfg.http_connect_timeout)
    .timeout(cfg.http_request_timeout)
    .redirect(reqwest::redirect::Policy::none())
    .build()?;
```

Build one client per process and clone it; it holds the connection pool. Disable or limit redirects when the URL comes from a user or a webhook registration (SSRF through redirect, see security-engineering). For a write that timed out, the outcome is unknown: send an idempotency key the server honors and retry with the same key, or mark the operation pending and reconcile by querying the provider. Never mark it failed and create a fresh attempt.

## Channels

- Capacity comes from config and is never 0 (`mpsc::channel(0)` panics).
- Inside a pipeline, `send().await` blocks the producer when the consumer is behind. That is the backpressure you want, provided the producer is not holding a lock or a DB transaction while it waits.
- At an HTTP edge, `try_send` and map `TrySendError::Full` to 503 or 429 so overload becomes a fast rejection instead of memory growth.
- `broadcast` overwrites the oldest value when full; a slow receiver gets `RecvError::Lagged(skipped)`, and its cursor jumps forward. Treat that as data loss: count it, and resynchronize from the source of truth if the messages were state updates.
- `watch` keeps only the latest value, which suits config and health state and does not suit events.

```rust
loop {
    match updates.recv().await {
        Ok(update) => view.apply(update),
        Err(RecvError::Lagged(skipped)) => {
            metrics.lagged_updates.inc_by(skipped);
            view = load_snapshot(&db).await?;
        }
        Err(RecvError::Closed) => break,
    }
}
```

## Explicit async cleanup

```rust
impl Exporter {
    pub async fn finish(mut self) -> io::Result<()> {
        self.writer.flush().await?;
        self.writer.get_ref().sync_all().await?;
        Ok(())
    }
}
```

Tokio's `BufWriter` throws away unflushed bytes on drop, and a Tokio `File` dropped with writes in flight is not closed immediately. `flush` hands data to the OS; `sync_all` asks the OS to put it on disk. Call the finishing method on the success path and decide explicitly what the error path does with a partial file (delete it, or write to a temp name and rename only after `finish` succeeds).

## `Send` errors

```rust
// Fails to compile under tokio::spawn: the Rc is alive across the await
let parsed = Rc::new(parse(&body)?);
let owner = lookup_owner(&db, parsed.owner_id).await?;

// Compiles: the !Send value is gone before the first await
let owner_id = {
    let parsed = Rc::new(parse(&body)?);
    parsed.owner_id
};
let owner = lookup_owner(&db, owner_id).await?;
```

The same fix applies to `MutexGuard` and `RefCell` borrows. When the error names a type you do not own, read the "has type ... which is not `Send`" line and the await it points at; the value is usually a temporary in a longer expression, and binding it in a block before the await fixes it.
