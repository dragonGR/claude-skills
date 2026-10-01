# Guard checks and mutation testing

Read this when you need to show that tests actually cover a guard (an ownership check, status check, limit, signature or replay check), or when setting up a mutation testing tool.

## The manual procedure

Use it on every guard added or changed in a diff that touches money, authorization or durable state. It takes minutes and catches the most common false comfort in a test suite.

1. List the guards. For each, write down the condition and the outcome it prevents: "`invoice.tenant_id != session.tenant_id` returns 404; prevents cross-tenant reads".
2. Run the relevant tests once on the unmodified code and confirm they pass. A red baseline makes the rest meaningless.
3. Break one guard at a time. Pick the mutation that matches the bug you fear:
   - delete the check, or replace its condition with `false` so it never rejects;
   - flip a comparison at the boundary: `>=` to `>`, `<` to `<=`;
   - drop one clause of a compound condition (`status == PENDING and owner == user` to just `status == PENDING`);
   - remove the `WHERE` guard from a conditional `UPDATE`, or ignore its affected-row count;
   - delete the `await` on a check that returns a promise (a floating promise is truthy and never throws in time).
4. Run the tests again. At least one test must fail, and it must fail on an assertion about that guard's outcome, not on an unrelated setup error. Read the failure message.
5. If nothing fails, or only an unrelated test fails, write the missing test first, watch it fail against the broken guard, restore the guard, watch it pass.
6. Restore the code and confirm the working tree matches the original before moving on. Never leave a mutation behind.

Report the result per guard: which test caught it, or which test you added. "Tests pass" without this is not evidence that a guard is covered.

Boundary mutations deserve special attention on money paths. A limit test with amounts of 10 and 10,000 against a cap of 1,000 survives `>` versus `>=`. The test needs the cap itself, the cap plus one minor unit and the cap minus one.

## Mutation testing tools

Tools automate the same idea across a module. They are slow on large code bases, so point them at the modules that move money or decide access, and run them on changed files in CI rather than on everything.

JavaScript and TypeScript, Stryker:

```bash
npm init stryker@latest
npx stryker run
```

Restrict the `mutate` option in `stryker.config.mjs` to the sensitive modules, and set `thresholds.break` so the run fails when the mutation score drops below the level you have reached.

Python, mutmut:

```toml
[tool.mutmut]
source_paths = ["src/billing/"]
```

```bash
mutmut run
mutmut browse
```

Rust, cargo-mutants:

```bash
cargo mutants
cargo mutants -f src/ledger.rs
```

`--in-diff DIFF_FILE` limits the run to mutants in the lines a diff changes (for example `git diff` against the base branch, saved to a file), which is the practical CI setting. cargo-mutants needs a test suite that is not flaky; a flaky test shows up as noise in the results.

## Reading the results

- **Caught** (killed): some test failed. Good.
- **Missed** (survived): no test failed. Either a test is missing, the assertion is too weak, or the mutant is equivalent.
- **Unviable** or compile error: the mutant did not build. Ignore.
- **Timeout**: the mutant made the code loop or hang. Usually counts as caught, but check that the test would have failed with a clear message.

An equivalent mutant changes the code without changing behavior (for example `i < len` to `i != len` in a loop where both stop at the same place). Mark it and move on; do not write a test that pins implementation to kill it.

For each surviving mutant in a sensitive module, ask what bug it represents in production terms. "Removing `amount > 0` survives" means a zero or negative transfer is untested; write that test. A survivor in logging or formatting code is usually fine to leave.

## What not to do

- Do not chase a mutation score across the whole code base. Glue code with surviving mutants costs nothing; one surviving mutant in the refund path can cost a lot.
- Do not kill mutants by asserting on internal calls or exact SQL. That turns a missing behavioral test into an implementation-pinned one.
- Do not run mutation tools against a suite that uses retries. A retried flaky test hides the mutant it should have caught.
