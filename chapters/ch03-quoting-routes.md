# Chapter 3: Quoting Routes

## Core Idea
`quoteBest` returns the single best route; `quoteAll` returns every route sorted by output; `quoteDepositTrades` returns routes where the user sends funds to a deposit address (no source wallet signature needed).

## Key Concepts
- **quoteBest** — single highest-output route; returns `QuoteResponseInterface` with `id` for the swap call
- **quoteAll** — all routes, sorted best-first; `{ routes[], failed? }` envelope
- **quoteDepositTrades** — deposit-based routes only (ChangeNOW, Exolix, SimpleSwap); `fromAddress` optional
- **QuoteResponseInterface** — id, providerType, swapType, estimate, fees, routing, warnings, useRubicContract
- **Native sentinel** — `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for native tokens
- **showDangerousRoutes** — filter routes with priceImpact >20% / <-5% or slippage >20%; default false
- **skipFeeProviders** — only routes where `fees.gasTokenFees.provider.fixedAmount = 0`

## Common Request Parameters

| Field | Type | Description |
|---|---|---|
| `srcTokenBlockchain` | BlockchainName | Source chain (e.g. `'ETH'`, `'BSC'`) |
| `srcTokenAddress` | string | Source token address; native sentinel `0xEeee...EEeE` |
| `srcTokenAmount` | string | Amount in token units (not wei) |
| `dstTokenBlockchain` | BlockchainName | Destination chain |
| `dstTokenAddress` | string | Destination token address |
| `fromAddress` | string | (optional) Sender wallet; required by some providers |
| `receiver` | string | (optional) Recipient on destination chain |
| `slippage` | number | (optional) Max slippage percent (`1` = 1%) |
| `integratorAddress` | string | (optional) Per-request fee override |
| `preferredProvider` | string | (optional) Force a provider |
| `enableChecks` | boolean | (optional) Balance/gas checks; default `true` |
| `showFailedRoutes` | boolean | (optional) Include failed routes in quoteAll; default `false` |

## REST API (raw) endpoints

```
POST https://api-v2.rubic.exchange/api/routes/quoteBest
POST https://api-v2.rubic.exchange/api/routes/quoteAll
GET  https://api-v2.rubic.exchange/api/info/chains?includeTestnets=false
GET  https://api-v2.rubic.exchange/api/info/status?srcTxHash=0x...
```

REST quote body wraps params: `{ params: { srcTokenAddress, srcTokenBlockchain, srcTokenAmount, dstTokenAddress, dstTokenBlockchain, referrer, ...optional } }`

## QuoteResponseInterface (key fields)

| Field | Type | Description |
|---|---|---|
| `id` | string | Trade identifier — pass to `swap()` |
| `providerType` | string | Bridge/DEX used (e.g. `'across'`, `'squidrouter'`) |
| `swapType` | `'cross-chain' \| 'on-chain'` | Swap type |
| `estimate.destinationTokenAmount` | string | Expected output |
| `estimate.destinationTokenMinAmount` | string | Min output after slippage |
| `estimate.durationInMinutes` | number | Estimated time |
| `estimate.priceImpact` | number \| null | Price impact percent |
| `routing[]` | RoutingInterface[] | Step-by-step path (type: on-chain/cross-chain, provider, path tokens) |
| `warnings[]` | ErrorInterface[] | Non-fatal warnings |

## Worked Example — quoteAll with comparison table

```typescript
const result = await sdk.quoteAll({
    srcTokenBlockchain: 'ETH',
    srcTokenAddress: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
    srcTokenAmount: '1000',
    dstTokenBlockchain: 'POLYGON',
    dstTokenAddress: '0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174', // USDC
    showFailedRoutes: true,
});

const best = result.routes[0];
console.log(best.providerType, best.estimate.destinationTokenAmount);

result.failed?.forEach(fail => {
    console.log(fail.providerType, 'failed:', fail.data.reason);
});

const comparison = result.routes.map(r => ({
    provider: r.providerType,
    receive: r.estimate.destinationTokenAmount,
    minReceive: r.estimate.destinationTokenMinAmount,
    estimatedTime: r.estimate.durationInMinutes + ' min',
    priceImpact: r.estimate.priceImpact + '%',
}));
console.table(comparison);
```

## Worked Example — deposit trade from Bitcoin (no wallet on source)

```typescript
const result = await sdk.quoteDepositTrades({
    srcTokenBlockchain: 'BTC',
    srcTokenAddress: '0x0000000000000000000000000000000000000000',
    srcTokenAmount: '0.1',
    dstTokenBlockchain: 'ETH',
    dstTokenAddress: '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE',
    receiver: '0xYourEthereumAddress',
});
const route = result.routes[0];
console.log('Provider:', route.providerType);
console.log('You receive:', route.estimate.destinationTokenAmount, 'ETH');
```

## Key Takeaways
1. Amounts are in **token units**, not wei — but the REST response also carries `destinationWeiAmount`
2. Quotes are **indicative** — swap recalculates and returns the authoritative amount (see Ch07 amount-change handling)
3. Deposit trades need NO source wallet — ideal for CEXes, hardware wallets, Bitcoin sources
4. `showDangerousRoutes: false` (default) hides priceImpact >20% / <-5% and slippage >20% routes

## Anti-patterns
- **Trusting the quote amount as final**: always compare quote vs swap amounts (±0.5% threshold, re-confirm with user)
- **Hardcoding the native-token zero address**: the SDK sentinel is `0xEeee...EEeE`; the REST API accepts `0x0000...0` — know which surface you're on

## Connects To
- **Ch04 Executing Swaps**: what to do with the quote `id`
- **Ch07 Error Handling & Amount Changes**: expired quotes (code 1003) and amount drift
- **Ch06 Chains & Providers**: what `getChains` returns for chain discovery
