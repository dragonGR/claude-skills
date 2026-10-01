---
name: test-engineering
description: Tests that catch real bugs: levels, mocks versus real dependencies, concurrency, idempotency and authorization tests, property-based tests, fake clocks, flaky tests, snapshots, coverage, Playwright. Load it before writing or reviewing any test or test suite, when tests pass but bugs ship, or when a test is flaky.
license: MIT
metadata:
  author: Alex Tsanis
---

# Test engineering

A test earns its place when it fails if the behavior it is named after breaks. Suites that let money and data bugs through are rarely short of tests. They are full of tests that stay green with the guard deleted, the database swapped for SQLite, the race left in or the response shape wrong. Review test code like production code: for each test, look for the way it passes while the product is broken.

## Before you judge a suite

- Rank the code by damage: money, authorization, durable state, irreversible side effects, then the rest. Depth goes to the top of that list.
- Read the fixtures, `conftest.py`, setup files and test config before the tests. The lies usually live there: auth overridden, an in-memory database, a global mock, retries switched on.
- For each test, establish what it runs against: the real router and middleware, the production database engine and major version, real migrations, the real serializer and driver.
- Establish how isolation works: rollback per test, truncation, a fresh database, or nothing.
- Establish what is faked: which modules are mocked, which hosts are intercepted, whether clock and randomness are controlled.
- For every "this test is weak" claim, name the concrete production change it would let through. If you cannot name one, drop the claim.

## Failure catalogue

### Tests that prove nothing

**Passes with the guard removed.** The ownership check, status check or limit in a sensitive path can be deleted and the suite stays green, usually because every test uses the owner, a valid state and small amounts. For each guard, delete or invert it locally, run the tests, and confirm one fails for the right reason. Procedure and mutation tools in `references/guard-checks.md`.

**Assertions that never run.** The assertion sits somewhere that is skipped when the code is wrong.

```ts
// Before: passes when transfer() succeeds, because the catch block never runs
try {
  await transfer(fromId, toId, amountOverBalance);
} catch (e) {
  expect(e).toBeInstanceOf(InsufficientFunds);
}

// After
await expect(transfer(fromId, toId, amountOverBalance)).rejects.toBeInstanceOf(InsufficientFunds);
```

Same family: `expect(p).rejects...` without `await`; assertions inside a `forEach` over a result that came back empty; assertions in an event callback that never fires; lines after the raising call inside `with pytest.raises(...)`, which never execute; tests with no assertion at all.

**Mocking the thing under test.** The test patches the repository, validator or pricing function, then checks that the service returned what the mock returned. The logic that can be wrong never runs. Mock only at the process edge: third-party HTTP, email, payment provider, clock. In Python, `mock.patch` must target the name where it is looked up: if `billing.py` did `from payments.client import charge`, patching `payments.client.charge` leaves `billing.charge` pointing at the real function, and the real call goes out. Use `autospec=True` so a mock rejects calls the real signature would reject.

**Call counts instead of outcomes.** `expect(repo.debit).toHaveBeenCalledWith(walletId, 100)` passes when the SQL in `debit` is wrong, the transaction never commits, or the credit fails after the debit. Assert what a caller can observe: balances, ledger rows, the response, the emitted event. The exception is a side effect that leaves your system (a charge, a payout, an email): a fake at that boundary that records what was sent is the outcome, and asserting it happened exactly once is correct.

**Tautological oracles.** The expected value is computed with the same function or a copy of its formula: `expect(fee(amount)).toBe(amount * FEE_RATE)`. A wrong formula passes. Use hand-computed literal cases for known inputs (rounding edges included) or an independent property: shares sum to the total, fee never exceeds the amount.

**Weak matchers.** `toBeTruthy()`, `toBeDefined()`, `assert resp`, `status_code != 200`, bare `toThrow()`, `pytest.raises(Exception)`. Each passes on failures unrelated to the behavior: a `TypeError` from a typo in the test, a 422 from a malformed body, a 500. Assert the exact status, the error type or code, and the resulting state.

**Negative tests that pass for the wrong reason.** "Bob cannot read Alice's invoice" gets its 403 because Bob's token was malformed, the CSRF header was missing, or the URL was wrong, not because of the ownership check. Pair every denial with a positive control: the identical request by an allowed caller succeeds. After a denial, assert the state did not change and no side effect was emitted.

**Auth bypassed in the harness.** A global fixture sets `app.dependency_overrides[get_current_user] = lambda: admin`, a test settings flag disables auth middleware, or the user factory defaults to staff. Every authorization test then runs as an admin and proves nothing. Keep the real authentication path in tests and mint real sessions or tokens per role through a helper. Template in `references/authz-matrix.md`.

**Asserting error message text.** `toThrow("Insufficient funds: balance 500 < 600")` or `pytest.raises(E, match="...")` on prose breaks when copy changes, so people loosen it until it matches anything. Assert the error class or a machine-readable code (`body.error.code == "insufficient_funds"`). Check text only where the text is the contract (user-facing copy under review, CLI output that scripts parse), and match the stable part.

**Unread snapshots.** A whole API response or component tree is snapshotted, including ids and timestamps, so it changes on every run and gets updated with `-u` without anyone reading it. Snapshot small, stable output, normalize volatile fields, prefer explicit field assertions for anything that matters, and treat a snapshot diff in review as a behavior change to read. Jest with `--ci` and Vitest under `CI` fail on missing snapshots instead of writing them.

**Implementation-pinned tests.** Tests of private methods, internal call order, exact SQL strings or internal state. Every refactor breaks them, people learn to update them mechanically, and then they catch nothing. Test through the public interface at the level where the behavior is observable.

**Gamed coverage.** Tests that execute code without meaningful assertions, `# pragma: no cover` or `/* istanbul ignore */` on error branches, hard files excluded from measurement, line coverage reported where branch coverage would show the untaken `else`. Coverage shows what never ran, never what was checked. Use it to find untested risky code; use mutation testing on money and auth modules to measure the assertions.

**Tests that never run.** A committed `.only`, a `skip` with no linked issue, files that do not match the collector pattern, a pytest class with an `__init__` (pytest skips collecting it with a warning), `--passWithNoTests`, a CI job running a subset. Compare the test count CI reports with a local run. Playwright has `forbidOnly`; Vitest's `allowOnly` defaults to false when `CI` is set.

### Fakes that lie

**A different database engine.** SQLite, H2 or an in-memory repository standing in for PostgreSQL or MySQL. Non-`STRICT` SQLite accepts `'abc'` in an `INTEGER` column, ignores foreign keys unless `PRAGMA foreign_keys = ON`, treats `LIKE` as case-insensitive for ASCII, has no `SELECT ... FOR UPDATE`, and allows one writer at a time, so races cannot happen. An in-memory fake has no constraints at all, so check-then-insert looks correct. Run data-access and integration tests against the production engine and major version (Testcontainers or a CI service container), with the schema built by the real migrations rather than `create_all()` from models, which silently drops migration-only constraints and indexes.

**Rollback isolation hiding commit-time behavior.** Each test runs inside a transaction that is rolled back. Nothing commits, so: deferred constraints are never checked (Django's `TestCase` checks them at the end of each test; most hand-rolled fixtures do not); `on_commit` hooks and outbox relays never fire; code that opens its own connection cannot see the test's rows; two "concurrent" requests share one connection and never contend; serialization failures and deadlocks cannot occur; and PostgreSQL `now()` returns the transaction start time, so every row the test writes gets the same timestamp and ordering by `created_at` ties. Tests of commit-dependent behavior need committed data and separate connections: truncation between tests (Django `TransactionTestCase`). Details in `references/real-dependencies.md`.

**Mocked HTTP that does not match the real API.** A hand-written mock returns `{ id, status: "succeeded" }`. The real API nests the object, returns amounts in minor units, answers 200 with an error body, paginates, or sends `null` for fields the mock always fills. Tests pass and production parses garbage. Build fixtures from recorded real responses with secrets scrubbed, parse them through the same schema the production client uses, cover the error, 429, 5xx, timeout and malformed shapes, and fail on any unmatched request (MSW `onUnhandledFrame: "error"`, named `onUnhandledRequest` before MSW 3).

**Mocked driver returning the wrong types.** node-postgres returns `int8`/`bigint` columns as strings, because a JavaScript number cannot hold every 64-bit value. A mocked repository returning numbers hides this:

```ts
repo.getWallet.mockResolvedValue({ id: WALLET_ID, balance_cents: 500 });
// Real driver: balance_cents is "500", and the code under test does
const next = wallet.balance_cents + creditCents; // "500" + 100 === "500100"
```

Test data access against the real driver and engine. The same applies to decimals, dates and JSON columns in every language.

### Nondeterminism

**Shared state and order dependence.** Module-level variables carrying ids between tests, tests that depend on rows an earlier test created, fixed emails or ids that collide when workers share a database, cached settings, mutated env vars, mocks never reset. Symptom: passes alone and fails in the suite, or the reverse. Each test creates what it needs with unique values and restores global state, and CI runs in random order with the seed printed (pytest-randomly, Jest `--randomize`, Vitest `--sequence.shuffle`).

**Uncontrolled time.** `datetime.now()`, `Date.now()` and SQL `now()` in logic under test. The test passes except around midnight UTC, month end, DST changes, or on a runner in another zone, and expiry tests sleep for real. Inject a clock or use a fake one, pin the zone, and test the boundary explicitly: one unit before expiry, exactly at it, after it. Faking the process clock does not move the database clock; if SQL compares against `now()`, pass the time as a parameter. Templates in `references/determinism.md`.

**Uncontrolled randomness.** Random test data with no recorded seed, assertions on UUIDv4 order, set iteration order. Seed and print the seed. Keep counterexamples found by property tests as fixed regression cases.

**Sleeps instead of conditions.** `sleep(1)` then assert: flaky on a slow runner, slow everywhere else. Wait on the condition with a deadline (Playwright web-first assertions, `expect.poll`, `vi.waitFor`, a polling helper) or drive the async work directly: run the worker once, advance fake timers. In Playwright, `expect(await locator.isVisible()).toBe(true)` checks once; `await expect(locator).toBeVisible()` retries until the timeout.

**Flaky tests retried away.** Retries are on (Playwright `retries`, `jest.retryTimes`, pytest-rerunfailures) and nobody reads the flaky report. On a money or auth path, a test that fails one run in twenty is often the only signal of a real race. Reproduce with repetition (Playwright `--repeat-each`, a loop), find the cause and fix it. If retries stay for infrastructure noise, make flaky results fail the build on critical paths (Playwright `--fail-on-flaky-tests`) and quarantine only with an owner and an issue.

### Missing tests where it hurts

**Happy path only.** Every rejection branch of a sensitive operation needs a test: wrong owner, wrong state, over the limit, zero, negative, maximum and overflow amounts, unknown fields, expired token, duplicate request. Each asserts the exact rejection, unchanged state and no side effect.

**No concurrency test on money paths.** Transfers, withdrawals, redemptions, stock and anything guarded by a balance, limit or uniqueness check. Fire conflicting requests at the same moment against the real database with one connection each, then assert the invariant: exactly one success, balance never negative, totals conserved. For a reliable reproduction, pause one request between its read and its write with a hook and run the other to completion. Templates in `references/concurrency-and-idempotency.md`.

**Idempotency tested only as a sequential replay.** The full set: same key sequential returns the stored result with one side effect; same key concurrent gives one side effect; same key with a different body is rejected; same key from another user neither collides nor leaks; provider succeeded but the response was lost, and the retry does not charge twice.

**Missing negative authorization tests.** A matrix of caller (anonymous, expired token, owner, same-tenant non-owner, lower role, other tenant's admin) against every action, run at the API rather than by checking that the UI hides a button. Include list endpoints (no foreign rows), ids in the body, nested routes mixing tenants, and a test that fails when a new route is not in the matrix.

**Dependency failures untested.** Timeout after the provider succeeded, 5xx, 429, malformed body, slow response. The fake at the network edge must be able to return each, and the test asserts the operation ends in a known state (pending for reconciliation, not "failed" and retried blindly).

### Fixtures and environments

**Real secrets and PII in fixtures.** HTTP cassettes recorded with `Authorization` headers, production database dumps, `.env.test` holding live keys, committed Playwright `storageState` files with session cookies. Treat any committed live credential as compromised and rotate it. Scrub at record time (vcrpy `filter_headers`, `before_record_response`), take sandbox keys from the environment, and build data with factories.

**Tests that can reach production.** A base URL or key that falls back to a real endpoint when an env var is missing. Test config fails closed: required settings with no production default, and outbound network blocked except for explicit fakes.

**Property tests sharing a per-test fixture.** A pytest function-scoped fixture runs once per test function, not once per Hypothesis example, so a database row or counter leaks across generated inputs. Hypothesis flags this with the `function_scoped_fixture` health check; build per-example state inside the test body.

**Brittle E2E selectors.** `#root > div > div:nth-child(3) > button.btn-primary` or generated class names break on any layout change, and then the test gets deleted. Use `getByRole` with the accessible name, then `getByLabel`, then `getByTestId` for elements with no semantic role. Details in `references/e2e-playwright.md`.

## Decision rules

Test level:
- Pure logic with many cases (calculations, parsers, state machines): unit tests plus properties.
- Correctness depends on SQL, constraints, transactions, the driver, serialization or middleware: integration tests through the real router against the real engine. Most business bugs live here.
- A journey where breakage is expensive and spans the browser and several services: a handful of E2E tests, not one per feature.

Real or fake:
- You own it and it runs in a container: real.
- Third party over the network: fake at the HTTP edge using recorded real shapes, plus a scheduled check against the provider's sandbox.
- Clock and randomness: inject or fake, always.

Isolation:
- Rollback per test by default, for speed.
- Truncation or a fresh database when the test involves commit, `on_commit`, deferred constraints, locks, a second connection or concurrency.

Examples or properties:
- Fixed examples for known edges and every past bug.
- Properties when a rule holds for all inputs: round trip, conservation, idempotence, agreement with a simple model.
- Stateful or invariant testing when bugs need a sequence of operations: ledgers, state machines, contracts.

Flaky test: reproduce first. Fix the cause when it is in the code or the test. Quarantine with an owner only when the cause is outside the codebase.

Error assertions: type or machine-readable code, never full prose.

## Review checklist

- Does each guard on a money, auth or state path have a test that fails when the guard is deleted?
- Can every assertion actually execute on the failure path, and is every async expectation awaited?
- Is the code under test real, with mocks only at the process edge?
- Do assertions check observable outcomes (state, response, external side effect) rather than internal calls?
- Does every denial test have a positive control and assert unchanged state?
- Does the test harness run the real authentication and authorization path?
- Do integration tests use the production engine and major version, schema from real migrations, and the real driver?
- Are commit-dependent and concurrent behaviors tested with committed data and separate connections?
- Do HTTP fakes use recorded real shapes, include failure shapes, and reject unmatched requests?
- Are time, zone and randomness controlled, with boundaries tested explicitly?
- Are there no sleeps used as synchronization and no non-retrying checks of async UI?
- Does the suite pass in random order and in parallel?
- Are retries off for critical paths, or do flaky results fail the build?
- Do money paths have concurrency and full idempotency tests?
- Are fixtures free of live credentials and personal data?
- Does CI run the same number of tests as a local full run, with no `.only`?

## References

- `references/guard-checks.md`: read when checking whether tests cover a guard, or setting up Stryker, mutmut or cargo-mutants.
- `references/concurrency-and-idempotency.md`: read when writing race, double-submit, idempotency-key or lost-response tests, in TypeScript or Python.
- `references/authz-matrix.md`: read when writing or reviewing authorization tests for an API.
- `references/property-based.md`: read when writing Hypothesis, fast-check, proptest or Foundry invariant tests.
- `references/determinism.md`: read when a test depends on time, time zones, timers, randomness or order.
- `references/real-dependencies.md`: read when setting up database containers, choosing test isolation, or faking third-party HTTP.
- `references/e2e-playwright.md`: read when writing, reviewing or de-flaking Playwright tests.

Related skills: security-engineering for what the authorization and input tests must cover, database-engineering for the races and constraints the concurrency tests target, backend-architecture for idempotency and reconciliation design, performance-benchmarking for load and latency tests, solidity-engineering for Foundry fuzz, invariant and fork tests of smart contracts.
