---
name: rubic-widget
description: "Knowledge base from the Cryptorubic/docs repository (Rubic SDK Lite, API v2, MCP server, monetization). Use when integrating Rubic cross-chain swaps, quoting routes, executing swaps, tracking status, handling errors, or building AI-agent swap flows."
---

<!-- argument-hint: [topic, method name, or 'ch01'..'ch08'] -->

# Rubic — SDK, API & MCP
**Author**: Cryptorubic | **Source**: docs.rubic.finance (Cryptorubic/docs) | **Chapters**: 8 | **Generated**: 2026-10-12

## How to Use This Skill

- **Without arguments** — load the core flow and decision rules
- **With a topic** — ask about `quoteBest`, `waitForStatus`, `RubicApiError`, `integratorAddress`, `MCP`, etc.
- **With a chapter** — ask for `ch01` .. `ch08`; I load that chapter
- **Browse** — ask "what chapters do you have?" to see the full index

---

## Core Flow (memorize this)

```
SDK Lite (@cryptorubic/sdk-lite):
  quoteBest / quoteAll / quoteDepositTrades   →  route + estimate + id
  swap / swapBest / swapDepositTrade          →  { to, data, value } or { depositAddress, amountToSend }
  waitForStatus / getStatusExtended           →  terminal status (SUCCESS, READY_TO_CLAIM, FAIL...)
  allowance / checkApprove / claim / celerRefund
  getChains / getProviders / healthcheck

REST v2 (raw HTTP):
  POST api-v2.rubic.exchange/api/routes/quoteBest | quoteAll | swap
  GET  api-v2.rubic.exchange/api/info/chains | status?srcTxHash=...
```

**The SDK never broadcasts.** It builds calldata; your signer sends it.

## Decision Rules (the essentials)

- **Native token**: SDK sentinel `0xEeee...EEeE`; REST zero `0x0000...0`
- **Quote vs swap amount drift > ±0.5%** → pause, confirm with user, re-issue swap (integrator's responsibility — API never signals it)
- **Quote expired (error 1003)** → re-quote, don't retry swap with the old id
- **Approvals**: approve against `swapData.transaction.approvalAddress` — it's provider-specific, never hardcode the router
- **Deposit trades**: poll status with `id` ONLY (no srcTxHash)
- **READY_TO_CLAIM** → Arbitrum Bridge; call `sdk.claim()` then user signs — never auto-complete
- **Error codes**: 6-digit = domain(3) + specific(3). 2001 = no routes; 3001 = need approve; 3004 = low native; 1003 = quote expired
- **waitForStatus timeout** → plain `Error('Polling timeout')`, NOT a RubicApiError — catch separately
- **Fee collection**: whitelisted EVM `integratorAddress` + `referrer` required; without it the default fixed native fee applies
- **0% protocol fees** on swaps < $100, most stable/native on-chain swaps, USDT→TRON bridges, all Solana swaps
- **MCP server**: read-only (quotes, search, tracking, URLs) without a key; signing needs local `EVM_WALLET_PRIVATE_KEY` (no `0x`), EVM only

## Integration Surface Selection

| Need | Surface |
|---|---|
| Node/TS app | SDK Lite `@cryptorubic/sdk-lite` |
| Edge runtime (Workers/Deno/RN) | SDK + custom fetch `HttpClient` |
| Any language / raw HTTP | REST API v2 |
| AI agent in MCP host | MCP `@cryptorubic/mcp` (local) or `mcp-api-v2.rubic.exchange/mcp` (hosted read-only) |
| Zero-code UI | `app.rubic.exchange` swap URL |

## Quickstart (30 seconds)

```typescript
import { SDK } from '@cryptorubic/sdk-lite';
const sdk = await SDK.create({ referrer: 'my-app' });

const quote = await sdk.quoteBest({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    srcTokenAmount: '1',
    dstTokenBlockchain: 'POLYGON',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
});
const swapData = await sdk.swap({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    srcTokenAmount: '1',
    dstTokenBlockchain: 'POLYGON',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    id: quote.id,
    fromAddress: '0xYourWallet',
    receiver: '0xDestination',
});
const tx = await signer.sendTransaction(swapData.transaction);
const done = await sdk.waitForStatus({ id: swapData.id, srcTxHash: tx.hash });
```

## Chapter Index

| # | Chapter | Key Content |
|---|---------|-------------|
| [ch01](chapters/ch01-getting-started.md) | Getting Started | SDK Lite install, SDK.create, 3-step flow, swapBest |
| [ch02](chapters/ch02-configuration.md) | Configuration | SdkParams, integratorAddress, custom HttpClient |
| [ch03](chapters/ch03-quoting-routes.md) | Quoting Routes | quoteBest/All/DepositTrades, params, REST endpoints, response shapes |
| [ch04](chapters/ch04-executing-swaps.md) | Executing Swaps | swap/swapBest/deposit trades, 4 execution models, approvals |
| [ch05](chapters/ch05-status-and-claims.md) | Status & Claims | waitForStatus, statuses, claim(), celerRefund, refunds |
| [ch06](chapters/ch06-chains-and-providers.md) | Chains & Providers | getChains/getProviders, chain list, provider blacklists |
| [ch07](chapters/ch07-errors-and-amount-changes.md) | Errors & Amount Changes | 6-digit error codes, RubicApiError, ±0.5% amount guard |
| [ch08](chapters/ch08-monetization-mcp-platform.md) | Monetization, MCP & Platform | Fee models, MCP tools/config, Private Mode, MEV protection |

## Topic Index

- **amount change** → ch07 (compare quote vs swap, ±0.5% threshold)
- **API key** → ch01, ch02 (from Rubic BD, optional)
- **approval** → ch04, ch07 (approvalAddress, error 3001/3002)
- **Arbitrum claim** → ch05 (READY_TO_CLAIM → sdk.claim())
- **Bitcoin swaps** → ch04, ch06 (non-EVM execution models)
- **blockchainId** → ch03, ch06 (EVM only, null for non-EVM)
- **Celer refund** → ch05, ch08 (manual refunds)
- **chains list** → ch06 (getChains, live query)
- **claim()** → ch05
- **cross-chain swap** → ch03, ch04
- **custom HTTP client** → ch02, ch07 (edge runtimes, error forwarding)
- **deposit trade** → ch03, ch04 (no source wallet, poll by id)
- **enableChecks** → ch03, ch04 (default true)
- **error codes** → ch07 (6-digit decoder)
- **fee models** → ch08 (percentage / fixed / mixed)
- **getChains** → ch06
- **getProviders** → ch06
- **healthcheck** → ch06
- **hosted MCP** → ch08 (mcp-api-v2.rubic.exchange/mcp)
- **integratorAddress** → ch02, ch08 (fee whitelisting)
- **MEV protection** → ch08 (bloXroute Private RPC)
- **MCP server** → ch08 (tools, config, security model)
- **native token sentinel** → ch01, ch03 (0xEeee...EEeE)
- **on-chain swap** → ch03, ch06
- **PENDING / LONG_PENDING** → ch05
- **Private Mode** → ch08 (privacy providers)
- **quote expiry** → ch07 (error 1003)
- **quoteAll / quoteBest** → ch03
- **rate limits** → ch03 (200/2hr without key)
- **READY_TO_CLAIM** → ch05 (Arbitrum Bridge claim)
- **REST API** → ch03, ch06 (raw endpoints)
- **referrer** → ch01, ch08 (required identifier)
- **refunds** → ch05, ch08 (automatic vs Celer manual)
- **RubicApiError** → ch07 (code, reason, traceId)
- **Rubic Widget (iframe embed)** → NOT in this skill (legacy skill content; see rubic.exchange widget docs)
- **showDangerousRoutes** → ch03
- **slippage** → ch03
- **Solana swaps** → ch03, ch06 (chain id 7565164, JUPITER)
- **swap / swapBest** → ch04
- **srcTxHash** → ch05 (required for decentralized providers)
- **statuses** → ch05 (terminal vs pending)
- **TRON / TON swaps** → ch04, ch06 (non-EVM execution)
- **waitForStatus** → ch05 (interval, timeout, onStatusUpdate)

## Supporting Files

- [glossary.md](glossary.md) — all key terms and types
- [patterns.md](patterns.md) — techniques: retry loops, approval flows, deposit trades
- [cheatsheet.md](cheatsheet.md) — decision rules, error decoder, status tree, thresholds

---

## Scope & Limits

Covers Rubic SDK Lite (`@cryptorubic/sdk-lite`), API v2, and the MCP server per the Cryptorubic/docs repo as of 2026-10-12. Chain/provider lists change constantly — query `getChains()` / `getProviders()` live. For the legacy iframe Widget embed and historical integrator contract addresses, see rubic.exchange docs. Internal Rubic contract details, specific bridge mechanics, and partner terms require Rubic BD contact (`t.me/RubicPartnership`).
