# Deploying and upgrading on Polygon

Read this before deploying or upgrading a contract on Polygon PoS mainnet or Amoy. The commands use Foundry (`forge`, `cast`, `anvil`).

A deployment is a state change you cannot take back. Treat it like a database migration on a production system: planned, rehearsed, scripted, checked afterwards, and recorded.

## Build settings

```toml
[profile.default]
solc_version = "0.8.37"
evm_version = "osaka"
optimizer = true
optimizer_runs = 200
via_ir = true
extra_output = ["storageLayout"]

[rpc_endpoints]
polygon = "${POLYGON_RPC_URL}"
amoy = "${AMOY_RPC_URL}"
```

- Pin `solc_version` to a release with no known bugs that affect your settings, and check it against the compiler's `docs/bugs_by_version.json` before every deployment. 0.8.28 to 0.8.33 miscompile `delete` on transient variables under `via_ir` with a Cancun-or-later target (fixed in 0.8.34).
- Set `evm_version` explicitly to the newest fork Polygon has activated. Polygon and Amoy have run Osaka's only new opcode, `clz`, since the Lisovo hardfork (March 2026), so `osaka` fits today. Leaving it to the compiler default means a compiler upgrade can target a fork Polygon has not activated yet.
- Choose `optimizer_runs` for the contract: lower values give smaller code, higher values cheaper calls. Whatever you choose, tests, deployment and verification must all use the same settings.
- RPC URLs often contain an API key. Keep them in the environment, never in the repository.
- Use a dedicated RPC provider for deployments. Public endpoints rate-limit and drop transactions under load.

## Per-network configuration

Put every address and parameter the script needs in a file per network (`deployments/config/polygon.json`, `deployments/config/amoy.json`): multisig, timelock, token addresses, oracle feeds, heartbeat limits, fees, role holders. The script reads the file for the chain it is running on, checks that `block.chainid` matches the file, and reverts on any missing or zero value. Nothing environment-specific belongs in the Solidity source or the script body.

## Keys

- Use a hardware wallet (`--ledger --sender <address>`) or an encrypted keystore created with `cast wallet import deployer --interactive` and used with `--account deployer`.
- Never pass `--private-key`, never put a deployer key in `.env`, and never paste one into a terminal where shell history records it.
- The deployer key should hold only the POL needed for deployment and no lasting authority. Grant roles to the multisig and timelock in the deployment itself, then renounce or never hold them with the deployer, and check that on chain afterwards.

## Rehearsal

1. Run the full test suite, including fork tests, on the exact commit you will deploy.
2. Start a local fork and run the real script against it with broadcasting, then run the post-deploy check script against the fork:

   ```sh
   anvil --fork-url "$POLYGON_RPC_URL"
   forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --account deployer --broadcast
   forge script script/CheckDeployment.s.sol --rpc-url http://127.0.0.1:8545
   ```

3. Deploy to Amoy with the same script and settings, run the check script, verify, and exercise the main flows through the real frontend or backend.
4. Only then deploy to mainnet, from a clean checkout of the tagged commit.

## Gas on Polygon

- Mainnet requires a priority fee of at least 25 gwei. Transactions below it are not mined.
- The base fee moves quickly during congestion. Read current values from `https://gasstation.polygon.technology/v2` (Amoy: `/amoy`) or `eth_feeHistory`, and set an upper limit for the maximum fee from configuration so a spike cannot burn the deployer's balance.
- Pass the values explicitly for important deployments: `--priority-gas-price` for the tip and `--with-gas-price` for the maximum fee.

## Broadcasting

```sh
forge script script/Deploy.s.sol \
  --rpc-url polygon \
  --account deployer \
  --broadcast \
  --slow \
  --verify
```

- `--slow` sends each transaction only after the previous one is confirmed. Use it whenever later steps depend on earlier ones, which is nearly always.
- Initialize proxies in the same transaction that deploys them by passing the init call data to the proxy constructor. Never leave a window between deployment and initialization.
- If the script stops partway (a dropped connection, a stuck transaction, a gas spike), do not rerun it from the start: that deploys a second set of contracts. Check what reached the chain using `broadcast/<script>/<chain id>/run-latest.json` and `cast receipt <tx hash>`, then continue with `--resume`, or write a follow-up script for the remaining steps.
- One deployer address, one script at a time. Two processes sending from the same key fight over nonces.
- A transaction stuck below the market fee is replaced by sending a new transaction with the same nonce and higher fees, at least 10% above the old ones. Check `cast nonce <deployer> --block pending` against `cast nonce <deployer>` to see what is pending.

## Finality

A transaction in the latest block can still be reorganized. Since Heimdall v2, Polygon finalizes blocks within about 2 to 5 seconds through milestones. Treat the deployment as done only when every deployment transaction is in a block at or below `cast block finalized --rpc-url polygon`. Backends that react to your contracts should read events and state at the finalized tag for anything that moves money.

## Verification

With `--verify`, Foundry verifies each contract through the Etherscan API v2, which covers Polygonscan with one key from `ETHERSCAN_API_KEY`. For a contract deployed earlier:

```sh
forge verify-contract <address> src/Vault.sol:Vault \
  --chain 137 \
  --constructor-args "$(cast abi-encode 'constructor(address,uint256)' "$TOKEN" "$CAP")" \
  --watch
```

- Verification fails when compiler settings differ from the deployment. Verify from the same commit and `foundry.toml`.
- For proxies, verify the implementation and the proxy, then use the explorer's proxy detection so users read the implementation ABI through the proxy address.
- An unverified contract asks users to trust bytecode nobody can read. Do not announce an address before it is verified.

## Post-deploy check script

A script that reads everything back from the chain and compares it with the network configuration, reverting on the first mismatch:

- every role holder (`hasRole`), owner (`owner()`), pending owner and admin;
- that the deployer holds no roles;
- each proxy's implementation (`cast implementation <proxy>` or the ERC-1967 slot) and that implementations cannot be initialized again;
- every parameter: fees, caps, oracle addresses and staleness limits, token addresses, timelock delay;
- paused state;
- code size and code hash of each deployed contract against the build.

Run it after every deployment and every upgrade, on Amoy and mainnet, and keep its output with the deployment record.

## Upgrades

1. Diff the storage layout of the new implementation against the deployed one (`forge inspect <Contract> storageLayout`) and fail on any change other than appending in a namespaced struct.
2. Test the upgrade on a fork at a recent mainnet block with real state: deploy the new implementation, run the upgrade through the real path (timelock and multisig), then run the invariant and post-deploy checks.
3. Deploy the new implementation, verify it, and lock its initializers.
4. Schedule the upgrade through the timelock, publish the implementation address and diff, and execute after the delay.
5. Run the post-deploy check script and watch the contract's events and balances for a while afterwards.

## Records

Commit a deployment record per network: contract names, addresses, transaction hashes, block numbers, the commit hash, compiler version and settings, constructor arguments and the role holders. Either commit the `broadcast/` run files or generate that record from them in the script. Foundry writes the RPC URL of every transaction into the run files under `cache/`, and RPC URLs usually carry an API key, so `cache/` never goes into git.

## Mainnet checklist

- Tests, fuzz, invariants and fork tests pass on the tagged commit.
- Compiler pinned to a version with no known bugs for these settings, `evm_version` set explicitly to a fork Polygon has activated, settings identical to the tested build.
- Configuration file reviewed by a second person; chain id check in the script.
- Rehearsed on a local fork and on Amoy, with the check script passing.
- Deployer uses a hardware wallet or encrypted keystore and holds no lasting roles.
- Gas tip at least 25 gwei, maximum fee capped from configuration, `--slow` set.
- All transactions finalized, all contracts verified, check script passing on mainnet.
- Addresses, hashes and commit recorded in the repository.
