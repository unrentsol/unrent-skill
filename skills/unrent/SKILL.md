---
name: unrent
description: Reclaim overpaid rent from Solana token accounts with UNRENT. Use when a user asks to recover SOL locked in token accounts, close empty token accounts, clean up dust tokens, check how much rent a wallet overpays, or asks about SIMD-0437 / Solana's rent reduction (lamports_per_byte 6,960 → 696).
---

# UNRENT: reclaim Solana rent

SIMD-0437 lowers Solana's `lamports_per_byte` from 6,960 to 696 in five steps. Existing accounts keep their old, higher rent deposit, and nothing is refunded automatically. UNRENT scans a wallet and returns the excess:

- `WithdrawExcessLamports` on token accounts and mints: the account stays open, only the surplus comes back.
- `CloseAccount` on empty token accounts: the whole deposit comes back.
- Optional, only on the owner's explicit request: `BurnChecked` + `CloseAccount` for dust tokens.

UNRENT is non-custodial. It builds unsigned transactions; the wallet signs them. A service fee (shown in every scan, `fee.bps`) is the last instruction of each transaction.

## Connect

MCP server (Streamable HTTP, no API key): `https://unrent.xyz/mcp`

```bash
claude mcp add --transport http unrent https://unrent.xyz/mcp
```

Other clients: add a remote MCP server with that URL.

## Tools

| Tool | What it does | Side effects |
| --- | --- | --- |
| `get_rent_status` | Live `lamports_per_byte`, which SIMD-0437 steps are live, the next one | none |
| `scan_wallet` | Reclaimable SOL (excess, empty accounts, mint excess), fee, net, forecast per remaining step, burn candidates | none |
| `build_reclaim` | Re-scans and returns unsigned, simulated transactions (base64) plus a `claimId` | stores a pending claim |
| `submit_reclaim` | Checks the signed transactions against the build, then sends them | moves SOL on-chain |

## Workflow

1. `scan_wallet { wallet }`. Tell the user `reclaimable.totalSol`, `fee.sol` and `netSol`. If `totalSol` is 0, stop; the `forecast` shows what later steps will free.
2. Ask before going on. Never pass `burn` unless the user explicitly asked to burn specific tokens (burning destroys them; `burn.candidates` lists only priced tokens, `dust: true` means worth less than selling costs).
3. `build_reclaim { wallet }` (optionally `accounts`, `burn`, `burnMaxUsd`). The wallet needs a little SOL for network fees.
4. Sign every transaction in `transactions` with the wallet, in order, without changing the message. Do not send them yourself.
5. Within about 60 seconds: `submit_reclaim { claimId, signed }`. Report each `signature` and `explorer` link. On `expired: true`, build again.

If `summary.leftForNextClaim.accounts > 0`, run the flow again for the rest.

## Signing

`@solana/kit`:

```ts
import { getBase64EncodedWireTransaction, getBase64Encoder, getTransactionDecoder, signTransaction } from '@solana/kit';

const signed = await Promise.all(
	built.transactions.map(async (b64) => {
		const tx = getTransactionDecoder().decode(getBase64Encoder().encode(b64));
		return getBase64EncodedWireTransaction(await signTransaction([signer.keyPair], tx));
	})
);
```

`@solana/web3.js` 1.x:

```ts
import { VersionedTransaction } from '@solana/web3.js';

const signed = built.transactions.map((b64) => {
	const tx = VersionedTransaction.deserialize(Buffer.from(b64, 'base64'));
	tx.sign([keypair]);
	return Buffer.from(tx.serialize()).toString('base64');
});
```

A browser wallet (Wallet Standard `solana:signTransaction`) works the same way: sign only, do not sign-and-send.

## Rules

- Show the fee and the net amount before asking for a signature.
- Never ask for or handle a private key the user did not give you for this purpose.
- `submit_reclaim` refuses transactions that differ from the build (other instructions, other fee, other signer). Adding compute budget instructions is fine.
- Errors come back as plain text with the reason (rate limit, claims paused, simulation failed with program logs). Rate limits are per IP and minute; wait and retry.
- Referral: pass `ref` (an UNRENT referral code) to `scan_wallet` and `build_reclaim`. The referrer gets part of the service fee; the user pays the same.

More: https://unrent.xyz/how-it-works
