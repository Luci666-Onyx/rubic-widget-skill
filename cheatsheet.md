# Rubic Cheatsheet

## Decision: which integration surface?

| Need | Surface | Package / endpoint |
|---|---|---|
| Node/TS app, full control | SDK Lite | `@cryptorubic/sdk-lite` |
| Edge runtime (Workers/Deno/RN) | SDK + custom HttpClient | fetch-based client |
| Raw HTTP (any language) | REST API v2 | `api-v2.rubic.exchange` |
| AI agent (MCP host) | MCP server | `@cryptorubic/mcp` / `mcp-api-v2.rubic.exchange/mcp` |
| Zero-code UI | Browser redirect | `app.rubic.exchange/?from=...&to=...` |

## Decision: which swap method?

| Situation | Method |
|---|---|
| Show user a quote preview | `quoteBest` → `swap` |
| Compare all routes | `quoteAll` |
| No preview needed | `swapBest` |
| User can't sign on source chain (BTC/CEX) | `quoteDepositTrades` → `swapDepositTrade` |

## Decision: quote vs swap amount drift

| Deviation (swap vs quote) | Action |
|---|---|
| ≤ ±0.5% | Execute silently |
| > ±0.5% | Pause → confirmation UI → NEW swap request on approval |

## REST endpoints (quick list)

```
POST https://api-v2.rubic.exchange/api/routes/quoteBest
POST https://api-v2.rubic.exchange/api/routes/quoteAll
POST https://api-v2.rubic.exchange/api/routes/swap
GET  https://api-v2.rubic.exchange/api/info/chains?includeTestnets=false
GET  https://api-v2.rubic.exchange/api/info/status?srcTxHash=0x...
GET  https://api-v2.rubic.exchange/api/swagger/
```

## Error code decoder (6-digit = domain 3 + specific 3)

| Prefix | Domain | Common suffixes |
|---|---|---|
| 001 | PARAMS | 001 receiver, 002 equal tokens, 003 quote mismatch, 004 missing id, 005 missing field, 006 bad address |
| 002 | CALCULATION | 001 no routes, 002 chain down, 003 rate limit, 004 max amount, 005 min amount, 006 decimals, 007 no auth |
| 003 | SWAP | 001 need approve, 002 permit2, 003 low balance, 004 low native, 005 sim failed, 006 bad receiver, 100 arb hash |
| 004 | CONTRACT | 001 no direct routes, 002 no selector, 003 no contract, 004 unlisted |
| 1000s | SDK layer | 1001 liquidity, 1002 min amount, 1003 quote expired, 1004 bad pair, 1101/1102 bad address, 1103/1104 low balance |

## Status decision tree

```
waitForStatus resolves:
├── SUCCESS            → done (use destinationTxHash)
├── READY_TO_CLAIM     → show claim button → sdk.claim() → send
├── FAIL / REVERT / REVERTED → error UI (check refund object)
├── NOT_FOUND / INDETERMINATE → check explorer manually
└── Polling timeout (plain Error) → "taking too long" UI, not an API error

Deposit trades: poll with id only (no srcTxHash)
```

## Execution model → user action

| Returned shape | Model | User does |
|---|---|---|
| `{ to, data, value }` | EVM decentralized | wallet contract call |
| `{ to, data: 0x, value }` or deposit addr | EVM deposit | transfer to deposit address |
| native tx object | non-EVM decentralized | sign with chain's wallet (TON/Tron/Solana/Sui) |
| `{ depositAddress, amountToSend }` | non-EVM deposit | manual transfer, poll by id |

## Thresholds & defaults

| Setting | Default | Notes |
|---|---|---|
| SDK timeout | 10,000 ms | raise for congested networks |
| waitForStatus interval | 3,000 ms | 5–10s fine in practice |
| waitForStatus timeout | 300,000 ms | 1,800,000 for deposit trades |
| Amount-change threshold | ±0.5% | integrator's responsibility |
| showDangerousRoutes | false | hides priceImpact >20% / <-5%, slippage >20% |
| enableChecks | true | balance/gas validation |
| Slippage param | per quote | `1` = 1% |
| Rate limits (REST) | 200 req / 2hr (no key) | API key raises limits |
| 0% protocol fees | swaps < $100 | + most stable/native on-chain, USDT→TRON, all Solana |

## Tells & smells

| Signal | Likely problem |
|---|---|
| Error 1003 on swap | Quote expired — re-quote |
| Error 3001 / 3002 | Approval or permit2 needed — use `checkApprove` |
| Error 3004 | Not enough native token for gas |
| Status stuck PENDING > 30 min | Check explorer; LONG_PENDING is normal for bridges |
| `id` is null for a chain | Non-EVM chain (Solana/Tron/TON/Bitcoin) — don't cast to number |
| `proxyAvailable: false` | No fee collection via proxy contracts on that chain |
| Amount differs quote vs swap | Market/liquidity moved — apply the ±0.5% guard |
| MCP `WALLET_NOT_CONFIGURED` | EVM_WALLET_PRIVATE_KEY missing (read-only mode) |

## Key addresses & constants

| Constant | Value |
|---|---|
| Native token sentinel (SDK) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` |
| Native token zero (REST) | `0x0000000000000000000000000000000000000000` |
| Default crossChain integrator | `0x3fFF...DbE` |
| Default onChain integrator | `0x3b9C...0d4` |
| MCP key format | EVM private key WITHOUT `0x` |
| Solana chain id | 7565164 |
| Private RPC (ETH) | `https://rubic-eth.rpc.blxrbdn.com` |
| BD contact | `t.me/RubicPartnership` |
