# Testing smart contracts

Read this when writing or reviewing tests for contracts. The examples use Foundry and forge-std; the Hardhat section covers the differences for projects that use it.

A contract test suite has one job: make it impossible for a change to steal, lock or misaccount money without a test failing. Coverage percentages do not measure that. Ask of every test whether it would fail if the bug it guards against came back.

## Layers

| Layer | Finds | Tool |
| --- | --- | --- |
| Unit | wrong results and missing reverts for cases you listed | `function test_...` |
| Fuzz | wrong results for inputs you did not list: zero, one wei, max values, rounding edges | `function testFuzz_...(uint256 amount)` |
| Invariant | broken properties after sequences of calls by many actors | `invariant_...` with handlers |
| Fork | wrong assumptions about deployed tokens, oracles and routers on Polygon | `vm.createSelectFork` |
| Static analysis | known bug patterns, shadowing, unchecked calls | Slither, compiler warnings |

Money-moving contracts need every layer. A green unit suite with no invariants proves only the scenarios someone imagined.

## Unit tests that mean something

```solidity
function test_withdraw_revertsForNonOwnerOfPosition() public {
    uint256 positionId = _openPosition(alice, DEPOSIT_AMOUNT);

    vm.prank(bob);
    vm.expectRevert(Vault.NotPositionOwner.selector);
    vault.withdraw(positionId);
}

function test_withdraw_paysOwnerAndClearsPosition() public {
    uint256 positionId = _openPosition(alice, DEPOSIT_AMOUNT);
    uint256 balanceBefore = token.balanceOf(alice);

    vm.expectEmit(address(vault));
    emit Vault.Withdrawn(positionId, alice, DEPOSIT_AMOUNT);
    vm.prank(alice);
    vault.withdraw(positionId);

    assertEq(token.balanceOf(alice) - balanceBefore, DEPOSIT_AMOUNT);
    assertEq(vault.positionAmount(positionId), 0);
}
```

- Expect specific custom errors (`expectRevert(Error.selector)`, or `expectPartialRevert` when arguments vary), not a bare `expectRevert()` that passes for any failure, including the wrong one.
- Assert balance changes, emitted events and the state that must be cleared, not only that the call succeeded.
- Use `makeAddr("alice")` for actors and named constants for amounts, so failures are readable and nothing is a magic number.
- For every access-controlled function, test the allowed caller, a random caller and each other role. A role matrix test catches the function someone forgot to protect.

## Fuzz tests

```solidity
function testFuzz_depositThenWithdraw_neverReturnsMore(uint256 amount) public {
    amount = bound(amount, MIN_DEPOSIT, MAX_DEPOSIT);
    deal(address(token), alice, amount);

    vm.startPrank(alice);
    token.approve(address(vault), amount);
    uint256 shares = vault.deposit(amount, alice);
    uint256 assetsOut = vault.redeem(shares, alice, alice);
    vm.stopPrank();

    assertLe(assetsOut, amount);
}
```

- Use `bound()` to keep inputs in range instead of `vm.assume()`, which discards runs and can leave the fuzzer testing almost nothing.
- Include the edges deliberately in unit tests too: 0, 1, the minimum, the maximum, and values just around rounding boundaries.
- Raise `[fuzz] runs` in CI for money paths; keep the local default fast.

## Invariant tests

Invariant tests call your contracts in random sequences and check properties after every call. Without a handler, most calls revert on random input and the run proves little.

```solidity
contract VaultHandler is Test {
    Vault internal immutable vault;
    IERC20 internal immutable token;
    address[] internal actors;

    uint256 public ghostDeposited;
    uint256 public ghostWithdrawn;

    constructor(Vault vault_, IERC20 token_, address[] memory actors_) {
        vault = vault_;
        token = token_;
        actors = actors_;
    }

    function deposit(uint256 actorSeed, uint256 amount) external {
        address actor = actors[bound(actorSeed, 0, actors.length - 1)];
        amount = bound(amount, MIN_DEPOSIT, MAX_DEPOSIT);
        deal(address(token), actor, amount);

        vm.startPrank(actor);
        token.approve(address(vault), amount);
        vault.deposit(amount, actor);
        vm.stopPrank();

        ghostDeposited += amount;
    }

    function withdrawAll(uint256 actorSeed) external {
        address actor = actors[bound(actorSeed, 0, actors.length - 1)];
        uint256 shares = vault.balanceOf(actor);
        if (shares == 0) return;

        vm.prank(actor);
        ghostWithdrawn += vault.redeem(shares, actor, actor);
    }
}

contract VaultInvariants is StdInvariant, Test {
    Vault internal vault;
    IERC20 internal token;
    VaultHandler internal handler;

    function setUp() public {
        (vault, token) = _deploySystem();
        handler = new VaultHandler(vault, token, _actors());
        targetContract(address(handler));
    }

    function invariant_vaultHoldsWhatItOwes() public view {
        assertGe(token.balanceOf(address(vault)), vault.totalAssets());
    }

    function invariant_nothingLeavesThatDidNotComeIn() public view {
        assertLe(handler.ghostWithdrawn(), handler.ghostDeposited());
    }
}
```

- Ghost variables track what the system should know (total deposited, total withdrawn, used nonces), so invariants compare the contract against an independent model.
- Target the handler, not the raw contract, and restrict selectors with `targetSelector` when some handler functions are helpers.
- Add handler functions for admin actions, oracle moves (`vm.warp` past the heartbeat, a price change) and direct token transfers to the contract, so invariants are tested against donations and stale prices too.
- Configure `[invariant] runs` and `depth` in `foundry.toml`. Decide `fail_on_revert` deliberately: `true` catches handlers that call with invalid input by accident; `false` lets expected reverts happen.
- When an invariant fails, Foundry prints the call sequence. Turn it into a unit test so the bug stays fixed.

Useful invariants: assets cover liabilities; the sum of user balances equals the tracked total; shares and assets stay consistent within rounding; a nonce or signature is never accepted twice; only authorized roles change parameters; paused functions cannot move value.

## Fork tests on Polygon

```solidity
function setUp() public {
    vm.createSelectFork("polygon", vm.envUint("POLYGON_FORK_BLOCK"));
}
```

```toml
[rpc_endpoints]
polygon = "${POLYGON_RPC_URL}"
amoy = "${AMOY_RPC_URL}"
```

- Pin the fork to a block number so the test is deterministic and the RPC can cache it. Update the pin deliberately.
- Read real token, oracle and router addresses from configuration shared with deployment, not from literals scattered across tests.
- Fork tests show how your contract behaves with the real USDC, the real Chainlink feed and the real decimals. They are where fee-on-transfer surprises, stale feeds and unexpected reverts show up.
- Keep RPC URLs in environment variables. Use a provider with enough rate limit; public endpoints fail under a full fork suite.

## Signatures in tests

```solidity
(address signer, uint256 signerKey) = makeAddrAndKey("signer");
bytes32 digest = vault.hashClaim(claim);
(uint8 v, bytes32 r, bytes32 s) = vm.sign(signerKey, digest);
```

Test the replay cases explicitly: the same signature twice, after a nonce change, past the deadline, on a contract deployed at another address, and under a different chain id (`vm.chainId`). Assert that each one reverts.

## Gas and size

- `forge test --gas-report` shows gas per function; `forge snapshot` records it and `forge snapshot --check` fails CI when it changes unexpectedly.
- `forge build --sizes` shows each contract's size against the 24,576-byte limit.
- Test the worst case: the largest batch, the longest list a loop can see, the most positions a user can hold. If the worst case does not fit in a block, the function will fail in production.

## Coverage

`forge coverage --report lcov` shows lines no test reaches. With `via_ir`, coverage may need `--ir-minimum`, which changes code generation, so treat its numbers as a map of untested code, not proof of correctness. Uncovered branches in value paths are findings; covered lines are not evidence of good tests.

## Static analysis

Run Slither on every change and triage each finding: real, false positive with a reason, or accepted risk. Treat compiler warnings as errors in CI. Tools do not replace invariants; they catch the known patterns cheaply.

## Hardhat equivalents

| Foundry | Hardhat |
| --- | --- |
| `vm.prank`, `vm.startPrank` | connect a signer, or impersonate with `hardhat_impersonateAccount` |
| `vm.warp`, `vm.roll` | `time.increase`, `time.increaseTo`, `mine` from `@nomicfoundation/hardhat-network-helpers` |
| `vm.snapshotState`, `vm.revertToState` | `loadFixture` or `takeSnapshot` |
| `deal` | `setBalance` for native POL; token balances need a funded holder or storage writes |
| `vm.createSelectFork` | `forking` in the network config with a pinned `blockNumber` |
| `vm.expectRevert(Error.selector)` | `revertedWithCustomError(contract, "Error")` from the chai matchers |

Hardhat has no built-in invariant testing. For contracts that hold value, add a Foundry suite next to the Hardhat one rather than going without.
