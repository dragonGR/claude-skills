# Contract security review

Read this for a security review of contracts, or when writing signed-message schemes, upgradeable proxies, role designs, oracle integrations and vault or AMM accounting. It has the full code templates behind the failure catalogue in SKILL.md. Examples use OpenZeppelin Contracts 5.x; check the installed version's docs before copying an API, since names moved between 4.x and 5.x. Chainlink's sequencer uptime check in the oracle template applies only on rollups; on Polygon PoS leave the sequencer feed unset.

## Review order

1. Write the invariants first: assets in equal liabilities out plus fees; shares never worth more than their backing; each signed authorization used at most once; only role X can move funds; no action while the oracle is stale.
2. Enumerate every external and public function, fallback, receive hook, callback (token hooks, flash loan receivers, bridge messages), upgrade path and privileged role. For each role, write down which key holds it (EOA, multisig, timelock, contract) and what it can do in the worst case.
3. For each external call, check state is final before the call, which functions share that state, and what a reentrant or reverting callee can do.
4. For each value computation, check units, decimals, rounding direction and who can move the inputs within one transaction.
5. Test invariants with fuzzing against hostile tokens, reordered calls, manipulated prices and role changes, and fork tests against the real deployed dependencies.

## Signed messages

The replayable version:

```solidity
// Before: no chain id, no contract address, no action type, and raw ecrecover.
function claim(uint256 amount, uint256 nonce, uint8 v, bytes32 r, bytes32 s) external {
    bytes32 hash = keccak256(abi.encodePacked(msg.sender, amount, nonce));
    require(ecrecover(hash, v, r, s) == signer, "bad sig");
    require(!used[nonce], "used");
    used[nonce] = true;
    token.transfer(msg.sender, amount);
}
```

Problems: the same signature works on every chain and every deployment sharing the signer; if `signer` is ever `address(0)`, any invalid signature passes; a global nonce space lets one user's claim block another's; `transfer` ignores tokens that return `false`.

```solidity
import {EIP712} from "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import {ECDSA} from "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract RewardClaims is EIP712 {
    using SafeERC20 for IERC20;

    error Expired();
    error BadSignature();
    error ZeroAddress();

    bytes32 private constant CLAIM_TYPEHASH =
        keccak256("Claim(address account,uint256 amount,uint256 nonce,uint256 deadline)");

    IERC20 public immutable rewardToken;
    address public immutable claimSigner;
    mapping(address account => uint256) public nonces;

    constructor(IERC20 token, address signer_) EIP712("RewardClaims", "1") {
        if (address(token) == address(0) || signer_ == address(0)) revert ZeroAddress();
        rewardToken = token;
        claimSigner = signer_;
    }

    function claim(uint256 amount, uint256 deadline, bytes calldata signature) external {
        if (block.timestamp > deadline) revert Expired();
        uint256 nonce = nonces[msg.sender]++;
        bytes32 digest = _hashTypedDataV4(
            keccak256(abi.encode(CLAIM_TYPEHASH, msg.sender, amount, nonce, deadline))
        );
        if (ECDSA.recover(digest, signature) != claimSigner) revert BadSignature();
        rewardToken.safeTransfer(msg.sender, amount);
    }
}
```

Why each piece is there:

- The domain (`name`, `version`, `chainId`, `verifyingContract`) binds the signature to this contract on this chain. OpenZeppelin's `EIP712` recomputes the separator if the chain id changes after deployment (a fork), which a hand-cached immutable separator does not.
- One typehash per action. Two functions that accept structurally identical messages under the same domain can be fed each other's signatures.
- `ECDSA.recover` reverts on invalid signatures and rejects high-s malleable values. Raw `ecrecover` returns `address(0)`.
- The nonce is per account and consumed before the transfer. Sequential nonces require the backend to issue in order; if it cannot, use a per-account bitmap of used nonces.
- `block.timestamp` for a deadline measured in minutes or hours is fine; validators can shift it by seconds, not hours.
- If signers may be smart contract wallets (Safe, ERC-4337 accounts), use `SignatureChecker.isValidSignatureNow`, which supports ERC-1271.
- `abi.encodePacked` with two or more dynamic types (`string`, `bytes`, dynamic arrays) is ambiguous (`("ab","c")` and `("a","bc")` hash the same). Use `abi.encode`.

Off-chain signing services are part of this boundary: the backend deciding what to sign must derive `amount` from its own records, rate-limit per account, and use a key held in an HSM or KMS, not a file on the API server.

**Permit griefing.** A contract that calls `token.permit(...)` then `transferFrom` in one function reverts if someone front-runs the permit (the nonce is spent). Wrap the permit call in `try/catch` and proceed if the allowance is already sufficient.

## Roles and privileged keys

For each role, answer: what is the most value this key can move or destroy in one transaction, and how long would it take to notice and respond? Then:

- Hot keys (held by servers for automation) get narrowly bounded powers: update a price within a band, pause, push rewards up to a per-epoch cap. They never hold `DEFAULT_ADMIN_ROLE`, upgrade rights, mint rights or unbounded sweep.
- Fund-moving, upgrade and role-admin powers sit behind a multisig with independent signers and a timelock long enough for users to exit. Pausing can be faster than unpausing.
- Rescue functions send only to a fixed treasury address set through the timelock, and exclude the tokens the contract holds on behalf of users.
- Role handover is two-step (`Ownable2Step`, `AccessControlDefaultAdminRules`) so a typo does not burn admin.
- Never authorize with `tx.origin`.

Bridge and custody losses (Ronin, Harmony in 2022) came from a small set of keys controlling everything. The review question is how many keys, held by whom, and on what infrastructure.

## Upgradeable contracts

```solidity
contract Vault is UUPSUpgradeable, AccessControlUpgradeable {
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address admin, address upgrader) external initializer {
        __AccessControl_init();
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(UPGRADER_ROLE, upgrader);
    }

    function _authorizeUpgrade(address) internal override onlyRole(UPGRADER_ROLE) {}
}
```

Deployment must initialize atomically:

```solidity
// Before: two transactions; anyone watching the mempool calls initialize first.
// OpenZeppelin 5.6.0+ reverts here with ERC1967ProxyUninitialized unless
// _unsafeAllowUninitialized() is overridden; older versions deploy it.
ERC1967Proxy proxy = new ERC1967Proxy(address(impl), "");
Vault(address(proxy)).initialize(admin, upgrader);

// After: initializer runs inside the proxy constructor.
ERC1967Proxy proxy = new ERC1967Proxy(
    address(impl),
    abi.encodeCall(Vault.initialize, (admin, upgrader))
);
```

Also check:

- `_disableInitializers()` in every implementation constructor. OpenZeppelin warns that an uninitialized proxy or implementation can be taken over.
- `_authorizeUpgrade` has an access modifier, and the upgrader is the timelock or multisig.
- Storage layout is append-only across versions; run the OpenZeppelin upgrades plugin validation or a storage layout diff in CI. Namespaced storage (ERC-7201) in 5.x removes most gap bookkeeping but not the need to diff.
- New initialization logic in an upgrade uses `reinitializer(n)` with a version number never used before, called in the same transaction as the upgrade (`upgradeToAndCall`).
- The Nomad bridge loss in 2022 followed an upgrade whose initialization marked the zero root as trusted, after which unproven messages passed verification. Test initializers and upgrades against the invariants, not only for "does not revert".

## Oracles

```solidity
error SequencerDown();
error SequencerGracePeriod();
error InvalidPrice(int256 answer);
error StalePrice(uint256 updatedAt);

function _readPrice() internal view returns (uint256) {
    if (address(sequencerUptimeFeed) != address(0)) {
        (, int256 status, uint256 startedAt,,) = sequencerUptimeFeed.latestRoundData();
        if (status != 0) revert SequencerDown();
        if (block.timestamp - startedAt <= sequencerGracePeriod) revert SequencerGracePeriod();
    }
    (, int256 answer,, uint256 updatedAt,) = priceFeed.latestRoundData();
    if (answer <= 0) revert InvalidPrice(answer);
    if (block.timestamp - updatedAt > maxPriceAge) revert StalePrice(updatedAt);
    return uint256(answer);
}
```

- `maxPriceAge` is per feed, set from that feed's documented heartbeat plus a margin, and changed only through governance. One constant for all feeds is wrong for most of them.
- On L2s, Chainlink's sequencer uptime feed answers 0 when up and 1 when down; after it comes back, wait a grace period before trusting prices, because users could not act while it was down.
- `answeredInRound` is deprecated in Chainlink's API reference; do not build checks on it.
- Normalise decimals from `priceFeed.decimals()` and the token's decimals explicitly. Mixing an 8-decimal price with an 18-decimal amount is a classic value bug.
- Spot prices from an AMM pool (`getReserves`, `slot0`) move within one transaction under a flash loan. Never use them to value collateral or mint against. A TWAP over a window longer than the attack's cost horizon, or an external oracle with a deviation check, is the minimum.
- Decide what happens when the oracle reverts: liquidations and withdrawals need a defined behaviour, not a frozen protocol.

## Token behaviour to assume

- `transfer` and `transferFrom` may return `false`, return nothing, or revert. Use `SafeERC20`.
- Received amount can be lower than requested (fee-on-transfer) or change over time (rebasing). Measure the balance delta when crediting deposits, or explicitly refuse such tokens.
- Tokens can pause, blacklist addresses and have callbacks (ERC-777 hooks, ERC-721/1155 `onReceived`). A blacklisted recipient must not block other users' withdrawals: use pull payments.
- `approve` to a non-zero allowance from non-zero reverts on some tokens; use `forceApprove`.

## Vault and AMM accounting

| Risk | What goes wrong | Control |
| --- | --- | --- |
| First-depositor inflation | Attacker mints 1 share, donates assets directly, next depositor's shares round to 0 | Internal accounting instead of `balanceOf(this)`, or virtual shares and assets (OpenZeppelin ERC4626 decimals offset); test donation before first deposit |
| Rounding direction | Rounding in the user's favour on both mint and redeem drains the vault in dust-sized loops | Round against the caller: down on shares minted and assets withdrawn, up on shares burned and assets required |
| Read-only reentrancy | Another protocol reads your share price during a callback while reserves are half-updated | Update state before callbacks, and guard view functions used as prices or expose a reentrancy status |
| Missing health check | A function that changes a user's collateral or debt skips the solvency check (Euler's `donateToReserves` in 2023) | Every function touching collateral or debt ends with the same solvency check; fuzz with that invariant |
| Slippage and deadline | Swaps and deposits with no minimum out or deadline are sandwiched | Caller passes `minOut` and `deadline`; contract enforces both |

## Governance and flash loans

- Voting weight comes from a snapshot taken before the proposal (`ERC20Votes` checkpoints). Weight read at vote time can be borrowed with a flash loan, which is how Beanstalk's governance was taken in 2022.
- Execution goes through a timelock; emergency paths are narrow and cannot move funds.
- Flash loan receivers verify `msg.sender` is the expected lender and that the initiator is this contract, and bind callback parameters to what the initiating call intended.

## Test matrix

- Unit tests per state transition and per role, including a caller without the role.
- Invariant tests (Foundry or Echidna) for conservation of assets, share price monotonicity where expected, single use of each signature, and no fund movement by non-privileged callers.
- Fuzzing around zero, one wei, maximum values, rounding boundaries, and odd tokens (fee-on-transfer, no return value, 6 and 24 decimals).
- Fork tests against the real deployed tokens, oracles and routers the protocol depends on.
- Static analysis (Slither) triaged by hand; each suppressed detector has a written reason.
- An upgrade rehearsal on a fork: deploy the new implementation, upgrade through the real timelock path, and rerun the invariant suite against the upgraded state.
