# Chapter 1: Getting Started — SDK Lite

## Core Idea
Rubic SDK Lite (`@cryptorubic/sdk-lite`) is a typed TypeScript wrapper around the Rubic API v2 — install it, call `SDK.create()`, then quote → swap → track.

## Key Concepts
- **SDK Lite** — `@cryptorubic/sdk-lite`, the official lightweight SDK (axios included)
- **referrer** — required identifier sent in every request; identifies your integration
- **apiKey** — optional; obtain via Rubic BD on Telegram (`t.me/RubicPartnership`)
- **integratorAddress** — crossChain + onChain fee recipient wallets (get whitelisted first)
- **waitForStatus** — built-in polling helper (interval 3s default, timeout 5min default)

## Mental Models
- **The 3-step flow is quote → swap → track**: quote calculates (no execution), swap returns calldata (you send it), waitForStatus polls until terminal
- **swapBest = quote + swap in one call** — use when you don't show the user a quote preview
- **The SDK never broadcasts**: it only builds `{ to, data, value }` — your signer sends the transaction

## Installation

```bash
npm install @cryptorubic/sdk-lite
# or: yarn add / pnpm add
```

## Code Example — full quickstart

```typescript
import { SDK } from '@cryptorubic/sdk-lite';

const sdk = await SDK.create({
    referrer: 'my-app',
    apiKey: process.env.RUBIC_API_KEY,        // optional
    integratorAddress: {
        crossChain: '0xYourFeeReceiverForCrossChain',
        onChain: '0xYourFeeReceiverForOnChain',
    },
    timeout: 15_000,
});

// 1. Quote
const quote = await sdk.quoteBest({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE', // native ETH
    srcTokenAmount: '1',
    dstTokenBlockchain: 'POLYGON',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE', // native MATIC
});
console.log('Provider:', quote.providerType, '| ID:', quote.id);

// 2. Swap (builds tx data)
const swapData = await sdk.swap({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    srcTokenAmount: '1',
    dstTokenBlockchain: 'POLYGON',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    id: quote.id,
    fromAddress: '0xYourWalletAddress',
    receiver: '0xDestinationAddress',
});
const tx = await signer.sendTransaction(swapData.transaction);

// 3. Track
const finalStatus = await sdk.waitForStatus(
    { id: swapData.id, srcTxHash: tx.hash },
    { interval: 5000, onStatusUpdate: s => console.log(s.status) }
);
```

## Worked Example — swapBest one-call

```typescript
const swapData = await sdk.swapBest({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    srcTokenAmount: '0.5',
    dstTokenBlockchain: 'BSC',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    fromAddress: '0xYourWalletAddress',
    receiver: '0xDestinationAddress',
});
// swapData.transaction = { to, data, value } — send with your signer
```

## Key Takeaways
1. Native token address is the sentinel `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` — not `0x0000...0`
2. `SDK.create()` is async (dynamic axios import) — always `await` it
3. `swapBest` skips the quote-inspection step — right for XO-swaps, wrong for UI route previews
4. Timeout default is 10s; raise it for slow quote calculations on congested networks

## Anti-patterns
- **Hardcoding token addresses**: use `sdk.getChains()` / token search instead — addresses change across deployments
- **Calling swap without a fresh quote id**: quote ids expire (error code 1003 "Trade expired") — re-quote

## Connects To
- **Ch02 Configuration**: custom HTTP clients, integrator addresses
- **Ch03 Quoting**: quoteBest / quoteAll parameter details
