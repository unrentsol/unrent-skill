# UNRENT agent skill

Teaches AI agents to reclaim overpaid rent from Solana token accounts with [UNRENT](https://unrent.xyz).

SIMD-0437 lowers Solana's `lamports_per_byte` from 6,960 to 696 in five steps. Existing accounts keep their old, higher deposit, and nothing is refunded automatically. UNRENT scans a wallet and returns the excess (`WithdrawExcessLamports`, `CloseAccount`). It is non-custodial: it builds unsigned transactions and the wallet signs them.

## Install

```bash
npx skills add unrentsol/unrent-skill
```

The skill uses the UNRENT MCP server (Streamable HTTP, no API key):

```bash
claude mcp add --transport http unrent https://unrent.xyz/mcp
```

| Tool | What it does |
| --- | --- |
| `get_rent_status` | Live `lamports_per_byte` and which SIMD-0437 steps are live |
| `scan_wallet` | Reclaimable SOL, fee, net, forecast, dust that could be burned (read-only) |
| `build_reclaim` | Unsigned, simulated transactions for the wallet to sign |
| `submit_reclaim` | Checks the signed transactions against the build and sends them |

See [skills/unrent/SKILL.md](skills/unrent/SKILL.md) for the workflow, signing examples and rules. Also served at https://unrent.xyz/skill.md.
