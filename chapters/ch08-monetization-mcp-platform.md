# Chapter 8: Monetization, MCP Server & Platform

## Core Idea
Integrators earn fees via whitelisted `integratorAddress` + `referrer`; the MCP server exposes the whole API to AI agents (read-only without a key); Rubic aggregates 340+ DEXs/bridges across 70+ chains with private-mode routes and MEV protection.

## Key Concepts
- **Fee models** — Percentage-based, Fixed Amount (USD → native token), Mixed (percentage + fixed)
- **integratorAddress** — EVM wallet whitelisted by Rubic BD; collects fees
- **referrer** — identifies your integration; required for fee attribution
- **MCP server** — `@cryptorubic/mcp` via npx; read-only tools without a key, signing tools with `EVM_WALLET_PRIVATE_KEY`
- **Private Mode** — privacy providers (ClearSwap, Houdini Swap, RocketX, Railgun...) break the on-chain link between sender and receiver
- **MEV protection** — bloXroute Private RPC for ETH/BSC/Polygon swaps >$1,000
- **0% protocol fees** — swaps under $100; most stablecoin/native on-chain swaps; USDT→TRON bridges; all Solana swaps

## Monetization Models

| Model | Fee charged in | Example |
|---|---|---|
| **Percentage** | User's initial asset token | Swap 100 USDT with 1% commission → 99 USDT traded, 1 USDT to Rubic's contract (shared integrator/Rubic) |
| **Fixed Amount** | Native gas token of source chain (defined in USD, auto-converted) | USDT(Polygon)→ETH(Blast): fee paid in POL |
| **Mixed** | Percentage of initial asset + native gas token | DAI(BNB)→USDT(Optimism): fees in BNB + DAI |

### Setup process (EVM)
1. Provide an EVM fee-sharing address to Rubic BD (`t.me/RubicPartnership`)
2. Specify the fee model (percentage / fixed / mixed) and value
3. Rubic whitelists the address → set it as `integratorAddress` in requests
4. Without it: default fixed fee in native tokens deducted per transaction

### Deposit-based (non-EVM) fee sharing
- Providers: ChangeNOW, Changelly, Exolix, Near Intents
- Percentage-only; 1% total split 0.5% Rubic / 0.5% integrator
- Paid monthly in USDT on BNB Chain or Ethereum
- Always pass the whitelisted EVM `integratorAddress` + `referrer` even for non-EVM sources

## MCP Server

### Tools

| Tool | Requires EVM key | Description |
|---|---|---|
| `rubic_get_instructions` | — | Usage guide + workflow tips |
| `rubic_get_balances` | — | Non-zero balances across EVM chains |
| `rubic_get_supported_chains` | — | List supported chains |
| `rubic_search_tokens` | — | Search by symbol/name/address |
| `rubic_quote_routes` | — | Best or all routes |
| `rubic_simulate_swap` | — | Execution preview without signing |
| `rubic_build_swap_tx` | — | Build executable tx payload |
| `rubic_sign_tx` | ✅ | Sign EVM tx |
| `rubic_broadcast_tx` | — | Broadcast signed raw tx |
| `rubic_sign_and_broadcast_tx` | ✅ | Sign + broadcast in one call |
| `rubic_quote_swap_sign_and_broadcast_tx` | ✅ | Full flow |
| `rubic_track_status` | — | Track cross-chain status |
| `rubic_get_swap_url` | — | Pre-filled Rubic app swap URL |

### Config

```json
{
  "mcpServers": {
    "rubic": {
      "command": "npx",
      "args": ["-y", "@cryptorubic/mcp"],
      "env": { "EVM_WALLET_PRIVATE_KEY": "YOUR_PRIVATE_KEY" }
    }
  }
}
```
- Key format: EVM private key WITHOUT `0x`
- Read-only mode: omit the key (quotes, search, chains, swap URLs work)
- Hosted read-only endpoint: `https://mcp-api-v2.rubic.exchange/mcp`
- Local install: `git clone https://github.com/Cryptorubic/rubic-mcp.git && npm install && npm run build`; Docker: `rubicfinance/rubic-mcp:latest`

### MCP security model
- Non-custodial: keys never leave the machine (local `.env`), signing in-process via viem
- API never receives the private key
- No non-EVM signing: for Solana/Tron/TON/Bitcoin use `build_swap_tx` + external signing, or `get_swap_url`

### MCP error envelope

```json
{ "ok": true, "traceId": "uuid", "data": {}, "error": { "code": "RUBIC_1001", "message": "...", "statusCode": 400, "details": {} } }
```
Codes: `QUOTE_ROUTES_FAILED`, `ROUTE_ID_NOT_FOUND`, `BUILD_SWAP_TX_FAILED`, `SIGN_TX_FAILED`, `BROADCAST_TX_FAILED`, `WALLET_NOT_CONFIGURED`, `TOOL_TIMEOUT`, `HTTP_400`, `HTTP_NETWORK`, `RUBIC_<N>`, `INTERNAL_ERROR`

## Platform Facts

| Fact | Value |
|---|---|
| Scale | 340+ DEXs/bridges/intent protocols, 70+ chains, 70k tokens, 1M+ pairs |
| Min swap | ~$0.50 |
| Stable swaps | 1:1 stable-to-stable, no hidden fees |
| Security | Non-custodial; funds stay in user wallets; MixBytes + Stellar audits |
| KYC | Not required on Rubic; deposit providers (ChangeNOW etc.) may KYC flagged txs |
| Private mode | ClearSwap, Houdini Swap, RocketX, Railgun (beta: Zama/Hinkal disabled) |
| MEV protection | bloXroute Private RPC (ETH/BSC/Polygon, >$1,000 auto-prompt) |

### Private RPC endpoints (MEV protection)
- Ethereum: `https://rubic-eth.rpc.blxrbdn.com`
- BSC: `https://rubic-bnb.rpc.blxrbdn.com`
- Polygon: `https://rubic-polygon.rpc.blxrbdn.com`

## Worked Example — MCP agent swap workflow

```text
1. rubic_get_supported_chains        # List supported chains
2. rubic_search_tokens               # Get token addresses
3. rubic_quote_routes                # Get best route
4. rubic_simulate_swap (optional)    # Simulate transaction
5. rubic_build_swap_tx               # Build swap transaction
6. rubic_sign_and_broadcast_tx       # Execute transaction
7. rubic_track_status                # Track cross-chain progress
```

## Worked Example — swap URL fallback (no signing)

```json
{ "data": "https://app.rubic.exchange/?fromChain=ARBITRUM&from=ETH&to=USDC&toChain=ARBITRUM&amount=0.1" }
```

## Key Takeaways
1. Fee addresses must be whitelisted by Rubic BD BEFORE use — the defaults cost you the default fixed fee
2. Non-EVM deposit-based fees: percentage-only, monthly USDT payouts, always pass the EVM integratorAddress + referrer
3. MCP without a key = full read-only capability (quotes, search, tracking, URL generation)
4. `rubic_simulate_swap` returns a `summary` with `riskLevel` — use it as a pre-execution guard
5. 0% protocol fees under $100 makes small-trade integrations economically clean

## Anti-patterns
- **Skipping the BD onboarding** then wondering why fees vanish into the default recipient
- **Using MCP signing on non-EVM chains**: only EVM signing works; build + externally sign otherwise
- **Trusting quote guarantees**: quotes are estimates; execution price differs with slippage/MEV/market movement

## Connects To
- **Ch02 Configuration**: integratorAddress plumbing
- **Ch06 Chains**: proxyAvailable chains support fee collection
- **Ch07 Errors**: MCP error envelope codes
