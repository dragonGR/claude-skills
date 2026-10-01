# Property-based and invariant tests

Read this when writing Hypothesis, fast-check, proptest or Foundry invariant tests, or when deciding whether a property test is worth it.

## Finding the property

Most code has at least one of these. Write the property in one sentence before writing the test.

- **Round trip.** `decode(encode(x)) == x`; parse then print then parse again gives the same value.
- **Conservation.** Money in equals money out plus balances; shares of a split sum to the total; no item appears in two buckets.
- **Idempotence.** Applying the operation twice equals applying it once (normalization, dedupe, retry handling).
- **Agreement with a model.** A simple, obviously correct implementation (a dict, a sorted list, a brute-force search) gives the same answer as the optimized one.
- **Never crashes.** A parser given arbitrary bytes returns a value or a typed error, never a panic, exception from the wrong layer, or hang.
- **Bounds.** A fee never exceeds the amount; a computed discount never makes a price negative; a pagination cursor always advances.

A property that restates the implementation (`total == sum(price * qty)` copied from the code) is a tautology and catches nothing.

## Pitfalls that make property tests pass while proving nothing

- **Generators too narrow.** `integers(min_value=0, max_value=100)` never reaches the overflow, the rounding edge or the limit. Generate across the full valid domain, and add explicit examples for known edges (zero, one minor unit, the cap, the cap plus one, the largest value the type holds).
- **Filtering instead of constructing.** Heavy `assume(...)` or `.filter(...)` throws away most inputs, and the tool gives up or tests only easy cases. Build valid inputs directly (generate a list, then derive a valid index from it).
- **Shared state across examples.** A pytest function-scoped fixture runs once per test function, not per generated example. Hypothesis reports this through the `function_scoped_fixture` health check; do not suppress it, create the state inside the test.
- **Discarded counterexamples.** A failure found once and not kept will not be retried after the database directory is cleaned in CI. Copy each found counterexample into an explicit example (`@example`, a plain unit test) and commit proptest's `proptest-regressions` files.
- **Non-reproducible CI failures.** Log the seed. Hypothesis defaults to `derandomize=True` on CI; fast-check prints a `seed` and `path` you can pass back to `fc.assert` to replay.

## Hypothesis

```python
from hypothesis import example, given, strategies as st

from app.money import split_evenly

MAX_AMOUNT_MINOR = 10**12
MAX_PARTS = 1_000


@given(
    total=st.integers(min_value=0, max_value=MAX_AMOUNT_MINOR),
    parts=st.integers(min_value=1, max_value=MAX_PARTS),
)
@example(total=1, parts=3)
@example(total=MAX_AMOUNT_MINOR, parts=MAX_PARTS)
def test_split_conserves_total_and_is_fair(total, parts):
    shares = split_evenly(total, parts)
    assert len(shares) == parts
    assert sum(shares) == total
    assert max(shares) - min(shares) <= 1
```

Stateful testing finds bugs that need a sequence of operations. The machine runs random sequences of rules against the real implementation and a simple model, checking invariants after every step:

```python
import pytest
from hypothesis import strategies as st
from hypothesis.stateful import Bundle, RuleBasedStateMachine, invariant, rule

from app.ledger import InsufficientFunds, Ledger, SameAccountTransfer

MAX_AMOUNT_MINOR = 10**9
amounts = st.integers(min_value=0, max_value=MAX_AMOUNT_MINOR)


class LedgerMachine(RuleBasedStateMachine):
    accounts = Bundle("accounts")

    def __init__(self):
        super().__init__()
        self.ledger = Ledger()
        self.model: dict[str, int] = {}

    @rule(target=accounts, opening=amounts)
    def open_account(self, opening):
        account_id = self.ledger.open(opening)
        self.model[account_id] = opening
        return account_id

    @rule(src=accounts, dst=accounts, amount=amounts)
    def transfer(self, src, dst, amount):
        if src == dst:
            with pytest.raises(SameAccountTransfer):
                self.ledger.transfer(src, dst, amount)
            return
        if self.model[src] < amount:
            with pytest.raises(InsufficientFunds):
                self.ledger.transfer(src, dst, amount)
            return
        self.ledger.transfer(src, dst, amount)
        self.model[src] -= amount
        self.model[dst] += amount

    @invariant()
    def balances_match_model(self):
        for account_id, expected in self.model.items():
            assert self.ledger.balance(account_id) == expected

    @invariant()
    def money_is_conserved(self):
        assert self.ledger.total() == sum(self.model.values())


TestLedger = LedgerMachine.TestCase
```

If `Ledger` talks to a database, each machine run needs a fresh schema or transaction created in `__init__` and torn down in `teardown()`, not a pytest fixture.

## fast-check

```ts
import fc from "fast-check";
import { expect, it } from "vitest";
import { splitEvenly } from "../src/money";

const MAX_AMOUNT_MINOR = 10n ** 18n;
const MAX_PARTS = 1_000;

it("split conserves the total and differs by at most one unit", () => {
  fc.assert(
    fc.property(
      fc.bigInt({ min: 0n, max: MAX_AMOUNT_MINOR }),
      fc.integer({ min: 1, max: MAX_PARTS }),
      (total, parts) => {
        const shares = splitEvenly(total, parts);
        expect(shares).toHaveLength(parts);
        expect(shares.reduce((a, b) => a + b, 0n)).toBe(total);
        const sorted = [...shares].sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));
        expect(sorted[sorted.length - 1] - sorted[0] <= 1n).toBe(true);
      },
    ),
  );
});
```

Use `fc.asyncProperty` for async code, and `fc.scheduler()` for race conditions inside one process (example in `concurrency-and-idempotency.md`).

## proptest

```rust
use proptest::prelude::*;

const MAX_WIRE_AMOUNT: u64 = 1 << 53;
const MAX_FRAME_LEN: usize = 4096;

proptest! {
    #[test]
    fn amount_round_trips_through_wire_format(minor in 0u64..=MAX_WIRE_AMOUNT) {
        let encoded = encode_amount(minor);
        prop_assert_eq!(decode_amount(&encoded).unwrap(), minor);
    }

    #[test]
    fn frame_decoder_never_panics(bytes in proptest::collection::vec(any::<u8>(), 0..MAX_FRAME_LEN)) {
        let _ = decode_frame(&bytes);
    }
}
```

Commit the `proptest-regressions` directory: it records failing inputs so they are retried first on later runs. For input that comes from the network, add a coverage-guided fuzzer (cargo-fuzz) as well; random generation rarely reaches deep parser states.

## Foundry invariant tests

Invariant tests call random sequences of functions on target contracts and check `invariant_*` functions after each call. Without a handler, the fuzzer spends most calls on reverts (zero amounts, unfunded senders) and the run proves little. A handler bounds inputs to meaningful values, pranks as a set of actors and records ghost variables for the invariants.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {Vault} from "../src/Vault.sol";

contract VaultHandler is Test {
    uint256 internal constant MAX_DEPOSIT = 1_000_000 ether;

    Vault internal immutable vault;
    address[] internal actors;

    uint256 public ghostDeposited;
    uint256 public ghostWithdrawn;

    constructor(Vault vault_, address[] memory actors_) {
        vault = vault_;
        actors = actors_;
    }

    function deposit(uint256 actorSeed, uint256 amount) external {
        address actor = actors[bound(actorSeed, 0, actors.length - 1)];
        amount = bound(amount, 1, MAX_DEPOSIT);
        vm.deal(actor, amount);
        vm.prank(actor);
        vault.deposit{value: amount}();
        ghostDeposited += amount;
    }

    function withdraw(uint256 actorSeed, uint256 amount) external {
        address actor = actors[bound(actorSeed, 0, actors.length - 1)];
        uint256 balance = vault.balanceOf(actor);
        if (balance == 0) return;
        amount = bound(amount, 1, balance);
        vm.prank(actor);
        vault.withdraw(amount);
        ghostWithdrawn += amount;
    }

    function actorCount() external view returns (uint256) {
        return actors.length;
    }

    function actorAt(uint256 index) external view returns (address) {
        return actors[index];
    }
}

contract VaultInvariantTest is Test {
    uint256 internal constant ACTOR_COUNT = 3;

    Vault internal vault;
    VaultHandler internal handler;

    function setUp() public {
        vault = new Vault();
        address[] memory actors = new address[](ACTOR_COUNT);
        for (uint256 i = 0; i < ACTOR_COUNT; i++) {
            actors[i] = makeAddr(string.concat("actor", vm.toString(i)));
        }
        handler = new VaultHandler(vault, actors);
        targetContract(address(handler));
    }

    function invariant_assetsMatchNetDeposits() public view {
        assertEq(address(vault).balance, handler.ghostDeposited() - handler.ghostWithdrawn());
    }

    function invariant_liabilitiesCoveredByAssets() public view {
        uint256 owed;
        for (uint256 i = 0; i < handler.actorCount(); i++) {
            owed += vault.balanceOf(handler.actorAt(i));
        }
        assertLe(owed, address(vault).balance);
    }
}
```

Configure the campaign in `foundry.toml` under `[invariant]` (`runs`, `depth`, `fail_on_revert`). With a handler that bounds its inputs, set `fail_on_revert = true`: any revert is then a bug in the handler or the contract rather than noise. Use `bound()` rather than `vm.assume()` for ranges; assume discards runs. An `afterInvariant()` function runs once at the end of each run, for checks too expensive to run after every call.

Watch for handlers that return early most of the time (the `balance == 0` branch above on a fresh vault). Count calls per branch in ghost variables and check that the interesting branches actually ran. An invariant that held over a thousand runs that never withdrew proves nothing about withdrawals.

Strict equality on contract balance breaks as soon as the contract can receive value the handler did not track (forced sends, donations, fee accrual). Choose the invariant that reflects the real guarantee: solvency (`>=`) is usually the one that matters.
