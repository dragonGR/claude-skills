# Attack tests

Read this when checking the tests that came with an AI change, or when writing the tests an AI change is missing.

A test exists to fail when the behavior it names breaks. AI agents are very good at producing tests that pass: tests that check what the code does instead of what it should do, mocks that hand back the asserted value, expected values copied from the current output. The audit question for every test is the same: if someone broke this, would this test go red? The only convincing answer is to break it and watch.

## How faked green suites look

- **Expected values taken from the output.** The test was written or edited after running the code, so it asserts whatever the code returns, bugs included. Signs: oddly specific numbers with no derivation, snapshot updates in the same commit as the code, a commit that changes both the logic and the expectation.
- **Mocks that answer the question.** The repository, client or calculator under test is mocked to return exactly the value the assertion checks. The test exercises the mock.
- **Asserting calls instead of outcomes.** `toHaveBeenCalledWith(...)` on a debit, with no check that the balance changed or that a second debit is refused.
- **Assertions that never run.** An `expect` inside a callback that is never invoked, an async test whose promise is not awaited or returned, a `try` block in the test that catches the assertion error, a loop over an empty list.
- **Rejections that pass for the wrong reason.** `toThrow()` or `expectRevert()` with no specific error, so a typo, a missing fixture or an unrelated revert satisfies it. A 403 that comes from a missing auth header instead of the ownership check under test.
- **Harness that disables the guard.** A global fixture that bypasses authentication, a test config that turns off validation, a mocked clock that makes every token valid.
- **Weakened or removed tests.** Covered by the diff checks in [verification-commands.md](verification-commands.md); every removed assertion needs a reason.

## Prove each test fails without the change

Do this for every test that claims to cover a fix or a guard. Use worktrees, so the user's working tree is never touched.

**For a bug fix:** run the new test against the base.

```sh
base=$(git merge-base HEAD origin/main)
git worktree add /tmp/audit-base "$base"
cp path/to/new.test.ts /tmp/audit-base/path/to/new.test.ts
(cd /tmp/audit-base && npm ci && npx vitest run path/to/new.test.ts)
git worktree remove --force /tmp/audit-base
```

It must fail, and fail on the assertion that describes the bug. A failure from a missing import, a missing fixture or a function that does not exist yet on the base proves nothing; use the next method instead.

**For a new guard or a change in new code:** remove the guard and run the test.

```sh
git worktree add /tmp/audit-head HEAD
```

In `/tmp/audit-head`, delete or invert the guard the test claims to cover (the ownership condition, the nonce check, the limit, the `nonReentrant`), run the test and confirm it fails. Repeat for each guard. Then remove the worktree. A test that stays green with its guard gone is not a test of that guard; report it and write one that is.

Mutation testing tools automate this across a module. Setup and reading results are in the test-engineering skill.

## What an attack test looks like

An attack test plays the hostile caller and then checks two things: the attack was refused, and nothing changed.

### Another tenant's object

```ts
it("refuses another organization's invoice", async () => {
  const invoice = await createInvoice({ orgId: orgB.id, total: INVOICE_TOTAL });

  const res = await request(app)
    .get(`/orgs/${orgA.id}/invoices/${invoice.id}`)
    .set(authHeadersFor(memberOfOrgA));

  expect(res.status).toBe(404);
  expect(res.body).not.toHaveProperty("total");
});
```

The caller is authenticated and a real member of an organization, so a 404 here can only come from the ownership check, not from missing credentials.

### Two requests racing for one resource

```ts
it("redeems a single-use coupon once under concurrent requests", async () => {
  const coupon = await createCoupon({ maxUses: 1 });

  const results = await Promise.all(
    users.slice(0, CONCURRENT_ATTEMPTS).map((user) =>
      request(app).post(`/coupons/${coupon.code}/redeem`).set(authHeadersFor(user)),
    ),
  );

  expect(results.filter((r) => r.status === 200)).toHaveLength(1);
  expect(await countRedemptions(coupon.code)).toBe(1);
});
```

This only means something against the real database engine with real connections. A mocked repository or an in-memory store serializes the calls and the race never happens.

### Retry after an unknown outcome

```python
def test_retry_after_provider_timeout_charges_once(client, payment_provider):
    payment_provider.charge_then_time_out_once()
    headers = {"Idempotency-Key": IDEMPOTENCY_KEY}

    client.post(f"/orders/{ORDER_ID}/pay", headers=headers)
    retry = client.post(f"/orders/{ORDER_ID}/pay", headers=headers)

    assert payment_provider.charges_for(ORDER_ID) == 1
    assert retry.status_code in (HTTPStatus.OK, HTTPStatus.CONFLICT)
```

The fake provider must behave like the real one: it records the charge and then fails to answer. A fake that simply raises before charging tests the easy case.

### Replayed signature on a contract

```solidity
function test_claim_rejectsReplayedSignature() public {
    bytes memory signature = _signClaim(alice, CLAIM_AMOUNT, vault.nonces(alice), block.timestamp + CLAIM_TTL);

    vm.prank(alice);
    vault.claim(CLAIM_AMOUNT, block.timestamp + CLAIM_TTL, signature);
    uint256 balanceAfterFirstClaim = token.balanceOf(alice);

    vm.prank(alice);
    vm.expectRevert(Vault.InvalidSignature.selector);
    vault.claim(CLAIM_AMOUNT, block.timestamp + CLAIM_TTL, signature);

    assertEq(token.balanceOf(alice), balanceAfterFirstClaim);
}
```

Write the siblings too: the same signature on another chain id, on another deployment, after the deadline, and signed by a key that is not the authorized signer.

## Choosing the attacks

For each changed behavior, list who could misuse it and how, then write the test for each line of that list:

- **Identity:** anonymous, expired session, another user, another tenant, a lower role, a revoked role.
- **Input:** missing, empty, zero, negative, maximum, one past the maximum, wrong type, unknown fields, oversized, malformed encoding.
- **Time and order:** replay, out of order, after expiry, before activation, concurrent duplicates.
- **Dependencies:** timeout, error response, malformed response, success that arrives after the caller gave up, restart between steps.
- **Chain:** wrong caller, reentrant callback, manipulated price, stale oracle, donation to the contract, extreme amounts.

Assert outcomes and state: the refusal, the unchanged balance, the single row, the event that was not emitted. Never only that a function was called.

Test design in depth, including authorization matrices, concurrency harnesses, idempotency suites and property-based tests, is in the test-engineering skill.
