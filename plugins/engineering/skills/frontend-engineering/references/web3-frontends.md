# Web3 frontends

Read this when the UI connects a wallet, reads chain state, requests a signature or sends a transaction. Examples use viem actions, which wagmi hooks wrap. In wagmi v3, `useAccount` is `useConnection` and mutation hooks expose `mutate`/`mutateAsync`; in v2 they are `useAccount` and hook-specific names such as `writeContract` and `switchChain`. Check the installed major before writing hook code.

The contract and the API (the Worker) are the authorities. The wallet is a user-controlled signer that reports whatever it reports, and the RPC endpoint is a third-party service that can be slow, stale or wrong.

## RPC endpoints and keys

wagmi's `http()` transport falls back to the chain's public RPC when given no URL, and wagmi's docs recommend an authenticated RPC URL to avoid rate limits. In a static SPA that URL, key included, is in the bundle the moment it comes from `import.meta.env.VITE_*`. Anyone can copy it and spend your quota or get it suspended.

- Use a key the provider lets you restrict to your site's origins and to read methods, and treat it as public.
- For anything the restriction cannot cover, proxy reads through the Worker, which holds the real key as a secret and applies its own rate limit and method allowlist.
- Keep RPC URLs per chain in the validated config module, next to the contract address map, so a chain switch can never pair one chain's contracts with another chain's RPC.

## Chain mismatch

The wallet's chain and your app's target chain are independent. Users switch networks in the wallet at any time, including between "Approve" and "Deposit".

- Read the wallet's actual chain from the connection (`useConnection().chainId` in wagmi v3). The connection's `chain` object is `undefined` when the wallet is on a chain your config does not include, so do not use `chain?.id` as the check.
- Before every write, compare against the target chain id from your config and offer a switch. Disable the action while mismatched.
- Pass the target chain into the write itself. viem's `writeContract` and `sendTransaction` throw when the wallet's current chain differs from the `chain` parameter; wagmi's `chainId` parameter validates against the connected chain. That closes the gap where the user switches after your check.
- Contract addresses come from a per-chain map in config. The same address on another chain can be a different contract or nothing.
- Include `chainId` and the account address in every query key for chain data, and reset in-progress UI state when either changes.

## Wallet-reported state is not proof

A connected address tells you which account the wallet wants to use. It does not prove to the API that the user controls it, and a balance read in the browser does not prove the user holds anything.

Sign-in verified by the API (EIP-4361, "Sign-In with Ethereum"):

1. The Worker generates a single-use nonce (`generateSiweNonce` in `viem/siwe`), stores it against the pre-auth session and returns it.
2. The SPA builds the message with `createSiweMessage` including `domain`, `uri`, `address`, `chainId`, `nonce`, `issuedAt` and a short `expirationTime`, and asks the wallet to sign it.
3. The Worker calls `verifySiweMessage` on a public client with the expected `domain` and `nonce`. It checks expiry, handles smart contract wallets (ERC-1271, ERC-6492), and returns a boolean. The expected `domain` comes from the Worker's configuration, not from the message or the request.
4. On success the Worker consumes the nonce, then creates the session (an `HttpOnly` cookie, see `security-boundaries.md`) with the verified address.

When the wallet switches account, the SPA's session still belongs to the old address. Watch the connection, and on an address change either sign out or run sign-in again before any further API call that acts for the user.

Before:

```ts
const signature = await walletClient.signMessage({ account, message: 'Sign in to Acme' })
await fetch('/api/login', { method: 'POST', body: JSON.stringify({ address: account, signature }) })
```

That signature never expires, works on any site that asks for the same text, and can be replayed forever. Recovering the address with `ecrecover`-style logic also rejects every smart contract wallet.

Token gating, allowlists and "holders only" features re-read balances or ownership in the Worker from its own RPC, for the verified address, at the time of the privileged action.

## Signature requests people can read

Signing is the step where users lose funds to phishing. Make legitimate requests easy to tell apart from malicious ones.

- Use EIP-712 typed data (`signTypedData`) with a domain containing `name`, `version`, `chainId` and `verifyingContract`, and named fields. Wallets display the fields; a hex blob tells the user nothing.
- Never ask users to sign a raw hash. Wallets warn about it, and users who learn to click through the warning are the ones who get drained.
- The UI text next to the button states what the signature authorizes: which token, how much, to whom, until when. "Sign to continue" is not acceptable for anything that moves value.
- Permits (EIP-2612, Permit2) are token approvals. Request the exact amount for this operation and a short deadline, not the maximum value with no deadline.
- A signature that authorizes an off-chain order or a gasless action includes a nonce and deadline that the verifying contract or server enforces.

## Transaction lifecycle

A transaction hash means the wallet broadcast something. It does not mean the action happened.

```ts
type TxState =
  | { phase: 'idle' }
  | { phase: 'awaiting-wallet' }
  | { phase: 'rejected-by-user' }
  | { phase: 'not-sent'; error: unknown }
  | { phase: 'submitted'; hash: Hash }
  | { phase: 'succeeded'; hash: Hash; blockNumber: bigint }
  | { phase: 'reverted'; hash: Hash }
  | { phase: 'cancelled'; hash: Hash }
  | { phase: 'unknown'; hash: Hash }
```

```ts
import { BaseError, UserRejectedRequestError, WaitForTransactionReceiptTimeoutError, type Hash } from 'viem'

async function claim() {
  setTx({ phase: 'awaiting-wallet' })

  let hash: Hash
  try {
    const { request } = await publicClient.simulateContract({
      account,
      address: contracts[targetChain.id].rewards,
      abi: rewardsAbi,
      functionName: 'claim',
    })
    hash = await walletClient.writeContract({ ...request, chain: targetChain })
  } catch (error) {
    if (error instanceof BaseError && error.walk((e) => e instanceof UserRejectedRequestError)) {
      setTx({ phase: 'rejected-by-user' })
      return
    }
    setTx({ phase: 'not-sent', error })
    return
  }

  pendingTxStore.add({ hash, chainId: targetChain.id, account, intent: 'claim' })
  setTx({ phase: 'submitted', hash })

  let intentReplaced = false
  try {
    const receipt = await publicClient.waitForTransactionReceipt({
      hash,
      confirmations: chainConfig[targetChain.id].confirmations,
      onReplaced: (replacement) => {
        intentReplaced = replacement.reason !== 'repriced'
      },
    })

    if (intentReplaced) setTx({ phase: 'cancelled', hash })
    else if (receipt.status === 'reverted') setTx({ phase: 'reverted', hash })
    else setTx({ phase: 'succeeded', hash: receipt.transactionHash, blockNumber: receipt.blockNumber })

    pendingTxStore.remove(hash)
  } catch (error) {
    setTx({ phase: 'unknown', hash })
    if (!(error instanceof WaitForTransactionReceiptTimeoutError)) reportError(error)
  }
}
```

Points the code encodes:

- `publicClient` is created for `targetChain`, not for whatever chain the wallet reports, so simulation and receipt polling hit the right network.
- Simulate first. A revert found in simulation costs the user nothing; one found on chain costs gas. A simulation revert or any other failure before a hash exists ends in `not-sent`, so the UI never sits in `awaiting-wallet` after the wallet is gone.
- `receipt.status === 'reverted'` is a mined failure. The hash exists, the block exists, and the action did not happen.
- When the user speeds up or cancels in the wallet, viem calls `onReplaced` and resolves with the replacement's receipt. For `repriced` that is still your action. For `cancelled` (a zero-value self-transfer at the same nonce) and `replaced` (different data or value), the receipt usually says `success` but your action never ran. Checking only `receipt.status` shows "Claimed" for a cancelled claim.
- A timeout, or an RPC error while polling for the receipt, is an unknown outcome. The transaction may still be pending or may land later. Keep the hash, keep watching, link to the block explorer, and do not tell the user to resubmit: a second transaction can double the action or get stuck behind the first nonce.
- Persist pending hashes (keyed by chain id and account) and resume watching on load. A refresh must not lose a pending deposit.
- One confirmation can be reorged away. For balances and actions with real value, use the per-chain confirmation count from config, or wait for the finalized block, before showing a final state or enabling dependent steps. Show "confirming" in between.
- Invalidate chain reads (balances, allowances, positions) after success, since wallet and RPC caches will not do it for you. wagmi's read hooks keep their results in the TanStack Query cache, so invalidate them through the same `QueryClient` the rest of the app uses.

## Amounts

- On-chain amounts are `bigint` in base units. `Number()` silently loses precision above 2^53, which is under 0.01 of an 18-decimal token.
- Parse user input with `parseUnits(input, decimals)` after validating the string format; format for display with `formatUnits(value, decimals)`.
- Read `decimals` from the token contract or a verified token list. USDC has 6 on most chains; assuming 18 is off by a factor of 10^12.
- A "Max" button uses the exact bigint balance, and for the native gas token leaves room for fees.
- Display values from the chain are for display. The contract decides what the user actually receives; show the simulated result and a minimum-received or slippage bound for swaps.

## Review questions

- What happens if the wallet switches chain or account between the check and the write?
- Which privileged decision relies on an address or balance the browser reported?
- Could a user explain, from the wallet prompt alone, what they are signing and what it allows?
- Which UI state does a cancelled, replaced, reverted or timed-out transaction produce?
- Does a refresh during a pending transaction lose track of it?
- Which RPC URL and key ship in the bundle, and what can someone do with them outside your site?
- Does an account switch in the wallet end or redo the API session?
