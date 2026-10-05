---
name: solidity-engineering
description: Solidity contracts with Foundry on Polygon PoS and other EVM chains: access control, token accounting, reentrancy, EIP-712 signatures, Keccak hashing, upgrades, oracles, gas and loops, fuzz, invariant and fork tests, deploy scripts, verification and finality. Load it before writing, reviewing, testing or deploying a contract or a script that calls one.
license: MIT
metadata:
  author: Alex Tsanis
---

# Solidity engineering

A deployed contract is public, permanent and holds money. Anyone can call every external function in any order, with any arguments, from a contract written to exploit yours, in the same block as a flash loan. Write as if that caller is already waiting, because on a chain with value they usually are. Most losses come from a small set of mistakes: a privileged function anyone can reach, accounting that trusts a balance, a signature that can be replayed, an external call made before state is updated, an upgrade path nobody locked. This skill lists them, shows the safe patterns, and covers testing and deployment on Polygon PoS.

Before changing anything, read `foundry.toml`, the pinned compiler, the OpenZeppelin version in `lib/` or `package.json`, the deploy scripts and the existing invariant tests. Work with the versions the project pins.

## Polygon PoS facts that change code and operations

| Fact | Consequence |
| --- | --- |
| Chain ids: 137 mainnet, 80002 Amoy | Bind every signature to the chain id; read RPC URLs and chain ids from configuration, never from code |
| Gas token is POL | Contracts and scripts that mention MATIC need review; native value is POL |
| Minimum priority fee of 25 gwei on mainnet | Transactions with a lower tip do not get mined; take fees from the Polygon Gas Station or `eth_feeHistory`, with a ceiling from configuration |
| Block time is set by governance: about 2 s until May 2026, then 1.75 s, and 1.5 s by July 2026 (PIP-86) | `block.number` is not a clock, and every block-count-to-time conversion broke at each change; use `block.timestamp` for time, and never for randomness |
| Deterministic finality in about 2 to 5 seconds through Heimdall v2 milestones, queried with the `"finalized"` block tag | Credit deposits, mark payouts done and react to events only from finalized blocks; anything newer can still be reorganized |
| Hardforks follow Polygon's own schedule, not Ethereum's: Pectra's EIP-7702 arrived with Bhilai (July 2025), Osaka's only new opcode, `clz`, with Lisovo (March 2026) | Set `evm_version` explicitly in `foundry.toml` to the newest fork the chain has activated (`osaka` covers Polygon and Amoy today). Solidity 0.8.31+ and Foundry 1.8 default to `osaka`, and a future compiler default can emit opcodes Polygon does not run yet; check Polygon's hardfork announcements before raising the target |
| No L2 sequencer | Chainlink's sequencer uptime check applies to rollups, not to Polygon PoS; staleness checks still apply |
| Verification through the Etherscan API v2 with one key for all chains | `forge verify-contract --chain 137` (or `--chain 80002` for Amoy) with `ETHERSCAN_API_KEY` |

## Failure catalogue

### Authority and roles

**Privileged function with no access check.** An `initialize`, `setOracle`, `mint`, `sweep`, `setFee` or `upgradeTo` reachable by anyone, often added late or copied from a test helper. Every state-changing external function needs a deliberate answer to "who may call this", enforced in the function. List them with `forge inspect <Contract> methodIdentifiers` and check each one.

**One key that can do everything.** The deployer or an operations hot wallet holds the role that can upgrade, mint, pause, change the oracle and move funds. One leaked key drains the contract. Split roles by power: fund movement, upgrades and role administration behind a multisig and a `TimelockController` or `AccessManager` delay; hot keys limited to bounded, rate-limited actions with fixed destinations. Use two-step transfers for admin roles (`Ownable2Step`, `AccessControlDefaultAdminRules`).

**`tx.origin` for authorization.** Any contract the owner interacts with can act as the owner. Use `msg.sender`.

**Treating an address with no code as unable to run code.** Under EIP-7702 (live on Polygon since Bhilai) an EOA can delegate to contract code and still sign transactions. `require(msg.sender == tx.origin)` therefore no longer keeps out reentrancy, flash-loan callbacks or batched calls; `code.length == 0` never did, because a contract has no code while its constructor runs; and `code.length > 0` does not prove an ordinary contract either: a delegated EOA has 23 bytes of code (`0xef0100` followed by the delegate's address). Protect with `nonReentrant` and explicit authorization, not with EOA checks.

**Pausing that cannot stop the damage.** A pause flag that does not cover every value-moving path, or a pauser that is the same key as the attacker's target. Decide what pausing stops, test it, and make sure unpausing needs more authority than pausing.

### Value accounting and tokens

**Accounting from `balanceOf(address(this))`.** Anyone can send tokens or POL directly to the contract and change share prices, unlock thresholds or the first deposit's share count (the ERC-4626 inflation attack). Track deposits in your own state. For vaults, use OpenZeppelin's ERC-4626 with a decimals offset, or seed the vault at deployment.

**Assuming tokens behave.** Fee-on-transfer and rebasing tokens deliver less than requested; some tokens return no boolean; some revert on zero transfers or block addresses. Use `SafeERC20` (`safeTransfer`, `safeTransferFrom`, `forceApprove`), measure the received amount as a balance difference when it matters, and allowlist the tokens a contract accepts.

**Rounding in the user's favor.** Shares minted rounded up, assets paid out rounded up, or division before multiplication. Repeated deposits and withdrawals of tiny amounts drain the pool. Round against the caller (`Math.mulDiv` with an explicit `Rounding`), multiply before dividing, and fuzz the boundaries.

**Narrowing casts.** `uint128(x)` silently truncates. Checked arithmetic does not cover casts. Use `SafeCast`, and keep `unchecked` blocks for loop counters and proven-safe arithmetic only.

**Push payments to many recipients.** One reverting recipient blocks everyone. Let recipients withdraw (pull), and never loop transfers over a user-sized list.

### External calls and reentrancy

**State updated after the call.** A token hook (ERC-777, ERC-721 and ERC-1155 receivers), a native transfer to a contract or a callback re-enters before balances change. Follow checks, effects, then interactions, and add `nonReentrant` to every function that shares the state, not only the one you were looking at. OpenZeppelin's `ReentrancyGuardTransient` uses transient storage, which Polygon supports. From OpenZeppelin 5.5, upgradeable contracts import `ReentrancyGuard` and `ReentrancyGuardTransient` from `@openzeppelin/contracts`: they are stateless, so `ReentrancyGuardUpgradeable` no longer exists.

**Read-only reentrancy.** Another protocol reads your view function (a price, a share rate) while your state is half updated in the middle of a call. Do not expose prices from state that is inconsistent during a call, or guard the views.

**Unchecked low-level calls.** `addr.call(data)` returns `false` on failure instead of reverting, and calling an address with no code succeeds. Check the return value, check `code.length` when the target must be a contract, and prefer typed interface calls or `Address.functionCall`.

**Untrusted return data and gas.** A malicious callee can return enormous data that costs you gas to copy, or a relayer can forward too little gas so an inner call fails while the outer one records success. Cap copied return data with an assembly call when calling untrusted addresses, and check `gasleft()` against the gas the inner call needs before making it.

### Signatures and hashing

**Signatures that replay.** `ecrecover(keccak256(abi.encodePacked(user, amount)))` with no chain id, no contract address, no nonce and no deadline works again on another chain, on a redeployed contract and on the next call. Use EIP-712 through OpenZeppelin's `EIP712` (which handles the domain including `chainId` and `verifyingContract`), one typed struct per action, a per-signer nonce consumed before any external call, and a deadline.

**Accepting garbage signatures.** Raw `ecrecover` returns `address(0)` on invalid input, so `recovered == signer` passes when `signer` was never set; malleable signatures defeat a "used signatures" map. Use `ECDSA.recover` (it rejects invalid and malleable signatures), key replay protection on nonces, and `SignatureChecker` when the signer may be a smart-contract wallet.

**`abi.encodePacked` collisions.** Packing two dynamic types (`abi.encodePacked(string a, string b)`) lets `("ab","c")` and `("a","bc")` hash the same. Use `abi.encode` for anything that is hashed for authorization.

**Off-chain code using the wrong hash.** Node's `crypto.createHash('sha3-256')` is NIST SHA-3. Solidity's `keccak256` is the original Keccak-256, which pads differently. Both return 32 bytes and nothing errors, so selectors, event topics, EIP-712 digests, storage slots and addresses computed off-chain with SHA3-256 never match. Use `keccak256` from viem or ethers, or `keccak_256` from `@noble/hashes`, and test against a known vector: Keccak-256 of empty input is `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`.

**Hashing text instead of bytes.** Off-chain code hashes the string `"0x1234..."` instead of the bytes it represents, or JSON instead of the ABI encoding the contract uses. Build digests with the same ABI encoding as the contract, using viem's `encodeAbiParameters` or `hashTypedData`, and assert equality against the contract in a test.

### Upgrades and storage

**Initializer left open.** An implementation without `_disableInitializers()` in its constructor, or a proxy deployed in one transaction and initialized in the next, lets anyone front-run the initializer and take admin. Pass the init call data to the proxy constructor in the same transaction, and lock implementations. OpenZeppelin 5.6 and later make `ERC1967Proxy` and `TransparentUpgradeableProxy` revert with `ERC1967ProxyUninitialized` on empty init data; older versions and hand-written proxies do not.

**Upgrade authority without control.** UUPS `_authorizeUpgrade` without an access check lets anyone replace the logic. Gate it with the highest authority you have, behind a delay.

**Storage layout changed between versions.** Reordered, removed or retyped state variables corrupt existing data after an upgrade. Use ERC-7201 namespaced storage, never reorder or remove, and diff `forge inspect <Contract> storageLayout` against a committed baseline in CI.

**Assuming immutables and constructors run for proxies.** Constructor code runs on the implementation, not the proxy. State a proxy needs belongs in the initializer; `immutable` values live in the implementation bytecode and change with every upgrade.

### Oracles, time and randomness

**Stale or bad prices.** `latestRoundData()` used without checking `updatedAt` against the feed's heartbeat, or a non-positive answer accepted. Check both, from configured limits, and decide what the contract does when the feed is stale: revert or pause, never use the stale value.

**Manipulable prices.** A spot price read from an AMM pool in the same transaction can be moved with a flash loan. Value collateral from a manipulation-resistant source (Chainlink, or a time-weighted average over a long enough window) and bound how fast it may change.

**Randomness from block values.** `block.timestamp`, `blockhash` and `block.prevrandao` are known or influenced by the block producer. Use Chainlink VRF or a commit and reveal scheme.

### Gas, loops and unbounded growth

Solidity has no memory leaks in the usual sense: memory is cleared after every external call. The equivalent failures are state and loops that grow with usage until a function can no longer run.

**Loops over user-sized arrays.** A function that iterates over all stakers, all proposals or all positions works in tests and runs out of gas after enough users join, locking withdrawals or payouts forever. Process in bounded batches with a cursor, use per-user accounting such as reward-per-share accumulators, and never require a full loop for a user to get their money out.

**Arrays and sets that only grow.** Every entry costs storage forever, and removal from the middle of an array is either expensive or breaks ordering. Use mappings for lookup, `EnumerableSet` only when you must enumerate, and design how entries are removed.

**Views that return everything.** `getAllPositions()` returns a growing array that eventually exceeds RPC limits, breaking the frontend and any contract that calls it. Paginate.

**Memory expansion in loops.** Building large `bytes` or arrays in memory inside a loop costs gas that grows faster than linearly. Size buffers once, or keep the work off-chain.

**Contracts near the code size limit.** EIP-170 caps deployed code at 24,576 bytes. Check with `forge build --sizes`. Lower `optimizer_runs`, move logic into libraries or split contracts; do not strip checks to fit.

### Compiler and build

**Floating pragma for deployed contracts.** `pragma solidity ^0.8.20` lets a different compiler build what you deploy than what you tested. Pin an exact version for contracts you deploy.

**Compiler version with a known bug.** Some compiler releases miscompile code under specific settings, and nothing warns at build time. Solidity 0.8.28 to 0.8.33 with `via_ir` and a Cancun-or-later target clear only one of two locations when a contract `delete`s a transient variable and also clears persistent storage (TransientStorageClearingHelperCollision, fixed in 0.8.34). Before pinning or deploying, look up the version in the compiler's `docs/bugs_by_version.json` and check each listed bug's conditions against your settings.

**Compiler target the chain does not support.** With no `evm_version` set, the compiler picks its own default, which tracks Ethereum mainnet. Set it to the newest fork the target chain has activated so tests, deployment and verification all use what the chain runs.

**Settings that differ between test, deploy and verification.** `via_ir`, optimizer runs and EVM version change the bytecode. Keep them in `foundry.toml`, deploy from a clean build of the tagged commit, and verify with the same settings, or verification fails and the deployed code is not the code you reviewed.

## Testing

Unit tests prove the cases you thought of. Fuzz tests try values you did not think of, and invariant tests try call sequences you did not think of. Money-moving contracts need all three, plus fork tests against real Polygon state when they integrate with deployed tokens, oracles or routers.

Write invariants first, in words: total shares match total assets within rounding, no user can withdraw more than they deposited plus earned, the sum of balances equals the tracked total, only the admin role can change parameters, a used nonce is never accepted twice. Then make them executable with handlers and ghost variables.

For every access check and every require that protects value, write the test that fails if the check is removed. If deleting a line does not break a test, that line is untested.

Foundry patterns, invariant handlers, fork tests and the Hardhat equivalents: [references/testing.md](references/testing.md).

## Deployment

Deployment is where a correct contract becomes an exploitable one: a proxy initialized late, a role granted to the wrong address, a constructor argument from the wrong environment, a verification that never ran.

- Deploy with a Foundry script, never by hand. Read addresses and parameters from a configuration file per network, and fail the script if any value is missing.
- Sign with a hardware wallet or an encrypted keystore (`--ledger`, `--account`), never a raw private key in an environment variable or shell history.
- Run the whole script against a fork of the target network first, then on Amoy, then on mainnet with the same commit and settings.
- After deployment, a check script reads every role, parameter, implementation address and owner back from the chain and compares them with the plan.
- Verify every contract and record addresses, transaction hashes and the commit in the repository.

Polygon gas settings, script flags, verification, key handling and the post-deploy checklist: [references/deployment.md](references/deployment.md).

## Decision rules

- **Upgradeable or immutable:** immutable when the logic is small and fully specified; upgradeable (UUPS, behind a timelock and multisig) when the system will change. Never upgradeable with a single key.
- **Roles:** `Ownable2Step` for a single owner; `AccessControlDefaultAdminRules` or `AccessManager` when several roles exist. Every role gets the least power that works.
- **Pull or push payments:** pull for user payouts; push only to a fixed, trusted address.
- **Signatures or transactions:** signatures (EIP-712) when a relayer submits for a user; otherwise let the user send the transaction.
- **On-chain or off-chain work:** on-chain only what needs trustless enforcement; enumeration, history and analytics come from events.
- **Finality:** credit, pay and notify only on finalized blocks.

## Review checklist

- Does every state-changing external function have an explicit, tested access rule?
- Can any single key move funds, upgrade or change roles without a multisig and delay?
- Is accounting based on internal state, not on token or native balances?
- Are token transfers wrapped with `SafeERC20`, and are only allowlisted tokens accepted?
- Does rounding favor the protocol, with multiplication before division and `SafeCast` for narrowing?
- Are state changes made before external calls, with `nonReentrant` on every function sharing the state?
- Are low-level call results checked, and is untrusted return data capped?
- Are signatures EIP-712 with chain id, verifying contract, nonce and deadline, recovered with `ECDSA` or `SignatureChecker`?
- Does off-chain code use Keccak-256 and the same ABI encoding as the contract, with a test proving it?
- Are implementations locked, proxies initialized in the deploy transaction, and storage layouts diffed in CI?
- Are oracle answers checked for staleness and sign, from configured limits?
- Is every loop bounded independently of the number of users, and is every growing list paginated?
- Is the compiler pinned to a version with no known bugs for these settings (`bugs_by_version.json`), `evm_version` set explicitly to a fork the chain has activated, and are build settings identical for test, deploy and verification?
- Does any guard rely on `tx.origin == msg.sender` or `code.length` to mean "EOA" or "contract"?
- Do unit, fuzz, invariant and (where relevant) fork tests cover every value-moving path, and does removing any guard fail a test?
- Was the deploy script run on a fork and on Amoy, and does a post-deploy check confirm every role and parameter on chain?

## References

- [references/contract-security.md](references/contract-security.md): read for a security review or audit of contracts, and for the full code templates: EIP-712 claims, role layouts, upgradeable proxies, oracle reads, vault and AMM accounting, governance and flash loans.
- [references/testing.md](references/testing.md): read when writing or reviewing contract tests; Foundry unit, fuzz, invariant and fork tests, cheatcodes, coverage and gas, and Hardhat equivalents.
- [references/deployment.md](references/deployment.md): read before deploying or upgrading on Polygon or Amoy; scripts, gas and nonce handling, keys, verification, finality and post-deploy checks.
