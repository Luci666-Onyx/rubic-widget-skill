# Chapter 6: Chains, Providers & Info Methods

## Core Idea
`sdk.getChains()`, `sdk.getProviders()`, and `sdk.healthcheck()` return live metadata — use them instead of hardcoding chain IDs, provider lists, or token addresses.

## Key Concepts
- **getChains(includeTestnets?)** — `ChainInterface[]`; `id` is null for non-EVM chains
- **getProviders(includeTestnets?)** — `{ crossChain[], onChain[] }` providers with chain support
- **healthcheck()** — returns `'I am alive'` when the API is up
- **proxyAvailable** — whether Rubic can collect fees on that chain via proxy contracts
- **ChainInterface** — `name`, `type` ('evm' | 'solana' | 'tron' | 'ton' | 'bitcoin' | ...), `id`, `proxyAvailable`, `testnet`
- **ProvidersInterface** — `crossChain` (bridges/aggregators) + `onChain` (DEXes), each with `name` + `chains[]`
- **CrossChainTradeType** — `symbiosis`, `lifi`, `dln`, `bridgers`, `multichain`, `xy`, `celer_bridge`, `changenow`, `stargate`, `stargate_v2`, `arbitrum`, `squidrouter`, `scroll_bridge`, `taiko_bridge`, `rango`, `pulsechain_bridge`, `orbiter_bridge`, `owl_to_bridge`, `layerzero`, `archon_bridge`, `meson`, `eddy_bridge`, `router`
- **OnChainTradeType** — 130+ DEXes: `UNISWAP_V2`, `UNI_SWAP_V3`, `PANCAKE_SWAP`, `ONE_INCH`, `ODOS`, `JUPITER`, `OPEN_OCEAN`, `XY_DEX`, `LIFI`, `RANGO`, `DLN`, `SUSHI_SWAP`, `QUICK_SWAP`, `CAMELOT`, `IZUMI`, `SYNC_SWAP`, `REN_BTC`, ...

## ChainInterface

| Field | Type | Description |
|---|---|---|
| `name` | BlockchainName | Identifier used in all SDK requests (`'ETH'`, `'BSC'`, `'SOLANA'`) |
| `type` | string | `'evm'`, `'solana'`, `'tron'`, `'ton'`, `'bitcoin'`, etc. |
| `id` | number \| null | Chain ID (EVM only; `null` for non-EVM) |
| `proxyAvailable` | boolean | Fee collection via Rubic proxy contracts |
| `testnet` | boolean | Whether this is a testnet |

## Major supported chains (EVM ids from live docs)

ETH (1), OPTIMISM (10), ROOTSTOCK (30), BSC (56), GNOSIS (100), FUSE (122), POLYGON (137), FANTOM (250), ZK_SYNC (324), POLYGON_ZKEVM (1101), MOONBEAM (1284), MOONRIVER (1285), MANTLE (5000), BASE (8453), MODE (34443), ARBITRUM (42161), CELO (42220), AVALANCHE (43114), LINEA (59144), BLAST (81457), SCROLL (534352), AURORA (1313161554) — plus non-EVM: SOLANA, TRON, TON, BITCOIN, SUI, COSMOS.

> Chain lists change constantly — ALWAYS query `getChains()` at runtime rather than trusting this table.

## Code Example — chain/provider discovery

```typescript
const chains = await sdk.getChains();
chains.forEach(chain => {
    console.log(chain.name, chain.type, chain.id);
});
// → 'ETH'     'evm'     1
// → 'SOLANA'  'solana'  null

const evmChains = chains.filter(c => c.type === 'evm' && !c.testnet);

const providers = await sdk.getProviders();
providers.crossChain.forEach(p => {
    console.log(p.name, '→ supports', p.chains.length, 'chains');
});

// Check a specific provider supports a chain pair
const across = providers.crossChain.find(p => p.name === 'across');
const supportsEth = across?.chains.some(c => c.from === 'ETH' && c.to === 'ARBITRUM');
```

## REST info endpoints

```
GET https://api-v2.rubic.exchange/api/info/chains?includeTestnets=false
GET https://api-v2.rubic.exchange/api/info/providers
```

## Worked Example — safe routing against live data

```typescript
// 1. Validate the chain pair exists before quoting
const chains = await sdk.getChains();
const srcName = 'ARBITRUM', dstName = 'BSC';
const src = chains.find(c => c.name === srcName);
const dst = chains.find(c => c.name === dstName);
if (!src || !dst) throw new Error(`Unsupported chain: ${!src ? srcName : dstName}`);

// 2. Optionally restrict providers
const quote = await sdk.quoteBest({
    /* ... */
    nativeBlacklist: ['celer_bridge'],           // exclude native providers
    foreignBlacklist: { lifi: ['stargate'] },    // exclude foreign (lifi/rango) providers
    preferredProvider: 'squidrouter',
});
```

## Provider blacklist semantics

| Field | Applies to | Example |
|---|---|---|
| `nativeBlacklist` | Native providers (CrossChainTradeType list) | `["xy", "symbiosis"]` |
| `foreignBlacklist` | Foreign providers — `{ lifi?, rango? }` | `{ lifi: ["stargate"] }` |
| `preferredProvider` | Forces one provider for calculation | `lifi` |

## Key Takeaways
1. `getChains()` `id` is null for non-EVM chains — never assume a number
2. `proxyAvailable: false` chains (GNOSIS, FUSE, MOONBEAM, CELO) can't collect fees via proxy contracts — integrator fees may not apply
3. Query live metadata at runtime; the docs' chain list is a snapshot
4. `healthcheck()` before a batch of quotes gives you a fast fail on API downtime

## Anti-patterns
- **Hardcoding chain IDs from docs**: new chains ship constantly; IDs get reordered in responses
- **Assuming all chains support all providers**: e.g. CELO has crossChain providers but ZERO onChain providers
- **Treating `proxyAvailable: false` as "no fees"**: it means no fee collection via Rubic proxy contracts — integrator fees can't be collected there

## Connects To
- **Ch03 Quoting**: where chain names/tokens feed quote requests
- **Ch08 Monetization**: what proxyAvailable means for fee collection
